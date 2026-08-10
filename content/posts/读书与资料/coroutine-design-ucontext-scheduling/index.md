---
title: 协程设计原理：从异步回调到 ucontext 调度
slug: coroutine-design-ucontext-scheduling
description: 从同步阻塞、异步回调和线程池出发，理解协程上下文、独立栈、setjmp、ucontext、yield、resume、I/O 调度与多核模式
date: 2026-08-10T00:00:00+08:00
draft: false
image: cover.svg
tags:
  - 协程
  - Linux
  - ucontext
  - 异步 I/O
  - Reactor
categories:
  - 后端开发
---

> 本篇承接已有的网络 I/O、epoll 与 Reactor 笔记。这份笔记重点理解：为什么需要协程、协程怎样把异步 I/O 写成同步流程、上下文切换的三种实现方式，以及 yield、resume 和调度器之间的关系。

---

## 一、这份笔记要解决的问题

以开源协程库 ntyco 为参照，尝试逐步理解并实现一套用户态协程框架。

完整学习路线包括：

~~~text
1. 为什么需要协程
2. 协程有哪些原语操作
3. 如何定义 struct coroutine
4. 如何定义 struct scheduler
5. 调度器采用什么策略
6. 如何兼容或封装 POSIX API
7. 协程从创建到退出的执行流程
8. 协程如何扩展到多核
9. 如何测试协程性能
~~~

当前真正落到代码上的内容主要是：

~~~text
异步回调为什么难写
  ↓
协程如何保留同步代码结构
  ↓
上下文切换需要保存什么
  ↓
setjmp/longjmp
  ↓
ucontext
  ↓
yield / resume
  ↓
最简单的轮询调度器
~~~

后续的 struct coroutine、struct scheduler、I/O 调度、多核模式和性能测试，是建立在这些基础之上的下一阶段。

---

## 二、协程要解决什么问题

### 2.1 同步请求

同步代码通常按顺序执行：

~~~c
func(){
send（）；
recv();
}
~~~

执行过程：

~~~text
send()发起请求
  ↓
recv()等待结果
  ↓
结果返回以后继续执行
~~~

优点是代码符合人的思考顺序：

~~~text
发请求 → 等结果 → 处理结果
~~~

缺点是等待 I/O 时，当前执行单元不能继续做其他工作。

### 2.2 异步回调

异步写法可以表示为：

~~~c
callback(){
 recv();
 send(fd,cb);
 }
 func{
 send(fd,callback)
 }
~~~

执行过程：

~~~text
func()发起请求
  ↓
立即返回，不阻塞当前流程
  ↓
fd就绪
  ↓
事件系统调用callback()
  ↓
callback()继续处理结果
~~~

异步的关键不是函数名字叫 async，而是：

> 发起操作和处理结果不在同一次连续调用栈中，中间通过事件和回调重新进入业务逻辑。

### 2.3 异步不等于并行

异步表示等待期间可以处理其他任务；并行表示多个任务在同一时刻由不同 CPU 核执行。

~~~text
异步 → 关注等待关系
并行 → 关注是否同时执行
~~~

单线程 epoll 也可以实现异步，但它并没有同时使用多个 CPU 核。

### 2.4 协程的目标

回调的性能模型很好，但业务流程一复杂，就会出现多层嵌套：

~~~text
请求A完成
  → 回调中发起请求B
      → B的回调中发起请求C
          → C的回调继续业务
~~~

协程希望实现：

~~~text
底层：仍然使用异步事件和非阻塞I/O
上层：代码看起来像同步顺序执行
~~~

即：

> 底层异步，上层同步写法。

---

## 三、线程池异步与单线程同步的对比

线程池任务示例：

~~~c
client_t *rClient = (client_t*)malloc(sizeof(client_t));
					memset(rClient, 0, sizeof(client_t));				
					rClient->fd = clientfd;
					
					job_t *job = malloc(sizeof(job_t));
					job->job_function = client_job;
					job->user_data = rClient;
					workqueue_add_job(&workqueue, job);
~~~

这里把 clientfd 封装进任务：

~~~text
rClient->fd = clientfd
  ↓
job->job_function = client_job
  ↓
job->user_data = rClient
  ↓
workqueue_add_job()
  ↓
工作线程取出任务并执行
~~~

两种结构如下。

### 3.1 同一个线程检测和处理 I/O

~~~c
while(1){
int nready =epll_wait();
for(int i=0;i<nready;i++){
recv(event[i].data.fd,buffer)
send(event[i]data.fd,buffer)
}
}
}
//单线程
~~~

流程：

~~~text
epoll_wait()
  ↓
遍历就绪fd
  ↓
当前线程执行recv/send
  ↓
处理完成后才能继续下一个事件
~~~

### 3.2 epoll 线程只分发任务

~~~c
void  *task_callback(void *arg){
	recv(event[i].data.fd,buffer)
send(event[i]data.fd,buffer)
}
while(1){
int nready =epll_wait();
for(int i=0;i<nready;i++){
task.fd=event[i].fd
task.callback=task_callback
push_task_to_threadpool(&task);
}//多线程
~~~

流程：

~~~text
epoll_wait()检测就绪
  ↓
把fd包装成task
  ↓
推入线程池
  ↓
工作线程执行recv/send
~~~

一次示例测试得到：

~~~text
线程池方式：每1000连接约1500ms
直接处理：每1000连接约7300ms
~~~

这个结果只能说明当前代码、机器和负载下，任务并行处理改善了吞吐。它不能直接推导出“所有异步程序一定比同步程序快六倍”。

性能差异还可能来自：

~~~text
任务中是否有阻塞操作
CPU核心数量
连接和请求模型
锁与队列开销
日志输出
测试方法
数据大小
~~~

协程与线程池也不是同一种实现：

~~~text
线程池 → 内核线程并行执行任务
协程   → 通常在线程内由用户态调度器协作切换
~~~

---

## 四、为什么互联网业务需要并发等待

一个网页可能需要同时请求多个动态接口：

~~~text
用户信息
文章列表
消息数量
推荐内容
广告数据
权限信息
~~~

如果完全串行：

~~~text
请求1完成
  ↓
请求2完成
  ↓
请求3完成
~~~

总等待时间会累积。

如果这些请求相互独立，可以在等待某个请求时推进其他请求：

~~~text
发起请求1 ─┐
发起请求2 ─┼→ 分别等待 → 谁先完成先处理谁
发起请求3 ─┘
~~~

同样的模型也会出现在：

~~~text
DNS查询
HTTP请求
数据库查询
Redis请求
Kafka或MongoDB访问
文件I/O
多个客户端连接
~~~

如果请求之间存在依赖：

~~~text
A的结果决定B的参数
B的结果决定C的参数
~~~

业务顺序仍然必须是 A → B → C。协程不能消除真实依赖，但可以在 A 等待期间调度其他不相关协程。

---

## 五、从回调流程演化到协程流程

### 5.1 回调写法

三个请求流程可以写成：

~~~c
请求流程1
callback(){
 recv();
 send(fd,cb);
 }
 func{
 send_http(fd,callback)
switch(1,2)
 }
~~~

~~~c
请求流程2
callback(){
 recv();
 send(fd,cb);
 }
 func{
 switch(2,3)
 send(fd,callback)
 }
~~~

~~~c
请求流程3
callback(){
 recv();
 send(fd,cb);
 }
 func{
 send(fd,callback)
 switch(3,1)
 }
~~~

业务被拆散到多个 callback 中，切换关系也混在业务代码里。

### 5.2 协程期望的写法

希望得到的目标形式：

~~~c
请求流程1

 func{
 setjmp();
asyn_send(fd,callback)

asyn_recv(fd,buffer)
 }
~~~

其他请求也保持相同的顺序结构：

~~~c
请求流程2

 func{
asyn_send(fd,callback)

asyn_recv(fd,buffer)
 }
~~~

~~~c
请求流程3

 func{
asyn_send(fd,callback)

asyn_recv(fd,buffer)
 }
~~~

表面看起来仍然是：

~~~text
发送
  ↓
接收
  ↓
处理
~~~

但 asyn_recv() 发现 fd 尚未就绪时，不阻塞整个线程，而是让出当前协程：

~~~c
async_xxx(){
if(1==poll(fd,0)){
switch();
}
}
~~~

它表达的思想是：

~~~text
I/O已经就绪
  → 当前协程继续执行

I/O尚未就绪
  → 保存当前协程上下文
  → yield到调度器
  → 调度其他协程
  → fd就绪后resume回来
~~~

所以协程不是把阻塞 I/O 变快，而是在等待发生时切走。

---

## 六、协程必须具备哪些原语

最基础的协程原语可以概括为：

~~~text
create  → 创建协程
resume  → 从调度器进入协程
yield   → 协程主动让出执行权
exit    → 协程执行结束
switch  → 保存一个上下文并恢复另一个上下文
~~~

与 I/O 调度结合后还需要：

~~~text
wait_io(fd, event) → 当前协程等待某个fd事件
wakeup(coroutine)  → fd就绪后把协程变为可运行
schedule()         → 选择下一个可运行协程
~~~

三者之间的关系：

~~~text
resume和yield → 面向协程业务语义
switch        → 底层上下文切换动作
schedule      → 决定下一次resume谁
~~~

---

## 七、上下文到底是什么

协程执行到一半切走，之后还要从原位置继续，因此必须保存执行现场。

上下文通常包括：

~~~text
程序计数器/指令位置
栈指针
通用寄存器
部分调用约定要求保存的寄存器
协程自己的栈
协程状态
~~~

如果只记住“执行到了哪一行”，但没有保存栈和寄存器，函数局部变量、返回地址和调用关系就无法恢复。

栈式协程通常需要：

~~~text
每个协程一个独立栈
每个协程一份上下文
一个调度器上下文
~~~

I/O 模型中常见的是：

~~~text
fd就绪事件
  → 找到等待这个fd的协程
  → 将协程加入ready队列
~~~

并不是 fd 本身天然等于协程；实现可以选择“一连接一协程”“一请求一协程”，也可以让一个协程等待多个事件。

---

## 八、实现上下文切换的三种方式

常见的实现方式：

~~~text
1. setjmp / longjmp
2. ucontext
3. 汇编保存和恢复寄存器，例如 ntyco
~~~

### 8.1 setjmp / longjmp

setjmp() 保存可以恢复的执行环境，longjmp() 跳回该位置。

它适合帮助理解“保存位置并跳回来”，但它本身不自动为每个协程创建独立栈。要实现完整栈式协程，还要管理协程栈和更完整的上下文。

### 8.2 ucontext

ucontext 提供：

~~~text
getcontext()  → 获取当前上下文
makecontext() → 为上下文指定入口函数
swapcontext() → 保存当前上下文并切换到目标上下文
setcontext()  → 直接恢复某个上下文
~~~

它非常适合教学，因为能够明确看到栈、入口函数和上下文切换。

ucontext 已从 POSIX.1-2008 中移除，不适合作为现代跨平台库的统一标准接口，但很多 Unix/Linux 环境仍然提供它。

### 8.3 汇编切换

汇编实现会按照目标架构和调用约定，直接保存、恢复必要寄存器和栈指针。

优点：

~~~text
可控制保存哪些寄存器
切换路径短
适合高性能协程库
~~~

代价：

~~~text
与CPU架构相关
与ABI和调用约定相关
移植成本更高
实现错误容易破坏栈
~~~

ntyco 采用汇编方式实现底层上下文切换。

---

## 九、setjmp / longjmp 原代码

setjmp / longjmp 示例代码：

~~~c
#include<setjmp.h>
#include<stdio.h>
jmp_buf env;//上下文环境
void func(int arg){
    printf("func: %d\n",arg);
    longjmp(env,++arg);
}
int main(){
    
    int ret=setjmp(env);
    if(ret==0){
        func(ret);
    }else if(ret==1){
        func(ret);
    }else if(ret==2){
        func(ret);
    }else if(ret==3){
        func(ret);
    }


    return 0;       

}
~~~

### 9.1 jmp_buf

~~~c
jmp_buf env;//上下文环境
~~~

env 用来保存 setjmp() 所需的执行环境。

### 9.2 第一次 setjmp()

~~~c
int ret=setjmp(env);
~~~

直接调用 setjmp() 时返回 0：

~~~text
ret = 0
  ↓
进入if(ret==0)
  ↓
func(0)
~~~

### 9.3 longjmp()

~~~c
longjmp(env,++arg);
~~~

func(0) 中 arg 先变成 1，再跳回 setjmp() 保存的位置。

这一次 setjmp() 看起来像重新返回，并得到：

~~~text
ret = 1
~~~

于是进入：

~~~c
}else if(ret==1){
    func(ret);
}
~~~

执行过程：

~~~text
setjmp首次返回0
  ↓
func(0)打印func: 0
  ↓ longjmp(...,1)
setjmp恢复后返回1
  ↓
func(1)打印func: 1
  ↓ longjmp(...,2)
setjmp恢复后返回2
  ↓
func(2)打印func: 2
  ↓ longjmp(...,3)
setjmp恢复后返回3
  ↓
func(3)打印func: 3
  ↓ longjmp(...,4)
setjmp恢复后返回4
  ↓
没有匹配分支，main结束
~~~

预期打印：

~~~text
func: 0
func: 1
func: 2
func: 3
~~~

这段代码展示的是控制流跳转，还不是完整的多协程调度器。

---

## 十、ucontext 原代码：三个协程和 main 上下文

ucontext 示例代码：

~~~c
#include<stdio.h>
#include<ucontext.h>

ucontext_t ctx[3];
ucontext_t main_ctx;
int count=0;
//coroutine1
void func1(void){
    while(count++<30){
    printf("1\n");
    //swapcontext(&ctx[0],&ctx[1]);
    swapcontext(&ctx[0],&main_ctx);
    printf("4\n");
    }

}
//coroutine2
void func2(void){
    while(count++<30){
    printf("2\n");
    //swapcontext(&ctx[1],&ctx[2]);
    swapcontext(&ctx[1],&main_ctx);
    printf("5\n");
    }



}
//coroutine3
void func3(void){
    while(count++<30){
    printf("3\n");
	//swapcontext(&ctx[2],&ctx[0]);
	swapcontext(&ctx[2],&main_ctx);
    printf("6\n");
    }



}
//schedule
int main(){

    char stack1[2048]={0};
    char stack2[2048]={0};
   char stack3[2048]={0}; 
    getcontext(&ctx[0]);
    ctx[0].uc_stack.ss_sp=stack1;
    ctx[0].uc_stack.ss_size=sizeof(stack1);
    ctx[0].uc_link=&main_ctx;
    makecontext(&ctx[0],func1,0);

    
    getcontext(&ctx[1]);
    ctx[1].uc_stack.ss_sp=stack2;
    ctx[1].uc_stack.ss_size=sizeof(stack2);
    ctx[1].uc_link=&main_ctx;
    makecontext(&ctx[1],func2,0); 

    getcontext(&ctx[1]);
    ctx[2].uc_stack.ss_sp=stack3;
    ctx[2].uc_stack.ss_size=sizeof(stack3);
    ctx[2].uc_link=&main_ctx;
    makecontext(&ctx[1],func3,0); 

    printf("swapcontext\n");
    swapcontext(&main_ctx,&ctx[0]);
    int i=30;
    while(count<30){
    swapcontext(&main_ctx,&ctx[count%3]);
    }
    printf("\n");






    return 0;
}
~~~

这段代码希望建立：

~~~text
main_ctx → 调度器上下文
ctx[0]   → func1协程
ctx[1]   → func2协程
ctx[2]   → func3协程
~~~

阅读第三个协程的初始化时，要按照这个对应关系理解：

~~~text
ctx[2]使用stack3
ctx[2]的入口函数是func3
~~~

---

## 十一、如何创建一个 ucontext 协程

以第一个协程为例。

### 11.1 保存初始上下文

~~~c
getcontext(&ctx[0]);
~~~

ctx[0] 获得一份可继续配置的上下文。

### 11.2 指定独立栈

~~~c
ctx[0].uc_stack.ss_sp=stack1;
ctx[0].uc_stack.ss_size=sizeof(stack1);
~~~

含义：

~~~text
stack1                  → 栈起始地址
sizeof(stack1)          → 栈大小
ctx[0].uc_stack         → ctx[0]使用的协程栈
~~~

三个协程分别使用：

~~~text
ctx[0] → stack1
ctx[1] → stack2
ctx[2] → stack3
~~~

独立栈让不同协程能够保存自己的函数调用关系和局部变量。

### 11.3 指定返回去向

~~~c
ctx[0].uc_link=&main_ctx;
~~~

如果 func1 正常执行完并返回，运行时会转到 main_ctx。

uc_link 处理的是入口函数正常返回后的去向；func1 中主动调用 swapcontext() 则是显式 yield。

### 11.4 指定入口函数

~~~c
makecontext(&ctx[0],func1,0);
~~~

含义：

~~~text
ctx[0]第一次被resume时
  ↓
从func1开始执行
  ↓
func1没有参数
~~~

最后一个 0 表示传给 func1 的参数数量为 0。

---

## 十二、swapcontext()：保存当前上下文并恢复另一个

函数形式：

~~~c
swapcontext(&from,&to);
~~~

可以理解成：

~~~text
把当前执行现场保存到from
  ↓
恢复to中的执行现场
  ↓
开始运行to
~~~

### 12.1 main resume 第一个协程

~~~c
swapcontext(&main_ctx,&ctx[0]);
~~~

含义：

~~~text
保存main当前执行位置到main_ctx
  ↓
恢复ctx[0]
  ↓
第一次进入func1
~~~

func1 打印：

~~~c
printf("1\n");
~~~

### 12.2 func1 yield 回 main

~~~c
swapcontext(&ctx[0],&main_ctx);
~~~

含义：

~~~text
保存func1当前执行位置到ctx[0]
  ↓
恢复main_ctx
  ↓
main从刚才swapcontext之后继续
~~~

func1 的执行位置停在：

~~~c
swapcontext(&ctx[0],&main_ctx);
~~~

下一次 main 再 resume ctx[0] 时，swapcontext() 返回，继续执行：

~~~c
printf("4\n");
~~~

这就是协程能够“从上次让出的位置继续”的原因。

---

## 十三、yield、resume 和 switch

三者的对应关系：

~~~text
main → ctx[0] → resume协程
ctx[0] → main → yield协程
swapcontext() → 底层switch动作
~~~

### 13.1 resume

~~~text
调度器选择一个协程
  ↓
从main_ctx切到该协程ctx
  ↓
协程开始或继续运行
~~~

### 13.2 yield

~~~text
协程遇到I/O等待或主动让出
  ↓
保存自己的ctx
  ↓
切回main_ctx
  ↓
调度器选择其他协程
~~~

### 13.3 switch

switch 不关心为什么切换，只负责：

~~~text
保存from
恢复to
~~~

resume 和 yield 是对 switch 在不同业务方向上的命名。

---

## 十四、当前调度器如何工作

原代码：

~~~c
while(count<30){
    swapcontext(&main_ctx,&ctx[count%3]);
}
~~~

count % 3 的结果不断在 0、1、2 之间变化：

~~~text
count % 3 == 0 → resume ctx[0]
count % 3 == 1 → resume ctx[1]
count % 3 == 2 → resume ctx[2]
~~~

因此它相当于一个简单轮询调度器：

~~~text
main
  ↓
ctx[0]运行一段后yield
  ↓
main
  ↓
ctx[1]运行一段后yield
  ↓
main
  ↓
ctx[2]运行一段后yield
  ↓
main继续轮询
~~~

这些上下文的栈关系可以画成：

~~~text
ctx3独立栈
ctx2独立栈
ctx1独立栈
main栈
~~~

这些栈并不是垂直嵌套成一条普通函数调用链，而是各自保存执行现场，由 swapcontext() 在它们之间切换。

---

## 十五、为什么当前轮询还不是 I/O 调度器

当前策略只根据：

~~~c
ctx[count%3]
~~~

选择协程，它不知道：

~~~text
哪个协程正在等待fd
哪个fd已经就绪
哪个协程已经结束
哪个协程可以立即运行
~~~

真正的 I/O 协程调度需要至少维护：

~~~text
ready队列   → 当前可以运行的协程
waiting集合 → 等待I/O、定时器或条件的协程
epoll       → 检测fd就绪
协程状态    → NEW、READY、RUNNING、WAITING、DEAD
~~~

典型过程：

~~~text
协程调用异步recv
  ↓
数据尚未就绪
  ↓
把“fd → 当前协程”注册给调度器
  ↓
当前协程状态变成WAITING
  ↓
yield回调度器
  ↓
调度器resume其他READY协程
  ↓
epoll报告fd可读
  ↓
对应协程进入READY队列
  ↓
调度器再次resume它
  ↓
recv继续执行
~~~

这个过程才真正实现：

> 代码看起来在等待 recv，但线程没有被这个协程阻塞。

---

## 十六、协程结构体需要保存什么

进一步实现时需要定义 struct coroutine。根据前面的执行过程，它至少需要表达：

~~~text
协程ID
协程状态
上下文
独立栈和栈大小
入口函数
入口参数
所属调度器
等待的fd和事件
退出结果或错误
~~~

概念结构：

~~~text
struct coroutine
  ├── id
  ├── state
  ├── context
  ├── stack
  ├── function
  ├── argument
  ├── waiting_fd
  └── scheduler
~~~

这里先理解字段为什么存在，不需要脱离后续 ntyco 代码自行发明另一套结构。

---

## 十七、调度器需要保存什么

调度器负责管理所有协程并决定下一次运行谁。

概念结构：

~~~text
struct scheduler
  ├── main_context
  ├── current_coroutine
  ├── ready_queue
  ├── waiting_coroutines
  ├── epoll_fd
  ├── timer
  └── all_coroutines
~~~

调度器要回答：

~~~text
当前运行的是谁
哪些协程可以运行
哪些协程在等待I/O
哪些协程等待超时
fd就绪后应该唤醒谁
协程退出后怎样回收
~~~

---

## 十八、如何封装 POSIX API

一个重要设计目标，是让协程网络库尽量保持熟悉的 POSIX 调用方式。

期望上层仍然写：

~~~text
connect
accept
recv
send
close
~~~

但协程封装后的行为是：

~~~text
调用recv
  ↓
先尝试非阻塞recv
  ├── 已就绪：直接返回结果
  └── EAGAIN：注册EPOLLIN并yield
                  ↓
               fd就绪
                  ↓
               resume协程
                  ↓
               再次尝试recv
~~~

这样业务代码保留同步风格，但底层仍然由 epoll 和协程调度实现异步等待。

“接口一致”不代表直接调用阻塞版本的 POSIX API，而是协程库提供行为兼容的包装层或 hook。

---

## 十九、协程怎样用于 WebServer

已有 Reactor WebServer 的逻辑：

~~~text
epoll_wait
  ↓
recv_cb
  ↓
http_request
  ↓
send_cb
  ↓
http_response
~~~

协程模型可以把一个连接或一个请求组织成顺序流程：

~~~text
协程开始
  ↓
等待并接收HTTP请求
  ↓
解析HTTP
  ↓
查询数据库或缓存
  ↓
构造HTTP响应
  ↓
发送响应
  ↓
等待下一次请求或退出
~~~

遇到 fd 尚未就绪时：

~~~text
当前协程yield
  ↓
线程继续处理其他连接协程
  ↓
fd就绪后resume
~~~

Reactor 与协程不是互斥关系：

~~~text
Reactor/epoll → 检测事件
协程调度器   → 把事件转换成协程唤醒
协程         → 承载顺序业务流程
~~~

---

## 二十、微信群聊场景应该怎样思考

思考这个场景时，不必急着寻找固定答案，先建立分析框架。

面对“微信群聊如何使用协程”时，可以依次问：

~~~text
1. 一个协程对应一个连接、一个用户，还是一次消息处理？
2. 连接没有消息时，协程处于什么状态？
3. fd可读后，谁把对应协程唤醒？
4. 一条消息需要广播给多人时，哪些步骤能够并发？
5. 慢客户端发送不出去时，是否会阻塞其他用户？
6. 每个连接的发送队列放在哪里？
7. 用户断开以后，协程、fd和缓冲区如何回收？
8. 心跳和超时由谁管理？
9. 群消息顺序如何保证？
10. 多核模式下，同一个连接能否跨调度器迁移？
~~~

可以先画：

~~~text
用户A连接协程
  ↓ 收到群消息
服务端业务处理
  ↓
找到群成员
  ↓
把消息加入B/C/D的发送队列
  ↓
对应连接可写时唤醒发送流程
~~~

重点不是简单回答“一个用户一个协程”，而是明确：

~~~text
协程在哪里等待
什么事件唤醒
状态保存在哪里
慢连接如何隔离
~~~

---

## 二十一、协程的多核模式

单个调度器通常运行在一个线程中，同一时刻只能使用一个 CPU 核。

多核模式常见思路：

~~~text
一个线程一个调度器
  ↓
每个调度器拥有自己的epoll和ready队列
  ↓
多个线程并行运行
~~~

这形成 M:N 模型：

~~~text
M个协程
  ↓ 用户态调度
N个内核线程
  ↓
多个CPU核
~~~

需要处理：

~~~text
连接如何分配到各调度器
协程是否允许跨线程迁移
跨线程唤醒
共享队列和锁
负载均衡
work stealing
线程局部状态
~~~

上下文通常只能在它所属的线程和栈环境中安全切换，不能随意把 setjmp/ucontext 保存的上下文拿到另一个线程恢复。

---

## 二十二、协程性能如何测试

不能只看一个“每 1000 连接耗时”。协程性能至少分成四层。

### 22.1 原语性能

~~~text
创建一个协程的耗时
一次yield/resume的耗时
每秒可以上下文切换多少次
销毁协程的耗时
~~~

### 22.2 内存

~~~text
每个协程结构体大小
每个协程栈大小
一万或百万协程的总内存
栈是否按需增长
~~~

### 22.3 I/O 服务性能

~~~text
并发连接数
QPS
吞吐量
P50/P95/P99延迟
超时和错误率
CPU使用率
上下文切换次数
~~~

### 22.4 对比基线

应在相同业务和负载下比较：

~~~text
单线程Reactor
Reactor + 线程池
Reactor + 协程
一连接一线程
~~~

只有测试业务、连接数、响应大小和日志配置相同，结果才有可比性。

---

## 二十三、核心总结

### 重点一：协程不是为了创造真正并行

~~~text
单线程协程仍然只使用一个CPU核
协程的主要价值是等待I/O时切换任务
多核需要多个调度线程配合
~~~

### 重点二：协程结合了两种优点

~~~text
上层：同步顺序代码，容易理解
底层：异步事件等待，不阻塞整个线程
~~~

### 重点三：上下文和独立栈是恢复执行的基础

~~~text
上下文 → 记录执行现场
协程栈 → 保存函数调用和局部变量
switch → 保存一个上下文，恢复另一个
~~~

### 重点四：yield、resume 与调度器

~~~text
resume → 调度器进入协程
yield  → 协程返回调度器
switch → 完成底层上下文切换
scheduler → 决定下一个运行谁
~~~

### 重点五：真正的 I/O 协程需要 epoll

~~~text
recv遇到EAGAIN
  ↓
协程注册等待EPOLLIN
  ↓
yield
  ↓
epoll报告fd可读
  ↓
协程进入ready队列
  ↓
resume
  ↓
继续recv和后续业务
~~~

最终理解：

> 协程不是替代 epoll，而是建立在事件驱动之上，把回调式异步流程重新组织成可顺序阅读的业务代码。

