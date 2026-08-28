---
title: KV 存储项目：客户端测试、压力测试与多引擎扩展
slug: kv-store-testing-multi-engine
description: 从自动化客户端用例和 QPS 粗测出发，复盘数组状态问题的定位过程，并梳理红黑树以统一 5+2 接口接入 KV 项目的方式
date: 2026-08-25T00:00:00+08:00
draft: false
image: cover.svg
tags:
  - 项目记录
categories:
  - 后端开发
---

> 阅读目标：沿着“建立可重复的测试—用压力测试暴露状态问题—在现有协议后接入第二种 KV 引擎”这条主线，回顾项目从数组存储继续向多引擎结构演进的过程。
>
> 本文承接 [KV 存储项目网络层：统一接入 Reactor、Proactor 与协程]({{< ref "kv-store-network-layer" >}}) 和 [KV 存储项目：协议层与数组存储]({{< ref "kv-store-protocol-array" >}})。网络层的 Reactor、Proactor、ntyco，以及 `kvs_protocol()`、token 切分、数组五种操作等内容，前文已经完整讲过，这里只保留理解本阶段所需的连接点。
>
> 本文以学习笔记和当前代码为准，重点解释实现过程、测试现象与架构关系，不展开红黑树算法本身。

---

## 1. 这一阶段解决什么问题

前一阶段已经形成了一条可以工作的请求链路：

```text
客户端发送文本命令
        ↓
网络层接收数据
        ↓
kvs_protocol() 切分 token
        ↓
kvs_filter_protocol() 识别命令
        ↓
数组引擎执行 SET / GET / DEL / MOD / EXIST
        ↓
协议层生成 OK / EXIST / NO EXIST / value
        ↓
网络层把响应发回客户端
```

项目能处理命令以后，下一步不只是继续添加代码，而是先回答三个问题：

1. 已有功能能否被稳定、重复地验证？
2. 测试次数增加后，内部状态是否仍然正确？
3. 协议层能否在数组之外接入新的存储引擎？

所以本阶段的主线是：

```text
Makefile 统一构建
        ↓
TCP 客户端实现自动化功能测试
        ↓
重复执行测试并粗测 QPS
        ↓
通过高次数测试定位数组状态问题
        ↓
保留原有数组引擎，接入红黑树引擎
        ↓
用同一组语义测试第二种引擎
```

这里体现了一种项目迭代思路：先观察当前代码已经处于什么形态，再决定下一项功能应该接入哪一层。

---

## 2. 用 Makefile 固定构建过程

随着源文件增加，手工重复输入完整的 `gcc` 命令会越来越不方便。最初可以直接把编译过程写进 `all`：

```makefile
all:
	gcc -o kvstore kvstore.c reactor.c proactor.c kvs_array.c ntyco.c \
		-I ./NtyCo/core/ -L ./NtyCo/ -luring -lntyco

clean:
	rm -rf kvstore
```

这条命令由四部分组成：

| 部分 | 当前内容 | 作用 |
|---|---|---|
| 编译器 | `gcc` | 编译并链接 C 程序 |
| target | `kvstore` | 最终生成的程序 |
| sources | `kvstore.c` 等 | 参加编译的源文件 |
| flags | `-I`、`-L`、`-l` | 指定头文件、库目录和链接库 |

把这些部分提取成变量后，Makefile 中各项职责更清楚：

```makefile
CC=gcc
TARGET = kvstore
SRCS = kvstore.c reactor.c proactor.c kvs_array.c ntyco.c kvs_rbtree.c
INC = -I ./NtyCo/core/
LIBS = -L ./NtyCo/ -luring -lntyco

all:
	$(CC) -o $(TARGET) $(SRCS) $(INC) $(LIBS)

clean:
	rm -rf kvstore
```

当前阶段加入红黑树后，构建层面的变化很小：在 `SRCS` 中加入 `kvs_rbtree.c`。也就是说，存储引擎虽然增加了，但整个项目仍由同一个 target 统一构建。

### 2.1 NtyCo 是项目的外部依赖

`INC` 和 `LIBS` 都引用了 `./NtyCo/`：

```text
INC  → ./NtyCo/core/ 中的头文件
LIBS → ./NtyCo/ 中的 libntyco
```

因此，Makefile 描述的是“依赖已经存在时怎样编译”。如果本地没有 `NtyCo` 目录，还要先完成依赖准备，也就是把对应代码下载到项目约定的位置，再进行编译。

这可以把构建理解成两个阶段：

```text
准备第三方依赖
        ↓
编译当前项目
```

Makefile 的价值不只在于少输入一条命令，更重要的是把当前项目由哪些文件、头文件路径和库组成固定下来。

---

## 3. 客户端测试用例：把人工验证变成固定流程

测试客户端的基本流程很直接：

```text
1. 建立 TCP 连接
2. 发送一条协议命令
3. 接收服务端响应
4. 将实际响应与预期响应比较
5. 一致则 PASS，不一致则 FAILED
```

测试代码把这五步拆成几个职责单一的函数：

| 函数 | 作用 |
|---|---|
| `connect_tcpserver()` | 创建 socket，并连接指定 IP 和端口 |
| `send_msg()` | 发送一条测试命令 |
| `recv_msg()` | 接收服务端返回值 |
| `testcase()` | 完成一次“发送—接收—比较” |
| `array_testcase()` | 组织数组引擎的一组功能场景 |
| `rbtree_testcase()` | 组织红黑树引擎的一组功能场景 |

代码中的注释“一个函数只做一件事”在这里得到了体现：网络连接、一次断言和一组业务场景分别由不同函数承担。

### 3.1 一个测试用例包含什么

`testcase()` 接收四项信息：

```c
void testcase(int connfd, char *msg, char *pattern, char *casename)
```

它们分别表示：

| 参数 | 含义 | 示例 |
|---|---|---|
| `connfd` | 已连接的 TCP socket | 与 KV 服务端的连接 |
| `msg` | 发给服务端的命令 | `SET Dad Jasper` |
| `pattern` | 期望得到的响应 | `OK\r\n` |
| `casename` | 用于输出的测试名称 | `SET-Dad` |

测试执行过程可以压缩为：

```text
msg ──send──> KV 服务端
                  │
                  └── response ──recv──> result
                                             │
                                  strcmp(result, pattern)
                                      │             │
                                    相同           不同
                                      │             │
                                    PASS          FAILED
```

因此，测试用例并不需要知道数组或红黑树内部怎样保存数据。它只站在客户端角度检查协议行为，这正好覆盖了完整链路：

```text
TCP 收发 + 协议解析 + 引擎操作 + 响应生成
```

---

## 4. 一组完整场景不只是“五条命令”

项目一共有五种基础操作：

```text
SET / GET / DEL / MOD / EXIST
```

但“覆盖五个命令”不等于每个命令只执行一次。当前 `array_testcase()` 实际组织了九个连续步骤：

| 顺序 | 请求 | 预期响应 | 验证的状态 |
|---:|---|---|---|
| 1 | `SET Dad Jasper` | `OK` | 新键可以写入 |
| 2 | `GET Dad` | `Jasper` | 写入后的值可以读出 |
| 3 | `MOD Dad Sao` | `OK` | 已存在的键可以修改 |
| 4 | `GET Dad` | `Sao` | 修改结果已经生效 |
| 5 | `EXIST Dad` | `EXIST` | 已存在的键能被识别 |
| 6 | `DEL Dad` | `OK` | 已存在的键可以删除 |
| 7 | `GET Dad` | `NO EXIST` | 删除后无法再读取 |
| 8 | `MOD Dad Jasper` | `NO EXIST` | 删除后无法再修改 |
| 9 | `EXIST Dad` | `NO EXIST` | 删除后状态确实不存在 |

这九步不是彼此独立的命令，而是一条状态变化链：

```text
不存在
  │ SET
  ▼
Dad = Jasper
  │ MOD
  ▼
Dad = Sao
  │ DEL
  ▼
不存在
```

测试在每个关键状态后使用 `GET` 或 `EXIST` 进行确认。这样不仅验证命令返回了什么，还验证前一条命令是否真正改变了存储状态。

### 4.1 测试覆盖的两个方向

当前场景同时覆盖：

- 正向结果：写入成功、读取成功、修改成功、存在、删除成功；
- 反向结果：删除后读取、修改和判断存在均返回 `NO EXIST`。

“尽量覆盖项目所有结果，目标达到 90% 以上”可以理解为：不仅让每个函数运行一次，还要让函数中的主要结果分支都被执行到。对于协议层而言，要关注的结果包括：

```text
OK
EXIST
NO EXIST
ERROR
具体 value
```

测试用例会贯穿后续项目：新增命令、新增引擎或调整内部结构以后，都可以重复运行同一套预期，确认原有行为仍然成立。这就是回归测试的作用。

---

## 5. 从功能测试扩展到压力测试

功能测试回答的是：

> 一组操作的结果是否正确？

压力测试进一步回答：

> 同一组操作重复很多次以后，结果是否仍然正确？大约能处理多少请求？

`array_testcase_10w()` 将九条命令循环执行：

```c
int count = 10000;

for (i = 0; i < count; i++) {
    /* 每轮执行 9 条请求 */
}
```

所以请求总数为：

```text
10000 轮 × 9 条/轮 = 90000 条请求
```

计时使用 `gettimeofday()`：

```text
tv_begin：循环开始时间
tv_end：循环结束时间
time_used：两者相差的毫秒数
```

宏 `TIME_SUB_MS` 把秒和微秒的差值统一换算为毫秒：

```c
#define TIME_SUB_MS(tv1, tv2) \
    ((tv1.tv_sec - tv2.tv_sec) * 1000 + \
     (tv1.tv_usec - tv2.tv_usec) / 1000)
```

QPS 的基本计算关系是：

```text
QPS = 请求总数 ÷ 耗时秒数
    = 请求总数 × 1000 ÷ 耗时毫秒数
```

这里的“请求总数”应按实际循环次数和每轮命令数计算。数组测试中是 `90000`；红黑树测试同样是 `10000 × 9 = 90000`。阅读输出时先统一这个口径，两个引擎的数据才可以放在一起比较。

### 5.1 三次粗测说明了什么

笔记记录了 9000 次请求附近的三次结果：

| 测试环境 | 耗时 | 粗测 QPS |
|---|---:|---:|
| 客户端输出每次 PASS，服务端也打印 | 477 ms | 18867 |
| 屏蔽客户端 PASS 输出 | 415 ms | 21686 |
| 继续去掉服务端打印 | 285 ms | 31578 |

三组数据的重点不是得到一个绝对性能结论，而是观察到：

```text
测试总耗时
  = 网络收发耗时
  + 协议处理耗时
  + 引擎操作耗时
  + 客户端打印耗时
  + 服务端打印耗时
```

当 `printf()` 位于高频路径中，它也属于被计时的工作。逐步屏蔽打印后，QPS 上升，说明这次粗测测到的是整个端到端过程，而不只是 KV 引擎本身。

### 5.2 当前 QPS 的准确含义

当前压力客户端只有一个进程、一条 TCP 连接，并且每次都按下面的顺序执行：

```text
send 一条请求
    ↓
等待 recv 一条响应
    ↓
比较结果
    ↓
发送下一条请求
```

因此当前结果更准确地说是：

> 单客户端、单连接、串行请求、包含协议校验的端到端吞吐量。

它适合观察同一环境下的相对变化，例如开关日志前后的差异，以及数组与红黑树在相同测试场景下的差异。

---

## 6. 为什么小次数正常，高次数却失败

压力测试增加到 90000 次请求后，出现了：

```text
FAILED-> SET-Dad,ERROR!=OK
```

从客户端场景看，每轮都执行：

```text
SET Dad Jasper
...
DEL Dad
```

一轮结束时，`Dad` 已经删除，所以下一轮的 `SET Dad Jasper` 应该仍然能够成功。实际数据量始终只在 0 和 1 之间变化：

```text
SET 后：1 个元素
DEL 后：0 个元素
SET 后：1 个元素
DEL 后：0 个元素
```

但是数组实例使用 `total` 记录有效元素数量。如果插入成功执行 `total++`，删除成功却没有对应地维护数量，就会形成两套不同的状态：

```text
实际元素数量：0 → 1 → 0 → 1 → 0 ...
total：        0 → 1 → 1 → 2 → 2 ...
```

循环次数少时，`total` 还没有触及 `KVS_ARRAY_SIZE`，现象不明显；循环次数持续增加后，最终会到达容量判断：

```c
if (inst->total >= KVS_ARRAY_SIZE)
```

于是后续 `SET` 返回错误。与此同时，查找范围由 `total` 决定，`total` 不断增大也会使遍历范围越来越长，所以压力测试还能观察到性能逐渐变化。

### 6.1 `total` 的核心约定

数组引擎需要维持一个清楚的不变量：

```text
table[0] 到 table[total - 1]：有效元素
table[total] 以及后面的位置：空闲空间
total：当前有效元素数量
```

按照这个约定：

```text
插入成功 → total++
删除成功 → total--
```

如果删除数组中间的元素，可以用最后一个有效元素补到删除位置，从而继续保持有效区间连续：

```text
删除前：[Dad, Mom, Son]    total = 3
删除 Mom 后出现空位：[Dad, 空, Son]
把最后一个元素移到空位：[Dad, Son, 空]
最终：total = 2
```

这个问题说明，容量字段不是只在创建时赋值、插入时递增的计数器，而是数据结构状态的一部分。每一种改变元素数量的操作都必须共同维护它。

---

## 7. 压力测试进一步暴露的几个语义问题

高次数运行不仅检查容量，还会不断经过删除、查询不存在和修改不存在等分支，因此能把返回值、内存和状态维护问题一起暴露出来。

### 7.1 删除 value 时的内存顺序

下面这个表达式先执行赋值：

```c
kvs_free(inst->table[i].value = NULL);
```

它的实际求值顺序是：

```text
inst->table[i].value = NULL
        ↓
kvs_free(NULL)
```

原来保存在 `value` 中的地址因此没有被交给 `kvs_free()`。正确理解“释放后置空”的顺序是：

```text
先把原地址交给 kvs_free
        ↓
再把保存该地址的指针设为 NULL
```

也就是：

```c
kvs_free(inst->table[i].key);
inst->table[i].key = NULL;

kvs_free(inst->table[i].value);
inst->table[i].value = NULL;
```

这里要记住的不是某一行写法，而是指针生命周期：地址仍然可达时释放内存，释放完成后再清除悬空指针。

### 7.2 循环下标与业务状态不是同一种含义

如果同时存在：

```c
int i = 0;

for (int i = 0; i < inst->total; i++) {
    ...
}
```

`for` 中的 `i` 属于内层作用域，会遮蔽外层的 `i`。循环结束后，外层 `i` 仍是初始值 `0`。

更关键的是，即使只保留一个 `i`，`return i` 仍然混合了两种概念：

```text
i：遍历到了哪个数组下标
返回值：本次 KV 操作是什么结果
```

数组为空时尤其容易看清这个区别：

```text
i = 0
total = 0
循环一次都不执行
return i 得到 0
```

这里的 `0` 只是下标初始值，并不能说明修改或删除成功。

因此，学习时应把业务状态单独记忆：

| 返回值 | 协议层理解 |
|---:|---|
| `< 0` | 参数或执行错误，生成 `ERROR` |
| `0` | 操作成功，或 `EXIST` 查询确认存在 |
| `> 0` | SET 时表示已存在；DEL/MOD/EXIST 时表示不存在 |

### 7.3 两种失败现象来自同一套返回值约定

删除成功后如果返回 `1`，协议层会按照既定规则生成：

```text
DEL 返回 1 → NO EXIST
```

于是客户端看到：

```text
FAILED-> DEL-Dad,NO EXIST!=OK
```

删除以后再执行 `MOD`，如果数组为空、循环未执行，最后又通过 `return i` 返回初始值 `0`，协议层会生成：

```text
MOD 返回 0 → OK
```

于是客户端看到：

```text
FAILED-> MOD-Dad,OK!=NO EXIST
```

这两种现象虽然出现在不同命令中，核心都是同一件事：引擎函数的返回值属于内部业务协议，协议层会严格按照该约定翻译成文本响应。

可以把这条关系记成：

```text
引擎内部状态
      ↓
固定返回值
      ↓
kvs_filter_protocol()
      ↓
客户端可见响应
      ↓
testcase() 与预期比较
```

测试失败信息最终帮助定位到的，不一定是 TCP 层，也可能是更深处的状态维护或返回值语义。

---

## 8. 从单一数组引擎扩展为多引擎

前一阶段的协议层只连接数组：

```text
SET / GET / DEL / MOD / EXIST
              ↓
        global_array
```

本阶段保留已经成型的数组实现，并通过宏加入红黑树：

```c
#define ENABLE_ARRAY  1
#define ENABLE_RBTREE 1
```

这两个开关控制相关类型、接口、全局实例、协议分支、初始化和销毁代码是否参与编译。这样可以在接入期间让两种引擎同时存在，并分别运行测试。

项目结构由此变成：

```text
                         kvs_protocol
                              ↓
                     kvs_filter_protocol
                              ↓
              根据命令选择对应的引擎
                    ↙                   ↘
   SET/GET/DEL/MOD/EXIST       RSET/RGET/RDEL/RMOD/REXIST
              ↓                            ↓
        global_array                 global_rbtree
```

这里没有改变底层网络。无论使用 Reactor、Proactor 还是 ntyco，网络层仍然只调用同一个 `kvs_protocol()`。增加的是协议层后面的存储选择。

---

## 9. 为什么给红黑树命令增加 `R` 前缀

一条普通的 `SET Dad Jasper` 只表达了“设置键值”，没有说明数据应该存进数组还是红黑树。

当前阶段采用最直接的区分方法：在协议命令中加入引擎标识。

| 数组命令 | 红黑树命令 | 语义 |
|---|---|---|
| `SET` | `RSET` | 新增 key-value |
| `GET` | `RGET` | 查询 value |
| `DEL` | `RDEL` | 删除 key-value |
| `MOD` | `RMOD` | 修改 value |
| `EXIST` | `REXIST` | 判断 key 是否存在 |

命令字符串表因此扩展为十项：

```c
const char *command[] = {
    "SET", "GET", "DEL", "MOD", "EXIST",
    "RSET", "RGET", "RDEL", "RMOD", "REXIST"
};
```

枚举与字符串表保持相同顺序，`kvs_filter_protocol()` 找到命令编号后，通过 `switch` 进入对应分支：

```text
RSET
  ↓
KVS_CMD_RSET
  ↓
kvs_rbtree_set(&global_rbtree, key, value)
  ↓
按照返回值生成 OK / EXIST / ERROR
```

因此，`R` 前缀在当前阶段承担的是一个简单的路由信息：让协议层知道应该调用哪种引擎。

---

## 10. 每种数据结构都遵守“5 + 2”接口

数组引擎已经形成七个接口：

```text
create
destory
set
get
del
mod
exist
```

可以压缩为：

```text
2 个生命周期操作 + 5 个 KV 操作 = 5 + 2
```

红黑树接入时也提供相同形态：

```c
int   kvs_rbtree_create(kvs_rbtree_t *inst);
void  kvs_rbtree_destory(kvs_rbtree_t *inst);
int   kvs_rbtree_set(kvs_rbtree_t *inst, char *key, char *value);
char *kvs_rbtree_get(kvs_rbtree_t *inst, char *key);
int   kvs_rbtree_del(kvs_rbtree_t *inst, char *key);
int   kvs_rbtree_mod(kvs_rbtree_t *inst, char *key, char *value);
int   kvs_rbtree_exist(kvs_rbtree_t *inst, char *key);
```

协议层因此不需要理解旋转、着色、后继节点等红黑树内部细节。它只关心：

```text
用哪个实例
调用哪个 KV 操作
如何解释返回值
```

这正是本次“加入红黑树”最值得回顾的部分：不是从零推导红黑树算法，而是把已有数据结构包装成项目已经形成的 KV 引擎接口。

同样的思路也可以用于后续哈希引擎：哈希表内部怎样定位 key 属于数据结构实现；接入项目时，仍围绕生命周期和五种 KV 语义组织接口。

---

## 11. 红黑树在项目中的最小理解范围

为了理解本项目中的接入，只需要掌握下面几层关系。

### 11.1 树实例和节点

红黑树实例保存：

```c
typedef struct _rbtree {
    rbtree_node *root;
    rbtree_node *nil;
} rbtree;
```

其中：

- `root` 指向根节点；
- `nil` 是哨兵节点，用于表示空孩子和空树边界；
- `nil` 初始化为黑色；
- 创建完成时，`root` 指向 `nil`，表示树中还没有 KV 节点。

节点中与 KV 项目直接相关的是：

```text
key   → 用于比较和查找
value → 与 key 对应的数据
```

父节点、左右孩子和颜色则服务于红黑树结构本身。

### 11.2 五种 KV 操作如何映射到底层树操作

| KV 接口 | 在红黑树中的主要过程 |
|---|---|
| `kvs_rbtree_set()` | 为 key/value 分配空间，构造节点并插入树 |
| `kvs_rbtree_get()` | 按 key 搜索节点，返回节点的 value |
| `kvs_rbtree_del()` | 搜索节点，删除节点并释放相关空间 |
| `kvs_rbtree_mod()` | 搜索节点，替换已有 value |
| `kvs_rbtree_exist()` | 搜索节点，通过是否到达 `nil` 判断存在性 |

这些包装函数把“树节点操作”转换成与数组引擎一致的“KV 操作”。

### 11.3 生命周期怎样进入程序主流程

程序启动时，两种引擎分别初始化：

```text
init_kvengine()
  ├─ kvs_array_create(&global_array)
  └─ kvs_rbtree_create(&global_rbtree)
```

网络服务结束后，再进入统一销毁阶段：

```text
dest_kvengine()
  ├─ kvs_array_destory(&global_array)
  └─ kvs_rbtree_destory(&global_rbtree)
```

红黑树销毁时需要逐步删除树中的节点，最后处理树实例的哨兵节点。具体采用最小节点、最大节点或其他顺序，是树内部的销毁策略；对项目主线而言，重要的是 `destory` 与 `create` 共同构成完整生命周期。

---

## 12. 用同一套语义验证红黑树引擎

`rbtree_testcase()` 与 `array_testcase()` 的业务顺序保持一致，只是命令增加了 `R` 前缀：

```text
数组：   SET → GET → MOD → GET → EXIST → DEL → GET → MOD → EXIST
红黑树：RSET → RGET → RMOD → RGET → REXIST → RDEL → RGET → RMOD → REXIST
```

这使测试具有两个层次：

1. 单独验证每个引擎是否满足五种 KV 操作；
2. 验证两种引擎在相同业务场景下是否表现出相同语义。

可以把两组用例看成一份“行为契约”：

| 初始状态和操作 | 数组响应 | 红黑树响应 |
|---|---|---|
| 不存在时 SET | `OK` | `OK` |
| SET 后 GET | 对应 value | 对应 value |
| 已存在时 MOD | `OK` | `OK` |
| 已存在时 EXIST | `EXIST` | `EXIST` |
| 已存在时 DEL | `OK` | `OK` |
| DEL 后 GET/MOD/EXIST | `NO EXIST` | `NO EXIST` |

只要对外行为一致，客户端就不需要了解底层究竟使用数组还是红黑树。当前协议用命令前缀显式选择引擎，但测试验证的是二者共同的 KV 语义。

---

## 13. 数组与红黑树在当前项目中的角色

本阶段不是简单地用红黑树替换数组，而是让两种实现同时存在：

| 维度 | 数组引擎 | 红黑树引擎 |
|---|---|---|
| 协议命令 | 五种普通命令 | 五种 `R` 前缀命令 |
| 全局实例 | `global_array` | `global_rbtree` |
| 生命周期 | `create/destory` | `create/destory` |
| KV 接口 | `set/get/del/mod/exist` | `set/get/del/mod/exist` |
| 内部组织 | 连续数组区间 | 根节点、哨兵和树节点 |
| 查找方式 | 在有效区间中遍历 | 按 key 在树中搜索 |

真正稳定下来的不是某一种数据结构，而是数据结构外面的项目边界：

```text
协议命令
    ↓
统一的 KV 语义
    ↓
不同引擎各自实现
```

这也解释了“要什么需求、用什么技术方案实现”的关系：

- 需求是保存、查询、删除、修改和判断 key；
- 数组和红黑树是两种实现这些需求的技术方案；
- 协议层负责把客户端命令路由给具体方案；
- 测试用例负责确认不同方案满足相同的外部预期。

---

## 14. 关于多进程、多线程压力测试的课后思考

当前单连接测试中，每条请求都要等待上一条响应，无法同时给服务端制造多个在途请求。要观察并发场景，可以让多个执行单元各自建立连接并运行同一组测试。

### 14.1 多进程思路

```text
父进程
  ├─ fork → 子进程 1 → 建立连接 → 重复测试
  ├─ fork → 子进程 2 → 建立连接 → 重复测试
  └─ fork → 子进程 N → 建立连接 → 重复测试
```

每个子进程拥有自己的客户端 socket 和测试循环。父进程等待所有子进程结束，再汇总：

```text
总请求数 = 进程数 × 每个进程的请求数
整体 QPS = 总请求数 × 1000 ÷ 整体耗时毫秒数
```

### 14.2 多线程思路

```text
一个客户端进程
  ├─ 线程 1 → 独立连接 → 重复测试
  ├─ 线程 2 → 独立连接 → 重复测试
  └─ 线程 N → 独立连接 → 重复测试
```

每个线程使用独立连接时，测试流程和当前代码最接近；主线程统一记录开始与结束时间，并统计所有线程完成的请求总数。

### 14.3 并发数增加后观察什么

并发测试不只是期待 QPS 一定上升，还要同时观察：

- 所有响应是否仍然符合预期；
- 总吞吐量怎样随客户端数量变化；
- 单次请求延迟是否增加；
- 服务端采用的网络模型是否能发挥并发处理能力；
- 打印、网络、协议或存储引擎中的哪一部分先成为主要耗时。

因此，多进程或多线程的意义是改变负载模型：从“一个客户端串行发请求”变成“多个客户端同时发请求”。

---

## 15. 这一阶段形成的完整项目画面

把前三个阶段放在一起，可以得到当前项目结构：

```text
测试客户端
  ├─ 功能用例：检查每一步响应
  └─ 压力用例：重复场景、计时、计算 QPS
             │
             ▼
          TCP 网络
             │
             ▼
网络层：Reactor / Proactor / ntyco
             │
             ▼
kvs_protocol()：切分 token
             │
             ▼
kvs_filter_protocol()：识别命令并选择引擎
        ┌────┴────┐
        ▼         ▼
   数组引擎    红黑树引擎
  5 + 2 接口   5 + 2 接口
        │         │
        └────┬────┘
             ▼
    统一返回值与文本响应
```

Makefile 则从构建角度把这些模块放到一起：

```text
kvstore.c
reactor.c / proactor.c / ntyco.c
kvs_array.c
kvs_rbtree.c
NtyCo 与 liburing
        ↓
      kvstore
```

---

## 16. 最后的复习提纲

### 16.1 测试主线

```text
connect
  → send command
  → recv response
  → compare pattern
  → PASS / FAILED
```

### 16.2 压力测试主线

```text
固定业务场景
  → 重复执行
  → 统计实际请求数
  → 记录总耗时
  → 计算 QPS
  → 从失败轮次和响应反查内部状态
```

### 16.3 数组问题定位主线

```text
少量测试正常、增加次数后 SET 返回 ERROR
  → 检查容量条件
  → 检查 total 的增减
  → 发现实际元素已删除但 total 仍增长
  → 恢复“有效元素数量”的统一约定
```

同时记住：

```text
释放内存：先 free 原地址，再把指针置 NULL
循环下标：只表示位置
函数返回值：表示业务结果
协议响应：由业务返回值翻译得到
```

### 16.4 多引擎扩展主线

```text
保留数组成型代码
  → 用 ENABLE_ARRAY / ENABLE_RBTREE 控制编译
  → 为红黑树提供同样的 5 + 2 接口
  → 增加 RSET/RGET/RDEL/RMOD/REXIST
  → 协议层按命令选择实例
  → 用同一套业务场景验证两种引擎
```

### 16.5 本阶段最重要的认识

> 测试用例不是项目完成后的附属代码，而是项目继续迭代时用来确认行为的固定标准。

> 小次数测试验证基本功能，高次数测试还会验证状态能否长期保持一致。

> `total`、节点数量和指针生命周期都属于数据结构必须维护的内部状态。

> 数组、红黑树以及后续哈希表可以采用不同内部实现，但都可以通过相同的 KV 语义接入协议层。

> 本阶段加入红黑树的重点，是把一种已有数据结构组织成项目需要的存储引擎，而不是从零实现红黑树算法。
