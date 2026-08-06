---
title: IO 多路复用与 Reactor
slug: network-io-and-io-multiplexing
description: 从一连接一线程出发，结合核心代码理解 select、poll、epoll 的事件机制，以及 Reactor 如何把 IO 管理抽象为事件与回调管理
date: 2026-08-06T00:00:00+08:00
lastmod: 2026-08-07T00:00:00+08:00
draft: false
image: cover.svg
tags:
  - 网络编程
  - Linux
  - IO 多路复用
  - Reactor
categories:
  - 后端开发
---

## 1. 为什么需要 IO 多路复用

网络服务器主要管理两类 socket：

1. **监听 socket（`listenfd` / 代码中的 `sockfd`）**
   它出现可读事件，通常表示已经完成 TCP 握手的连接正在等待 `accept()`。
2. **已连接 socket（`clientfd` / `connfd`）**
   它出现可读事件，可能表示收到了数据，也可能表示对端关闭连接；随后由 `recv()` 判断具体情况。它也可能出现可写、错误、挂断等事件。

最直接的服务器模型是“一连接一线程”：主线程执行 `accept()`，每建立一个连接便创建一个线程，由该线程阻塞在 `recv()` 上。

```c
int clientfd = accept(sockfd, ...);
pthread_create(&thid, NULL, client_thread, &clientfd);
```

这种模型的优点是代码直观，一个线程可以按顺序处理一个连接的收、发逻辑。缺点是连接数增加时，需要大量线程；线程会占用栈空间和内核资源，还会产生调度及上下文切换成本。许多长连接大部分时间并没有数据，线程却仍然存在，因此不适合大量空闲连接并发存在的场景。

**IO 多路复用的核心思想**：让一个线程同时等待多个 fd。线程不再阻塞于某一个 `recv()`，而是阻塞在 `select()`、`poll()` 或 `epoll_wait()` 上；哪些 fd 已经就绪，线程就处理哪些 fd。

> “多路”指多个 IO/fd，“复用”指复用同一个线程进行等待和处理。它复用的是等待能力，并不会自动并行执行业务代码。

---

## 2. “IO 就绪”与“完成 IO”

`select`、`poll`、`epoll` 提供的是**就绪通知**机制：

- 监听 fd 可读：通常可以调用 `accept()`。
- 客户端 fd 可读：可以调用 `recv()`；也可能是对端关闭或发生错误。
- 客户端 fd 可写：内核发送缓冲区当前有空间，并不等于全部数据已经发送完成。

它们只告诉应用程序“现在适合尝试某项 IO 操作”，不会替应用程序完成请求解析、业务计算和响应组装。

一个长连接的生命周期会经历许多次事件，而不是“连接一次只触发一个事件”：

```text
建立连接 → 收到请求 → 发送响应 → 再次收到请求 → …… → 断开连接
```

因此，用户在线数或 TCP 连接数并不等于同一时刻的活跃请求数。例如服务器可能维护 100 万个连接，但某一时刻只有 1 万个连接真正产生事件。高并发服务器真正希望快速取得并处理的是这 1 万个**就绪 fd**。

---

## 3. select

### 3.1 接口

```c
int select(int nfds,
           fd_set *readfds,
           fd_set *writefds,
           fd_set *exceptfds,
           struct timeval *timeout);
```

五类参数的含义：

- `nfds`：三个集合中最大 fd 值加 1，而不是 fd 的数量。
- `readfds`：关注可读事件的 fd 集合。
- `writefds`：关注可写事件的 fd 集合。
- `exceptfds`：关注异常条件的 fd 集合；它并不是所有网络错误的统一集合。
- `timeout`：最长等待时间。
  - `NULL`：一直阻塞，直到出现事件或被信号中断。
  - `{0, 0}`：立即返回，成为轮询。
  - 指定时间：超时后返回 0，可让事件循环顺便检查定时任务，但精确定时器通常还需要专门的数据结构。

返回值：

- `> 0`：返回集合中就绪项的数量。
- `= 0`：超时。
- `= -1`：调用失败，例如被信号中断时 `errno == EINTR`。

### 3.2 如何理解 fd_set

`fd_set` 可以理解成一个**位图集合**：fd 是非负整数，位图中编号为 fd 的那一位表示该 fd 是否属于集合。

```text
fd 编号： 0 1 2 3 4 5 6 7 8
位图值：  0 0 0 1 1 1 1 0 0
                    ↑
             关注 3、4、5、6
```

常用宏：

```c
FD_ZERO(&set);       // 清空集合
FD_SET(fd, &set);    // 加入 fd
FD_CLR(fd, &set);    // 移除 fd
FD_ISSET(fd, &set);  // 判断 fd 是否位于集合中
```

在典型 Linux/glibc 环境中，`FD_SETSIZE` 为 1024，`fd_set` 可表示编号 0～1023 的 fd，通常占 1024 bit，即 128 byte。但应以当前平台的 `FD_SETSIZE` 和 `sizeof(fd_set)` 为准。

需要注意：在 Linux 上，仅仅尝试修改 `FD_SETSIZE` 并不能可靠地突破 `select()` 的 1024 fd 限制；处理大量 fd 应改用 `poll` 或 `epoll`。此外，“最多 1024 个 fd”并不完全等于“最多 1024 个客户端”，因为进程还会占用标准输入输出、监听 socket、文件等其他 fd，而且 fd 编号必须小于 `FD_SETSIZE`。

### 3.3 为什么代码需要 rfds 和 rset 两份集合

附件代码中：

```c
fd_set rfds, rset;
FD_ZERO(&rfds);
FD_SET(sockfd, &rfds);

while (1) {
    rset = rfds;
    int nready = select(maxfd + 1, &rset, NULL, NULL, NULL);
    ...
}
```

原因是 `select()` 的 fd 集合是**传入兼传出参数**：

- 调用前，集合表示“应用程序关注哪些 fd”。
- 返回后，集合被改写，只保留本次已经就绪的 fd。

因此：

- `rfds` 保存长期关注的完整集合（整集）。
- `rset` 是每次调用时复制出的工作集合，返回后用于判断本轮哪些 fd 就绪。

代码先判断监听 fd：

```c
if (FD_ISSET(sockfd, &rset)) {
    int clientfd = accept(sockfd, ...);
    FD_SET(clientfd, &rfds);
}
```

然后从较小 fd 一直扫描到 `maxfd`，找出就绪客户端并执行 `recv()`：

```c
for (int i = sockfd + 1; i <= maxfd; i++) {
    if (FD_ISSET(i, &rset)) {
        recv(i, ...);
    }
}
```

### 3.4 优缺点

优点：

- 一个线程可以等待多个 fd。
- 跨 Unix 类系统的可移植性较好。
- fd 数量较少时足够简单实用。

缺点：

- 每次调用都要把集合从用户态复制到内核态，返回时还要复制回来。
- 内核需要检查关注集合；应用层也要扫描到 `maxfd` 才知道谁就绪。
- 集合会被改写，调用前必须重新准备。
- Linux 上受 `FD_SETSIZE` 限制。
- 参数和集合管理较烦琐。

---

## 4. poll

### 4.1 数据结构与接口

`poll` 使用结构体数组描述关注对象：

```c
struct pollfd {
    int   fd;       // 被监控的 fd
    short events;   // 输入：关注的事件
    short revents;  // 输出：实际发生的事件
};

int poll(struct pollfd *fds, nfds_t nfds, int timeout);
```

常见事件包括：

- `POLLIN`：可读。
- `POLLOUT`：可写。
- `POLLERR`：发生错误。
- `POLLHUP`：挂断。
- `POLLNVAL`：fd 无效。

`timeout` 的单位是毫秒：

- `-1`：一直阻塞。
- `0`：立即返回。
- `> 0`：最多等待指定毫秒数。

与 `select` 相比，`pollfd` 把“关注的事件”和“实际发生的事件”分开，因此不必在每轮调用前重建整个关注集合。它也没有 `FD_SETSIZE` 这种固定位图上限；实际容量仍受进程 fd 限额和内存限制。

### 4.2 poll 是否从根本上解决了性能问题

没有。每次调用 `poll()` 时，仍需把 `pollfd` 数组交给内核；内核和应用层仍需要在数组中检查事件。因此大量 fd 中只有少量就绪时，工作量仍与整个关注集合的规模相关。

`poll` 的主要价值是接口更统一、没有 `select` 的固定 fd 编号限制，而不是在大量连接场景中获得 `epoll` 那种只返回就绪项的机制。

适合场景通常包括：

- 需要比 `select` 更方便的事件表达方式。
- fd 数量不特别大。
- 需要较好的 Unix 平台可移植性，而不能使用 Linux 专有的 `epoll`。

### 4.3 附件 poll 代码中的两个重要问题

原代码连续调用了两次 `poll()`：

```c
poll(fds, maxfd + 1, -1);
int nready = poll(fds, maxfd + 1, -1);
```

第一次调用已经取得事件，第二次又重新阻塞；应删除第一次，只保留：

```c
int nready = poll(fds, maxfd + 1, -1);
```

此外，代码用 fd 值作为数组下标，但数组被 `{0}` 初始化。未使用位置的 `fd` 因而是 `0`，会意外重复监控标准输入。`poll` 约定 `fd < 0` 的项会被忽略，因此所有空槽应初始化为 `-1`。更常见的写法是紧凑地保存有效 `pollfd` 项，而不是把 fd 直接当数组下标。

---

## 5. epoll

`epoll` 是 Linux 专用的 IO 事件通知机制。它将“维护关注集合”和“等待就绪事件”拆开，尤其适合**大量连接、少量活跃**的服务器场景。

### 5.1 三个核心接口

#### 1. 创建 epoll 实例

```c
int epfd = epoll_create(1);
```

返回的 `epfd` 本身也是一个 fd，代表一个内核中的 epoll 实例。较新的代码通常使用：

```c
int epfd = epoll_create1(EPOLL_CLOEXEC);
```

`epoll_create(size)` 的 `size` 在现代 Linux 中已被忽略，但必须大于 0；保留它是为了兼容旧接口。它不是“单次最多返回多少个就绪事件”。单次最多返回多少项由 `epoll_wait()` 的 `maxevents` 决定。

#### 2. 增删改关注对象

```c
int epoll_ctl(int epfd, int op, int fd,
              struct epoll_event *event);
```

三种操作：

- `EPOLL_CTL_ADD`：加入一个 fd。
- `EPOLL_CTL_DEL`：删除一个 fd。
- `EPOLL_CTL_MOD`：修改该 fd 关注的事件或关联数据。

事件结构体：

```c
struct epoll_event {
    uint32_t events;  // EPOLLIN、EPOLLOUT 等
    epoll_data_t data; // 可保存 fd、指针等用户数据
};
```

附件先注册监听 fd：

```c
ev.events = EPOLLIN;
ev.data.fd = sockfd;
epoll_ctl(epfd, EPOLL_CTL_ADD, sockfd, &ev);
```

`accept()` 得到新客户端后，再单独将 `clientfd` 加入关注集合。

#### 3. 等待就绪事件

```c
int nready = epoll_wait(epfd, events, maxevents, timeout);
```

- `events`：用户提供的结果数组，用来接收本次就绪项。
- `maxevents`：结果数组最多能装多少项，必须大于 0；它不是 epoll 能管理的 fd 总数。
- `timeout`：`-1` 一直等待，`0` 立即返回，正数表示最多等待的毫秒数。
- 返回值：本轮写入 `events` 数组的就绪项数，超时返回 0，失败返回 -1。

应用只需遍历 `events[0..nready-1]`：

```c
for (int i = 0; i < nready; i++) {
    int fd = events[i].data.fd;
    ...
}
```

### 5.2 “整集”和“就绪集”

从概念上看，epoll 实例在内核中维护两个重要集合：

1. **关注集合（interest list）**：应用通过 `epoll_ctl()` 增、删、改的所有 fd。
2. **就绪集合（ready list）**：其中当前有事件需要应用处理的 fd。

传统实现说明中常把关注集合描述为红黑树、就绪集合描述为链表。但这些属于 Linux 内核实现细节，理解接口时更重要的是“长期关注集合”和“当前就绪集合”的分离，不应让业务代码依赖具体内部数据结构。

### 5.3 为什么 epoll 更适合大并发

以 100 万个连接、当前 1 万个连接活跃为例：

- `select/poll`：每轮等待都需要提交并检查规模接近 100 万的关注集合，应用还要从大集合里寻找就绪项。
- `epoll`：连接建立或状态变化时，通过 `epoll_ctl()` 增量维护关注集合；`epoll_wait()` 主要把当前就绪项返回给应用，应用遍历约 1 万个结果。

因此，epoll 的优势不应简单概括成“完全没有遍历或拷贝”，而应表述为：

- 不必在每次等待时重新提交整个关注集合。
- 应用遍历的是本轮返回的就绪项，而不是从 0 扫描至最大 fd。
- 连接管理是增量进行的，适合大量长连接中只有少量活跃的情形。

不过，epoll 不会消除业务处理、数据复制、系统调用等全部成本；当绝大部分连接始终活跃时，它相对 `poll` 的优势也会缩小。

### 5.4 LT 与 ET

epoll 有两种常见触发方式：

- **LT（Level Triggered，水平触发）**：默认方式。只要 fd 仍处于就绪状态，后续 `epoll_wait()` 还会继续报告，较容易编写。
- **ET（Edge Triggered，边沿触发）**：只有状态发生变化时重点通知。通常必须配合非阻塞 fd，并持续 `accept()` / `recv()` / `send()`，直到返回 `-1` 且 `errno` 为 `EAGAIN` 或 `EWOULDBLOCK`，否则可能遗漏尚未处理完的数据。

附件只设置了 `EPOLLIN`，没有设置 `EPOLLET`，因此使用的是 LT。

---

## 6. 从 IO 多路复用到 Reactor

`select`、`poll`、`epoll` 解决的是：

> 哪些 fd 上发生了应用关心的 IO 事件？

但实际服务器还需要解决：

> 某类事件发生以后，应当执行哪段业务逻辑？

Reactor 是一种**事件驱动的程序组织模式**。应用先把“事件”和“处理函数”注册起来；事件循环等待事件，事件发生后再把它分派给对应处理函数。

```text
注册 fd + 关注事件 + 回调函数
              ↓
    select / poll / epoll 等待
              ↓
         得到就绪事件
              ↓
       Reactor 分派事件
              ↓
 accept_cb / recv_cb / send_cb
```

例如：

```c
listenfd + READ  -> accept_cb()
clientfd + READ  -> recv_cb()
clientfd + WRITE -> send_cb()
```

因此，可以把两者的关系概括为：

- `select/poll/epoll` 是底层的**事件检测或事件分离机制**。
- Reactor 是上层的**事件注册、循环等待和回调分派模式**。
- Reactor 关注的中心从“对某个 fd 写一段固定流程”转为“某个 fd 的某种事件应该调用哪个 handler”。

它和路由有相似之处：路由根据请求路径找到处理器，Reactor 根据 fd 上发生的事件找到处理器。但 Reactor 还负责事件监听、生命周期管理和持续循环，因此不能简单等同于路由。

典型 Reactor 由以下部分组成：

- **事件源**：监听 socket、客户端 socket、定时器等。
- **事件多路分离器**：`select`、`poll` 或 `epoll`。
- **Reactor / Dispatcher**：事件循环与分派器。
- **Handler / Callback**：处理连接、读取、写入、关闭等事件的函数。

Reactor 只是组织 IO 事件处理的模式，并不意味着业务一定只使用一个线程。常见结构包括：

- 单 Reactor、单线程。
- 单 Reactor + 工作线程池。
- 主 Reactor 负责接收连接，多个从 Reactor 负责已连接 socket，再配合工作线程池。

耗时业务不能长期阻塞事件循环，否则一个回调会拖延其他连接；CPU 密集型或阻塞型任务通常交给工作线程池。

---

## 7. 三种工具对比

| 特性 | select | poll | epoll |
|---|---|---|---|
| 平台 | 多种 Unix 类系统 | 多种 Unix 类系统 | Linux |
| 关注集合表示 | `fd_set` 位图 | `pollfd` 数组 | 内核维护的 epoll 关注集合 |
| 每轮是否重新提交整个关注集合 | 是 | 是 | 否，使用 `epoll_ctl` 增量维护 |
| 应用如何找就绪项 | 扫描到 `maxfd` | 扫描数组 | 遍历 `epoll_wait` 返回的就绪数组 |
| 固定 1024 fd 编号限制 | Linux 上通常有 | 无此固定限制 | 无此固定限制 |
| 主要适用情况 | fd 较少、重视可移植性 | fd 中等、重视可移植性和统一事件接口 | Linux 大量连接、少量活跃 |

不能笼统地说谁在所有场景下一定最快。fd 很少时差别可能很小；平台兼容性、接口复杂度、连接活跃比例和业务负载都应纳入选择。

---

## 8. 附件代码的演进关系

代码使用条件编译展示了四个阶段：

```text
单连接阻塞处理
    ↓
循环 accept，但一个连接的 recv 会阻塞后续连接
    ↓
一连接一线程，逻辑简单但线程成本随连接数增长
    ↓
select / poll / epoll，一个线程等待多个 fd
    ↓
进一步封装事件、状态与回调，即 Reactor
```

当前真正编译的是最后的 `#else`，即 epoll 版本；前面的分支因为使用 `#if 0` / `#elif 0` 而不会参与编译。

---

## 9. 先记住三套代码共同的骨架

学习这三个接口时，不要一开始背所有参数。先固定服务器永远重复的四步：

```text
1. 注册：告诉系统要关注哪些 fd、哪些事件
2. 等待：阻塞到至少一个事件发生
3. 分发：判断是监听 fd，还是客户端 fd
4. 更新：新连接加入，断开的连接删除，需要时修改关注事件
```

把三种接口代入同一骨架：

| 阶段 | select | poll | epoll |
|---|---|---|---|
| 保存关注对象 | `fd_set rfds` | `struct pollfd fds[]` | 内核中的 epoll 关注集合 |
| 注册新客户端 | `FD_SET(clientfd, &rfds)` | 添加一个 `pollfd` | `epoll_ctl(...ADD...)` |
| 等待事件 | `select(...)` | `poll(...)` | `epoll_wait(...)` |
| 找出就绪对象 | 扫描并调用 `FD_ISSET` | 扫描并检查 `revents` | 直接遍历返回的 `events` |
| 删除客户端 | `FD_CLR(fd, &rfds)` | 删除数组项或设为 `-1` | `epoll_ctl(...DEL...)` |

下面三段代码都实现同一个核心功能：

```text
监听 fd 可读   → accept 新连接 → 注册 clientfd
clientfd 可读  → recv 数据     → send 回显
连接关闭/出错  → 删除 clientfd → close
```

为了突出多路复用主干，示例省略了非阻塞设置、完整发送缓冲区、信号处理等生产级细节。

---

## 10. select 核心代码逐段对照

### 10.1 初始化：建立长期关注集合

```c
fd_set rfds, rset;           // 完整关注集合、本轮就绪集合
FD_ZERO(&rfds);              // 初始为空
FD_SET(sockfd, &rfds);       // 首先关注监听 fd

int maxfd = sockfd;          // select 要知道最大的 fd
```

此时可以想象：

```text
rfds = { sockfd }
```

### 10.2 每轮复制并等待

```c
while (1) {
    rset = rfds;
    int nready = select(maxfd + 1, &rset, NULL, NULL, NULL);
```

这里两个集合的职责一定要分清：

```text
调用前：rfds = 长期关注的完整集合
        rset = 复制品，也包含完整集合

返回后：rfds = 保持不变，供下一轮继续使用
        rset = 被 select 改写，只保留本轮就绪 fd
```

`maxfd + 1` 中的 `+1` 是因为 `nfds` 表示检查区间的右边界：内核检查 fd `0` 到 `maxfd`，即 `[0, maxfd + 1)`。

### 10.3 监听 fd 就绪：接收并注册新连接

```c
    if (FD_ISSET(sockfd, &rset)) {
        // sockfd 可读不是有业务数据，而是有连接可以取出
        int clientfd = accept(sockfd,
                              (struct sockaddr *)&clientaddr,
                              &len);

        // accept 不会删除 sockfd；它返回一个新的 clientfd
        FD_SET(clientfd, &rfds);

        if (clientfd > maxfd)
            maxfd = clientfd;
    }
```

要点：

- 检查的是返回集合 `rset`，因为要问“本轮谁就绪”。
- 新连接加入的是长期集合 `rfds`，因为要从下一轮开始持续关注它。
- `accept()` 从已完成连接队列取出一个连接，但 `sockfd` 仍然继续监听。
- 新 `clientfd` 本轮不在 `rset` 中，从下一轮开始接受监控。

### 10.4 扫描客户端：处理并删除

```c
    for (int fd = sockfd + 1; fd <= maxfd; ++fd) {
        if (!FD_ISSET(fd, &rset))
            continue;  // 这个 fd 本轮不可读

        char buffer[1024] = {0};
        int count = recv(fd, buffer, sizeof(buffer), 0);

        if (count > 0) {
            send(fd, buffer, count, 0); // Echo：原样回发
        } else {
            // count == 0 表示对端关闭；教学代码也在出错时删除
            FD_CLR(fd, &rfds);
            close(fd);
        }
    }
}
```

这里的“可读”应理解为：现在调用读取类操作能够立即得到结果，而不会因为没有事件一直阻塞：

```text
监听 fd 可读   → 调用 accept()，取出一个连接
客户端 fd 可读 → 调用 recv()，读取数据或得知对端关闭
```

`maxfd = sockfd` 也不是说监听 fd 永远最大，而是初始化时集合中只有 `sockfd`。加入新客户端以后，再通过比较更新 `maxfd`。

select 版最值得背下来的代码主干是：

```c
rset = rfds;
nready = select(maxfd + 1, &rset, NULL, NULL, NULL);

if (FD_ISSET(sockfd, &rset)) {
    clientfd = accept(...);
    FD_SET(clientfd, &rfds);
}

for (fd = sockfd + 1; fd <= maxfd; fd++) {
    if (FD_ISSET(fd, &rset))
        recv(fd, ...);
}
```

> 记忆关键词：**复制集合、`maxfd + 1`、扫描、`FD_ISSET`。**

---

## 11. poll：对照附件代码理解

你的写法直接把 fd 当作 `fds` 数组下标：fd 3 放在 `fds[3]`，fd 4 放在 `fds[4]`。下面保留这种写法，方便和原代码逐行对应。

### 11.1 初始化数组

```c
#define MAX_FDS 1024

struct pollfd fds[MAX_FDS];

// poll 会忽略 fd < 0 的项
for (int i = 0; i < MAX_FDS; ++i) {
    fds[i].fd = -1;
    fds[i].events = 0;
    fds[i].revents = 0;
}

fds[sockfd].fd = sockfd;
fds[sockfd].events = POLLIN;

int maxfd = sockfd;
```

此时：

```text
假设 sockfd = 3：

fds[3].fd     = 3
fds[3].events = POLLIN
```

`events = POLLIN` 表示“我希望观察可读”，`revents` 表示 `poll()` 返回时“本轮实际发生了什么”。

### 11.2 等待事件

```c
while (1) {
    int nready = poll(fds, maxfd + 1, -1);
```

调用前后：

```text
events  = 输入：希望关注什么
revents = 输出：实际上发生了什么
```

与 select 不同，`events` 不会被替换成就绪集合，因此下一轮通常不必重建所有关注事件；但应把 `revents` 当作当轮结果使用。

附件原本连续调用了两次 `poll()`，应删掉第一次，否则第一次取得事件后，第二次又会重新阻塞。

### 11.3 监听 fd 就绪：注册新客户端

```c
    if (fds[sockfd].revents & POLLIN) {
        // 监听 fd 可读：取出一个新连接
        int clientfd = accept(sockfd,
                              (struct sockaddr *)&clientaddr,
                              &len);

        // clientfd 直接作为数组下标
        fds[clientfd].fd = clientfd;
        fds[clientfd].events = POLLIN;

        if (clientfd > maxfd)
            maxfd = clientfd;
    }
```

例如 `sockfd = 3`、`accept()` 返回 `clientfd = 4`：

```text
fds[3] 管理监听 fd 3：可读时调用 accept()
fds[4] 管理客户端 fd 4：可读时调用 recv()
```

### 11.4 扫描客户端的 revents

```c
    for (int fd = sockfd + 1; fd <= maxfd; ++fd) {
        if (!(fds[fd].revents & POLLIN))
            continue;  // fd 本轮没有发生可读事件

        char buffer[1024] = {0};
        int count = recv(fd, buffer, sizeof(buffer), 0);

        if (count > 0) {
            send(fd, buffer, count, 0);
        } else {
            close(fd);
            fds[fd].fd = -1;  // 让 poll 忽略这个位置
            fds[fd].events = 0;
        }
    }
}
```

poll 版最值得背下来的代码主干是：

```c
fds[clientfd].fd = clientfd;
fds[clientfd].events = POLLIN;

nready = poll(fds, maxfd + 1, -1);

for (fd = sockfd + 1; fd <= maxfd; ++fd) {
    if (fds[fd].revents & POLLIN)
        recv(fd, ...);
}
```

> 记忆关键词：**`events` 是我想观察什么，`revents` 是本轮实际发生了什么。**

---

## 12. epoll 核心代码逐段对照

epoll 的明显区别是：应用不再把完整关注数组传给每一次等待，而是提前通过 `epoll_ctl()` 增量维护关注集合。

### 12.1 创建实例并注册监听 fd

```c
int epfd = epoll_create(1);

struct epoll_event ev;
ev.events = EPOLLIN;
ev.data.fd = sockfd;

epoll_ctl(epfd, EPOLL_CTL_ADD, sockfd, &ev);
```

此处可以理解为：

```text
epoll_create：创建一个事件管理器
epoll_ctl ADD：向管理器登记“关注 sockfd 的可读事件”
```

其中 `ev.data.fd = sockfd` 可以看成给这次登记贴上“sockfd 身份标签”。内核会保存这个标签，等该 fd 就绪时再原样放进返回结果中。

### 12.2 等待并直接遍历就绪数组

```c
while (1) {
    struct epoll_event events[1024] = {0};
    int nready = epoll_wait(epfd, events, 1024, -1);

    for (int i = 0; i < nready; ++i) {
        int fd = events[i].data.fd;
```

这里两个变量要严格分开：

```text
ev       ：调用 epoll_ctl() 时送进去的登记卡
events[] ：调用 epoll_wait() 时拿回来的结果数组
```

例如注册时写入：

```c
ev.data.fd = 3;
epoll_ctl(epfd, EPOLL_CTL_ADD, 3, &ev);
```

当 fd 3 可读时，`epoll_wait()` 可能返回：

```text
nready = 1
events[0].data.fd = 3
events[0].events  = EPOLLIN
```

所以 `events[i].data.fd` 不是凭空出现的，它就是注册时 `ev.data.fd` 中保存的身份标签。

### 12.3 监听 fd 就绪：ADD 新客户端

```c
        if (fd == sockfd) {
            // 监听 fd 可读：从连接队列中取出一个连接
            int clientfd = accept(sockfd,
                                  (struct sockaddr *)&clientaddr,
                                  &len);

            // 给新客户端准备新的登记信息
            ev.events = EPOLLIN;
            ev.data.fd = clientfd;

            epoll_ctl(epfd, EPOLL_CTL_ADD, clientfd, &ev);
            continue;
        }
```

`accept()` 不会删除监听 fd。假设 `sockfd = 3`、它返回 `clientfd = 5`，注册完成后可以这样理解：

```text
fd 3 → 可读时返回标签 3 → 调用 accept()
fd 5 → 可读时返回标签 5 → 调用 recv()
```

### 12.4 客户端就绪：读取或 DEL

```c
        if (events[i].events & EPOLLIN) {
            char buffer[1024] = {0};
            int count = recv(fd, buffer, sizeof(buffer), 0);

            if (count > 0) {
                send(fd, buffer, count, 0);
            } else {
                // 客户端断开：先从 epoll 删除，再关闭 fd
                epoll_ctl(epfd, EPOLL_CTL_DEL, fd, NULL);
                close(fd);
            }
        }
    }
}
```

epoll 版最值得背下来的代码主干是：

```c
epfd = epoll_create(1);

ev.events = EPOLLIN;
ev.data.fd = sockfd;
epoll_ctl(epfd, EPOLL_CTL_ADD, sockfd, &ev);

nready = epoll_wait(epfd, events, 1024, -1);

for (i = 0; i < nready; i++) {
    fd = events[i].data.fd;
    if (fd == sockfd) {
        clientfd = accept(...);
        ev.events = EPOLLIN;
        ev.data.fd = clientfd;
        epoll_ctl(epfd, EPOLL_CTL_ADD, clientfd, &ev);
    } else if (events[i].events & EPOLLIN) {
        recv(fd, ...);
    }
}
```

> 记忆关键词：**`create` 创建、`ctl` 管理、`wait` 等待、只遍历就绪项。**

---

## 13. 把三段代码放在一起看

### 13.1 注册客户端

```c
// select：修改用户态位图
FD_SET(clientfd, &rfds);

// poll：修改用户态结构体数组
fds[clientfd].fd = clientfd;
fds[clientfd].events = POLLIN;

// epoll：通过系统调用修改内核关注集合
ev.events = EPOLLIN;
ev.data.fd = clientfd;
epoll_ctl(epfd, EPOLL_CTL_ADD, clientfd, &ev);
```

### 13.2 等待事件

```c
// select：集合会被改写，所以先复制
rset = rfds;
nready = select(maxfd + 1, &rset, NULL, NULL, NULL);

// poll：传入完整 pollfd 数组
nready = poll(fds, maxfd + 1, -1);

// epoll：关注集合已在内核，只传结果数组
nready = epoll_wait(epfd, events, 1024, -1);
```

### 13.3 寻找就绪客户端

```c
// select：监听 fd 已单独处理，这里遍历客户端 fd
for (fd = sockfd + 1; fd <= maxfd; ++fd)
    if (FD_ISSET(fd, &rset)) { ... }

// poll：遍历完整 pollfd 数组
for (fd = sockfd + 1; fd <= maxfd; ++fd)
    if (fds[fd].revents & POLLIN) { ... }

// epoll：遍历本轮就绪数组
for (i = 0; i < nready; ++i)
    if (events[i].events & EPOLLIN) { ... }
```

### 13.4 删除客户端

```c
// select
FD_CLR(fd, &rfds);
close(fd);

// poll
close(fd);
fds[fd].fd = -1;
fds[fd].events = 0;

// epoll
epoll_ctl(epfd, EPOLL_CTL_DEL, fd, NULL);
close(fd);
```

最核心的差别就体现在“完整集合放在哪里、每轮遍历谁”：

```text
select：应用保存位图 → 每轮复制 → 扫描 0..maxfd
poll：  应用保存数组 → 每轮提交 → 扫描完整数组
epoll： 内核保存关注集合 → 每轮只取出并遍历就绪数组
```

## 总结

**一连接一线程**用线程等待每个连接；**IO 多路复用**让一个线程等待多个 fd；`select` 和 `poll` 每轮仍需处理整个关注集合，而 `epoll` 将关注集合长期维护在内核中并返回当前就绪项；**Reactor**再进一步把这些就绪事件分派给预先注册的回调函数，使服务器从“管理 fd”上升为“管理事件及其处理逻辑”。
