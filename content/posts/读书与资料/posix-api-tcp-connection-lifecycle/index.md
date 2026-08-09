---
title: POSIX API 与 TCP 连接生命周期
slug: posix-api-tcp-connection-lifecycle
description: 从 POSIX Socket API 出发，理解 socket、bind、listen、connect、accept、send、recv 与 close 背后的 TCP 三次握手、连接队列、可靠传输和四次挥手
date: 2026-08-10T00:00:00+08:00
draft: false
image: cover.svg
tags:
  - 网络编程
  - Linux
  - POSIX
  - TCP
  - Socket
categories:
  - 后端开发
---

> 本篇承接已有的《网络IO》《IO 多路复用与 Reactor》《Reactor 与 epoll：百万并发网络服务器课后笔记》。  
> socket、fd、服务端基础代码、select/poll/epoll 和 Reactor 不再重复；

---

## 一、为什么网络代码可以跨 Linux、Unix 和 macOS

Linux 受到 Unix 设计影响，不同厂商和社区又形成了不同发行版。它们的内核、工具和扩展功能可能不同，但应用程序仍然可以使用一组相似的接口：

~~~c
socket();
bind();
listen();
accept();
connect();
send();
recv();
close();
~~~

重要原因是这些系统广泛支持 POSIX 和 BSD Socket 风格的接口规范。

标准接口主要约定：

~~~text
函数叫什么
函数接收什么参数
返回值如何表达成功和失败
错误通过什么方式报告
应用程序可以依赖哪些行为
~~~

不同系统的内核实现可以不同，但只要对应用程序提供兼容接口，使用标准接口编写的代码就比较容易迁移：

~~~text
应用程序
  ↓
POSIX / Socket API
  ↓
不同操作系统的内核实现
~~~

代码可以直接迁移还要求它没有依赖某个系统的专有能力。例如 socket、bind、listen 是通用 Socket API；epoll 是 Linux 专用机制，迁移到 macOS 时通常要换成 kqueue。

---

## 二、客户端和服务端使用的 API

### 2.1 TCP 客户端

课堂中的客户端调用顺序：

~~~c
socket();
bind();//optional
connect();
send();
recv();
close();
~~~

对应作用：

~~~text
socket()  → 创建 socket 和 fd
bind()    → 可选，指定本地 IP 和本地端口
connect() → 指定对端并发起 TCP 连接
send()    → 把应用数据交给内核
recv()    → 从内核取出收到的数据
close()   → 关闭 fd，并推动 TCP 关闭过程
~~~

客户端通常不主动调用 bind()。如果直接 connect()，内核会选择合适的本地 IP 和临时端口。

临时端口范围由系统配置决定，不固定为 1024～65535。Linux 上可以查看：

~~~bash
cat /proc/sys/net/ipv4/ip_local_port_range
~~~

客户端需要固定源 IP 或源端口时，才会在 connect() 前显式调用 bind()。

### 2.2 TCP 服务端

课堂中的服务端调用顺序：

~~~c
socket();
bind();
listen();
accept();
recv();
send();
close();
~~~

与 Reactor 结合后还会使用：

~~~c
epoll_create();
epoll_ctl();
epoll_wait();
fcntl();
~~~

这两组 API 位于不同层次：

~~~text
socket/bind/listen/accept/connect/send/recv/close
  → 管理 TCP socket 和数据传输

epoll_create/epoll_ctl/epoll_wait
  → 管理大量 fd 的就绪事件
~~~

fcntl() 常用于把 fd 设置为非阻塞。

---

## 三、一次 TCP 连接的三个阶段

~~~text
1. 建立连接：三次握手
2. 数据传输：序号、确认、流量控制、拥塞控制和重传
3. 断开连接：FIN/ACK 与 TCP 状态迁移
~~~

应用程序看到的是：

~~~text
connect() / accept()
send() / recv()
close()
~~~

内核真正执行的是 TCP 协议状态机。

---

## 四、socket() 在应用层与内核之间建立了什么

课堂代码：

~~~c
fd=socket();
~~~

socket 的英文含义是插座。可以用“插头和插座”帮助理解应用层 fd 与内核 socket 对象的关系：

~~~text
应用层
  fd：应用程序使用的整数句柄
    ↓
内核
  socket/TCP相关对象：保存协议、地址、状态、缓冲区等
~~~

socket() 成功后主要完成两类事情。

### 4.1 分配 fd

fd 是当前进程文件描述符表中的一个位置。内核找到可用位置，将它标记为正在使用，然后返回对应整数。

~~~text
进程fd表
0 → 标准输入
1 → 标准输出
2 → 标准错误
3 → 新创建的socket
~~~

调用 close(fd) 后，该 fd 位置会被释放，之后可能被其他文件或 socket 再次使用。

### 4.2 创建并关联内核 socket 状态

应用程序用 fd 查找内核中的 socket 对象。TCP 相关对象会逐步保存：

~~~text
本地IP
本地端口
远端IP
远端端口
协议类型
TCP状态
发送缓冲区
接收缓冲区
序号和确认号
窗口及重传信息
~~~

课堂中把这组 TCP 连接状态概括为 TCB（Transmission Control Block，传输控制块）。

五元组是：

~~~text
源IP
源端口
目的IP
目的端口
传输层协议
~~~

例如：

~~~text
192.168.1.10:52000
  → 192.168.1.20:2000
  → TCP
~~~

同一个服务端端口可以同时拥有大量连接，因为每条连接的五元组不同。

---

## 五、bind()：把本地地址写入 socket

课堂代码：

~~~c
bind(fd,);
~~~

bind() 的作用是给 socket 指定本地 IP 和本地端口：

~~~text
通过fd找到内核socket对象
  ↓
设置本地IP
  ↓
设置本地端口
~~~

服务端必须有稳定的监听地址，所以通常显式绑定：

~~~text
0.0.0.0:2000
~~~

客户端通常直接调用 connect()；如果之前没有 bind()，内核会自动选择本地地址和临时端口。

---

## 六、listen()：让 socket 进入监听状态

课堂代码：

~~~c
listen(fd,backlog);
~~~

listen() 将 socket 变成监听 socket：

~~~text
普通TCP socket
  ↓ listen()
LISTEN状态
~~~

如果服务端只完成 bind() 而没有 listen()，对应端口没有处于 TCP 监听状态，客户端连接通常会被拒绝。

### 6.1 两类连接队列

监听 socket 的连接可以从两个阶段理解：

~~~text
收到客户端SYN
  ↓
SYN队列：握手尚未完成
  ↓
收到第三次握手ACK
  ↓
已完成连接队列：等待应用accept()
~~~

常见名称：

~~~text
SYN queue / 半连接队列
accept queue / 已完成连接队列
~~~

“半连接”不是只能传一半数据，而是三次握手还没有完成。

### 6.2 backlog

现代 Linux 中，listen(fd, backlog) 的 backlog 主要限制等待 accept() 的已完成连接队列长度：

~~~c
listen(fd,backlog);
~~~

可以理解成：

> 应用程序暂时没来得及 accept 时，内核允许多少条已经建立完成的连接排队等待。

SYN 队列还有独立内核配置：

~~~bash
cat /proc/sys/net/ipv4/tcp_max_syn_backlog
~~~

实际队列长度还受内核上限、系统配置和 SYN cookies 等机制影响。

---

## 七、TCP 三次握手

三次握手由内核 TCP 协议栈完成。应用程序通过 connect() 发起；服务端 listen() 后由内核接收握手；accept() 取出的是已经完成握手的连接。

假设双方初始序号分别是：

~~~text
客户端初始序号：1234
服务端初始序号：5647
~~~

### 7.1 第一次：客户端发送 SYN

~~~text
client → server

SYN = 1
SEQ = 1234
~~~

含义：

> 客户端希望建立连接，自己的初始发送序号从 1234 开始。

客户端状态：

~~~text
CLOSED → SYN_SENT
~~~

SYN 会占用一个序号，所以服务端确认时使用 1234 + 1。

### 7.2 第二次：服务端返回 SYN + ACK

~~~text
server → client

SYN = 1
ACK = 1
SEQ = 5647
ACK number = 1235
~~~

ACK number = 1235 表示客户端序号 1234 的 SYN 已经收到，下一次期望从 1235 开始。

同时，服务端告诉客户端自己的初始发送序号是 5647。

服务端相关连接状态：

~~~text
LISTEN → SYN_RECEIVED
~~~

### 7.3 第三次：客户端返回 ACK

~~~text
client → server

ACK = 1
ACK number = 5648
~~~

5648 = 5647 + 1，表示服务端的 SYN 已经收到。随后双方进入：

~~~text
ESTABLISHED
~~~

完整过程：

~~~text
客户端                              服务端

SYN, SEQ=1234
  ───────────────────────────────→

                         SYN+ACK,
               SEQ=5647, ACK=1235
  ←───────────────────────────────

ACK, ACK=5648
  ───────────────────────────────→

双方进入ESTABLISHED
~~~

### 7.4 为什么要交换序号

双方分别告诉对方自己的初始序号，后续 TCP 才能利用序号和确认号实现：

~~~text
识别数据顺序
发现数据缺失
排除重复数据
确认已经收到的字节范围
进行超时重传
~~~

初始序号不是固定从 0 开始，而是由内核生成。这样可以减少旧连接延迟报文的干扰，也提高序号的不可预测性。

---

## 八、connect()、握手和 accept() 的关系

客户端：

~~~c
connect();
~~~

connect() 让内核发起 TCP 三次握手。阻塞 socket 上，connect() 通常要等待连接成功或失败后才返回，因此服务端不可达时可能等待较长时间。

服务端：

~~~c
listen();
accept();
~~~

三次握手不是由 accept() 执行的。握手由服务端内核协议栈在监听状态下完成：

~~~text
内核完成三次握手
  ↓
连接进入已完成连接队列
  ↓
listenfd出现可读事件
  ↓
应用调用accept()
~~~

accept() 主要完成：

~~~text
从已完成连接队列取出一条连接
  ↓
为它分配新的clientfd
  ↓
让clientfd关联这条已连接socket
~~~

因此：

~~~text
listenfd → 继续监听新连接
clientfd → 与某个客户端收发数据
~~~

### 8.1 Reactor 中的对应关系

~~~text
listenfd出现EPOLLIN
  ↓
accept_cb()
  ↓
accept()
  ↓
得到clientfd
  ↓
把clientfd注册到epoll
~~~

LT 下，如果队列中还有连接，监听 fd 会继续报告可读。

ET 下应循环 accept()，直到当前队列已经取空。课堂代码：

~~~c
while（1）{
fd =accept();
if(fd==-1){
break

}
}
~~~

这段代码表达的核心是：

~~~text
一次EPOLLIN通知
  ↓
连续accept
  ↓
把当前已经完成的连接全部取出
~~~

实际使用非阻塞 fd 时，accept() 返回 -1 且 errno 为 EAGAIN/EWOULDBLOCK，表示本轮队列已经取空。

---

## 九、SYN 队列如何匹配第三次握手

收到第三次握手报文后，内核需要找到它属于哪一条正在建立的连接。

IP 头和 TCP 头能够提供：

~~~text
源IP
目的IP
源端口
目的端口
协议类型TCP
~~~

内核结合五元组、序号和连接状态查找对应的半连接状态，验证 ACK 后，将连接推进到已建立状态并放入已完成连接队列。

连接状态的生命周期不是从 accept() 才开始。服务端收到 SYN 后，内核已经需要保存握手状态；accept() 只是把完成后的连接交给应用程序。

---

## 十、SYN Flood

SYN Flood 的基本过程：

~~~text
攻击者发送大量SYN
  ↓
服务端为握手保存状态
  ↓
攻击者不完成第三次握手
  ↓
半连接状态和相关资源被持续占用
~~~

常见防护包括：

- SYN cookies。
- 调整 SYN 队列和重试参数。
- 防火墙、负载均衡器和云安全组。
- 限速和异常源识别。
- DDoS 清洗服务。

listen() 的 backlog 主要用于已完成连接队列，不能单独解决 SYN Flood。

---

## 十一、send() 和 recv() 到底做了什么

课堂中的数据路径：

~~~text
发送方application
  ↓ send/write
发送方kernel TCP协议栈
  ↓
网络
  ↓
接收方kernel TCP协议栈
  ↓ recv/read
接收方application
~~~

### 11.1 send()

课堂例子：

~~~c
send（fd,buffer,100,0）;
send（fd,buffer,200,0）;
send（fd,buffer,400,0）;
~~~

send() 返回正数，表示本次有多少字节被内核 socket 发送路径接受。它不表示对端应用已经读取这些数据。

后续什么时候组成 TCP 段、什么时候交给 IP 层、何时重传，由内核协议栈处理。

TCP 是字节流，不保留应用层每次 send() 的边界。上面三次调用总共交给 TCP 700 字节，接收端不一定按照 100、200、400 三次收到。

### 11.2 recv()

课堂例子：

~~~c
recv（fd，buffer，50,0）
recv（fd，buffer，250,0）
recv（fd，buffer，250,0）
recv（fd，buffer，150,0）
~~~

接收端可以按照 50 + 250 + 250 + 150 取出相同的 700 字节。

recv() 把已经到达 socket 接收缓冲区的数据复制到应用层 buffer。一次返回多少字节取决于：

~~~text
当前已经到达多少数据
应用提供的buffer有多大
socket是否阻塞
协议栈和调度时机
~~~

因此：

~~~text
发送端三次send
不等于
接收端必须三次recv
~~~

这也是 TCP 半包、粘包问题的来源。应用层协议需要自己定义消息边界，例如固定长度、长度字段或分隔符。

### 11.3 一次接收 1000 与十次接收 100

课堂对比：

~~~c
recv（fd,buffer,1000,0）;
~~~

与：

~~~c
recv（fd,buffer,100,0）;
// 循环10次
~~~

复制总数据量可以相同，但十次 recv() 会产生更多系统调用、循环判断和状态处理开销，因此性能并非完全没有差异。

实际代码应以协议完整性和程序结构为主，选择合理缓冲区大小，不必把每次读取切得非常小。

---

## 十二、MTU、MSS 与 TCP 分段

MTU 是链路能够承载的最大 IP 包大小。常见以太网 MTU 是 1500 字节，但它包含 IP 头，不等于 TCP 应用数据长度。

TCP 常用 MSS 表示一个 TCP 段最多携带的 payload 大小：

~~~text
MSS ≈ MTU - IP头 - TCP头
~~~

常见 IPv4、无额外选项时：

~~~text
1500 - 20 - 20 = 1460字节
~~~

课堂例子：

~~~c
send（fd,buffer,1700,0）;
~~~

应用一次交给内核 1700 字节，不代表网络上一定只有一个包。协议栈会根据 MSS、网卡卸载、路径 MTU 等条件分段。

接收端 recv() 的边界与网络分段也没有一一对应关系。网络上传输多个 TCP 段，接收端可能一次取出合并后的数据，也可能分多次取出。

---

## 十三、TCP 如何可靠并尽量快速地传输

这些机制主要运行在两端内核 TCP 协议栈之间，不由应用程序逐包控制。

### 13.1 序号与确认号

~~~text
SEQ → 当前报文携带的数据从哪个字节序号开始
ACK → 下一次期望收到哪个字节序号
~~~

如果 ACK number = 1235，可以理解为序号 1235 之前的数据已经连续收到，下一次希望从 1235 开始。

### 13.2 接收窗口与滑动窗口

接收方通过窗口告诉发送方：

> 当前接收缓冲区还允许接收多少数据。

发送方可以在没有逐段等待 ACK 的情况下发送窗口范围内的多段数据。ACK 推进后，允许发送的范围继续向前滑动。

它解决：

~~~text
既要连续发送提高吞吐
又不能超过接收方处理能力
~~~

### 13.3 慢启动

连接开始时，发送方不知道网络能承受多少在途数据，因此从较小的拥塞窗口开始。在慢启动阶段，拥塞窗口按每个 RTT 观察通常接近指数增长。

### 13.4 拥塞避免

拥塞窗口达到阈值后，增长转为更平缓的方式。检测到丢包或拥塞信号后，具体如何降低窗口取决于拥塞控制算法和丢包检测方式，不能把所有情况都理解为固定“减半重新开始”。

### 13.5 延迟确认

接收方可以短暂等待，把 ACK 与反向数据一起发送，或者用一个 ACK 确认更多数据，以减少纯 ACK 包数量。

延迟时间由协议实现和当前状态决定，不是所有系统都固定延迟 200 ms。

### 13.6 超时重传与快速重传

发送方在规定时间内没有收到确认，会认为数据可能丢失并触发重传。连续收到重复 ACK 时，也可能在超时前触发快速重传。

这些机制共同实现：

~~~text
可靠传输
按序交付
避免重复
适应接收端能力
适应网络拥塞程度
~~~

---

## 十四、close() 与四次挥手

课堂代码：

~~~c
close(fd);
~~~

close() 首先是通用的 fd 关闭接口。对 TCP socket 来说，当最后一个引用被关闭时，内核会根据 socket 状态和配置推进 TCP 关闭过程。

TCP 关闭使用 FIN 标志，不是 final。

### 14.1 为什么通常是四次

TCP 是全双工连接，两个方向需要分别关闭：

~~~text
主动关闭方                           被动关闭方

FIN
  ───────────────────────────────→

                                  ACK
  ←───────────────────────────────

                                  FIN
  ←───────────────────────────────

ACK
  ───────────────────────────────→
~~~

过程：

~~~text
1. 主动方发送FIN：我没有更多数据要发送
2. 被动方返回ACK：我知道你不再发送
3. 被动方处理完自己的发送后，也发送FIN
4. 主动方返回ACK：确认对方也不再发送
~~~

第二步和第三步有时会合并在同一个报文中，所以抓包时不一定总能机械地看到四个独立数据包。

### 14.2 主动关闭方状态

~~~text
ESTABLISHED
  ↓ close()，发送FIN
FIN_WAIT_1
  ↓ 收到对方对FIN的ACK
FIN_WAIT_2
  ↓ 收到对方FIN，返回ACK
TIME_WAIT
  ↓ 等待2MSL
CLOSED
~~~

### 14.3 被动关闭方状态

~~~text
ESTABLISHED
  ↓ 收到FIN，返回ACK
CLOSE_WAIT
  ↓ 本地应用调用close()，发送FIN
LAST_ACK
  ↓ 收到对方ACK
CLOSED
~~~

### 14.4 为什么 recv() 返回 0

被动方收到 FIN 后，如果此前还有已到达的数据，应用应先读出这些数据。全部读完以后，再次调用：

~~~c
recv(fd,buffer,size,0);
~~~

会返回：

~~~text
0
~~~

它表示对方正常关闭了发送方向，后面不会再有数据。非阻塞 socket 暂时没有数据返回的是 -1/EAGAIN，不是 0。

### 14.5 数据与 FIN

应用先调用：

~~~c
send（fd,buffer,100,0）
~~~

随后调用：

~~~c
close(fd);
~~~

TCP 会保持字节流顺序：内核已经接受的待发送数据排在 FIN 之前。接收端先读到数据，数据全部读完后才观察到 EOF，即 recv() == 0。

### 14.6 shutdown()

shutdown() 可以只关闭 TCP 的一个方向：

~~~text
SHUT_RD   → 关闭读方向
SHUT_WR   → 关闭写方向，向对端发送FIN
SHUT_RDWR → 关闭读写方向
~~~

它适合“我已经发送完，但还要继续接收对方响应”的半关闭协议。普通请求响应程序只使用 close() 也很常见，但 shutdown() 并不是没有用途。

---

## 十五、TIME_WAIT、CLOSING 与同时关闭

### 15.1 TIME_WAIT

通常由最终发送 ACK 的主动关闭方进入 TIME_WAIT。作用：

~~~text
如果最后一个ACK丢失，可以重新确认对方重发的FIN
让旧连接的延迟报文在网络中消失
~~~

服务端出现大量 TIME_WAIT，通常说明服务端经常主动关闭连接，不等于双方一定同时调用了 close()。

### 15.2 同时关闭

如果双方几乎同时调用 close()：

~~~text
双方都发送FIN
  ↓
双方都可能在FIN_WAIT_1时收到对方FIN
  ↓
进入CLOSING
  ↓
收到自己FIN对应的ACK
  ↓
进入TIME_WAIT
~~~

简化状态：

~~~text
ESTABLISHED
  ↓ 本地发送FIN
FIN_WAIT_1
  ↓ 先收到对端FIN
CLOSING
  ↓ 收到ACK
TIME_WAIT
~~~

如果主动方先收到 ACK，再收到 FIN，则通常经过：

~~~text
FIN_WAIT_1 → FIN_WAIT_2 → TIME_WAIT
~~~

TCP 状态机用于处理这些不同的报文到达顺序。

---

## 十六、同时打开与 TCP P2P

常规应用使用客户端/服务端角色：

~~~text
客户端调用connect()
服务端调用listen()/accept()
~~~

但 TCP 建立后，双方都是全双工通信端点。客户端和服务端主要描述谁主动连接、谁被动监听，以及应用协议中的角色。

TCP 还定义了“同时打开”：双方都主动发送 SYN，并在 SYN_SENT 状态收到对方 SYN。

~~~text
双方CLOSED
  ↓ 同时主动打开
双方SYN_SENT
  ↓ 收到对方SYN并返回SYN+ACK
双方SYN_RECEIVED
  ↓ 收到ACK
双方ESTABLISHED
~~~

课堂中的 P2P 方向：

~~~c
fd=socket（）
bind（）//optional
connect（）；
~~~

真正跨公网实现 P2P 还需要考虑 NAT、端口映射、防火墙和连接协调。单纯同时调用 connect() 不保证一定能穿透网络设备。

本节需要理解：

> TCP 建立后没有“只能客户端发、只能服务端收”的限制，双方都可以 send 和 recv。

---

## 十七、把 API 与 TCP 状态对应起来

| 应用调用或事件 | 内核中的主要含义 |
|---|---|
| socket() | 创建 fd 和 socket 相关内核对象 |
| bind() | 设置本地 IP 和端口 |
| listen() | 进入 LISTEN，准备接收连接 |
| connect() | 主动发起三次握手 |
| 收到 SYN | 创建或查找握手状态，进入 SYN_RECEIVED |
| 握手完成 | 连接进入 ESTABLISHED，并等待 accept |
| accept() | 从已完成连接队列取出连接，返回 clientfd |
| send() | 把应用字节交给 socket 发送路径 |
| recv() | 把 socket 接收数据复制到应用缓冲区 |
| 收到 FIN | 对端关闭发送方向，最终让 recv 返回 0 |
| close() | 释放 fd 引用，并推动 TCP 关闭过程 |

完整主线：

~~~text
socket()
  ↓
bind()
  ↓
listen() / connect()
  ↓
三次握手
  ↓
accept()得到clientfd
  ↓
send()/recv()
  ↓
序号、确认、窗口、拥塞控制、重传
  ↓
close()
  ↓
FIN/ACK与四次挥手
  ↓
TIME_WAIT或CLOSED
~~~

---

## 十八、本节课需要真正掌握的内容

### 重点一：API 是入口，协议栈完成真正的 TCP 工作

~~~text
connect()不是应用自己发送三个包
accept()不负责执行三次握手
send()不等于对端应用已经收到
recv()不对应某一次send()
close()会推动TCP关闭状态机
~~~

### 重点二：监听 fd 和客户端 fd 对应不同状态

~~~text
listenfd
  → LISTEN状态
  → 管理握手和等待accept的连接

clientfd
  → 对应某条已建立TCP连接
  → 用于send和recv
~~~

### 重点三：TCP 是可靠字节流

~~~text
没有应用消息边界
使用序号和ACK保证顺序与可靠性
使用接收窗口进行流量控制
使用拥塞窗口适应网络承载能力
使用重传恢复丢失数据
~~~

### 重点四：建立和关闭都是状态机

~~~text
三次握手：
CLOSED → SYN_SENT/SYN_RECEIVED → ESTABLISHED

主动关闭：
ESTABLISHED → FIN_WAIT_1 → FIN_WAIT_2 → TIME_WAIT

被动关闭：
ESTABLISHED → CLOSE_WAIT → LAST_ACK → CLOSED
~~~

### 课后实践

准备两台虚拟机，观察 TCP P2P：

~~~text
1. 双方创建TCP socket
2. 根据实验设计绑定本地地址
3. 协调双方连接时机
4. 抓包观察SYN、SYN+ACK、ACK
5. 建立后让双方都执行send和recv
6. 同时或分别close，观察TCP状态
~~~

可以配合：

~~~bash
ss -ant
tcpdump -nn -i any tcp
~~~

