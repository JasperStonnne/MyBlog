---
title: KV 进化（六）：命令协议改造
slug: kv-evolution-length-prefixed-protocol
description: 将 91kvstore 的按行文本命令改为 RESP 风格的长度帧，打通动态缓冲区、AOF、Snapshot 与主从复制
date: 2026-10-06T00:00:00+08:00
draft: false
image: cover.svg
tags:
  - KV 存储
  - 命令协议
  - RESP
  - TCP
  - AOF
  - 主从同步
  - C 语言
categories:
  - 项目记录
---

实现命令协议改造

> 行文至此，已经接近燃尽了。

上一篇完成 Primary / Replica 主从同步后，91kvstore 已经有了客户端命令、AOF、Snapshot、全量同步和在线增量复制。回头看，原来的 `SET key value` 不只是客户端输入格式，AOF、Snapshot 和复制增量也都依赖这种按空格拆字段、按行拆命令的表示。

这一轮的需求是让 key 和 value 支持原本用来分隔字段的特殊字符，并且不再受 1024 字节固定网络缓冲区限制。表面上是“改协议”，实际上需要逐层找出对空格、换行和固定容量的假设。

```text
客户端输入
    ↓
协议编解码
    ↓
命令执行
    ↓
内存引擎 + AOF
    ↓
Snapshot / 主从复制
```

## 原协议的问题

旧命令是一行文本：

```text
SET name Jasper\r\n
GET name\r\n
```

网络层寻找 `\r\n`，执行层调用 `strtok(msg, " ")`。只要 key 是 `user name`，或者 value 是 `hello world`，空格既可能是数据，又可能是分隔符，接收端无法恢复用户本来的意思。

换行也一样。如果 value 本身包含 `\r\n`，逐行拆命令可能在 value 中途结束。即使改好了解析器，网络层固定的 `BUFFER_LENGTH = 1024` 仍会让较长命令在接收时耗尽空间。

所以要解决的是三个边界：

```text
字段边界：不能靠空格
命令边界：不能靠数据中也可能出现的换行
缓冲区边界：不能固定在 1024 字节
```

## 先把旧链路画完整

改协议前，先确认一条命令是怎样走完生命周期的。否则只改网络入口，下一次重启就可能把数据还原错。

```text
客户端发送 "HSET user name hello world\r\n"
    ↓
网络后端调用 recv / nty_recv / io_uring recv
    ↓
输入缓冲区累计字节
    ↓
kvs_line_batch_protocol 找到行尾
    ↓
kvs_client_protocol
    ↓
kvs_execute_command 去掉行尾
    ↓
kvs_split_token 用 strtok 按空格切分
    ↓
kvs_filter_protocol 选择 HSET、HGET 等命令
    ↓
内存引擎写入；成功写命令追加到 AOF
```

旧 `kvs_split_token` 的关键部分大致是：

```c
char *token = strtok(msg, " ");
while (token != NULL) {
    tokens[idx++] = token;
    token = strtok(NULL, " ");
}
```

`strtok` 会改写 `msg`：它把找到的空格位置替换为 `\0`，让 `tokens[i]` 指向同一块报文中的不同位置。它并不知道哪个空格来自用户的 key/value，哪个空格是协议分隔符。

以 `HSET user name hello world` 为例，旧解析结果会趋向于：

```text
tokens[0] = HSET
tokens[1] = user
tokens[2] = name
tokens[3] = hello
tokens[4] = world
```

即使执行器只使用前三个 token，也已经把用户希望保存的 key/value 拆坏。更严重的是，命令执行层此时拿不到原始字段边界，无法靠后续逻辑“猜回来”。

这个问题不只发生在客户端请求。原来的 AOF 使用 `fprintf(aof_fp, "%s %s %s\n", ...)`，Snapshot 的每条键值记录也写成 `SET key value\n`。即使客户端入口暂时通过引号或转义规则把空格保住，落盘后再按空格拆分仍会丢失含义。

因此需求不能只表述成“网络层支持特殊字符”。更完整的要求是：

```text
同一组命令字段，经过网络输入、执行、AOF、Snapshot、复制、恢复后，
字段数量、每个字段的字节长度和内容都不能变化。
```

## 为什么不继续给空格加转义

另一种思路是让客户端把空格写成 `\ `，把换行写成 `\n`。这看起来比改整条协议轻，但会把复杂度转移到更多位置：

```text
反斜杠自己如何转义？
转义后的长度怎么算？
AOF 保存转义前还是转义后？
Snapshot 和主从是否用完全相同的规则？
收到半个转义序列时应等待还是报错？
```

如果用户数据原本就是 `\n` 两个普通字符，还要区分它与一个真正的换行字节。每增加一种特殊字符，编码与解码规则都会增长。

长度格式的核心规则更稳定：先知道后续内容有多少字节，然后**不解释内容中的分隔符**。字段内的空格、CRLF、反斜杠都只是内容。协议层只检查长度之外的结构标记。

## 一条命令与一个 TCP 包不是一回事

旧协议按行时已经碰到过 TCP 半包与粘包，新协议同样需要面对，只是判断完整性的方式改变。

假设一条 SET 帧共 40 字节。第一次 `recv` 可能只得到 7 字节，第二次得到 33 字节；也可能一次得到 40 字节和下一条 GET 的前 12 字节。

正确处理方法是：

```text
7 字节：解析器返回 0，缓冲区保留 7 字节
再到 33 字节：累计到 40，解析器返回 1，执行并消费 40 字节
若还有 GET 前半段：只保留 GET 的 12 字节，不提前执行
```

这里的“包”只是观察网络收发的方便说法。应用不能依赖一次 `recv` 的返回值代表一条命令，必须维护自己的输入缓冲区，并按协议决定何时消费。

## 本轮范围

选择 RESP 风格的长度格式：命令先声明字段数量，每个字段再声明自己的字节长度。这样可以保留文本 key/value 中的空格和换行，并让编解码、持久化和复制使用同一套格式。

“长度不受限制”在这里是指**不再被原来的固定网络缓冲区截住**，不是数学意义上的无限大。实际数据仍受可用内存、`size_t`、当前网络接口使用的 `int` 长度以及存储引擎限制。

本轮没有把内嵌 `\0` 的任意二进制数据、完整 RESP2 响应或 `redis-cli` 兼容放进完成条件。这些分别涉及 C 字符串引擎和更完整的客户端协议语义。

## 这轮实际按什么顺序推进

如果一上来就同时修改三个网络后端、命令执行、AOF、Snapshot 和复制，编译错误会混在一起，很难知道是哪一层的契约断了。这轮实际推进可以拆成下面几段；每一段都先明确“输入是什么、输出是什么”，再继续往下接。

| 阶段 | 主要修改 | 当时要确认的事情 |
| --- | --- | --- |
| 1 | `protocol.h`、`codec.c` | 单独给定字节数组，能区分完整、未完成、非法帧；能算出编码大小 |
| 2 | `kvstore.c` 的命令入口 | 长度字段能转换为旧执行器认识的 token，含空格的 SET/GET 能在内存中往返 |
| 3 | 输入缓冲区与三个网络后端 | 半帧留下、完整帧消费、下一帧继续处理，超过 1024 字节也能接收 |
| 4 | 输出缓冲区与 GET | 已存在的长 value 能被完整编码和发送；部分发送不覆盖未发完的数据 |
| 5 | AOF 写入、重放 | 含空格的命令落盘后重启，恢复出的字段不变 |
| 6 | Snapshot 保存、加载 | offset 头部保留，正文改用长度帧；全量恢复可读取长 value |
| 7 | 复制增量接入 | FULL_SYNC 仍传文件原始字节，CATCH_UP/ONLINE 识别长度格式 AOF 命令 |
| 8 | 单元与集成测试 | 编译、协议边界、重启恢复、全量与在线复制分别通过 |

这些阶段并不是八个彼此孤立的功能。比如阶段 3 的“输入可扩容”只保证长 SET 能进入服务；如果阶段 4 没跟上，长 GET 仍可能失败。阶段 5 的 AOF 格式改变又会立即影响阶段 7，因为 Primary 发给 Replica 的增量数据来自 AOF。前一步的输出，常常就是后一步的输入。

可以把最低可用闭环写成：

```text
客户端编码 → 网络累计 → 解析字段 → 命令执行 → 响应编码
                                      ├→ AOF 追加 → 重放 / 复制增量
                                      └→ 内存引擎 → Snapshot 保存 → 加载 / 全量复制
```

所以“SET 在当前进程返回 OK”只验证了图的前半截，不能作为整个改造完成的证据。

## 在独立分支开发

本轮使用 `feat/length-prefixed-protocol` 分支。初期提交 `b406726 WIP：接入长度前缀协议及动态缓冲区` 记录了基础接入；持久化、复制和测试收尾形成 `ef42eb6 完善长度前缀协议的持久化与主从复制支持`。两个提交已经推送到 GitHub 和 GitLab。

工作区曾有独立的 `third_party/NtyCo` 子模块变化，没有纳入这次协议提交。

## 新命令格式

以 key 为 `user name`、value 为 `hello world` 的 SET 为例，用可读的转义形式表示：

```text
*3\r\n
$3\r\nSET\r\n
$9\r\nuser name\r\n
$11\r\nhello world\r\n
```

`*3` 表示三个字段。`$9` 表示后面恰好读取九个**字节**，所以 `user name` 中间的空格属于 key；`$11` 后面的空格属于 value。这里的数字是字节长度，既不是“读到下个空格”，也不是 C 字符串终止符的位置。

例如 value 为 `a\r\nb` 时，内容长度是四字节：`a`、`\r`、`\n`、`b`。解析器按已声明的四字节读取内容，内容中的 CRLF 不能提前结束字段。

TCP 依然只是字节流。一次 `recv` 可以得到半条命令、一条命令，或者多条命令粘在一起。长度格式让应用有办法判断边界，不会改变 TCP 自身的行为。

## 先用同一条命令做新旧对照

把 key 设为 `user name`、value 设为 `hello world`。旧格式若直接写成：

```text
SET user name hello world\r\n
```

执行器看见的是一串靠空格分隔的 token，无法知道 `user name` 原本是一个 key。即使临时修改客户端解析，只要 AOF 继续写 `SET user name hello world\n`，重启时又会被拆坏。

新格式在各个环节的职责不同：

| 环节 | 接到什么 | 做了什么 | 交给下一层什么 |
| --- | --- | --- | --- |
| 客户端 | 三个独立字段 | 对每个字段写长度和原始内容 | 一条长度帧 |
| 网络后端 | 任意分段的 TCP 字节 | 累计到输入缓冲区 | 当前缓冲区地址和实际字节数 |
| 协议解析器 | 缓冲区中的字节 | 校验 `*`、`$`、十进制长度、CRLF | 三个 slice 与 `parsed_bytes` |
| 命令适配层 | 三个 slice | 复制字段并补 `\0`，仅在旧执行器边界转换 | `SET`、`user name`、`hello world` 三个 token |
| 引擎 | key、value 两个 C 字符串 | 写入内存 | 执行结果 |
| AOF | 成功写命令的三个 token | 重新按长度帧编码并追加 | 以后可无歧义重放的字节 |
| Snapshot | 引擎遍历得到的 key/value | 写 offset 头部，再写长度帧正文 | 可作为全量复制的文件 |
| Replica | Snapshot 文件块或 AOF 增量 | 由复制状态决定安装文件还是解析命令 | 同一组 key/value |

这里有一个容易忽略的细节：**协议解析器没有直接执行 SET**。它只回答“字节流里是否已经有一条完整命令”“字段分别在哪里”“应消费多少字节”。`kvstore.c` 再决定这条命令的业务含义。这样，AOF 重放和 Replica 收到增量命令时，可以复用同一个字节解析器，却以 `RECOVERY` 或 `REPLICATION` 来源执行。

更直观地说，同一条数据在四个位置的形式并不完全相同：

```text
网络/AOF/Snapshot 正文：*3\r\n$3\r\nSET\r\n$9\r\nuser name\r\n$11\r\nhello world\r\n
解析器输出：           ("SET",3), ("user name",9), ("hello world",11)
旧执行器输入：         "SET\0", "user name\0", "hello world\0"
内存引擎内容：         key="user name", value="hello world"
```

同样是 `\r\n`，在字段声明之后、字段内容之外，它是协议结构；出现在字段内容内部时，它是用户数据。解析器依赖长度而不是靠肉眼或 `strtok` 区分两者。

## 公共协议模块

新增 `include/protocol.h` 和 `src/protocol/codec.c`。协议模块不关心 NtyCo、epoll、io_uring 或具体存储引擎；它只关心输入字节、字段长度和帧边界。

```c
typedef struct {
    const char *data;
    size_t length;
} kvs_slice_t;
```

`kvs_slice_t` 只引用原始输入中的一段字节，不分配、不释放，也不保证 `data` 后面有 `\0`。因此在字段仍被使用时，输入缓冲区不能被移动或释放。网络层处理完命令、按已消费字节数整理缓冲区后，旧 slice 就不能继续保存使用。

三个网络后端复用同一个协议入口：

```text
NtyCo / Reactor / Proactor
          ↓ 收到字节
       输入缓冲区
          ↓
   kvs_parse_command()
          ↓
        字段数组
          ↓
        命令执行
```

协议解析器只写一份；三个后端各自处理收发、等待、扩容和连接生命周期。

## parse_number、parse_field 和 parse_command

解析过程按格式层次拆开：

```text
parse_number       读取十进制数字以及随后的 \r\n
parse_field        读取 $长度\r\n内容\r\n
kvs_parse_command  读取 *字段数量\r\n，再读取指定数量的字段
```

`parse_number` 和 `parse_field` 是 `codec.c` 内部的 `static` 函数，不需要在公共头文件暴露。读取数字时要检查 `value * 10 + digit` 是否会超过 `SIZE_MAX`；字段解析时先检查剩余字节，再计算内容与结尾位置，避免超大长度造成偏移溢出。

最外层 `kvs_parse_command` 从输入开头尝试解析**第一条**命令：

```text
返回  1：第一条命令完整
返回  0：还缺字节，等待更多输入
返回 -1：格式错误
```

调用方准备 `fields` 数组并给出容量。成功后，`field_count` 是字段数量，`parsed_bytes` 是这条完整命令从输入开头占用的字节数。

假如一次接收到 `[完整 SET][半条 GET]`，解析器只报告 SET 的字段和 `parsed_bytes`。网络层消费这段前缀，保留后面的半条 GET，等下一次接收。这里不能用 `strlen(input)`：TCP 收到的是“地址 + 当前字节数”，输入未必以 `\0` 结尾。

## 解析函数的参数逐个看

接口最重要的不是函数名，而是它和调用者之间的契约。可以把 `kvs_parse_command` 读成：

```c
int kvs_parse_command(
    const char *input,
    size_t input_length,
    kvs_slice_t *fields,
    size_t field_capacity,
    size_t *field_count,
    size_t *parsed_bytes
);
```

各参数的责任：

| 参数 | 谁提供 | 含义 |
| --- | --- | --- |
| `input` | 调用方 | 当前缓冲区中尚未处理的数据起点 |
| `input_length` | 调用方 | 当前实际收到的字节数，不是缓冲区容量 |
| `fields` | 调用方 | 用于接收解析结果的 slice 数组 |
| `field_capacity` | 调用方 | `fields` 最多能放几个元素，防止越界写入 |
| `field_count` | 解析器输出 | 第一条完整命令实际有几个字段 |
| `parsed_bytes` | 解析器输出 | 第一条完整命令占用了多少输入字节 |

`input_length` 和 `field_capacity` 容易混淆。前者描述网络已经给了多少原始字节，后者描述输出字段数组有多少个位置。两个容量都必须检查，但检查的是不同对象。

`parsed_bytes` 也不是“解析了几个字段”。例如两条完整命令粘在同一个缓冲区：

```text
input = [命令 A 的 46 字节][命令 B 的 25 字节]
input_length = 71
```

第一次调用只返回命令 A 的字段，`parsed_bytes = 46`。调用方把前 46 字节消费掉，再用剩余 25 字节进行下一次处理。如果把整个 `input_length = 71` 都消费掉，命令 B 会静默丢失。

## parse_number 为什么先用局部 pos

数字函数只识别类似 `123\r\n` 的片段，不读取字段正文。它从调用方给的 `*offset` 出发，但开始时先复制到局部变量：

```c
size_t pos = *offset;
size_t value = 0;
```

然后逐个处理十进制字符：

```c
size_t digit = (size_t)(input[pos] - '0');
if (value > (SIZE_MAX - digit) / 10) {
    return -1;
}
value = value * 10 + digit;
pos++;
```

检查式来自 `value * 10 + digit <= SIZE_MAX`。移项后，只要 `value > (SIZE_MAX - digit) / 10`，下一步就一定溢出。不能先计算、再看结果有没有变小，因为无符号溢出后的值已经不是用户声明的长度。

数字读完还要确认后面是 `\r\n`：

```text
数字后没有任何字节：返回 0，等待
只有 \r，没有 \n：返回 0，等待
数字后不是 \r：返回 -1，格式错
\r 后不是 \n：返回 -1，格式错
```

只有 `数字\r\n` 全部到齐时，才把结果写进 `*number`，把 `*offset` 更新为后一个位置，返回 1。半包时不推进调用方的 offset，这让上层可以安全地从同一位置重新尝试。

## parse_field 如何区分“还没到齐”和“格式错误”

字段形式是：

```text
$<length>\r\n<length 个正文原始字节>\r\n
```

关键逻辑可以按顺序读：

```c
if (pos >= input_length) return 0;
if (input[pos] != '$') return -1;
pos++;

size_t length = 0;
int result = parse_number(input, input_length, &pos, &length);
if (result <= 0) return result;

size_t remaining = input_length - pos;
if (length > remaining || remaining - length < 2) return 0;

if (input[pos + length] != '\r' ||
    input[pos + length + 1] != '\n') {
    return -1;
}
```

`remaining` 是当前位置之后实际收到多少字节。如果声明正文 2048 字节，而现在只有 300 字节，应该返回 0，不能把它判成坏命令。即使正文已经到齐，还需要两个字节作为字段结尾；所以 `remaining - length < 2` 时同样等待。

这段代码先确认 `length <= remaining`，再做 `remaining - length` 和 `pos + length`。顺序很重要：若反过来先把一个极大的 length 加到 pos 上，可能造成整数溢出或越界访问。

在字段完整时才写：

```c
field->data = input + pos;
field->length = length;
*offset = pos + length + 2;
```

`data` 指向正文第一个字节，而不是 `$`。`length` 不包括前缀和尾部 CRLF。`offset` 更新到下一字段或下一条命令的起点。

## parse_command 为什么只解析第一条

最外层先检查输入第一个字节是 `*`，再用 `parse_number` 得到字段个数。如果字段个数是 0，或者超过调用方 `fields` 的容量，就拒绝。接着循环调用 `parse_field`，每次成功才推进到下一字段。

只有全部字段完整后，才写出最终的 `field_count` 和 `parsed_bytes`。如果中途返回 0，调用方应继续保留输入并等待网络数据；不应把 `fields` 中可能已经写过的前几个元素当作一条可执行命令。

这个函数不执行 SET/GET，不识别具体引擎，也不分配 key/value 的长期存储。它只把字节流解释成第一条完整命令的字段视图。

## 空字段、零字节与内嵌 NUL

协议格式本身能够表达 `$0\r\n\r\n`，即长度为零的字段；是否允许空 key 或空 value 是命令层的决定，不能只由协议解析器替业务决定。

协议视图同样能够指向值为 `0x00` 的字节，但后续 C 字符串适配层会拒绝内嵌 NUL。这个差别体现出两个不同层次：

```text
协议可以描述哪些字节
业务引擎当前能保存哪些数据
```

把协议做成 `slice(data, length)`，是为将来支持二进制数据保留接口空间；并不意味着这一轮已经完成四种引擎的二进制改造。

## 编码器为什么也要共用

解析器只负责读。客户端请求、AOF、Snapshot 和复制增量还需要以同样的规则写出命令。如果它们各自手拼 `*`、`$`、长度数字和 CRLF，格式很容易分叉。

`kvs_encoded_command_size()` 先计算总字节数，`kvs_encode_command()` 再写入调用方提供的输出空间。长度数字本身也占空间，例如 9 是一位，2048 是四位，因此不能把字段内容长度直接当作编码后总长度。

GET 类长度响应也采用先计算、确认容量、再编码的方式。例如长度格式 GET 的结果可以是：

```text
$11\r\nhello world\r\n
```

当前 SET 成功仍返回旧式 `OK\r\n`，不是 RESP2 的 `+OK\r\n`。所以“请求使用 RESP 风格数组和 bulk string”并不等于“服务端响应已经全面兼容 Redis”。

## 一条 SET 帧到底占多少字节

以 `SET`、`user name`、`hello world` 三个字段为例，命令总长度不是 `3 + 9 + 11 = 23`。还要算字段数、长度数字、`*`、`$` 与各处 CRLF：

| 部分 | 字节数 | 计算 |
| --- | ---: | --- |
| `*3\r\n` | 4 | `*` 1 + 数字 1 + CRLF 2 |
| `$3\r\nSET\r\n` | 9 | `$` 1 + 数字 1 + CRLF 2 + 内容 3 + CRLF 2 |
| `$9\r\nuser name\r\n` | 15 | 1 + 1 + 2 + 9 + 2 |
| `$11\r\nhello world\r\n` | 18 | 1 + 2 + 2 + 11 + 2 |
| **合计** | **46** | 4 + 9 + 15 + 18 |

这解释了手工 `xxd` 示例中第一条 SET 写入后，Snapshot 的 `AOF_OFFSET` 是 46：offset 记录的是 AOF 的**字节位置**，不是写了几条命令。

大小计算函数的思路可以写成：

```c
size_t total = 1 + decimal_digits(field_count) + 2;

for (size_t i = 0; i < field_count; i++) {
    size_t overhead = 1 + decimal_digits(fields[i].length) + 4;
    /* 先检查 total + overhead + 字段正文是否会溢出。 */
    total += overhead + fields[i].length;
}
```

上面的 `4` 是每个字段前后两组 CRLF，共四字节。实际实现还需要检查每次相加是否超过 `SIZE_MAX`，并确认字段指针有效。`decimal_digits(0)` 也应返回 1，因为长度为零时仍要把字符 `0` 写出来。

把编码大小算出来之后再分配：

```text
计算 required
    ↓
分配恰好需要的 required 字节
    ↓
kvs_encode_command 写入 output
    ↓
拿到 encoded_bytes，按这个长度发送或 fwrite
```

编码器生成的是协议字节，不要求末尾附加 C 字符串的 `\0`。对二进制安全的网络发送和文件写入来说，真正重要的是 `encoded_bytes`，不是 `strlen(output)`。

## GET 的大小为什么也要先计算

旧 GET 习惯写 `sprintf(response, "%s\r\n", value)`。`sprintf` 不知道输出缓冲区容量。如果 value 有 2048 字节，而输出只有 1024 字节，问题已经不是“显示不完整”，而可能是越界写入。

长度响应的大小可以直接由 value 长度算出。例如 2048 字节 value：

```text
$2048\r\n<2048 字节 value>\r\n
```

总大小是 `1 + 4 + 2 + 2048 + 2 = 2057` 字节。输出缓冲区先保证至少有 2057 字节，再调用编码函数写入。`kvs_encoded_value_response_size` 和 `kvs_encode_value_response` 正是为了把“算大小”和“写字节”分别交给协议模块。

响应中的 `$2048` 属于协议内容，不是 value 的一部分。使用 `wc -c` 看网络输出时，统计到的是**整个响应帧**，不是纯 value 长度，所以数值比 value 的字节数大是正常的。

## 现有命令执行器如何接入

解析结果是 `data + length`，但现有 Array、RBTree、Hash、SkipList 引擎使用 `char *`，并在键比较、复制数据等位置依赖 `strlen`、`strcmp`。本轮用适配层把每个 slice 复制成 `length + 1` 字节的临时 C 字符串，在末尾补 `\0`，然后交给原有命令执行器。执行后释放这些临时 token。

```text
长度格式字段
    ↓ kvs_fields_to_tokens
临时的、以 \0 结尾的 token
    ↓ kvs_execute_tokens
原有命令执行逻辑
```

旧行格式和新长度格式最终共用 `kvs_execute_tokens`。这避免为同一个 SET、GET、DEL、MOD 维护两份业务规则，也让 Replica 只读拦截与 AOF 追加规则继续集中在原来的执行路径。

这里的代价是：字段内容一旦含有内嵌 `\0`，C 字符串接口就不能完整表达它。当前转换层会拒绝这种字段。若以后需要真正的二进制 key/value，不能只改解析器；执行器、四种引擎、AOF、Snapshot 和响应都需要持续携带长度。

## 为什么先做适配，而不是立刻重写四个引擎

理想中的二进制 KV 引擎会始终保存 `key_data + key_length` 和 `value_data + value_length`，比较 key 时用长度与 `memcmp`，而不是用 `strcmp`。但现有四个引擎都建立在 C 字符串语义上；一次把它们、AOF、Snapshot 和复制改成完全二进制接口，会让当前需求的改动范围进一步扩大。

本轮的要求是包含空格与换行的**文本** key/value。对不含内嵌 NUL 的文本来说，把 slice 复制为独立的 C 字符串仍可保留字段内容。于是用一个明确的边界适配现有引擎：

```c
/* 示意：fields[i] 本身不是以 \0 结尾的字符串。 */
tokens[i] = malloc(fields[i].length + 1);
memcpy(tokens[i], fields[i].data, fields[i].length);
tokens[i][fields[i].length] = '\0';
```

调用前需要检查 `fields[i].length + 1` 不溢出，并拒绝正文中的内嵌 `\0`。所有 token 必须在命令执行后释放；若第 N 个 token 分配失败，前面已经分配的也要清理。

这样做不等于 protocol 模块负责引擎内存。协议只返回 slice；边界适配负责临时 token；引擎在 SET 成功时仍按自己的规则复制需要长期保存的 key/value。

## SAVE、普通命令和命令来源

旧行格式和新长度格式最后共用 `kvs_execute_tokens`，因此 SAVE 这种特殊命令也能从同一个入口执行。对于普通命令，继续进入 `kvs_filter_protocol`，由命令名决定 Array、RBTree、Hash、SkipList 中的哪一路处理。

执行来源仍然要保留：

```text
CLIENT       普通客户端命令；Replica 上的写请求应被拒绝
RECOVERY     从 AOF / Snapshot 恢复；不能再次追加同一条 AOF
REPLICATION  Primary 的增量命令；Replica 可以执行并追加本地 AOF
```

如果只看命令文本，Replica 收到 `HSET` 时无法判断是普通客户端想越权写入，还是 Primary 正在同步数据。改协议时不能丢掉原来主从模块建立的命令来源信息。

## 为什么把四种 GET 的查找合到一起

项目有 `GET`、`RGET`、`HGET`、`SGET`，分别对应不同引擎。为新长度响应增加扩容时，如果在四个 `case` 中复制“算大小、扩容、编码”四次，以后修改响应格式就需要改四处。

这次先把“按命令类型查找 value”集中为 `kvs_get_value(cmd, key)`，再让新协议的 GET 路径共用 `kvs_encode_value_to_output`。职责变成：

```text
kvs_get_value         从对应引擎取出原始 C 字符串
kvs_encode_value_to_output
                      按长度格式生成响应，必要时扩输出缓冲区
```

查找函数不知道网络缓冲区，编码函数不知道 Array / Hash 的内部数据结构；这比把四种引擎和协议格式混在每个分支里更容易维护。

目前 `value_slice.length` 仍由 `strlen(value)` 得到。这是另一处内嵌 NUL 不受支持的明确信号，并不是编码器自身只能处理字符串。

## 动态输入缓冲区

旧连接缓冲区类似：

```c
char data[BUFFER_LENGTH];
int length;
```

即使解析器识别出“帧尚未收完整”，1024 字节的数组满了之后也没有地方接收剩余内容。改造后缓冲区同时保存 `data`、`length` 和 `capacity`：

```text
data      动态分配的内存地址
length    当前已接收且尚未消费的字节数
capacity  当前分配容量
```

接收前调用 `kvs_input_buffer_ensure_space` 保证至少有可写空间。不够时按需扩容。处理完命令后，`kvs_input_buffer_consume` 移走已执行的前缀，保留尚未构成完整命令的尾部。

```text
收到：[完整帧 A][半帧 B]
执行：A
消费：A 的 parsed_bytes
保留：[半帧 B]
下次接收：把 B 的剩余字节接在后面
```

输出也有相似接口：`kvs_output_buffer_ensure_space` 和 `kvs_output_buffer_free`。长 GET 响应要先算出 `$长度\r\n内容\r\n` 的字节数，再确保输出有足够容量。只改输入而不改输出，会出现“长 SET 能写入，长 GET 却不能完整返回”的不对称情况。

两个方向共享底层扩容逻辑，但调用时机不同。扩容使用 `realloc`，它可能原地扩张，也可能分配新块并移动数据。因此只能在 `realloc` 成功后更新地址和容量；失败时旧地址仍需保持有效。关闭连接时要释放输入和输出的动态内存。

## ensure_space 的不变量

扩容函数看起来只是 `realloc`，但运行前先要确认结构体状态可信：

```text
0 <= length <= capacity

未分配：data == NULL 且 capacity == 0
已分配：data != NULL 且 capacity > 0
```

若 `capacity > 0` 却 `data == NULL`，后面计算 `data + length` 会出错；若 `length > capacity`，所谓“剩余空间”已经没有意义。函数先拒绝这些不一致状态。

设调用方希望再写入 `minimum_free` 字节：

```text
已有空间足够：capacity - length >= minimum_free，直接返回
已有空间不足：required = length + minimum_free
```

在求和前要检查 `length > INT_MAX - minimum_free`，因为当前缓冲区的 `length` 和 `capacity` 使用 `int`。如果需要的容量超出 `int` 可表示范围，本轮实现返回错误，而不是发生溢出后分配过小的内存。

新容量一般从原容量或 `BUFFER_LENGTH` 开始倍增：

```text
1024 → 2048 → 4096 → 8192 ...
```

倍增是为了避免每到一个新字节就分配一次。接近 `INT_MAX` 时不能盲目再乘 2，必须限制到刚好需要的 `required`，避免整数溢出。

`realloc` 返回新地址后才更新 `buffer->data` 与 `buffer->capacity`。若它失败，原地址仍有效，调用方可以按错误路径清理连接。关闭连接统一调用 `kvs_input_buffer_free` 或 `kvs_output_buffer_free`，同时把地址、长度、容量等状态清零，防止下一次误用。

## 输入消费比扩容更容易漏

空间足够并不代表 TCP 拆包处理正确。每次业务 handler 需要明确告诉网络层：“这次成功处理了输入开头的多少字节”。网络层再调用 `kvs_input_buffer_consume`：

```c
int remaining_length = buffer->length - consumed_length;
if (consumed_length > 0 && remaining_length > 0) {
    memmove(buffer->data,
            buffer->data + consumed_length,
            remaining_length);
}
buffer->length = remaining_length;
```

`memmove` 而不是 `memcpy`，因为源区域和目标区域位于同一个缓冲区，可能重叠。

三个典型情形：

```text
收到半条命令：consumed_length = 0，所有字节保留
收到一条完整命令：消费这一条的 parsed_bytes
收到两条命令：先消费第一条，再处理缓冲区中剩下的第二条
```

不能在返回“等更多输入”时把缓冲区清空；也不能因为第一次执行产生了响应，就覆盖尚未发送的输出，或者把下一条输入也误算成已消费。这与上一篇主从同步处理 TCP 合包的经验直接相连。

## 输出缓冲区不只是容量，还包含发送进度

输出侧除了 `data`、`capacity`、`length`，还需要 `offset`。`length` 是本次响应总共有多少字节，`offset` 是已经成功发送了多少字节。

假设长度响应总共 2057 字节，第一次 `send` 只发出去 800 字节：

```text
output.length = 2057
output.offset = 800
下一次从 output.data + 800 继续发送 1257 字节
```

在 `offset == length` 前，不能把输出数据覆盖为下一条命令的响应。否则客户端可能收到前半条旧响应和后半条新响应。长 GET 测试正好覆盖了固定输出容量之外的情况，但不等于已经枚举了所有部分发送时机。

## 三个网络后端各自要注意什么

NtyCo、Reactor 和 Proactor 使用相同的缓冲区接口和协议解析入口，但等待 I/O 的方式不同。

NtyCo 协程每次 `nty_recv` 前确认剩余空间；Reactor 在 `recv_cb` 接收前扩容；Proactor 提交新一轮 `io_uring` 接收前确认地址和容量。Proactor 额外需要注意：**上一轮接收请求完成之前不能对它正在使用的缓冲区 `realloc`**。如果内存地址移动，已提交的接收操作可能还在引用旧地址。

所以不是把一个 `realloc` 调用机械复制三遍，而是让各后端在自己安全的时机使用公共缓冲区管理函数。收到完整命令后的业务逻辑仍然集中在上层。

动态缓冲区这轮没有强行接入原有内存池。内存池是否支持连续增长、如何处理大块内存与归还时机，是另一个需要单独设计的问题。

### NtyCo：协程循环中的先后顺序

NtyCo 的 `server_reader` 为每条连接保留自己的输入和输出缓冲区。每轮接收之前确认 `input.capacity - input.length` 还有空间；收到字节后增加 `input.length`，调用业务 handler，按 `consumed_length` 消费前缀，再发送已生成的输出。

协程可以反复在同一连接上处理尚存的输入；但需要在“没有任何字节被消费”时等待下一次网络数据，不能在循环中一直解析同一个半帧。

协程结束时要释放输入和输出动态内存。Replica 主动连接 Primary 时还会先生成一个初始 PING/PSYNC 消息，发送完成后再进入 `server_reader`。这块初始输出也需要先分配有效空间，并在成功和失败路径释放，不能在其他函数的作用域里误释放一个不存在的 `output` 变量。

### Reactor：epoll 告诉我们“现在可读/可写”

Reactor 的 `recv_cb` 在调用 `recv` 前确认至少有一个可写字节，随后按 `input.capacity - input.length` 计算本次可接收上限。业务处理若消费了第一条帧，而缓冲区还有第二条，就继续尝试处理；若没有消费任何字节，说明暂时不完整，恢复监听可读事件。

响应可能需要多次 `send_cb` 才能发完。关闭连接时从 epoll 移除 fd，关闭 socket，释放动态输入和输出缓冲区，再通知可选的复制关闭回调。把释放逻辑集中到统一关闭路径，可以覆盖正常断开、协议错误和发送失败。

### Proactor：完成事件之后才可以重新决定地址

Proactor 提交 `set_event_recv` 时，会把当前 `input.data + input.length` 的地址交给 io_uring。提交之后、对应完成事件到来之前，不能调用可能移动这块内存的 `realloc`。

因此顺序是：

```text
接收完成事件到来
    ↓
把完成的字节数计入 input.length
    ↓
执行并消费当前已完整的命令
    ↓
确认下一次接收需要的空间，必要时 realloc
    ↓
使用此时的新 data 地址提交下一次 recv
```

这一阶段曾把“确保空间并提交 recv”的代码误粘了两遍，导致同一 fd 可能提交两次接收。删除重复块后，才保证每次只为下一轮接收注册一次操作。这比简单的重复代码更危险，因为 `io_uring` 可能在同一缓冲区上形成意料外的并发写入。

### msg_handler 的接口为什么要调整

旧业务回调只拿到 `char *response` 和一个固定的 `response_capacity`。如果业务层要返回 2048 字节 GET，就算网络连接的输出结构已支持动态内存，业务回调也没有办法安全地让它扩容。

因此网络层与业务层之间改为传 `kvs_output_buffer_t *`。客户端协议处理器可以先扩容，再编码 value。复制协议当前仍有按行控制消息，因此把 `response->data` 和 `response->capacity` 传给旧的行批处理接口。修改函数指针类型时，`server.h` 的 typedef、`kvstore.c` 的客户端 handler、`replication.c` 的上下游 handler、三个网络后端调用点都必须一致。

## AOF：从文本行变成长度帧

旧 AOF 使用 `fprintf` 写入，例如：

```text
HSET Dad Jasper\n
HMOD Dad Sao\n
```

一旦 key/value 包含空格，重放时再按空格拆分就会还原出不同的字段。网络写入成功不代表重启后还能读回相同数据。

新 `kvs_aof_append` 把命令字段交给公共编码器：先计算所需大小、分配临时空间、编码，再用 `fwrite` 按字节写入。AOF 文件里相邻命令不再依赖额外换行分隔。

对应地，`kvs_aof_replay` 不能继续用 `getline` 把每一行当作一条命令。它读取字节、累计当前帧，调用 `kvs_parse_command`，完整后通过字段回调执行。恢复命令使用 `RECOVERY` 来源，不会在执行时又写一遍 AOF。

一次手工验证中，带空格的 `user name` 和 `hello world` 被写成 `*3\r\n$3\r\nSET...`。`xxd -g1` 能看到真实的 `0d 0a` 字节。服务重启后再次 GET，仍读到 `hello world`，说明写入、落盘、重放、查询这条路径闭合。

原有按行 AOF 是旧测试数据，本轮在确认无需保留后将其清空，重新从长度格式开始。新重放器没有为旧格式文件做自动迁移，这一点不能忽略。

### AOF 写入步骤

`kvs_aof_append(char **tokens, int count)` 暂时仍从命令执行器接收 C 字符串，因为引擎这一轮没有改成二进制接口。它为每个 token 构造 `kvs_slice_t`：

```c
fields[i].data = tokens[i];
fields[i].length = strlen(tokens[i]);
```

随后执行：

```text
1. kvs_encoded_command_size 计算 required
2. malloc(required) 获取临时编码缓冲区
3. kvs_encode_command 得到 encoded_length
4. fwrite(encoded, 1, encoded_length, aof_fp)
5. 释放 encoded
6. fflush + fsync，沿用原来的落盘策略
```

这里必须对比 `written` 与 `encoded_length`。`fwrite` 可能短写，只检查它是否大于零是不够的。`fsync` 之后，Primary 才能认为对应的 AOF 字节进入了自己的持久化进度。

`kvs_encoded_command_size` 的复用价值并不局限于客户端发命令。它让 AOF 和 Snapshot 的字节格式由同一个 codec 决定；未来若要改变字段头的写法，只改协议模块即可。

### AOF 重放为什么需要新的回调签名

旧恢复回调接收 `(char *msg, int length, char *response)`。这默认一条记录可以被表示为一段独立的、接近 C 字符串的文本。新 AOF 每条记录已经是字段数组：

```c
typedef int (*aof_replay_handler)(
    const kvs_slice_t *fields,
    size_t field_count,
    char *response,
    int response_capacity
);
```

重放器只负责“从文件里找到第一条完整命令并取出字段”；回调负责“执行这些字段”。它不需要重新把命令拼成 `HSET key value`，否则含空格字段又会丢失边界。

`src/kvstore.c` 最后使用 `kvs_recovery_fields_protocol` 作为 AOF 与 Snapshot 的共同恢复回调。这个名字强调参数已经是字段，不是原始命令文本。`RECOVERY` 来源保证恢复期间不会再次追加到 AOF。

### 逐字节读取会不会让复杂度爆炸

当前重放逻辑为了容易说明边界，逐字节累计输入并尝试 `kvs_parse_command`。这会比一次读一大块产生更多函数调用，但 `FILE *` 本身有缓冲，不等于每个字节发起一个磁盘系统调用。

还要区分“每字节调用一次解析函数”和“每次重新复制、扫描整个已有 value”。对已经读到字段长度但正文仍未到齐的输入，解析器能够通过长度与剩余字节数判断继续等待；它不需要每次遍历整个长 value。大文件性能仍值得单独测量与优化，但不能只看到 `fgetc` 就断言每字节一次物理磁盘读取。

## Snapshot：保留 offset 头部，正文改格式

Snapshot 第一行仍然是：

```text
AOF_OFFSET <offset>\n
```

这个数字记录 Snapshot 所对应的 AOF 字节位置，主从全量同步还需要它。变化的是后面的键值记录。原来一条记录是 `SET key value\n`，现在 `snapshot_write_item` 调用公共编码器，把命令名、key 和 value 写成长度格式帧。

加载时先用 `getline` 读取并解析**头部这一行**；正文则按字节累计，交给 `kvs_parse_command`。一条完整命令执行后，复用缓冲区读取下一条；文件结束时如果还留有半条命令，就视为损坏。

```text
Snapshot 文件
    ├─ AOF_OFFSET 46\n
    ├─ *3\r\n...SET...\r\n
    └─ *3\r\n...HSET...\r\n
```

当前加载端以 `rb` 打开文件，使用 `fgetc` 逐字节读取，每加入一个字节就尝试解析。`FILE *` 有内部缓冲，所以这不是每字节一次系统调用；不过函数调用次数较多。Snapshot 很大时可以再考虑分块读取。本轮优先保证帧边界正确。

不要把“Snapshot 文件里如何表示命令”和“FULL_SYNC 如何传送文件”混为一谈。主从传输仍把 Snapshot 当作原始字节流分块发送；`kvs_snapshot_reader_open/read/close` 负责读文件块。Replica 收完整个文件并安装时，`kvs_snapshot_load` 才解析其中的长度命令。

这三个 reader 函数曾在编辑过程中意外丢失，链接阶段出现 `undefined reference`。从原实现恢复后构建通过。这个插曲提醒我：修改 Snapshot 正文格式时，原始文件传输接口仍然是同一模块的重要部分，不能顺手删掉。

手工执行 `SAVE` 后，`xxd` 显示首行 `AOF_OFFSET 46\n`，后面紧跟 `*3` 开头的长度帧。再次重启并查询含空格的数据，验证了 Snapshot 保存和加载路径。

### Snapshot 为什么仍保留第一行文本

Snapshot 首行的 `AOF_OFFSET` 是文件元数据，不是一条 KV 命令。它描述“这个快照已经包含 AOF 的哪个字节位置之前的效果”。Replica 加载快照后，从这个位置继续追赶增量日志。

因此文件结构没有必要一刀切：

```text
文件头：AOF_OFFSET N\n
命令正文：一条接一条的长度格式命令
```

加载端读一次 `getline` 只用于头部。之后便不能再用 `getline` 读取命令正文。两种解析方式处在明确的边界上，不会因为“文件中还出现换行”就发生冲突。

### snapshot_write_item 的 context

四种引擎的遍历函数把每个 key/value 交给同一个回调 `snapshot_write_item(const char *key, const char *value, void *context)`。`context` 不是神秘的全局状态，而是调用遍历时传进去的一块小结构体：

```c
typedef struct {
    FILE *fp;
    const char *command;
} snapshot_write_context_t;
```

遍历 Array 时 `command` 可以是 `SET`，遍历 Hash 时改成 `HSET`。同一个写入回调根据当前引擎选择命令名，再把命令名、key、value 组成三个 slice，交给公共编码器。

```text
Array 遍历 → context.command = SET  → *3 ... SET ...
Hash 遍历  → context.command = HSET → *3 ... HSET ...
```

回调只负责写一条记录；Snapshot 保存函数负责文件头、遍历顺序、`fflush`、`fsync`、临时文件替换和最终文件大小。这样没有把整个 Snapshot 生命周期塞进单个回调。

### Snapshot 正文如何加载

头部读完后，加载器为当前命令维护：

```text
input          当前命令已读字节的地址
input_length   已读入多少字节
input_capacity 已分配多少空间
```

每读到一个字节，先确保 `input` 有容量，再追加，然后调用解析器。解析器返回 0 时继续读；返回 -1 时中止加载；返回 1 时执行刚得到的字段。

成功执行一条记录后，只把 `input_length` 置零，已分配内存可以继续复用。这样多条 Snapshot 记录不会每条都重新 `malloc` 一个新缓冲区。

文件尾有两种不同情况：

```text
input_length == 0：刚好在记录边界结束，正常
input_length > 0：还留着半条记录，Snapshot 不完整或损坏
```

处理完要释放 `input` 并关闭 `FILE *`。若解析错误、恢复回调失败、读文件出错或关闭失败，`kvs_snapshot_load` 返回错误，启动或安装流程不能把部分恢复当作完整快照。

恢复回调的响应缓冲区仍是固定的 1024 字节，因为 Snapshot 正文里只有写入命令，预期成功响应是很短的 `OK\r\n`。这个局部固定容量与“客户端 GET 可以返回长 value”不是同一个输出场景；调用时仍应把容量传给回调并检查返回长度。

### 为什么 Snapshot offset 必须按新的 AOF 字节计算

旧文本行和新长度帧对同一条逻辑命令使用的字节数不同。例如新帧的字段数、每个字段的长度头和 CRLF 都计入 AOF。Snapshot 记录的 offset 必须来自**实际新 AOF 文件位置**，不能沿用“写了几行”或“命令内容加空格”的计算。

Primary 发出 Snapshot 后，Replica 从这个 offset 继续接收 AOF。如果两个节点对同一条增量命令编码后的字节数不一致，后面收到 `ONLINE <offset>` 时就会发现本地与 Primary 的进度不相等。

## 复制链路里不是所有字节都属于 KV 命令

现有复制状态机同时承载控制行、原始文件块和 AOF 命令：

```text
HANDSHAKE   PING / PONG / PSYNC 等控制消息
FULL_SYNC   Snapshot 文件的原始字节
CATCH_UP    长度格式的 AOF 增量命令
ONLINE      长度格式增量命令，以及 ONLINE <offset> 控制行
```

因此不能把整条复制连接一律交给长度解析器。FULL_SYNC 中的某个原始文件块即便恰好以 `*` 开头，也仍然是 Snapshot 文件的一部分；必须先按协商的文件大小接收、提交并安装它。

在 CATCH_UP / ONLINE 阶段，输入以 `*` 开头时，才调用 `kvs_parse_command`。一条完整命令的字段由复制回调交给 `kvstore` 执行，并追加到 Replica 本地 AOF，然后以本地 AOF 末尾更新复制 offset。帧还不完整时消费 0 字节，等待网络补充。

`ONLINE <offset>` 仍按行解析。Replica 收到后检查自己的 AOF offset 是否与 Primary 声明的一致。其他文本行不应被悄悄当作合法控制消息消费；收尾检查时把这条路径改为返回协议错误。

复制模块从原来传 `(msg, length, response)` 改为传 `(fields, count, response, capacity)`。KV 模块接到字段后进入共用命令执行逻辑，因此复制不再把新格式重新拼回旧空格文本。

## 复制状态机里怎样选择解析方式

上一轮已经建立复制状态机。这一轮没有再造一套“RESP 专用复制器”，而是在 Replica 的上游消息入口中，根据当前状态选择原有处理方式或新命令解析器。

```text
收到 Primary 字节
    ↓
HANDSHAKE？  按控制行处理 PONG / FULLRESYNC 等消息
FULL_SYNC？  按 Snapshot 剩余文件字节数写入临时文件
CATCH_UP？   若是 * 开头，解析长度命令；否则检查 ONLINE 控制行
ONLINE？     若是 * 开头，解析长度命令；否则检查 ONLINE 控制行
```

状态判断必须先于字节内容判断。FULL_SYNC 中收到的第一个字节也可能是 `*`：Snapshot 文件的正文恰好开始了一条长度命令，但这一阶段要把它当作文件内容写入，等完整 Snapshot 安装后再由 `kvs_snapshot_load` 统一解析。

一个 `recv` 甚至可能包含 `[Snapshot 最后一块][ONLINE 控制行]`。网络层用 `consumed_length` 先消费文件最后的字节，再让剩余字节按新的复制状态重新进入处理函数。这一机制在上一篇主从同步中已建立，协议改造不能破坏它。

## CATCH_UP 增量命令的完整路径

Primary 的 AOF 已经改写成长度帧。Primary 的 stream handler 按原有逻辑从 AOF reader 读取原始字节，网络层按块发送；它不需要理解每条命令内容。

Replica 接收到这些字节后：

```text
input[0] == '*'
    ↓
kvs_parse_command(input, 当前已收长度, fields, 3, ...)
    ├─ 0：半条命令，consumed_length = 0，等待后续字节
    ├─ -1：格式错，终止本次复制连接处理
    └─ 1：执行字段，追加 Replica 本地 AOF
              ↓
           更新 replication_offset
              ↓
           consumed_length = parsed_bytes
```

字段数组容量 3 对应当前写命令的字段个数，不能当作“协议永远只支持三个字段”的通用结论。客户端协议的字段数组有自己的最大容量。

复制命令的执行回调使用 `REPLICATION` 来源。它应该像本地成功写命令一样修改内存并追加 Replica 本地 AOF，但不向 Primary 返回普通客户端响应。上游处理函数返回的 0 表示“这次没有需要发给 Primary 的响应字节”，不表示命令没有执行。

## AOF offset 是字节位置

假设快照生成时 Primary 的 AOF 末尾是 offset 1000：

```text
Snapshot 已含 [0, 1000) 的执行效果
后续命令 A 占 46 字节，范围 [1000, 1046)
后续命令 B 占 58 字节，范围 [1046, 1104)
```

Replica 安装 Snapshot 后应从 1000 开始追赶。执行并追加命令 A 后，它的本地 AOF 末尾应变为 1046；执行 B 后是 1104。若 Primary 宣布 `ONLINE 1104`，Replica 就能用本地 `kvs_aof_get_offset()` 检查自己是否真正追到这个位置。

这里比较的是**字节数**，不是“两条增量命令”这个数量。新旧编码若字节数不同，不能只比较执行了几次 SET。

## ONLINE 为什么仍是一条文本控制行

`ONLINE <Primary当前AOF offset>` 的作用是告知 Replica：这一轮 AOF 追赶的边界已经到达，可以进入持续在线同步。它不是 Array 或 Hash 的 KV 命令，不需要写入 AOF。

因此复制模块保留一个专门的文本行回调，只处理这条控制消息。它要解析出完整的 offset，检查行中没有额外尾随内容，并确认 Replica 本地 AOF 末尾与 Primary 声明的 offset 相等。

修改过程中，旧的文本回调被改名为 `kvs_replication_online_control_frame_protocol`，原本执行普通 KV 文本命令的分支删除。代码审查发现末尾还留了一个 `return 0`，使意外文本行也被当作成功消费。改为 `return -1` 后，控制消息与异常输入才有明确边界。

这个细节不是测试脚本通常会主动制造的错误输入，却是状态机很重要的防误判规则：识别出的 `ONLINE` 返回成功；不认识的文本不能静默跳过。

## 为什么改动量比最初想象的大

刚开始容易把任务理解成“加一个解析函数，把 `strtok` 换掉”。实际依赖是：

```text
新请求格式
    ↓
解析出带长度的字段
    ↓
原有引擎只接受 C 字符串，需要适配
    ↓
长请求需要动态输入缓冲区
    ↓
长 GET 需要动态输出缓冲区
    ↓
AOF / Snapshot 不能继续按行保存和重放
    ↓
复制增量使用 AOF 字节和 offset，也要接入新格式
```

这些地方不是要重复写协议逻辑，而是它们原本都假设“命令是一行、字段按空格分”。公共编解码器只写一份；网络、持久化、复制各自在自己的职责内调用它。

## 编译和联调中遇到的错误

这次是边写边验证，不是一口气改完。典型问题包括：

```text
kvs_slice_t unknown type
    使用了新类型，但没有包含 protocol.h

SIZE_MAX undeclared / memcpy implicit declaration
    缺少对应标准头文件

undefined reference to 编码大小计算函数
    公共头文件已声明，但实现尚未接入或遗漏

redefinition of ensure_buffer_space
    修改输入、输出缓冲区时重复粘贴了共用函数

undefined reference to kvs_output_buffer_ensure_space/free
    多个后端已调用，buffer.c 还没有对应定义

invalid storage class for function ...
    Proactor 某个函数结尾漏了大括号，后续函数被当作嵌套定义

incompatible function pointer type
    修改 msg_handler / 复制回调签名后，声明和调用没有同步
```

这些错误说明在 C 项目里改公共接口，必须顺着“头文件声明 → 实现 → 每个调用点 → 构建规则”检查。函数指针签名不一致尤其不能因为最终能够链接就忽略。

更稳妥的排查顺序是：先看第一条编译或链接错误，判断是缺声明、缺定义、作用域错误还是调用方仍用旧接口；修好这一处再构建，不把多个不相干的问题混在一起。

## 手工验证：含空格的命令能否重启恢复

用 Packet Sender 或 `nc` 发送：

```text
*3\r\n$3\r\nSET\r\n$9\r\nuser name\r\n$11\r\nhello world\r\n
```

服务器返回 `OK`。用长度格式 GET 读取 `user name` 时，得到 `$11\r\nhello world\r\n`。再检查 AOF：`xxd -g1` 显示 `2a 33 0d 0a 24 33 ...`，对应 `*3\r\n$3...`，不是旧的空格分隔行。

重启后 GET 仍返回 `hello world`，说明 AOF 重放成功。执行 `SAVE` 后，Snapshot 首行出现 `AOF_OFFSET 46`，正文是长度帧；再次重启后查询也成功。

十六进制检查非常重要。Packet Sender 按行显示报文时，容易把真实的 `\r\n` 字节和界面换行混在一起；`xxd` 则能确认文件中的实际格式。

## 协议单元测试

新增 `tests/test_protocol.c`，并在 Makefile 里加入 `make test-protocol`。它直接链接 `src/protocol/codec.c`，不启动服务，专门验证编解码器。

测试覆盖：

```text
value 内部含 CRLF，仍按字段声明长度保留
输入连续包含两条命令，第一次只解析第一条
最后少一个字节，返回“还需要更多输入”
字段长度写成非十进制字符，返回格式错误
2048 字节字段编码后再解析，长度与每个字节都一致
```

运行结果：

```text
./bin/test_protocol
protocol parser tests passed
```

这里的 2048 字节往返只证明编解码器正确，不能单独证明实际网络连接、Snapshot 和 Replica 也能够处理这个长度。

### 为什么把协议测试从网络测试中拆出来

若直接启动完整服务器，只看到“客户端没收到响应”，很难立刻判断是解析器、连接缓冲区、命令执行、AOF 还是复制出了错。`tests/test_protocol.c` 只链接 `codec.c`，对同一段明确的输入字节直接检查返回值和字段内容，能把协议规则单独验证。

手工编译曾使用：

```bash
gcc -Iinclude -std=gnu11 -Wall -Wextra -Werror \
    tests/test_protocol.c src/protocol/codec.c \
    -o /tmp/test_protocol
/tmp/test_protocol
```

之后把相同构建规则接入 Makefile，日常只需：

```bash
make test-protocol
```

`-Werror` 的含义是让编译警告变成错误。新增 `test_large_value` 函数后，曾忘记在 `main` 调用它，于是编译器提示 `defined but not used`。修正方式是在 `main` 中调用它，而不是去掉 `-Werror`。测试代码写在文件里却未执行，本身就是测试覆盖的缺口。

### 含 CRLF 的字段测试

测试命令正文可以写成：

```c
static const char input[] =
    "*3\r\n"
    "$3\r\nSET\r\n"
    "$3\r\nkey\r\n"
    "$4\r\na\r\nb\r\n";
```

这里 `$4` 之后的四个内容字节是 `a\r\nb`，最后的 `\r\n` 才是字段结尾。测试不仅检查返回 1，还要检查 `fields[2].length == 4`，并用 `memcmp(fields[2].data, "a\r\nb", 4)` 验证内部换行确实保留。

若只检查“解析成功”，即使解析器不小心截成 `a`，测试也可能漏掉语义错误。

### 连续两条命令测试

测试把完整 SET 和完整 GET 字节直接拼在一个数组里，调用一次 `kvs_parse_command`。期望是：

```text
返回值 == 1
field_count == SET 的字段数
parsed_bytes == 第一条 SET 的总字节数
```

不要求解析器一次返回两条命令。外层网络循环负责根据 `parsed_bytes` 再处理剩余部分。这正是协议与网络缓冲区的分工。

### 少一个字节与无效数字

对一条完整命令，故意只提供 `sizeof(command) - 2` 字节：减去 C 字符串自带的末尾 `\0` 后，再少给最后一个协议字节。解析器应返回 0。这验证“不完整”不能被当作“错误”，否则网络稍微拆包，正常客户端就会被错误断开。

另一个输入是 `*1\r\n$X\r\n`。`$` 后应该是十进制长度，`X` 不是数字，因此应返回 -1。这与“数字尚未收完”不同：多等几个字节也不会让 `X` 变成合法长度。

### 2048 字节往返

测试先分配一个 2048 字节数组，填入已知字符，再用 `kvs_encoded_command_size` 计算输出容量并调用 `kvs_encode_command`，最后把编码结果送回 `kvs_parse_command`。

关键断言：

```text
编码函数成功
解析函数返回 1
解析出预期字段数量
parsed_bytes == encoded_length
value 字段长度 == 2048
memcmp(解析结果, 原始 value, 2048) == 0
```

逐字节比较比“只检查长度”更强：长度对了仍可能在扩容、复制或结尾处理时把正文改坏。这个测试覆盖公共编解码器；实际 TCP 输入和输出要靠下一层的集成测试。

## 主从集成测试

扩展 `tests/test_replication.sh`，让它启动相互隔离目录下的 Primary 和 Replica，再发送长度格式的 HSET。测试顺序是：

```text
启动 Primary
写入含空格的初始 value
写入 2048 字节的初始 value
启动 Replica，等待它进入 ONLINE
从 Replica 查询两条初始数据
ONLINE 后向 Primary 再写入含空格的 value
等待 Replica 查询到这条增量数据
```

这里踩过两个很有代表性的坑。

第一个是人工计算字段长度：`after online value` 实际是 18 字节，误写成 `$16` 或 `$17` 时，解析器会等待剩余字节或把分隔符当作正文，客户端收不到预期响应。后来测试脚本为长字段提供了按长度构造帧的辅助函数。当前 shell 用例是 ASCII 文本；如果以后加入多字节字符，不能直接假设 Bash 的 `${#value}` 一定等于 UTF-8 字节数。

第二个是测试步骤放错位置。起初把 2048 字节写入放在 Replica 已经进入 ONLINE **之后**，却拿它验证 Snapshot。Primary 日志的 `snapshot_size=72` 清楚显示那份 Snapshot 生成时只有先前的小数据。把长 value 写入移到“启动 Replica”之前，测试才真正覆盖全量同步。**测试的时间顺序本身就是断言的一部分。**

修正后，完整回归输出：

```text
protocol parser tests passed
[PASS] Primary started
[PASS] Initial RESP data written to Primary
[PASS] 2048-byte value written to Primary
[PASS] Replica entered ONLINE
[PASS] Snapshot full synchronization with spaces
[PASS] Snapshot full synchronization with a 2048-byte value
[PASS] ONLINE incremental synchronization with spaces

replication integration test passed
```

最后再次执行 `make && make test-protocol && bash tests/test_replication.sh`，构建和两组测试均通过。集成测试说明当前默认网络后端的接收、Snapshot 全量同步、Replica 长 value 查询响应以及带空格 value 的在线增量同步已经形成闭环；它没有自动覆盖三个后端的完整测试矩阵。

### 为什么使用独立临时目录

集成测试不是在平时工作的目录里直接运行 Primary 和 Replica。脚本用 `mktemp -d` 创建专用测试目录，再分别创建 `primary/`、`replica/` 子目录。

```text
/tmp/91kv-replication-test.XXXXXX/
    primary/
    replica/
    primary.log
    replica.log
```

两个进程在各自的工作目录里生成 AOF 和 Snapshot，避免与开发目录旧数据混用。脚本保存自己启动的 PID，退出时停止两个进程并删除自己的临时目录。测试失败时先输出两份日志，帮助确定是 Primary 写入、全量同步还是在线增量出了问题。

等待 `replication: enter ONLINE` 使用带重试的日志检查，而不是固定睡一秒。网络和文件操作速度可能变化，固定时间既可能太短导致误报，也可能太长拖慢测试。

### send_resp_frame 为什么只适合已编码的测试数据

脚本中 `send_resp_frame` 通过 `printf '%b' "$frame"` 把字符串中的 `\r\n` 转成实际 CRLF 字节，然后交给 `nc -N` 发送。这个辅助函数不会替我们计算 `$长度`。手工写 `$16` 但实际正文有 18 字节时，它照样会把错误帧发出去。

之后的 `send_resp_hset` 根据传入的 key/value 构造 `$<length>`。这里曾把 `printf` 格式串写成只有三个占位符，却传入四个参数。Bash 的 `printf` 会复用格式串处理多余参数，把 2048 个 `v` 当成整数，报出 `invalid number`。正确关系应该是：

```text
%d  ← key 的长度
%s  ← key 内容
%d  ← value 的长度
%s  ← value 内容
```

用的格式串需要完整包含最后一个 `%s`：

```bash
printf '*3\r\n$4\r\nHSET\r\n$%d\r\n%s\r\n$%d\r\n%s\r\n' \
    "${#key}" "$key" "${#value}" "$value"
```

这里 `HSET` 固定四字节；测试中 key/value 都是 ASCII，因此 `${#key}`、`${#value}` 与字节数一致。若改成中文或其他多字节 UTF-8 字符，Bash 在当前 locale 下的字符计数未必等于网络协议要求的字节计数，应改用字节计数或明确设置合适的 locale。

### 如何从日志看出测试步骤放错

最初失败输出里是：

```text
[PASS] Replica entered ONLINE
[PASS] 2048-byte value written to Primary
replication test failed: Snapshot replication did not preserve the 2048-byte value
```

从顺序就能看出：Replica 已经拿到 Snapshot 并进入 ONLINE，长 value 才写入 Primary。Primary 同时报告 `offset=58 snapshot_size=72`，与只有前一条小记录的 Snapshot 相符。

这时不能仓促断言“Snapshot 解析长帧有 bug”。先确认测试是不是在正确的时间点创建数据。把整段长 value 的写入移到 `# 启动 Replica。` 之前，运行输出变为：

```text
[PASS] 2048-byte value written to Primary
[PASS] Replica entered ONLINE
[PASS] Snapshot full synchronization with a 2048-byte value
```

这个例子说明：集成测试的日志和步骤顺序也是证据。`[PASS]` 只证明某一步单独成功，不自动证明它发生在所需的同步阶段。

### 这组测试能证明什么，不能证明什么

它证明当前默认网络后端下，长度大于旧 1024 字节容量的请求能够进入 Primary，Snapshot 能保存并传到 Replica，Replica 的长度格式查询能返回完整 2048 字节。含空格 value 还分别经过 Snapshot 全量与 ONLINE 增量路径。

它没有专门通过脚本人为控制 TCP 把某个 `$长度` 分成两次发送，也没有对三种网络后端逐一运行自动化矩阵。协议单元测试中的“不完整帧”覆盖解析器返回值，但网络层真正的拆包时序仍有继续补充端到端测试的价值。

## 为什么还不能直接用 redis-cli

`redis-cli` 是客户端，会把用户输入的命令自动编码成 RESP，请求端不需要再手写 `*3` 和 `$11`。但要让它直接操作本项目，服务端响应也必须满足它能解析的 RESP 语义。当前 GET 类响应已有长度形式，SET、错误、空值等响应尚未全部统一为完整 RESP2；项目命令的名称和行为也不必然与 Redis 一致。

本轮准确的说法是：

```text
已支持：RESP 风格的长度格式命令请求，以及项目中的部分长度响应
未承诺：完整 RESP2 响应或 redis-cli 的通用兼容
```

以后如果决定支持 `redis-cli -2`，需要单独整理成功、错误、整数、空值和 bulk string 等响应，再用真实客户端验证。若只是想省去手算命令长度，也可以为 91kvstore 做自己的小型 CLI，由客户端调用编码器。

## 回头回答开发中的几个疑问

### `<stddef.h>` 为什么出现在 protocol.h

公共头文件要声明 `size_t`，所以需要包含定义这个类型的标准头文件 `<stddef.h>`。`size_t` 用来表示对象大小和数组长度，适合描述“这段输入有多少字节”和“字段长度是多少”。

`size_t` 与 `int` 不同：它是无符号类型，宽度与平台相关。网络层既有接口仍使用 `int` 保存连接缓冲区长度，所以从协议的 `size_t parsed_bytes` 转回 `int consumed_length` 时，要确保这个值不会超出 `int` 范围。本轮解析的是从 `int length` 传入的输入区间，成功消费量不应大于输入长度；以后若扩大网络长度接口，需要重新审视这个转换。

### 为什么称为“帧”

帧在这里不是以太网帧，只是指一条有明确开头和结尾的协议消息。网络层看到的是连续字节：

```text
[命令帧 1][命令帧 2][命令帧 3 的前半段]
```

“帧长度”强调一条完整消息占用了多少字节；“命令”强调业务含义。本文在讲语义时说命令，在讲边界与消费量时说帧，不需要把两者当成不同业务对象。

### parse_field 与 parse_command 为什么分开

单个字段只需要理解 `$长度\r\n内容\r\n`。整条命令要先理解 `*字段数量\r\n`，再重复解析字段。把两个层级写进一个大函数会混淆“不完整数字”“不完整字段”和“字段数超过容量”等不同错误来源。

`parse_field` 是内部辅助函数，不出现在头文件里；外部调用者只需要 `kvs_parse_command`。这也让以后更换内部解析步骤时，不必修改所有使用协议的模块。

### consumed_length 与 parsed_bytes 是不是一回事

两者在新命令入口通常表示同一个数量，但处在不同接口：

```text
parsed_bytes    协议解析器计算：第一条完整命令的字节数
consumed_length 网络 handler 告诉后端：本次可以从输入缓冲区移除多少字节
```

协议解析成功且命令执行成功，handler 才把 `parsed_bytes` 写入 `consumed_length`。若命令尚未收完整，消费量是 0。若执行失败，连接可能走错误处理，不能随意宣称这段输入已经成功处理。

复制 FULL_SYNC 状态还有另一个用法：它消费的是 Snapshot 原始文件块，而不是长度格式命令。这个例子也说明 `consumed_length` 是网络层的通用契约，`parsed_bytes` 是协议解析器的具体结果。

### 为什么不能把 parser 做成每个后端的 feed 状态机

每条连接已经保存自己的输入缓冲区。解析器接收 `input + input_length`，返回“完整/等待/错误”和已消费字节数；半帧保存在连接缓冲区，下一次收到数据后再调用同一个无状态解析器。

这样协议模块不需要拥有 socket、不需要管理协程或 io_uring，也不需要再维护一份独立的 per-connection parser 状态。若未来有性能需求，可以设计增量状态机，但当前实现用同一解析函数即可满足三个后端的正确性要求。

### `realloc` 会不会造成外部碎片

它可能移动内存，也可能使堆内存布局出现碎片；这是通用堆分配器需要面对的代价。当前主要目标是处理超出 1024 字节的连接数据，并避免每增加一个字节就分配一次，因此使用倍增容量减少重新分配次数。

是否要用内存池替代，得先知道池是否支持连续增长、如何处理变长对象、一个连接何时归还整块内存，以及并发连接的峰值使用量。没有这些设计就直接把 `realloc` 改成池分配，可能只是把碎片问题换成数据复制和容量浪费。

### Packet Sender、nc 与 redis-cli 的分工

Packet Sender 适合观察、手工构造协议字节，尤其能用十六进制检查 CRLF；`nc` 适合把 `printf` 生成的确切字节送到服务端，放进 shell 集成测试。

`redis-cli` 会把用户输入的命令编码成 RESP，并期待服务端返回符合 RESP 的响应。它很方便，但它是兼容性目标，不是这次长度帧改造能够自动获得的能力。自己实现的小 CLI 只需遵守本项目已经实现的请求和响应约定，工作量则与 Redis 兼容不同。

## 当前完成边界

以“文本 key/value 可以包含原分隔符，命令长度不再被固定网络缓冲区截住，并能经过持久化与主从同步”为目标，关键路径已经完成。

```text
✅ protocol 模块集中编解码长度命令
✅ 解析器区分完整、未完成和格式错误
✅ 解析结果提供字段视图与第一条命令消费字节数
✅ 三个网络后端接入动态输入/输出缓冲区接口
✅ AOF 追加、重放使用长度帧
✅ Snapshot 保留 offset 头部，正文使用长度帧
✅ CATCH_UP / ONLINE 执行长度格式增量命令
✅ 协议单元测试和默认后端主从集成测试通过
```

尚未解决或需要另立需求的边界：

```text
⬜ 内嵌 NUL 的二进制 key/value：四种引擎仍依赖 C 字符串
⬜ 完整 RESP2 响应与 redis-cli 兼容
⬜ 旧按行客户端协议的长期保留或移除策略
⬜ 旧格式 AOF / Snapshot 的自动迁移
⬜ 超大命令的资源上限与拒绝策略
⬜ 更多恶意长度、TCP 拆包和粘包的端到端用例
⬜ NtyCo / Reactor / Proactor 自动化测试矩阵
⬜ Snapshot 加载阶段逐字节读取的性能优化
```

目前新客户端入口仍保留旧按行格式的兼容路径；复制握手和 `ONLINE` 控制行则因职责不同，继续使用各自的控制格式。未来若移除旧客户端兼容，不能把这些复制控制消息一并删除。

## 代码地图：遇到问题先看哪里

这轮改动横跨多个目录。重新打开项目时，可以按职责找文件，而不是从 `main()` 一路滚动到末尾。

| 文件 | 主要职责 | 出问题时先看什么 |
| --- | --- | --- |
| `include/protocol.h` | slice 与编解码接口 | 声明、参数、返回值是否一致 |
| `src/protocol/codec.c` | 数字、字段、整条命令的解析与编码 | 长度、CRLF、溢出、返回 0/-1/1 |
| `include/server.h` | handler 与缓冲区结构 | 输出是否给了扩容所需结构 |
| `src/network/buffer.c` | 输入消费和输入/输出扩容 | `length <= capacity`、重复定义、释放 |
| `src/network/ntyco.c` | 协程连接读写 | recv 前空间、循环消费、关闭释放 |
| `src/network/reactor.c` | epoll 连接读写 | EPOLLIN/EPOLLOUT 切换与部分发送 |
| `src/network/proactor.c` | io_uring 完成事件 | pending recv 使用的地址是否稳定 |
| `src/kvstore.c` | 协议入口与命令执行适配 | slice 转 token、命令来源、GET 响应 |
| `src/persistence/aof.c` | AOF 追加与重放 | 写入字节数、恢复回调、旧格式遗留 |
| `src/persistence/snapshot.c` | Snapshot 保存与加载 | 第一行 offset、正文帧、原始文件 reader |
| `src/replication/replication.c` | 复制状态机与增量命令 | FULL_SYNC/CATCH_UP/ONLINE 是否走对分支 |
| `tests/test_protocol.c` | 解析器单元测试 | 边界字节、返回值、字段内容 |
| `tests/test_replication.sh` | 端到端主从测试 | 写入时机、日志、全量与在线验证 |

例如“SET 返回 OK，但重启后 GET 不到”，优先看 AOF 是否使用长度格式、恢复是否按字段回调执行；“只在 Proactor 下长命令出错”，优先看 io_uring 接收完成前是否错误扩容；“全量测试失败，但 Primary 日志的 Snapshot 很小”，优先确认测试是否在生成 Snapshot 前完成写入。

## 一次从零回归的顺序

如果隔一段时间重新接手这个分支，可以按由小到大的顺序验证：

```bash
make
make test-protocol
bash tests/test_replication.sh
```

`make` 说明声明、实现和链接至少在当前配置下吻合。`make test-protocol` 排查纯协议格式；主从脚本才覆盖两个服务进程、端口、AOF、Snapshot 与在线复制。不要只看到 `make` 成功就宣称长命令已能端到端使用。

手工调试时再补：

```bash
xxd -g1 -l 100 appendonly.aof
xxd -g1 -l 100 snapshot.db
```

这些命令直接显示文件字节。Snapshot 第一行应该是可读的 `AOF_OFFSET`，正文应从 `*` 长度帧开始；AOF 则不应回到 `HSET key value\n` 的旧行格式。

如果测试失败，先分清失败位置：

```text
没有响应：请求帧可能不完整、长度数字可能错、连接可能提前关闭
SET 有响应但 AOF 不对：检查追加路径和命令来源
AOF 正确但重启读不到：检查重放与恢复回调
Snapshot 大小不对：检查生成时机与 foreach 写入
Replica 与 Primary offset 不同：检查增量执行、AOF 编码字节和 ONLINE 校验
```

这比看到一个空响应就同时怀疑协议、网络、AOF 和 Snapshot 更容易定位。

## 与上一篇主从同步的关系

上一篇的主从复制建立了状态机、Snapshot offset、AOF 追赶、三种网络后端与输入输出缓冲区契约。这一篇没有推翻那些设计，而是改变了**KV 命令本身在网络和持久化中的字节表示**。

最容易误以为“主从同步早已完成，不用改”的是增量阶段。Primary 传送的正是 AOF 字节；AOF 一旦从文本行改为长度帧，Replica 就必须按新帧解析、执行并正确更新 offset。相反，FULL_SYNC 的 Snapshot 文件分块传输仍然是原始字节流，不因为正文格式改变而重新设计底层发送。

可以把两篇的关系概括为：

```text
第五篇：字节怎样从 Primary 可靠地到达 Replica，并在正确 offset 衔接
第六篇：这些字节里的 KV 命令怎样无歧义地表示 key/value
```

## 去 AI 化复盘：必须自己讲清楚的五件事

### 1. 长度解决了什么

空格和换行本来既可能是数据，也被旧协议当作边界。长度格式先声明字段有多少字节，再精确读取这么多字节，于是数据里的旧分隔符不再参与拆字段。

### 2. parsed_bytes 为什么重要

TCP 可能一次送来半条、一条或多条命令。解析器返回第一条完整命令消费的字节数，网络层才能移除已处理的前缀，并留下后面的数据。

### 3. slice 与 C 字符串有什么不同

slice 是地址加长度，不自动以 `\0` 结尾，也不拥有内存。现有引擎仍用 C 字符串，所以边界层复制并补终止符。这让含空格文本工作，也说明内嵌 NUL 为什么仍不支持。

### 4. 为什么 AOF 和 Snapshot 也必须改

持久化保存的是将来要再次执行的命令。如果网络能够接收 `user name`，AOF 却仍写 `SET user name ...\n`，重启后的字段会变样。复制增量又直接使用 AOF 字节，三者必须对命令边界达成一致。

### 5. 为什么不能把所有复制字节都交给同一个解析器

复制连接同时传输握手控制行、Snapshot 原始文件字节、AOF 增量命令和 ONLINE 控制行。状态机决定当前字节的含义。只有增量阶段的长度命令交给 `kvs_parse_command`，FULL_SYNC 的文件块不能当作命令解释。

## 面试版总结

> 我把 91kvstore 原来按空格拆字段、按行拆命令的文本协议，改造成 RESP 风格的长度字段协议。公共 protocol 模块计算帧大小、编码命令，并从字节流解析第一条完整命令；解析结果用只引用输入的 slice 表示，同时返回已消费字节数，供网络层处理半包和粘包。三个网络后端共用解析入口，输入和输出缓冲区按需扩容。为了让含空格和长 value 在重启及主从同步后仍保持一致，我同步改造 AOF、Snapshot 正文和复制增量命令处理，保留握手与 ONLINE 控制行各自的格式。协议单元测试覆盖 CRLF、半包、连续帧、错误长度和 2048 字节字段；主从集成测试验证带空格和长 value 的全量同步，以及带空格 value 的在线增量同步。当前引擎仍以 C 字符串为核心，因此不支持内嵌 NUL，服务端响应也尚未完整兼容 RESP2。

这轮命令协议改造的核心需求至此完成。完整二进制值与 `redis-cli` 兼容，应作为后续独立需求设计和验证。
