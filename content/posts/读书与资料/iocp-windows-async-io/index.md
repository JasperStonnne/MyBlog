---
title: "IOCP：Windows 异步机制"
slug: iocp-windows-async-io
description: 从阻塞 IO 与 Reactor 出发，梳理 Windows IOCP 的完成通知、线程池、OVERLAPPED、重叠 IO 与连接断开判断
date: 2026-08-20T00:00:00+08:00
draft: false
image: cover.svg
tags:
  - IOCP
  - Windows
  - 网络编程
  - 异步 IO
  - Proactor
categories:
  - 后端开发
---

## 一、网络编程要解决什么问题

在 C/S 模式中，服务端通常先执行：

```text
socket -> bind -> listen
```

客户端也会先创建 socket。各种网络模型前面的准备过程基本相同，主要区别在后续如何处理连接和 I/O。

网络编程中有四件无法预先确定的事情：

1. 客户端什么时候建立连接；
2. 客户端什么时候断开连接；
3. 客户端什么时候发送数据；
4. 数据什么时候能够写入发送缓冲区。

因此，网络编程主要解决四类问题：

1. 连接的建立；
2. 连接的断开；
3. 数据的接收；
4. 数据的发送。

每个 socket 都对应接收缓冲区（receive buffer）和发送缓冲区（send buffer）。`write` 只负责把数据从用户态复制到内核态的发送缓冲区；数据什么时候到达对端、如何到达对端，属于网络协议栈要解决的问题。

常见的网络模型包括：

- 阻塞 I/O 模型；
- Reactor；
- Proactor（IOCP）。

## 二、阻塞 I/O 与 Reactor

一次 I/O 可以分成两个阶段：

1. **I/O 检测**：判断 I/O 是否就绪；
2. **I/O 操作**：真正执行 `accept`、`read`、`write`、`connect` 等操作。

### 1. 阻塞 I/O

阻塞 I/O 通过阻塞当前线程来等待 I/O 就绪，从而解决“不知道事件什么时候发生”的问题。

阻塞和非阻塞主要描述 I/O 检测阶段：

- 阻塞：I/O 没有就绪时，线程会等待；
- 非阻塞：I/O 没有就绪时，函数立即返回。

### 2. Reactor

Reactor 使用：

```text
I/O 多路复用（检测 I/O）+ 非阻塞 I/O（操作 I/O）
```

它先注册事件。当事件触发时，系统发出就绪通知，用户层收到通知后，再调用 POSIX I/O 函数完成操作。因此，Reactor 属于：

```text
同步 I/O + 异步事件通知
```

`select`、`poll`、`epoll` 用于同时检测多路 I/O 是否就绪，它们只负责 I/O 检测；Reactor 则包含 I/O 检测和后续的 I/O 操作。

所以它们的关系是：

```text
Reactor = select/poll/epoll 等 I/O 多路复用 + 非阻塞 I/O 操作
```

## 三、IOCP 与 Reactor 的区别

IOCP（I/O Completion Port，I/O 完成端口）是 Windows 提供的一种高效异步 I/O 机制。

Reactor 的过程：

```text
注册事件
    ↓
内核检测到 I/O 就绪
    ↓
用户层收到就绪通知
    ↓
用户层调用 accept/read/write 等函数完成 I/O
```

这是一种**就绪通知**。

IOCP 的过程：

```text
用户层投递异步 I/O 请求
    ↓
内核检测并完成 I/O
    ↓
用户层收到完成通知
```

因此：

- Reactor：同步 I/O、异步事件；
- IOCP：异步 I/O、异步事件。

两者都是为了解决网络 I/O 问题，因此 IOCP 在整体概念上更接近 Reactor；但 IOCP 并不是 `select`、`poll`、`epoll` 这样的 I/O 多路复用机制。

在 Reactor 中，即使 socket 已经就绪，真正的数据复制仍由用户线程调用 I/O 函数完成。IOCP 则由用户层发起异步操作，由内核完成 I/O；在此期间，用户线程可以处理其他任务。

## 四、IOCP 的工作原理

网络 I/O 属于外设资源，相关数据需要通过系统调用，由内核进行操作。

IOCP 的基本流程如下：

1. 使用 `CreateIoCompletionPort` 创建完成端口；
2. 将 socket 与 IOCP 绑定；
3. 投递异步 I/O 操作；
4. 内核进行 I/O 检测和 I/O 操作；
5. 操作完成后，内核把完成信息放入完成队列；
6. 用户层通过 `GetQueuedCompletionStatus` 取得完成通知。

常见的异步操作包括：

- `AcceptEx`：接受连接；
- `ConnectEx`：主动建立连接；
- `DisconnectEx`：主动断开连接；
- `WSARecv`：接收数据；
- `WSASend`：发送数据。

例如建立连接时：

```text
socket 绑定 IOCP
    ↓
投递 AcceptEx
    ↓
内核完成连接接收
    ↓
GetQueuedCompletionStatus 取得完成通知
```

与传统 `accept` 不同，`AcceptEx` 是预先投递的异步请求。服务器可以在启动时投递多个 `AcceptEx`，从而并发接收多条连接。

### IOCP 与线程池

IOCP 通常会绑定一个线程池。

当线程调用 `GetQueuedCompletionStatus` 时，该线程会与 IOCP 产生关联。线程退出或关闭后，关联取消；一个线程只能关联一个 IOCP。

`GetQueuedCompletionStatus` 是阻塞接口：

```text
等待完成队列出现数据
    ↓
取得完成状态
    ↓
在用户态处理完成事件
```

最后可以使用 `CloseHandle` 关闭 I/O 完成端口句柄。

## 五、完成事件如何与连接关联

收到完成通知后，需要知道是哪一条连接、哪一次操作完成了。

### 1. Reactor 中的关联方式

以 `epoll` 为例，注册事件时可以把一个值交给内核保存：

```cpp
ev.data.fd = fd;
```

也可以关联用户态指针：

```cpp
ev.data.ptr = user_data;
```

`epoll_wait` 返回事件时会带回这个值，因此可以判断是哪一条连接发生了事件。

### 2. IOCP 中的关联方式

IOCP 有两种关联信息。

第一种是 `CreateIoCompletionPort` 的第三个参数 `CompletionKey`。它可以保存一个用户态指针，并在其中记录 socket 等连接信息。

第二种是异步函数最后传入的 `OVERLAPPED` 指针。可以把 `OVERLAPPED` 放进自定义结构体：

```cpp
struct OverlappedPerIO {
    OVERLAPPED overlapped;
    SOCKET socket;
    // 其他与本次 I/O 相关的数据
};
```

内核使用其中的 `OVERLAPPED`，用户层则可以在结构体中附加需要关联的数据。

可以这样理解：

- `CompletionKey`：关联连接级信息；
- `OVERLAPPED*`：关联某一次 I/O 操作的信息。

## 六、重叠 I/O

异步 I/O 是投递请求后无需原地等待，可以由其他线程等待完成通知。

重叠 I/O 是指：无需等待上一个 I/O 完成，就可以继续投递下一个操作，让多个操作同时处于进行状态。

例如，如果每次只投递一个 `AcceptEx`，完成一次后再投递下一次，那么客户端较多时，建立连接的效率会比较低。服务器可以在启动时投递多个 `AcceptEx`：

```text
AcceptEx 请求 1 ─┐
AcceptEx 请求 2 ─┼─→ 同时等待多个客户端连接
AcceptEx 请求 3 ─┘
```

每个重叠 I/O 都使用一个独立的 `OVERLAPPED` 结构。操作完成后，`GetQueuedCompletionStatus` 会返回对应的 `OVERLAPPED*`，用户层据此找到这次操作的上下文。

## 七、连接断开的判断

客户端主动断开连接时，可以通过以下方式判断。

### Reactor

1. `epoll_wait` 返回 `EPOLLHUP` 或 `EPOLLRDHUP`；
2. `read` / `recv` 返回 `0`。

### IOCP

1. `GetLastError` 返回相关错误，例如异常断开时的 `ERROR_NETNAME_DELETED`；
2. `GetQueuedCompletionStatus` 返回的完成字节数 `dwBytes == 0`。

超时情况下还可能出现 `WAIT_TIMEOUT`。

## 八、投递操作

- `AcceptEx` 可以多次投递，以并发接收连接；
- `WSARecv` 涉及接收顺序和线程安全问题；
- `WSASend` 可以多次投递。
