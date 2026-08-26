---
title: KV 存储项目：Hash 接入、多语言客户端与 SkipList 扩展
slug: kv-store-hash-multilingual-clients-skiplist
description: 在已有 Array、RBTree 和统一协议基础上，将 Hash 完整接入 KVEngine，并梳理多语言客户端边界与 SkipList 范围查询扩展
date: 2026-08-26T00:00:00+08:00
draft: false
image: cover.svg
tags:
  - KV 存储
  - Hash
  - SkipList
  - 多语言客户端
  - C 语言
categories:
  - 后端开发
---

> 阅读目标：在已有 Array、RBTree 和统一协议的基础上，重点复盘 Hash 如何真正成为一个可用的 KV 引擎；随后理解多语言客户端与服务端的边界，并按同一套接入模式继续扩展 SkipList。
>
> 本文不讲 Hash 数据结构算法本身，重点是 Hash 与现有 KV 项目的融合。

## 与前三篇的关系

本文承接以下三篇已有笔记：

1. [《KV 存储项目网络层》]({{< ref "kv-store-network-layer" >}})已经讲过 Reactor、Proactor、NtyCo 和 `msg_handler` 回调。
2. [《KV 存储项目：协议层与数组存储》]({{< ref "kv-store-protocol-array" >}})已经讲过 TCP 消息边界、token 解析、协议响应和 Array 的五种操作。
3. [《KV 存储项目：客户端测试、压力测试与多引擎扩展》]({{< ref "kv-store-testing-multi-engine" >}})已经讲过 Makefile 基础、测试用例、压力测试、宏开关、5+2 接口以及 RBTree 的接入模式。

因此本文不再重复网络收发、协议解析、Array/RBTree CRUD、通用测试方法和 5+2 的基础概念，只保留理解 Hash 接入所需的连接点。

---

## 1. 本阶段增加了什么

项目之前已经完成：

```text
统一网络入口
    → 文本协议解析
    → Array 引擎
    → RBTree 引擎
    → 客户端测试和压力测试
```

本阶段增加三项能力：

```text
1. 把 Hash 接入现有 KVEngine
2. 用 Go / Java / Node.js / Python / Rust 操作同一个 KV 服务
3. 按相同模式接入 SkipList，并为范围查询做准备
```

Hash 部分最值得掌握的不是“冲突链表怎样写”，而是：已经有一份 Hash 代码以后，还要修改项目的哪些位置，它才能从客户端真正访问到？

```text
类型与接口声明
  → 编译和链接
  → 全局实例
  → create 初始化
  → 协议命令分发
  → set/get/del/mod/exist
  → 客户端测试
  → destroy 释放
```

任何一环缺失，Hash 都没有完整融入项目。

---

## 2. Hash 在项目中的位置

Hash 接入后，核心关系变成：

```text
                    kvs_protocol()
                          │
                  kvs_filter_protocol()
                          │
        ┌─────────────────┼─────────────────┐
        │                 │                 │
     Array             RBTree             Hash
  普通命令前缀         R 命令前缀         H 命令前缀
        │                 │                 │
 global_array      global_rbtree       global_hash
```

网络层仍然只调用 `kvs_protocol()`，不需要为了 Hash 修改 Reactor、Proactor 或 NtyCo。

协议层也不需要知道桶、哈希下标和冲突链表，只负责把 Hash 命令映射到 `kvs_hash_set/get/del/mod/exist`。

这体现了已有分层设计带来的扩展性：**新增存储引擎，主要修改核心层和构建层，不侵入网络层。**

---

## 3. 第一个接入点：`kvstore.h`

### 3.1 增加编译开关

```c
#define ENABLE_HASH 1
```

它应当像 `ENABLE_ARRAY`、`ENABLE_RBTREE` 一样，控制以下内容是否参与编译：

- Hash 类型定义；
- Hash 函数声明与实现；
- `global_hash`；
- Hash 协议分支；
- Hash 的初始化与销毁。

宏开关的目的不是只隐藏一个结构体，而是完整地装上或拆下整个引擎。

### 3.2 Hash 实例在项目中的最小理解

```c
typedef struct hashnode_s {
    char *key;
    char *value;
    struct hashnode_s *next;
} hashnode_t;

typedef struct hashtable_s {
    hashnode_t **nodes;
    int max_slots;
    int count;
} kvs_hash_t;
```

从 KV 项目角度只需知道：

```text
global_hash
├── nodes      桶数组，每个元素是一条冲突链的头指针
├── max_slots  桶数量
└── count      当前键值对数量
```

`nodes` 是二级指针，因为它先表示桶数组，再由每个桶元素指向节点：

```text
nodes[0] → node → node → NULL
nodes[1] → NULL
nodes[2] → node → NULL
```

发生 Hash 冲突时，相同桶下标的节点通过 `next` 串在一起。这里只把它作为理解资源释放和实例生命周期的前提，不展开 Hash 算法。

### 3.3 对外仍然使用已有的 5+2 形状

```c
int   kvs_hash_create(kvs_hash_t *hash);
void  kvs_hash_destroy(kvs_hash_t *hash);
int   kvs_hash_set(kvs_hash_t *hash, char *key, char *value);
char *kvs_hash_get(kvs_hash_t *hash, char *key);
int   kvs_hash_del(kvs_hash_t *hash, char *key);
int   kvs_hash_mod(kvs_hash_t *hash, char *key, char *value);
int   kvs_hash_exist(kvs_hash_t *hash, char *key);
```

5+2 的设计价值和 RBTree 接入方式已在上一篇说明。这里的新增认识是：**Hash 内部可以完全不同，但仍然服从项目已经形成的引擎契约。**

---

## 4. 第二个接入点：Makefile

本阶段开始把每个 `.c` 文件单独编译成 `.o`。Hash 必须进入对象文件集合：

```make
OBJS = kvstore.o reactor.o proactor.o kvs_array.o \
       ntyco.o kvs_rbtree.o kvs_hash.o

kvs_hash.o: kvs_hash.c kvstore.h
	$(CC) $(CPPFLAGS) $(CFLAGS) -c kvs_hash.c -o kvs_hash.o

kvstore: $(OBJS)
	$(CC) -o $@ $(OBJS) $(LDFLAGS) $(LDLIBS)
```

原笔记中的 `OBJS` 没有 `kvs_hash.o`。这意味着即使 `kvs_hash.c` 已经写好，最终的 `kvstore` 也没有编译、链接 Hash 实现。

建议把参数按职责拆开：

```make
CC       = gcc
CPPFLAGS = -I./NtyCo/core
CFLAGS   = -Wall -Wextra -g
LDFLAGS  = -L./NtyCo
LDLIBS   = -luring -lntyco -lpthread -ldl
```

相对上一篇 Makefile 笔记，本阶段只需新增一个检查点：

> 每增加一个 KV 引擎，都要确认它的目标文件同时进入编译规则和最终的 `OBJS` 链接集合。

如果还要分别生成 `kvstore`、NtyCo 库和 `testcase`，可设为三个明确目标：

```make
.PHONY: all ntyco clean

all: ntyco kvstore testcase

ntyco:
	$(MAKE) -C NtyCo
```

`testcase` 应有自己的对象文件和链接规则，不要与服务端的 `main()` 混在同一个目标中。

---

## 5. 第三个接入点：实例生命周期

### 5.1 定义全局实例

```c
#if ENABLE_HASH
static kvs_hash_t global_hash;
#endif
```

它与已有的 `global_array`、`global_rbtree` 并列，是 Hash 引擎在当前进程中的状态容器。

### 5.2 网络服务启动前创建

```c
#if ENABLE_HASH
if (kvs_hash_create(&global_hash) != 0) {
    /* 初始化失败，不能继续提供 Hash 命令 */
}
#endif
```

必须先创建引擎，再进入网络循环。否则客户端已经能够发来 `HSET`，协议层却会访问尚未初始化的桶数组。

### 5.3 网络服务停止后销毁

```c
#if ENABLE_HASH
kvs_hash_destroy(&global_hash);
#endif
```

完整顺序为：

```text
创建 Array / RBTree / Hash
        ↓
启动所选网络层
        ↓
处理客户端命令
        ↓
网络层退出
        ↓
销毁 Hash / RBTree / Array
```

如果网络启动函数永不返回，还需要设计退出信号和优雅关闭流程，否则 `destroy` 虽然写在 `main()` 中，正常运行时却永远不会执行。

---

## 6. 第四个接入点：协议命令

上一篇已经为 RBTree 增加 `RSET/RGET/RDEL/RMOD/REXIST`。教学阶段为了让多个引擎同时存在，可以继续采用同一规则：

| Array | RBTree | Hash | 语义 |
|---|---|---|---|
| `SET` | `RSET` | `HSET` | 新增键值对 |
| `GET` | `RGET` | `HGET` | 查询 value |
| `DEL` | `RDEL` | `HDEL` | 删除键值对 |
| `MOD` | `RMOD` | `HMOD` | 修改已有 value |
| `EXIST` | `REXIST` | `HEXIST` | 判断 key 是否存在 |

协议层识别到 `HSET` 后调用：

```c
kvs_hash_set(&global_hash, tokens[1], tokens[2]);
```

其余命令同理。协议层负责将引擎内部返回值翻译成项目已有响应：

```text
OK\r\n
EXIST\r\n
NO EXIST\r\n
具体 value\r\n
ERROR\r\n
```

需要保持的边界是：

```text
协议层：参数数量、命令选择、响应文本
Hash 层：key/value 保存、查询、修改、删除、资源管理
```

协议层不能直接访问 `global_hash.nodes`，否则 Hash 内部实现一改，协议代码也必须跟着改，统一接口就失去了意义。

### 一条 `HSET` 的新增路径

前三篇已经讲过网络路径和 `SET` 完整路径，此处只看 Hash 分支的差异：

```text
HSET Teacher King
  → 协议层识别 HSET
  → 检查 token 数量为 3
  → kvs_hash_set(&global_hash, "Teacher", "King")
  → 返回内部状态码
  → 协议层生成 OK / EXIST / ERROR
```

从网络层到 token 切分的部分与以前完全相同，不需要为 Hash 重写。

---

## 7. 当前 Hash 代码必须修正的工程问题

这些问题的重点是内存所有权和项目稳定性，而不是 Hash 算法。

### 7.1 创建后必须把桶数组清零

```c
hash->nodes = kvs_malloc(sizeof(hashnode_t *) * hash->max_slots);
memset(hash->nodes, 0,
       sizeof(hashnode_t *) * hash->max_slots);
```

`kvs_malloc` 返回的内存默认包含不确定值。不清零时，第一次读取 `hash->nodes[idx]` 就可能取得随机地址。

### 7.2 下标计算应使用实例容量

当前各操作固定使用 `MAX_TABLE_SIZE`：

```c
int idx = _hash(key, MAX_TABLE_SIZE);
```

应改为：

```c
int idx = _hash(key, hash->max_slots);
```

否则 `max_slots` 字段形同虚设，将来改变容量或扩容时也会出错。

### 7.3 创建节点失败要逆序回滚

指针模式下，一个节点的申请顺序是：

```text
node → key 副本 → value 副本
```

如果 value 申请失败，必须释放已经成功申请的 key 和 node。当前代码释放的是本来就为 `NULL` 的 `kvalue`，遗漏了前两块内存。

`kvs_hash_set()` 也要判断 `_create_node()` 是否返回 `NULL`，不能直接执行 `new_node->next`。

### 7.4 三条释放路径必须一致

开启 `ENABLE_KEY_POINTER` 后，一个节点拥有三块内存：

```text
node
├── key
└── value
```

以下三条路径都必须依次释放 `key`、`value`、`node`：

1. 删除桶的头节点；
2. 删除桶内的普通节点；
3. 销毁整张 Hash 表。

当前普通节点删除路径释放完整，但头节点删除和 `destroy` 只释放了 node，会发生内存泄漏。

### 7.5 MOD 要先申请，再替换

当前代码先 `free(node->value)`，然后申请新 value。如果申请失败，旧值已经丢失，节点中还留下悬空指针。

安全顺序是：

```text
申请新 value
  → 复制成功
  → 保存 old_value
  → node->value = new_value
  → free(old_value)
```

这样内存申请失败时，旧值仍然有效。

### 7.6 其他接口一致性问题

- `kvs_hash_mod()` 还要检查 `value != NULL`。
- 固定数组模式下，`strncpy` 后必须保证字符串以 `\0` 结束。
- `kvs_hash_count()` 已实现，但头文件中缺少声明。
- `destory` 应逐步统一为正确拼写 `destroy`。
- `kvs_hash_set()` 的参数类型建议统一写成 `kvs_hash_t *`。
- `exist` 的 `0=存在、1=不存在` 必须与 Array、RBTree 和协议响应保持一致。
- 销毁后应将 `nodes=NULL`、`max_slots=0`、`count=0`，降低误用风险。

---

## 8. 多语言支持：本阶段新增的认识

上一篇已经介绍过 TCP 客户端测试，第二篇也解释过消息边界，因此这里不再重复 socket 收发和粘包、拆包原理。

本阶段要建立的新认识是：

> Go、Java、Node.js、Python 和 Rust 并不直接调用 C 的 Hash 函数；它们都是同一个文本协议的客户端。

| 语言 | TCP 客户端 API | 发出的内容 |
|---|---|---|
| Go | `net.Dial` | UTF-8 协议文本 |
| Java | `Socket` | UTF-8 协议文本 |
| Node.js | `net.createConnection` | UTF-8 协议文本 |
| Python | `socket.socket` | UTF-8 协议文本 |
| Rust | `TcpStream` | UTF-8 协议文本 |

五种语言的差异只在 socket API，服务端看到的都应该是：

```text
HSET Teacher King\r\n
HGET Teacher\r\n
```

因此“支持第三方语言”的核心不是编写五套服务端适配层，而是稳定以下契约：

- 服务器地址和端口；
- UTF-8 编码；
- 请求结束标记；
- 命令名称和参数数量；
- 响应格式；
- 连接是单请求、长连接还是支持流水线；
- 超时与异常处理方式。

原示例中的 IP 地址有两组：`172.16.145.132` 和 `192.168.243.131`，实际测试时应统一为配置项。

### 用同一场景验证五种语言

不要让五份示例各自测试不同命令。它们应执行同一条状态链：

```text
HSET lang:test C       → OK
HGET lang:test         → C
HMOD lang:test common  → OK
HGET lang:test         → common
HEXIST lang:test       → EXIST
HDEL lang:test         → OK
HGET lang:test         → NO EXIST
```

这样验证的是“协议与 Hash 引擎对所有语言行为一致”，而不只是证明五种语言都能建立 TCP 连接。

正式测试还应给每个客户端加入：

- `send_all/write_all` 语义，防止一次发送没有写完；
- 按 `\r\n` 或长度字段读取完整响应；
- 连接和读取超时；
- 非 UTF-8 或异常响应处理；
- 失败时返回非零退出码，便于自动化测试发现失败。

---

## 9. 课后扩展：把 SkipList 融入 KVEngine

通常称为 **Skip List（跳表）**，不是 SkipTable。

这道动手题可以直接复用 Hash、RBTree 的接入模板：

```text
1. kvstore.h 增加 ENABLE_SKIPLIST
2. 定义 kvs_skiplist_t
3. 声明 SkipList 的 5+2 接口
4. 新增 kvs_skiplist.c
5. Makefile 加入 kvs_skiplist.o
6. 核心层定义 global_skiplist
7. 启动时 create，退出时 destroy
8. 协议层增加 SSET/SGET/SDEL/SMOD/SEXIST
9. 复制已有状态链，编写 SkipList 端到端测试
10. 检查失败回滚、删除和销毁时的内存所有权
```

这能完成“把 SkipList 当作普通 KV 引擎接入”，但还没有发挥它的有序性。

### 9.1 范围查询需要扩展 5+2

原有 5+2 只表达单个 key 的操作，不能表达范围查询。可以增加：

```c
int kvs_skiplist_range(
    kvs_skiplist_t *inst,
    const char *start_key,
    const char *end_key,
    kvs_pair_t *out,
    size_t capacity,
    size_t *count
);
```

协议层增加：

```text
SRANGE user:100 user:200 LIMIT 50
```

设计范围接口时要明确：

- 起止 key 是否包含；
- key 按字符串还是数值排序；
- 最大结果数量；
- 输出缓冲区由谁分配和释放；
- 大结果集如何分页；
- 遍历期间发生修改时怎样保证一致性。

Hash 适合精确点查，但没有顺序；SkipList 和 RBTree 保持 key 有序，更适合范围查询。

---

## 10. 面试题

### 10.1 KV 存储为什么常采用单线程执行命令？

首先修正问题前提：KV 存储不是必须使用单线程。

教学项目或某些内存 KV 的核心命令路径采用单线程，主要因为：

- 非阻塞 I/O 可以让一个事件循环管理大量连接；
- 内存 KV 的单次操作通常很短；
- 串行执行避免共享数据上的锁竞争；
- 不需要频繁进行线程上下文切换；
- 命令顺序和数据一致性更容易理解与调试。

它的限制也很明显：慢命令会阻塞后续请求，CPU 密集型任务无法充分利用多核，单实例吞吐最终受单核限制。

常见扩展方式是按 key 分片，每个分片由单线程串行执行，多个分片并行利用多核；也可以让网络 I/O、持久化、内存回收由不同线程负责。

结合当前项目回答时还要看所选网络模型：如果多个线程可能同时调用协议处理函数并访问同一个 `global_hash`，Hash 就需要互斥锁、分片锁或单写者队列。不能因为包含了 `pthread.h`，就认为数据已经线程安全。

### 10.2 范围查询应选择什么 KVEngine？

范围查询要求 key 有序：

| 引擎 | 范围查询特点 |
|---|---|
| Hash | 无序，适合点查，不适合直接做范围查询 |
| SkipList | 定位起点后沿底层链表扫描，适合本项目扩展 |
| RBTree | 中序遍历有序，也能实现范围查询 |
| B/B+Tree | 适合磁盘页和大规模范围扫描 |
| LSM-Tree | 适合高写入吞吐的持久化 KV |

本项目的课后题应优先使用 SkipList，并在原有 5+2 之外增加 RANGE 接口。

### 10.3 频繁 malloc/free 时怎样管理 KV 内存？

当前项目已经通过 `kvs_malloc/kvs_free` 留出了统一入口。后续可以逐步演进：

1. 在包装层统计分配次数、字节数、峰值和泄漏。
2. 对固定大小的节点使用对象池或 slab。
3. 对不同长度的 key/value 使用 size class 和空闲链表。
4. 一次申请大块内存，再切成多个小对象复用。
5. 多线程下使用线程本地缓存或分片内存池，降低锁竞争。
6. 修改 value 时，如果旧缓冲区容量足够就复用；否则先申请新块，成功后再替换。
7. 设置最大内存、淘汰策略并监控碎片率。

内存池解决的是性能和碎片问题，不能替代正确的所有权规则。无论使用系统分配器还是内存池，都必须明确：谁申请、谁持有、谁释放，以及失败时如何回滚。

---

## 11. 本阶段的实施顺序

```text
统一 Hash 返回值与其他引擎的语义
  → 修正初始化、失败回滚、MOD 和三条释放路径
  → Makefile 加入 kvs_hash.o
  → 添加 global_hash 的 create/destroy
  → 协议层增加 HSET/HGET/HDEL/HMOD/HEXIST
  → 复用上一篇测试状态链验证 Hash
  → 五种语言运行完全相同的协议用例
  → 按相同模板接入 SkipList
  → 单独设计 RANGE 接口和测试
  → 压测后再决定分片、加锁和内存池方案
```

## 总结

本阶段相对于前三篇真正新增的主线是：

```text
Hash 实现
  → 用统一接口包装
  → 进入 Makefile
  → 进入全局生命周期
  → 进入协议分发
  → 被同一测试链验证
  → 通过统一 TCP 协议服务五种语言
```

最重要的认识是：

> 项目扩展的重点不是孤立地增加一种数据结构，而是把它组织成符合现有接口、生命周期、协议和测试约定的 KV 引擎。

SkipList 的课后扩展也应遵循同一思路；只有范围查询属于新的业务能力，需要在原有 5+2 之外明确增加接口。
