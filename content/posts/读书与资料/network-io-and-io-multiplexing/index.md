---
title: 网络 I/O 与 I/O 多路复用
slug: network-io-and-io-multiplexing
description: 从网络 I/O 使用场景出发，理解 socket、文件描述符、TCP 服务端执行流程，以及一连接一线程模型面临的资源问题
date: 2026-08-06T00:00:00+08:00
draft: false
image: cover.svg
tags:
  - 网络编程
  - Linux
  - Socket
  - I/O 多路复用
categories:
  - 后端开发
---

## 一、网络 I/O 的使用场景

网络是后端服务开发中的重要环节。生活中大量功能都依赖客户端与服务端之间的网络通信。

### 1. 微信发送文字、语音和视频

当用户在微信发送消息时，大致过程是：

```
发送方微信
    ↓
通过网络将数据发送给微信服务器
    ↓
服务器找到接收方
    ↓
将数据转发给接收方微信
```

文字、语音、图片、视频在网络上传输时，最终都会被转换成字节数据。

网络 I/O 主要关心：

```
数据从哪里读取
数据向哪里发送
读取了多少字节
发送了多少字节
```

网络 I/O 本身并不关心这些字节具体表示文字、图片还是视频。

### 2. 刷抖音

刷视频时：

```
抖音 App 请求视频
    ↓
服务器返回视频信息和资源地址
    ↓
App 通过网络下载视频数据
    ↓
客户端解码并播放
```

通常视频会被切分成多个数据块，App 可以一边下载、一边播放。

### 3. 使用 `git clone`

执行：

```
git clone 仓库地址
```

大致过程是：

```
本地 Git 客户端连接 GitHub
    ↓
向 GitHub 请求仓库数据
    ↓
GitHub 通过网络发送 Git 对象
    ↓
本地接收、校验和解压
    ↓
形成本地仓库
```

代码能够到达本地，本质上也是客户端和 GitHub 服务端之间进行了网络 I/O。

这些场景都可以概括成：

```
客户端发起请求 → 服务端处理请求 → 服务端返回响应
```

## 二、什么是网络 I/O

网络 I/O 是建立在客户端和服务端之间的数据传输过程，类似二者之间连接的网络管道。例如十个客户端分别连接服务器，服务端通常会获得十个连接 fd：

```
客户端 A ←→ fd 4
客户端 B ←→ fd 5
客户端 C ←→ fd 6
...
客户端 J ←→ fd 13
```

随着客户端数量增加，fd 数量也会增加，于是就产生了一个重要问题：

> 服务端如何同时管理大量 fd？

这正是后面需要学习 `select`、`poll` 和 `epoll` 的原因。

## 三、socket 与 fd

在 Linux 中，Linux 通过 socket API 通信。创建 socket：

```
int sockfd = socket(AF_INET, SOCK_STREAM, 0);
```

参数的含义：
- `AF_INET`：IPv4
- `SOCK_STREAM`：面向字节流，对应 TCP
- `0`：使用该组合默认的 TCP 协议

fd 本质是进程中的非负整数（此处了解，记忆也没啥用）：

```
0：标准输入
1：标准输出
2：标准错误
3：监听 socket
4：客户端连接 A
5：客户端连接 B
```

在当前服务端进程中，一个已连接 socket 的 fd 通常对应一条 TCP 连接。（存在 ACCEPT 的情况）

## 四、TCP 服务端的简便理解

可以把 TCP 服务端理解成酒店：
- `socket()`：聘请迎宾员
- `bind()`：安排迎宾员在哪个门口工作
- `listen()`：正式开始迎宾
- `accept()`：接待一位到达的客人
- `recv()`：听客人提出需求
- `send()`：将处理结果交给客人
- `close()`：结束服务

说说两类重要 fd：

**1. 监听 fd**

定义完 sockfd 经过 bind、listen 后，sockfd 就成为监听 fd。主要职责是监听新连接，不负责收发数据。

**2. 连接 fd**

参考代码中的 `int clientfd = accept(sockfd, ...);`。`accept()` 成功后会返回一个新的 fd：

```
sockfd：负责监听新客户端
clientfd：负责和某个客户端通信
```

关系如下：

```
                     ┌→ clientfd 4 ←→ 客户端 A
listenfd → accept() ─┼→ clientfd 5 ←→ 客户端 B
                     └→ clientfd 6 ←→ 客户端 C
```

监听 fd 通常只有一个，但每建立一个客户端连接，服务端就会获得一个新的连接 fd。

## 五、服务端代码执行流程

### 1. 创建 socket

```
int sockfd = socket(AF_INET, SOCK_STREAM, 0);
```

作用：
- 在内核中创建 socket 对象。
- 返回一个 fd。
- 当前还没有绑定 IP 和端口。
- 当前也没有开始监听。

更准确的变量名可以理解为：

```
int listenfd;
```

因为它后面会被设置为监听 socket。

### 2. 准备服务器地址

```
struct sockaddr_in servaddr;

servaddr.sin_family = AF_INET;
servaddr.sin_addr.s_addr = htonl(INADDR_ANY);
servaddr.sin_port = htons(2000);
```

#### `sin_family`

```
servaddr.sin_family = AF_INET;
```

表示使用 IPv4。

#### `INADDR_ANY`

```
servaddr.sin_addr.s_addr = htonl(INADDR_ANY);
```

`INADDR_ANY` 表示监听本机所有 IPv4 网络接口，对应：

```
0.0.0.0
```

假设服务器有多个 IP：

```
127.0.0.1
192.168.1.10
10.0.0.10
```

绑定 `0.0.0.0`，表示这些本机地址上的 2000 端口都可以接收连接。

#### `htons()`

```
servaddr.sin_port = htons(2000);
```

`htons` 表示：

```
host to network short
```

它把主机字节序的 16 位端口号转换成网络字节序。

同理：

```
htonl
```

表示把 32 位数据从主机字节序转换成网络字节序。

端口范围是：

```
0～65535
```

通常：

- `0`：让操作系统自动分配端口。
- `1～1023`：传统意义上的特权端口。
- `1024～65535`：普通程序通常可以使用。

但 1024 以上的端口也可能已经被其他程序占用。

### 3. 绑定 IP 和端口

```
bind(sockfd,
     (struct sockaddr *)&servaddr,
     sizeof(struct sockaddr));
```

`bind()` 将 socket 与本地 IP、端口建立联系：

```
sockfd ←→ 0.0.0.0:2000
```

可以理解为：

> 给迎宾人员安排具体的工作地点。

如果端口已经被其他服务绑定，可能出现：

```
Address already in use
```

你的代码通过下面的方式查看错误：

```
printf("bind error:%s\n", strerror(errno));
```

不过如果 `bind()` 失败，程序理论上不应该继续执行 `listen()`。

---

### 4. 进入监听状态

```
listen(sockfd, 10);
```

执行成功后，socket 从普通 socket 变成监听 socket。

此时可以使用：

```
ss -lntp | grep ':2000'
```

查看监听状态。

可能看到：

```
LISTEN 0 10 0.0.0.0:2000
```

---

### 5. 接收客户端连接

```
int clientfd = accept(
    sockfd,
    (struct sockaddr *)&clientaddr,
    &len
);
```

如果当前没有已经建立完成的客户端连接，阻塞模式下的 `accept()` 会暂停程序：

```
程序阻塞在 accept()
        ↓
等待客户端连接
        ↓
客户端完成 TCP 三次握手
        ↓
accept() 返回 clientfd
```

`accept()` 不会把原来的 `sockfd` 变成客户端连接。

它会返回一个全新的 `clientfd`。

## 六、代码中的三个服务端版本（代码文末附）

### 第一阶段：只处理一个客户端的一次数据

```
#if 0
    int clientfd = accept(...);

    char buffer[1024] = {0};
    int count = recv(clientfd, buffer, 1024, 0);

    printf("RECV:%s\n", buffer);

    count = send(clientfd, buffer, count, 0);
#endif
```

执行过程：

```
accept 一个客户端
    ↓
recv 一次
    ↓
send 一次
    ↓
后面不再 accept
```

这个版本只能：

- 接收一个客户端。
- 读取一次数据。
- 返回一次数据。

如果客户端想再次发送，服务端没有循环继续读取。

### 第二阶段：循环接收客户端，但串行处理

```
#elif 0
while (1) {
    int clientfd = accept(...);

    char buffer[1024] = {0};
    int count = recv(clientfd, buffer, 1024, 0);

    printf("RECV:%s\n", buffer);

    count = send(clientfd, buffer, count, 0);
}
#endif
```

这里增加了：

```
while (1)
```

因此服务端可以不断接收新客户端。

但是它仍然是串行处理：

```
accept 客户端 A
    ↓
recv 等待 A 发送数据
    ↓
send 返回给 A
    ↓
重新 accept 客户端 B
```

关键问题在这里：

```
recv(clientfd, buffer, 1024, 0);
```

默认情况下，`recv()` 是阻塞的。

假设客户端 A 已经连接，但一直没有发送数据：

```
服务端阻塞在 A 的 recv()
```

此时即使客户端 B 已经发起连接，应用程序也暂时无法回到 `accept()` 处理 B。

### 第三阶段：一个客户端对应一个线程

当前实际编译的是 `#else`：

```
#else
while (1) {
    int clientfd = accept(...);

    pthread_t thid;
    pthread_create(
        &thid,
        NULL,
        client_thread,
        &clientfd
    );
}
#endif
```

每接收一个客户端，创建一个线程。

线程函数：

```
void *client_thread(void *arg)
{
    int clientfd = *(int *)arg;

    while (1) {
        char buffer[1024] = {0};

        int count = recv(
            clientfd,
            buffer,
            1024,
            0
        );

        printf("RECV:%s\n", buffer);

        count = send(
            clientfd,
            buffer,
            count,
            0
        );
    }
}
```

运行关系变成：

```
主线程
  │
  ├─ accept 客户端 A
  │      └─ 创建线程 1 → recv(A)
  │
  ├─ accept 客户端 B
  │      └─ 创建线程 2 → recv(B)
  │
  └─ accept 客户端 C
         └─ 创建线程 3 → recv(C)
```

主线程只负责：

```
不断执行 accept()
```

工作线程负责：

```
recv() → 处理数据 → send()
```

这样即使线程 1 阻塞在客户端 A 的 `recv()`，主线程仍然可以继续 `accept()` 客户端 B。

准确来说，这是：

> 一个连接对应一个线程。

不是严格的”一次请求对应一个线程”，因为一个连接线程内部可以通过 `while` 处理多次请求。

## 七、对于阶段三一个连接一个线程的问题

这种模型简单直观，但连接数量增大后会产生问题。

### 1. 线程需要内存

每创建一个线程，都需要线程栈及相关管理资源。连接越多，线程越多，百万并发的时候...

### 2. 上下文切换成本

CPU 核心数量有限。

假设服务器只有 8 个 CPU 核心，却创建了 5000 个线程，操作系统必须不停地：

```
保存线程 A 的状态
切换到线程 B
保存线程 B 的状态
切换到线程 C
```

线程太多时，大量 CPU 时间可能消耗在线程调度上。

### 3. 大部分线程可能只是在等待

很多网络连接并不是一直都有数据。

例如即时通信中的长连接：

```
客户端保持在线
但大部分时间没有发送消息
```

对应线程会长期阻塞在：

```
recv(clientfd, ...);
```

如果一个空闲连接占用一个线程，就会浪费大量线程资源。

## 代码

```
#include <stdio.h>      // printf
#include <string.h>     // memset
#include <sys/socket.h>
#include <arpa/inet.h>  // htonl, htons
#include <errno.h>      // errno
#include <netinet/in.h> // struct sockaddr_in
#include <unistd.h>     // close
#include <pthread.h>

void *client_thread(void *arg) {
    int clientfd = *(int *)arg;
    while (1) {
        char buffer[1024] = {0};
        int count = recv(clientfd, buffer, 1024, 0); // 全部阻塞在 recv 上
        printf("RECV:%s\n", buffer);
        count = send(clientfd, buffer, count, 0);
        printf("SEND:%s\n", count);
    }
}

int main() {
    int sockfd = socket(AF_INET, SOCK_STREAM, 0); // create socket 聘请迎宾者
    // 绑定本地端口
    struct sockaddr_in servaddr;
    servaddr.sin_family = AF_INET;
    servaddr.sin_addr.s_addr = htonl(INADDR_ANY); // 0.0.0.0
    servaddr.sin_port = htons(2000); // 0-1023 系统默认，1024 之后都可以用，不能与其他的冲突
    // 在哪个门迎宾，安排的位置
    if (-1 == bind(sockfd, (struct sockaddr *)&servaddr, sizeof(struct sockaddr))) {
        printf("bind error:%s\n", strerror(errno));
    }
    // 登记绑定
    listen(sockfd, 10); // 正式上榜
    printf("listen finished\n");
    struct sockaddr_in clientaddr;
    socklen_t len = sizeof(clientaddr);
#if 0
    printf("accept\n");
    int clientfd = accept(sockfd, (struct sockaddr *)&clientaddr, &len); // 建立链接
    printf("accept finished\n");
    char buffer[1024] = {0};
    int count = recv(clientfd, buffer, 1024, 0);
    printf("RECV:%s\n", buffer);
    count = send(clientfd, buffer, count, 0);
    printf("SEND:%s\n", count);
#elif 0
    while (1) {
        printf("accept\n");
        int clientfd = accept(sockfd, (struct sockaddr *)&clientaddr, &len); // 建立链接
        printf("accept finished\n");
        char buffer[1024] = {0};
        int count = recv(clientfd, buffer, 1024, 0); // 全部阻塞在 recv 上
        printf("RECV:%s\n", buffer);
        count = send(clientfd, buffer, count, 0);
        printf("SEND:%s\n", count);
    } // 多个 io 如何接收数据
#else
    while (1) {
        printf("accept\n");
        int clientfd = accept(sockfd, (struct sockaddr *)&clientaddr, &len); // 建立链接
        printf("accept finished\n");
        pthread_t thid;
        pthread_create(&thid, NULL, client_thread, &clientfd);
    }
#endif
    getchar();
    printf("exit\n");
    return 0;
}
```
