---
title: KV 进化（一）：给 91kvstore 接上一条跳表
slug: kv-evolution-skiplist-engine
description: 从整数跳表改造为字符串 KV，补齐统一接口，并完成内存管理、Makefile、生命周期、命令协议与端到端测试的完整接入
date: 2026-08-28T00:00:00+08:00
draft: false
image: cover.svg
tags:
  - KV 存储
  - SkipList
  - C 语言
  - 数据结构
  - 自动化测试
categories:
  - 后端开发
---

> 这篇不打算深挖跳表算法，而是记录一个外部数据结构如何从“能够独立运行”，一步步变成 91kvstore 中真正可用的存储引擎。

目前这个 KV 存储引擎已经支持三种数据结构：

- 结构体数组
- 红黑树
- 哈希表

那么问题来了：能不能再加入一种新的存储引擎，并且让它像现有的三个引擎一样，通过统一接口和网络命令工作？

这次选择的是——跳表。

最终希望实现这样的效果：

```
SSET name Jasper
SGET name
SMOD name Sao
SEXIST name
SDEL name
```

表面上看，只是多写一个 `kvs_skiplist.c`；但真正有价值的部分并不是手搓跳表，而是研究如何把一个外部数据结构完整地融合进现有项目。

## 先对跳表有个大致印象

开始之前，我先看了一些简短的跳表介绍，比如这个视频：

[B站：跳表相关介绍](https://www.bilibili.com/video/BV1uCYLeiEzw/?spm_id_from=333.337.search-card.all.click&vd_source=6bacd16ae2a23fd992d2eb359c2d027b)

现在并不打算深入研究跳表的每一个概率细节，只需要先形成几个基本认知：

- 跳表的底层仍然是一条有序链表；
- 它在普通链表上增加了多层索引；
- 查询时可以从高层快速跳过大量节点；
- 查找、插入、删除的平均时间复杂度是 `O(log n)`。

在工程中使用一种数据结构时，确实可以把它当成一个黑盒，但也不能完全不理解。至少要知道：

- 它需要什么样的输入；
- 它如何管理内存；
- 它能提供哪些操作；
- 它的节点和数据由谁创建、由谁释放。

理解到这个程度，就足够开始融合了。

## 写跳表之前，我先整理了项目

我多少有点目录洁癖。

原来的 `.c`、`.o`、可执行文件和头文件全部堆在项目根目录，看着实在难受。因此没有立刻开始写跳表，而是先参照常见开源项目的结构，把工程重新整理了一遍：

```
91kvstore/
├── .gitignore
├── .gitmodules
├── Makefile
│
├── clients/
│   ├── go-kvstore.go
│   ├── javakvstore.java
│   ├── js-kvstore.js
│   ├── py-kvstore.py
│   └── rust-kvstore.rs
│
├── include/
│   ├── kvstore.h
│   └── server.h
│
├── src/
│   ├── kvstore.c
│   ├── engines/
│   │   ├── kvs_array.c
│   │   ├── kvs_hash.c
│   │   ├── kvs_rbtree.c
│   │   └── kvs_skiplist.c
│   │
│   └── network/
│       ├── ntyco.c
│       ├── proactor.c
│       └── reactor.c
│
├── tests/
│   └── testcase.c
│
├── third_party/
│   └── NtyCo/              # Git 子模块
│
├── build/                  # 编译产物，不提交
│   ├── kvstore.o
│   ├── reactor.o
│   ├── proactor.o
│   ├── ntyco.o
│   ├── kvs_array.o
│   ├── kvs_hash.o
│   ├── kvs_rbtree.o
│   └── kvs_skiplist.o
│
└── bin/                    # 可执行文件，不提交
    ├── kvstore
    └── testcase
```

这样一来，各部分的职责就很直观：

- `include/` 放公共声明；
- `src/engines/` 放存储引擎；
- `src/network/` 放网络模型；
- `tests/` 放测试程序；
- `third_party/` 放第三方依赖；
- `build/` 和 `bin/` 只保存编译产物。

规整多了，right？

## 第一步：把整数跳表改造成字符串 KV

找到的简易跳表实现只支持：

```
int key;
int value;
```

但 KV 存储引擎需要的是：

```
char *key;
char *value;
```

因此节点最终变成：

```
typedef struct kvs_skiplist_node {
    char *key;
    char *value;
    struct kvs_skiplist_node **forward;
} kvs_skiplist_node_t;
```

这里不只是把 `int` 改成 `char *` 那么简单。

整数可以直接比较：

```
node->key < key
```

字符串必须使用：

```
strcmp(node->key, key) < 0
```

同时，字符串不能只保存调用方传进来的地址。节点需要为 `key` 和 `value` 单独申请内存并复制内容，否则调用方的字符串失效后，跳表里保存的指针也会失效。

创建一个节点，大致需要依次完成：

1. 为节点结构体申请内存；
2. 为 `key` 申请内存；
3. 复制 `key`；
4. 为 `value` 申请内存；
5. 复制 `value`；
6. 为每层 `forward` 指针申请空间。

每一步都可能失败。

如果中途失败，就必须把前面已经申请成功的内存释放掉。否则节点虽然没有创建成功，却会留下内存泄漏。

这部分看起来啰嗦，但它实际上比跳表算法本身更接近真实的 C 工程开发。

## 第二步：补齐统一接口

原始代码只有插入、查询和展示，但项目中的每个存储引擎都需要提供七个接口：

```
kvs_skiplist_create
kvs_skiplist_destory
kvs_skiplist_set
kvs_skiplist_get
kvs_skiplist_del
kvs_skiplist_mod
kvs_skiplist_exist
```

其中比较复杂的是：

- `set`：查找插入位置，并更新多层指针；
- `get`：沿索引层逐级查找；
- `del`：不仅要释放节点，还必须先从每一层中摘除节点。

`mod` 和 `exist` 相对简单，因为它们都可以复用内部查询函数：

```
static kvs_skiplist_node_t *
skiplist_search_node(kvs_skiplist_t *inst, const char *key);
```

`exist` 只需要判断查询结果是否为 `NULL`。

`mod` 则是在找到节点后，先申请并复制新的 value；成功之后再释放旧 value。这个顺序非常重要：

```
先申请新内存
    ↓
申请成功
    ↓
释放旧 value
    ↓
替换指针
```

如果先释放旧 value，再发现新内存申请失败，原来的数据也就丢了。

## 第三步：统一返回值

接口能工作还不够，它必须遵守现有引擎的返回规则。

最终统一为：

|返回值|含义|
|---|---|
|`0`|操作成功，或者 key 存在|
|`1`|key 重复，或者 key 不存在|
|`-1`|参数或实例不合法|
|`-2`|内存申请失败|

具体含义需要结合接口理解。

例如：

```
kvs_skiplist_set(...)
```

返回 `1` 表示 key 已经存在。

而：

```
kvs_skiplist_del(...)
```

返回 `1` 表示 key 不存在。

统一返回规则非常重要，因为协议层并不关心跳表内部发生了什么，它只根据返回值生成响应：

```
0   → OK
1   → EXIST 或 NO EXIST
< 0 → ERROR
```

## 第四步：别让测试变成纯苦力

最开始，每修改一个函数，就在 `main()` 中增加一堆 `if/else`，重新编译、运行、检查输出。

写到后面发现，大量测试代码无非是在做：

```
实际返回值是否等于预期返回值？
```

于是写了一个辅助函数：

```
static void check_test(int condition, const char *testName)
{
    if (condition) {
        printf("[PASS] %s\n", testName);
    } else {
        printf("[FAIL] %s\n", testName);
    }
}
```

测试就能简化为：

```
result = kvs_skiplist_set(
    skipList,
    "Dad",
    "Jasper"
);

check_test(
    result == 0,
    "set new key"
);
```

开发接口期间，可以先用 `#if 0` 屏蔽 `main()`，每完成一批接口只进行对象文件编译：

```
gcc -Iinclude \
    -c src/engines/kvs_skiplist.c \
    -o /tmp/kvs_skiplist.o
```

等全部接口完成后，再临时启用 `main()`，统一进行一次完整测试。

这样既能及时发现编译错误，也不用每改一个函数就维护一遍测试流程。

## 第五步：从“能运行”走向“融入项目”

底层接口通过测试，只能说明跳表自己能工作。接下来才是真正的工程融合。

首先，把公共内容移动到 `include/kvstore.h`：

- 最大层数定义；
- 跳表节点类型；
- 跳表实例类型；
- 七个公开接口声明。

而这些内部辅助函数仍然留在 `.c` 文件中，并使用 `static` 隐藏：

```
skiplist_create_node
skiplist_random_level
skiplist_search_node
```

这样外部模块只知道跳表“能做什么”，不需要知道它“具体怎么做”。

接着，把直接使用的：

```
malloc
free
```

替换为项目统一封装的：

```
kvs_malloc
kvs_free
```

再把：

```
build/kvs_skiplist.o
```

加入 Makefile，让跳表参与整个项目的编译与链接。

Makefile 的具体语法还需要继续学习，但目前已经能看懂它最核心的关系：

```
源文件
  ↓ 编译
对象文件
  ↓ 链接
可执行文件
```

## 第六步：创建全局跳表实例

服务器需要一个长期存在的跳表实例，保存所有客户端写入的数据。

因此在 `kvs_skiplist.c` 中定义：

```
kvs_skiplist_t global_skiplist;
```

在 `kvstore.c` 中声明：

```
extern kvs_skiplist_t global_skiplist;
```

这里的 `extern` 并不会创建第二个跳表，它只是告诉 `kvstore.c`：

> 这个变量在其他文件中已经定义，你可以直接使用它。

然后把跳表加入存储引擎生命周期：

```
kvs_skiplist_create(&global_skiplist);
```

服务器退出时：

```
kvs_skiplist_destory(&global_skiplist);
```

至此，跳表拥有了完整的生命过程：

```
定义 → 初始化 → 处理请求 → 销毁
```

## 第七步：命令协议融合

最后，也是最关键的一步，是让客户端真的能够使用跳表。

加入五条新命令：

```
SSET
SGET
SDEL
SMOD
SEXIST
```

项目并不存在一个“当前使用哪个 engine”的开关，而是通过命令前缀直接选择引擎：

```
SET    → array
RSET   → rbtree
HSET   → hash
SSET   → skiplist
```

例如客户端发送：

```
SSET Dad Jasper
```

服务器的处理过程是：

```
解析 SSET
    ↓
匹配 KVS_CMD_SSET
    ↓
进入 switch 对应分支
    ↓
调用 kvs_skiplist_set()
    ↓
根据返回值生成 OK / EXIST / ERROR
```

这一步完成后，跳表就不再是一个孤立的 `.c` 文件，而是真正成为了 KV 存储引擎的一员。

## 最后：用 testcase 做端到端测试

底层 `main()` 测试的是跳表接口，而 `tests/testcase.c` 测试的是完整链路：

```
测试客户端
 → TCP
 → 网络层
 → 协议解析
 → global_skiplist
 → 跳表接口
 → 协议响应
```

测试程序会自动发送：

```
SSET
SGET
SEXIST
SMOD
SDEL
```

然后将服务器的实际响应与预期结果比较：

```
相同 → PASS
不同 → FAILED
```

相比手动输入命令，这种测试不仅省事，还能在后续修改代码时快速确认：跳表功能有没有被意外破坏。

## 小结

这次表面上是在“增加一个跳表”，实际上完整经历了一个新模块接入现有项目的过程：

```
选择基础实现
 → 改造成字符串 KV
 → 补齐统一接口
 → 统一返回规则
 → 完成底层测试
 → 暴露公共声明
 → 接入内存管理
 → 接入 Makefile
 → 创建全局实例
 → 接入生命周期
 → 接入命令协议
 → 完成端到端测试
```

跳表算法当然重要，但这次更重要的收获是：

> 一个数据结构能独立运行，和它能成为项目中真正可用的一部分，是两件完全不同的事。

前者解决算法问题，后者解决接口、内存、构建、生命周期、协议和测试问题。

而后面这些，才是这次“KV 进化”真正有意思的地方。
