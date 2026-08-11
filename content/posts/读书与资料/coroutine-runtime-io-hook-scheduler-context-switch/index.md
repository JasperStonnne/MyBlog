---
title: 协程运行时：IO Hook、调度器与汇编上下文切换
slug: coroutine-runtime-io-hook-scheduler-context-switch
description: 从 read/write Hook 出发，理解协程调度器如何组织 READY、WAITING 与 SLEEPING 状态，以及 x86-64 汇编如何完成上下文切换
date: 2026-08-11T00:00:00+08:00
draft: false
image: cover.svg
tags:
  - 协程
  - Linux
  - IO Hook
  - epoll
  - 汇编
categories:
  - 后端开发
---

> 本篇承接 yield、resume、ucontext 和简单轮询调度，重点理解阻塞式 I/O 的 Hook、协程状态管理、调度器与 epoll 的协作，以及 x86-64 汇编上下文切换。

---

## 一、从上下文切换走向完整协程运行时

已经掌握的两个方向：

~~~text
main → ctx   ：resume
ctx  → main  ：yield
~~~

它们底层都依赖一次 context switch：

~~~text
保存当前执行现场
  ↓
恢复目标执行现场
  ↓
目标协程从上次暂停位置继续
~~~

但只有 switch、resume 和 yield 还不够。真正处理网络 I/O 还需要回答：

~~~text
1. 普通read/recv没有数据时，怎样自动yield？
2. fd就绪以后，怎样找到并resume对应协程？
3. 十万个协程如何保存和组织？
4. READY、WAITING、SLEEPING、EXITED如何管理？
5. 定时器和I/O超时如何处理？
6. 上下文切换怎样用汇编实现？
7. 多核机器上怎样运行多个调度器？
~~~

最终需要形成：

~~~text
POSIX I/O调用
  ↓
Hook层
  ↓
非阻塞I/O + epoll
  ↓
yield / resume
  ↓
协程状态和调度器
  ↓
ucontext或汇编context switch
~~~

---

## 二、为什么需要 Hook I/O

一个协程中的业务代码希望保持顺序结构：

~~~c
while(1){
recv();
parser();
send();
}
~~~

阅读时仍然是：

~~~text
接收数据
  ↓
解析数据
  ↓
发送响应
~~~

但如果这里直接调用阻塞式 recv()，没有数据时会阻塞整个线程，线程内其他协程也无法运行。

协程库希望在不大改业务流程的情况下，把调用替换为自己的实现：

~~~text
业务调用recv/read
  ↓
进入协程库Hook函数
  ↓
先尝试非阻塞I/O
  ├── 已就绪：调用真实系统函数并返回
  └── 未就绪：注册epoll事件并yield
                    ↓
                 fd就绪
                    ↓
                 resume协程
                    ↓
                 再调用真实系统函数
~~~

这就是：

> 上层保持同步代码结构，底层通过 epoll 和协程实现异步等待。

---

## 三、Hook 的概念流程

笔记中的伪代码：

~~~c
recv（fd,buffer,length）{
struct pollfd fds[1]={0};
fds[0].fd=fd;
if(0<poll(fd,1,0)){//io可读
recv_f(fd,buffer,length);
}else{
epoll_ctl(epfd,EPOLL_CTL_ADD,fd,&ev);
swapcontext();

}
}
~~~

它表达了四步。

### 3.1 检查 fd 是否就绪

~~~c
poll(fd,1,0)
~~~

timeout 为 0，表示立即检查，不在 poll() 中等待。

~~~text
返回值 > 0 → 当前fd有事件
返回值 = 0 → 当前没有事件
返回值 < 0 → 调用发生错误
~~~

### 3.2 已就绪时调用真实函数

~~~c
recv_f(fd,buffer,length);
~~~

recv_f 指向真正的系统 recv，而不是 Hook 后的同名函数。

### 3.3 未就绪时注册 epoll

~~~c
epoll_ctl(epfd,EPOLL_CTL_ADD,fd,&ev);
~~~

同时记录：

~~~text
哪个fd
等待什么事件
哪个协程正在等待它
是否存在超时时间
~~~

### 3.4 当前协程 yield

~~~c
swapcontext();
~~~

这里代表概念上的 yield。实际实现需要明确传入当前协程上下文和调度器上下文。

~~~text
当前协程WAITING
  ↓
yield回调度器
  ↓
调度器运行其他READY协程
~~~

---

## 四、使用 dlsym 拦截 read 和 write

以下是提供的 Hook 代码：

~~~c
#define _GNU_SOURCE

#include <dlfcn.h>

#include <stdio.h>
#include <ucontext.h>
#include <string.h>
#include <unistd.h>
#include <fcntl.h>

#include <sys/socket.h>
#include <errno.h>
#include <netinet/in.h>


#include <pthread.h>
#include <sys/poll.h>
#include <sys/epoll.h>
~~~

### 4.1 为什么需要 _GNU_SOURCE

~~~c
#define _GNU_SOURCE
~~~

它在包含系统头文件前启用 GNU 扩展声明，使 dlsym()、RTLD_NEXT 等接口在相应环境中可见。

### 4.2 函数指针类型

原代码：

~~~c
typedef ssize_t (*read_t)(int fd, void *buf, size_t count);
read_t read_f = NULL;

typedef ssize_t (*write_t)(int fd, const void *buf, size_t count);
write_t write_f = NULL;
~~~

read_t 描述真实 read() 的函数类型：

~~~text
参数1：fd
参数2：接收缓冲区
参数3：最大读取长度
返回值：实际读取长度或错误
~~~

write_t 对应真实 write()。

~~~text
read_f  → 保存原始read地址
write_f → 保存原始write地址
~~~

---

## 五、init_hook() 如何取得真实函数

原代码：

~~~c
void init_hook(void) {

	if (!read_f) {
		read_f = dlsym(RTLD_NEXT, "read");
	}

	
	if (!write_f) {
		write_f = dlsym(RTLD_NEXT, "write");
	}

}
~~~

dlsym() 根据符号名查找函数地址：

~~~c
dlsym(RTLD_NEXT, "read");
~~~

RTLD_NEXT 的含义可以理解为：

> 从当前 Hook 定义之后继续寻找下一个名为 read 的实现。

因此调用关系是：

~~~text
业务代码调用read()
  ↓
进入当前文件定义的Hook read()
  ↓
Hook内部调用read_f()
  ↓
进入真正的系统read()
~~~

如果 Hook 内部再次直接调用 read()：

~~~text
Hook read()
  ↓
又进入Hook read()
  ↓
无限递归
~~~

所以必须使用 read_f 调用真实函数。

使用 dlsym() 时通常需要链接动态加载库：

~~~bash
gcc ... -ldl
~~~

---

## 六、read() Hook 的执行过程

原代码：

~~~c
ssize_t read(int fd, void *buf, size_t count) {

	struct pollfd fds[1] = {0};

	fds[0].fd = fd;
	fds[0].events = POLLIN;

	int res = poll(fds, 1, 0);
	if (res <= 0) { //不可读


		// fd --> epoll_ctl();

		swapcontext(); // fd --> ctx fd与context一对一
		
	}
	// io

	
	ssize_t ret = read_f(fd, buf, count);
	printf("read: %s\n", (char *)buf);
	return ret;

	
	
}
~~~

### 6.1 构造 pollfd

~~~c
struct pollfd fds[1] = {0};

fds[0].fd = fd;
fds[0].events = POLLIN;
~~~

表示只检查一个 fd 的可读事件。

### 6.2 立即检查

~~~c
int res = poll(fds, 1, 0);
~~~

第三个参数为 0：

~~~text
不等待
立即返回当前状态
~~~

### 6.3 不可读时暂停协程

~~~c
if (res <= 0) {
    // fd --> epoll_ctl();
    swapcontext();
}
~~~

完整运行时还需要在 yield 前完成：

~~~text
将fd设置为非阻塞
将fd注册到epoll
记录fd等待POLLIN/EPOLLIN
记录当前等待协程
设置可选超时时间
当前协程状态改为WAITING
yield到scheduler
~~~

fd 就绪后：

~~~text
epoll_wait返回fd
  ↓
找到等待它的协程
  ↓
协程进入READY队列
  ↓
scheduler resume协程
  ↓
从swapcontext()之后继续
  ↓
read_f()真正读取数据
~~~

### 6.4 调用真实 read

~~~c
ssize_t ret = read_f(fd, buf, count);
~~~

这一步才真正从内核读取数据。

poll 告诉程序“当前可能可读”，read_f 才完成实际读取。就绪状态可能在检查和读取之间变化，因此 fd 仍然应该是非阻塞的，并正确处理 EAGAIN。

---

## 七、write() Hook

原代码：

~~~c
ssize_t write(int fd, const void *buf, size_t count) {

	printf("write: %s\n", (const char *)buf);

	return write_f(fd, buf, count);
}
~~~

当前实现主要展示：

~~~text
业务调用write
  ↓
进入Hook write
  ↓
调用write_f进入真实write
~~~

完整协程版 write() 也需要处理：

~~~text
真实write成功一部分
  → 记录已写偏移，继续写剩余数据

真实write返回EAGAIN
  → 注册EPOLLOUT
  → 当前协程yield
  → fd可写后resume
~~~

Hook write() 内部使用 printf() 记录日志时要特别小心，因为 printf() 最终也可能调用 write()，从而再次进入 Hook。实际实现通常使用不会再次触发该 Hook 的日志路径，或者直接调用保存下来的真实 write_f。

---

## 八、Hook 示例怎样接入普通服务端

原代码：

~~~c
int main() {

	init_hook();

	int sockfd = socket(AF_INET, SOCK_STREAM, 0);

	struct sockaddr_in serveraddr;
	memset(&serveraddr, 0, sizeof(struct sockaddr_in));

	serveraddr.sin_family = AF_INET;
	serveraddr.sin_addr.s_addr = htonl(INADDR_ANY);
	serveraddr.sin_port = htons(2048);

	if (-1 == bind(sockfd, (struct sockaddr*)&serveraddr, sizeof(struct sockaddr))) {
		perror("bind");
		return -1;
	}

	listen(sockfd, 10);

	struct sockaddr_in clientaddr;
	socklen_t len = sizeof(clientaddr);
	int clientfd = accept(sockfd, (struct sockaddr*)&clientaddr, &len);
	printf("accept\n");

	while (1) {

		char buffer[128] = {0};
		int count = read(clientfd, buffer, 128);
		if (count == 0) {
			break;
		}
		write(clientfd, buffer, count);
		printf("sockfd: %d, clientfd: %d, count: %d, buffer: %s\n", sockfd, clientfd, count, buffer);

	}

	return 0;

}
~~~

业务代码仍然调用：

~~~c
read(clientfd, buffer, 128);
write(clientfd, buffer, count);
~~~

但由于程序中定义了同名 read() 和 write()，符号解析会先进入 Hook。

这正是 Hook 的价值：

~~~text
业务代码结构保持不变
  ↓
read/write调用被协程库接管
  ↓
等待I/O时自动yield
  ↓
就绪后自动resume
~~~

当前示例只接收一个 clientfd，重点是展示 Hook 链路。与真正协程服务器结合时，还需要 accept Hook、多连接协程、epoll mainloop 和 scheduler。

---

## 九、为什么第三方库也可以受益

例如本地程序使用 hiredis 连接 Redis。hiredis 内部会调用 read/write 或 send/recv 完成网络操作。

不修改 hiredis 源码时：

~~~text
业务调用hiredis
  ↓
hiredis内部调用read/write
  ↓
进入协程库Hook
  ↓
没有数据时yield
  ↓
Redis响应到达后resume
~~~

因此原本按照阻塞式 POSIX API 编写的库，有机会直接运行在协程调度环境中。

前提是 Hook 覆盖了它实际调用的 API，并正确处理：

~~~text
阻塞标志
超时
EINTR
EAGAIN
部分读写
close和fd复用
线程安全
~~~

---

## 十、如何定义一个协程

笔记中的初始设计：

~~~c
struct coroutinue{
int fd;
ucontext_t ctx;//stack ，stack_size,func包含在里面
void *arg

};
~~~

一个协程需要保存两类信息。

### 10.1 执行现场

~~~text
ctx         → 寄存器、栈指针、执行位置等上下文
stack       → 协程自己的栈
stack_size  → 栈大小
entry       → 协程入口函数
arg         → 入口参数
~~~

### 10.2 调度状态

~~~text
id          → 协程标识
state       → READY、RUNNING、WAITING等
fd          → 当前等待的I/O对象
events      → 等待EPOLLIN或EPOLLOUT
expire_at   → 超时时间或唤醒时间
scheduler   → 所属调度器
~~~

概念上的完整结构：

~~~text
struct coroutine
  ├── id
  ├── state
  ├── context
  ├── stack / stack_size
  ├── entry / arg
  ├── waiting_fd / events
  ├── expire_at
  ├── ready_node
  ├── wait_node
  ├── sleep_node
  └── scheduler
~~~

本篇沿用原笔记中的 coroutinue 拼写来对应代码，实际项目中可以统一为 coroutine。

---

## 十一、协程入口与创建

原始设计：

~~~c
void func(void){

}


typedef void *(*coroutinue_entry)(void *)
int create_coroutinue(co_id id,coroutinue_entry entry,void *arg){
struct coroutinue *co =malloc(sizeof(struct coroutinue));
co.ss_sp=
makecontext(co->ctx,func,0);
}
~~~

创建协程需要完成：

~~~text
1. 分配struct coroutine
2. 分配或绑定独立栈
3. 保存entry和arg
4. 初始化context
5. 设置入口包装函数
6. 设置协程退出后的返回目标
7. state设为READY
8. 加入ready_queue
~~~

为什么通常需要一个入口包装函数：

~~~text
scheduler resume新协程
  ↓
进入wrapper
  ↓
wrapper调用entry(arg)
  ↓
entry返回
  ↓
wrapper把协程状态设为EXITED
  ↓
yield回scheduler
  ↓
scheduler回收资源
~~~

如果直接让入口函数随意 return，而没有统一退出路径，调度器很难正确修改状态和释放栈。

---

## 十二、十万个协程如何组织

假设采用“一连接一协程”：

~~~text
10万个fd
  ↓
10万个coroutine对象
~~~

但它们不会全部同时运行。

例如：

~~~text
10万个协程
  ├── 2万个READY
  ├── 5万个WAITING
  ├── 2万个SLEEPING
  └── 1万个已结束或待回收
~~~

调度器必须按状态组织它们，不能每轮扫描全部十万个对象。

---

## 十三、READY 为什么使用队列

笔记中的设计：

~~~c
queue_node (corountine,)ready_queue
~~~

READY 协程表示现在就可以运行。

常见操作：

~~~text
协程变为READY → 从队尾加入
scheduler取任务 → 从队头取出
协程yield但仍可运行 → 再放回队尾
~~~

因此队列适合：

~~~text
O(1)入队
O(1)出队
天然支持轮询公平调度
~~~

如果有 2 万个 READY 协程：

~~~text
ready_head
  → co1 → co2 → co3 → ... → co20000
~~~

scheduler 每次取队头协程 resume。

---

## 十四、WAITING 与 SLEEPING 为什么需要有序结构

笔记中的设计：

~~~c
rbtree_node(coroutinue,)wait_rb;
rbtree_node(coroutinue,)sleep_rb;
~~~

等待 I/O 的协程可能还设置超时：

~~~text
协程A等待fd=4，100ms后超时
协程B等待fd=5，500ms后超时
协程C等待fd=6，50ms后超时
~~~

睡眠协程也有不同唤醒时间：

~~~text
协程D睡眠到10:00:00.100
协程E睡眠到10:00:00.800
~~~

如果以 expire_at 为 key 放入有序结构，调度器可以快速找到最早到期节点：

~~~text
最小expire_at
  ↓
与当前时间比较
  ↓
已经到期就移出
  ↓
状态改为READY
  ↓
加入ready_queue
~~~

红黑树、最小堆、跳表、时间轮都可以实现定时管理，具体选择取决于插入、删除、精度和规模要求。

I/O fd 到协程的查找，还可以另外使用：

~~~text
epoll_event.data.ptr
哈希表 fd → coroutine
数组 fd → coroutine
~~~

时间树解决“谁先超时”，fd 映射解决“哪个事件唤醒谁”，它们是两个不同查找维度。

---

## 十五、一个协程为什么可以同时拥有多个节点

笔记中的思路是把数据结构节点直接放进 coroutine：

~~~text
ready_node
wait_node
sleep_node
~~~

这种方式称为侵入式数据结构：

~~~text
coroutine对象本身包含链表节点或树节点
  ↓
不需要额外分配包装节点
  ↓
从节点可以找回所属coroutine
~~~

但协程状态必须保持一致：

~~~text
READY    → 位于ready_queue
WAITING  → 位于I/O等待集合，可能同时位于超时树
SLEEPING → 位于sleep定时结构
RUNNING  → 当前正在执行，通常不在ready_queue
EXITED   → 位于回收集合或等待立即回收
~~~

协程从一个状态切到另一个状态时，需要从旧集合移除，再加入新集合，避免同一个节点被重复挂载。

---

## 十六、scheduler 如何定义

笔记中的设计：

~~~c
scheduler
struct scheduler{
int epfd
struct epoll_event events[];

queue_node (corountine,)ready_head
rbtree_root(coroutinue,)wait_rb;
rbtree_root(coroutinue,)sleep_rb;

};
~~~

各字段含义：

~~~text
epfd       → epoll实例
events[]   → epoll_wait返回的就绪事件
ready_head → 当前可以运行的协程
wait_rb    → 等待I/O并带超时的协程
sleep_rb   → 主动睡眠的协程
~~~

完整调度器还会需要：

~~~text
main_context      → scheduler自己的上下文
current           → 当前正在运行的协程
all_coroutines    → 所有协程的所有权集合
exited_queue      → 等待回收的协程
stop              → 退出标志
thread_id         → 所属线程
~~~

---

## 十七、哪些操作由协程执行，哪些由调度器执行

### 17.1 协程主动执行

~~~text
调用read/recv/send/write
发现需要等待后yield
主动sleep
业务处理
入口函数返回并exit
~~~

### 17.2 调度器执行

~~~text
维护ready_queue
计算最近超时时间
调用epoll_wait
处理fd就绪事件
处理wait/sleep超时
把协程转为READY
选择下一个协程resume
回收EXITED协程
~~~

核心边界：

> 协程表达“我要等待什么”，调度器决定“等待期间运行谁，以及何时唤醒我”。

---

## 十八、调度循环的完整流程

一个合理的 scheduler mainloop 可以概括为：

~~~text
while没有停止：

  1. 处理已经到期的sleep节点
  2. 处理已经超时的I/O等待节点
  3. 处理ready_queue中的协程
  4. 根据最近定时器计算epoll_wait超时
  5. 调用epoll_wait
  6. 根据events找到等待协程
  7. 将就绪协程加入ready_queue
  8. 回收已经退出的协程
~~~

如果 ready_queue 非空：

~~~text
可以先继续运行READY协程
epoll_wait使用timeout=0避免阻塞
~~~

如果 ready_queue 为空：

~~~text
根据最近timer计算等待时间
  ↓
epoll_wait可以阻塞到I/O到达或定时器到期
~~~

这样线程不会忙等。

---

## 十九、协程状态怎样流转

典型状态：

~~~text
NEW
  ↓ create
READY
  ↓ scheduler resume
RUNNING
  ├── 等待I/O → WAITING
  ├── sleep   → SLEEPING
  ├── 主动yield且仍可运行 → READY
  └── entry返回 → EXITED

WAITING
  ├── fd就绪 → READY
  └── 超时   → READY，并携带timeout结果

SLEEPING
  └── 时间到 → READY

EXITED
  ↓ scheduler回收
DESTROYED
~~~

状态与容器必须同步：

~~~text
state = READY
  ↔ 在ready_queue中

state = WAITING
  ↔ 在I/O等待映射中

state = SLEEPING
  ↔ 在timer结构中
~~~

---

## 二十、setjmp、ucontext 和汇编怎样选择

三种方案：

~~~text
setjmp / longjmp
ucontext
architecture-specific assembly
~~~

### 20.1 setjmp / longjmp

优点：

~~~text
标准C接口
可移植性相对好
适合理解保存点和恢复点
~~~

限制：

~~~text
不直接提供独立协程栈管理
构造完整栈式协程较复杂
~~~

### 20.2 ucontext

优点：

~~~text
getcontext/makecontext/swapcontext语义清晰
可以直接指定独立栈和入口函数
教学和原型实现简单
~~~

限制：

~~~text
已经不属于现代POSIX标准
不同平台支持情况不一致
~~~

### 20.3 汇编

优点：

~~~text
直接控制保存和恢复的寄存器
切换路径短
适合性能敏感的运行时
~~~

限制：

~~~text
与CPU架构、ABI和调用约定绑定
x86-64、ARM64等需要不同实现
测试和维护成本高
~~~

性能不能只靠固定排名判断，需要在相同编译器、架构和切换模型下基准测试。汇编通常能做得更精简，但正确性和可移植性同样重要。

---

## 二十一、x86-64 汇编上下文切换原代码

~~~c
#elif defined(__x86_64__)

__asm__ (
"    .text                                  \n"
"       .p2align 4,,15                                   \n"
".globl _switch                                          \n"
".globl __switch                                         \n"
"_switch:                                                \n"
"__switch:                                               \n"
"       movq %rsp, 0(%rsi)      # save stack_pointer     \n"
"       movq %rbp, 8(%rsi)      # save frame_pointer     \n"
"       movq (%rsp), %rax       # save insn_pointer      \n"
"       movq %rax, 16(%rsi)                              \n"
"       movq %rbx, 24(%rsi)     # save rbx,r12-r15       \n"
"       movq %r12, 32(%rsi)                              \n"
"       movq %r13, 40(%rsi)                              \n"
"       movq %r14, 48(%rsi)                              \n"
"       movq %r15, 56(%rsi)                              \n"
"       movq 56(%rdi), %r15                              \n"
"       movq 48(%rdi), %r14                              \n"
"       movq 40(%rdi), %r13     # restore rbx,r12-r15    \n"
"       movq 32(%rdi), %r12                              \n"
"       movq 24(%rdi), %rbx                              \n"
"       movq 8(%rdi), %rbp      # restore frame_pointer  \n"
"       movq 0(%rdi), %rsp      # restore stack_pointer  \n"
"       movq 16(%rdi), %rax     # restore insn_pointer   \n"
"       movq %rax, (%rsp)                                \n"
"       ret                                              \n"
);
#endif 
~~~

对应的上下文结构：

~~~c
typedef struct _nty_cpu_ctx {
	void *esp; //
	void *ebp;
	void *eip;
	void *edi;
	void *esi;
	void *ebx;
	void *r1;
	void *r2;
	void *r3;
	void *r4;
	void *r5;
} nty_cpu_ctx;



int _switch(nty_cpu_ctx *new_ctx, nty_cpu_ctx *cur_ctx);
~~~

函数语义：

~~~text
new_ctx → 即将恢复的目标上下文
cur_ctx → 当前需要保存的上下文
~~~

在 System V AMD64 ABI 中，前两个整数或指针参数通常放在：

~~~text
第一个参数new_ctx → RDI
第二个参数cur_ctx → RSI
~~~

因此：

~~~text
RSI作为基地址 → 保存当前上下文
RDI作为基地址 → 加载目标上下文
~~~

---

## 二十二、汇编第一部分：保存当前上下文

### 22.1 保存栈指针

~~~asm
movq %rsp, 0(%rsi)
~~~

含义：

~~~text
把当前RSP保存到cur_ctx偏移0的位置
~~~

RSP 指向当前线程正在使用的栈顶。恢复 RSP 才能回到该协程自己的栈。

### 22.2 保存帧指针

~~~asm
movq %rbp, 8(%rsi)
~~~

一个指针占 8 字节，所以第二个字段位于偏移 8。

### 22.3 保存返回地址

~~~asm
movq (%rsp), %rax
movq %rax, 16(%rsi)
~~~

调用 _switch 时，返回地址位于当前栈顶。先读到 RAX，再保存到 cur_ctx 偏移 16。

这个地址就是以后恢复协程时需要继续执行的位置。

### 22.4 保存被调用者保存寄存器

~~~asm
movq %rbx, 24(%rsi)
movq %r12, 32(%rsi)
movq %r13, 40(%rsi)
movq %r14, 48(%rsi)
movq %r15, 56(%rsi)
~~~

这些寄存器按照 x86-64 System V ABI 属于 callee-saved registers。函数返回后调用方有权期待它们保持不变，因此 context switch 必须保存和恢复。

偏移每次增加 8：

~~~text
0  → rsp
8  → rbp
16 → rip/返回地址
24 → rbx
32 → r12
40 → r13
48 → r14
56 → r15
~~~

原结构中的 esp、ebp、eip 是 32 位风格名称；在这段 x86-64 汇编中实际保存的是 rsp、rbp 和返回地址。

---

## 二十三、汇编第二部分：恢复目标上下文

~~~asm
movq 56(%rdi), %r15
movq 48(%rdi), %r14
movq 40(%rdi), %r13
movq 32(%rdi), %r12
movq 24(%rdi), %rbx
movq 8(%rdi), %rbp
movq 0(%rdi), %rsp
movq 16(%rdi), %rax
movq %rax, (%rsp)
ret
~~~

这部分按相反方向加载 new_ctx：

~~~text
恢复r15、r14、r13、r12、rbx
  ↓
恢复rbp
  ↓
恢复目标协程rsp
  ↓
把目标返回地址放到目标栈顶
  ↓
ret弹出返回地址
  ↓
从目标协程保存位置继续执行
~~~

最关键的一步：

~~~asm
movq 0(%rdi), %rsp
~~~

RSP 一旦切换，当前使用的栈就从旧协程栈变成了目标协程栈。

最后：

~~~asm
ret
~~~

从目标栈弹出保存的指令地址，于是代码看起来像目标协程之前调用的 _switch() 正常返回。

---

## 二十四、为什么不是保存所有寄存器

x86-64 有更多通用寄存器，但按照函数调用约定分为：

~~~text
caller-saved → 调用方负责在需要时保存
callee-saved → 被调用函数必须保证返回后不变
~~~

_switch() 以普通函数调用形式进入，编译器会按照 ABI 处理 caller-saved 寄存器中的临时值，因此最小切换实现主要保存 callee-saved 寄存器、栈指针和返回位置。

这依赖：

~~~text
明确的ABI
正确的函数声明
编译器遵循调用约定
汇编与结构体布局完全一致
~~~

System V AMD64 中，前六个整数或指针参数通常依次使用：

~~~text
RDI、RSI、RDX、RCX、R8、R9
~~~

函数并不是不能超过六个参数。超过六个的参数会按照 ABI 放到栈上，只是调用方式不再全部使用参数寄存器。

---

## 二十五、从 Hook read 到调度器唤醒的完整链路

~~~text
协程A调用read(fd)
  ↓
进入Hook read
  ↓
poll或真实read发现暂时不可读
  ↓
fd注册EPOLLIN
  ↓
记录fd → 协程A
  ↓
协程A状态RUNNING → WAITING
  ↓
_switch(scheduler_ctx, coA_ctx)
  ↓
scheduler运行其他READY协程
  ↓
epoll_wait返回fd
  ↓
根据event找到协程A
  ↓
从等待集合删除协程A
  ↓
协程A状态WAITING → READY
  ↓
加入ready_queue
  ↓
scheduler选择协程A
  ↓
_switch(coA_ctx, scheduler_ctx)
  ↓
协程A从Hook read中的yield位置继续
  ↓
调用read_f真正读取
  ↓
返回业务代码
~~~

这一整条链路把四个模块连接起来：

~~~text
Hook层
epoll事件层
scheduler调度层
context switch底层
~~~

---

## 二十六、多线程与多进程模式

### 26.1 多线程模式

一种设计是多个线程各自运行调度器：

~~~text
线程1 → scheduler1 → epoll1 → 一组协程
线程2 → scheduler2 → epoll2 → 一组协程
线程3 → scheduler3 → epoll3 → 一组协程
~~~

这通常比多个线程共同操作一个 scheduler 更容易控制，因为每个协程和上下文固定属于一个线程。

如果多个线程共享：

~~~text
ready_queue
wait tree
sleep tree
coroutine对象
~~~

就需要考虑锁、并发状态迁移、跨线程唤醒和缓存竞争。

CPU affinity 可以让调度线程尽量固定在某个 CPU 核，减少迁移和缓存失效，但是否绑定需要通过测试决定。

### 26.2 多进程模式

~~~text
进程1 → scheduler1 → 一组连接
进程2 → scheduler2 → 一组连接
进程3 → scheduler3 → 一组连接
~~~

每个进程拥有独立地址空间和调度器，协程数据天然隔离，减少共享锁。

需要额外考虑：

~~~text
监听端口如何共享
连接如何负载均衡
进程间通信
共享状态
进程崩溃恢复
~~~

多线程和多进程都能利用多核。区别主要在共享内存、隔离性和通信成本。

---

## 二十七、怎样把协程能力用到实际项目

### 27.1 网络框架

以 ntyco 为基础理解：

~~~text
coroutine
scheduler
epoll
Hook
timer
assembly switch
~~~

重点不是只复现 switch，而是打通：

~~~text
I/O等待 → yield → epoll唤醒 → ready → resume
~~~

### 27.2 WebServer

~~~text
一个连接或请求由一个协程处理
  ↓
协程recv HTTP请求
  ↓
解析协议
  ↓
查询Redis/MySQL
  ↓
发送HTTP响应
~~~

每个等待点都由 Hook 和 scheduler 自动让出。

### 27.3 KV 存储

例如 dkvstore：

~~~text
连接协程
  ↓
接收命令
  ↓
解析SET/GET
  ↓
访问存储结构
  ↓
返回结果
~~~

### 27.4 图床

~~~text
上传连接
  ↓
接收文件数据
  ↓
写磁盘或对象存储
  ↓
写数据库
  ↓
返回图片URL
~~~

这些项目能够体现的不是“使用了协程”一句话，而是：

~~~text
怎样Hook阻塞API
怎样管理十万连接状态
怎样设计ready/wait/sleep结构
怎样处理超时和部分读写
怎样在多核上部署调度器
怎样测试切换和网络性能
~~~

---

## 二十八、核心总结

### 重点一：Hook 让同步代码接入异步调度

~~~text
业务仍然调用read/write
  ↓
Hook判断I/O状态
  ↓
不可用时注册epoll并yield
  ↓
就绪后resume
~~~

### 重点二：协程对象既保存执行现场，也保存调度状态

~~~text
context/stack → 回来后从哪里继续
state/fd/time → 什么时候可以回来
data nodes    → 当前属于哪个调度集合
~~~

### 重点三：不同状态需要不同数据结构

~~~text
READY    → queue
I/O映射  → epoll data、数组或哈希
超时/睡眠 → 红黑树、最小堆或时间轮
EXITED   → 回收队列
~~~

### 重点四：scheduler 是整个运行时的中心

~~~text
处理epoll事件
处理定时器
维护协程状态
选择READY协程
执行resume
接收yield
回收退出协程
~~~

### 重点五：汇编 switch 只做保存和恢复

~~~text
RSI → 保存当前上下文
RDI → 恢复目标上下文
RSP → 切换协程栈
ret → 跳回目标协程保存位置
~~~

最终可以把协程运行时理解成：

> Hook 决定何时等待，epoll 决定何时就绪，scheduler 决定运行谁，context switch 决定怎样切过去。
