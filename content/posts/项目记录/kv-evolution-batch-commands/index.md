---
title: KV 进化（四）：Batch Commands 批量命令处理
slug: kv-evolution-batch-commands
description: 围绕 TCP 字节流边界、CRLF 命令拆分、响应缓冲区拼接和不完整命令处理，实现 KV 服务端的批量命令能力
date: 2026-09-14T00:00:00+08:00
draft: false
image: cover.svg
tags:
  - KV 存储
  - Batch Commands
  - TCP
  - 协议解析
  - C 语言
categories:
  - 项目记录
---

实现批处理功能 

已知现在服务端发送给客户端的响应数据 已经有\r\n 作为TCP字节流的数据分割了 
我们在 kvs_protocol 前 添加 对客户端向服务端发送指令 的处理 
static int kvs_trim_command_end(char *msg,int length){
    if(msg==NULL||length<=0){
        return -1;
    }
    while(length>0&&(msg[length-1]=='\r'||msg[length-1]=='\n')){
        msg[length-1]='\0';
        length--;
    }
    return length;
}

很容易的值 是将 发送指令末尾的\r\n 去除 因为字节流没有区分 何时终止  

这里需要区分两种“拆分”：

- `kvs_split_token()` 使用空格拆分一条命令中的命令名、key 和 value。
- `\r\n` 用于拆分 TCP 字节流中的多条命令。

例如：

```text
SET name Jasper\r\nGET name\r\n
```

第一层先按照 `\r\n` 拆成：

```text
SET name Jasper
GET name
```

第二层再分别调用 `kvs_split_token()`：

```text
SET / name / Jasper
GET / name
```

因此，`kvs_trim_command_end()` 还不是真正的批量处理。它只是让原来的单指令处理器能够接收以 `\r\n` 结尾的完整命令，避免 `\r\n` 被当作 key 或 value 的一部分。

## 在一次 recv 中处理多条完整命令

先增加一个函数，在接收到的字节中寻找第一个 `\r\n`：

```c
static int kvs_find_crlf(const char *msg, int length)
{
    if (msg == NULL || length < 2) {
        return -1;
    }

    for (int i = 0; i + 1 < length; i++) {
        if (msg[i] == '\r' && msg[i + 1] == '\n') {
            return i;
        }
    }

    return -1;
}
```

返回值是 `\r` 相对于当前命令起点的下标。小于 0 表示当前数据中还没有找到完整的命令结束标记；等于 0 则表示收到的是空命令 `\r\n`。

批量处理函数维护两个偏移量：

```c
int request_offset = 0;
int response_offset = 0;
```

`request_offset` 表示请求已经处理到哪里，`response_offset` 表示响应缓冲区已经写入多少字节。

每找到一条完整命令，就把 `\r` 替换成 `\0`，从原始接收缓冲区中临时分离出一个 C 字符串：

```c
char *command_start = msg + request_offset;
int remaining_length = length - request_offset;
int command_end = kvs_find_crlf(command_start, remaining_length);

command_start[command_end] = '\0';
```

这里的 `command_start` 没有创建或复制新数组，只是通过指针运算指向 `msg` 中当前命令开始的位置。

随后继续复用原来的单命令函数：

```c
int response_length = kvs_protocol(
    command_start,
    command_end,
    response + response_offset
);
```

一条响应本身已经包含 `\r\n`。例如：

```c
sprintf(response, "OK\r\n");
```

它会写入 `O K 0D 0A 00`，返回的有效长度是 4，不包含末尾的 `\0`，但已经包含 `\r\n`。因此响应位置只需要增加：

```c
response_offset += response_length;
```

请求位置则需要跨过命令正文以及两个结束字节：

```c
request_offset += command_end + 2;
```

最终，批量函数不断寻找完整命令、调用 `kvs_protocol()`，并把每一条响应依次写入同一个响应缓冲区。

## 接入网络入口

最初的网络回调只返回响应长度：

```c
typedef int (*msg_handler)(
    char *msg,
    int length,
    char *response
);
```

这对于“一次 `recv()` 就处理全部数据”的旧模型足够，但引入持久请求缓冲区后，网络层还必须知道：协议层究竟消费了多少请求字节？

因此回调扩展成五个参数：

```c
typedef int (*msg_handler)(
    char *msg,                 // 当前累计的请求数据
    int length,                // 请求缓冲区有效长度
    char *response,            // 响应写入位置
    int response_capacity,     // 响应缓冲区容量
    int *consumed_length       // 返回已经处理的请求长度
);
```

这里存在两个完全不同的长度：

```text
函数返回值                 // 生成了多少响应字节
*consumed_length           // 消费了多少请求字节
```

批量入口在开始时把消费长度清零：

```c
*consumed_length = 0;
```

每处理一条完整命令，`request_offset` 都会跨过命令正文和 `\r\n`。批量处理结束后：

```c
*consumed_length = request_offset;
return response_offset;
```

网络适配函数只负责转发，不自己理解命令内容：

```c
static int kvs_network_protocol(
    char *msg,
    int length,
    char *response,
    int response_capacity,
    int *consumed_length
)
{
    return kvs_batch_protocol(
        msg,
        length,
        response,
        response_capacity,
        consumed_length
    );
}
```

网络模型启动时统一接收这个入口：

```c
reactor_start(port, kvs_network_protocol);
ntyco_start(port, kvs_network_protocol);
proactor_start(port, kvs_network_protocol);
```

完整调用链可以压缩成：

```text
网络层接收字节
    ↓
kvs_network_protocol()       // 网络协议入口
    ↓
kvs_batch_protocol()         // 按 CRLF 取出完整命令
    ↓
kvs_protocol()               // 处理一条命令
    ↓
kvs_split_token()            // 按空格拆参数
    ↓
kvs_filter_protocol()        // 分发 SET、GET、DEL 等命令
    ↓
具体存储引擎
```

快照加载和 AOF 重放仍然使用 `kvs_protocol()`。它们读取的是本地持久化文件中的单条命令，不需要经过网络批量入口。

## 第一阶段验证：一次 recv 中的多条命令

一次发送：

```text
GET batch_missing_1\r\nGET batch_missing_2\r\n
```

一次返回：

```text
NO EXIST\r\nNO EXIST\r\n
```

混合批量测试：

```text
SET batch_test 123\r\nGET batch_test\r\n
```

返回：

```text
OK\r\n123\r\n
```

这说明服务端已经能够处理一次 `recv()` 中存在多条完整命令的情况，也就是通常所说的“粘包”。客户端不必再严格执行“发送一条、等待一条、再发送下一条”，多条命令可以共用一次应用层网络往返。

但是，这时只解决了“一个接收缓冲区里有多条命令”，还没有解决“一条命令分多次到达”。

## 从粘包继续走向半包

TCP 是字节流，不保存应用层消息边界。一条命令可能被拆成：

```text
第一次 recv：SET name Jas
第二次 recv：per\r\n
```

第一次数据中找不到 `\r\n`，协议层能够判断命令不完整，但如果网络层在下一次 `recv()` 时覆盖旧数据，`SET name Jas` 仍然会丢失。

解决请求半包需要建立下面这条规则：

```text
recv 新数据
    ↓
追加到该连接的请求缓冲区末尾
    ↓
协议层执行其中所有完整的 CRLF 命令
    ↓
删除已经处理的前缀
    ↓
保留末尾不完整的数据，等待下一次 recv
```

关键不是简单地“增大 `recv` 缓冲区”，而是让缓冲区拥有跨多次接收的生命周期，并准确维护有效长度。

## 公共请求缓冲区

在 `server.h` 中定义通用输入缓冲区：

```c
typedef struct kvs_input_buffer {
    char data[BUFFER_LENGTH]; // 保存累计请求和末尾半包
    int length;               // data 中当前有效字节数
} kvs_input_buffer_t;
```

三个核心概念必须分开：

```text
recv 返回值              // 本次新收到多少字节
input.length             // 当前累计有多少请求字节
consumed_length          // 协议层本次处理了多少请求字节
```

追加接收的基本写法：

```c
int available = BUFFER_LENGTH - input.length;

int received = recv(
    fd,
    input.data + input.length, // 从已有数据末尾继续写
    available,                 // 防止越过数组边界
    0
);

input.length += received;
```

为什么传给协议层的是 `input.length`，而不是本次 `recv()` 的返回值？

```text
第一次收到 "GET na"       // received = 6，input.length = 6
第二次收到 "me\r\n"       // received = 4，input.length = 10
```

第二次解析时，协议层需要看到累计的 `GET name\r\n` 共 10 字节，而不是只看到本次新收到的 4 字节。

## consume：删除已处理前缀，保留末尾半包

命令执行完成，不等于命令已经从请求缓冲区中消失。`kvs_handler()` 负责解析和执行，但网络层仍然持有原始 `input`。

因此增加公共函数：

```c
int kvs_input_buffer_consume(
    kvs_input_buffer_t *buffer,
    int consumed_length
)
{
    if (buffer == NULL ||
        consumed_length < 0 ||
        consumed_length > buffer->length) {
        return -1;
    }

    int remaining_length =
        buffer->length - consumed_length;

    if (consumed_length > 0 && remaining_length > 0) {
        memmove(
            buffer->data,
            buffer->data + consumed_length,
            remaining_length
        );
    }

    buffer->length = remaining_length;
    return 0;
}
```

例如缓冲区内容是：

```text
[GET a\r\n][SET name Jas]
```

第一条命令完整，第二条命令不完整。协议层返回第一条命令的长度作为 `consumed_length`，随后 `memmove()` 把未处理部分移动到开头：

```text
consume 前：[GET a\r\n][SET name Jas]
consume 后：[SET name Jas]
```

如果当前只有半包：

```text
input = "GET na"
consumed_length = 0
```

调用 `consume(input, 0)` 不会删除任何数据，半包会原样保留。

`memmove()` 而不是 `memcpy()` 的原因是源区间和目标区间来自同一个数组，可能发生重叠。

## 五个重要长度

随着功能增加，最容易混淆的是各种长度。可以统一记成：

```text
received / ret           // 本次 recv 新收到的字节数
input.length             // 请求缓冲区累计有效长度
available                // 请求缓冲区剩余容量
consumed_length          // 协议层处理掉的请求长度
wlength / slength        // 协议层生成的响应长度
```

关系如下：

```c
available = BUFFER_LENGTH - input.length;
input.length += received;
remaining = input.length - consumed_length;
```

其中：

```text
Reactor 使用 c->wlength
NtyCo 使用局部变量 slength
Proactor 使用 connection->wlength
```

名字不同，但三者都表示响应缓冲区中有效的响应字节数。

## Reactor 的半包接入

Reactor 使用 `conn_list[fd]` 保存每个连接的状态。原来的：

```c
char rbuffer[BUFFER_LENGTH];
int rlength;
```

替换为：

```c
kvs_input_buffer_t input;
```

接收时不再清空请求缓冲区，而是追加到末尾：

```c
int available = BUFFER_LENGTH - conn_list[fd].input.length;

int count = recv(
    fd,
    conn_list[fd].input.data + conn_list[fd].input.length,
    available,
    0
);

conn_list[fd].input.length += count;
```

`kvs_request()` 负责连接网络层和协议层：

```c
int kvs_request(struct conn *c)
{
    int consumed_length = 0;

    c->wlength = kvs_handler(
        c->input.data,
        c->input.length,
        c->wbuffer,
        BUFFER_LENGTH,
        &consumed_length
    );

    if (c->wlength < 0) {
        return -1;
    }

    if (kvs_input_buffer_consume(
            &c->input,
            consumed_length) < 0) {
        c->wlength = 0;
        return -1;
    }

    return 0;
}
```

事件切换规则：

```text
wlength < 0     // 协议失败，关闭连接
wlength == 0    // 没有完整命令，继续监听 EPOLLIN
wlength > 0     // 已经生成响应，切换到 EPOLLOUT
```

Reactor 的关键是：`kvs_request()` 的一次调用会结束，但 `conn_list[fd].input` 与连接一起长期存在。

## NtyCo 的半包接入

NtyCo 为每个客户端创建一个 `server_reader()` 协程。它不需要全局连接数组，因为协程函数自身的栈帧就可以保存该连接的状态。

缓冲区必须放在 `while` 外面：

```c
void server_reader(void *arg)
{
    int fd = *(int *)arg;
    int ret = 0;
    kvs_input_buffer_t input = {0}; // 整个连接期间一直存在

    while (1) {
        // 反复接收同一客户端的数据
    }
}
```

如果把 `input` 定义在 `while` 内，每轮都会重新初始化，上一轮保存的半包就会消失。

NtyCo 的一轮处理：

```text
计算 available
    ↓
recv 到 input 尾部
    ↓
input.length += ret
    ↓
kvs_handler 得到 slength 和 consumed_length
    ↓
consume 删除完整请求
    ↓
slength == 0：continue，等待下一次 recv
slength > 0：send 响应
```

这里的 `consumed_length` 放在循环内是合理的，因为它只描述本轮协议调用的结果；而 `input` 必须放在循环外，因为请求数据需要跨多轮保留。

## Proactor / io_uring 的半包接入

Proactor 比前两种模型更复杂，因为 `recv` 和 `send` 是先提交、后完成的异步操作。

原实现只有一组公共缓冲区：

```c
char buffer[BUFFER_LENGTH];
char response[BUFFER_LENGTH];
```

所有客户端共用它们会产生两个问题：

```text
客户端之间可能互相覆盖数据
异步操作尚未完成时，缓冲区可能已经被下一项操作改写
```

改造后使用 `struct conn` 保存每个客户端自己的：

```text
fd
input
wbuffer
wlength
```

并维护 fd 到连接状态的映射：

```c
static struct conn *proactor_connections[CONNECTION_SIZE];
```

接受新连接后动态创建状态：

```c
struct conn *connection = calloc(1, sizeof(*connection));

connection->fd = connfd;
proactor_connections[connfd] = connection;
```

使用 `calloc()` 是因为它在申请 `struct conn` 内存后会将内容清零，使 `input.length` 和 `wlength` 从 0 开始。相比之下，`malloc()` 申请的内存内容是不确定的，还需要额外 `memset()`。

### SQE、CQE 与 user_data

提交读取操作时，`conn_info` 被复制到 SQE 的 `user_data`：

```c
struct conn_info info = {
    .fd = sockfd,
    .event = EVENT_READ,
};

memcpy(&sqe->user_data, &info, sizeof(info));
```

操作完成后，CQE 会把相同的 `user_data` 带回来：

```text
result.fd       // 哪个客户端完成了操作
result.event    // ACCEPT、READ 或 WRITE
entries->res    // 操作结果，例如 recv 收到多少字节
```

`user_data` 可以理解为提交异步任务时贴上的标签，任务完成后内核把标签还给程序。

收到 READ 完成事件后，通过 fd 找回连接：

```c
struct conn *connection =
    proactor_connections[result.fd];
```

这行代码没有新建 `struct conn`。真正创建对象的是 `calloc()`；这里仅仅取回之前保存的地址。

### EVENT_READ

`EVENT_READ` 不是在现场同步调用 `recv()`，而是处理“之前提交的 recv 已经完成”这一结果：

```text
读取 entries->res
    ↓
通过 result.fd 找到 connection
    ↓
connection->input.length += ret
    ↓
调用 kvs_handler
    ↓
consume 已处理请求
    ↓
有响应则提交 send
只有半包则重新提交 recv
```

半包时不能只写 `continue`。因为上一项异步 `recv` 已经完成，必须显式调用 `set_event_recv()` 提交下一项读取任务：

```c
set_event_recv(
    &ring,
    result.fd,
    connection->input.data + connection->input.length,
    available,
    0
);
```

### EVENT_WRITE

`EVENT_WRITE` 也不是现场调用 `send()`，而是处理“之前提交的异步 send 已经完成”。

发送完成后必须重新提交下一次 `recv`。而且接收位置不能退回旧的公共 `buffer`，必须继续使用当前连接自己的缓冲区：

```c
struct conn *connection =
    proactor_connections[result.fd];

int available =
    BUFFER_LENGTH - connection->input.length;

set_event_recv(
    &ring,
    result.fd,
    connection->input.data + connection->input.length,
    available,
    0
);
```

如果不修改 `EVENT_WRITE`，第一次接收会写入 `connection->input`，但响应发送完成后的第二次接收又会写回旧公共 `buffer`，整个半包链路会在第二轮断开。

Proactor 的闭环是：

```text
提交 recv
    ↓
EVENT_READ
    ↓
解析请求并提交 send
    ↓
EVENT_WRITE
    ↓
再次提交 recv
```

## 三种网络模型的共同点与差异

共同规则完全一致：

```text
追加接收
只处理完整命令
消费已处理前缀
保留末尾半包
```

区别只在于连接状态保存在哪里、何时继续接收：

| 网络模型 | 每连接状态保存位置 | 半包后如何继续接收 |
| --- | --- | --- |
| Reactor | `conn_list[fd]` | 继续监听 `EPOLLIN` |
| NtyCo | `server_reader()` 协程栈 | `continue` 后再次 `recv()` |
| Proactor | `proactor_connections[fd]` | 再提交一次异步 `recv` |

因此，协议层不需要知道 `epoll`、协程或 io_uring。协议层只提供统一契约：

```text
给我累计请求和响应空间
我返回响应长度和请求消费长度
```

网络层也不需要知道 `GET`、`SET` 的语义。它只负责：

```text
保存未消费数据
发送协议层生成的响应
```

## 请求半包验证

最小半包测试：

第一次发送：

```text
GET half_
```

此时没有 `\r\n`，服务端不响应。第二次在同一 Persistent TCP 连接发送：

```text
key\r\n
```

服务端累计得到：

```text
GET half_key\r\n
```

返回：

```text
NO EXIST\r\n
```

进一步验证“完整命令 + 末尾半包”：

第一次发送：

```text
GET one\r\nGET two_
```

只返回第一条命令的响应：

```text
NO EXIST\r\n
```

随后发送：

```text
key\r\n
```

服务端将保留的 `GET two_` 与新数据拼成：

```text
GET two_key\r\n
```

再返回一次：

```text
NO EXIST\r\n
```

Reactor、NtyCo 和 Proactor 均通过了上述半包测试。

## 半包、批量命令与 AOF

请求缓冲区和 `consume()` 本身不会写入 AOF。只有完整并实际执行的写命令才沿用原有持久化流程：

```text
recv 半包
    ↓
input 保存                    // 不写 AOF
    ↓
发现完整 CRLF 命令
    ↓
kvs_protocol 执行 SET / DEL
    ↓
写入 AOF
```

例如分两次发送：

```text
第一次：SET user Jas
第二次：per\r\n
```

第一次没有完整命令，不执行，也不写 AOF。第二次拼成 `SET user Jasper\r\n` 后，只执行一次，并且 AOF 只记录一次完整写命令。

```text
GET                    // 不写 AOF
SET、DEL 等写命令       // 写 AOF
批量写命令              // 按执行顺序分别写入 AOF
不完整半包              // 不执行，不写 AOF
```

快照机制也不需要直接理解网络半包；它看到的仍然是已经执行后的存储状态。

## 当前阶段完成情况

当前已经完成：

```text
CRLF 请求边界定义
一次 recv 中的批量命令解析
响应按 CRLF 分隔
协议层返回 consumed_length
公共请求缓冲区及 consume
Reactor 请求半包
NtyCo 请求半包
Proactor / io_uring 请求半包
三种网络模型的半包实测
```

对应提交：

```text
1ccfcc1  feat(protocol): support CRLF-delimited command batches
32301d3  feat(network): 支持三种网络模型的请求半包缓冲
```

## 下一阶段：partial send

当前已经解决的是请求方向：

```text
一条请求可能需要多次 recv 才能收完整
```

还没有解决的是响应方向：

```text
一次 send 可能只接受响应的一部分
```

例如响应长度为 10：

```c
int sent = send(fd, response, 10, 0);
```

`sent` 可能只等于 4。此时前 4 字节已经发送，剩余 6 字节必须从正确偏移继续发送：

```c
send(fd, response + 4, 6, 0);
```

下一阶段需要维护：

```text
wbuffer     // 完整响应
wlength     // 响应总长度
woffset     // 已经发送的长度
remaining   // wlength - woffset
```

三种网络模型的方向相同，但继续发送的方式不同：

| 网络模型 | partial send 后的动作 |
| --- | --- |
| Reactor | 保持 `EPOLLOUT`，下次从 `woffset` 继续发送 |
| NtyCo | 在协程中循环 `send()`，直到全部完成 |
| Proactor | 根据 WRITE CQE 累加 `woffset`，重新提交剩余 `send` |

只有当：

```c
woffset == wlength
```

才能认为这次响应真正发送完毕。

需要特别区分：即使服务端一次 `send()` 全部成功，客户端也不一定一次 `recv()` 就收到完整响应。TCP 仍然是字节流，客户端同样需要根据响应协议边界进行累计和拆分。

## 响应 partial send 改造正式开始

前面的设计仍然沿用了旧名字：

```text
wbuffer
wlength
woffset
```

真正落地时，为了让请求方向和响应方向的状态更加对称，最终将它们整理成了统一的输出缓冲区：

```c
typedef struct kvs_output_buffer {
    char data[BUFFER_LENGTH]; // 保存完整响应内容
    int length;               // 完整响应总长度
    int offset;               // 已经成功发送的长度
} kvs_output_buffer_t;
```

随后将原来 `struct conn` 中零散的：

```c
char wbuffer[BUFFER_LENGTH];
int wlength;
```

替换为：

```c
kvs_output_buffer_t output; // 当前连接的响应发送状态
```

至此，一条连接拥有两类明确的状态：

```c
struct conn {
    int fd;                         // 客户端连接 fd
    kvs_input_buffer_t input;       // 请求方向：累计尚未处理的数据
    kvs_output_buffer_t output;     // 响应方向：保存尚未发完的数据
};
```

它们的职责并不完全相同：

| 缓冲区 | 保存什么 | 推进方式 |
| --- | --- | --- |
| `input` | 尚未处理完的请求字节 | `consume()` 后通过 `memmove()` 将剩余数据移到开头 |
| `output` | 尚未发送完的响应字节 | 响应内容保持不动，只增加 `offset` |

输入缓冲区没有 `offset`，是因为每次消费后都会整理数据：

```text
已处理请求 + 未处理请求
        ↓ consume
未处理请求被移动到 data[0]
```

输出缓冲区不能这样做。异步发送过程中，内核可能仍在使用原来的响应内存，所以响应数据应保持稳定，只记录发送位置：

```text
data[0 ... offset - 1]       // 已发送
data[offset ... length - 1] // 尚未发送
```

剩余长度统一按下面的方式计算：

```c
int remaining = output.length - output.offset; // 尚未发送的响应长度
```

下一次发送地址则是：

```c
output.data + output.offset // 跳过已经发送成功的字节
```

## 这不是 POSIX API 的缺陷

`send()` 不保证一次发送完，并不是 POSIX 忘记处理“半包”，而是职责划分的结果。

TCP 提供的是可靠、有序的字节流：

```text
应用层 output
    ↓ send
本机内核 TCP 发送缓冲区
    ↓ 网络
对端内核 TCP 接收缓冲区
    ↓ recv
对端应用程序
```

`send()` 成功返回的数字表示：

```text
本次有多少字节被本机内核接收
```

它并不表示：

```text
对端应用程序已经一次性读到了多少字节
```

如果应用层希望发送 1000 字节，而内核发送缓冲区暂时只能容纳 300 字节，那么 `send()` 可以返回 300。剩余的 700 字节必须由应用程序保存并再次发送。

操作系统不能替应用无限保存剩余数据，因为不同应用可能选择不同策略：

```text
继续缓存并稍后发送
丢弃低优先级响应
限制慢客户端占用的内存
连接超时后主动关闭
合并、压缩或重新编码响应
```

因此 POSIX 提供机制，应用层负责策略：

```text
POSIX：返回本次实际完成的字节数
应用：维护 length、offset 和错误处理策略
```

## Reactor 的响应续发

Reactor 使用 epoll 驱动状态转换。

需要先理解两个事件：

```text
EPOLLIN   // 当前连接可以读取，请调用 recv
EPOLLOUT  // 当前连接可以写入，请调用 send
```

正常流程是：

```text
监听 EPOLLIN                         // 等待客户端请求
    ↓
recv_cb                              // 请求追加到 input
    ↓
kvs_request                          // 解析请求并生成 output
    ↓
监听 EPOLLOUT                        // 等待可以发送响应
    ↓
send_cb                              // 从 output.offset 开始发送
    ↓
未发送完 → 继续监听 EPOLLOUT         // 下次进入 send_cb 续发
发送完成 → 清空 output，监听 EPOLLIN  // 等待下一批请求
```

### epoll 的基本组成

创建 epoll 实例后，可以把它理解为维护了两类列表：

```text
关注列表  // 应用希望监听哪些 fd、哪些事件
就绪列表  // 内核已经发现哪些 fd 可以读写
```

事件描述结构体主要包含：

```c
struct epoll_event {
    uint32_t events;    // EPOLLIN、EPOLLOUT 等事件类型
    epoll_data_t data;  // 与事件绑定的用户数据
};
```

本项目使用 fd 识别连接：

```c
struct epoll_event ev;
ev.events = event; // 设置希望监听的事件
ev.data.fd = fd;   // 事件发生后，通过 fd 找回连接
```

`set_event()` 最终整理成：

```c
int set_event(int fd, int event, int flag) {
    struct epoll_event ev;
    ev.events = event; // 设置读或写事件
    ev.data.fd = fd;   // 绑定客户端 fd

    if (flag) {
        return epoll_ctl(epfd, EPOLL_CTL_ADD, fd, &ev); // 第一次注册
    }

    return epoll_ctl(epfd, EPOLL_CTL_MOD, fd, &ev);     // 修改已有监听
}
```

这里 `flag` 的含义是：

```text
flag != 0  // 使用 EPOLL_CTL_ADD，将 fd 首次加入 epoll
flag == 0  // 使用 EPOLL_CTL_MOD，切换已有 fd 的事件
```

### Reactor 生成响应

`kvs_request()` 不再写入旧的 `wbuffer`，而是写入 `output`：

```c
int kvs_request(struct conn *c) {
    int consumed_length = 0; // 协议层实际消费的请求长度

    c->output.offset = 0;    // 新响应尚未发送

    c->output.length = kvs_handler(
        c->input.data,        // 已累计的请求数据
        c->input.length,      // 请求数据总长度
        c->output.data,       // 响应写入输出缓冲区
        BUFFER_LENGTH,        // 输出缓冲区容量
        &consumed_length      // 返回已经处理的请求字节数
    );

    if (c->output.length < 0) {
        return -1;            // 协议处理失败
    }

    if (kvs_input_buffer_consume(&c->input, consumed_length) < 0) {
        c->output.length = 0; // 输入缓冲区状态异常
        return -1;
    }

    return 0;
}
```

### Reactor 从偏移位置继续发送

`send_cb()` 的核心逻辑变为：

```c
int remaining =
    conn_list[fd].output.length -
    conn_list[fd].output.offset; // 计算剩余响应长度

if (remaining > 0) {
    count = send(
        fd,
        conn_list[fd].output.data +
            conn_list[fd].output.offset, // 从上次停止位置继续
        remaining,                       // 只发送剩余响应
        0
    );
}
```

成功发送部分字节后推进进度：

```c
if (count > 0) {
    conn_list[fd].output.offset += count; // 累加本次实际发送长度

    if (conn_list[fd].output.offset <
        conn_list[fd].output.length) {
        set_event(fd, EPOLLOUT, 0);       // 没发完，继续等待可写
        return count;                     // 结束本次回调
    }

    conn_list[fd].output.length = 0;      // 完整响应发送完成
    conn_list[fd].output.offset = 0;      // 重置发送进度
}

set_event(fd, EPOLLIN, 0);                // 重新等待客户端请求
```

### Reactor 的临时错误

非阻塞 `send()` 可能暂时无法发送：

```c
if (count < 0) {
    if (errno == EAGAIN ||
        errno == EWOULDBLOCK ||
        errno == EINTR) {
        set_event(fd, EPOLLOUT, 0); // 响应仍在 output 中，稍后继续
        return 0;                   // offset 保持不变
    }

    epoll_ctl(epfd, EPOLL_CTL_DEL, fd, NULL); // 先移除监听
    close(fd);                                // 再关闭 fd
    conn_list[fd].output.length = 0;          // 清理响应状态
    conn_list[fd].output.offset = 0;
    return -1;
}
```

这里最容易犯的错误是：`send()` 返回 `EAGAIN` 后直接切换成 `EPOLLIN`。

错误路径会变成：

```text
响应尚未发完
    ↓
send 返回 EAGAIN
    ↓
错误切换到 EPOLLIN
    ↓
剩余响应仍在 output，却没有下一次发送机会
```

正确做法是保留 `offset`，继续监听 `EPOLLOUT`。

## NtyCo 的响应续发

NtyCo 的网络接口看起来仍然像同步的 `recv()` 和 `send()`，但等待 I/O 时会让出当前协程。因此它不需要像 Reactor 一样手动将事件切换成 `EPOLLOUT`，可以在当前协程中循环发送。

每个 `server_reader()` 对应一个客户端连接：

```c
void server_reader(void *arg) {
    int fd = *(int *)arg;                  // 当前客户端 fd
    int ret = 0;                           // recv/send 的实际结果
    kvs_input_buffer_t input = {0};        // 跨循环保存请求数据
    kvs_output_buffer_t output = {0};      // 跨循环保存响应状态
```

接收时仍然追加到请求尾部：

```c
int available = BUFFER_LENGTH - input.length; // 计算剩余请求空间

ret = recv(
    fd,
    input.data + input.length,                  // 从已有请求尾部继续写
    available,
    0
);

if (ret > 0) {
    input.length += ret;                        // 不能写成 input.length = ret
}
```

协议层生成响应：

```c
output.offset = 0; // 新响应从开头发送

output.length = kvs_handler(
    input.data,         // 已累计请求
    input.length,       // 请求总长度
    output.data,        // 响应输出位置
    BUFFER_LENGTH,      // 响应容量
    &consumed_length    // 已处理的请求长度
);
```

如果 `output.length == 0`，说明尚未发现一条完整的 CRLF 命令：

```c
if (output.length == 0) {
    continue; // input 保持不变，继续 recv 后续半包
}
```

发现完整命令后，先消费已经处理的请求：

```c
if (kvs_input_buffer_consume(&input, consumed_length) < 0) {
    close(fd); // 输入缓冲区状态异常
    break;
}
```

再循环发送完整响应：

```c
while (output.offset < output.length) {
    int remaining = output.length - output.offset; // 计算剩余长度

    ret = send(
        fd,
        output.data + output.offset,               // 从发送断点继续
        remaining,
        0
    );

    if (ret > 0) {
        output.offset += ret;                       // 保存发送进度
        continue;                                   // 仍未结束则继续循环
    }

    if (ret < 0 && errno == EINTR) {
        continue;                                   // 被信号打断，原位置重试
    }

    close(fd);                                      // 其他错误关闭客户端
    return;                                         // 结束整个 server_reader
}

output.length = 0;                                  // 响应全部发送完成
output.offset = 0;                                  // 为下一次响应重置状态
```

这里使用 `return` 而不是 `break` 很重要。代码此时处在发送内层循环中：

```text
break   // 只退出发送循环，可能继续在已关闭 fd 上 recv
return  // 结束整个客户端协程
```

## Proactor / io_uring 的响应续发

Proactor 与前两种模型不同。它提交异步操作，然后通过 CQE 得知实际完成结果。

```text
准备 SQE
    ↓
提交异步 send
    ↓
程序继续处理其他事件
    ↓
内核产生 WRITE CQE
    ↓
entries->res 返回实际发送结果
```

### 为什么 Proactor 必须保存每个连接的状态

提交操作和收到完成通知发生在不同时间，因此使用 fd 找回对应连接：

```c
static struct conn *proactor_connections[CONNECTION_SIZE];
```

注意：数组中的每个元素不是 `struct conn` 对象，而是 `struct conn *` 指针。

建立连接时真正分配对象：

```c
struct conn *connection = calloc(1, sizeof(*connection)); // 创建连接状态

if (connection == NULL) {
    close(connfd);                                        // 分配失败
    continue;
}

connection->fd = connfd;                                  // 保存客户端 fd
proactor_connections[connfd] = connection;                // 建立 fd 到状态的映射
```

收到 READ 或 WRITE CQE 后，通过 fd 找回来：

```c
struct conn *connection =
    proactor_connections[result.fd]; // 取回已有对象，不是重新创建
```

连接关闭时才释放：

```c
close(result.fd);                         // 关闭 fd
free(connection);                         // 释放堆上的连接对象
proactor_connections[result.fd] = NULL;   // 清除映射，避免悬空指针
```

### READ CQE 生成响应

在 `EVENT_READ` 中：

```c
int ret = entries->res; // 本次异步 recv 实际接收的字节数

connection->input.length += ret; // 新数据已经写在旧数据尾部

int consumed_length = 0;
connection->output.offset = 0;   // 新响应尚未发送

connection->output.length = kvs_handler(
    connection->input.data,      // 当前请求数据
    connection->input.length,    // 当前请求总长度
    connection->output.data,     // 响应写入这里
    BUFFER_LENGTH,               // 响应容量
    &consumed_length             // 返回已消费请求长度
);
```

如果生成了响应，提交第一次发送：

```c
if (set_event_send(
        &ring,
        result.fd,
        connection->output.data,   // 第一次从 data[0] 发送
        connection->output.length, // 提交完整响应长度
        0
    ) < 0) {
    close(result.fd);               // 无法取得 SQE
    free(connection);
    proactor_connections[result.fd] = NULL;
    continue;
}
```

### WRITE CQE 推进响应进度

在 `EVENT_WRITE` 中：

```c
int ret = entries->res; // 本次异步 send 实际完成的字节数
```

这里的 `ret` 与其他几个长度含义不同：

```text
ret                     // 一次异步 recv/send 实际完成多少字节
output.length           // 完整响应总长度
output.offset           // 已经发送成功的总长度
consumed_length         // 协议层已经处理的请求长度
```

发送成功后累加：

```c
connection->output.offset += ret; // 记录本次 WRITE CQE 的发送量
```

如果仍未发送完，就再次准备 SQE：

```c
if (connection->output.offset < connection->output.length) {
    int remaining =
        connection->output.length -
        connection->output.offset; // 计算剩余响应长度

    if (set_event_send(
            &ring,
            result.fd,
            connection->output.data +
                connection->output.offset, // 从发送断点继续
            remaining,
            0
        ) < 0) {
        close(result.fd);                    // 无法准备新的发送 SQE
        free(connection);
        proactor_connections[result.fd] = NULL;
    }

    continue; // 等待下一次 WRITE CQE，不提交 recv
}
```

响应全部完成后才重新接收请求：

```c
connection->output.length = 0; // 清理完整响应长度
connection->output.offset = 0; // 清理发送进度

int available =
    BUFFER_LENGTH - connection->input.length; // 请求缓冲区剩余空间

set_event_recv(
    &ring,
    result.fd,
    connection->input.data + connection->input.length,
    available,
    0
);
```

### io_uring 的错误表示方式

普通 POSIX `send()` 失败时通常表现为：

```text
返回 -1
errno = EAGAIN
```

io_uring 的 CQE 不通过当前线程的 `errno` 返回错误，而是直接把负错误码放进：

```c
entries->res
```

因此 Proactor 判断的是：

```c
if (ret == -EAGAIN || ret == -EINTR) {
    // offset 没有增加，从原来的位置重新提交 send
}
```

两种写法不能混用：

```text
Reactor：errno == EAGAIN
Proactor：ret == -EAGAIN
```

### SQE 辅助函数补全返回值

严格编译还暴露了三个旧问题：

```text
set_event_send
set_event_recv
set_event_accept
```

它们声明返回 `int`，但原来没有返回值，也没有检查 `io_uring_get_sqe()` 是否失败。

以发送为例，整理为：

```c
int set_event_send(
    struct io_uring *ring,
    int sockfd,
    void *buf,
    size_t len,
    int flags
) {
    struct io_uring_sqe *sqe = io_uring_get_sqe(ring);

    if (sqe == NULL) {
        return -1; // 提交队列暂时没有空闲 SQE
    }

    struct conn_info event_info = {
        .fd = sockfd,
        .event = EVENT_WRITE,
    };

    io_uring_prep_send(sqe, sockfd, buf, len, flags); // 只准备操作
    memcpy(&sqe->user_data, &event_info, sizeof(event_info));

    return 0; // 只表示 SQE 准备成功，不表示响应已经发送
}
```

这里必须区分两个阶段：

```text
set_event_send 返回 0  // SQE 已经准备好
WRITE CQE 的 ret       // send 最终实际完成的字节数
```

## 三种网络模型的共同状态与不同驱动方式

三种实现现在共享同一套应用层状态：

```text
output.data    // 完整响应
output.length  // 响应总长度
output.offset  // 已发送长度
```

但推进发送的方式不同：

| 网络模型 | 谁通知可以继续 | 如何继续 |
| --- | --- | --- |
| Reactor | epoll 返回 `EPOLLOUT` | 再次进入 `send_cb()` |
| NtyCo | 协程 I/O 封装恢复当前协程 | 在 `while` 中再次 `send()` |
| Proactor | io_uring 返回 WRITE CQE | 根据 `entries->res` 重新提交 SQE |

可以统一概括为：

```text
生成响应                           // length 已知，offset = 0
    ↓
尝试发送                           // 每次可能只完成一部分
    ↓
offset += 实际发送长度              // 保存进度
    ↓
offset < length                    // 未完成
    └── 按网络模型的方式安排下一次发送
    ↓
offset == length                   // 完成
    ↓
重置 output                        // 回到请求接收状态
```

## 为什么客户端可能仍然只显示一条响应

即使服务端执行了四次 `send()`：

```text
3 字节 + 3 字节 + 3 字节 + 1 字节
```

Packet Sender 也可能一次显示完整的：

```text
NO EXIST\r\n
```

这是正常的。TCP 可以在传输和接收过程中重新组合字节：

```text
服务端多次 send
    ↓
TCP 字节流传输
    ↓
客户端一次或多次 recv
```

因此不能只看客户端显示了几个 packet 来判断服务端是否真的发生 partial send。需要观察服务端记录的每次实际发送长度和 `offset`。

## 强制构造 partial send 测试

项目当前响应缓冲区只有 1024 字节，本机 TCP 发送缓冲区通常足以一次接收整段响应。为了确定续发状态机真的工作，测试时临时将每次发送限制为 3 字节。

以长度为 10 的响应为例：

```text
NO EXIST\r\n
```

测试期间临时使用：

```c
int send_length = remaining > 3 ? 3 : remaining; // 每次最多发送 3 字节
```

并打印每次发送进度。三种网络模型均得到：

```text
第一次：ret = 3，offset = 3，length = 10
第二次：ret = 3，offset = 6，length = 10
第三次：ret = 3，offset = 9，length = 10
第四次：ret = 1，offset = 10，length = 10
```

### NtyCo 强制分段结果

```text
partial send: ret=3, offset=3, length=10
partial send: ret=3, offset=6, length=10
partial send: ret=3, offset=9, length=10
partial send: ret=1, offset=10, length=10
```

### Reactor 强制分段结果

```text
reactor partial send: count=3, offset=3, length=10
reactor partial send: count=3, offset=6, length=10
reactor partial send: count=3, offset=9, length=10
reactor partial send: count=1, offset=10, length=10
```

每一行都对应一次独立的 `EPOLLOUT → send_cb()`。

### Proactor 强制分段结果

```text
proactor partial send: ret=3, offset=3, length=10
proactor partial send: ret=3, offset=6, length=10
proactor partial send: ret=3, offset=9, length=10
proactor partial send: ret=1, offset=10, length=10
```

每一行都对应一次独立的 `send SQE → WRITE CQE`。

三种模型的客户端最终都收到了完整响应：

```text
NO EXIST\r\n
```

这证明 `offset` 不是形式上的字段，而是真正参与了响应断点续发。

测试完成后，删除了所有临时限制和日志：

```text
删除 send_length = min(remaining, 3)
恢复真实 remaining 长度
删除 partial send 调试输出
恢复默认 NETWORK_SELECT = NETWORK_NTYCO
```

测试代码没有进入正式提交。

## 严格编译与完整构建

三个网络模型分别通过：

```bash
make -B \
  build/reactor.o \
  build/ntyco.o \
  build/proactor.o \
  CFLAGS="-std=gnu11 -Wall -Wextra -Werror -g"
```

随后执行完整构建：

```bash
make -B
```

最终成功生成：

```text
bin/kvstore
```

提交前还使用 CRLF-aware 方式检查行尾：

```bash
git -c core.whitespace=blank-at-eol,blank-at-eof,space-before-tab,cr-at-eol \
  diff --check
```

## 本轮提交与合并

响应续发提交：

```text
4fa3d83  feat(network): 支持三种网络模型的响应分段续发
```

提交说明覆盖：

```text
引入统一输出缓冲区
Reactor 保持 EPOLLOUT 并处理临时错误
NtyCo 在协程中循环发送
Proactor 根据 WRITE CQE 更新偏移并重新提交 SQE
避免 send 未完整写入时丢失响应尾部
```

功能分支最终通过 merge commit 合并进 `main`：

```text
bb49955  merge: 集成批量命令与网络分包处理
```

GitHub 和 GitLab 的 `main` 均已推进到：

```text
bb49955
```

需要再次强调：merge 只会合并提交历史，不会自动删除功能分支。`feature/batch-commands` 仍然存在是正常现象，可以在确认主分支稳定后再清理。

## 本章最终完成情况

本章已经不只是“批量解析两条命令”，而是完成了 TCP 应用层协议所需的一条完整链路：

```text
CRLF 定义命令边界                     // 协议帧边界
    ↓
批量解析一段缓冲区中的多条完整命令      // 解决粘包
    ↓
input 跨 recv 保存不完整命令            // 解决请求拆包
    ↓
consumed_length 精确消费已处理请求       // 保留末尾半包
    ↓
output 保存完整响应和发送进度            // 解决响应 partial send
    ↓
Reactor / NtyCo / Proactor 分别续发       // 适配三种网络模型
```

最终不再依赖下面两个错误假设：

```text
一次 send 等于对方的一次 recv
一次 recv 等于一条完整命令
```

而是建立了真正符合 TCP 字节流语义的处理方式：

```text
请求按照协议边界累计和拆分
响应按照实际发送长度记录和续发
```

## 后续仍值得改进的方向

本轮目标已经完成，但继续演进时仍有几个清晰方向：

```text
连接关闭逻辑统一封装              // 减少 close、free、置 NULL 的重复代码
所有 SQE 提交点检查失败返回        // 不让连接进入“无事件等待”的悬空状态
补全 recv 负错误码处理            // 区分临时错误与不可恢复错误
为输出增加队列或动态扩容           // 支持超过固定缓冲区容量的大批量响应
客户端实现响应缓冲与 CRLF 拆分      // 客户端同样不能假设一次 recv 是一条响应
增加自动化协议与网络分段测试        // 替代手工修改每次发送 3 字节
增加批处理性能基准                 // 对比单命令往返和 pipeline 吞吐量
```

其中，下一轮最值得优先考虑的是自动化测试和批处理性能基准。只有测出：

```text
每秒完成命令数
平均延迟与尾延迟
不同 batch size 的吞吐变化
单连接与多连接的差异
```

才能真正回答最初的问题：批量命令究竟减少了多少网络往返开销，又将 CPU 利用率和系统吞吐提升了多少。
