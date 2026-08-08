---
title: Reactor 与 epoll：百万并发网络服务器
slug: reactor-epoll-million-connections
description: 从课堂 Echo Server 代码梳理 Reactor 事件流转、epoll 使用方式，以及百万连接测试中的 fd、端口、内存与内核限制
date: 2026-08-08T00:00:00+08:00
draft: false
image: cover.svg
tags:
  - 网络编程
  - Linux
  - epoll
  - Reactor
  - 高并发
categories:
  - 后端开发
---

## 1. 本节课要解决什么问题

本节课不是直接实现完整 WebServer，而是先搭建一个能够承载大量网络连接的底层 I/O 框架。在这个框架之上，后续才能继续实现 HTTP 解析、路由、业务处理等功能。

核心目标有两个：

1. 使用 Linux `epoll` 管理大量 socket。
2. 使用 Reactor 思想，把不同 I/O 事件分发给不同回调函数。

当前代码实现的是一个简单的多端口 Echo Server：客户端发来什么，服务端就回写什么。

---

## 2. Reactor 的核心思想

Reactor 可以概括为：

```text
I/O 对象（fd） → 发生事件（event） → 执行回调（callback）
```

本例中的映射关系如下：

| I/O 对象 | 事件 | 含义 | 回调函数 |
|---|---|---|---|
| `listenfd` | `EPOLLIN` | 有新连接到达 | `accept_cb()` |
| `clientfd` | `EPOLLIN` | 客户端数据可读 | `recv_cb()` |
| `clientfd` | `EPOLLOUT` | socket 当前可写 | `send_cb()` |

因此，主循环不需要判断“这个 fd 到底要做什么业务”，而只需要：

1. 调用 `epoll_wait()` 等待事件。
2. 从事件中取得 fd。
3. 根据事件类型，调用该 fd 已经注册好的回调。

这就是“事件驱动”：没有事件时线程阻塞等待，有事件时才处理对应连接。

### 2.1 Reactor 的两个重点

1. 建立 `event` 与 `callback` 的对应关系。
2. 为每一个 I/O 保存独立的连接状态。

第二点非常重要。多个连接的数据不能共用一组读写缓冲区，否则连接之间会互相覆盖。因此每个 fd 都需要自己的：

- 读缓冲区 `rbuffer`
- 当前读取长度 `rlength`
- 写缓冲区 `wbuffer`
- 待发送长度 `wlength`
- 读事件回调
- 写事件回调

---

## 3. `epoll` 在 Reactor 中的职责

`epoll` 是 Linux 提供的 I/O 多路复用机制。一个线程可以通过一个 `epoll` 实例监听大量 fd，而不必给每条 TCP 连接创建一个线程。

三个最重要的接口是：

```c
int epoll_create1(int flags);
int epoll_ctl(int epfd, int op, int fd, struct epoll_event *event);
int epoll_wait(int epfd, struct epoll_event *events,
               int maxevents, int timeout);
```

其作用分别为：

- `epoll_create1()`：创建一个 epoll 实例。
- `epoll_ctl()`：添加、修改或删除所关注的 fd 和事件。
- `epoll_wait()`：等待已经注册的 fd 产生事件。

当前代码使用的是默认的 LT（Level Triggered，水平触发）模式，因为没有设置 `EPOLLET`。

LT 模式下，只要 fd 仍处于可读或可写状态，`epoll_wait()` 就会继续报告它。若以后改成 ET（边沿触发），则 socket 必须设置成非阻塞，并且 `accept()`、`recv()` 等操作通常要循环执行，直到返回 `EAGAIN`。

---

## 4. 代码中的连接对象

```c
struct conn {
    int fd;

    char rbuffer[BUFFER_LENGTH];
    int rlength;

    char wbuffer[BUFFER_LENGTH];
    int wlength;

    RCALLBACK send_callback;
    union {
        RCALLBACK recv_callback;
        RCALLBACK accept_callback;
    } r_action;
};
```

### 4.1 为什么需要 `struct conn`

TCP 是字节流协议，一次 `recv()` 不保证收到一个完整业务消息：

- 一个完整消息可能被拆成多次接收。
- 多个消息也可能在一次 `recv()` 中一起到达。
- 一次 `send()` 也不保证发送全部待发送数据。

所以 Reactor 不仅要保存回调，还必须保存每个连接的处理进度。严格来说，后续还应增加：

- 已读取但尚未解析的数据偏移量；
- 待发送数据的起始偏移量；
- 协议解析状态；
- 连接状态和超时信息；
- 动态扩容或链式缓冲区。

### 4.2 为什么 `accept_callback` 与 `recv_callback` 使用 `union`

监听 fd 的读事件表示“有连接可接受”，其回调是 `accept_cb()`；客户端 fd 的读事件表示“有数据可接收”，其回调是 `recv_cb()`。

同一个 fd 在这里不会同时既是监听 fd 又是客户端 fd，因此两个函数指针可以共用同一块存储空间。`union` 表达了二者“二选一”的关系。

不过，代码在赋值时使用了：

```c
conn_list[sockfd].r_action.recv_callback = accept_cb;
```

语义上更清晰的写法是：

```c
conn_list[sockfd].r_action.accept_callback = accept_cb;
```

调用时也应根据 fd 的角色使用对应成员，或者直接将二者合并成一个名称更通用的 `read_callback`。

### 4.3 为什么可以用 fd 作为数组下标

当前代码使用：

```c
struct conn conn_list[CONNECTION_SIZE];
```

并直接通过 `conn_list[fd]` 找到连接。这种方法查找复杂度为 `O(1)`，非常直接。

但它有两个条件：

1. 必须保证 `0 <= fd < CONNECTION_SIZE`，否则会越界。
2. 数组内存非常大。

每个连接对象包含两个 1024 字节缓冲区，再加上长度、fd 和函数指针。100 多万个元素大约需要 2 GiB 以上的地址空间和可观的实际内存。因此“支持 100 万个 fd”不等于“采用这种结构后机器就一定能稳定承载 100 万连接”。生产实现通常会根据实际连接动态分配，或者按块分配连接对象与缓冲区。

---

## 5. 程序完整执行流程

### 5.1 初始化 epoll

代码中：

```c
epfd = epoll_create(1);
```

参数在现代 Linux 中只要大于 0 即可，并不表示只能监听一个 fd。更推荐：

```c
epfd = epoll_create1(EPOLL_CLOEXEC);
```

创建失败时必须检查返回值。

### 5.2 创建 20 个监听 socket

```c
for (i = 0; i < MAX_PORTS; i++) {
    int sockfd = init_server(port + i);
    ...
}
```

程序监听 `2000~2019` 共 20 个端口。每个监听 fd 都注册 `EPOLLIN`，并对应 `accept_cb()`。

监听 fd 上的 `EPOLLIN` 不是普通业务数据可读，而是表示全连接队列中存在已经完成握手、可以由 `accept()` 取出的连接。

### 5.3 接受新连接

`accept_cb()` 的逻辑是：

1. 调用 `accept()`，获得 `clientfd`。
2. 检查 `clientfd < 0`。
3. 初始化 `conn_list[clientfd]`。
4. 给客户端 fd 注册 `EPOLLIN`。

这时事件映射从：

```text
listenfd + EPOLLIN → accept_cb
```

变成新连接上的：

```text
clientfd + EPOLLIN → recv_cb
```

### 5.4 接收数据

当客户端 fd 可读时执行 `recv_cb()`：

```c
count = recv(fd, conn_list[fd].rbuffer, BUFFER_LENGTH, 0);
```

当前 Echo 逻辑把收到的数据复制到写缓冲区：

```c
conn_list[fd].wlength = conn_list[fd].rlength;
memcpy(conn_list[fd].wbuffer,
       conn_list[fd].rbuffer,
       conn_list[fd].wlength);
```

随后把关注事件修改为 `EPOLLOUT`：

```text
clientfd + EPOLLOUT → send_cb
```

### 5.5 发送数据

当 fd 可写时执行 `send_cb()`。数据发送结束后，再把关注事件切换回 `EPOLLIN`，等待客户端下一次发送。

完整状态流转如下：

```text
监听 EPOLLIN
    ↓
accept_cb：建立 clientfd
    ↓
客户端 EPOLLIN
    ↓
recv_cb：接收并准备响应
    ↓
客户端 EPOLLOUT
    ↓
send_cb：发送响应
    ↓
重新监听客户端 EPOLLIN
```

---

## 6. `set_event()` 与事件注册

```c
int set_event(int fd, int event, int flag)
```

当前约定：

- `flag == 1`：使用 `EPOLL_CTL_ADD`。
- `flag == 0`：使用 `EPOLL_CTL_MOD`。

功能上可行，但 `flag` 的含义不直观。更清晰的设计是直接传入 `EPOLL_CTL_ADD`、`EPOLL_CTL_MOD`，或者分别封装成 `add_event()` 和 `mod_event()`。

同时必须检查 `epoll_ctl()` 的返回值。例如：

```c
if (epoll_ctl(epfd, op, fd, &ev) == -1) {
    perror("epoll_ctl");
    return -1;
}
```

`set_event()` 和 `event_rigister()` 声明为返回 `int`，但成功路径没有 `return`，这属于未定义行为。应补充 `return 0`。另外，`rigister` 应改为 `register`。

---

## 7. 主事件循环

```c
while (1) {
    int nready = epoll_wait(epfd, events, 1024, -1);

    for (i = 0; i < nready; i++) {
        int connfd = events[i].data.fd;

        if (events[i].events & EPOLLIN)
            conn_list[connfd].r_action.recv_callback(connfd);

        if (events[i].events & EPOLLOUT)
            conn_list[connfd].send_callback(connfd);
    }
}
```

`timeout == -1` 表示没有事件时一直阻塞，避免空转消耗 CPU。

### 两个独立的 `if` 还是 `if ... else if`？

两个独立 `if` 理论上允许同一次返回同时处理读、写事件；`else if` 每轮只处理其中一种。

但不能简单认为两个 `if` 一定更好。若读回调发现连接已经断开并关闭了 fd，随后继续执行写回调就可能访问已经失效或被复用的 fd。稳妥做法是：

1. 先处理 `EPOLLERR`、`EPOLLHUP`、`EPOLLRDHUP`。
2. 回调返回关闭状态后，不再处理该 fd 的其他事件。
3. 再依据连接状态处理读写事件。

---

## 8. 课堂代码补充

这份代码主要是为了演示 Reactor 的整体流程，所以有些细节课堂上只是带了一下。自己再写一遍时，可以顺手补上下面这些地方。

**关于 `send()`**

课上代码里有一句：

```c
int count = send(fd, conn_list[fd].wbuffer, count, 0);
```

这里最后的 `count` 还没有赋值，应该写成 `conn_list[fd].wlength`。

另外，一次 `send()` 不一定能把 `wlength` 个字节全部发完。可以在连接对象里再记一个 `woffset`，每次发送后往后移动。数据还没发完就继续等 `EPOLLOUT`，全部发完后再切回 `EPOLLIN`。

**关于 `recv()`**

记住三个返回值：

- 大于 0：本次收到的字节数；
- 等于 0：对端关闭连接；
- 小于 0：要结合 `errno` 判断，非阻塞情况下可能只是暂时没有数据。

不能让 `-1` 继续参与后面的 `memcpy()`，否则长度转换后可能越界。`EINTR` 可以重试，`EAGAIN/EWOULDBLOCK` 表示这次先读到这里，其他情况再按真正的错误处理。

还有一点：`recv()` 收到的是一段字节，不会自动在末尾补 `\0`。要用 `%s` 打印时需要自己留位置并补结束符；如果处理二进制数据，就一直按长度来处理。

**关于非阻塞**

Reactor 一般要和非阻塞 socket 配合。监听 fd 和客户端 fd 都可以设置成 `O_NONBLOCK`，避免某一次 `accept()`、`recv()` 或 `send()` 卡住整个事件循环。

```c
fcntl(fd, F_SETFL, fcntl(fd, F_GETFL, 0) | O_NONBLOCK);
```

如果以后从 LT 改成 ET，`accept()` 和 `recv()` 通常都要放在循环里，一直处理到返回 `EAGAIN/EWOULDBLOCK`。当前是 LT，一次只 `accept()` 一个连接也能继续触发，不过连接集中到来时，循环取完会更快一些。

**关于 fd 和连接状态**

当前写法直接用 fd 作为 `conn_list` 的下标，使用前要判断：

```c
if (fd < 0 || fd >= CONNECTION_SIZE)
    return -1;
```

连接关闭时也不只是 `close(fd)`，对应的读写长度、偏移量和回调最好一起清空。因为 fd 的数字会被复用，下一个新连接有可能再次拿到同一个编号。

fd 也不能当作累计连接数。想每建立 1000 个连接打印一次，应该另外维护一个 `accepted_count`，不能用 `clientfd % 1000` 来判断。

**其他随手记下的点**

- `socket()`、`bind()`、`listen()`、`epoll_ctl()`、`epoll_wait()` 等系统调用都要看返回值；
- 监听 socket 通常会设置 `SO_REUSEADDR`；
- `listen(sockfd, 10)` 中的 `10` 是 backlog，不是服务器最多只能保持 10 条连接；
- 主循环除了 `EPOLLIN`、`EPOLLOUT`，还要留意 `EPOLLERR`、`EPOLLHUP` 和 `EPOLLRDHUP`；
- 对端关闭后继续发送可能触发 `SIGPIPE`，可以忽略该信号，或在 `send()` 时使用 `MSG_NOSIGNAL`；
- `TIME_SUB_MS(current, begin)` 当前记录的是两次打印之间的间隔，不是从程序启动开始的总耗时。

---

## 9. 大包、拆包与粘包

代码中的读缓冲区只有 1024 字节。若客户端发送 2048 字节，一次 `recv()` 可能只收到其中一部分；即使缓冲区足够大，TCP 也不保证一次接收对应一次发送。

因此不能把：

```text
一次 send = 一个业务消息 = 一次 recv
```

当成可靠规律。

业务协议需要定义消息边界，常见方法有：

1. 固定长度消息。
2. 特殊分隔符，例如文本协议中的换行。
3. 消息头携带正文长度，例如“4 字节长度 + 消息体”。
4. HTTP 中使用请求行、首部、`Content-Length` 或分块传输规则。

Reactor 收到数据后应先追加到该连接自己的缓冲区，再交给协议解析器：

```text
recv → 追加到连接缓冲区 → 判断是否形成完整消息
     → 未完整：保留并等待下次 EPOLLIN
     → 已完整：取出消息并处理，剩余数据继续保留
```

如果固定数组空间不足，可以使用动态缓冲区、环形缓冲区或链式缓冲区。

---

## 10. 为什么会出现 `Too many open files`

报错对应 `errno == EMFILE`（常见值为 24），表示当前进程已经达到可打开 fd 的上限。

一条 TCP 连接在客户端和服务端各自都占用一个 fd。百万连接测试至少要同时考虑：

### 10.1 进程级限制

查看当前 shell 的限制：

```bash
ulimit -n
ulimit -a
```

临时提高当前 shell 的 soft limit：

```bash
ulimit -n 1048576
```

程序必须从修改后的 shell 中启动，才能继承该限制。还要确保 hard limit 足够高。

### 10.2 系统级限制

```bash
cat /proc/sys/fs/file-max
cat /proc/sys/fs/file-nr
```

`fs.file-max` 是系统范围的文件句柄限制，`ulimit -n` 是单进程/会话限制，两者不是同一个概念。持久化设置还取决于系统的 PAM、systemd unit 和发行版配置，不能只改一个参数就假定生效。

### 10.3 程序自身限制

- `CONNECTION_SIZE` 是否覆盖可能出现的 fd 值；
- 进程虚拟内存和实际物理内存是否足够；
- 每连接缓冲区是否过大；
- 是否存在 fd 泄漏；
- epoll 实例和监听 socket 自己也会占 fd。

---

## 11. 为什么单个目标端口大约只能建立数万连接

一条 TCP 连接由五元组唯一标识：

```text
源 IP、源端口、目的 IP、目的端口、传输层协议
```

在单台压测客户端连接单台服务器的单个端口时：

- 源 IP 固定；
- 目的 IP 固定；
- 目的端口固定；
- 协议固定为 TCP；
- 主要只能变化源端口。

当可用临时源端口耗尽时，客户端可能报：

```text
Cannot assign requested address
```

### 11.1 一个需要校正的细节

客户端临时端口范围并非固定为 `1024~65535`。Linux 实际自动分配范围应通过下面的命令查看：

```bash
sysctl net.ipv4.ip_local_port_range
```

不同系统、不同配置的范围可能不同，部分端口还可能被占用、保留，或暂时处于 `TIME_WAIT` 等状态。因此可建立数量通常小于理论上的 65535。

### 11.2 为什么增加到 20 个服务端端口有效

将目标端口从一个增加到 `2000~2019`，改变了五元组中的“目的端口”。只要客户端把连接分散到这些端口，同一个源端口就可以与不同目的端口组成不同连接，理论连接空间随之扩大。

其他扩展方式还包括：

- 使用多个客户端源 IP；
- 使用多台压测机；
- 使用多个服务端 IP；
- 使用多个网络命名空间，但必须正确配置各自地址和路由。

不能仅用单台客户端的结果直接代表服务端的最大承载能力，因为压测端本身可能先耗尽端口、fd、CPU、内存或 conntrack 容量。

---

## 12. `nf_conntrack` 应该怎样理解

`nf_conntrack` 是 Linux Netfilter 的连接跟踪机制，防火墙、NAT 等功能会用到它。它不是 `epoll` 工作所必需的组成部分。

如果测试路径启用了连接跟踪，并且连接数达到表容量，才需要观察：

```bash
sysctl net.netfilter.nf_conntrack_max
cat /proc/sys/net/netfilter/nf_conntrack_count
```

旧资料中的 `ip_conntrack` 是较早的名称；现代内核通常使用 `nf_conntrack`。只有确实需要且模块尚未加载时，才考虑：

```bash
sudo modprobe nf_conntrack
```

不应看到连接问题就盲目加载模块或增大参数。应先确认机器是否启用了防火墙/NAT、模块是否存在、表是否真的已满。在服务端还是客户端调整，取决于哪台机器实际承担连接跟踪；中间的 NAT/防火墙设备也可能成为瓶颈。

---

## 13. 百万连接断开时为什么 CPU 会突然升高

如果一次性终止百万连接的一端，另一端会在短时间内集中处理大量 TCP 状态变化与关闭事件，例如 FIN/RST、epoll 唤醒、连接对象清理、定时器更新等。

这会形成“惊群式的业务处理洪峰”或资源回收洪峰，表现为 CPU 短时暴增。它不一定表示程序死循环，而可能是系统在集中处理积压事件。

优化方向包括：

- 分批建立和分批关闭连接；
- 减少关闭路径上的日志和同步操作；
- 将耗时清理异步化；
- 使用高效定时器结构，如时间轮；
- 避免每连接大量内存分配和释放；
- 做好背压和过载保护；
- 使用多 Reactor、多线程或多进程分摊事件处理；
- 用火焰图、`perf`、系统指标确认 CPU 真正消耗在哪里。

优化目标不是保证 CPU 永不升高，而是让峰值可控、持续时间缩短，并保证服务仍能继续响应。

---

## 14. “百万并发”不等于“百万 QPS”

性能测试至少要区分四个维度：

| 指标 | 含义 |
|---|---|
| 并发连接数 | 同一时刻保持的连接数量 |
| QPS/吞吐量 | 单位时间完成的请求或消息数量 |
| 延迟 | 一次请求从发出到收到响应所需时间，通常关注 P50/P95/P99 |
| 测试用例 | 报文大小、收发频率、连接是否长连、业务是否计算或访问存储 |

保持 100 万条空闲 TCP 连接，和让 100 万条连接持续高速收发数据，是完全不同的负载。前者更考验 fd、内存和内核连接管理；后者还会迅速消耗 CPU、网络带宽、内存带宽以及业务依赖资源。

因此，一份可信的测试报告至少应说明：

- 客户端和服务端硬件配置；
- 内核和发行版版本；
- 所有相关内核参数与 fd 限制；
- 连接建立速率；
- 活跃连接与空闲连接比例；
- 请求/响应大小和频率；
- QPS、吞吐量、错误率和延迟分位数；
- CPU、内存、网络、fd、conntrack 使用率；
- 测试持续时间和连接关闭方式。

---

---

## 15. 本节课一句话总结

Reactor 的本质不是“用了 epoll 就很快”，而是把每个 fd 的状态独立保存起来，由事件循环将 `I/O + event` 精确分发给相应 callback；百万连接则是在这个正确模型之上，再共同解决 fd、内存、端口空间、内核网络栈和压测方法等系统性限制。

---
