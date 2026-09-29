---
title: KV 进化（三）：内存管理与内存池
slug: kv-evolution-memory-management
description: 从 Hash 节点的 malloc/free、悬空指针和内存泄漏讲起，梳理 91kvstore 中内存池的设计、生命周期与测试验证
date: 2026-09-08T00:00:00+08:00
draft: false
image: cover.svg
tags:
  - KV 存储
  - 内存管理
  - 内存池
  - C 语言
  - Hash
categories:
  - 项目记录
---

# 91kvstore 内存池学习与实现笔记

## 1. 从 Hash 节点的内存说起

我们先拿 Hash 数据结构举例。Hash 由节点组成，节点中保存 `key`、`value` 和指向下一个节点的 `next`。

在开启指针形式的 `key/value` 后，一个节点通常会这样创建：

```c
hashnode_t *node = malloc(sizeof(hashnode_t));
node->key = malloc(4);
node->value = malloc(7);
```

`malloc` 的作用是申请一块动态内存。上面三次 `malloc` 会得到三块彼此独立的空间：

```text
node ───→ [hashnode_t]
              │
              ├── key ───→ [key 数据]
              │
              └── value ─→ [value 数据]
```

`node->key` 和 `node->value` 保存的不是字符串本身，而是字符串所在内存的地址。

因此，释放顺序应当是：

```c
free(node->key);
free(node->value);
free(node);
```

如果先执行：

```c
free(node);
```

之后就不能再安全地通过 `node->key` 和 `node->value` 找到对应内存。没有保存其他地址时，那两块内存将无法释放，从而造成内存泄漏。

## 2. 指针变量、动态内存与悬空指针

执行 `free` 后，指针不会自动变成 `NULL`。

```text
释放前：

node->key ───→ ["Dad\0"]

释放后：

node->key ───→ [已经释放的空间]
```

`free` 只是告诉内存分配器：这块动态内存可以重新使用了。保存地址的指针变量不会被自动修改。

```text
指针变量 p                 动态内存
┌────────────┐            ┌──────────┐
│ 地址 0x100 │ ─────────→ │ 10 字节  │
└────────────┘            └──────────┘
```

执行：

```c
free(p);
```

只是释放右边的动态内存，左边的 `p` 仍然保存旧地址。此时 `p` 被称为悬空指针，不能再解引用或再次释放。

如果希望当前指针变量不再保存旧地址，需要手动赋值：

```c
p = NULL;
```

需要注意，`p = NULL` 只改变 `p` 这个变量。如果还有另一个指针也保存了同一地址，它仍然可能是悬空指针。

对同一块内存执行两次 `free` 称为重复释放，可能导致程序崩溃或破坏分配器的内部状态。

## 3. 什么是内存碎片

假设一段内存被切成四块：

```text
[使用 16B][空闲 16B][使用 16B][空闲 16B]
```

空闲内存总量是：

```text
16B + 16B = 32B
```

但是如果程序申请一块连续的 24B 内存，两块空闲区域彼此分开，任何一块都放不下 24B。

这种“空闲总量足够，但是被分散成许多小块”的现象叫作外部碎片。

频繁申请和释放不同大小的对象，可能让内存布局越来越零散。引入内存池的目的之一，就是让相同或相近大小的内存有规律地重复使用。

## 4. 内存池为什么可能更快

假设节点固定为 24 字节，普通方式反复执行：

```c
node = malloc(24);
free(node);

node = malloc(24);
free(node);
```

通用分配器需要处理许多情况：

1. 申请大小可能是任意值。
2. 可能存在多个线程。
3. 需要维护内存块的元数据。
4. 需要查找合适的空闲块。
5. 可能需要拆分或合并内存块。

专用的固定大小内存池提前知道对象大小，可以先申请一大块内存：

```text
一个大块内存
┌──────┬──────┬──────┬──────┐
│ 24B  │ 24B  │ 24B  │ 24B  │
└──────┴──────┴──────┴──────┘
```

然后维护一条空闲链表：

```text
空闲块1 → 空闲块2 → 空闲块3 → NULL
```

申请时取下链表头，释放时再放回链表头。主要操作只是修改几个指针。

内存池只能复用已经被业务释放的对象，不能擅自回收仍然在使用的对象。

空闲块耗尽后，可以采用不同策略：

1. 扩容：再申请一大块内存。
2. 降级：这一笔申请临时使用普通 `malloc`。
3. 失败：返回 `NULL`。

我们的第一版采用自动扩容。

## 5. Chunk、Block 与 Free List

- Chunk：内存池一次向 `malloc` 申请的大块内存。
- Block：从 Chunk 中划分出来、交给业务使用的小块。
- Free List：把所有空闲 Block 串起来的链表。

假设每个 Block 为 32 字节，每个 Chunk 包含 4 个 Block：

```text
一个 Chunk
┌────────┬────────┬────────┬────────┐
│ Block1 │ Block2 │ Block3 │ Block4 │
│  32B   │  32B   │  32B   │  32B   │
└────────┴────────┴────────┴────────┘
```

刚扩容时，所有 Block 都进入空闲链表。连续申请 4 次后，空闲链表变成 `NULL`；第 5 次申请触发新 Chunk 的创建。

## 6. 用空闲 Block 自身保存 next

空闲 Block 中没有有效业务数据，因此可以借用它的前几个字节保存 `next`：

```c
typedef struct free_node {
    struct free_node *next;
} free_node_t;
```

这不是把结构体完整地嵌套进自己。`next` 只是一个地址大小固定的指针。

下面这种写法才是无法成立的完整嵌套：

```c
struct free_node {
    struct free_node next;
};
```

多个 `free_node_t` 通过指针连接起来，就形成了 C 语言中的单向链表：

```text
free_list
    │
    ▼
┌────────┐    ┌────────┐    ┌────────┐
│ next ──────→│ next ──────→│ NULL   │
└────────┘    └────────┘    └────────┘
 Block1        Block2         Block3
```

申请链表头：

```c
free_node_t *block = free_list;
free_list = free_list->next;
return block;
```

归还到链表头：

```c
block->next = free_list;
free_list = block;
```

这种链表信息直接保存在被管理对象内部的方式，叫作侵入式链表。

它要求 Block 至少能放下一个指针：

```c
if (block_size < sizeof(free_node_t)) {
    block_size = sizeof(free_node_t);
}
```

## 7. 为什么还需要 Chunk List

`free_list` 只记录当前哪些 Block 可以再次分配，却不能完整反映系统一共申请过哪些 Chunk。

因此还需要一条 `chunk_list`：

```c
typedef struct memory_chunk {
    void *memory;
    struct memory_chunk *next;
} memory_chunk_t;
```

- `memory` 指向真正存放多个 Block 的大块内存。
- `next` 指向下一个 Chunk 管理节点。

销毁内存池时，遍历 `chunk_list`，逐个释放 Chunk。

## 8. 内存池的数据结构与接口

头文件使用宏避免重复包含：

```c
#ifndef KVS_MEMORY_POOL_H
#define KVS_MEMORY_POOL_H

#include <stddef.h>

typedef struct free_node {
    struct free_node *next;
} free_node_t;

typedef struct memory_chunk {
    void *memory;
    struct memory_chunk *next;
} memory_chunk_t;

typedef struct memory_pool {
    size_t block_size;
    size_t blocks_per_chunk;
    free_node_t *free_list;
    memory_chunk_t *chunk_list;
} memory_pool_t;

int memory_pool_init(memory_pool_t *pool,
                     size_t block_size,
                     size_t blocks_per_chunk);

void *memory_pool_alloc(memory_pool_t *pool);
void memory_pool_free(memory_pool_t *pool, void *ptr);
void memory_pool_destroy(memory_pool_t *pool);

#endif
```

`memory_pool_alloc` 返回 `void *`，因为内存池只负责返回一段通用内存，并不知道调用者最终会把它当成 `hashnode_t`、链表节点还是其他对象。

## 9. 创建独立的 Git 分支

```shell
git switch -c feature/memory-pool
```

它完成两件事：

1. 创建 `feature/memory-pool` 分支。
2. 将当前工作分支切换到新分支。

刚创建时，两个分支指向同一个提交：

```text
main                 → C
feature/memory-pool  → C
```

提交内存池代码后：

```text
main                 → C
feature/memory-pool  → D
```

这样内存池开发不会直接改变 `main` 所指向的稳定版本。

## 10. 初始化内存池

```c
int memory_pool_init(memory_pool_t *pool,
                     size_t block_size,
                     size_t blocks_per_chunk) {
    if (pool == NULL || block_size == 0 || blocks_per_chunk == 0) {
        return -1;
    }

    if (block_size < sizeof(free_node_t)) {
        block_size = sizeof(free_node_t);
    }

    size_t alignment = sizeof(void *);
    size_t remainder = block_size % alignment;

    if (remainder != 0) {
        block_size += alignment - remainder;
    }

    pool->block_size = block_size;
    pool->blocks_per_chunk = blocks_per_chunk;
    pool->free_list = NULL;
    pool->chunk_list = NULL;

    return 0;
}
```

初始化函数完成四件事：

1. 检查参数是否合法。
2. 保证 Block 至少能保存一个 `free_node_t`。
3. 将 Block 大小向上对齐。
4. 保存配置并清空两条链表。

初始化阶段没有申请 Chunk。我们采用懒加载：第一次申请 Block 时再扩容。

## 11. 为什么要对齐 Block

每个空闲 Block 都会被转换为 `free_node_t *`，并在其中保存指针，因此每个 Block 的起始地址需要满足指针的基本对齐要求。

假设指针大小是 8 字节，用户要求的 Block 大小是 20 字节：

```text
未对齐的起始偏移：0、20、40、60……
```

将 20 向上对齐到 24：

```text
20 % 8 = 4
20 + (8 - 4) = 24
```

对齐后的起始偏移为：

```text
0、24、48、72……
```

它们都是 8 的整数倍。

## 12. 扩容前的整数溢出检查

一个 Chunk 的大小为：

```c
chunk_size = blocks_per_chunk * block_size;
```

两个 `size_t` 相乘可能超过 `SIZE_MAX`，因此乘法前检查：

```c
if (pool->blocks_per_chunk > SIZE_MAX / pool->block_size) {
    return -1;
}
```

这里不是说 Block 数量太多时应该继续扩容，而是说这次乘法已经无法得到可信结果，申请参数不合法，必须失败。

`SIZE_MAX` 由标准整数类型相关头文件提供：

```c
#include <stdint.h>
```

## 13. 实现 Chunk 扩容

```c
static int memory_pool_grow(memory_pool_t *pool) {
    if (pool->blocks_per_chunk > SIZE_MAX / pool->block_size) {
        return -1;
    }

    size_t chunk_size =
        pool->blocks_per_chunk * pool->block_size;

    memory_chunk_t *chunk = malloc(sizeof(*chunk));
    if (chunk == NULL) {
        return -1;
    }

    chunk->memory = malloc(chunk_size);
    if (chunk->memory == NULL) {
        free(chunk);
        return -1;
    }

    chunk->next = pool->chunk_list;
    pool->chunk_list = chunk;

    unsigned char *start = (unsigned char *)chunk->memory;

    for (size_t i = 0; i < pool->blocks_per_chunk; i++) {
        free_node_t *node =
            (free_node_t *)(start + i * pool->block_size);

        node->next = pool->free_list;
        pool->free_list = node;
    }

    return 0;
}
```

这里有两次 `malloc`：

```c
memory_chunk_t *chunk = malloc(sizeof(*chunk));
chunk->memory = malloc(chunk_size);
```

第一次申请 Chunk 管理结构，第二次申请真正存放 Block 的大块内存。

如果第二次失败，要释放第一次已经成功申请的 `chunk`，否则会发生内存泄漏。

新 Chunk 使用头插法加入 `chunk_list`：

```c
chunk->next = pool->chunk_list;
pool->chunk_list = chunk;
```

## 14. 划分 Block 的循环究竟做了什么

```c
unsigned char *start = (unsigned char *)chunk->memory;

for (size_t i = 0; i < pool->blocks_per_chunk; i++) {
    free_node_t *node =
        (free_node_t *)(start + i * pool->block_size);

    node->next = pool->free_list;
    pool->free_list = node;
}
```

这个循环没有继续分配内存。真正的内存申请只有：

```c
chunk->memory = malloc(chunk_size);
```

循环只是计算各个 Block 的起始地址，并把它们登记到空闲链表。

`unsigned char` 的大小是一个字节，把地址转换成 `unsigned char *` 后，便可以按照字节偏移计算地址。

假设：

```text
start = 1000
block_size = 32
```

那么：

```text
i = 0：1000 + 0 × 32 = 1000
i = 1：1000 + 1 × 32 = 1032
i = 2：1000 + 2 × 32 = 1064
i = 3：1000 + 3 × 32 = 1096
```

因此：

> 不同 Chunk 之间不保证连续，但是同一个 Chunk 内部的 Block 是连续的。

## 15. 再次理解头插法

假设当前链表为：

```text
free_list → B → C → NULL
```

现在需要插入 A：

```c
A->next = free_list;
free_list = A;
```

第一句让 A 指向原来的表头 B：

```text
A → B → C → NULL
```

第二句让表头改为 A：

```text
free_list → A → B → C → NULL
```

扩容循环不断进行相同操作，因此地址按从低到高计算出来，但最终链表顺序会被反转。

## 16. 从内存池申请 Block

```c
void *memory_pool_alloc(memory_pool_t *pool) {
    if (pool == NULL) {
        return NULL;
    }

    if (pool->free_list == NULL) {
        if (memory_pool_grow(pool) != 0) {
            return NULL;
        }
    }

    free_node_t *node = pool->free_list;
    pool->free_list = node->next;

    return (void *)node;
}
```

自动扩容来自：

```c
if (pool->free_list == NULL) {
    memory_pool_grow(pool);
}
```

扩容成功后不能直接 `return NULL`，因为此时 `free_list` 已经重新拥有空闲 Block，应该继续向下取出一个。

假设：

```text
free_list → A → B → C → NULL
```

执行：

```c
free_node_t *node = pool->free_list;
pool->free_list = node->next;
```

结果为：

```text
返回 node → A
free_list → B → C → NULL
```

## 17. 将 Block 放回内存池

```c
void memory_pool_free(memory_pool_t *pool, void *ptr) {
    if (pool == NULL || ptr == NULL) {
        return;
    }

    free_node_t *node = (free_node_t *)ptr;
    node->next = pool->free_list;
    pool->free_list = node;
}
```

这里没有调用系统 `free()`，只是把 Block 重新插入空闲链表头部。

```text
释放前：free_list → B → C → NULL
释放 A：free_list → A → B → C → NULL
```

下一次申请会直接取出 A，这就是自动复用的来源。

## 18. 销毁整个内存池

```c
void memory_pool_destroy(memory_pool_t *pool) {
    if (pool == NULL) {
        return;
    }

    memory_chunk_t *chunk = pool->chunk_list;

    while (chunk != NULL) {
        memory_chunk_t *next = chunk->next;

        free(chunk->memory);
        free(chunk);

        chunk = next;
    }

    pool->block_size = 0;
    pool->blocks_per_chunk = 0;
    pool->free_list = NULL;
    pool->chunk_list = NULL;
}
```

必须在释放当前 Chunk 前保存：

```c
memory_chunk_t *next = chunk->next;
```

如果先 `free(chunk)`，之后再访问 `chunk->next`，就是访问已经释放的内存。

不需要逐个释放 Block，因为 Block 不是分别调用 `malloc` 得到的。释放整个 `chunk->memory` 后，其中所有 Block 会一起失效。

## 19. 使用 ASan 测试

测试中设置：

```c
memory_pool_init(&pool, 32, 2);
```

`blocks_per_chunk = 2` 表示每次扩容创建两个 Block。

Makefile 中的测试目标开启 AddressSanitizer：

```make
test-memory-pool: $(BIN_DIR)/test_memory_pool
	./$(BIN_DIR)/test_memory_pool

$(BIN_DIR)/test_memory_pool: \
	$(TEST_DIR)/test_memory_pool.c \
	$(MEMORY_DIR)/memory_pool.c \
	$(INCLUDE_DIR)/memory_pool.h | $(BIN_DIR)
	$(CC) $(CPPFLAGS) $(CFLAGS) -Werror \
		-fsanitize=address -fno-omit-frame-pointer \
		$(TEST_DIR)/test_memory_pool.c \
		$(MEMORY_DIR)/memory_pool.c \
		-o $@
```

运行：

```shell
make test-memory-pool
```

测试输出：

```text
a = 0x506000000040
b = 0x506000000020
c = 0x506000000040 (same as a)
d = 0x5060000000a0 (new chunk)
memory pool test passed
```

## 20. 分析测试结果

`a` 与 `b` 的地址相差：

```text
0x40 - 0x20 = 0x20 = 32
```

这与 `block_size = 32` 一致，说明它们是同一个 Chunk 中相邻的 Block。

释放 A 后再次申请 C：

```text
c == a
```

说明被释放的 A 已经放回空闲链表，并被下一次申请成功复用。

A、B 对应的两个 Block 都被占用后，继续申请 D。此时 `free_list == NULL`，触发 `memory_pool_grow()`：

```text
d = 0x5060000000a0
```

D 位于新的地址区域，说明第二个 Chunk 创建成功。

## 21. ASan 是什么，后续是否关闭

AddressSanitizer 简称 ASan，可以帮助发现：

- 内存越界。
- 使用已经释放的内存。
- 重复释放。
- 释放非法地址。
- 内存泄漏。

我们的 ASan 只在 `test-memory-pool` 目标中开启。正常的 KV Store 构建没有自动开启。

未来测试 QPS、虚拟内存和物理内存时必须关闭 ASan，因为它会加入额外检查和内存区域，影响性能与内存数据。

## 22. 当前已经完成的内容

```text
固定大小内存池
├── 参数初始化
├── Block 大小对齐
├── Chunk 自动扩容
├── Block 连续划分
├── Free List 分配
├── Block 回收与复用
├── Chunk List 统一销毁
├── ASan 单元测试
└── Makefile 测试目标
```

代码已经提交到独立分支：

```text
feature/memory-pool
```

提交为：

```text
feat(memory): add fixed-size memory pool
```

## 23. 为什么现在不能直接替换所有 kvs_malloc

当前实现的是固定大小内存池。一个池只能返回同样大小的 Block。

但是通用接口可能收到各种尺寸：

```c
kvs_malloc(16);
kvs_malloc(100);
kvs_malloc(4096);
```

单个固定大小池无法合理处理这些不同大小的申请，所以不能直接在 `kvs_malloc()` 内无条件调用当前的 `memory_pool_alloc()`。

可变大小的内存管理当然存在，例如：

- 多个固定大小池组成的大小类别。
- Arena 或区域分配器。
- Buddy allocator。
- Slab allocator。
- jemalloc 这样的通用分配器。

jemalloc 内部会管理多个大小类别，因此能够提供类似 `malloc/free` 的通用接口。

## 24. 下一步为什么先集成 Hash 节点

Hash 中主要涉及三类动态内存：

```text
hashnode_t：大小固定
key：长度不固定
value：长度不固定
```

当前固定大小内存池最适合管理 `hashnode_t`。

第一版的分工应该是：

```text
hashnode_t → memory_pool_alloc / memory_pool_free

key/value  → 继续使用 kvs_malloc / kvs_free
```

未来的总体结构可以是：

```text
业务代码
   │
   ├── hashnode_t
   │       │
   │       └── 固定大小内存池
   │
   └── key/value 等可变大小数据
           │
           └── kvs_malloc/kvs_free
                       │
                ┌──────┴──────┐
                │             │
             glibc malloc   jemalloc
```

这样可以进行有意义的对比：

1. 全部使用系统 `malloc/free`。
2. Hash 节点使用自研固定大小内存池。
3. 通用分配切换为 jemalloc。

之后分别测量：

- QPS。
- 虚拟内存。
- 物理内存。
- 分配与释放的耗时。

当前进度正好停在这里：

> 内存池已经实现、测试并提交。下一步开始将它接入 Hash 节点的创建、删除和 Hash 销毁流程。

现在开始接入hash的具体流程

首当其冲的 是 将memorypool的头文件 加入的kv.h里 
此外 在对应hash.c 的文件开头 定义 一次性 扩建的 blocks数量
#define HASH_NODE_BLOCKS_PER_CHUNK 1024

在hash数据结构定义的地方 
就是那个结构体 多加一个 memorypool 类型的 成员变量 可以直接写 而不是加*避免还要 申请内存的繁琐

在hash_create 函数 参数验证之后 初始化我们的内存池  顺带参数验证```
  if (memory_pool_init(&hash->node_pool,
                         sizeof(hashnode_t),
                         HASH_NODE_BLOCKS_PER_CHUNK) != 0) {
        return -1;
    }

## 25. 将内存池嵌入 Hash 结构

在 Hash 的管理结构中加入节点内存池：

```
typedef struct hashtable_s {
    hashnode_t **nodes;
    int max_slots;
    int count;

    memory_pool_t node_pool;
} kvs_hash_t;
```

这里的：

```
memory_pool_t node_pool;
```

没有 `*`，表示 `node_pool` 管理结构直接嵌入 `kvs_hash_t`。

```
kvs_hash_t
├── nodes
├── max_slots
├── count
└── node_pool
    ├── block_size
    ├── blocks_per_chunk
    ├── free_list
    └── chunk_list
```

创建 `kvs_hash_t` 时，`node_pool` 也已经拥有存储空间，不需要再执行：

```
malloc(sizeof(memory_pool_t));
```

但是“拥有结构体空间”不等于“内存池已经初始化”，所以仍然需要调用：

```
memory_pool_init(&hash->node_pool,
                 sizeof(hashnode_t),
                 HASH_NODE_BLOCKS_PER_CHUNK);
```

这里必须使用 `&`：

```
&hash->node_pool
```

因为 `memory_pool_init()` 需要的是：

```
memory_pool_t *
```

而 `hash->node_pool` 本身是一个 `memory_pool_t` 对象，使用 `&` 才能取得它的地址。

## 26. 初始化 Hash 节点内存池

每个 Chunk 暂定包含 1024 个 Hash 节点 Block：

```
#define HASH_NODE_BLOCKS_PER_CHUNK 1024
```

在 `kvs_hash_create()` 中初始化：

```
if (memory_pool_init(&hash->node_pool,
                     sizeof(hashnode_t),
                     HASH_NODE_BLOCKS_PER_CHUNK) != 0) {
    return -1;
}
```

参数含义：

```
&hash->node_pool
    要初始化哪个内存池

sizeof(hashnode_t)
    每个 Block 能够存放一个 hashnode_t

HASH_NODE_BLOCKS_PER_CHUNK
    每次扩容产生 1024 个 Block
```

此时仍然没有真正申请 Chunk，因为当前内存池采用懒加载。第一次创建 Hash 节点时才会触发扩容。

## 27. 为 Hash 节点封装存储策略

为了能够在“内存池”和“普通 malloc”之间切换，加入编译开关：

```
#ifndef KVS_HASH_USE_MEMORY_POOL
#define KVS_HASH_USE_MEMORY_POOL 1
#endif
```

含义：

```
KVS_HASH_USE_MEMORY_POOL = 1
    Hash 节点来自内存池

KVS_HASH_USE_MEMORY_POOL = 0
    Hash 节点来自 kvs_malloc
```

节点申请被封装为一个独立函数：

```
static hashnode_t *hash_node_alloc(kvs_hash_t *hash) {
    if (hash == NULL) {
        return NULL;
    }

#if KVS_HASH_USE_MEMORY_POOL
    return (hashnode_t *)
        memory_pool_alloc(&hash->node_pool);
#else
    return (hashnode_t *)
        kvs_malloc(sizeof(hashnode_t));
#endif
}
```

节点存储空间的释放也统一封装：

```
static void hash_node_storage_free(kvs_hash_t *hash,
                                   hashnode_t *node) {
    if (hash == NULL || node == NULL) {
        return;
    }

#if KVS_HASH_USE_MEMORY_POOL
    memory_pool_free(&hash->node_pool, node);
#else
    kvs_free(node);
#endif
}
```

这样业务代码不用到处分散编写条件编译：

```
#if KVS_HASH_USE_MEMORY_POOL
...
#else
...
#endif
```

`_create_node()` 只负责调用：

```
hashnode_t *node = hash_node_alloc(hash);
```

其他地方只负责调用：

```
hash_node_storage_free(hash, node);
```

这相当于在 Hash 业务逻辑与底层内存策略之间加了一层隔离。

## 28. Hash 节点和 key/value 使用不同的管理方式

当前只把固定大小的 `hashnode_t` 接入内存池：

```
hashnode_t
    → memory_pool_alloc
    → memory_pool_free

key/value
    → kvs_malloc
    → kvs_free
```

因此释放完整节点时，顺序仍然是：

```
static void _hash_node_free(kvs_hash_t *hash,
                            hashnode_t *node) {
    if (hash == NULL || node == NULL) {
        return;
    }

#if ENABLE_KEY_POINTER
    kvs_free(node->key);
    kvs_free(node->value);
#endif

    hash_node_storage_free(hash, node);
}
```

不能先把 `node` 放回内存池，再访问：

```
node->key
node->value
```

因为 `memory_pool_free()` 会把这个 Block 的前几个字节用作空闲链表的 `next`，原来的节点内容不再属于业务代码。

## 29. 删除节点时必须先断开链表

删除 Hash 冲突链中的头节点时，曾经出现过一个错误：

```
_hash_node_free(hash, head);
```

只释放节点，却没有修改桶数组中的表头指针。

正确顺序是：

```
hash->nodes[idx] = head->next;
_hash_node_free(hash, head);
```

执行过程：

```
释放前：

hash->nodes[idx] → head → next → NULL
```

先更新表头：

```
hash->nodes[idx] ───────→ next → NULL
                          ↑
head 即将被释放
```

再释放 `head`：

```
_hash_node_free(hash, head);
```

删除中间节点时同样要先保存目标节点并断开连接：

```
hashnode_t *tmp = cur->next;
cur->next = tmp->next;
_hash_node_free(hash, tmp);
```

这个原则可以概括为：

> 先保证数据结构可以延续，再清除旧节点。

如果先释放节点，但链表或桶数组仍然指向它，就会留下悬空指针。

之前的 ASan 错误正是由这个问题导致的：

```
attempting free on address which was not malloc()-ed
```

原因是已经归还内存池的 Block 后来被复用，但 Hash 桶仍然保存旧地址，销毁时又把它当成有效业务节点处理。

## 30. 销毁 Hash 与销毁内存池

销毁 Hash 时，需要依次处理：

```
节点中的 key/value
→ Hash 桶数组
→ 内存池的全部 Chunk
→ 清空 Hash 状态
```

核心流程：

```
void kvs_hash_destory(kvs_hash_t *hash) {
    if (hash == NULL) {
        return;
    }

    for (int i = 0; i < hash->max_slots; i++) {
        hashnode_t *node = hash->nodes[i];

        while (node != NULL) {
            hashnode_t *next = node->next;

            _hash_node_free(hash, node);
            node = next;
        }

        hash->nodes[i] = NULL;
    }

    kvs_free(hash->nodes);
    hash->nodes = NULL;

#if KVS_HASH_USE_MEMORY_POOL
    memory_pool_destory(&hash->node_pool);
#endif

    hash->max_slots = 0;
    hash->count = 0;
}
```

这里仍然要在释放当前节点前保存：

```
hashnode_t *next = node->next;
```

否则释放后再访问 `node->next` 就属于使用已经释放的内存。

需要注意，当前项目中的函数名实际拼写为：

```
destory
```

标准英文拼写应当是：

```
destroy
```

以后可以统一重命名，但必须同时修改声明、定义和所有调用位置。

## 31. Hash 与内存池的 ASan 集成测试

集成测试验证了以下内容：

1. Hash 能够成功创建。
2. 第一次插入节点会触发 Chunk 懒加载。
3. `HSET/HGET/HDEL` 行为正确。
4. 删除节点后，Block 会归还空闲链表。
5. 再次插入时能够复用刚释放的 Block。
6. 销毁 Hash 后，桶数组和内存池状态被清空。
7. ASan 没有发现非法释放、越界或内存泄漏。

内存池模式输出：

```
first node  = 0x52b0000061e8
second node = 0x52b0000061e8
memory pool reused the released node block
hash memory pool integration test passed
```

两个地址相同，说明第二次创建节点时复用了第一次删除后归还的 Block。

普通 `malloc` 模式输出：

```
first node  = 0x503000000040
second node = 0x503000000070
malloc mode does not require address reuse
hash memory pool integration test passed
```

两个地址不同也不是错误，因为 `malloc` 不保证下一次申请一定返回刚释放的地址。

因此测试中的地址断言只能在内存池模式启用：

```
#if KVS_HASH_USE_MEMORY_POOL
assert(second_node == first_node);
#endif
```

## 32. 同一份测试验证两种内存策略

Hash 的通用功能在两种模式下都必须成立：

```
assert(kvs_hash_set(&hash, "name", "Jasper") == 0);
assert(kvs_hash_get(&hash, "name") != NULL);
assert(kvs_hash_del(&hash, "name") == 0);
```

内存池内部状态只在内存池模式检查：

```
#if KVS_HASH_USE_MEMORY_POOL
assert(hash.node_pool.chunk_list != NULL);
assert(hash.node_pool.free_list != NULL);
#endif
```

测试逻辑因此被分为两部分：

```
Hash 通用行为
    两种模式都检查

内存池专属行为
    只在内存池模式检查
```

只有先证明两个版本的业务功能一致，后面的性能对比才有意义。

## 33. 使用 Makefile 切换两种服务器版本

Makefile 中加入：

```
HASH_USE_MEMORY_POOL ?= 1
CPPFLAGS += -DKVS_HASH_USE_MEMORY_POOL=$(HASH_USE_MEMORY_POOL)
```

其中：

```
HASH_USE_MEMORY_POOL ?= 1
```

表示外部没有指定时，默认值为 `1`。

编译内存池版本：

```
make clean
make HASH_USE_MEMORY_POOL=1 all testcase
```

编译普通 `malloc` 版本：

```
make clean
make HASH_USE_MEMORY_POOL=0 all testcase
```

切换模式时必须执行 `make clean`。

原因是 Make 通常根据源文件和目标文件的修改时间判断是否需要重新编译，它不知道编译参数中的宏已经发生变化。

另外：

```
make HASH_USE_MEMORY_POOL=0
```

只会影响当前这一次 `make`，不会永久修改 Makefile 变量。

## 34. 压测前移除调试输出

客户端原来每个成功请求都会打印：

```
printf("==>PASS-> %s\n", casename);
```

大量终端输出会严重干扰 QPS，因此压测时将其注释：

```
// printf("==>PASS-> %s\n", casename);
```

失败输出仍然保留，这样请求结果错误时测试会立即终止。

服务器中还存在协议解析调试输出：

```
printf("idx:%d,%s\n", idx, token);
```

它位于 `src/kvstore.c`，同样被注释：

```
// printf("idx:%d,%s\n", idx, token);
```

性能测试时，客户端和服务器都不应该为每条请求打印日志。

## 35. 将 Hash 压测集中到申请和释放

原来的 Hash 测试每轮包含 9 个命令：

```
HSET
HGET
HMOD
HGET
HEXIST
HDEL
HGET
HMOD
HEXIST
```

其中只有：

```
HSET → 创建节点
HDEL → 删除节点
```

直接涉及节点内存的申请和释放。

其他 7 个请求会稀释内存分配策略对 QPS 的影响。因此第一版定向压测改为：

```
for (int i = 0; i < count; i++) {
    testcase(connfd,
             "HSET Dad Jasper",
             "OK\r\n",
             "HSET-Dad");

    testcase(connfd,
             "HDEL Dad",
             "OK\r\n",
             "HDEL-Dad");
}
```

当前设置：

```
int count = 10000;
```

因此实际请求总数为：

```
10000 × 2 = 20000
```

QPS 不能继续使用原来的 `90000`，而应该根据实际请求数计算：

```
int total_requests = count * 2;
int qps = total_requests * 1000 / time_used;
```

这里故意重复使用同一个 key，是为了让业务状态始终在以下两种状态之间切换：

```
不存在节点
    ↓ HSET
存在一个节点
    ↓ HDEL
不存在节点
```

这样可以集中观察节点反复申请和释放的开销。

## 36. 隔离快照与 AOF 对压测的影响

服务器启动时会读取：

```
kvs_snapshot_load("snapshot.db", kvs_protocol);
kvs_aof_open("appendonly.aof");
```

这两个都是相对路径。

如果直接从项目目录启动服务器，旧的快照和 AOF 可能影响：

- 初始节点数量。
- Hash 冲突链长度。
- RSS 和 VIRT。
- 测试 key 是否已经存在。
- 两次实验的公平性。

因此使用独立的临时工作目录启动服务器：

```
mkdir /tmp/kvbench-pool-churn1
cd /tmp/kvbench-pool-churn1
/home/jaspersao/dev/91kvstore/bin/kvstore
```

普通 `malloc` 版本使用另一个目录：

```
mkdir /tmp/kvbench-malloc-churn1
cd /tmp/kvbench-malloc-churn1
/home/jaspersao/dev/91kvstore/bin/kvstore
```

虽然可执行文件位于项目目录，但服务器的当前工作目录位于 `/tmp`，所以相对路径文件会创建在对应的临时目录中：

```
/tmp/kvbench-*/snapshot.db
/tmp/kvbench-*/appendonly.aof
```

这不会移动或覆盖项目目录中的原始持久化数据。

## 37. 第一轮定向 QPS 对比

测试条件：

```
每种模式都从空数据启动
使用相同客户端
使用相同协议
执行 10000 轮
每轮一次 HSET 和一次 HDEL
总请求数为 20000
关闭逐请求成功日志
关闭服务器 idx 调试日志
未开启 ASan
```

第一轮结果：

|内存策略|请求数|耗时|QPS|
|---|---|---|---|
|普通 `malloc/free`|20000|3658 ms|5467|
|自研固定大小内存池|20000|3688 ms|5422|

内存池版本在这一轮中大约慢：

```
(5467 - 5422) / 5467 ≈ 0.8%
```

0.8% 的差距很小，暂时应当认为两者基本持平，不能据此断言 `malloc` 一定更快。

当前实验没有证明内存池能够提升整个服务器的 QPS。

## 38. 为什么当前 QPS 没有明显提升

可能原因包括：

1. 测试包含 TCP 网络收发。
2. 每条命令还需要协议解析。
3. `HSET/HDEL` 可能写入 AOF。
4. 内存池只管理 `hashnode_t`。
5. key 和 value 仍然使用 `malloc/free`。
6. glibc 的小对象分配本身已经经过优化。
7. 虚拟机调度会带来一定波动。
8. 当前只进行了一轮对比。
9. 当前构建使用 `-g`，还没有使用统一的发布优化参数。

因此当前测试测量的是：

```
客户端
→ TCP
→ 网络模块
→ 协议解析
→ Hash
→ 内存管理
→ AOF
→ 返回响应
```

内存池只是整条链路中的一个环节，它的差异可能被其他开销淹没。

## 39. 服务器整体性能与分配器性能需要分开测

后续应该保留两类实验。

第一类是端到端测试：

```
客户端 → 网络 → KV Server → Hash → 内存管理
```

它回答：

> 接入内存池后，真实服务器整体 QPS 是否提高？

第二类是进程内测试：

```
直接调用 Hash 创建和删除
不经过网络
不写 AOF
```

它回答：

> 单独观察节点申请和释放时，内存池本身是否比 malloc 更快？

如果进程内测试更快，但服务器整体 QPS 没变化，说明瓶颈不在节点分配。

如果进程内测试也没有变快，则需要继续检查：

- Chunk 大小是否合适。
- 函数调用开销。
- Block 布局。
- 是否需要批量分配。
- 是否需要为 key/value 引入其他管理策略。

## 40. 当前项目进度

目前已经完成：

```
固定大小内存池
├── Chunk 扩容
├── Block 对齐
├── Free List
├── Chunk List
├── Block 回收复用
├── 统一销毁
└── ASan 单元测试

Hash 接入
├── hashtable 内嵌 node_pool
├── 创建时初始化内存池
├── 节点申请策略封装
├── 节点释放策略封装
├── key/value 与节点分开释放
├── 删除节点前断开链表
├── Hash 销毁时销毁内存池
└── ASan 集成测试

对比能力
├── 编译时切换 malloc/内存池
├── Makefile 传递切换宏
├── 关闭客户端逐请求日志
├── 关闭服务器调试日志
├── 使用独立目录隔离快照和 AOF
└── 完成第一轮定向 QPS 对比
```

当前得到的阶段性结论：

> 自研固定大小内存池已经正确接入 Hash 节点，并能够复用释放的节点 Block；但在当前端到端测试中，它与普通 `malloc/free` 的 QPS 基本持平，尚未表现出明显优势。

尚未完成：

```
1. 重复多轮 QPS 测试并统计平均值。
2. 使用统一的 -O2 优化参数重新测试。
3. 编写不经过网络和 AOF的进程内分配性能测试。
4. 构造大量节点同时存活的负载。
5. 测量 VIRT 和 RSS。
6. 接入 jemalloc 形成第三组对比。
7. 根据结果决定是否调整 Chunk 大小。
8. 决定是否把内存池接入红黑树和跳表节点。
```

## 41. 进程内 Hash 分配器压测

之前的端到端压测需要经过：

```
客户端 → TCP → 协议解析 → Hash → 内存分配 → AOF → TCP 返回
```

这条链路包含很多与内存池无关的开销，所以即使节点分配变快，最终的服务器 QPS 也可能没有明显变化。

为了单独观察分配器，我们新增了：

```
tests/benchmark_hash_allocator.c
```

它直接在进程内反复执行：

```
HSET 对应的 kvs_hash_set()
        ↓
HDEL 对应的 kvs_hash_del()
```

不经过网络、不解析协议，也不写 AOF。

测试始终使用同一个 key，意味着任意时刻最多只有一个 Hash 节点存活。这样做不是为了模拟大量数据，而是为了集中制造节点的反复申请与释放，尽量隔离分配器本身的开销。

本次采用：

```
rounds:     100000000
operations: 200000000
```

每轮包含一次 SET 和一次 DEL，所以总操作数等于 `rounds * 2`。

测试使用 `clock_gettime(CLOCK_MONOTONIC, ...)` 计时。`CLOCK_MONOTONIC` 是单调递增时钟，不会因为系统时间被校准而突然向前或向后跳动，适合测量耗时。

## 42. 进程内吞吐量测试结果

内存池版本：

```
allocator: memory_pool
rounds: 100000000
operations: 200000000
elapsed: 3.180859 seconds
ops/sec: 62876099
```

malloc 版本：

```
allocator: malloc
rounds: 100000000
operations: 200000000
elapsed: 3.577090 seconds
ops/sec: 55911367
```

按照这一次实验的数据计算：

```
吞吐量提升 ≈ (62876099 - 55911367) / 55911367
          ≈ 12.5%
```

阶段性结论：

> 在反复申请和释放固定大小 `hashnode_t` 的进程内场景中，自研内存池比普通 malloc 具有明显的吞吐量优势。

这个结果与端到端压测并不矛盾：

- 进程内压测说明内存池这个局部组件确实更快。
- 端到端压测基本持平，说明节点分配不是当前服务器整条请求链路的主要耗时。

因此不能简单地说“内存池没有用”，也不能说“服务器已经快了 12.5%”。准确说法是：

> 内存池优化了 Hash 节点分配，但这部分优势目前被网络、协议和持久化等开销稀释了。

## 43. 修复字符串复制与修改失败的问题

在使用 `-Werror` 编译基准测试时，编译器指出原代码存在以下写法：

```c
strncpy(kcopy, key, strlen(key));
```

虽然前面使用 `memset` 清零可以让字符串在当前情况下以 `\0` 结尾，但这种写法把“正确性”分散在两条语句中，也会触发字符串截断警告。

最终改为：

```c
memcpy(kcopy, key, strlen(key) + 1);
```

`+ 1` 会把字符串末尾的 `\0` 一起复制过去。

在修改 value 时，还调整了操作顺序：

```
先申请新 value
→ 确认申请成功
→ 复制新内容
→ 再释放旧 value
→ 替换指针
```

如果先释放旧 value，再申请新 value，一旦申请失败，节点就会留下一个已经失效的悬空指针。这里再次使用了“可延续清除”的思想：先准备好后继状态，再破坏旧状态。

## 44. 为什么内存占用测试要同时创建大量节点

分配速度测试只保留一个节点，是为了反复刺激 `alloc/free`。

内存占用测试的目标不同。它需要让大量节点同时存活，才能观察：

- 每个节点的分配开销。
- Chunk 的总体占用。
- malloc 的元数据和对齐开销。
- 删除和销毁前后的内存变化。

因此新增：

```
tests/benchmark_hash_memory.c
```

测试插入 20 万个不同的 key：

```c
for (int i = 0; i < entries; i++) {
    snprintf(key, sizeof(key), "key-%d", i);
    snprintf(value, sizeof(value), "value-%d", i);
    kvs_hash_set(&hash, key, value);
}
```

`snprintf` 的作用是按照格式生成字符串，同时限制最多写入多少字节，避免写出数组边界。例如当 `i == 12` 时：

```
key   = "key-12"
value = "value-12"
```

虽然循环一直复用栈上的 `key` 和 `value` 数组，但 `kvs_hash_set` 会复制字符串内容，因此已经插入 Hash 的节点不会依赖这两个临时数组。

## 45. 用阶段暂停观察内存

测试中定义了：

```c
static void wait_for_measurement(const char *phase) {
    long pid = (long)getpid();

    printf("Phase: %s\n", phase);
    printf("PID: %ld\n", pid);
    printf("press Enter to continue...\n");

    fflush(stdout);
    getchar();
}
```

其中：

- `phase` 表示当前测量阶段。
- `getpid()` 取得当前测试进程的 PID。
- `fflush(stdout)` 立即刷新标准输出缓冲区，确保提示信息已经显示到终端。
- `getchar()` 等待一个字符，为我们人为制造暂停点；按 Enter 后程序继续。

这里不使用 `fsync`，因为两者解决的问题不同：

- `fflush` 刷新 C 标准库的用户态输出缓冲区。
- `fsync` 要求把文件描述符对应的数据同步到底层存储设备，主要用于文件持久化。

终端提示只需要 `fflush(stdout)`。

测试设置了三个暂停阶段：

```
after insert  → 20 万个节点仍然存活
after delete  → 节点全部删除
after destroy → Hash 和内存池已经销毁
```

## 46. 用 ps 查看 VSZ 和 RSS

测量命令为：

```bash
ps -o pid,vsz,rss,comm -p PID
```

各字段含义：

- `ps`：查看进程状态的快照。
- `-o`：指定需要显示的字段。
- `pid`：进程编号。
- `vsz`：Virtual Size，进程虚拟地址空间大小。
- `rss`：Resident Set Size，当前驻留在物理内存中的部分。
- `comm`：程序的命令名。
- `-p PID`：只查看指定 PID。

Linux 中 `VSZ` 和 `RSS` 默认以 KiB 为单位。

需要特别记住：

```
VSZ / VIRT：虚拟内存，不等于实际占用的物理内存
RSS / RES：当前驻留在物理内存中的内存
```

## 47. 内存池与 malloc 的内存测量结果

测试条件：20 万个不同 Hash 节点同时存活。

| 分配方式 | 阶段 | VSZ | RSS |
|---|---|---:|---:|
| memory pool | after insert | 19416 KiB | 18688 KiB |
| memory pool | after delete | 19416 KiB | 18688 KiB |
| memory pool | after destroy | 19416 KiB | 18688 KiB |
| malloc | after insert | 21000 KiB | 20232 KiB |
| malloc | after delete | 21000 KiB | 20232 KiB |
| malloc | after destroy | 21000 KiB | 20232 KiB |

插入完成时，malloc 相比内存池多出：

```
VSZ: 21000 - 19416 = 1584 KiB
RSS: 20232 - 18688 = 1544 KiB
```

RSS 大约多出 `1.51 MiB`。平均到 20 万个节点：

```
1544 * 1024 / 200000 ≈ 7.9 字节/节点
```

这与 malloc 的单次分配元数据、大小分类及地址对齐开销相符。内存池提前知道所有 Block 大小相同，因此不需要为每个正在使用的节点单独记录完整的大小信息；空闲 Block 还可以直接把自身空间当作 `free_node_t` 来保存 `next`。

但这只是当前机器、当前结构大小和当前负载下的实验结果，不能直接推广到所有程序。

## 48. 为什么 delete 和 destroy 后 RSS 没有下降

内存池模式执行删除时：

```
Hash 节点从桶链表断开
→ Block 放回 free_list
→ Chunk 仍然存在
```

因此 `after delete` 内存不下降完全符合设计。保留的 Block 可以在后续插入时立即复用。

内存池执行 destroy 时会调用：

```c
free(chunk->memory);
free(chunk);
```

但是 `free` 通常只表示把空间归还给当前进程内部的分配器。分配器可能为了后续申请而保留这些内存，不一定立即归还操作系统，所以 `ps` 看到的 VSZ/RSS 可能暂时不变。

malloc 模式也出现了同样现象：每个节点已经执行 `free`，但 glibc 仍可能保留释放后的堆内存。

因此：

> RSS 没有立即下降，不能单独作为内存泄漏的证据。

内存是否被错误遗失，应该结合以下手段判断：

- ASan/LeakSanitizer 是否报告泄漏。
- 分配与释放的所有权是否匹配。
- 内存池的 Chunk 是否都能从 `chunk_list` 找到并销毁。
- 进程长期运行时占用是否无上限增长。

当测试进程最终退出后，再次执行 `ps -p PID` 只剩表头，没有进程记录。这说明操作系统已经回收该进程的全部资源。

## 49. glibc 是不是“内存缓存页”

不是。

`glibc` 是 GNU C Library，即 GNU C 标准库。它向 C 程序提供大量常用能力，例如：

```
printf / fopen / memcpy
malloc / free
线程和系统调用封装
```

其中 `malloc/free` 只是 glibc 的一个组成部分。glibc 内部的内存分配器会向操作系统申请较大的内存区域，再把它们切分给程序；程序调用 `free` 后，分配器也可能暂时缓存这些空闲区域，以便后续复用。

“内存页”则是操作系统进行虚拟内存映射和物理内存管理的基本单位，在常见系统中一页通常是 4 KiB，但实际页大小需要以具体系统为准。

可以把层级暂时理解为：

```
我们的 Hash
    ↓ 申请 hashnode_t
自研内存池 或 glibc malloc
    ↓ 需要更大的内存区域
Linux 虚拟内存系统
    ↓ 按页建立映射
物理内存
```

所以准确说法是：

> glibc 不是内存缓存页；glibc 的 malloc 分配器会管理和复用进程的堆内存，而操作系统在更底层以页为单位管理虚拟内存与物理内存。

## 50. 当前阶段结论与下一步

目前已经完成原计划中的两项核心对比：

```
进程内吞吐量：内存池约 6288 万 ops/s
                malloc 约 5591 万 ops/s

20 万节点 RSS：内存池约 18688 KiB
                malloc 约 20232 KiB
```

当前结论：

1. 固定大小内存池适合管理 `hashnode_t`。
2. 在节点高频申请/释放的局部测试中，内存池速度更快。
3. 在大量节点同时存活时，内存池的 RSS 和 VSZ 更低。
4. 在完整服务器链路中，内存池带来的局部优势暂时被其他开销掩盖。
5. `free` 完成与 RSS 立即下降不是同一个概念。

接下来的计划：

```
1. 把 Hash 内存占用测试加入 Makefile。
2. 适当重复内存实验，确认数据稳定性。
3. 接入 jemalloc，形成 malloc / memory pool / jemalloc 三组对比。
4. 比较三者的进程内吞吐量、VSZ 和 RSS。
5. 根据结果评估 Chunk 大小是否需要调整。
6. 再决定是否把内存池接入红黑树和跳表。
```

## 51. 先整理当前测试体系

随着测试文件增加，一度很难分清每个文件究竟在测什么。现在把它们按“正确性、速度、内存、完整链路”重新整理：

| 文件 | 类型 | 回答的问题 |
|---|---|---|
| `tests/test_memory_pool.c` | 内存池单元测试 | 内存池自身能否申请、归还、复用、扩容和销毁？ |
| `tests/test_hash_memory_pool.c` | Hash 集成测试 | Hash 接入内存池后，CRUD 和节点复用是否正常？ |
| `tests/benchmark_hash_allocator.c` | 进程内速度测试 | 不经过网络时，不同分配器处理 Hash SET/DEL 的速度如何？ |
| `tests/benchmark_hash_memory.c` | 进程内内存测试 | 20 万节点时，不同分配器的 VSZ/RSS 如何变化？ |
| `tests/testcase.c` | 端到端测试 | 经过 TCP、协议、Hash 和 AOF 的完整服务器表现如何？ |

最简记忆：

```
test      → 判断对不对
benchmark → 测量快不快、占多少
```

测试顺序也形成了清晰层次：

```
先证明内存池自身正确
→ 再证明 Hash 接入正确
→ 再测局部分配速度
→ 再测进程内存
→ 最后观察完整服务器
```

## 52. 简化 Makefile，重新取得控制权

原先为了同时保存 pool/malloc 两套 benchmark 二进制，Makefile 中复制了多份几乎相同的规则。虽然能够工作，但重复内容较多，已经不容易掌控。

现在改为通过一个变量控制分配策略：

```make
HASH_USE_MEMORY_POOL ?= 1
```

两个 benchmark 目标分别只保留一份规则：

```text
benchmark-hash        → 编译速度测试
benchmark-hash-memory → 编译内存测试
```

使用方法：

```bash
# 编译自研内存池版本
make HASH_USE_MEMORY_POOL=1 benchmark-hash

# 编译 glibc malloc 版本
make HASH_USE_MEMORY_POOL=0 benchmark-hash
```

内存测试同理：

```bash
make HASH_USE_MEMORY_POOL=1 benchmark-hash-memory
make HASH_USE_MEMORY_POOL=0 benchmark-hash-memory
```

Makefile 现在只负责编译，不再自动运行 benchmark。这样“编译哪一种分配器”和“什么时候运行”都由开发者明确控制。

检查 Makefile 时使用了：

```bash
make -n HASH_USE_MEMORY_POOL=1 benchmark-hash-memory
```

`make -n` 只打印 Make 准备执行的命令，不真正执行，也叫 dry run。它适合检查最终宏值、编译参数以及是否存在意外运行步骤。

## 53. malloc 是接口，glibc/jemalloc 是实现

之前容易把 `malloc` 理解为一个固定的内存分配器。更准确的理解是：

```c
void *malloc(size_t size);
void free(void *ptr);
```

`malloc/free` 是程序调用的统一接口，具体如何管理内存由背后的实现决定。

Ubuntu 默认情况：

```
业务代码
→ kvs_malloc
→ malloc
→ glibc 提供的分配器实现
```

链接 jemalloc 后：

```
业务代码
→ kvs_malloc
→ malloc
→ jemalloc 提供的分配器实现
```

jemalloc 提供了与标准 `malloc/free` 兼容的符号，所以业务代码不需要把 `malloc` 改成其他函数名。链接时增加：

```bash
-ljemalloc
```

就可以让程序链接 `libjemalloc.so`。

最简记忆：

> 代码决定“调用 malloc”，链接决定“谁实现 malloc”。

只有需要使用 jemalloc 独有的 `mallctl`、`mallocx` 等接口时，才需要显式包含：

```c
#include <jemalloc/jemalloc.h>
```

本轮只是公平替换标准分配器，没有使用专用接口。

## 54. 安装并验证 jemalloc

Ubuntu 24.04 ARM64 软件源提供：

```text
libjemalloc-dev 5.3.0-2build1
```

安装命令：

```bash
sudo apt install libjemalloc-dev
```

简单区分：

```
libjemalloc2    → 让已链接 jemalloc 的程序运行
libjemalloc-dev → 让自己的程序能够编译并链接 jemalloc
```

安装 `-dev` 包时，apt 自动安装了运行时包 `libjemalloc2`。

使用以下命令检查系统动态库：

```bash
ldconfig -p | grep jemalloc
```

结果：

```text
libjemalloc.so.2 => /lib/aarch64-linux-gnu/libjemalloc.so.2
libjemalloc.so   => /lib/aarch64-linux-gnu/libjemalloc.so
```

编译 jemalloc 版 benchmark 时同时使用：

```bash
-DKVS_HASH_USE_MEMORY_POOL=0
-ljemalloc
```

第一项关闭自研节点池，第二项真正链接 jemalloc。

使用 `ldd` 验证：

```bash
ldd /tmp/benchmark_hash_jemalloc | grep jemalloc
```

结果出现 `libjemalloc.so.2`，证明程序运行时确实加载了 jemalloc。

即使 jemalloc 版同时显示 `libc.so.6` 也不矛盾，因为 `printf`、字符串处理等其他 C 标准库能力仍然来自 glibc；被替换的是内存分配接口的实现。

## 55. 自研内存池能否和 jemalloc 组合

可以，因为二者工作在不同层级：

```
Hash 节点
→ 自研内存池管理固定大小 Block
→ 内存池需要扩容 Chunk
→ Chunk 通过底层 malloc 申请
→ malloc 可以由 jemalloc 实现
```

此外，当前 key/value 没有进入固定大小节点池：

```
hashnode_t → memory_pool_alloc
key/value  → kvs_malloc → malloc
```

因此链接 jemalloc 后可以形成：

```
hashnode_t → 自研内存池
             └── Chunk 由 jemalloc 提供

key/value  → jemalloc
```

理论上可以形成四种组合：

| 模式 | Hash 节点 | key/value |
|---|---|---|
| glibc | glibc | glibc |
| jemalloc | jemalloc | jemalloc |
| memory pool + glibc | 自研内存池 | glibc |
| memory pool + jemalloc | 自研内存池 | jemalloc |

本轮先比较前三种主要模式，混合模式留到后续作为扩展实验。

## 56. Arena 的简单理解

Arena 可以理解为分配器内部的一座独立仓库：

```
Block = 仓库中的一个货位
Chunk = 一批一起申请的货架
Arena = 管理多批货架和货位的仓库
```

Arena 内部会管理空闲内存、大小类别和必要的同步结构。多个线程使用不同 Arena，可以减少同时申请内存时的锁竞争：

```
线程 A → Arena A
线程 B → Arena B
线程 C → Arena C
```

但 Arena 不是越多越好。各 Arena 独立管理内存，会增加固定管理开销，也可能导致某个 Arena 有空闲空间、另一个 Arena 却不能直接利用，从而增加碎片或虚拟内存。

一句话：

> 多个 Arena 用更多独立仓库换取更少排队，代价是更多管理开销和潜在空闲空间。

## 57. 修正 benchmark 的分配器标签

最初 jemalloc 版仍打印：

```text
allocator: malloc
```

原因是原程序只区分“自研内存池开启/关闭”，并不知道标准 `malloc` 背后来自 glibc 还是 jemalloc。

标签判断改为：

```c
#if KVS_HASH_USE_MEMORY_POOL
    const char *allocator = "memory_pool";
#elif defined(KVS_BENCHMARK_JEMALLOC)
    const char *allocator = "jemalloc";
#else
    const char *allocator = "glibc_malloc";
#endif
```

编译 jemalloc 版本时增加：

```bash
-DKVS_BENCHMARK_JEMALLOC=1
```

需要明确：

```
KVS_BENCHMARK_JEMALLOC → 只负责显示正确标签
-ljemalloc             → 真正改变 malloc/free 的实现
```

## 58. 三方进程内吞吐量对比

为了减少单次运行波动，在同一时段依次运行 glibc、自研内存池和 jemalloc，共进行三轮。每个程序执行：

```text
100000000 轮 SET + DEL
200000000 次 Hash 操作
```

原始结果：

| 轮次 | 分配器 | elapsed | ops/sec |
|---:|---|---:|---:|
| 1 | glibc malloc | 3.863011 秒 | 51,773,090 |
| 1 | memory pool | 3.413726 秒 | 58,587,006 |
| 1 | jemalloc | 3.196506 秒 | 62,568,315 |
| 2 | glibc malloc | 3.847538 秒 | 51,981,298 |
| 2 | memory pool | 3.466388 秒 | 57,696,942 |
| 2 | jemalloc | 3.253883 秒 | 61,465,034 |
| 3 | glibc malloc | 3.812338 秒 | 52,461,241 |
| 3 | memory pool | 3.426403 秒 | 58,370,256 |
| 3 | jemalloc | 3.224374 秒 | 62,027,550 |

三轮平均值：

| 分配器 | 平均 elapsed | 平均 ops/sec |
|---|---:|---:|
| glibc malloc | 3.840962 秒 | 52,071,876 |
| memory pool | 3.435506 秒 | 58,218,068 |
| jemalloc | 3.224921 秒 | 62,020,300 |

按平均吞吐量计算：

```text
memory pool 比 glibc malloc 高约 11.8%
jemalloc 比 glibc malloc 高约 19.1%
jemalloc 比 memory pool 高约 6.5%
```

三轮排名一致，并且每种分配器自身波动不大，因此比此前单轮结果更有参考意义。

但实验仍有局限：三轮始终按照 glibc → pool → jemalloc 的固定顺序运行，后续还可以轮换或随机化顺序，减少 CPU 温度、调度等顺序因素。

## 59. 为什么这组数据不能证明 jemalloc 的节点分配一定比内存池快

每次 `kvs_hash_set` 实际可能包含三次分配：

```
1 个 hashnode_t
1 段 key
1 段 value
```

三种模式的实际路径为：

```
glibc：
node、key、value 全部由 glibc 管理

memory pool：
node 由自研内存池管理
key、value 仍由 glibc 管理

jemalloc：
node、key、value 全部由 jemalloc 管理
```

因此当前 benchmark 测到的是完整 Hash SET/DEL 分配路径，不是只测 `hashnode_t` 的裸分配器性能。

当前可以准确地说：

> 在这次完整 Hash SET/DEL 分配路径中，jemalloc 最快，自研内存池其次，glibc malloc 最慢。

但不能直接说：

> jemalloc 的单次固定大小节点分配一定比自研内存池快。

如果将来要回答后一个问题，需要编写更纯粹的分配器微基准，只申请和释放 `hashnode_t` 大小的内存，不进行 key/value 分配和 Hash 查找。

## 60. jemalloc 的 VSZ/RSS 测量

jemalloc 内存测试仍使用 20 万个不同节点和三个暂停点：

```text
after insert
after delete
after destroy
```

jemalloc 结果：

| 阶段 | VSZ | RSS |
|---|---:|---:|
| after insert | 24120 KiB | 16572 KiB |
| after delete | 24120 KiB | 12476 KiB |
| after destroy | 24120 KiB | 12476 KiB |

删除节点后：

```text
VSZ 变化：0 KiB
RSS 变化：-4096 KiB
```

这说明虚拟地址空间仍然被保留，但约 4 MiB 页面在本次测量时已经不再驻留于物理内存。

## 61. 三方 VSZ/RSS 对比

完整对比数据：

| 分配器 | 阶段 | VSZ | RSS |
|---|---|---:|---:|
| memory pool | after insert | 19416 KiB | 18688 KiB |
| memory pool | after delete | 19416 KiB | 18688 KiB |
| memory pool | after destroy | 19416 KiB | 18688 KiB |
| glibc malloc | after insert | 21000 KiB | 20232 KiB |
| glibc malloc | after delete | 21000 KiB | 20232 KiB |
| glibc malloc | after destroy | 21000 KiB | 20232 KiB |
| jemalloc | after insert | 24120 KiB | 16572 KiB |
| jemalloc | after delete | 24120 KiB | 12476 KiB |
| jemalloc | after destroy | 24120 KiB | 12476 KiB |

只看插入 20 万节点后的状态：

| 分配器 | VSZ 排名 | RSS 排名 |
|---|---:|---:|
| memory pool | 最低 | 第二 |
| glibc malloc | 第二 | 最高 |
| jemalloc | 最高 | 最低 |

jemalloc 相比自研内存池：

```text
VSZ 多 4704 KiB
RSS 少 2116 KiB，约少 11.3%
```

jemalloc 相比 glibc malloc：

```text
VSZ 多 3120 KiB
RSS 少 3660 KiB，约少 18.1%
```

这说明“内存占用”不能只用一个数字描述：

```
看虚拟地址空间 VSZ → 自研内存池最低
看实际驻留物理内存 RSS → jemalloc 最低
```

jemalloc 可以保留较大的虚拟地址范围，同时让较少页面实际驻留。删除后 RSS 下降、VSZ 不变，也体现了“保留地址空间”和“物理页仍然驻留”是两回事。

需要注意，`ps` 是某个时刻的进程快照，结果会受分配器配置、系统负载和页面回收策略影响。当前数据是实验事实，但仍应通过多轮运行确认稳定性。

## 62. 当前阶段结论

目前已经完成：

```
glibc malloc
vs 自研 memory pool
vs jemalloc

├── 进程内 Hash 吞吐量对比
└── 20 万节点 VSZ/RSS 对比
```

在当前 Ubuntu ARM64 虚拟机和当前 Hash 负载下：

1. jemalloc 的平均吞吐量最高。
2. 自研节点池的平均吞吐量高于 glibc malloc。
3. 自研节点池的 VSZ 最低。
4. jemalloc 的 RSS 最低，并且删除节点后 RSS 下降约 4 MiB。
5. glibc 在本轮吞吐量和 RSS 数据中均处于第三位。
6. 这不代表 jemalloc 在所有场景全方位超过 glibc。
7. 自研内存池只管理固定大小节点，jemalloc 则是能够处理不同尺寸的通用分配器，二者职责不同且可以组合。

下一步计划：

```
1. 用一个清楚的 Makefile 变量加入 jemalloc 构建模式，避免复制规则。
2. 重复 jemalloc 内存测试，确认 VSZ/RSS 的稳定性。
3. 轮换三方 benchmark 的运行顺序。
4. 根据目标选择是否编写纯 hashnode_t 分配微基准。
5. 可选：测试 memory pool + jemalloc 混合模式。
6. 最后再决定是否将节点池接入红黑树和跳表。
```

## 63. 红黑树节点接入内存池

Hash 阶段完成后，将同一个固定大小内存池继续接入红黑树。

红黑树节点结构中包含颜色、左右孩子、父节点以及 key/value 指针。虽然 key 和 value 指向的字符串长度不同，但指针本身大小固定，因此：

```text
rbtree_node 节点外壳 → 大小固定，可以使用节点池
key/value 字符串      → 长度不固定，继续使用 kvs_malloc/kvs_free
```

在红黑树管理结构中直接嵌入：

```c
memory_pool_t node_pool;
```

这里没有使用 `memory_pool_t *`。这表示内存池管理结构本身就是红黑树实例的一部分，不需要再单独为管理器执行一次 malloc。

### nil 哨兵节点为什么不进入节点池

红黑树使用一个 `nil` 哨兵代表不存在的孩子。它是一个真实节点，并且从树创建一直存活到树销毁，不会像业务节点一样频繁创建和删除。

因此采用：

```text
nil 哨兵             → kvs_malloc/kvs_free
普通 rbtree_node     → node_pool
```

这样保留了节点池的懒扩容：创建空树时只初始化 node pool，还没有 Chunk；第一次插入普通节点时才申请第一个 Chunk。

nil 初始化为黑色，并使它的 left、right、parent 都指向自身：

```text
nil->color  = BLACK
nil->left   = nil
nil->right  = nil
nil->parent = nil
```

### 节点释放入口

普通节点不能再直接 `kvs_free(node)`，因为它可能来自节点池。于是增加统一的节点释放函数：

```text
先释放 node->key
再释放 node->value
最后把 node 外壳归还 node_pool
```

删除时必须释放 `rbtree_delete()` 返回的节点。该函数可能真正摘除目标节点，也可能摘除它的后继节点并交换 key/value 指针；返回值才是最终脱离树结构、可以释放的节点。

销毁时循环选择当前树中的最小节点：

```text
rbtree_mini(inst, inst->root)
→ rbtree_delete()
→ 释放 key/value
→ 归还节点外壳
```

全部普通节点处理完成后，再释放 nil，最后销毁 node pool 的所有 Chunk。

## 64. 红黑树重复 key 与 Engine 层防护

协议层已经会检查重复的 `RSET`，但如果测试或其他模块绕过协议层，直接调用 `kvs_rbtree_set()`，Engine 层仍可能收到重复 key。

原始底层插入函数遇到相同 key 会直接返回。如果在发现重复之前已经申请 node、key、value，这三块内存将无法进入树，也无法再被找到，形成泄漏。

因此在申请任何新内存前增加：

```c
if (rbtree_search(inst, key) != inst->nil) {
    return 1;
}
```

最终顺序为：

```text
检查 key 是否重复
→ 不重复才申请 node
→ 再申请 key/value
→ 插入红黑树
```

这形成两层防线：协议层维护命令语义，Engine 层保证自身接口直接使用时也不会泄漏。

## 65. 红黑树分配器开关与测试

为了公平对比，在红黑树中增加编译开关：

```c
KVS_RBTREE_USE_MEMORY_POOL=1  // 普通节点使用自研节点池
KVS_RBTREE_USE_MEMORY_POOL=0  // 普通节点使用 kvs_malloc/kvs_free
```

节点申请和节点外壳释放分别通过统一入口完成。即使关闭内存池，上层创建逻辑也仍调用统一入口，只是入口内部会选择 `kvs_malloc`。

集成测试覆盖：

```text
create 与 nil 哨兵初始化
SET/GET
重复 key 被拒绝且原 value 不变
DEL
节点 Block 地址复用
多节点红黑树销毁
pool/malloc 两种编译模式
ASan 内存安全检查
```

pool 模式的地址结果：

```text
first node  = 0x52f00000c3d0
second node = 0x52f00000c3d0 (reused)
rbtree memory pool integration test passed
```

malloc 模式中不要求地址相同，测试只验证 CRUD、重复 key 和销毁路径。

## 66. 红黑树进程内吞吐量对比

红黑树 benchmark 不经过网络和 AOF。每个进程执行：

```text
100000000 轮 SET + DEL
200000000 次红黑树操作
```

三轮原始结果：

| 轮次 | 分配器 | elapsed | ops/sec |
|---:|---|---:|---:|
| 1 | glibc malloc | 3.131742 秒 | 63,862,213 |
| 1 | memory pool | 2.678489 秒 | 74,668,968 |
| 1 | jemalloc | 2.507363 秒 | 79,765,072 |
| 2 | glibc malloc | 3.178343 秒 | 62,925,868 |
| 2 | memory pool | 2.684461 秒 | 74,502,851 |
| 2 | jemalloc | 2.579492 秒 | 77,534,646 |
| 3 | glibc malloc | 3.170954 秒 | 63,072,494 |
| 3 | memory pool | 2.672549 秒 | 74,834,912 |
| 3 | jemalloc | 2.472445 秒 | 80,891,603 |

三轮平均：

| 分配器 | 平均 elapsed | 平均 ops/sec |
|---|---:|---:|
| glibc malloc | 3.160346 秒 | 63,286,858 |
| memory pool | 2.678500 秒 | 74,668,910 |
| jemalloc | 2.519767 秒 | 79,397,107 |

按平均吞吐量计算：

```text
memory pool 比 glibc malloc 快约 18.0%
jemalloc 比 glibc malloc 快约 25.5%
jemalloc 比 memory pool 快约 6.3%
```

每轮树中最多只有一个普通节点，所以查找、旋转和平衡成本很低，分配器开销在结果中占比较高。

同样需要注意：自研池只管理 `rbtree_node` 外壳；key/value 仍由 glibc 管理。jemalloc 模式则同时接管 node、key 和 value。因此这是完整 SET/DEL 路径的比较，不是裸节点分配速度比较。

## 67. 红黑树 VSZ/RSS 对比

内存测试插入 20 万个不同节点，并在插入、删除、销毁三个阶段暂停测量。

| 分配器 | 阶段 | VSZ | RSS |
|---|---|---:|---:|
| memory pool | after insert | 24104 KiB | 23388 KiB |
| memory pool | after delete | 24104 KiB | 23388 KiB |
| memory pool | after destroy | 24104 KiB | 23388 KiB |
| glibc malloc | after insert | 27208 KiB | 26472 KiB |
| glibc malloc | after delete | 27208 KiB | 26472 KiB |
| glibc malloc | after destroy | 27208 KiB | 26472 KiB |
| jemalloc | after insert | 29244 KiB | 19636 KiB |
| jemalloc | after delete | 29244 KiB | 19636 KiB |
| jemalloc | after destroy | 29244 KiB | 19636 KiB |

插入 20 万节点时：

```text
自研池比 glibc 少约 3104 KiB VSZ
自研池比 glibc 少约 3084 KiB RSS
jemalloc 的 VSZ 最高，但 RSS 最低
```

这次 jemalloc 删除后 RSS 没有下降。与 Hash 实验不同，说明释放后是否立刻回收物理页不是固定规律，会受对象尺寸、分配布局、回收策略和测量时机影响。

## 68. 跳表节点为什么只能部分接入固定大小内存池

跳表节点为：

```c
typedef struct kvs_skiplist_node {
    char *key;
    char *value;
    struct kvs_skiplist_node **forward;
} kvs_skiplist_node_t;
```

在当前 64 位环境中，三个字段都是 8 字节指针，所以节点外壳通常约为：

```text
8 + 8 + 8 = 24 字节
```

`forward` 字段本身只是固定大小的地址，但它指向的数组长度由节点 level 决定：

```text
level 0 → 1 个 forward 指针
level 3 → 4 个 forward 指针
level 6 → 7 个 forward 指针
```

因此最终所有权划分为：

```text
kvs_skiplist_node_t 外壳 → 固定大小 node_pool
key 字符串                → kvs_malloc
value 字符串              → kvs_malloc
forward 指针数组          → kvs_malloc
```

可以把 `forward` 字段理解为一张写着仓库地址的纸条：纸条大小固定，但纸条指向的仓库大小可以变化。

## 69. 跳表 header 与懒扩容

跳表的 header 与红黑树 nil 类似，创建后长期存在，不会随业务命令频繁创建和删除。因此采用：

```text
header 节点外壳 → kvs_malloc
普通业务节点    → node_pool
```

这使 node pool 保持懒扩容：

```text
kvs_skiplist_create()
→ 初始化 node_pool
→ header 使用 kvs_malloc
→ node_pool 还没有 Chunk

第一次 SSET
→ 普通节点调用 memory_pool_alloc
→ 创建第一个 Chunk
```

`skiplist_create_node()` 同时用于 header 和普通节点，因此额外接收 `bool use_node_pool`：

```text
false → 创建 header，走 kvs_malloc
true  → 创建普通节点，根据编译宏选择 pool/malloc
```

统一的 `skiplist_node_storage_alloc/free` 不是强制使用内存池，而是分配方式选择器。

## 70. 跳表失败路径与状态提交顺序

创建一个普通跳表节点依次申请：

```text
node 外壳
→ key
→ value
→ forward 数组
```

失败时按照已经成功取得的资源逐项回收：

```text
key 失败     → 归还 node 外壳
value 失败   → 释放 key，再归还 node 外壳
forward 失败 → 释放 key/value，再归还 node 外壳
```

key/value/forward 均由 `kvs_malloc` 取得，因此使用 `kvs_free`；节点外壳来源可变，因此通过 `skiplist_node_storage_free` 判断应归还 node pool 还是 malloc。

forward 数组申请成功后，将每个位置初始化为 NULL，避免节点暂时携带未初始化的垃圾地址。

另一个失败路径发生在随机 level 高于当前跳表 level 时。不能在节点申请成功前就提高 `skipList->level`，否则申请失败后会留下“层数已经提高、节点却没有插入”的不一致状态。

正确顺序为：

```text
生成随机 level
→ 准备 update 数组
→ 创建节点
→ 节点创建成功后更新 skipList->level
→ 接入各层 forward
```

原则是：只有资源申请成功以后，才能提交对外可见的数据结构状态。

## 71. 跳表正确性与多 Chunk 测试

跳表使用编译开关：

```c
KVS_SKIPLIST_USE_MEMORY_POOL=1
KVS_SKIPLIST_USE_MEMORY_POOL=0
```

集成测试覆盖：

```text
header 创建与 node pool 懒扩容
普通节点 SET/GET/DEL
节点外壳归还 Free List
删除后节点地址复用
不同 level 的 forward 数组
2001 个同时存活的普通节点
1024 Blocks/Chunk 下触发第二个 Chunk
多个 Chunk 的销毁
pool/malloc 两种模式
ASan 内存安全检查
```

pool 模式结果：

```text
first node  = 0x52b0000061e8
second node = 0x52b0000061e8 (reused)
skiplist create/destroy test passed
```

测试中曾经把“第二个 Chunk 已存在”的断言放在 2000 次插入循环内部，并在第一次插入前立即执行，导致断言失败。移动到整个循环完成之后才正确验证扩容。

这个问题说明：断言不仅要写对条件，还必须放在被验证行为已经完成之后。

## 72. 跳表进程内吞吐量对比

每个程序执行：

```text
100000000 轮 SET + DEL
200000000 次跳表操作
```

三轮原始结果：

| 轮次 | 分配器 | elapsed | ops/sec |
|---:|---|---:|---:|
| 1 | glibc malloc | 5.615434 秒 | 35,616,122 |
| 1 | memory pool | 5.522943 秒 | 36,212,577 |
| 1 | jemalloc | 4.974261 秒 | 40,206,980 |
| 2 | glibc malloc | 5.878963 秒 | 34,019,604 |
| 2 | memory pool | 5.468806 秒 | 36,571,056 |
| 2 | jemalloc | 5.041026 秒 | 39,674,463 |
| 3 | glibc malloc | 5.775833 秒 | 34,627,039 |
| 3 | memory pool | 5.410201 秒 | 36,967,203 |
| 3 | jemalloc | 4.903869 秒 | 40,784,125 |

三轮平均：

| 分配器 | 平均 elapsed | 平均 ops/sec |
|---|---:|---:|
| glibc malloc | 5.756743 秒 | 34,754,255 |
| memory pool | 5.467317 秒 | 36,583,612 |
| jemalloc | 4.973052 秒 | 40,221,856 |

按平均吞吐量计算：

```text
memory pool 比 glibc malloc 快约 5.3%
jemalloc 比 glibc malloc 快约 15.7%
jemalloc 比 memory pool 快约 9.9%
```

跳表每次 SET 可能包含四类动态分配：node、key、value 和 forward。自研池只优化 node 外壳，jemalloc 则接管全部四类，因此自研池相对 glibc 的提升小于红黑树场景。

跳表 benchmark 没有调用 `srand()`。三个独立进程通常从相同的默认随机序列开始，因此三种分配器面对的 level 序列基本一致，有利于当前对比。

## 73. 跳表 VSZ/RSS 对比

20 万存活节点的完整数据：

| 分配器 | 阶段 | VSZ | RSS |
|---|---|---:|---:|
| memory pool | after insert | 26196 KiB | 25420 KiB |
| memory pool | after delete | 26196 KiB | 25420 KiB |
| memory pool | after destroy | 26196 KiB | 25420 KiB |
| glibc malloc | after insert | 27736 KiB | 26944 KiB |
| glibc malloc | after delete | 27736 KiB | 26944 KiB |
| glibc malloc | after destroy | 27736 KiB | 26944 KiB |
| jemalloc | after insert | 29244 KiB | 20000 KiB |
| jemalloc | after delete | 29244 KiB | 18784 KiB |
| jemalloc | after destroy | 29244 KiB | 18784 KiB |

插入后，自研池相对 glibc：

```text
VSZ 少 1540 KiB，约 5.6%
RSS 少 1524 KiB，约 5.7%
```

jemalloc：

```text
VSZ 最高
RSS 最低
删除后 RSS 再下降 1216 KiB
```

跳表中 forward 数组长度可变，并且仍由通用分配器管理。因此节点池降低了固定 node 外壳的开销，但无法消除 key/value/forward 的分配器元数据和碎片。

## 74. 当前阶段总进度与下一步

目前已完成：

```text
固定大小内存池实现
├── Free List 复用
├── Chunk 自动扩容
├── Chunk List 统一销毁
├── 对齐与溢出检查
└── ASan 单元测试

Hash 普通节点
├── 内存池接入
├── pool/malloc 开关
├── 吞吐量 benchmark
└── VSZ/RSS benchmark

红黑树普通节点
├── nil 哨兵独立管理
├── 内存池接入
├── 重复 key 防泄漏
├── pool/malloc 开关
├── 吞吐量 benchmark
└── VSZ/RSS benchmark

跳表普通节点
├── header 独立管理
├── node 外壳进入内存池
├── 可变 forward 数组继续走通用分配器
├── 失败路径和状态提交顺序修复
├── 多 Chunk ASan 测试
├── 吞吐量 benchmark
└── VSZ/RSS benchmark
```

实验反复得到同一个核心认识：

> 固定大小节点池擅长管理生命周期频繁、尺寸相同的节点外壳；它不负责替代所有动态内存分配。jemalloc 是通用分配器，可以与业务专用节点池组合使用。

下一步分析 Array 的真实内存模型。数组通常整体申请连续空间，并通过下标管理元素，可能不存在能够逐个回收到 Free List 的独立节点。因此不能为了“四种结构都接入”而机械套用节点池，需要先确认它的实际申请与释放方式，再决定应优化元素、key/value，还是根本不使用当前固定 Block 内存池。

## 75. 为什么 Array 不接入当前的节点内存池

Array 的核心结构为：

```c
typedef struct kvs_array_item_s {
    char *key;
    char *value;
} kvs_array_item_t;

typedef struct kvs_array_s {
    kvs_array_item_t *table;
    int idx;
    int total;
} kvs_array_t;
```

`kvs_array_create` 会一次性申请整张连续数组：

```c
inst->table = kvs_malloc(
    KVS_ARRAY_SIZE * sizeof(kvs_array_item_t)
);
```

因此 `kvs_array_item_t` 与 Hash、红黑树、跳表的普通节点不同：

```text
Hash/红黑树/跳表：普通节点被逐个申请和释放
Array：所有 item 槽位在 create 时一次性申请
```

Array 删除元素后，会把最后一个有效元素移动到被删除的位置，使有效元素继续紧密排列在：

```text
table[0] ... table[total - 1]
```

下一个空闲槽位自然是 `table[total]`。这已经具有固定容量、连续存储和槽位复用的特点。若再让每个 item 从节点池逐个取得，反而会破坏连续数组的缓存局部性和下标访问方式。

真正单独动态申请的是 `key` 和 `value`。但字符串长度不固定，而当前内存池每个 Block 大小固定，因此也不适合直接管理它们。

结论是：

> Array 保留连续数组设计，不接入当前固定大小节点池；只继续研究 key/value 使用 glibc malloc 或 jemalloc 时的差异。

## 76. Array 中发现并修复的内存问题

虽然 Array 不需要节点池，但检查内存所有权时发现了三类问题。

### 76.1 destroy 必须先释放存活的 key/value

原实现只释放 `table`，会丢失其中仍然存活的 key/value 地址，造成内存泄漏。

正确顺序为：

```text
遍历有效 item
→ 释放每个 key
→ 释放每个 value
→ 释放整个 table
→ 清空 table、total 和 idx
```

### 76.2 SET 的部分失败需要回滚

SET 先申请 key，再申请 value。如果 value 申请失败，已经申请成功的 key 必须释放：

```text
key 成功，value 失败
→ free(key)
→ 返回错误
```

否则一次失败就会泄漏一份 key。

### 76.3 MOD 必须先申请新 value，再释放旧 value

错误写法曾经类似：

```c
kvs_free(inst->table[i].value = NULL);
```

赋值表达式先把成员改为 NULL，再把 NULL 传给 `kvs_free`。旧地址因此丢失，旧 value 没有被释放。

修复后的顺序是：

```text
申请新 value
→ 申请失败时保留旧 value
→ 复制新内容
→ 释放旧 value
→ 保存新地址
```

这是一种“先准备新状态，成功后再提交”的写法。它不仅避免泄漏，也保证 malloc 失败时原数据仍然有效。

字符串复制同时改为：

```c
size_t size = strlen(value) + 1;
char *copy = kvs_malloc(size);
memcpy(copy, value, size);
```

`strlen` 不包含字符串末尾的 `\0`，因此申请和复制都需要 `+1`。这样不再需要 `memset + strncpy(strlen(...))` 的绕行组合，也消除了 `-Wstringop-truncation`。

## 77. Array 的 ASan 生命周期测试

新增 `tests/test_array_memory.c`，覆盖：

```text
create 整体申请 table
SET 两个键值对
拒绝重复 key
GET 验证数据
MOD 替换 value
DEL 释放 key/value 并搬移尾元素
删除不存在的 key
destroy 释放仍然存活的数据和 table
销毁后状态清零
```

测试使用：

```text
-fsanitize=address
-fno-omit-frame-pointer
-Werror
```

结果：

```text
array memory lifecycle test passed
```

说明本次覆盖的 SET、MOD、DEL 和 destroy 路径没有被 ASan 检测到越界、错误释放、重复释放或内存泄漏。

## 78. Array 的 glibc 与 jemalloc 吞吐量对比

Array 没有自研节点池组，因此只比较：

```text
Array + glibc malloc
Array + jemalloc
```

测试反复执行同一个 key 的 SET + DEL：

```text
100000000 轮
每轮 1 次 SET + 1 次 DEL
总计 200000000 次操作
```

三轮原始结果：

| 轮次 | 分配器 | elapsed | ops/sec |
|---:|---|---:|---:|
| 1 | glibc malloc | 2.061267 秒 | 97,027,707 |
| 1 | jemalloc | 1.705103 秒 | 117,295,001 |
| 2 | glibc malloc | 2.094470 秒 | 95,489,552 |
| 2 | jemalloc | 1.724745 秒 | 115,959,176 |
| 3 | glibc malloc | 2.127614 秒 | 94,001,995 |
| 3 | jemalloc | 1.763674 秒 | 113,399,655 |

三轮平均：

| 分配器 | 平均 elapsed | 平均 ops/sec |
|---|---:|---:|
| glibc malloc | 2.094450 秒 | 95,506,418 |
| jemalloc | 1.731174 秒 | 115,551,277 |

按平均吞吐量计算，jemalloc 比 glibc malloc 快约 21.0%；平均耗时降低约 17.3%。

但这是一项微基准。数组中最多同时存活一个元素，因此查重和查找几乎都是常数时间。它证明的是：

> 在这个进程和负载下，jemalloc 处理 key/value 的频繁申请与释放更快。

它不能证明 Array 能以同样速度处理一亿个同时存活的元素。

## 79. Array 的算法复杂度与内存测试边界

`kvs_array_set` 会先调用线性查询检查重复 key。连续插入 n 个不同 key 时，比较次数为：

```text
0 + 1 + 2 + ... + (n - 1)
= n * (n - 1) / 2
```

因此单次 SET 查重是 O(n)，连续插入 n 条数据的总成本是 O(n²)。如果假设插入 20 万条数据，理论比较次数约为 200 亿次。

这说明大规模场景下，Array 首先遇到的是数据结构复杂度瓶颈，而不是选择 glibc、内存池还是 jemalloc。

当前真实配置为：

```c
#define KVS_ARRAY_SIZE 1024
```

因此内存测试按真实上限插入 1024 项，而没有为了制造明显差异擅自扩大业务容量。

内存数据如下：

| 分配器 | 阶段 | VSZ | RSS |
|---|---|---:|---:|
| glibc malloc | after insert | 2256 KiB | 1548 KiB |
| glibc malloc | after delete | 2256 KiB | 1548 KiB |
| glibc malloc | after destroy | 2256 KiB | 1548 KiB |
| jemalloc | after insert | 10808 KiB | 3892 KiB |
| jemalloc | after delete | 10808 KiB | 3892 KiB |
| jemalloc | after destroy | 10808 KiB | 3892 KiB |

1024 份短字符串很小，三个阶段的变化被进程基础开销、分配器缓存和 `ps` 的观测粒度覆盖。jemalloc 还需要动态库、arena 和管理元数据，因此小规模场景中显示出更高的固定 VSZ/RSS。

这组数据说明：

```text
jemalloc 在该微基准中更快
但对只有 1024 容量的 Array，基础内存开销更高
删除或 destroy 后，分配器也未必立刻把内存归还操作系统
```

## 80. Array 测试的 Makefile 开关

Array benchmark 使用：

```makefile
ARRAY_USE_JEMALLOC ?= 0
```

当值为 0 时，不添加 jemalloc 参数，程序使用 glibc malloc；当值为 1 时，Makefile 添加：

```text
-DKVS_BENCHMARK_JEMALLOC
-ljemalloc
```

通过 `make -n` 可以只展开并打印编译命令，确认开关是否正确，而不真正执行编译。通过 `ldd` 则可以检查最终二进制是否实际依赖 `libjemalloc.so.2`。

## 81. 合并前的最终回归

合并前分别验证了两种完整配置。

节点池全部开启：

```text
HASH_USE_MEMORY_POOL=1
RBTREE_USE_MEMORY_POOL=1
SKIPLIST_USE_MEMORY_POOL=1
```

节点池全部关闭：

```text
HASH_USE_MEMORY_POOL=0
RBTREE_USE_MEMORY_POOL=0
SKIPLIST_USE_MEMORY_POOL=0
```

两种配置下均完成：

```text
内存池自身测试
Hash 分配器集成测试
红黑树分配器集成测试
跳表分配器集成测试
Array 内存生命周期测试
ASan 检查
完整 kvstore 编译和链接
```

开启节点池时，测试观察到删除节点的地址被 Free List 复用；关闭节点池时，输出明确进入 malloc mode。两条路径均未出现 ASan 错误。

## 82. 功能分支合并完成

本次工作在 `feature/memory-pool` 分支分阶段提交。合并前通过：

```bash
git log --oneline --decorate main..feature/memory-pool
```

确认了功能分支拥有、但 main 尚未拥有的提交范围。

随后使用：

```bash
git merge --no-ff feature/memory-pool \
  -m "merge: integrate memory pool and allocator benchmarks"
```

`--no-ff` 保留了明确的功能分支合并节点。最终 merge commit 为：

```text
3d3178a merge: integrate memory pool and allocator benchmarks
```

合并结果包含 11 个功能提交和 1 个 merge commit，共使本地 main 相对旧 main 前进 12 个提交。随后 main 已推送到 GitHub 和 GitLab。

## 83. 本轮真正内化的工程认识

这轮工作的核心并不是把 `malloc/free` 机械替换成两个新函数，而是建立了一套判断方法：

```text
先确认对象的尺寸是否固定
→ 再确认对象是否被频繁独立申请和释放
→ 明确每块内存的所有者
→ 申请失败时按相反顺序回滚
→ 删除时先断开数据结构连接，再释放或归还节点
→ destroy 释放所有长期持有的 Chunk
→ 用编译开关保留可比较的回退路径
→ 用 ASan 验证正确性
→ 用进程内 benchmark 隔离分配器成本
→ 用 VSZ/RSS 观察操作系统看到的内存结果
```

不同结构得出的选择也不同：

| 数据结构 | 当前内存策略 |
|---|---|
| Hash | 固定大小 node 外壳进入内存池；key/value 使用通用分配器 |
| 红黑树 | 普通 node 进入内存池；nil 独立管理；key/value 使用通用分配器 |
| 跳表 | 固定大小 node 外壳进入内存池；header 独立管理；可变 forward 和 key/value 使用通用分配器 |
| Array | item 保持连续数组整体分配；key/value 使用通用分配器 |

最终结论是：

> 内存优化必须服从数据结构本身的内存模型。适合节点池的对象才进入节点池；尺寸可变或已经连续预分配的对象继续使用通用分配器。自研内存池和 jemalloc 不是互斥关系，它们可以分别负责业务专用固定对象与通用动态内存。
