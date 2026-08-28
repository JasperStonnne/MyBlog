---
title: KV 存储项目：协议层与数组存储
slug: kv-store-protocol-array
description: 沿着收到命令、切分 token、分发协议、执行数组 CRUD 和生成响应的调用链，回顾 KV 存储项目的协议层与数组引擎
date: 2026-08-24T00:00:00+08:00
draft: false
image: cover.svg
tags:
  - KV 存储
  - C 语言
  - TCP
  - 自定义协议
  - 数据结构
  - CRUD
categories:
  - 项目记录
---

> 阅读目标：沿着“收到一条命令—解析命令—操作数组—生成响应”这条主线，回顾 KV 项目从 echo 协议发展到实际命令处理的过程。
>
> 本文承接 [KV 存储项目网络层：统一接入 Reactor、Proactor 与协程]({{< ref "kv-store-network-layer" >}})。上一篇已经讲过 Reactor、Proactor、ntyco、`msg_handler` 和网络收发流程，本文不再重复网络层实现，只说明它怎样把请求交给协议层。
>
> 本文以笔记和现有代码为准，解释代码在项目中的作用与执行过程，不讨论修改方案。

---

## 1. 从网络层进入协议层

上一篇已经完成了网络层。无论底层选择 Reactor、Proactor 还是 ntyco，收到请求后都会调用同一个入口：

```c
kvs_protocol(msg, length, response);
```

这一篇从这里继续：

```text
网络层收到客户端数据
        ↓
kvs_protocol() 进入协议层
        ↓
kvs_split_token() 切分命令
        ↓
kvs_filter_protocol() 识别命令
        ↓
kvs_array_xxx() 操作 KV 数据
        ↓
把执行结果写入 response
        ↓
网络层发送响应
```

网络层解决“数据怎样到达”，协议层解决“收到的数据表示什么、应该返回什么”。

---

## 2. 为什么 TCP 之上还需要业务协议

假设浏览器请求一个包含数据的网页，业务链路可以是：

```text
浏览器 → NodeServer → KV 存储服务器
```

`NodeServer` 与 KV 存储服务器建立 TCP 连接后，还要约定双方发送数据的格式。例如：

```text
SEND: SET Key Value\r\n
RECV: OK\r\n
```

```text
SEND: GET Key\r\n
RECV: King\r\n
```

TCP 是公共的传输协议，只负责按顺序传输字节；`SET` 和 `GET` 的含义属于项目自己定义的应用层协议，也可以称为业务协议或私有协议。

本项目采用容易观察的文本命令：

```text
SET Key Value
GET Key
DEL Key
MOD Key Value
EXIST Key
```

其中空格用于分隔命令和参数，响应通过 `response` 缓冲区交还给网络层。

---

## 3. TCP 消息边界：理解协议的前提

TCP 是字节流，一次 `send()` 不一定对应一次 `recv()`。

客户端连续发送：

```text
SET Key Value\r\n
SET Key1 Value1\r\n
SET Key2 Value2\r\n
```

接收方可能一次 `recv()` 收到多条命令，也可能只收到一条命令的一部分。因此应用层需要自己识别完整消息的边界。

笔记中记录了两种约定边界的思路。

### 3.1 在包头记录长度

在数据最前面使用固定字节表示后续消息长度：

```c
short length = 0;

recv(fd, &length, 2, 0);
recv(fd, buffer, length, 0);
```

概念流程是：

```text
先读取固定长度的包头
        ↓
得到消息体长度 length
        ↓
继续读取 length 个字节
        ↓
取得一条完整消息
```

### 3.2 使用结束标记

文本协议也可以约定用 `\r\n` 表示一行结束，再逐行解析命令。

无论采用长度字段还是结束标记，目的都是让接收方能够从连续的 TCP 字节流中还原出一条条完整命令。

ET、LT 对应不同的事件触发方式；它们都能处理多个数据包，具体读取过程属于上一篇网络层的延伸问题。本文继续关注完整命令到达 `kvs_protocol()` 之后的处理。

---

## 4. Redis 协议带来的解析思路

`SET Key Value` 可以拆成三个 token：

```text
SET
Key
Value
```

笔记中的 Redis 协议简化示意不仅传输 token 内容，还记录 token 数量和每个 token 的长度：

```text
3\r\n
3\r\n
SET\r\n
3\r\n
Key\r\n
5\r\n
Value\r\n
```

可以依次理解为：

```text
共有 3 个 token
SET   的长度是 3
Key   的长度是 3
Value 的长度是 5
```

`GET Teacher` 则有两个 token：

```text
2\r\n
3\r\n
GET\r\n
7\r\n
Teacher\r\n
```

对应的读取伪代码是：

```c
readline(fd, buffer);
count = atoi(buffer);

for (int i = 0; i < count; i++) {
    readline(fd, tokenlen);
    readline(fd, token);
}
```

解析主线是：

```text
读取 token 数量
        ↓
循环读取每个 token 的长度
        ↓
按照长度读取 token 内容
        ↓
组合成一条完整命令
```

当前项目代码采用更直接的形式：收到 `SET Key Value` 后，以空格调用 `strtok()` 进行切分。

---

## 5. 文件职责

本次代码可以按职责分为三部分：

```text
kvstore.h
  ├─ 公共宏与函数声明
  ├─ 数组 KV 数据结构
  └─ CRUD 接口

kvstore.c：协议部分
  ├─ 内存接口封装
  ├─ token 切分
  ├─ 命令识别与响应生成
  ├─ KV 引擎初始化
  └─ main

数组存储部分
  ├─ global_array
  ├─ create / destory
  └─ set / get / del / mod / exist
```

上一篇的 `kvs_protocol()` 只是把请求复制到响应中；本次代码让它开始解析并执行真实的 KV 命令。

---

## 6. `kvstore.h`：数组引擎的数据结构与接口

### 6.1 编译配置

```c
#define KVS_MAX_TOKENS 128
#define ENABLE_ARRAY 1
#define KVS_ARRAY_SIZE 1024
```

| 宏 | 代码中的含义 |
|---|---|
| `KVS_MAX_TOKENS` | `tokens` 数组容量 |
| `ENABLE_ARRAY` | 是否编译数组 KV 引擎 |
| `KVS_ARRAY_SIZE` | KV 数组的容量 |

### 6.2 一个 KV 元素

```c
typedef struct kvs_array_item_s {
    char *key;
    char *value;
} kvs_array_item_t;
```

数组中的每一个元素保存一对指针：

```text
table[i]
  ├─ key   → 键字符串
  └─ value → 值字符串
```

### 6.3 整个数组实例

```c
typedef struct kvs_array_s {
    kvs_array_item_t *table;
    int idx;
    int total;
} kvs_array_t;
```

| 字段 | 含义 |
|---|---|
| `table` | 指向 KV 元素数组 |
| `idx` | 数组实例中的索引字段 |
| `total` | 当前记录的元素数量 |

数组引擎向协议层提供以下操作：

```c
int kvs_array_create(kvs_array_t *inst);
void kvs_array_destory(kvs_array_t *inst);
int kvs_array_set(kvs_array_t *inst, char *key, char *value);
char *kvs_array_get(kvs_array_t *inst, char *key);
int kvs_array_del(kvs_array_t *inst, char *key);
int kvs_array_mod(kvs_array_t *inst, char *key, char *value);
int kvs_array_exist(kvs_array_t *inst, char *key);
```

---

## 7. 封装内存分配接口

代码没有让各个数据操作直接依赖 `malloc()` 和 `free()`，而是包了一层：

```c
void *kvs_malloc(size_t size) {
    return malloc(size);
}

void kvs_free(void *ptr) {
    free(ptr);
}
```

其他模块统一使用：

```c
kvs_malloc(size);
kvs_free(ptr);
```

笔记中的目的，是给内存管理保留统一入口。后续如果跨平台、引入内存池或定制内存分配，只需要从这一层接入，不必改变所有调用位置。

这体现了项目迭代中的一个思路：先通过统一接口隔开业务代码与底层实现。

---

## 8. 创建数组实例：`kvs_array_create()`

全局数组实例定义为：

```c
kvs_array_t global_array = {0};
```

创建函数：

```c
int kvs_array_create(kvs_array_t *inst) {
    if (inst == NULL) return -1;

    if (inst->table) {
        printf("table has been alloced");
        return -1;
    }

    inst->table = kvs_malloc(
        KVS_ARRAY_SIZE * sizeof(kvs_array_item_t)
    );

    if (!inst->table) return -1;

    inst->total = 0;
    return 0;
}
```

执行过程：

```text
检查 inst
   ↓
检查 table 是否已经指向数组
   ↓
申请 KVS_ARRAY_SIZE 个元素的空间
   ↓
total 设为 0
```

程序启动时，`init_kvengine()` 负责初始化它：

```c
int init_kvengine(void) {
#if ENABLE_ARRAY
    memset(&global_array, 0, sizeof(kvs_array_t));
    kvs_array_create(&global_array);
#endif
    return 0;
}
```

---

## 9. `SET`：保存一组 key-value

接口：

```c
int kvs_array_set(kvs_array_t *inst,
                  char *key,
                  char *value);
```

主要执行顺序：

```text
检查 inst、key、value
        ↓
检查 total 是否达到数组容量
        ↓
调用 kvs_array_get() 查找 key
        ↓
key 已存在时返回 1
        ↓
分别为 key 和 value 分配内存
        ↓
复制 key 和 value
        ↓
寻找存放位置并写入 table
        ↓
total 加 1，返回 0
```

代码为字符串申请 `strlen(...) + 1` 字节：

```c
char *kcopy = kvs_malloc(strlen(key) + 1);
char *kvalue = kvs_malloc(strlen(value) + 1);
```

多出的一个字节用于字符串结尾的 `\0`。

返回值在当前协议中的含义：

```text
ret < 0  → 执行错误
ret == 0 → 设置成功
ret > 0  → key 已存在
```

---

## 10. `GET`：按照 key 查找 value

接口：

```c
char *kvs_array_get(kvs_array_t *inst, char *key);
```

查询过程：

```text
检查 inst 和 key
        ↓
从 table[0] 开始遍历
        ↓
跳过 key 为空的元素
        ↓
strcmp() 比较元素 key 和目标 key
        ↓
找到后返回 value
        ↓
遍历结束仍未找到则返回 NULL
```

核心比较：

```c
if (strcmp(inst->table[i].key, key) == 0) {
    return inst->table[i].value;
}
```

因此 `kvs_array_get()` 的结果可以直接被 `SET`、`GET` 和 `EXIST` 复用。

---

## 11. `DEL`：删除 key-value

接口：

```c
int kvs_array_del(kvs_array_t *inst, char *key);
```

执行主线：

```text
检查 inst 和 key
        ↓
遍历 table
        ↓
strcmp() 找到相同 key
        ↓
释放 key 对应的内存
        ↓
释放 value 对应的内存
        ↓
返回执行结果
```

协议层按照约定把返回值转换为：

```text
ERROR / OK / NO EXIST
```

---

## 12. `MOD`：修改已有 key 的 value

接口：

```c
int kvs_array_mod(kvs_array_t *inst,
                  char *key,
                  char *value);
```

执行主线：

```text
检查 inst、key、value
        ↓
遍历 table 查找 key
        ↓
释放原来的 value
        ↓
按照新字符串长度申请内存
        ↓
复制新的 value
        ↓
把新 value 写回当前元素
```

`MOD` 保留原来的 key，只更新与它关联的 value。

---

## 13. `EXIST`：复用 GET 判断 key 是否存在

```c
int kvs_array_exist(kvs_array_t *inst, char *key) {
    char *str = kvs_array_get(inst, key);

    if (!str) {
        return 1;
    }
    return 0;
}
```

返回值含义：

```text
0 → key 存在
1 → key 不存在
```

这里没有再次编写数组遍历，而是调用 `kvs_array_get()` 完成查找。

---

## 14. 命令字符串与枚举的对应关系

命令字符串表：

```c
const char *command[] = {
    "SET", "GET", "DEL", "MOD", "EXIST"
};
```

枚举：

```c
enum {
    KVS_CMD_START = 0,
    KVS_CMD_SET = KVS_CMD_START,
    KVS_CMD_GET,
    KVS_CMD_DEL,
    KVS_CMD_MOD,
    KVS_CMD_EXIST,
    KVS_CMD_COUNT,
};
```

二者按下标形成对应关系：

| `cmd` | `command[cmd]` | 枚举 |
|---:|---|---|
| 0 | `SET` | `KVS_CMD_SET` |
| 1 | `GET` | `KVS_CMD_GET` |
| 2 | `DEL` | `KVS_CMD_DEL` |
| 3 | `MOD` | `KVS_CMD_MOD` |
| 4 | `EXIST` | `KVS_CMD_EXIST` |

协议解析通过同一个 `cmd` 同时连接字符串表与 `switch` 分支，所以代码注释强调二者顺序保持对应。

---

## 15. `kvs_split_token()`：把命令切成参数

函数入口：

```c
int kvs_split_token(char *msg, char *tokens[])
```

首先检查参数：

```c
if (msg == NULL || tokens == NULL) return -1;
```

然后以空格为分隔符调用 `strtok()`：

```c
int idx = 0;
char *token = strtok(msg, " ");

while (token != NULL) {
    tokens[idx++] = token;
    token = strtok(NULL, " ");
}

return idx;
```

对于：

```text
SET Key Value
```

切分结果是：

```text
tokens[0] → "SET"
tokens[1] → "Key"
tokens[2] → "Value"
返回 count = 3
```

对于：

```text
GET Key
```

切分结果是：

```text
tokens[0] → "GET"
tokens[1] → "Key"
返回 count = 2
```

笔记中强调：函数进入时先进行参数判断。完整项目还会逐步形成统一的参数错误、内存错误和协议错误等返回值约定。

---

## 16. `kvs_filter_protocol()`：识别命令并生成响应

函数入口：

```c
int kvs_filter_protocol(char **tokens,
                        int count,
                        char *response);
```

### 16.1 查找命令编号

```c
int cmd = KVS_CMD_START;

for (cmd = KVS_CMD_START;
     cmd < KVS_CMD_COUNT;
     cmd++) {
    if (strcmp(tokens[0], command[cmd]) == 0) {
        break;
    }
}
```

例如 `tokens[0]` 是 `GET`，循环最终得到：

```text
cmd = KVS_CMD_GET
```

随后统一取出参数位置：

```c
char *key = tokens[1];
char *value = tokens[2];
```

### 16.2 SET 分支

```c
ret = kvs_array_set(&global_array, key, value);
```

| 数组层结果 | 协议响应 |
|---|---|
| `ret < 0` | `ERROR\r\n` |
| `ret == 0` | `OK\r\n` |
| `ret > 0` | `EXIST\r\n` |

### 16.3 GET 分支

```c
char *result = kvs_array_get(&global_array, key);
```

| 查询结果 | 协议响应 |
|---|---|
| `result == NULL` | `NO EXIST\r\n` |
| 找到 value | `value\r\n` |

### 16.4 DEL 分支

```c
ret = kvs_array_del(&global_array, key);
```

| 数组层结果 | 协议响应 |
|---|---|
| `ret < 0` | `ERROR\r\n` |
| `ret == 0` | `OK\r\n` |
| `ret > 0` | `NO EXIST\r\n` |

### 16.5 MOD 分支

```c
ret = kvs_array_mod(&global_array, key, value);
```

| 数组层结果 | 协议响应 |
|---|---|
| `ret < 0` | `ERROR\r\n` |
| `ret == 0` | `OK\r\n` |
| `ret > 0` | `NO EXIST\r\n` |

### 16.6 EXIST 分支

```c
ret = kvs_array_exist(&global_array, key);
```

| 数组层结果 | 协议响应 |
|---|---|
| `ret == 0` | `EXIST` |
| `ret != 0` | `NO EXIST\r\n` |

`kvs_filter_protocol()` 的角色可以概括为：把文本命令转换成数组函数调用，再把数组函数的返回值转换成文本响应。

---

## 17. `kvs_protocol()`：本篇代码的总入口

```c
int kvs_protocol(char *msg,
                 int length,
                 char *response) {
    if (msg == NULL || length <= 0 || response == NULL)
        return -1;

    printf("recv: %d: %s\n", length, msg);

    char *tokens[KVS_MAX_TOKENS] = {0};

    int count = kvs_split_token(msg, tokens);
    if (count == -1) return -1;

    return kvs_filter_protocol(tokens, count, response);
}
```

三个参数分别来自网络层：

| 参数 | 含义 |
|---|---|
| `msg` | 收到的请求内容 |
| `length` | 请求长度 |
| `response` | 协议层写入响应的缓冲区 |

执行主线：

```text
检查参数
   ↓
创建 tokens 数组
   ↓
kvs_split_token(msg, tokens)
   ↓
得到 token 数量 count
   ↓
kvs_filter_protocol(tokens, count, response)
   ↓
执行命令并形成响应
```

它把上一篇的“网络回调入口”与本篇的“命令解析和数组 CRUD”连接起来。

---

## 18. 程序启动与数组引擎接入

`main()` 先取得端口，再初始化 KV 引擎：

```c
int port = atoi(argv[1]);
init_kvengine();
```

之后按照 `NETWORK_SELECT` 启动对应网络层，并继续把 `kvs_protocol` 作为回调传入：

```c
reactor_start(port, kvs_protocol);
/* 或 */
ntyco_start(port, kvs_protocol);
/* 或 */
proactor_start(port, kvs_protocol);
```

这里不再展开三种网络模型。只需记住：网络模型可以切换，协议入口和数组引擎的调用主线保持不变。

---

## 19. 一条 `SET` 命令的完整路径

客户端发送：

```text
SET name King
```

代码中的执行过程：

```text
网络层取得 msg
        ↓
kvs_protocol(msg, length, response)
        ↓
kvs_split_token()
        ↓
tokens[0] = "SET"
tokens[1] = "name"
tokens[2] = "King"
        ↓
kvs_filter_protocol()
        ↓
匹配 KVS_CMD_SET
        ↓
kvs_array_set(&global_array, "name", "King")
        ↓
为 key 和 value 分配内存并写入 table
        ↓
response = "OK\r\n"
        ↓
网络层发送响应
```

---

## 20. 一条 `GET` 命令的完整路径

客户端发送：

```text
GET name
```

执行过程：

```text
kvs_protocol()
        ↓
切分出 "GET" 和 "name"
        ↓
kvs_filter_protocol()
        ↓
匹配 KVS_CMD_GET
        ↓
kvs_array_get(&global_array, "name")
        ↓
遍历 table 并比较 key
        ↓
找到 value = "King"
        ↓
response = "King\r\n"
```

如果没有找到，则响应：

```text
NO EXIST\r\n
```

---

## 21. 返回值与协议响应是两个层次

数组函数首先返回内部执行结果，例如：

```text
小于 0：错误
等于 0：成功或存在
大于 0：已存在或不存在
```

协议层再将其转换成客户端能够理解的文本：

```text
OK
ERROR
EXIST
NO EXIST
具体的 value
```

这与 HTTP 使用 `200`、`404` 等状态表达处理结果的思路相似：一个完整项目会逐步形成统一的参数错误、内存分配失败、协议解析失败等返回值标准。

---

## 22. 从功能实现到项目架构

笔记给出的迭代顺序是：

```text
先保证功能
    ↓
保证协议统一
    ↓
保证参数统一
    ↓
保证返回值统一
```

数组法先直接实现了 KV 的基本操作：

```text
SET   → 增加 key-value
GET   → 查询 value
DEL   → 删除 key-value
MOD   → 修改 value
EXIST → 判断 key 是否存在
```

随着代码不断迭代，公共接口、模块边界和返回值规则逐渐稳定，进而形成项目架构。

可以记住两句话：

> 架构是在软件不断迭代的过程中逐渐产生的。

> 软件代码可以通过分层设计，将网络收发、协议处理和数据存储分开。

---

## 23. 最后的整体记忆

这一阶段最重要的不是分别记住所有函数，而是记住数据在各层之间怎样移动：

```text
客户端文本命令
     ↓
网络层交给 kvs_protocol
     ↓
kvs_split_token 拆出 command / key / value
     ↓
kvs_filter_protocol 匹配命令
     ↓
kvs_array_set/get/del/mod/exist
     ↓
global_array 中的 key-value 数据
     ↓
协议层生成 OK、ERROR、value 等响应
     ↓
网络层发送给客户端
```

按函数压缩为：

```text
kvs_protocol
  → kvs_split_token
  → kvs_filter_protocol
  → kvs_array_xxx
  → response
```
