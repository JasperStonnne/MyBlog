---
title: KV 进化（二）：Snapshot 全量快照 + AOF 增量日志
slug: kv-evolution-snapshot-aof-persistence
description: 从 AOF 逐命令持久化出发，实现多引擎全量快照、安全替换、偏移量衔接与启动混合恢复，完成 91kvstore 的持久化闭环
date: 2026-08-30T00:00:00+08:00
draft: false
image: cover.svg
tags:
  - KV 存储
  - AOF
  - Snapshot
  - 持久化
  - C 语言
  - 文件系统
categories:
  - 项目记录
---

> 这篇承接 [《KV 进化（一）：给 91kvstore 接上一条跳表》]({{< ref "kv-evolution-skiplist-engine" >}})。上一阶段完成了第四种存储引擎的接入，这一阶段继续解决一个更实际的问题：服务退出以后，内存中的数据怎样留下来？
>
> 本文记录从 AOF 基础闭环，到全量快照，再到通过 `AOF_OFFSET` 连接二者的完整实现过程。

## 1. 确立实现模式

项目采用混合持久化模式：

- 全量快照：保存某一时刻完整的数据集。
- AOF 增量日志：每次成功执行写命令后，记录这条命令。

AOF 的基础闭环为：

```text
执行写命令
→ 写入 AOF
→ 服务重启
→ 读取 AOF
→ 重新执行历史命令
→ 恢复内存数据
```

加入全量快照以后，最终恢复流程变成：

```text
加载全量快照
→ 获得快照对应的 AOF 偏移量
→ 从偏移量开始重放 AOF
→ 恢复出最新数据
```

## 2. 建立独立的持久化模块

持久化属于独立的辅助模块，不应该把所有文件操作都堆在 `kvstore.c` 中，因此新增：

```text
include/persistence.h
src/persistence/aof.c
src/persistence/snapshot.c
```

其他模块通过：

```c
#include "persistence.h"
```

使用持久化功能。

头文件中声明了 AOF 对外提供的接口：

```c
int kvs_aof_open(const char *path);
int kvs_aof_append(char **tokens, int count);
int kvs_aof_close(void);
long long kvs_aof_get_offset(void);
```

为了实现重放，又增加了回调函数类型：

```c
typedef int (*aof_replay_handler)(
    char *msg,
    int length,
    char *response
);
```

以及 AOF 重放接口：

```c
int kvs_aof_replay(
    const char *path,
    long long offset,
    aof_replay_handler handler
);
```

快照模块对外提供：

```c
int kvs_snapshot_save(const char *path);

long long kvs_snapshot_load(
    const char *path,
    aof_replay_handler handler
);
```

`aof_replay_handler` 是函数指针类型。重放函数读取到一条命令后，可以通过这个函数指针把命令交给 `kvs_protocol()` 执行。

## 3. 打开和关闭 AOF

AOF 文件使用一个模块内部的文件指针保存：

```c
static FILE *aof_fp = NULL;
```

打开文件时使用追加模式：

```c
aof_fp = fopen(path, "a");
```

`"a"` 表示 append，也就是从文件末尾追加内容，不会覆盖已有日志。

`kvs_aof_open()` 需要检查：

- `path` 是否为 `NULL`
- AOF 是否已经打开
- `fopen()` 是否成功

约定返回值：

```text
0   成功
-1  失败
```

关闭时使用：

```c
int ret = fclose(aof_fp);
aof_fp = NULL;
```

即使 `fclose()` 失败，也要把 `aof_fp` 设置为 `NULL`。因为调用 `fclose()` 以后，不应该继续使用原来的文件指针。

## 4. 把命令追加到 AOF

协议层已经把命令切分到了 `tokens` 中。例如：

```text
SET name Jasper
```

切分以后是：

```text
tokens[0] = "SET"
tokens[1] = "name"
tokens[2] = "Jasper"
count = 3
```

`kvs_aof_append()` 首先检查：

- AOF 文件是否已经打开
- `tokens` 是否为 `NULL`
- `count` 是否为 `2` 或 `3`
- 对应位置的 token 是否存在

之后使用 `fprintf()` 把 tokens 重新组合成一条完整命令：

```c
fprintf(aof_fp, "%s %s\n",
        tokens[0], tokens[1]);
```

或者：

```c
fprintf(aof_fp, "%s %s %s\n",
        tokens[0], tokens[1], tokens[2]);
```

因此 AOF 文件中的内容类似：

```text
SET persist_key persist_value
MOD persist_key new_value
DEL persist_key
```

每条命令占一行，这样重放时才能确定不同命令之间的边界。

## 5. 从缓冲区同步到文件

`fprintf()` 返回值大于等于 `0`，只能说明格式化写入没有在这一层报告错误。数据可能仍然位于 C 标准库维护的用户态缓冲区中。

因此先调用：

```c
fflush(aof_fp);
```

它会把 C 标准库缓冲区中的数据交给操作系统。

但是数据此时仍可能位于操作系统的页缓存中，所以继续调用：

```c
fsync(fileno(aof_fp));
```

这里涉及两种文件表示：

```text
FILE *  C 标准库使用的文件流
fd      操作系统使用的整数文件描述符
```

`fsync()` 接收文件描述符，所以需要使用：

```c
fileno(aof_fp)
```

把 `FILE *` 转换成对应的 `fd`。

于是一次追加的顺序是：

```text
fprintf
→ fflush
→ fileno
→ fsync
```

其中任何一步失败，`kvs_aof_append()` 都返回 `-1`。

## 6. 判断哪些命令需要写入 AOF

AOF 只记录会改变数据的命令：

```text
SET
DEL
MOD
```

查询命令不需要记录：

```text
GET
EXIST
```

当前项目中，每个数据结构引擎的命令都按照五个一组排列：

```text
SET GET DEL MOD EXIST
```

所以可以通过 `cmd % 5` 判断命令类型：

```c
static int kvs_is_write_command(int cmd)
{
    if (cmd < KVS_CMD_START || cmd >= KVS_CMD_COUNT) {
        return 0;
    }

    switch (cmd % 5) {
    case 0:
    case 2:
    case 3:
        return 1;
    default:
        return 0;
    }
}
```

对应关系是：

```text
0 → SET
1 → GET
2 → DEL
3 → MOD
4 → EXIST
```

## 7. 只有命令执行成功才记录

AOF 的写入位置是在业务命令执行完成、拿到 `ret` 之后。

原因是：如果命令执行失败，却仍然被写进 AOF，那么下次重启时可能恢复出一个与用户实际操作结果不一致的数据集。

当前判断为：

```c
if (!aof_replaying &&
    ret == 0 &&
    kvs_is_write_command(cmd)) {

    int aof_ret = kvs_aof_append(tokens, count);

    if (aof_ret < 0) {
        fprintf(stderr, "failed to append AOF\n");
        exit(EXIT_FAILURE);
    }
}
```

三个条件分别表示：

```text
!aof_replaying             当前不是重放阶段
ret == 0                   业务命令执行成功
kvs_is_write_command(cmd)  当前是写命令
```

如果业务数据已经修改成功，但是 AOF 写入失败，服务选择立即退出。否则程序继续运行，就会出现：

```text
内存中修改成功
→ AOF 中没有记录
→ 服务重启
→ 用户的数据丢失
```

## 8. AOF 重放

AOF 重放必须放在 KV 引擎初始化之后，因为历史命令最终要重新写进内存数据结构。

同时，重放必须发生在以追加模式打开 AOF 之前。

在只实现 AOF 的阶段，启动顺序是：

```text
初始化 KV 引擎
→ 进入 AOF 重放状态
→ 读取并执行历史 AOF
→ 退出 AOF 重放状态
→ 以追加模式打开 AOF
→ 启动网络服务
```

`aof_replaying` 的含义是：

```text
0 → 正常运行，可以记录新的写命令
1 → 正在恢复历史数据，不能写入 AOF
```

注意，它表示“正在重放”，不是“正在重写”。

它只能在函数外定义一次：

```c
static int aof_replaying = 0;
```

如果在 `kvs_filter_protocol()` 内再次定义同名变量，就会发生变量遮蔽。这样主函数修改的是外层变量，而协议处理函数读取的是内部变量，二者不是同一个变量。

## 9. AOF 重放函数的实际过程

`kvs_aof_replay()` 使用只读模式打开 AOF：

```c
FILE *fp = fopen(path, "r");
```

如果文件不存在，说明当前没有历史数据，可以直接返回成功：

```c
if (errno == ENOENT) {
    return 0;
}
```

之后使用 `getline()` 每次读取一行：

```c
while ((line_length = getline(&line, &capacity, fp)) != -1) {
```

假设读取到：

```text
SET persist_key persist_value
```

之后，通过传入的回调函数执行它：

```c
char response[1024] = {0};

int response_length = handler(
    line,
    (int)line_length,
    response
);
```

因为调用 `kvs_aof_replay()` 时传入的是：

```c
kvs_protocol
```

所以这次调用实际相当于：

```c
kvs_protocol(line, (int)line_length, response);
```

历史命令因此重新进入原来的协议层和业务层，不需要为重放重新写一套 `SET`、`MOD`、`DEL` 逻辑。

当返回值有效，并且响应为：

```text
OK\r\n
```

说明这一条历史命令重放成功。否则重放失败，服务不继续启动。

## 10. AOF 基础闭环

AOF 阶段已经测试：

```text
SET persist_key persist_value
MOD persist_key new_value
```

服务重启以后执行：

```text
GET persist_key
```

能够得到：

```text
new_value
```

同时，重放期间不会把历史命令再次追加到 AOF，所以文件不会随着每次重启产生重复日志。

至此，AOF 的基础闭环完成。但是如果 AOF 中存在大量历史命令，每次重启都从第一条开始执行，恢复时间会越来越长。所以接下来需要实现全量快照。

## 11. 开始实现全量快照

AOF 保存的是每一次成功执行的写命令，而全量快照保存的是某一个时刻内存中的完整数据集。

例如执行过：

```text
SET name Jasper
MOD name Sao
```

AOF 中保存的是操作过程：

```text
SET name Jasper
MOD name Sao
```

而快照只关心最终数据状态：

```text
SET name Sao
```

因此，保存快照时不能简单复制 AOF。我们需要遍历当前内存中的数据，把每一个有效的 `key-value` 写进快照文件。

项目中存在多个 KV 引擎：

```text
array
rbtree
hash
skiplist
```

每一种数据结构的遍历方式不同：

- 数组需要遍历每一个槽位。
- 红黑树需要按照节点关系递归遍历。
- 哈希表需要遍历桶以及桶中的链表。
- 跳表需要沿着最底层链表遍历。

但是持久化模块不应该知道这些数据结构的内部实现。

所以采用统一的设计：

```text
引擎负责找到 key 和 value
→ 持久化模块负责把 key 和 value 写入文件
```

## 12. 定义统一的访问函数

在 `kvstore.h` 中定义访问函数类型：

```c
typedef int (*kvs_visit_handler)(
    const char *key,
    const char *value,
    void *context
);
```

这个函数指针表示：

```text
引擎每找到一组 key-value
→ 就调用一次 visitor
→ 把 key、value 和 context 交给 visitor
```

三个参数分别表示：

```text
key      当前遍历到的键
value    当前遍历到的值
context  调用者额外传入的信息
```

每一个引擎分别提供自己的遍历函数：

```c
int kvs_array_foreach(
    kvs_array_t *inst,
    kvs_visit_handler visitor,
    void *context
);

int kvs_rbtree_foreach(
    kvs_rbtree_t *inst,
    kvs_visit_handler visitor,
    void *context
);

int kvs_hash_foreach(
    kvs_hash_t *inst,
    kvs_visit_handler visitor,
    void *context
);

int kvs_skiplist_foreach(
    kvs_skiplist_t *inst,
    kvs_visit_handler visitor,
    void *context
);
```

虽然四个引擎内部的遍历方法不同，但是对外接口的形式相同。

## 13. foreach 的职责

以数组引擎为例：

```c
int kvs_array_foreach(
    kvs_array_t *inst,
    kvs_visit_handler visitor,
    void *context)
{
    if (inst == NULL ||
        inst->table == NULL ||
        visitor == NULL) {
        return -1;
    }

    for (int i = 0; i < inst->total; i++) {
        if (inst->table[i].key == NULL ||
            inst->table[i].value == NULL) {
            continue;
        }

        int ret = visitor(
            inst->table[i].key,
            inst->table[i].value,
            context
        );

        if (ret < 0) {
            return -1;
        }
    }

    return 0;
}
```

每一次循环找到一组有效的 `key-value`，就调用一次：

```c
visitor(key, value, context);
```

所以遍历函数本身没有耦合具体的文件写入逻辑。它只负责：

```text
找到数据
→ 把数据交给 visitor
```

真正决定如何使用这些数据的是传进来的 `visitor`。

## 14. context 的意义

快照写入函数不只需要 `key` 和 `value`，它还需要知道：

- 应该写入哪个文件。
- 当前数据来自哪个引擎。
- 应该使用 `SET`、`RSET`、`HSET` 还是 `SSET`。

所以定义结构体：

```c
typedef struct snapshot_write_context {
    FILE *fp;
    const char *command;
} snapshot_write_context_t;
```

这里：

```text
fp       快照文件的文件指针
command  当前引擎对应的写命令
```

然后把这个结构体的地址作为 `void *context` 传入遍历函数：

```c
snapshot_write_context_t ctx;

ctx.fp = fp;
ctx.command = "SET";

kvs_array_foreach(
    &global_array,
    snapshot_write_item,
    &ctx
);
```

此时 `&ctx` 被转换成了 `void *`。

进入 `snapshot_write_item()` 以后，再把它转换回来：

```c
snapshot_write_context_t *ctx =
    (snapshot_write_context_t *)context;
```

所以 `context` 并不是“一个数”。它是一个通用指针，这里保存的是 `ctx` 这个结构体的地址。

## 15. 把一组 key-value 写进快照

真正负责写入文件的函数是：

```c
static int snapshot_write_item(
    const char *key,
    const char *value,
    void *context)
{
    if (key == NULL ||
        value == NULL ||
        context == NULL) {
        return -1;
    }

    snapshot_write_context_t *ctx =
        (snapshot_write_context_t *)context;

    if (ctx->fp == NULL ||
        ctx->command == NULL) {
        return -1;
    }

    int written = fprintf(
        ctx->fp,
        "%s %s %s\n",
        ctx->command,
        key,
        value
    );

    if (written < 0) {
        return -1;
    }

    return 0;
}
```

例如数组引擎中存在：

```text
name → Jasper
age  → 20
```

调用两次 `snapshot_write_item()` 后，快照中会得到：

```text
SET name Jasper
SET age 20
```

也就是说，每遍历到一个有效数据，就执行一次 `snapshot_write_item()`。

## 16. 为什么快照中只保存 SET 类型的命令

快照保存的是当前最终数据集，而不是数据变化的过程。

例如：

```text
SET name Jasper
MOD name Sao
DEL age
```

最终只剩下：

```text
name → Sao
```

所以快照只需要保存一条能够重新创建当前数据的命令：

```text
SET name Sao
```

对于不同引擎，分别使用：

```text
SET   数组引擎
RSET  红黑树引擎
HSET  哈希引擎
SSET  跳表引擎
```

快照不需要保存 `MOD` 和 `DEL`。因为已经被删除的数据不会出现在遍历结果中，而被修改的数据会直接以最终值保存。

## 17. 保存所有引擎的数据

在 `snapshot.c` 中访问四个全局引擎：

```c
#if ENABLE_ARRAY
extern kvs_array_t global_array;
#endif

#if ENABLE_RBTREE
extern kvs_rbtree_t global_rbtree;
#endif

#if ENABLE_HASH
extern kvs_hash_t global_hash;
#endif

#if ENABLE_SKIPLIST
extern kvs_skiplist_t global_skiplist;
#endif
```

然后依次遍历已经启用的引擎：

```c
#if ENABLE_ARRAY
if (result == 0) {
    ctx.command = "SET";

    if (kvs_array_foreach(
            &global_array,
            snapshot_write_item,
            &ctx) < 0) {
        result = -1;
    }
}
#endif
```

其他引擎对应的命令为：

```text
红黑树  ctx.command = "RSET"
哈希表  ctx.command = "HSET"
跳表    ctx.command = "SSET"
```

`result` 用于记录整个快照保存过程是否成功：

```text
0   到目前为止没有失败
-1  某一步已经失败
```

如果前一个引擎保存失败，后面的引擎就不再继续写。这样可以避免生成一个看起来正常、实际上缺少部分数据的快照。

## 18. 使用临时文件保存快照

不能直接使用：

```c
fopen(path, "w");
```

覆盖原来的正式快照。因为如果写到一半时程序崩溃，旧快照已经被破坏，新快照又没有写完整。

所以先生成临时文件路径：

```c
char temp_path[1024];

snprintf(
    temp_path,
    sizeof(temp_path),
    "%s.tmp",
    path
);
```

例如：

```text
正式文件：snapshot.db
临时文件：snapshot.db.tmp
```

这里使用 `snprintf()`，是因为它可以接收缓冲区大小，避免写出的字符串超过 `temp_path` 的容量。

然后使用 `"w"` 模式打开临时文件：

```c
FILE *fp = fopen(temp_path, "w");
```

`"w"` 表示重新写入。临时文件如果已经存在，原来的内容会被清空。

## 19. 把快照同步到磁盘

所有引擎遍历完成以后，依次执行：

```text
fflush(fp)
→ fsync(fileno(fp))
→ fclose(fp)
```

含义与 AOF 相同：

```text
fflush   把 C 标准库缓冲区交给操作系统
fsync    要求操作系统把内容同步到底层存储
fclose   关闭文件
```

如果任何一步失败：

```c
unlink(temp_path);
return -1;
```

`unlink(temp_path)` 的意思是删除失败的临时文件，不是关闭文件。文件已经由 `fclose()` 负责关闭。

全部成功以后执行：

```c
rename(temp_path, path);
```

这一步把：

```text
snapshot.db.tmp
```

替换成：

```text
snapshot.db
```

因此，快照保存过程是：

```text
写临时文件
→ 确认完整写入
→ 同步到磁盘
→ 关闭临时文件
→ 用临时文件替换正式快照
```

这样可以尽量避免生成半份快照。

## 20. 增加 SAVE 命令

在持久化头文件中暴露快照保存接口：

```c
int kvs_snapshot_save(const char *path);
```

协议层切分完 tokens 后，单独判断：

```c
if (count == 1 && strcmp(tokens[0], "SAVE") == 0) {
    int save_result = kvs_snapshot_save("snapshot.db");

    if (save_result < 0) {
        return sprintf(response, "ERROR\r\n");
    }

    return sprintf(response, "OK\r\n");
}
```

用户发送：

```text
SAVE
```

程序就会把当前四个引擎中的完整数据保存到 `snapshot.db`。

`SAVE` 本身不是修改 KV 数据的普通业务命令，所以不需要写进 AOF。

## 21. 快照与 AOF 的重复问题

如果快照已经保存了当前全部数据，而服务启动时又从 AOF 开头执行所有命令，就会出现重复恢复。

例如 AOF 中有：

```text
SET name Jasper
MOD name Sao
```

快照中已经有：

```text
SET name Sao
```

启动时先读取快照，内存里已经存在 `name`。如果又从 AOF 开头执行 `SET name Jasper`，就可能因为 key 已经存在而失败。

所以快照必须知道：

> 保存这份快照时，AOF 已经写到了哪个位置。

## 22. 获取 AOF 当前偏移量

在 AOF 模块中增加：

```c
long long kvs_aof_get_offset(void);
```

实现过程：

```c
long long kvs_aof_get_offset(void)
{
    if (aof_fp == NULL) {
        return -1;
    }

    off_t offset = ftello(aof_fp);

    if (offset == (off_t)-1) {
        return -1;
    }

    return (long long)offset;
}
```

`off_t` 是系统用来表示文件位置和文件大小的类型。

`ftello()` 返回当前文件位置，也就是 AOF 已经写了多少字节。

假设返回：

```text
189
```

表示保存快照时，AOF 的前 `189` 个字节已经被这份快照包含。

## 23. 把 AOF 偏移量写进快照

保存快照之前先获取：

```c
long long aof_offset = kvs_aof_get_offset();
```

然后把它写在快照第一行：

```c
fprintf(fp, "AOF_OFFSET %lld\n", aof_offset);
```

最终快照类似：

```text
AOF_OFFSET 189
SET persist_key new_value
SET snapshot_array value_array
RSET snapshot_rbtree value_rbtree
HSET snapshot_hash value_hash
SSET snapshot_skiplist value_skiplist
```

第一行是元数据，不是业务命令。后面的每一行才是用来恢复数据的命令。

## 24. 加载全量快照

在持久化头文件中增加：

```c
long long kvs_snapshot_load(
    const char *path,
    aof_replay_handler handler
);
```

这个函数做两件事：

```text
1. 读取并执行快照中的数据命令
2. 返回快照记录的 AOF_OFFSET
```

使用只读方式打开：

```c
FILE *fp = fopen(path, "r");
```

如果快照文件不存在，说明以前没有保存过快照：

```c
if (errno == ENOENT) {
    return 0;
}
```

返回 `0` 表示：

```text
没有快照数据
→ AOF 应该从第 0 个字节开始重放
```

## 25. 读取快照中的偏移量

首先使用 `getline()` 读取第一行，然后解析：

```c
long long aof_offset = -1;

int parsed = sscanf(
    line,
    "AOF_OFFSET %lld",
    &aof_offset
);
```

这里：

```text
parsed      成功解析出的字段数量
aof_offset  真正保存偏移量的变量
```

如果第一行是：

```text
AOF_OFFSET 189
```

那么：

```text
parsed = 1
aof_offset = 189
```

如果没有成功读取一个偏移量，或者偏移量小于 `0`，就说明快照格式错误。

## 26. 逐行恢复快照数据

读取完第一行以后，继续使用 `getline()` 循环读取剩余内容：

```c
while ((line_length = getline(&line, &capacity, fp)) != -1) {
```

`getline()` 每执行一次，就从文件当前位置读取一行。

读取完成以后，去掉末尾的换行符：

```c
while (line_length > 0 &&
       (line[line_length - 1] == '\n' ||
        line[line_length - 1] == '\r')) {
    line[--line_length] = '\0';
}
```

然后准备一个响应缓冲区：

```c
char response[1024] = {0};
```

把快照命令交给协议函数：

```c
int response_length = handler(
    line,
    (int)line_length,
    response
);
```

因为传进来的 `handler` 是 `kvs_protocol`，所以实际效果相当于：

```c
kvs_protocol(line, (int)line_length, response);
```

例如快照中的：

```text
HSET snapshot_hash value_hash
```

会重新进入原来的协议解析和哈希引擎处理流程。

`response` 不会发送给网络客户端，它只用于检查这条恢复命令是否返回：

```text
OK\r\n
```

所有快照命令恢复成功后，函数返回：

```c
return aof_offset;
```

## 27. 让 AOF 从指定位置开始重放

AOF 重放函数的接口改成：

```c
int kvs_aof_replay(
    const char *path,
    long long offset,
    aof_replay_handler handler
);
```

打开 AOF 后，先执行：

```c
fseeko(fp, (off_t)offset, SEEK_SET);
```

三个参数表示：

```text
fp        当前 AOF 文件
offset    要跳转到的字节位置
SEEK_SET  从文件开头计算位置
```

例如：

```text
offset = 189
```

就表示跳过 AOF 前 `189` 个字节，只重放后面的增量命令。

这里并不是逐条判断哪些命令应该跳过，而是直接移动文件的读取位置。

## 28. 最终启动恢复顺序

最终启动过程变成：

```text
初始化所有 KV 引擎
→ 进入恢复状态
→ 加载 snapshot.db
→ 得到 AOF_OFFSET
→ 从该偏移量开始重放 appendonly.aof
→ 退出恢复状态
→ 以追加模式打开 AOF
→ 启动网络服务
```

代码大致为：

```c
init_kvengine();

aof_replaying = 1;

long long aof_offset = kvs_snapshot_load(
    "snapshot.db",
    kvs_protocol
);

if (aof_offset < 0) {
    aof_replaying = 0;
    fprintf(stderr, "failed to load snapshot\n");
    return -1;
}

int replay_ret = kvs_aof_replay(
    "appendonly.aof",
    aof_offset,
    kvs_protocol
);

aof_replaying = 0;

if (replay_ret < 0) {
    fprintf(stderr, "failed to replay AOF\n");
    return -1;
}

if (kvs_aof_open("appendonly.aof") < 0) {
    fprintf(stderr, "failed to open AOF\n");
    return -1;
}
```

这里的 `aof_replaying` 虽然叫做 AOF 重放状态，但是现在它实际上覆盖了整个数据恢复阶段：

```text
加载快照期间     不允许写入 AOF
重放 AOF 期间    不允许写入 AOF
```

因为快照和 AOF 中的历史命令都会调用 `kvs_protocol()`。

如果没有这个标志，恢复出来的历史命令又会被当成新的写命令追加到 AOF。

## 29. 快照与 AOF 最终如何配合

可以把二者理解成：

```text
snapshot.db    保存稳定的历史基础
appendonly.aof 保存快照之后发生的新变化
```

假设：

```text
AOF 前 189 字节
→ 已经包含在 snapshot.db 中

AOF 第 189 字节以后
→ 是保存快照之后执行的新命令
```

恢复时：

```text
snapshot.db
→ 恢复保存快照时的完整数据

AOF_OFFSET 之后的 AOF
→ 补上保存快照以后发生的变化
```

因此最后恢复出的内存状态是：

```text
快照中的完整数据
+
快照之后的增量命令
=
服务退出前的最新数据
```

## 30. 最终验证

保存快照时的数据为：

```text
SET persist_key new_value
SET snapshot_array value_array
RSET snapshot_rbtree value_rbtree
HSET snapshot_hash value_hash
SSET snapshot_skiplist value_skiplist
```

执行：

```text
SAVE
```

快照记录：

```text
AOF_OFFSET 189
```

保存完成以后，再执行：

```text
SET after_snapshot from_aof
```

这条命令没有进入快照，只进入了 AOF 偏移量之后的增量部分。

服务重启以后，输出顺序为：

```text
先执行快照中的五条命令
→ 再执行 SET after_snapshot from_aof
→ 最后开始监听端口
```

客户端执行：

```text
GET after_snapshot
```

能够得到：

```text
from_aof
```

这说明：

```text
全量快照恢复成功
AOF 增量恢复成功
偏移量跳过成功
恢复期间没有重复写入 AOF
```

## 31. 当前完成的整体闭环

目前已经完成的混合持久化流程是：

```text
正常运行
→ 成功的写命令立即追加到 AOF
→ 用户执行 SAVE
→ 遍历所有引擎
→ 保存当前完整数据集
→ 在快照中记录 AOF_OFFSET
→ 快照保存后继续记录新的 AOF
→ 服务退出或者崩溃
→ 下次启动先加载快照
→ 再重放 AOF_OFFSET 之后的日志
→ 恢复出最新内存数据
→ 开始对外提供服务
```

我的理解是：

```text
AOF 负责尽量不丢掉每一次数据变化

快照负责保存某一时刻完整的数据集，
并缩短启动时需要处理的历史范围

AOF_OFFSET 负责把快照和 AOF 连接起来

aof_replaying 负责避免恢复过程再次产生日志

handler 负责让历史命令复用原来的业务处理流程

foreach 负责遍历不同引擎中的有效数据

visitor 负责处理遍历得到的每一组 key-value

context 负责向 visitor 传递文件指针和命令类型
```

至此，全量快照和 AOF 增量日志已经组成了一个完整的混合持久化闭环。
