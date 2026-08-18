---
title: DPDK 用户态 TCP 协议栈（四）：并发与自实现 epoll
slug: dpdk-userspace-tcp-concurrency-epoll
description: 从连接级 stream 与用户态 fd 出发，梳理自实现 epoll 的红黑树、就绪链表、事件回调、线程唤醒，以及 TCP 握手、数据和 FIN 的完整通知链路
date: 2026-08-18T00:00:00+08:00
draft: false
image: cover.svg
tags:
  - DPDK
  - 用户态协议栈
  - TCP
  - epoll
  - 网络编程
categories:
  - 后端开发
---

> 本篇只围绕一个问题展开：**用户态 TCP 协议栈怎样让一个应用线程并发管理多条 TCP 连接？**
>
> 前三篇已经反复讲过 DPDK 收发、mbuf、Ethernet/IPv4/TCP 解析、三次握手、SEQ/ACK、TCP 状态机和 POSIX Socket。本篇只在它们影响 epoll 逻辑时简要回顾，不再重新逐层讲包头和握手细节。

---

## 一、先给出整篇的核心答案

TCP 并发 epoll 的实现可以浓缩成一个闭环：

```text
1. 每条 TCP 连接拥有独立的 stream/TCB 和用户态 fd
2. 应用通过 epoll_ctl 注册“我关心哪些 fd 的哪些事件”
3. 协议栈处理报文时发现某个 fd 已经可读或可写
4. 协议栈调用 epoll 回调，把该 fd 放入就绪队列
5. 回调唤醒阻塞在 epoll_wait 的应用线程
6. epoll_wait 返回就绪 fd，应用执行 accept/recv/send/close
7. 应用重新进入 epoll_wait，继续管理所有连接
```

epoll 没有让 TCP 报文并行到达，也没有替协议栈处理握手。它解决的是：

> 当许多连接都可能产生数据时，应用怎样只处理当前已经就绪的连接，而不是为每条连接创建一个长期阻塞的线程。

理解本篇时始终抓住三种对象：

| 对象 | 解决的问题 |
|---|---|
| `ng_tcp_stream` | 这是谁的连接，当前 TCP 状态和数据是什么 |
| `epitem` | 应用是否关注这个 fd，它现在是否就绪 |
| `eventpoll` | 一个 epoll 实例管理的整集、就绪集和等待线程 |

---

## 二、只保留必要的旧知识：数据最终从哪里来到哪里去

完整程序有三个长期运行的执行流：

```text
main/lcore：网卡 RX/TX
    ↕ 全局 in/out ring
pkt_process/lcore：Ethernet/IP/TCP 解析、TCP 输出封包
    ↕ 每连接 rcvbuf/sndbuf
tcp_server_entry/lcore：accept、recv、send、epoll_wait
```

两层 ring 不要混淆：

- 全局 `ring->in/out` 传递 `rte_mbuf *`，用于网卡线程与协议处理线程之间传包。
- 每个 `ng_tcp_stream` 的 `rcvbuf/sndbuf` 传递 TCP fragment，用于协议栈与应用之间传数据。

一段客户端数据的最短路径是：

```text
网卡 RX
→ 全局 in ring
→ ng_tcp_process()
→ 按四元组找到 stream
→ payload 放入 stream->rcvbuf
→ 触发 EPOLLIN
→ nepoll_wait() 返回 connfd
→ 应用 nrecv(connfd)
```

应用响应的路径相反：

```text
nsend(connfd)
→ fragment 放入 stream->sndbuf
→ ng_tcp_out() 取出并封装 TCP 报文
→ 全局 out ring
→ 网卡 TX
```

这些收发与报文构造细节在前几篇已有完整说明。本篇真正关心的是中间这一步：

```text
stream 的状态发生变化
→ 怎样让正在等待的应用知道？
```

---

## 三、支持并发的第一前提：连接必须彼此独立

### 1. 一条连接对应一个 `ng_tcp_stream`

```c
struct ng_tcp_stream {
    int fd;

    uint32_t sip, dip;
    uint16_t sport, dport;
    uint8_t protocol;

    uint32_t snd_nxt;
    uint32_t rcv_nxt;
    NG_TCP_STATUS status;

    struct rte_ring *sndbuf;
    struct rte_ring *rcvbuf;

    pthread_cond_t cond;
    pthread_mutex_t mutex;
};
```

它是本代码中的简化 TCB。每个客户端都必须拥有独立的：

- 地址与端口；
- TCP 状态；
- SEQ/ACK；
- 接收和发送缓冲区；
- 用户态 fd；
- 阻塞与唤醒状态。

因此，TCP 并发的根基不是 epoll，而是**连接级状态隔离**。如果两条连接共用序列号或接收缓冲区，换成 epoll 也不能让协议栈正确并发。

### 2. 报文怎样找到自己的 stream

```c
if (iter->sip == sip && iter->dip == dip &&
    iter->sport == sport && iter->dport == dport) {
    return iter;
}
```

一般网络流用五元组标识；这里已经确定是 TCP，所以搜索时比较四元组。两个客户端连接同一个服务端地址和端口时，客户端 IP 或源端口不同，因此能找到不同的 stream。

### 3. 监听 stream 和连接 stream

服务端只有一个监听 stream，但可以有许多已连接 stream：

```text
listenfd / LISTEN stream
    ├── connfd A / ESTABLISHED stream A
    ├── connfd B / ESTABLISHED stream B
    └── connfd C / ESTABLISHED stream C
```

收到 SYN 时，协议栈保留监听 stream，并创建一个新的连接 stream 处理该客户端：

```c
struct ng_tcp_stream *syn = ng_tcp_stream_create(
    iphdr->src_addr, iphdr->dst_addr,
    tcphdr->src_port, tcphdr->dst_port);

LL_ADD(syn, table->tcb_set);
syn->status = NG_TCP_STATUS_SYN_RCVD;
```

监听对象负责“还能不能接新连接”，连接对象负责“这个客户端的数据和状态”。epoll 后面会分别监听这两类 fd。

---

## 四、一连接一线程为什么能工作，又为什么需要 epoll

### 1. 一连接一线程

最直观的并发方式是：

```text
主线程：不断 naccept()
连接 A：线程 A 阻塞在 nrecv(A)
连接 B：线程 B 阻塞在 nrecv(B)
连接 C：线程 C 阻塞在 nrecv(C)
```

它适合验证连接表、stream 隔离和收发缓冲区是否支持多个客户端。问题是大部分连接经常没有数据，线程却仍占用栈空间和调度资源。连接数增长时，线程数也同步增长。

### 2. epoll 的改变

epoll 模型下，应用线程不阻塞在某一个 `nrecv(connfd)` 上，而是阻塞在“所有已注册 fd 的就绪集合”上：

```text
应用线程：nepoll_wait(A, B, C, ...)

只有 B 有数据
→ nepoll_wait 只返回 B
→ 应用处理 B
→ 再次等待所有连接
```

所以两种模型的区别不是 TCP 连接数，而是等待位置：

```text
一连接一线程：每个线程等待一个 fd
epoll：一个线程等待一组 fd
```

---

## 五、为什么不能直接调用 Linux 原生 epoll

代码中的 `listenfd`、`connfd` 和 `epfd` 由自己的位图分配：

```c
static int get_fd_frombitmap(void) {
    for (int fd = DEFAULT_FD_NUM; fd < MAX_FD_COUNT; fd++) {
        if (fd_table 中该位空闲) {
            标记为已用;
            return fd;
        }
    }
    return -1;
}
```

这些整数只在用户态协议栈内部有意义：

```text
用户态 fd
→ get_hostinfo_fromfd(fd)
→ ng_tcp_stream / localhost / eventpoll
```

Linux 内核没有创建对应的 socket 对象，也不知道 `stream->rcvbuf` 是否有数据。因此把这种 `connfd` 传给内核 `epoll_ctl()`，内核无法监听它。

既然 Socket API 是用户态自己实现的，事件机制也要自己补齐：

```text
nsocket / nbind / nlisten / naccept / nrecv / nsend
                            +
nepoll_create / nepoll_ctl / nepoll_wait
```

---

## 六、理解 epoll 的关键：就绪是一种状态

应用关注的不是“是否来过一个包”，而是“现在执行某个操作会不会阻塞”。

### 1. 监听 fd 可读

监听 fd 的 `EPOLLIN` 表示：

```text
至少有一条已完成握手的连接可以被 naccept() 取出
```

它不是说监听 socket 收到了应用 payload，而是说 `accept` 现在不会阻塞。

### 2. 连接 fd 可读

连接 fd 的 `EPOLLIN` 表示：

```text
nrecv() 现在可以取得 payload，或者取得 EOF/关闭信息
```

因此，收到 FIN 也可以触发 `EPOLLIN`。应用随后调用 `nrecv()` 得到 0，知道对端已关闭。

### 3. 连接 fd 可写

`EPOLLOUT` 的一般含义是：发送缓冲区有空间，`send` 不会因为缓冲区已满而阻塞。它不等于“一个数据包已经从网卡发完”。

本节代码重点实现的是 `EPOLLIN` 链路；要完整支持 `EPOLLOUT`，还需要定义 `sndbuf` 从满变为可用时怎样产生通知。

这一点非常重要：

> epoll 事件是 Socket 操作条件的变化，不是对每个网络包做一次简单转发。

---

## 七、epoll 的两个集合

一个 epoll 实例内部有两份不同用途的集合。

### 1. 整集：我关注谁

整集保存所有通过 `EPOLL_CTL_ADD` 注册的 fd。代码用红黑树组织，以 `sockfd` 为 key：

```text
整集 = {listenfd, connfd_A, connfd_B, connfd_C, ...}
```

它回答：

- 这个 fd 是否被注册？
- 应用对它关心 `EPOLLIN` 还是 `EPOLLOUT`？
- `ADD/DEL/MOD` 应该操作哪个节点？

红黑树提供稳定的 `O(log n)` 查找、插入和删除，并能随注册数量按节点增长。哈希的平均查找更快，但要额外处理桶容量、冲突和扩容。这里选择红黑树的重点是动态集合与稳定性能，并不是红黑树在所有场景都绝对更快。

### 2. 就绪集：现在谁能处理

就绪集是链表，只保存当前已经产生事件的节点：

```text
整集：{listenfd, A, B, C, D}
就绪集：{B, D}
```

`nepoll_wait()` 无须扫描整棵红黑树，只需从就绪链表取出 B、D。

### 3. 同一个节点同时属于两份集合

```c
struct epitem {
    RB_ENTRY(epitem) rbn;
    LIST_ENTRY(epitem) rdlink;

    int rdy;
    int sockfd;
    struct epoll_event event;
};
```

一个 `epitem` 中同时有红黑树节点和链表节点：

```text
rbn：长期挂在整集，表示注册关系
rdlink：就绪时临时挂在就绪链表
rdy：防止同一节点被重复插入就绪链表
```

节点进入就绪链表后仍保留在红黑树里。应用消费一次就绪事件，不等于取消对这个 fd 的关注；只有 `EPOLL_CTL_DEL` 才删除注册关系。

### 4. `eventpoll` 管理整个 epoll 实例

```c
struct eventpoll {
    int fd;

    ep_rb_tree rbr;
    int rbcnt;

    LIST_HEAD(, epitem) rdlist;
    int rdnum;

    pthread_mutex_t mtx;
    pthread_spinlock_t lock;
    pthread_cond_t cond;
    pthread_mutex_t cdmtx;
};
```

关系可以画成：

```text
epfd
└── eventpoll
    ├── rbr：全部已注册 epitem
    ├── rdlist：当前就绪 epitem
    └── cond：没有就绪事件时，让 nepoll_wait 睡眠
```

---

## 八、三个外部接口怎样组成应用侧逻辑

### 1. `nepoll_create()`：创建事件管理器

它做的事情不是创建 TCP 连接，而是创建一个 `eventpoll`：

```text
分配 epfd
→ 分配 eventpoll
→ 初始化红黑树
→ 初始化就绪链表
→ 初始化锁与条件变量
→ 建立 epfd 到 eventpoll 的映射
```

应用以后通过 `epfd` 找到这一整套事件管理状态。

### 2. `nepoll_ctl()`：维护整集

```text
ADD：树中不存在 → 分配 epitem → 插入红黑树
DEL：树中存在 → 从红黑树移除 → 释放 epitem
MOD：树中存在 → 修改关注的事件
```

因此，`nepoll_ctl()` 只是在表达应用的兴趣：

> “如果这个 fd 以后出现我关心的状态，请通知我。”

注册本身不会凭空制造事件。

### 3. `nepoll_wait()`：没有事件就睡，有事件就返回

```c
while (ep->rdnum == 0 && timeout != 0) {
    if (timeout > 0)
        pthread_cond_timedwait(&ep->cond, &ep->cdmtx, ...);
    else if (timeout < 0)
        pthread_cond_wait(&ep->cond, &ep->cdmtx);
}
```

超时语义：

- `timeout < 0`：一直等待；
- `timeout == 0`：立即检查并返回；
- `timeout > 0`：最多等待指定时长。

醒来后，它从就绪链表取出最多 `maxevents` 个节点，把其中的 `epoll_event` 复制给应用：

```text
rdlist 中的 epitem
→ events[0...n-1]
→ 返回 nready
```

至此，epoll 只告诉应用“哪些 fd 可以处理”。真正的 `accept/recv/send` 仍由应用自己调用。

---

## 九、最关键的内部接口：协议栈怎样触发事件

三个 `nepoll_*` 接口面向应用；`epoll_event_callback()` 面向协议栈：

```c
int epoll_event_callback(struct eventpoll *ep,
                         int sockid, uint32_t event) {
    struct epitem *epi = RB_FIND(..., sockid);
    if (!epi)
        return -1;

    if (epi->rdy) {
        epi->event.events |= event;
        return 1;
    }

    pthread_spin_lock(&ep->lock);
    epi->rdy = 1;
    LIST_INSERT_HEAD(&ep->rdlist, epi, rdlink);
    ep->rdnum++;
    pthread_spin_unlock(&ep->lock);

    pthread_mutex_lock(&ep->cdmtx);
    pthread_cond_signal(&ep->cond);
    pthread_mutex_unlock(&ep->cdmtx);
}
```

它完成四步：

```text
按 sockid 查整集
→ 把事件记录到对应 epitem
→ 把 epitem 加入就绪链表
→ signal 唤醒 nepoll_wait
```

如果节点已经在就绪链表中，就不重复插入，只合并事件位：

```c
epi->event.events |= event;
```

这避免同一个 fd 连续收到多个通知时，在 `rdlist` 中出现多个相同节点。

### epoll 模块在整体架构中的位置

```text
协议栈                         epoll                         应用

握手完成 ─┐
收到数据 ─┼→ event_callback → rdlist → cond_signal → nepoll_wait
收到 FIN ─┘                                           ↓
                                               accept/recv/close
```

所以 epoll 不是协议栈之外的独立轮询器。它必须由协议栈在状态真正改变的地方主动通知。

---

## 十、用三条时间线彻底串起实现

### 时间线一：新客户端完成连接

```text
① 客户端发送 SYN
② ng_tcp_process 找到 LISTEN stream
③ 为客户端创建新的 stream，进入 SYN_RCVD
④ 协议栈发送 SYN+ACK
⑤ 收到客户端 ACK，新 stream 进入 ESTABLISHED
⑥ 协议栈触发 listenfd 的 EPOLLIN
⑦ nepoll_wait 返回 listenfd
⑧ 应用调用 naccept(listenfd)
⑨ naccept 取出已建立 stream，为它分配 connfd
⑩ 应用用 EPOLL_CTL_ADD 把 connfd 加入 epoll
```

对应代码的关键通知：

```c
stream->status = NG_TCP_STATUS_ESTABLISHED;

struct ng_tcp_stream *listener =
    ng_tcp_stream_search(0, 0, 0, stream->dport);

epoll_event_callback(table->ep, listener->fd, EPOLLIN);
```

为什么通知 `listener->fd`，而不是新连接的 fd？

因为此时应用还没有执行 `naccept()`，新 stream 还没有分配给应用使用的 `connfd`。对应用来说，当前可执行的动作是 `accept(listenfd)`，所以就绪的是监听 fd。

### 时间线二：已连接客户端发送 payload

```text
① 报文按四元组找到 established stream
② 协议栈复制 payload，构造 fragment
③ fragment 进入 stream->rcvbuf
④ 协议栈触发 stream->fd 的 EPOLLIN
⑤ callback 找到 connfd 对应的 epitem
⑥ epitem 进入 rdlist，唤醒 nepoll_wait
⑦ 应用得到 connfd
⑧ 应用调用 nrecv(connfd)，从该 stream 的 rcvbuf 取数据
```

关键顺序必须是：

```text
先让数据真正可读
→ 再发布 EPOLLIN
```

否则应用被唤醒后可能发现 `rcvbuf` 仍为空。

### 时间线三：客户端发送 FIN

```text
① 协议栈收到 FIN
② stream 进入 CLOSE_WAIT
③ rcvbuf 中放入长度为 0 的 fragment，表示 EOF
④ 触发 connfd 的 EPOLLIN
⑤ nepoll_wait 返回 connfd
⑥ nrecv(connfd) 返回 0
⑦ 应用 EPOLL_CTL_DEL，然后 nclose(connfd)
```

这条时间线解释了为什么“关闭”也能通过可读事件交给应用：读取操作已经不会阻塞，只是结果为 EOF。

---

## 十一、epoll 服务器循环现在应该怎样读

```c
int epfd = nepoll_create(1);

ev.events = EPOLLIN;
ev.data.fd = listenfd;
nepoll_ctl(epfd, EPOLL_CTL_ADD, listenfd, &ev);

while (1) {
    int nready = nepoll_wait(epfd, events, 128, 5);

    for (int i = 0; i < nready; i++) {
        if (events[i].data.fd == listenfd) {
            int connfd = naccept(listenfd, ...);

            ev.events = EPOLLIN;
            ev.data.fd = connfd;
            nepoll_ctl(epfd, EPOLL_CTL_ADD, connfd, &ev);
        } else {
            int connfd = events[i].data.fd;
            int n = nrecv(connfd, buff, BUFFER_SIZE, 0);

            if (n > 0) {
                nsend(connfd, buff, n, 0);
            } else {
                nepoll_ctl(epfd, EPOLL_CTL_DEL, connfd, NULL);
                nclose(connfd);
            }
        }
    }
}
```

不要只把它看成一个 API 调用顺序。这个循环其实在处理两类状态机入口：

```text
listenfd 就绪
→ 连接管理路径
→ accept 新连接并将 connfd 注册进 epoll

connfd 就绪
→ 数据路径或关闭路径
→ recv 数据，或者识别 EOF 后移除连接
```

多个客户端的并发体现在：所有 `connfd` 都在同一棵红黑树中，但某一时刻只有真正就绪的连接进入 `rdlist`。

---

## 十二、薄弱但非常重要：LT、ET 与“消费事件”

理解整集和就绪集后，还必须区分两种事件语义。

### 1. LT：Level Triggered，水平触发

只要可读条件仍然成立，就应该继续报告：

```text
rcvbuf 里还有数据
→ fd 仍然可读
→ 下一次 epoll_wait 仍应返回它
```

应用可以一次只读一部分，剩余数据会让 fd 继续保持就绪。

### 2. ET：Edge Triggered，边沿触发

只在状态从“不可读”变为“可读”的边沿通知一次：

```text
rcvbuf：空 → 非空
→ 触发一次 EPOLLIN
```

如果应用没有把数据一直读到 `EAGAIN`，而 `rcvbuf` 始终保持非空，就可能没有新的空→非空边沿，因此也不会再次收到通知。

### 3. 当前学习代码的核心行为

`epoll_event_callback()` 把节点加入 `rdlist`；`nepoll_wait()` 返回后将其移出并清除 `rdy`。以后再次发生协议栈回调时，节点才能重新加入。

这更接近“协议栈事件通知队列”的实现骨架。要形成严格的 LT 或 ET，还需要把**就绪谓词**定义清楚：

```text
监听 fd 可读：accept 队列非空
连接 fd 可读：rcvbuf 非空，或存在 EOF/错误
连接 fd 可写：sndbuf 有可用空间
```

然后选择策略：

- LT：`epoll_wait` 返回事件后，如果谓词仍为真，应保留或重新加入就绪集。
- ET：只有谓词从假变真时才入就绪集，应用必须把资源处理到再次变为假。

这部分是自实现 epoll 最容易遗漏的核心：**就绪链表只是结果，真正决定事件语义的是状态谓词和重新入队规则。**

---

## 十三、线程同步：为什么需要三种同步手段

代码中不同锁保护不同对象：

| 同步对象 | 主要保护内容 |
|---|---|
| `ep->mtx` | 红黑树的 ADD/DEL/MOD |
| `ep->lock` 自旋锁 | 短时间修改 `rdlist`、`rdnum`、`rdy` |
| `ep->cdmtx + cond` | `nepoll_wait` 睡眠与协议栈唤醒 |

### 1. 为什么等待必须检查条件

正确思路不是“收到 signal 就相信一定有事件”，而是：

```c
while (ep->rdnum == 0)
    pthread_cond_wait(...);
```

条件变量可能虚假唤醒，也可能多个等待者竞争同一批事件。醒来后必须重新检查 `rdnum`。

同理，`nrecv()` 等待数据时也使用循环检查 `rcvbuf`，而不是单次 `if`。

### 2. 为什么先入就绪集，再 signal

发布顺序是：

```text
修改受保护状态：epitem 进入 rdlist，rdnum++
→ 再 signal
```

signal 的作用只是提醒等待线程重新检查条件；真正可信的是受锁保护的 `rdlist/rdnum`。

### 3. `DEL` 与回调之间的关系

应用关闭连接时会删除 `epitem`，协议栈线程则可能正在为同一 fd 触发回调。完整实现必须让红黑树查找、节点删除和就绪链表访问遵守一致的生命周期规则，避免回调继续访问已经释放的节点。

这里不展开代码审查，只需记住：并发数据结构除了“查得快”，还要保证节点从注册、就绪到删除期间始终有效。

---

## 十四、单 epoll 与多个 epoll

当前代码在创建时执行：

```c
struct ng_tcp_table *table = tcpInstance();
table->ep = ep;
```

协议栈触发事件时也直接使用：

```c
epoll_event_callback(table->ep, stream->fd, EPOLLIN);
```

这意味着当前教学实现默认整个 TCP table 只有一个 epoll。它足以展示完整通知链路，但还不能表达：

- 同一进程创建多个 epoll 实例；
- 不同 epoll 注册不同连接；
- 同一个 fd 同时被多个 epoll 关注。

扩展为多个 epoll 时，不能只保留一个 `table->ep`。需要建立：

```text
epfd → eventpoll

以及

sockfd → 所有注册了该 sockfd 的 epitem/eventpoll
```

协议栈产生事件后，要找到所有相关订阅者并分别更新它们的就绪集。这实际上把单指针通知扩展成一对多的观察者关系。

---

## 十五、多连接 core dump 在本篇中的位置

多连接时暴露 core dump，说明“单连接路径成立”并不等于“所有连接级资源都已正确隔离”。本节笔记直接涉及的一点是：每条 stream 都会创建自己的 DPDK ring，而 DPDK 命名对象需要可区分的名称。

```c
sprintf(sbufname, "sndbuf%x%d", sip, sport);
stream->sndbuf = rte_ring_create(sbufname, ...);

sprintf(rbufname, "bufname%x%d", sip, sport);
stream->rcvbuf = rte_ring_create(rbufname, ...);
```

这里用客户端 IP 和源端口区分连接 ring。要记住三层不同的身份：

```text
四/五元组：让协议栈把报文分给正确 stream
DPDK ring 名称：让每条 stream 获得独立缓冲对象
用户态 fd：让应用和 epoll 找到正确 stream
```

本篇不继续展开具体崩溃排查，因为重点是 epoll 的事件闭环。

---

## 十六、把全部实现压缩成一张图

```text
                         应用线程
                            │
             nepoll_ctl ADD │ 注册兴趣
                            ▼
                    ┌───────────────┐
                    │   eventpoll   │
                    │               │
                    │ 红黑树 rbr    │ ← 全部注册 fd
                    │ 就绪表 rdlist │ ← 当前就绪 fd
                    │ cond          │ ← 没事件时睡眠
                    └───────┬───────┘
                            │ nepoll_wait 返回
                            ▼
               accept / recv / send / close
                            ▲
                            │ 用户态 fd → stream
                    ┌───────┴────────┐
                    │ ng_tcp_stream  │
                    │ 状态/SEQ/ACK   │
                    │ rcvbuf/sndbuf  │
                    └───────▲────────┘
                            │
              ng_tcp_process 按四元组定位
                            │
        握手完成 / payload 到达 / FIN 到达
                            │
              epoll_event_callback
                            │
                加入 rdlist + signal
```

从这张图可以看到，TCP 并发 epoll 不是单独一个数据结构，而是四套映射协作：

```text
报文四元组 → stream
用户态 fd → stream
epfd → eventpoll
eventpoll 中 sockfd → epitem
```

缺少任何一层，事件都无法从报文准确传递到应用。

---

## 十七、最终应掌握的实现逻辑

1. 多连接首先要求每条连接拥有独立 TCB、状态、序列号和收发缓冲区。
2. 报文通过四元组找到 stream，应用通过用户态 fd 找到同一个 stream。
3. 一连接一线程让每个线程阻塞等一个 fd；epoll 让一个线程等待一组 fd。
4. 用户态 fd 不属于内核，因此必须自实现 epoll，不能直接交给 Linux epoll。
5. 红黑树保存“关注谁”，就绪链表保存“现在谁能处理”。
6. `nepoll_ctl()` 建立兴趣关系，`epoll_event_callback()` 发布事件，`nepoll_wait()` 消费事件。
7. 握手完成触发监听 fd 的 `EPOLLIN`，因为此时 `accept` 不再阻塞。
8. payload 或 FIN 到达触发连接 fd 的 `EPOLLIN`，因为此时 `recv` 能返回数据或 EOF。
9. 必须先改变真实就绪状态，再把事件加入就绪集并唤醒等待线程。
10. LT/ET 的本质不在链表名字，而在就绪谓词以及事件消费后的重新入队规则。
11. 单 epoll 可以用 `tcp_table->ep`；多个 epoll 需要 fd 到多个订阅者的映射。
12. epoll 解决应用调度问题，不替代 TCP 状态机，也不能弥补连接状态未隔离的问题。

## 十八、反刍题

1. 为什么三次握手完成时触发 `listenfd`，而不是新连接 fd？
2. 为什么 FIN 可以通过 `EPOLLIN` 通知？
3. 红黑树和就绪链表各自回答什么问题？
4. 为什么一个 `epitem` 要同时包含 `RB_ENTRY` 和 `LIST_ENTRY`？
5. `nepoll_ctl()` 注册了一个 fd 后，为什么它不一定立刻出现在就绪链表？
6. 从 payload 到达到 `nepoll_wait()` 返回，中间依次发生了什么？
7. 如果 `rcvbuf` 仍有未读数据，LT 和 ET 下一步分别应该怎样表现？
8. 为什么条件变量的等待必须放在 `while` 中？
9. 为什么发布事件时要先更新 `rdlist/rdnum`，再调用 `signal`？
10. 当前的 `tcp_table->ep` 为什么只能自然表达单 epoll？
11. 如果一个 connfd 被两个 epoll 实例注册，协议栈该怎样分发一次 `EPOLLIN`？
12. epoll 已经实现正确，但不同连接仍共用 `rcvbuf`，并发为什么仍然会失败？
