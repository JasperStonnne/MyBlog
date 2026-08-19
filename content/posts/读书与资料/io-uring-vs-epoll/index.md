---
title: io_uring（一）：与 epoll 一较高下
slug: io-uring-vs-epoll
description: 从代码执行路径出发，对照 epoll 的就绪通知与 io_uring 的完成通知，梳理 SQE、CQE、Echo Server 事件循环及 Reactor 与 Proactor 的核心区别
date: 2026-08-19T00:00:00+08:00
draft: false
image: cover.svg
tags:
  - io_uring
  - epoll
  - Linux
  - 网络编程
  - 异步 IO
categories:
  - 后端开发
---

> 核心目标：能从代码上说清楚 `epoll` 和 `io_uring` 的区别，而不是只记 API。

## 1. 今天最重要的结论

一句话区分：

- **Reactor / epoll：内核通知“现在可以做 I/O 了”，真正的 `accept/recv/send` 由应用执行。**
- **Proactor / io_uring：应用先提交 I/O 操作，内核完成后通知“这个操作已经做完了”。**

最值得记住的一组对照：

```text
epoll 的 EPOLLIN       = 可以读了，但数据还没被应用读出
io_uring 的 READ CQE   = 读操作已结束，结果在 cqe->res，数据已进入指定 buffer

epoll 的 EPOLLOUT      = 现在可以写，但应用还要调用 send/write
io_uring 的 WRITE CQE  = 写操作已结束，cqe->res 表示实际写出的字节数
```

因此，二者虽然都有“注册/提交 → 等待 → 处理事件”的循环，但事件的含义完全不同：

- epoll 返回的是**就绪事件**。
- io_uring 返回的是**完成事件**。

---

## 2. “异步”与“异步 I/O”不是一回事

### 2.1 一般意义上的异步

发起任务后，不原地等待它结束，当前执行流可以继续做别的事；任务完成后再通过回调、消息、事件或协程恢复等方式取得结果。

例如：把日志写盘任务放入线程池。业务线程只投递任务，工作线程实际调用同步 `write()`。从业务流程看它是异步的，但底层 `write()` 本身仍可能是同步 I/O。

### 2.2 同步 I/O

普通的：

```c
read(fd, buffer, length);
write(fd, buffer, length);
recv(fd, buffer, length, 0);
send(fd, buffer, length, 0);
```

一次函数调用既发起操作，也返回该次操作的结果。即使 fd 被设为非阻塞，调用者仍然要亲自执行 `recv/send`；“非阻塞”不自动等于“异步 I/O”。

### 2.3 epoll 做到的是什么

`epoll_wait()` 可以高效等待大量 fd 的**就绪状态**，避免逐个 fd 轮询。它通常与非阻塞 socket 配合，构成 Reactor。

但 `epoll` 不替应用搬运数据：收到 `EPOLLIN` 后，代码仍要调用 `recv()`；收到 `EPOLLOUT` 后，仍要调用 `send()`。

### 2.4 io_uring 做到的是什么

应用把“请替我执行 accept/read/write……”描述成 SQE，提交给内核；操作完成后，内核写入 CQE。应用处理 CQE 时，看到的是操作结果，而不只是“可以开始操作”。

---

## 3. io_uring 的两个环

```text
应用准备 SQE → SQ（Submission Queue）→ 内核执行 I/O
                                           ↓
应用处理结果 ← CQ（Completion Queue）← 内核生成 CQE
```

### SQ / SQE

- SQ：提交队列。
- SQE：一次待执行 I/O 的描述，例如操作类型、fd、buffer、长度和用户数据。
- `io_uring_get_sqe()`：取得一个可填写的 SQE。
- `io_uring_prep_accept/read/write(...)`：只是在填写 SQE，**不会立刻执行 I/O**。
- `io_uring_submit()`：把准备好的请求提交给内核。

### CQ / CQE

- CQ：完成队列。
- CQE：一个已经完成的 I/O 结果。
- `cqe->res`：操作结果。成功时通常是新 fd 或字节数；失败时是负的 errno 值。
- `cqe->user_data`：应用提交时附带的标识，用来判断这是哪个连接、哪类操作。
- `io_uring_cq_advance()`：告诉 ring，这批 CQE 已处理，可以回收。

### 为什么使用环形队列和 mmap

环形队列可以复用固定槽位，减少频繁分配；SQ/CQ 通过 `mmap` 在用户态和内核态之间共享队列元数据，减少提交、取结果时的额外系统调用和元数据复制。

但要注意：这不代表所有 I/O 数据都天然“零拷贝”。例如普通 socket 接收的数据通常仍需进入应用提供的 buffer。

---

## 4. io_uring 三个底层系统调用

- `io_uring_setup`：创建并配置 ring，建立 SQ/CQ 所需资源。
- `io_uring_enter`：提交请求和/或等待完成事件。
- `io_uring_register`：预注册文件、buffer、事件等资源，减少热路径开销；它不是每个普通请求都必须调用。

课堂代码使用的是 `liburing` 封装：

```c
io_uring_queue_init_params(ENTRIES_LENGTH, &ring, &params);
io_uring_get_sqe(&ring);
io_uring_prep_accept(...);
io_uring_submit(&ring);
io_uring_wait_cqe(&ring, &cqe);
```

可近似理解为：初始化 ring → 取得请求槽位 → 描述操作 → 提交 → 等结果。

---

## 5. 你的 io_uring Echo Server 怎么走

关键入口：

```c
set_event_accept(&ring,sockfd,(struct sockaddr*)&clientaddr,&len,0);
```

`set_event_accept()` 做两件事：

1. 用 `io_uring_get_sqe()` 取得 SQE。
2. 用 `io_uring_prep_accept()` 填写 accept 请求，并在 `user_data` 中记录 `fd + EVENT_ACCEPT`。

此时只是**准备请求**，并没有等待客户端。

主循环：

```text
io_uring_submit
    ↓ 提交当前准备好的 accept/recv/send 请求
io_uring_wait_cqe
    ↓ 没有完成结果时在这里等待
io_uring_peek_batch_cqe
    ↓ 批量取得已完成操作
根据 user_data 判断 EVENT_ACCEPT
    ↓
再次 set_event_accept，同时为新 clientfd 提交 recv
    ↓
io_uring_cq_advance
```

你测试时发现程序阻塞在 `io_uring_wait_cqe()`，这是合理的：

- `io_uring_prep_accept()` 只准备请求，很快返回。
- `io_uring_submit()` 提交请求。
- 在客户端真正连接前，accept 尚未完成，CQ 中没有结果，所以 `wait_cqe()` 等待。
- 客户端连接后，CQE 到达，`cqe->res` 应是新连接的 client fd（失败则为负数）。

### 5.1 三种事件不是“就绪事件”，而是操作身份

```c
#define EVENT_ACCEPT 0
#define EVENT_READ 1
#define EVENT_WRITE 2
```

这里的 `EVENT_*` 是应用自己定义的标签。准备 SQE 时，代码把 `fd + event` 写入 `sqe->user_data`；操作完成后，再从 CQE 的 `user_data` 还原标签，据此判断完成的是 accept、recv 还是 send。

三种 SQE 的含义：

| 函数 | 提交给内核的操作 | CQE 到达时 `entries->res` 的含义 |
|---|---|---|
| `set_event_accept` | 接收一个新连接 | 新连接 fd；负数表示失败 |
| `set_event_recv` | 把数据读进指定 buffer | 实际读取字节数；0 表示对端关闭；负数表示失败 |
| `set_event_send` | 发送指定 buffer 中的数据 | 实际发送字节数；负数表示失败 |

### 5.2 完整状态循环

```text
启动：准备 ACCEPT SQE
          ↓ submit
      ACCEPT 完成
          ├─ entries->res 得到 connfd
          ├─ 再准备一个 ACCEPT，继续接收其他客户端
          └─ 为 connfd 准备 RECV
                         ↓ submit
                     READ 完成
          ├─ res == 0：对端断开，close(fd)
          └─ res > 0：buffer 中已有数据，准备 SEND
                                             ↓ submit
                                         WRITE 完成
                                             ↓
                                    再为该 fd 准备 RECV
```

对单个连接而言，核心循环就是：

```text
RECV 完成 → SEND 完成 → 再次 RECV
```

这份代码没有额外解析数据，而是把收到的 `buffer` 原样发送回去，因此它是一个 io_uring Echo Server。

### 5.3 为什么循环开头的 submit 能提交刚刚准备的操作

在处理 CQE 时调用的 `set_event_accept/recv/send()` 都只是取得并填写新的 SQE。它们不会立刻执行；本轮处理结束并 `cq_advance` 后，程序回到 `while` 开头，由下一次 `io_uring_submit()` 统一提交。

因此时间顺序是：

```text
本轮处理 CQE → 准备下一批 SQE → advance → 下一轮 submit
```

这种批量准备、批量提交正是 io_uring 的重要思路。

### 为什么处理后要再次设置 accept

普通的 `io_uring_prep_accept()` 描述的是**一次 accept 操作**。完成一次就消费掉一次请求；若想继续接收连接，需要再准备并提交新的 accept。

这与 epoll 的常规 LT 模式不同：监听 fd 注册 `EPOLLIN` 后，只要仍处于可读状态，epoll 可以继续报告就绪，无需每完成一次 accept 就重新 `EPOLL_CTL_ADD`。

### 为什么必须 advance

CQE 被读取不等于被消费：

```c
io_uring_cq_advance(&ring,nready);
```

这一步更新 CQ 的消费位置。漏掉它，旧 CQE 会一直被当成尚未处理，表现为循环反复看到同一结果。

---

## 6. 你的 Reactor 代码怎么走

### 6.1 初始化

```text
epoll_create
    ↓
创建 20 个监听 socket（端口 2000～2019）
    ↓
把监听 fd 以 EPOLLIN 加入 epoll
    ↓
进入 epoll_wait 主循环
```

监听 fd 的回调被设置为 `accept_cb`：

```c
conn_list[sockfd].r_action.recv_callback = accept_cb;
```

### 6.2 建立连接

```text
监听 fd 出现 EPOLLIN
    ↓
主循环调用 accept_cb
    ↓
accept_cb 主动执行 accept()
    ↓
event_register(clientfd, EPOLLIN)
```

关键点：`EPOLLIN` 只表示监听 socket 可以执行 accept；真正建立并取出 client fd 的动作仍由 `accept()` 完成。

### 6.3 接收数据

```text
clientfd 出现 EPOLLIN
    ↓
调用 recv_cb
    ↓
recv() 把数据读进 rbuffer
    ↓
ws_request() 处理请求
    ↓
把关注事件修改为 EPOLLOUT
```

如果 `recv()` 返回 0，说明对端正常断开；返回负数则表示错误。当前代码随后关闭 fd 并从 epoll 删除。

### 6.4 发送数据

```text
clientfd 出现 EPOLLOUT
    ↓
调用 send_cb
    ↓
ws_response() 生成响应
    ↓
send() 主动发送 wbuffer
    ↓
把关注事件改回 EPOLLIN
```

整个连接状态近似为：

```text
等待读 EPOLLIN → recv + 处理请求 → 等待写 EPOLLOUT
       ↑                                  ↓
       └──────── send + 改回读 ──────────┘
```

---

## 7. 逐段对照：Reactor 与 io_uring

| 阶段 | epoll / Reactor | io_uring / Proactor 思路 |
|---|---|---|
| 接收连接 | 注册监听 fd 的 `EPOLLIN` | 提交一个 accept SQE |
| 通知到达 | “监听 fd 可以 accept” | “accept 已完成” |
| 新连接 fd | 应用调用 `accept()` 得到 | 从 accept CQE 的 `res` 得到 |
| 接收数据 | `EPOLLIN` 后应用调用 `recv()` | 预先提交 recv/read SQE，完成后 buffer 已有数据 |
| 发送数据 | `EPOLLOUT` 后应用调用 `send()` | 预先提交 send/write SQE，完成后得到实际写入量 |
| 事件身份 | `events[i].data.fd` + 回调表 | `cqe->user_data` 标识 fd/事件/上下文 |
| 一次操作结束 | 修改下一次关注的就绪事件 | 消费 CQE，再准备下一次 I/O SQE |
| 批量处理 | `epoll_wait(..., events, 1024, ...)` | `io_uring_peek_batch_cqe(..., cqes, 128)` |

代码上的一一对应：

```text
epoll_ctl ADD/MOD             ↔ io_uring_get_sqe + prep_xxx
epoll_wait                    ↔ io_uring_wait_cqe / peek_batch_cqe
events[i].data.fd             ↔ cqe->user_data
EPOLLIN 后 recv               ↔ READ CQE 到达时数据已读完
EPOLLOUT 后 send              ↔ WRITE CQE 到达时写操作已完成
下一轮 set_event              ↔ 下一轮重新准备 SQE
                              ↔ io_uring_cq_advance 消费完成项
```

---

## 8. 对学习笔记中三个说法的校正

### ① “SQE 与 CQE 是同一个节点，共用一块内存”——不准确

SQE 和 CQE 是不同队列中的不同结构：SQE 描述请求，CQE 描述结果。它们通过 `user_data` 关联。共享内存指的是这些 ring 能被用户态与内核态共同访问，不是 SQE 原地变成 CQE。

### ② “io_uring_prep_accept 底层调用 register”——不准确

`io_uring_prep_accept()` 主要是填写 SQE。`io_uring_register` 是独立的资源注册系统调用，普通 accept 请求不要求每次 register。

### ③ “mmap 后就没有 copy”——范围过大

`mmap` 主要减少 SQ/CQ 控制信息在用户态和内核态之间的复制及系统调用开销；普通网络 I/O 的 payload 是否复制，是另一层问题。

---

## 9. 最小记忆框架

只背下面四句话：

1. `epoll` 给我的是**就绪**，我还要自己 `accept/recv/send`。
2. `io_uring` 给我的是**完成**，结果看 `cqe->res`，身份看 `cqe->user_data`。
3. io_uring 的路径是：**取 SQE → prep → submit → 等 CQE → 处理 → advance**。
4. 普通 io_uring 请求通常是一次性的：**完成一次，消费一次，再提交下一次**。
