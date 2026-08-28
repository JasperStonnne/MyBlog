---
title: KV 存储项目网络层：统一接入 Reactor、Proactor 与协程
slug: kv-store-network-layer
description: 从代码调用链出发，梳理 KV 协议层如何通过统一回调接入 epoll Reactor、io_uring Proactor 与 ntyco 协程网络层
date: 2026-08-22T00:00:00+08:00
draft: false
image: cover.svg
tags:
  - KV 存储
  - 网络编程
  - Reactor
  - io_uring
  - Proactor
  - 协程
  - C 语言
categories:
  - 项目记录
---

> 阅读目标：先恢复项目的整体认识，再通过带注释的代码块回忆具体实现。
>
> 本文只解析现有成品代码，不讨论修改方案。ntyco 只讲如何作为 KV 项目的网络层使用。

---

## 1. 先恢复项目的整体画面

这个项目的核心不是分别写了三个独立服务器，而是让三个网络服务器共同使用同一个 KV 协议入口：

```text
                    kvstore.c
             main + kvs_protocol
                       │
                msg_handler 回调
                       │
        ┌──────────────┼──────────────┐
        │              │              │
   reactor.c        ntyco.c       proactor.c
     epoll            协程          io_uring
        │              │              │
        └──────────────┼──────────────┘
                       │
                      TCP
```

三种网络层只负责连接和收发，`kvs_protocol()` 负责处理消息：

```text
客户端请求
   ↓
网络层收到 msg 和 length
   ↓
kvs_protocol(msg, length, response)
   ↓
返回 response 的长度
   ↓
网络层发送 response
```

项目目前的 `kvs_protocol()` 是 echo 协议：把请求复制成响应。这一实现展示的重点，是三种网络模型如何接入同一个 KV 协议回调。

六份文件分成三组：

```text
公共入口与接口
  ├─ kvstore.c
  └─ kvstore.h

Reactor 公共连接定义
  └─ server.h

三种网络层
  ├─ reactor.c
  ├─ proactor.c
  └─ ntyco.c
```

---

# 2. 公共入口：`kvstore.h` 与 `kvstore.c`

## 2.1 `kvstore.h`：把三种网络层统一成一种调用方式

```c
#ifndef KV_STORE_H
#define KV_STORE_H

/* 三种网络实现的编号，供 NETWORK_SELECT 选择 */
#define NETWORK_REACTOR  0
#define NETWORK_PROACTOR 1
#define NET_WORK_NTYCO   2

/* 当前编译使用 io_uring Proactor 网络层 */
#define NETWORK_SELECT NETWORK_PROACTOR

/*
 * KV 协议回调类型：
 * msg      网络层收到的请求
 * length   请求的实际长度
 * response 协议层写入响应的位置
 * 返回值   网络层需要发送的响应长度
 */
typedef int (*msg_handler)(char *msg, int length, char *response);

/* 三种网络层向 kvstore.c 暴露相同形式的启动函数 */
extern int reactor_start(unsigned short port, msg_handler handler);
extern int ntyco_start(unsigned short port, msg_handler handler);
extern int proactor_start(unsigned short port, msg_handler handler);

/* KV 项目支持的命令名称 */
const char *command[] = {
    "SET", "GET", "DEL", "MOD", "EXIST"
};

/* 响应字符串表，目前未填充 */
const char *response[] = {
};

#endif
```

这一文件的整体作用可以概括为：

```text
NETWORK_SELECT 决定使用谁
msg_handler    规定协议函数长什么样
三个 start     规定网络层怎样启动
command        列出 KV 命令集合
```

## 2.2 `kvstore.c`：选择网络层，并把协议函数交给它

```c
#include <stdio.h>
#include <string.h>
#include <stdlib.h>
#include "kvstore.h"

/*
 * 当前是 echo 形式的 KV 协议入口。
 * 网络层把请求传进来，本函数把请求复制成响应。
 */
int kvs_protocol(char *msg, int length, char *response) {
    /* 输出本次请求长度与内容 */
    printf("recv: %d: %s\n", length, msg);

    /* 将收到的 length 个字节复制到响应缓冲区 */
    memcpy(response, msg, length);

    /* 把响应长度交还给网络层 */
    return strlen(response);
}

int main(int argc, char *argv[]) {
    /* 启动形式要求为：./kvstore <port> */
    if (argc != 2) return -1;

    /* 将命令行中的端口字符串转换成整数 */
    int port = atoi(argv[1]);

    /*
     * 编译期选择网络模型。
     * 三个分支都把同一个 kvs_protocol 作为 handler 传入。
     */
#if (NETWORK_SELECT == NETWORK_REACTOR)
    reactor_start(port, kvs_protocol);
#elif (NETWORK_SELECT == NETWORK_NTYCO)
    ntyco_start(port, kvs_protocol);
#elif (NETWORK_SELECT == NETWORK_PROACTOR)
    proactor_start(port, kvs_protocol);
#endif
}
```

这里最需要记住的不是条件编译本身，而是下面三个调用具有完全相同的含义：

```c
reactor_start(port, kvs_protocol);
ntyco_start(port, kvs_protocol);
proactor_start(port, kvs_protocol);
```

可以统一读作：

> 在指定端口启动某个网络后端；以后收到消息时，请调用 `kvs_protocol()`。

---

# 3. Reactor 的连接结构：`server.h`

Reactor 是事件驱动的。一次读事件和下一次写事件不是在同一个函数中连续完成，因此需要 `struct conn` 保存跨事件存在的连接信息。

```c
#ifndef SERVER_H
#define SERVER_H

/* 每个连接的读、写缓冲区都是 1024 字节 */
#define BUFFER_LENGTH 1024

/* 当前 Reactor 上层启用 KVSTORE 协议 */
#define ENABLE_HTTP      0
#define ENABLE_WEBSOCKET 0
#define ENABLE_KVSTORE   1

/* epoll 事件回调：传入发生事件的 fd */
typedef int (*RCALLBACK)(int fd);

struct conn {
    int fd;                         // 当前连接的 socket fd

    char rbuffer[BUFFER_LENGTH];    // recv 接收到的请求
    int rlength;                    // 请求的实际长度

    char wbuffer[BUFFER_LENGTH];    // KV 协议生成的响应
    int wlength;                    // 响应的实际长度

    RCALLBACK send_callback;        // EPOLLOUT 时调用 send_cb

    union {
        RCALLBACK recv_callback;    // 客户端 fd 可读时调用 recv_cb
        RCALLBACK accept_callback;  // 监听 fd 可读时调用 accept_cb
    } r_action;

    int status;                     // 连接/协议状态字段

#if 1 // websocket
    char *payload;                  // WebSocket 数据字段
    char mask[4];                   // WebSocket 掩码字段
#endif
};

/* 同一套 Reactor 曾为不同上层协议保留入口 */
#if ENABLE_HTTP
int http_request(struct conn *c);
int http_response(struct conn *c);
#endif

#if ENABLE_WEBSOCKET
int ws_request(struct conn *c);
int ws_response(struct conn *c);
#endif

#if ENABLE_KVSTORE
int kvs_request(struct conn *c);
int kvs_response(struct conn *c);
#endif

#endif
```

`struct conn` 可以用一条数据流来理解：

```text
recv(fd)
  ↓
rbuffer + rlength
  ↓ kvs_handler
wbuffer + wlength
  ↓
send(fd)
```

两个函数指针则决定 epoll 返回事件时执行什么：

```text
监听 fd 的 EPOLLIN  → accept_cb
连接 fd的 EPOLLIN   → recv_cb
连接 fd的 EPOLLOUT  → send_cb
```

---

# 4. Reactor 网络层：`reactor.c`

## 4.1 Reactor 总体流程

```text
创建 epoll
  ↓
创建 20 个监听 socket，并注册 EPOLLIN
  ↓
epoll_wait
  ├─ 监听 fd 可读   → accept_cb
  ├─ 客户端 fd 可读 → recv_cb → kvs_protocol
  └─ 客户端 fd 可写 → send_cb
```

## 4.2 KV 回调和连接表

```c
#define CONNECTION_SIZE 1048576   // 连接表容量
#define MAX_PORTS 20               // 连续监听 20 个端口

/* 计算两次 timeval 之间的毫秒差，用于 accept 统计 */
#define TIME_SUB_MS(tv1, tv2) \
    ((tv1.tv_sec - tv2.tv_sec) * 1000 + \
     (tv1.tv_usec - tv2.tv_usec) / 1000)

#if ENABLE_KVSTORE
/* reactor_start 会把 kvs_protocol 保存到这里 */
static msg_handler kvs_handler;

int kvs_request(struct conn *c) {
    /*
     * 将连接的接收缓冲区交给 KV 层；
     * KV 层把响应写入同一连接的发送缓冲区；
     * 返回值记录到 wlength。
     */
    c->wlength = kvs_handler(
        c->rbuffer,
        c->rlength,
        c->wbuffer
    );
}

int kvs_response(struct conn *c) {
    /* 当前 KV 响应不再进行二次处理 */
}
#endif

int epfd = 0;                              // epoll 实例 fd
struct timeval begin;                      // accept 统计起点
struct conn conn_list[CONNECTION_SIZE];    // 用 fd 直接索引连接
```

## 4.3 添加和切换 epoll 事件

```c
int set_event(int fd, int event, int flag) {
    struct epoll_event ev;
    ev.events = event;     // EPOLLIN 或 EPOLLOUT
    ev.data.fd = fd;       // epoll 返回事件时带回这个 fd

    if (flag) {
        /* 新 fd 第一次进入 epoll */
        epoll_ctl(epfd, EPOLL_CTL_ADD, fd, &ev);
    } else {
        /* 已登记 fd 在读、写关注之间切换 */
        epoll_ctl(epfd, EPOLL_CTL_MOD, fd, &ev);
    }
}
```

这份 Reactor 的主要节奏，就是不断切换同一个客户端的事件：

```text
刚建立连接       → EPOLLIN
收到并处理请求   → EPOLLOUT
发送完响应       → EPOLLIN
```

## 4.4 初始化一个客户端连接

```c
int event_register(int fd, int event) {
    if (fd < 0) return -1;

    /* fd 同时也是 conn_list 的下标 */
    conn_list[fd].fd = fd;

    /* 客户端可读和可写时对应的处理函数 */
    conn_list[fd].r_action.recv_callback = recv_cb;
    conn_list[fd].send_callback = send_cb;

    /* 初始化这个连接的输入区域 */
    memset(conn_list[fd].rbuffer, 0, BUFFER_LENGTH);
    conn_list[fd].rlength = 0;

    /* 初始化这个连接的输出区域 */
    memset(conn_list[fd].wbuffer, 0, BUFFER_LENGTH);
    conn_list[fd].wlength = 0;

    /* 新连接加入 epoll，调用处传入的是 EPOLLIN */
    set_event(fd, event, 1);
}
```

## 4.5 创建监听 socket

```c
int r_init_server(unsigned short port) {
    /* 创建 IPv4 TCP socket */
    int sockfd = socket(AF_INET, SOCK_STREAM, 0);

    struct sockaddr_in servaddr;
    servaddr.sin_family = AF_INET;
    servaddr.sin_addr.s_addr = htonl(INADDR_ANY); // 0.0.0.0
    servaddr.sin_port = htons(port);              // 监听端口

    /* 把 socket 绑定到本机端口 */
    if (bind(sockfd,
             (struct sockaddr *)&servaddr,
             sizeof(struct sockaddr)) == -1) {
        printf("bind failed: %s\n", strerror(errno));
    }

    /* 将 socket 变为监听 socket */
    listen(sockfd, 10);

    return sockfd;
}
```

## 4.6 接受客户端

```c
int accept_cb(int fd) {
    struct sockaddr_in clientaddr;
    socklen_t len = sizeof(clientaddr);

    /*
     * fd 是监听 socket；
     * clientfd 是新建立的客户端连接。
     */
    int clientfd = accept(
        fd,
        (struct sockaddr *)&clientaddr,
        &len
    );

    printf("accept finshed: %d,fd:%d\n", clientfd, fd);

    if (clientfd < 0) {
        printf("accept errno: %d --> %s\n",
               errno, strerror(errno));
        return -1;
    }

    /* 新客户端从等待读事件开始 */
    event_register(clientfd, EPOLLIN);

    /* 每当 fd 到达 1000 的整数倍，打印一次时间间隔 */
    if ((clientfd % 1000) == 0) {
        struct timeval current;
        gettimeofday(&current, NULL);

        int time_used = TIME_SUB_MS(current, begin);
        memcpy(&begin, &current, sizeof(struct timeval));

        printf("accept finshed: %d, time_used: %d\n",
               clientfd, time_used);
    }

    return 0;
}
```

## 4.7 接收请求并进入 KV 协议

```c
int recv_cb(int fd) {
    /* 本次接收前清空该连接的读缓冲区 */
    memset(conn_list[fd].rbuffer, 0, BUFFER_LENGTH);

    /* 从客户端接收数据 */
    int count = recv(
        fd,
        conn_list[fd].rbuffer,
        BUFFER_LENGTH,
        0
    );

    if (count == 0) {
        /* recv 返回 0：客户端断开 */
        printf("client disconnect: %d\n", fd);
        close(fd);
        epoll_ctl(epfd, EPOLL_CTL_DEL, fd, NULL);
        return 0;
    } else if (count < 0) {
        /* recv 返回负数：打印错误并结束连接 */
        printf("count: %d, errno: %d, %s\n",
               count, errno, strerror(errno));
        close(fd);
        epoll_ctl(epfd, EPOLL_CTL_DEL, fd, NULL);
        return 0;
    }

    /* 保存这次实际收到的请求长度 */
    conn_list[fd].rlength = count;

#if 0
    /* echo 分支：直接把读缓冲复制到写缓冲 */
    conn_list[fd].wlength = conn_list[fd].rlength;
    memcpy(conn_list[fd].wbuffer,
           conn_list[fd].rbuffer,
           conn_list[fd].wlength);
#elif ENABLE_HTTP
    http_request(&conn_list[fd]);
#elif ENABLE_WEBSOCKET
    ws_request(&conn_list[fd]);
#elif ENABLE_KVSTORE
    /* 当前启用的分支：通过 handler 生成 KV 响应 */
    kvs_request(&conn_list[fd]);
#endif

    /* 请求处理完成，下一阶段等待发送 */
    set_event(fd, EPOLLOUT, 0);
    return count;
}
```

这里完成了项目中最重要的一次跨层调用：

```text
recv_cb
  → kvs_request(&conn_list[fd])
      → kvs_handler(rbuffer, rlength, wbuffer)
          → kvs_protocol
```

## 4.8 发送响应

```c
int send_cb(int fd) {
#if ENABLE_HTTP
    http_response(&conn_list[fd]);
#elif ENABLE_WEBSOCKET
    ws_response(&conn_list[fd]);
#elif ENABLE_KVSTORE
    /* KV 分支当前不对 wbuffer 做额外处理 */
    kvs_response(&conn_list[fd]);
#endif

    int count = 0;

    /* wlength 非零时，把 KV 响应发给客户端 */
    if (conn_list[fd].wlength != 0) {
        count = send(
            fd,
            conn_list[fd].wbuffer,
            conn_list[fd].wlength,
            0
        );
    }

    /* 响应阶段结束，再次等待客户端请求 */
    set_event(fd, EPOLLIN, 0);
    return count;
}
```

## 4.9 启动 Reactor 主循环

```c
int reactor_start(unsigned short port, msg_handler handler) {
    printf("reactor_entry %d\n", port);

    /* 保存由 main 传入的 kvs_protocol */
    kvs_handler = handler;

    /* 创建 epoll 实例 */
    epfd = epoll_create(1);

    /* 从 port 开始连续建立 20 个监听端口 */
    for (int i = 0; i < MAX_PORTS; i++) {
        int sockfd = r_init_server(port + i);

        /* 监听 fd 可读时执行 accept_cb */
        conn_list[sockfd].fd = sockfd;
        conn_list[sockfd].r_action.recv_callback = accept_cb;

        /* 将监听 fd 加入 epoll */
        set_event(sockfd, EPOLLIN, 1);
    }

    /* accept 性能统计的起始时间 */
    gettimeofday(&begin, NULL);

    while (1) {
        struct epoll_event events[1024] = {0};

        /* 阻塞等待，最多取得 1024 个就绪事件 */
        int nready = epoll_wait(epfd, events, 1024, -1);

        for (int i = 0; i < nready; i++) {
            int connfd = events[i].data.fd;

            /*
             * 监听 fd：read callback 是 accept_cb；
             * 客户端 fd：read callback 是 recv_cb。
             */
            if (events[i].events & EPOLLIN) {
                conn_list[connfd]
                    .r_action.recv_callback(connfd);
            }

            /* 客户端可写时调用 send_cb */
            if (events[i].events & EPOLLOUT) {
                conn_list[connfd]
                    .send_callback(connfd);
            }
        }
    }
}
```

把整份 Reactor 压缩成一句话：

> epoll 主循环根据 fd 和事件找到回调；读回调接收请求并调用 KV handler，写回调发送 handler 生成的响应。

---

# 5. io_uring Proactor 网络层：`proactor.c`

## 5.1 Proactor 总体流程

```text
准备 ACCEPT 操作
  ↓ submit
等待完成队列 CQ
  ↓
ACCEPT 完成 → 得到 connfd → 准备 RECV
  ↓
READ 完成   → 调 kvs_protocol → 准备 SEND
  ↓
WRITE 完成  → 再次准备 RECV
```

Reactor 是“事件就绪以后调用 I/O”，这份 io_uring 代码是“先提交具体 I/O，再处理完成结果”。

## 5.2 操作类型与操作标记

```c
/* 用于区分 CQE 对应哪一种操作 */
#define EVENT_ACCEPT 0
#define EVENT_READ   1
#define EVENT_WRITE  2

struct conn_info {
    int fd;       // 操作所属的 socket
    int event;    // ACCEPT / READ / WRITE
};

#define ENTRIES_LENGTH 1024  // io_uring 队列项参数
#define BUFFER_LENGTH  1024  // 网络缓冲区长度

/* proactor_start 保存 main 传入的 kvs_protocol */
static msg_handler kvs_handler;
```

每个 SQE 都通过 `user_data` 带上 `conn_info`。CQE 返回以后，程序就知道完成的是谁的什么操作。

## 5.3 准备 send

```c
int set_event_send(struct io_uring *ring,
                   int sockfd,
                   void *buf,
                   size_t len,
                   int flags) {
    /* 从提交队列取得一个 SQE */
    struct io_uring_sqe *sqe = io_uring_get_sqe(ring);

    /* 标记这是 sockfd 的写操作 */
    struct conn_info info = {
        .fd = sockfd,
        .event = EVENT_WRITE,
    };

    /* 将 SQE 填写为 send 请求 */
    io_uring_prep_send(sqe, sockfd, buf, len, flags);

    /* 操作完成时，通过 user_data 找回 fd 和事件类型 */
    memcpy(&sqe->user_data, &info, sizeof(struct conn_info));
}
```

## 5.4 准备 recv

```c
int set_event_recv(struct io_uring *ring,
                   int sockfd,
                   void *buf,
                   size_t len,
                   int flags) {
    struct io_uring_sqe *sqe = io_uring_get_sqe(ring);

    /* 标记这是 sockfd 的读操作 */
    struct conn_info info = {
        .fd = sockfd,
        .event = EVENT_READ,
    };

    /* 将 SQE 填写为 recv 请求 */
    io_uring_prep_recv(sqe, sockfd, buf, len, flags);

    memcpy(&sqe->user_data, &info, sizeof(struct conn_info));
}
```

## 5.5 准备 accept

```c
int set_event_accept(struct io_uring *ring,
                     int sockfd,
                     struct sockaddr *addr,
                     socklen_t *addrlen,
                     int flags) {
    struct io_uring_sqe *sqe = io_uring_get_sqe(ring);

    /* 标记这是监听 sockfd 的 accept 操作 */
    struct conn_info info = {
        .fd = sockfd,
        .event = EVENT_ACCEPT,
    };

    /* 将 SQE 填写为 accept 请求 */
    io_uring_prep_accept(sqe, sockfd, addr, addrlen, flags);

    memcpy(&sqe->user_data, &info, sizeof(struct conn_info));
}
```

三个函数的结构完全一致，只有 prep 函数和 event 类型不同：

| 准备函数 | liburing prep | 事件标记 |
|---|---|---|
| `set_event_accept` | `io_uring_prep_accept` | `EVENT_ACCEPT` |
| `set_event_recv` | `io_uring_prep_recv` | `EVENT_READ` |
| `set_event_send` | `io_uring_prep_send` | `EVENT_WRITE` |

## 5.6 创建监听 socket

```c
int p_init_server(unsigned short port) {
    /* 创建 IPv4 TCP socket */
    int sockfd = socket(AF_INET, SOCK_STREAM, 0);

    struct sockaddr_in servaddr;
    servaddr.sin_family = AF_INET;
    servaddr.sin_addr.s_addr = htonl(INADDR_ANY);
    servaddr.sin_port = htons(port);

    /* 绑定端口 */
    if (bind(sockfd,
             (struct sockaddr *)&servaddr,
             sizeof(struct sockaddr)) == -1) {
        printf("bind failed: %s\n", strerror(errno));
    }

    /* 开始监听 */
    listen(sockfd, 10);
    return sockfd;
}
```

## 5.7 初始化 io_uring 并准备第一个 accept

```c
int proactor_start(unsigned short port, msg_handler handler) {
    /* 创建监听 socket */
    int sockfd = p_init_server(port);

    /* 保存 kvs_protocol */
    kvs_handler = handler;

    /* 初始化 io_uring 参数 */
    struct io_uring_params params;
    memset(&params, 0, sizeof(params));

    /* 创建提交队列和完成队列 */
    struct io_uring ring;
    io_uring_queue_init_params(
        ENTRIES_LENGTH,
        &ring,
        &params
    );

    /* accept 完成后，内核会填写客户端地址 */
    struct sockaddr_in clientaddr;
    socklen_t len = sizeof(clientaddr);

    /* 在进入循环之前，先准备第一个 accept SQE */
    set_event_accept(
        &ring,
        sockfd,
        (struct sockaddr *)&clientaddr,
        &len,
        0
    );

    /* READ 使用 buffer，KV 输出和 WRITE 使用 response */
    char buffer[BUFFER_LENGTH] = {0};
    char response[BUFFER_LENGTH] = {0};
```

## 5.8 提交和取得完成事件

```c
    while (1) {
        /* 把当前已经准备好的 SQE 提交给 io_uring */
        io_uring_submit(&ring);

        /* 至少等待一个操作完成 */
        struct io_uring_cqe *cqe;
        io_uring_wait_cqe(&ring, &cqe);

        /* 一次批量取出最多 128 个完成项 */
        struct io_uring_cqe *cqes[128];
        int nready = io_uring_peek_batch_cqe(
            &ring,
            cqes,
            128
        );

        for (int i = 0; i < nready; i++) {
            struct io_uring_cqe *entry = cqes[i];

            /* 从 user_data 还原这个操作的 fd 和类型 */
            struct conn_info result;
            memcpy(&result,
                   &entry->user_data,
                   sizeof(struct conn_info));
```

这里的两个信息来源要分清：

```text
result.fd / result.event
    提交 SQE 时由应用保存
    用来识别“谁的什么操作”

entry->res
    操作完成时由内核返回
    用来表示 accept 的新 fd、recv 长度或 send 结果
```

## 5.9 分发 ACCEPT、READ 和 WRITE 完成事件

```c
            if (result.event == EVENT_ACCEPT) {
                /*
                 * 当前 accept 已完成；
                 * 先准备下一个 accept，保持继续接收新连接。
                 */
                set_event_accept(
                    &ring,
                    sockfd,
                    (struct sockaddr *)&clientaddr,
                    &len,
                    0
                );

                /* accept CQE 的 res 是新客户端 fd */
                int connfd = entry->res;

                /* 为新客户端准备第一次 recv */
                set_event_recv(
                    &ring,
                    connfd,
                    buffer,
                    BUFFER_LENGTH,
                    0
                );

            } else if (result.event == EVENT_READ) {
                /* recv CQE 的 res 是本次收到的字节数 */
                int ret = entry->res;

                if (ret == 0) {
                    /* 客户端关闭连接 */
                    close(result.fd);
                } else if (ret > 0) {
                    /*
                     * buffer + ret 构成请求；
                     * response 接收 KV 协议输出；
                     * 返回值重新赋给 ret，作为响应长度。
                     */
                    ret = kvs_handler(buffer, ret, response);

                    /* 准备把 KV 响应发送给原客户端 */
                    set_event_send(
                        &ring,
                        result.fd,
                        response,
                        ret,
                        0
                    );
                }

            } else if (result.event == EVENT_WRITE) {
                /* send CQE 的 res 是写操作完成结果 */
                int ret = entry->res;

                /* 本轮响应结束，再次等待该客户端请求 */
                set_event_recv(
                    &ring,
                    result.fd,
                    buffer,
                    BUFFER_LENGTH,
                    0
                );
            }
        }

        /* 告诉 ring：这一批 CQE 已经处理完 */
        io_uring_cq_advance(&ring, nready);
    }
}
```

把整份 Proactor 压缩成一句话：

> 每处理一个完成事件，就准备这个连接的下一项操作；READ 完成与 WRITE 准备之间插入一次 KV handler 调用。

---

# 6. ntyco 协程网络层：`ntyco.c`

## 6.1 ntyco 在项目中的结构

```text
ntyco_start
  ↓ 创建
server 协程：socket → bind → listen → accept
  ↓ 每个客户端创建
server_reader 协程：recv → kvs_protocol → send
```

这里不展开 `nty_coroutine_create` 和 `nty_schedule_run` 的内部实现，只观察项目怎样调用框架。

## 6.2 客户端处理协程

```c
/* ntyco_start 保存 main 传入的 kvs_protocol */
static msg_handler kvs_handler;

void server_reader(void *arg) {
    /* server 创建协程时传入的是 cli_fd 地址 */
    int fd = *(int *)arg;
    int ret = 0;

    while (1) {
        /* 每轮准备一个新的接收缓冲区 */
        char buf[1024] = {0};

        /* 接收这个客户端的请求 */
        ret = recv(fd, buf, 1024, 0);

        if (ret > 0) {
            /* 为本轮响应准备缓冲区 */
            char response[1024] = {0};

            /*
             * buf + ret 是请求；
             * response 是 KV 输出；
             * slength 是响应长度。
             */
            int slength = kvs_handler(buf, ret, response);

            /* 将 KV 响应发送给当前客户端 */
            ret = send(fd, response, slength, 0);

            if (ret == -1) {
                close(fd);
                break;
            }
        } else if (ret == 0) {
            /* 客户端正常关闭连接 */
            close(fd);
            break;
        }
    }
}
```

与 Reactor、Proactor 相比，协程版本把同一个客户端的处理过程写在一个顺序循环里：

```c
recv(...);
kvs_handler(...);
send(...);
```

## 6.3 监听协程

```c
void server(void *arg) {
    /* 还原 ntyco_start 传入的端口 */
    unsigned short port = *(unsigned short *)arg;

    /* 创建 IPv4 TCP 监听 socket */
    int fd = socket(AF_INET, SOCK_STREAM, 0);
    if (fd < 0) return;

    /* 填写本地监听地址 */
    struct sockaddr_in local, remote;
    local.sin_family = AF_INET;
    local.sin_port = htons(port);
    local.sin_addr.s_addr = INADDR_ANY;

    /* 绑定并监听 */
    bind(fd,
         (struct sockaddr *)&local,
         sizeof(struct sockaddr_in));
    listen(fd, 20);

    printf("listen port : %d\n", port);

    while (1) {
        socklen_t len = sizeof(struct sockaddr_in);

        /* 接受一个客户端连接 */
        int cli_fd = accept(
            fd,
            (struct sockaddr *)&remote,
            &len
        );

        /*
         * 每个客户端创建一个 server_reader 协程；
         * 该协程负责此后的 recv、KV 处理和 send。
         */
        nty_coroutine *read_co;
        nty_coroutine_create(
            &read_co,
            server_reader,
            &cli_fd
        );
    }
}
```

## 6.4 ntyco 网络层入口

```c
int ntyco_start(unsigned short port, msg_handler handler) {
    /* 保存 kvs_protocol */
    kvs_handler = handler;

    /* 创建负责监听和 accept 的 server 协程 */
    nty_coroutine *co = NULL;
    nty_coroutine_create(&co, server, &port);

    /* 启动 ntyco 调度器 */
    nty_schedule_run();
}
```

把 ntyco 版本压缩成一句话：

> server 协程负责产生连接，每个连接再由一个 reader 协程按顺序完成接收、协议处理和发送。

---

# 7. 三种网络层放在一起回顾

## 7.1 同一阶段的代码对应关系

| 阶段 | Reactor | io_uring Proactor | ntyco |
|---|---|---|---|
| 保存 KV 函数 | `kvs_handler = handler` | `kvs_handler = handler` | `kvs_handler = handler` |
| 等待连接 | 监听 fd 的 `EPOLLIN` | 提交 `ACCEPT` SQE | `server` 协程调用 `accept` |
| 获得连接 | `accept_cb()` | `EVENT_ACCEPT` CQE | `accept()` 返回 |
| 等待请求 | 客户端 fd 的 `EPOLLIN` | 提交 `RECV` SQE | reader 协程调用 `recv` |
| KV 接入点 | `kvs_request()` | `EVENT_READ` 分支 | `server_reader()` |
| 发送响应 | `send_cb()` | 提交 `SEND` SQE | reader 协程调用 `send` |
| 继续循环 | 改回 `EPOLLIN` | WRITE 完成后再提交 RECV | while 回到 recv |

## 7.2 一次请求的三条完整路径

### Reactor

```text
epoll_wait
  → EPOLLIN
  → recv_cb
  → conn.rbuffer / conn.rlength
  → kvs_request
  → kvs_protocol
  → conn.wbuffer / conn.wlength
  → EPOLLOUT
  → send_cb
  → EPOLLIN
```

### Proactor

```text
提交 RECV
  → READ CQE
  → buffer / entry->res
  → kvs_protocol
  → response / 响应长度
  → 提交 SEND
  → WRITE CQE
  → 再提交 RECV
```

### ntyco

```text
server_reader 协程
  → recv
  → buf / ret
  → kvs_protocol
  → response / slength
  → send
  → 下一轮 recv
```

## 7.3 两种回调不要混淆

```c
/* Reactor 内部的网络事件回调 */
typedef int (*RCALLBACK)(int fd);

/* 三种网络层共同使用的 KV 协议回调 */
typedef int (*msg_handler)(char *msg,
                           int length,
                           char *response);
```

它们的层次是：

```text
Reactor 中：
epoll → RCALLBACK → recv_cb → msg_handler → kvs_protocol

Proactor 中：
CQE → EVENT_READ 分支 → msg_handler → kvs_protocol

ntyco 中：
server_reader → msg_handler → kvs_protocol
```

---

# 8. 最后的整体记忆

重新回顾这个项目时，只需要先恢复四个关键词：

```text
NETWORK_SELECT
    决定使用 Reactor、Proactor 还是 ntyco

msg_handler
    把网络层与 KV 协议层连接起来

kvs_protocol
    接收请求、生成响应，当前实现为 echo

网络事件循环
    三种模型分别用 epoll、io_uring、协程完成连接和收发
```

整个程序的共同主干是：

```c
/* main 把协议函数交给网络层 */
network_start(port, kvs_protocol);

/* 网络层保存函数地址 */
kvs_handler = handler;

/* 网络层收到请求后调用它 */
response_len = kvs_handler(
    request,
    request_len,
    response
);

/* 网络层按返回长度发送响应 */
send(clientfd, response, response_len, 0);
```

这四步就是六份代码共同围绕的核心。三种网络模型改变的是“请求怎样到达这四步、响应怎样离开这四步”，而 KV 协议入口保持一致。
