---
title: Reactor 在 WebServer 与 WebSocket 中的应用
slug: reactor-webserver-websocket
description: 在 Reactor 网络层之上接入 HTTP 与 WebSocket，梳理 LT/ET、请求响应、sendfile 文件发送状态机、WebSocket 握手和 wrk 压测
date: 2026-08-09T00:00:00+08:00
draft: false
image: cover.svg
tags:
  - 网络编程
  - Linux
  - Reactor
  - HTTP
  - WebSocket
categories:
  - 后端开发
---

> 本篇是《网络IO》《IO 多路复用与 Reactor》《Reactor 与 epoll：百万并发网络服务器》的后续实践。  
> socket、fd、select、poll、epoll 接口、连接对象和 Reactor 基础不再重复；本篇只整理这节课新增的 LT/ET、HTTP WebServer、文件发送状态机和 WebSocket。

---

## 一、本节课做了什么

前一节已经搭好了 Reactor 网络层，本节在它上面接入两种协议业务：

```text
reactor.c：网络 I/O 层
  ├── recv_cb()
  └── send_cb()

webserver.c：HTTP 协议层
  ├── http_request()
  └── http_response()

websocket.c：WebSocket 协议层
  ├── ws_request()
  └── ws_response()
```

本节最重要的调用链：

```text
clientfd 出现 EPOLLIN
  ↓
recv_cb() 调用 recv()
  ↓
http_request() / ws_request()
  ↓
set_event(fd, EPOLLOUT, 0)
  ↓
clientfd 出现 EPOLLOUT
  ↓
send_cb()
  ↓
http_response() / ws_response()
  ↓
send()
  ↓
set_event(fd, EPOLLIN, 0)
```

因此，Reactor 与协议层的分工是：

```text
reactor.c   → 关心 fd 是否可读、可写，以及 recv/send
webserver.c → 关心 HTTP 请求和 HTTP 响应
websocket.c → 关心握手、帧解码和帧编码
```

---

## 二、LT 与 ET

### 2.1 LT：水平触发

只要内核接收缓冲区中还有数据，LT 就会继续报告可读事件。

例如客户端发送 32 字节，服务端一次只接收 10 字节：

```text
第一次 recv：10 字节，剩余 22
第二次 recv：10 字节，剩余 12
第三次 recv：10 字节，剩余 2
第四次 recv：2 字节，接收完成
```

### 2.2 ET：边沿触发

ET 主要关注从“没有数据”到“有数据”的状态变化。收到一次通知后，应在当前回调中尽量把数据处理完。

当前代码在 `accept_cb()` 中这样注册客户端：

```c
event_rigister(clientfd,EPOLLIN|EPOLLET);//|EPOLLET
```

这表示 `clientfd` 使用 ET，并且先关注可读事件。

ET 的处理思路：

```text
accept_cb() → 持续 accept，直到当前没有新连接
recv_cb()   → 持续 recv，直到当前数据读完
send_cb()   → 持续推进发送，直到发完或暂时不可写
```

循环读取时要配合非阻塞 fd。当前没有更多数据时，`recv()` 返回 `-1`，并且 `errno` 为 `EAGAIN` 或 `EWOULDBLOCK`，这表示本轮读取结束，应回到事件循环。

LT 和 ET 都不能决定 HTTP 请求或 WebSocket 帧是否完整。TCP 是字节流，协议层仍然需要根据协议格式判断消息边界。

---

## 三、`recv_cb()`：从网络层进入协议层

本节使用你提交的原代码：

```c
int recv_cb(int fd){
    memset(conn_list[fd].rbuffer,0,BUFFER_LENGTH);
	int count = recv(fd,  conn_list[fd].rbuffer, BUFFER_LENGTH, 0);
	if (count == 0) { // disconnect
		printf("client disconnect: %d\n", fd);
		close(fd);

        epoll_ctl(epfd, EPOLL_CTL_DEL, fd, NULL);//unfinished

		return -1;

				}else if(count<0){
                printf("count:%d,errno:%d ,%s\n", count,errno,strerror(errno));
		        close(fd);
                epoll_ctl(epfd, EPOLL_CTL_DEL, fd, NULL);//unfinished
                return 0;
                }
         conn_list[fd].rlength=count;
       // printf("RECV: %s\n", conn_list[fd].rbuffer);
#if 0//echo
                conn_list[fd].wlength=conn_list[fd].rlength;
                memcpy(conn_list[fd].wbuffer,conn_list[fd].rbuffer,conn_list[fd].wlength);
                printf("[%d]RECV: %s\n",conn_list[fd].rlength,conn_list[fd].rbuffer);
#elif0
    http_request(&conn_list[fd]);
#else 
    ws_request(&conn_list[fd]);
#endif

        set_event(fd,EPOLLOUT,0);
        return count;
}
```

### 3.1 接收数据

```c
int count = recv(fd,  conn_list[fd].rbuffer, BUFFER_LENGTH, 0);
```

数据从内核 socket 接收缓冲区复制到当前连接的 `rbuffer`，实际接收长度保存到：

```c
conn_list[fd].rlength=count;
```

### 3.2 三种业务切换

Echo 分支把收到的数据复制到写缓冲区：

```c
#if 0//echo
                conn_list[fd].wlength=conn_list[fd].rlength;
                memcpy(conn_list[fd].wbuffer,conn_list[fd].rbuffer,conn_list[fd].wlength);
                printf("[%d]RECV: %s\n",conn_list[fd].rlength,conn_list[fd].rbuffer);
```

	HTTP 分支把连接对象交给 `http_request()`：
	
	```c
	#elif0
	    http_request(&conn_list[fd]);
	```
	
	WebSocket 分支把连接对象交给 `ws_request()`：
	
	```c
	#else 
	    ws_request(&conn_list[fd]);
	```
	
	三种业务共用同一个 `recv_cb()`，只替换上层协议处理函数。这正是 Reactor 与业务分层后的好处。

### 3.3 从读事件切换到写事件

```c
set_event(fd,EPOLLOUT,0);
```

`http_request()` 或 `ws_request()` 处理完输入后，当前 fd 改为关注 `EPOLLOUT`。等 socket 可写时，Reactor 会调用 `send_cb()`。

---

## 四、HTTP 请求：`http_request()`

浏览器访问服务器时，会发送类似内容：

```http
GET / HTTP/1.1
Host: 172.16.145.129:2000
Connection: keep-alive
Upgrade-Insecure-Requests: 1
User-Agent: Mozilla/5.0 (Macintosh; Intel Mac OS X 10_15_7)
Accept: text/html,application/xhtml+xml,application/xml
Accept-Encoding: gzip, deflate
Accept-Language: zh-CN,zh;q=0.9,en;q=0.8
```

请求由请求行、请求头、空行和可选 body 组成：

```text
GET / HTTP/1.1 → 请求行
Host           → 请求头
Connection     → 请求头
User-Agent     → 请求头
空行           → 请求头结束
```

原代码：

```c
int http_request(struct conn *c){
    //printf("request: %s\n",c->rbuffer);
    memset(c->wbuffer,0,BUFFER_LENGTH);
    c->wlength=0;
    c->status=0;
}
```

当前 `http_request()` 主要完成一次新响应的状态初始化：

```text
c->rbuffer → 当前 HTTP 请求
c->wbuffer → 清空，准备写响应
c->wlength → 置 0
c->status  → 置 0，从响应状态机起点开始
```

它位于：

```text
recv_cb() 之后
send_cb() 之前
```

---

## 五、HTTP 响应：直接返回 HTML

原代码：

```c
int http_response(struct conn *c){
#if 1
    c->wlength=sprintf(c->wbuffer,"HTTP/1.1 200 OK\r\n"
    "Content-Type: text/html\r\n"
    "Accept-Ranges: bytes\r\n"
    "Content-Length: 82\r\n"
    "Date: Tue,30 Apr 2024 13:16:46 GMT\r\n\r\n"
    "<html><head><title>0voice.king</title></head><body><h1>King</h1></body></html>\r\n\r\n");
    return 0;
```

其中：

```http
HTTP/1.1 200 OK
Content-Type: text/html
Accept-Ranges: bytes
Content-Length: 82
Date: Tue,30 Apr 2024 13:16:46 GMT
```

是 HTTP 状态行和响应头。它们告诉浏览器响应是否成功、body 是什么类型、长度是多少。

下面是 HTTP body：

```html
<html><head><title>0voice.king</title></head><body><h1>King</h1></body></html>
```

浏览器看到 `Content-Type: text/html` 后，会把 body 当作 HTML 渲染，页面中显示 `King`。

这一分支中，HTTP 头和 body 一起写入：

```c
c->wbuffer
```

总长度由 `sprintf()` 的返回值保存到：

```c
c->wlength
```

调用链：

```text
recv_cb()
  ↓
http_request()
  ↓
设置 EPOLLOUT
  ↓
send_cb()
  ↓
http_response()
  ↓
HTTP 头和 HTML 写入 wbuffer
  ↓
send()
```

---

## 六、返回 `index.html`

原代码：

```c
#elif 0
    int filefd =open("index.html",O_RDONLY);
    struct stat stat_buf;
    fstat(filefd,&stat_buf);
    c->wlength=sprintf(c->wbuffer,"HTTP/1.1 200 OK\r\n"
    "Content-Type: text/html\r\n"
    "Accept-Ranges: bytes\r\n"
    "Content-Length: %ld\r\n"
    "Date: Tue,30 Apr 2024 13:16:46 GMT\r\n\r\n",
    stat_buf.st_size);

    int count=read(filefd,c->wbuffer+c->wlength,BUFFER_LENGTH-c->wlength);
    c->wlength +=count;

    close(filefd);
```

执行过程：

```text
open("index.html")
  ↓
fstat() 得到文件大小
  ↓
HTTP 响应头写入 wbuffer
  ↓
read() 把 index.html 追加在 HTTP 头后面
  ↓
wlength = HTTP 头长度 + 本次文件读取长度
  ↓
send_cb() 发送整个 wbuffer
```

`stat_buf.st_size` 被写入：

```http
Content-Length: 文件大小
```

这个分支的特点是：HTTP 头和文件内容都先进入应用层 `wbuffer`，然后统一调用 `send()`。

---

## 七、状态机与 `sendfile()`

当一个响应需要多个发送阶段时，课程使用 `status` 保存当前进度：

```text
status = 0 → 构造并发送 HTTP 头
status = 1 → 持续发送文件
status = 2 → 文件发送结束，清理状态
```

原代码：

```c
#elif 0
    int filefd =open("index.html",O_RDONLY);
    struct stat stat_buf;
    fstat(filefd,&stat_buf);
    if(c->status==0){
    c->wlength=sprintf(c->wbuffer,"HTTP/1.1 200 OK\r\n"
    "Content-Type: text/html\r\n"
    "Accept-Ranges: bytes\r\n"
    "Content-Length: %ld\r\n"
    "Date: Tue,30 Apr 2024 13:16:46 GMT\r\n\r\n",
    stat_buf.st_size);
    c->status=1;
    }else if (c->status==1){
        int ret=sendfile(c->fd,filefd,NULL,stat_buf.st_size);
        if(ret==-1){
            printf("error:%d\n",errno);
        }
        //c->wlength=0;
        memset(c->wbuffer,0,BUFFER_LENGTH);
        c->status=2;
    }else if(c->status==2){

        c->wlength=0;
        memset(c->wbuffer,0,BUFFER_LENGTH);
        c->status=0;
    }
    close(filefd);
```

状态流转：

```text
第一次进入 http_response()
  ↓ status == 0
构造 HTTP 响应头
  ↓ status = 1
send_cb() 发送 wbuffer，并继续关注 EPOLLOUT
  ↓
第二次进入 http_response()
  ↓ status == 1
sendfile() 发送 index.html
  ↓ status = 2
继续关注 EPOLLOUT
  ↓
第三次进入 http_response()
  ↓ status == 2
清理 wbuffer 和 wlength
  ↓ status = 0
重新关注 EPOLLIN
```

`sendfile()` 直接建立文件 fd 到 socket fd 的发送路径：

```c
int ret=sendfile(c->fd,filefd,NULL,stat_buf.st_size);
```

协议层仍然负责构造 HTTP 头，`sendfile()` 负责发送文件内容。状态机把两个阶段连接起来。

---

## 八、返回图片

图片与 HTML 在网络层都是字节数据，主要区别是 HTTP 的 `Content-Type`。

原代码：

```c
#else
int filefd =open("vip.png",O_RDONLY);
    struct stat stat_buf;
    fstat(filefd,&stat_buf);
    if(c->status==0){
    c->wlength=sprintf(c->wbuffer,"HTTP/1.1 200 OK\r\n"
    "Content-Type: image/png\r\n"
    "Accept-Ranges: bytes\r\n"
    "Content-Length: %ld\r\n"
    "Date: Tue,30 Apr 2024 13:16:46 GMT\r\n\r\n",
    stat_buf.st_size);
    c->status=1;
    }else if (c->status==1){
        int ret=sendfile(c->fd,filefd,NULL,stat_buf.st_size);
        if(ret==-1){
            printf("error:%d\n",errno);
        }
        //c->wlength=0;
        memset(c->wbuffer,0,BUFFER_LENGTH);
        c->status=2;
    }else if(c->status==2){

        c->wlength=0;
        memset(c->wbuffer,0,BUFFER_LENGTH);
        c->status=0;
    }
    close(filefd);
```

与 HTML 文件的区别：

| 响应内容 | 打开的文件 | Content-Type |
|---|---|---|
| 网页 | `index.html` | `text/html` |
| PNG 图片 | `vip.png` | `image/png` |

Reactor 的 `recv_cb()`、`send_cb()` 不需要知道 body 是网页还是图片。WebServer 协议层设置正确的响应头即可。

---

## 九、`send_cb()` 如何驱动多阶段发送

原代码：

```c
int send_cb(int fd){
#if 0

    http_response(&conn_list[fd]);

#else
    ws_response(&conn_list[fd]);

#endif
    int count=0;
    if(conn_list[fd].status==1){
        //printf("SEND:%s\n",conn_list[fd].wbuffer);
     count = send(fd, conn_list[fd].wbuffer,conn_list[fd].wlength, 0);
    set_event(fd,EPOLLOUT,0);
}else if(conn_list[fd].status==2){
    set_event(fd,EPOLLOUT,0);
}else if(conn_list[fd].status==0){
    if(conn_list[fd].wlength !=0){
    count = send(fd, conn_list[fd].wbuffer,conn_list[fd].wlength, 0); 
    }
    set_event(fd,EPOLLIN,0);
}
    //set_event(fd,EPOLLOUT,0);
    return count;
}
```

### 9.1 选择协议响应

学习 HTTP 时，让写回调进入：

```c
#if 0

    http_response(&conn_list[fd]);
```

学习 WebSocket 时，让写回调进入：

```c
#else
    ws_response(&conn_list[fd]);
```

### 9.2 状态与事件切换

```text
status == 1
  → send() 发送当前 wbuffer
  → 继续设置 EPOLLOUT
  → 等待下一阶段

status == 2
  → 继续设置 EPOLLOUT
  → 进入状态机清理阶段

status == 0
  → 如果 wlength 不为 0，就发送当前数据
  → 设置 EPOLLIN
  → 等待下一次请求
```

这就是“需要发送多次时怎么办”的答案：

> 保存当前发送状态，再通过后续的 EPOLLOUT 事件继续推进，而不是把整个过程强行塞进一次回调。

---

## 十、WebSocket 握手

WebSocket 用于浏览器与服务器之间的长连接和双向通信。

建立 WebSocket 连接时，浏览器先发送 HTTP Upgrade 请求，其中包括：

```http
Upgrade: websocket
Connection: Upgrade
Sec-WebSocket-Key: T29c4Rys861qGqbGGNdkwA==
Sec-WebSocket-Version: 13
```

你提交的 WebSocket 代码：

```c
#include"server.h"
#include<stdio.h>
#define GUID  "258EAFA5-E914-47DA-95CA-C5AB0DC85B11"
/*
Key："T29c4Rys861qGqbGGNdkwA=="
T29c4Rys861qGqbGGNdkwA==258EAFA5-E914-47DA-95CA-C5AB0DC85B11
SHA-1

20 bytes
base64

handshake

transmission
decode
encode

*/
```

服务端计算 `Sec-WebSocket-Accept` 的过程：

```text
客户端 Sec-WebSocket-Key
  +
固定 GUID
  ↓
SHA-1
  ↓
160 bit = 20 bytes
  ↓
Base64
  ↓
Sec-WebSocket-Accept
```

固定 GUID：

```c
#define GUID  "258EAFA5-E914-47DA-95CA-C5AB0DC85B11"
```

客户端 Key 放在前面，GUID 放在后面。

服务端响应格式：

```http
HTTP/1.1 101 Switching Protocols
Upgrade: websocket
Connection: Upgrade
Sec-WebSocket-Accept: 计算结果
```

返回 101 后，同一个 TCP 连接不再传输普通 HTTP 消息，而是进入 WebSocket 帧传输阶段。

---

## 十一、WebSocket 在 Reactor 中的位置

原代码：

```c
int ws_request(struct conn *c){
    printf("request: %s",c->rbuffer);
}

int ws_response(struct conn *c){
    
}
```

当前两个函数是课程为后续实现预留的协议接口。

`ws_request()` 后续负责：

```text
握手阶段 → 解析 HTTP Upgrade 请求和 Sec-WebSocket-Key
传输阶段 → 解析 WebSocket frame，完成 decode
```

`ws_response()` 后续负责：

```text
握手阶段 → 构造 101 和 Sec-WebSocket-Accept
传输阶段 → 把文本或二进制业务数据 encode 成 WebSocket frame
```

在 Reactor 中的调用位置：

```text
EPOLLIN
  ↓
recv_cb()
  ↓
ws_request()
  ↓
设置 EPOLLOUT
  ↓
send_cb()
  ↓
ws_response()
  ↓
send()
```

WebSocket 是应用层协议，底层仍然是普通 TCP `clientfd`。Reactor 不需要为 WebSocket 创建一种新 fd。

---

## 十二、使用 wrk 测试 WebServer

测试命令：

```bash
wrk -t10 -c50 -d10s http://172.16.145.129:2000/
```

课程中的测试现象：

```text
打印每次 request 和 Host 数据时：QPS 约 1.5 万
关闭逐请求打印后：QPS 约 4.5 万
```

逐请求打印会产生格式化、终端输出和系统调用开销，因此会明显降低服务器吞吐量。

wrk 结果中主要关注：

```text
Requests/sec → 每秒请求数，即 QPS
Latency      → 请求延迟
timeout      → 超时数量
read/write   → 读写错误
Transfer/sec → 每秒传输数据量
```

局域网延迟较小；公网还会受到网络时延、丢包、带宽和中间设备影响。

---

## 十三、Connection reset by peer

`Connection reset by peer` 表示连接被对端通过 TCP RST 重置。

常见情况：

```text
客户端提前关闭连接
压测工具结束请求
服务器准备发送时客户端已经退出
网络设备重置连接
```

服务器应识别错误并清理当前连接，不要让单个客户端断开导致整个进程退出。

---

## 十四、本节课的三个重点

### 重点一：事件与协议函数的对应关系

```text
clientfd + EPOLLIN
  → recv_cb()
  → http_request() / ws_request()

clientfd + EPOLLOUT
  → send_cb()
  → http_response() / ws_response()
```

### 重点二：网络层与协议层分离

```text
reactor.c
  → 管理 fd、事件、recv、send

webserver.c
  → 处理 HTTP request 和 response

websocket.c
  → 处理 handshake、decode 和 encode
```

### 重点三：多次发送使用状态机

```text
一次发送即可完成
  → wbuffer + wlength

需要多个发送阶段
  → status 保存进度
  → 多次 EPOLLOUT 推进

发送静态文件
  → HTTP 头放入 wbuffer
  → 文件内容交给 sendfile()
```

最终理解：

> Reactor 提供统一的事件驱动网络层；HTTP 和 WebSocket 负责协议，业务数据何时读取、何时发送，由 fd 的事件和回调共同驱动。
