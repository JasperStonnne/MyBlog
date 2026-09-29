---
title: KV 进化（五）：Primary / Replica 主从同步
slug: kv-evolution-primary-replica-sync
description: 在单机 KV、AOF 与 Snapshot 基础上实现 Primary / Replica 主从复制，梳理写命令转发、只读副本与同步状态管理
date: 2026-09-29T00:00:00+08:00
draft: false
image: cover.svg
tags:
  - KV 存储
  - 主从同步
  - Primary Replica
  - 复制
  - C 语言
categories:
  - 项目记录
---

实现主从同步功能

当前的 91kvstore 是一个单点 KV 服务：

```text
Client
  ↓
kvstore
  ↓
Array / RBTree / Hash / SkipList
```

所有 `SET`、`GET`、`DEL`、`MOD` 等操作都发生在同一个进程中。一旦这个进程退出或者所在机器发生故障，虽然可以通过 Snapshot 和 AOF 恢复本机数据，但服务在恢复期间不可用，也没有另一台机器可以继续提供数据副本。

这一轮准备在现有单机 KV、AOF 和 Snapshot 的基础上增加标准主从复制：

```text
客户端写入
    ↓
Primary
    ↓ 复制写命令
Replica
```

Primary 负责接受写请求并确定写命令顺序，Replica 负责复制 Primary 的数据并提供只读查询。

## Primary / Replica 与 Master / Slave

两组术语表示相同角色：

```text
Master  = Primary = 主库
Slave   = Replica = 从库
```

这一轮代码统一采用：

```text
Primary
Replica
```

Primary 表示主要写入节点，Replica 表示复制出来的数据副本节点。旧文章和旧项目仍可能使用 Master / Slave，但在本项目中不再混用两套名称。

标准主从的基本规则是：

```text
Primary：允许客户端读写
Replica：允许客户端读取，拒绝客户端直接写入
```

例如客户端向 Primary 写入：

```text
SET name jasper
```

Primary 执行成功后，把相同的写命令发送给 Replica。Replica 在自己的存储引擎中再次执行：

```text
SET name jasper
```

最终两台机器都保存：

```text
name = jasper
```

如果普通客户端直接向 Replica 发送写命令，后续应返回：

```text
READONLY
```

但来自 Primary 的复制命令必须允许执行。因此后续不能只看命令是不是 `SET`，还需要判断命令来源。

## 为什么选择标准主从而不是双主

双主意味着两台机器都可以接受写入：

```text
Client A → Primary A ⇄ Primary B ← Client B
```

如果两边同时写入同一个 key：

```text
A：SET name alice
B：SET name bob
```

系统必须决定最终保存哪个值，还要处理：

```text
写冲突
命令重复传播
同步环路
节点版本
逻辑时钟
网络分区
冲突合并
```

这已经不是把 AOF 双向发送那么简单，而是另一套分布式一致性设计。

当前 91kvstore 已经具备 AOF、Snapshot、TCP 协议和多种存储引擎，与标准主从的匹配程度更高。Primary 可以统一确定写命令顺序，Replica 只需要按相同顺序执行，不需要解决两个写入源之间的冲突。

因此本轮确定的范围是：

```text
实现：一主一从、全量同步、增量同步、断线恢复、只读 Replica
不实现：双主、Raft、自动选主、多数据中心冲突合并
```

这不是只转发一条 `SET` 的演示功能，也不会直接扩张成完整分布式集群。目标是实现一个范围清楚、流程完整、能够正确处理同步边界的主从复制模块。

## Snapshot 与 AOF 在复制中的职责

主从复制不是只发送当前内存，也不是每次都重新执行完整 AOF，而是组合使用 Snapshot 和 AOF。

两者的职责可以概括为：

```text
Snapshot：某个时间点的完整数据状态
AOF：这个时间点之后发生的数据变化
```

例如同一个 key 被修改五次：

```text
SET name a
MOD name b
MOD name c
MOD name d
MOD name jasper
```

AOF 保存数据变化过程，恢复时需要依次重放这些命令。Snapshot 只保存最终状态：

```text
SET name jasper
```

因此 Replica 第一次连接 Primary 时，计划采用：

```text
1. Replica 请求全量同步
2. Primary 自动生成用于复制的临时 Snapshot
3. Primary 记录 Snapshot 对应的 AOF offset
4. Primary 发送 Snapshot
5. Replica 加载 Snapshot
6. Primary 发送该 offset 之后的 AOF
7. Replica 追上 Primary 后进入在线增量复制
```

这里不依赖用户提前手动执行 `SAVE`。手动 `SAVE` 是用户主动保存持久化快照；首次主从同步需要的 Snapshot 将由复制系统自动触发。

当前 `snapshot.db` 第一行已经记录：

```text
AOF_OFFSET <offset>
```

这为 Snapshot 和后续 AOF 的衔接提供了基础。

## replication offset

Replication offset 表示 Replica 已经处理到复制流的哪个位置，可以理解为复制进度书签。

例如：

```text
Primary offset  = 1200
Replica offset  = 1000
```

说明 Replica 还缺少 200 字节的复制数据。

断线重连后，Replica 可以告诉 Primary：

```text
我已经处理到 offset 1000
```

Primary 只需要从 1000 之后继续发送，而不必每次重新传输所有数据。

需要区分两个概念：

```text
AOF offset          // 本地持久化日志位置
Replication offset  // 主从复制进度
```

第一阶段可以让复制流直接复用 AOF 的字节位置，但代码设计中仍应保留概念边界，避免以后调整复制日志时把本地持久化和主从进度永久绑定在一起。

## 全量同步期间的新写入

生成和发送 Snapshot 不是瞬间完成的。在这个过程中 Primary 仍可能收到新写命令。

锁和状态机解决的是两个不同问题：

```text
锁：保证 Snapshot 与 offset 对应同一个一致性时刻
状态机：记录 Replica 当前同步到了哪个阶段
```

计划中的一致性流程是：

```text
短暂锁定写入
    ↓
确定 offset 并生成一致性 Snapshot
    ↓
解除锁定，Primary 继续处理写入
    ↓
通过网络发送 Snapshot
    ↓
发送 Snapshot offset 之后新增的 AOF
```

锁不能覆盖整个网络传输过程。如果发送一个大 Snapshot 时一直持有数据库锁，Primary 会长时间无法写入。

Replica 的连接状态计划表示为：

```text
DISCONNECTED
    ↓
CONNECTING
    ↓
HANDSHAKE
    ↓
FULL_SYNC 或 PARTIAL_SYNC
    ↓
CATCHING_UP
    ↓
ONLINE
```

连接断开后进入重试流程：

```text
ONLINE → BACKOFF → CONNECTING
```

状态机与 Snapshot、增量 AOF、ACK 和重连将在后续阶段逐步实现。

## 运行方式设计

为了保留原来的单机用法，同时支持主从复制，最终确定三种运行角色：

```text
STANDALONE
PRIMARY
REPLICA
```

原来的启动方式保持不变：

```bash
./bin/kvstore 2000
```

Primary 使用业务端口和复制端口：

```bash
./bin/kvstore primary 2000 2100
```

含义：

```text
2000：普通客户端连接的业务端口
2100：Replica 连接的复制端口
```

Replica 使用自己的业务端口，并主动连接 Primary 的复制端口：

```bash
./bin/kvstore replica 2001 127.0.0.1 2100
```

含义：

```text
2001：Replica 自己的只读业务端口
127.0.0.1：Primary 地址
2100：Primary 的复制端口
```

Primary 使用两个端口，是为了让客户端协议与复制协议保持清晰边界：

```text
客户端命令 → service_port
复制握手、Snapshot、AOF、ACK → replication_port
```

## 创建功能分支

主从同步在独立功能分支开发：

```bash
git switch -c feature/primary-replica-replication
```

这一分支已经分别推送到：

```text
GitHub：JasperStonnne/91kvstore
GitLab：Shijingyuan/91kvstore
```

GitHub 被设置为默认 upstream，GitLab 使用显式远程名称同步同一功能分支。

## 定义运行角色

新增公共头文件：

```text
include/replication.h
```

首先定义运行角色：

```c
#ifndef KVS_REPLICATION_H
#define KVS_REPLICATION_H

typedef enum {
    KVS_ROLE_STANDALONE = 0,
    KVS_ROLE_PRIMARY,
    KVS_ROLE_REPLICA
} kvs_role_t;

#endif
```

枚举实际对应：

```text
KVS_ROLE_STANDALONE = 0
KVS_ROLE_PRIMARY    = 1
KVS_ROLE_REPLICA    = 2
```

使用枚举而不是散落的整数或字符串，可以让后续判断更明确：

```c
if (role == KVS_ROLE_REPLICA) {
    // 检查客户端写入
}
```

## 集中保存启动配置

同一个角色还需要携带端口和地址，因此在 `replication.h` 中增加：

```c
typedef struct {                                      // 保存启动配置
    kvs_role_t role;                                  // 当前进程角色
    unsigned short service_port;                      // 客户端连接端口
    unsigned short replication_port;                  // 主从复制连接端口
    const char *primary_host;                         // Replica 要连接的 Primary 地址
} kvs_server_config_t;
```

Primary 的配置示例：

```text
role             = KVS_ROLE_PRIMARY
service_port     = 2000
replication_port = 2100
primary_host     = NULL
```

Replica 的配置示例：

```text
role             = KVS_ROLE_REPLICA
service_port     = 2001
replication_port = 2100
primary_host     = "127.0.0.1"
```

`primary_host` 直接指向 `argv` 中的字符串。命令行参数在整个 `main()` 生命周期内都有效，因此当前阶段不需要额外复制这段字符串。

## 补充 Makefile 依赖

`src/kvstore.c` 引入：

```c
#include "replication.h"
```

因此 `kvstore.o` 的构建依赖也必须包含这个头文件：

```make
$(BUILD_DIR)/kvstore.o: \
	$(SRC_DIR)/kvstore.c \
	$(INCLUDE_DIR)/kvstore.h \
	$(INCLUDE_DIR)/replication.h | $(BUILD_DIR)
	$(CC) $(CPPFLAGS) $(CFLAGS) -c $< -o $@
```

如果不声明依赖，只修改 `replication.h` 后执行 `make`，构建系统可能认为 `kvstore.o` 不需要重新编译。

## 规范解析端口

旧代码直接使用：

```c
int port = atoi(argv[1]);
```

`atoi()` 无法区分：

```text
"abc"
"0"
```

它们都可能得到 `0`，而且不能可靠检查数值溢出。主从模式需要解析多个端口，因此先增加统一端口解析函数：

```c
static int kvs_parse_port(const char *text, unsigned short *port) {
    if (text == NULL || port == NULL) {
        return -1;
    }

    errno = 0;
    char *end = NULL;                            // 字符串解析结束的位置
    long value = strtol(text, &end, 10);         // 按十进制转换为 long

    if (errno != 0 ||                            // 转换发生错误
        end == text ||                           // 没有解析出任何数字
        *end != '\0' ||                          // 数字后仍有非法字符
        value < 1 ||                             // 端口不能小于 1
        value > 65535) {                         // 端口不能超过 65535
        return -1;
    }

    *port = (unsigned short)value;
    return 0;
}
```

`strtol()` 通过 `end` 返回停止解析的位置。

输入完整数字：

```text
text → 2000\0
            ↑ end
```

此时：

```c
*end == '\0'
```

输入数字后带非法字符：

```text
text → 2000abc
            ↑ end
```

此时：

```c
*end != '\0'
```

输入完全不是数字：

```text
text → abc
       ↑ text 与 end
```

此时：

```c
end == text
```

也就是一个数字字符都没有成功读取。

## 解析三种启动参数

`main()` 先创建默认配置：

```c
kvs_server_config_t config = {
    .role = KVS_ROLE_STANDALONE,
    .service_port = 0,
    .replication_port = 0,
    .primary_host = NULL
};
```

### 单机模式

```c
if (argc == 2) {
    if (kvs_parse_port(argv[1], &config.service_port) < 0) {
        fprintf(stderr, "invalid service port: %s\n", argv[1]);
        return -1;
    }
}
```

执行：

```bash
./bin/kvstore 2000
```

得到：

```text
role             = STANDALONE
service_port     = 2000
replication_port = 0
primary_host     = NULL
```

### Primary 模式

```c
else if (argc == 4 && strcmp(argv[1], "primary") == 0) {
    config.role = KVS_ROLE_PRIMARY;

    if (kvs_parse_port(argv[2], &config.service_port) < 0) {
        fprintf(stderr, "invalid service port: %s\n", argv[2]);
        return -1;
    }

    if (kvs_parse_port(argv[3], &config.replication_port) < 0) {
        fprintf(stderr, "invalid replication port: %s\n", argv[3]);
        return -1;
    }

    if (config.service_port == config.replication_port) {
        fprintf(stderr,
                "service port and replication port must differ\n");
        return -1;
    }
}
```

执行：

```bash
./bin/kvstore primary 2000 2100
```

得到：

```text
role             = PRIMARY
service_port     = 2000
replication_port = 2100
primary_host     = NULL
```

业务端口和复制端口不能相同，否则同一个进程中的两个监听 socket 无法同时绑定相同地址和端口。

### Replica 模式

```c
else if (argc == 5 && strcmp(argv[1], "replica") == 0) {
    config.role = KVS_ROLE_REPLICA;

    if (kvs_parse_port(argv[2], &config.service_port) < 0) {
        fprintf(stderr, "invalid service port: %s\n", argv[2]);
        return -1;
    }

    if (argv[3][0] == '\0') {
        fprintf(stderr, "primary host cannot be empty\n");
        return -1;
    }

    config.primary_host = argv[3];

    if (kvs_parse_port(argv[4], &config.replication_port) < 0) {
        fprintf(stderr, "invalid replication port: %s\n", argv[4]);
        return -1;
    }
}
```

执行：

```bash
./bin/kvstore replica 2001 127.0.0.1 2100
```

得到：

```text
role             = REPLICA
service_port     = 2001
primary_host     = "127.0.0.1"
replication_port = 2100
```

### 非法启动参数

不满足三种形式时，程序打印：

```c
fprintf(stderr,
        "usage:\n"
        "  %s <service-port>\n"
        "  %s primary <service-port> <replication-port>\n"
        "  %s replica <service-port> <primary-host> <replication-port>\n",
        argv[0],
        argv[0],
        argv[0]);
```

C 会自动拼接相邻字符串。三个 `%s` 分别使用三个 `argv[0]`，最终显示实际程序名：

```text
usage:
  ./bin/kvstore <service-port>
  ./bin/kvstore primary <service-port> <replication-port>
  ./bin/kvstore replica <service-port> <primary-host> <replication-port>
```

## 业务网络层改用配置端口

原来三个网络模型直接使用局部变量 `port`：

```c
reactor_start(port, kvs_network_protocol);
ntyco_start(port, kvs_network_protocol);
proactor_start(port, kvs_network_protocol);
```

改为统一使用：

```c
reactor_start(config.service_port, kvs_network_protocol);
ntyco_start(config.service_port, kvs_network_protocol);
proactor_start(config.service_port, kvs_network_protocol);
```

当前这三个调用仍然只负责业务端口。Primary 的复制端口监听和 Replica 的主动连接尚未接入。

## 启动参数验证

无参数启动：

```bash
./bin/kvstore
```

输出：

```text
usage:
  ./bin/kvstore <service-port>
  ./bin/kvstore primary <service-port> <replication-port>
  ./bin/kvstore replica <service-port> <primary-host> <replication-port>
```

Primary 使用相同的业务端口和复制端口：

```bash
./bin/kvstore primary 2000 2000
```

输出：

```text
service port and replication port must differ
```

Replica 使用非法业务端口：

```bash
./bin/kvstore replica abc 127.0.0.1 2100
```

输出：

```text
invalid service port: abc
```

完整 `make` 构建通过，编译过程没有 error 和 warning。

## 当前提交

这一阶段使用提交标题：

```text
feat(replication): 增加主从角色与启动参数解析
```

提交内容包括：

```text
新增 standalone、primary 和 replica 三种运行角色
增加服务端启动配置结构体
支持三种启动参数格式
校验非法端口与 Primary 端口冲突
补充 replication.h 的 Makefile 构建依赖
```

功能分支已经同步到 GitHub 和 GitLab：

```text
feature/primary-replica-replication
```

## 命令来源设计

下一阶段首先要解决 Replica 的只读限制。

同样一条写命令：

```text
SET name jasper
```

需要根据来源决定是否允许执行：

```text
普通客户端 → Replica       // 拒绝
Snapshot / AOF → Replica   // 允许恢复
Primary → Replica          // 允许同步
```

因此准备在 `kvstore.h` 中定义：

```c
typedef enum {
    KVS_COMMAND_SOURCE_CLIENT = 0,       // 普通客户端发送的命令
    KVS_COMMAND_SOURCE_RECOVERY,         // Snapshot 或 AOF 恢复命令
    KVS_COMMAND_SOURCE_REPLICATION       // Primary 同步给 Replica 的命令
} kvs_command_source_t;
```

这里使用 `RECOVERY`，而不是只叫 `AOF`，因为启动恢复流程同时包含：

```text
Snapshot 加载
    ↓
AOF 重放
```

两者都是内部恢复命令。

截至当前，这个命令来源枚举属于下一阶段设计，还没有接入命令执行流程。后续需要让协议层明确接收 source，不能简单复用现有的 `aof_replaying` 标志来冒充复制状态。

## 截至当前的完成情况

已经完成并提交：

```text
确定标准主从架构
明确 Primary / Replica 角色语义
明确 Snapshot + AOF 的全量同步方向
明确 replication offset 的用途
创建主从复制功能分支
新增 replication.h
定义 STANDALONE / PRIMARY / REPLICA
定义服务端启动配置
规范端口字符串解析
支持三种启动参数
完成非法参数验证
通过完整构建
提交并推送 GitHub、GitLab 功能分支
```

尚未实现：

```text
Replica 只读限制
命令来源接入协议执行流程
Primary 复制端口监听
Replica 主动连接 Primary
复制握手协议
Replication ID
自动生成和传输 Snapshot
Snapshot 后续 AOF 追赶
在线增量复制
ACK 与心跳
状态机
断线重连与部分同步
数据库锁和复制状态锁
ROLE 状态查询
主从集成测试
```

当前程序虽然能够识别：

```bash
./bin/kvstore primary 2000 2100
./bin/kvstore replica 2001 127.0.0.1 2100
```

但角色尚未改变后续运行路径。Primary 还没有监听 `replication_port`，Replica 也还没有连接 `primary_host:replication_port`。

## 下一阶段

下一阶段从命令来源开始，让角色第一次真正影响程序行为：

```text
识别客户端命令
识别启动恢复命令
识别主从复制命令
    ↓
Replica 拒绝客户端写命令
    ↓
Replica 允许恢复与复制写命令
```

完成只读边界后，再建立独立复制连接和握手协议。只有握手与传输边界确定后，才能安全加入自动 Snapshot 全量同步和后续 AOF 增量追赶。

---

## 第二阶段：让命令携带来源

第一阶段完成了角色和启动参数，但当时角色还只是配置：

```text
程序知道自己是 Primary 或 Replica
但命令执行层还不知道命令从哪里来
```

Replica 的“只读”不是任何时候都不能修改数据，而是不能被普通客户端直接修改。Replica 至少需要接受下面两类内部写入：

```text
Snapshot / AOF 恢复写入
Primary 发送的复制写入
```

所以，同样一条命令：

```text
HSET name jasper
```

可能有三种不同语义：

```text
CLIENT       // 普通客户端请求
RECOVERY     // Snapshot 或 AOF 恢复
REPLICATION  // Primary 同步给 Replica
```

这也是为什么只判断命令是不是写命令还不够。正确的权限判断至少需要同时知道：

```text
当前进程是什么角色
这一条命令来自哪里
这一条命令是读还是写
```

命令来源类型定义在 `include/kvstore.h`：

```c
typedef enum {
    KVS_COMMAND_SOURCE_CLIENT = 0,
    KVS_COMMAND_SOURCE_RECOVERY,
    KVS_COMMAND_SOURCE_REPLICATION
} kvs_command_source_t;
```

这个枚举不是状态机。它只描述一条命令的来源，不保存连接状态，也不负责状态迁移。

真正的复制状态机将在后面描述类似过程：

```text
DISCONNECTED
    ↓
CONNECTING
    ↓
HANDSHAKE
    ↓
FULL_SYNC / PARTIAL_SYNC
    ↓
ONLINE
```

## 为什么移除 aof_replaying

原来的启动恢复依赖一个文件级全局变量：

```c
static int aof_replaying = 0;
```

启动时先切换它：

```text
aof_replaying = 1
    ↓
加载 Snapshot
    ↓
重放 AOF
    ↓
aof_replaying = 0
```

命令执行成功后，原来的 AOF 判断是：

```c
if(!aof_replaying && ret == 0 && kvs_is_write_command(cmd)){
    kvs_aof_append(tokens,count);
}
```

这个变量能够回答：

```text
程序现在是不是处在启动恢复阶段？
```

但它不能准确回答：

```text
当前这一条命令究竟来自客户端、恢复流程还是复制连接？
```

主从复制加入后，程序可能同时处理不同来源的命令。用一个会切换的全局开关描述命令来源，不仅表达能力不足，也容易在并发执行时产生错误。

完成命令来源传递后，AOF 判断改为：

```c
if(ret == 0 &&
   kvs_is_write_command(cmd) &&
   source != KVS_COMMAND_SOURCE_RECOVERY){
    kvs_aof_append(tokens,count);
}
```

规则变成：

| 命令来源 | 是否执行 | 是否追加本机 AOF |
|---|---:|---:|
| CLIENT | 是 | 是 |
| RECOVERY | 是 | 否 |
| REPLICATION | 是 | 是 |

恢复命令不能重新追加 AOF，否则会出现：

```text
从 AOF 读取命令
    ↓
执行命令
    ↓
再次追加回 AOF
    ↓
AOF 在恢复过程中继续增长
```

复制命令则不同。Replica 接收 Primary 的写命令后，除了修改内存，通常还需要写入自己的 AOF，这样 Replica 重启后才能恢复已经同步的数据。

因此只排除 `RECOVERY`，没有排除 `REPLICATION`。

## 拆分统一执行入口与适配器

原来的 `kvs_protocol()` 同时承担单条命令解析和对外回调入口。加入命令来源后，如果直接给它增加第四个参数，会破坏现有三参数回调：

```text
Snapshot 加载回调
AOF 重放回调
批处理中的单命令调用
```

因此把真正的单命令解析与执行入口改为：

```c
static int kvs_execute_command(
    char *msg,
    int length,
    char *response,
    kvs_command_source_t source
);
```

然后分别增加两个三参数适配器。

客户端适配器：

```c
static int kvs_client_protocol(char *msg,int length,char *response){
    return kvs_execute_command(
        msg,
        length,
        response,
        KVS_COMMAND_SOURCE_CLIENT
    );
}
```

恢复适配器：

```c
static int kvs_recovery_protocol(char *msg,int length,char *response){
    return kvs_execute_command(
        msg,
        length,
        response,
        KVS_COMMAND_SOURCE_RECOVERY
    );
}
```

新的客户端调用链是：

```text
Reactor / NtyCo / Proactor
        ↓
kvs_network_protocol()
        ↓
kvs_batch_protocol()
        ↓
kvs_client_protocol()
        ↓ source = CLIENT
kvs_execute_command()
        ↓
kvs_filter_protocol()
        ↓
具体存储引擎 + AOF 判断
```

新的恢复调用链是：

```text
kvs_snapshot_load()
        └── kvs_recovery_protocol()

kvs_aof_replay()
        └── kvs_recovery_protocol()
                    ↓ source = RECOVERY
             kvs_execute_command()
                    ↓
             kvs_filter_protocol()
```

这样既保留了原有三参数回调接口，又把命令来源明确传入了统一执行流程。

迁移完成后，旧的 `aof_replaying` 定义和启动期间的所有赋值全部删除。Proactor 中已经失效的旧 `kvs_protocol` 声明也一并清理。

## 验证恢复不会重复写 AOF

只通过编译不能证明恢复逻辑正确，因此使用 SHA-256 比较启动前后的 AOF 内容。

第一次记录：

```bash
sha256sum appendonly.aof
```

得到：

```text
bd7fabf2bbcc594f77497733c6b94e67ae972606df977dfe1c61505c18250497  appendonly.aof
```

启动一次服务，让程序加载 Snapshot 并重放 AOF：

```bash
timeout 2s ./bin/kvstore 19000
```

重新计算 SHA-256，结果保持不变：

```text
bd7fabf2bbcc594f77497733c6b94e67ae972606df977dfe1c61505c18250497  appendonly.aof
```

这证明 `RECOVERY` 命令虽然被执行，但没有重新追加到 AOF。

随后通过 Packet Sender 发送一条客户端写命令：

```text
HSET source_test_20260917 client\r\n
```

返回：

```text
OK\r\n
```

AOF 的 SHA-256 变为：

```text
c8fbe5540cbcc88fd3fae8ae31fb2ce69d3fed6f62a06c76f324d8f0c935531b  appendonly.aof
```

说明 `CLIENT` 写命令仍会正常追加 AOF。

再次启动并恢复后，SHA-256 仍然是：

```text
c8fbe5540cbcc88fd3fae8ae31fb2ce69d3fed6f62a06c76f324d8f0c935531b  appendonly.aof
```

最终完成双向验证：

```text
RECOVERY → 执行命令，AOF 不变化
CLIENT   → 执行命令，AOF 发生变化
```

## 让命令执行层知道服务器角色

命令来源接入后，执行层还需要知道当前进程角色。

启动配置 `config` 是 `main()` 的局部变量，而 `kvs_filter_protocol()` 位于另一个函数中，不能直接读取这个局部变量。因此在 `src/kvstore.c` 内保存当前进程角色：

```c
static kvs_role_t kvs_server_role = KVS_ROLE_STANDALONE;
```

这里的 `STANDALONE` 只是默认初始值。参数解析结束后统一赋值：

```c
kvs_server_role = config.role;
```

三种启动方式分别得到：

```text
./bin/kvstore 2000
    → kvs_server_role = STANDALONE

./bin/kvstore primary 2000 2100
    → kvs_server_role = PRIMARY

./bin/kvstore replica 2001 127.0.0.1 2100
    → kvs_server_role = REPLICA
```

这个变量使用文件级 `static`，只在 `src/kvstore.c` 内可见。当前读取角色的 `kvs_filter_protocol()` 也在同一个文件中，因此不需要把角色变量暴露给其他模块。

它与被删除的 `aof_replaying` 本质不同：

```text
kvs_server_role：进程启动时确定，运行期间保持不变
aof_replaying：执行过程中切换，试图描述临时命令状态
```

服务器角色确实是进程级属性，而命令来源应该是每条命令自己的属性。

## Replica 客户端只读拦截

命令解析出 `cmd` 后、进入具体存储引擎之前，增加写权限检查：

```c
if(kvs_server_role == KVS_ROLE_REPLICA &&
   source == KVS_COMMAND_SOURCE_CLIENT &&
   kvs_is_write_command(cmd)){
    return sprintf(
        response,
        "READONLY replica does not accept client writes\r\n"
    );
}
```

这是一个执行前的守卫条件。只有下面三个条件同时成立才会拦截：

```text
当前角色是 Replica
命令来源是普通客户端
命令属于写命令
```

因为这里直接 `return`，被拒绝的命令不会进入下面的 `switch`，所以不会：

```text
修改 Replica 内存
追加 Replica AOF
```

最终权限规则是：

| 当前角色 | 命令来源 | 命令类型 | 结果 |
|---|---|---|---|
| Standalone | Client | 写 | 允许 |
| Primary | Client | 写 | 允许 |
| Replica | Client | 写 | 拒绝 |
| Replica | Client | 读 | 允许 |
| Replica | Recovery | 写 | 允许 |
| Replica | Replication | 写 | 允许 |

使用 Replica 模式启动：

```bash
./bin/kvstore replica 2001 127.0.0.1 2100
```

Packet Sender 向 Replica 的业务端口 `2001` 发送：

```text
HSET replica_write_test blocked\r\n
```

返回：

```text
READONLY replica does not accept client writes\r\n
```

向同一端口发送读命令仍能正常返回数据。随后用 Primary 模式启动，并向业务端口写入，写命令返回 `OK`。这证明只读判断没有误伤 Primary，也没有把 Replica 变成完全不可写的进程。

## 当前 Replica 为什么仍然像单机服务

虽然现在能够使用下面的命令启动 Replica：

```bash
./bin/kvstore replica 2001 127.0.0.1 2100
```

但当前真实执行路径仍然是：

```text
在 2001 启动原有客户端网络
加载本机 snapshot.db
重放本机 appendonly.aof
对客户端写命令增加 READONLY 拦截
```

启动参数中的：

```text
127.0.0.1
2100
```

目前只被保存到：

```c
config.primary_host
config.replication_port
```

还没有被用于地址解析、建立 socket 或调用 `connect()`。

同理，Primary 虽然能够使用：

```bash
./bin/kvstore primary 2000 2100
```

但当前只在 `2000` 上启动原有业务网络，尚未在 `2100` 上监听 Replica。

因此当前测试证明的是：

```text
角色识别正确
命令来源正确
Replica 客户端只读正确
```

它不能证明：

```text
Primary 和 Replica 已经建立连接
数据已经能够跨进程同步
```

Packet Sender 测试 Replica 时，目标地址应当是 Ubuntu 虚拟机的可访问地址，目标端口是 Replica 的业务端口 `2001`。启动参数中的 `primary_host:replication_port` 是未来 Replica 主动连接 Primary 时使用的另一条网络路径。

## 第二阶段提交

命令来源重构提交：

```text
9c2020d refactor(replication): 引入命令来源并区分客户端与恢复命令
```

内容包括：

```text
定义 CLIENT、RECOVERY、REPLICATION
拆分统一命令执行入口
增加客户端与恢复适配器
Snapshot 和 AOF 使用恢复来源
删除 aof_replaying
清理 Proactor 中失效的旧声明
```

Replica 只读提交：

```text
b51206e feat(replication): 拒绝 Replica 客户端写命令
```

内容包括：

```text
保存当前进程角色
执行前检查角色、来源和命令类型
拒绝 Replica 客户端写命令
允许 Replica 客户端读取与内部写入
```

两个阶段均通过完整 `make`，没有编译错误和警告。相关功能还分别通过 AOF 哈希、Replica 读写和 Primary 写入测试。

## 截至第二阶段的完成情况

目前已经完成：

```text
主从角色与启动参数
严格端口解析
命令来源模型
客户端与恢复执行入口
去除 aof_replaying 全局状态
Replica 客户端只读
恢复不重复追加 AOF 的哈希验证
Primary 与 Replica 权限行为测试
阶段性 Git 提交
```

如果按照预定的完整主从范围估算，目前整体约完成 20%。完成的是角色、命令来源和权限边界，也就是复制功能的基础层。

尚未实现的核心复制功能：

```text
Primary 监听 replication_port
Replica 连接 primary_host:replication_port
复制消息帧与握手协议
Replication ID 与 offset
自动 Snapshot 全量同步
全量同步期间缓存新写命令
Snapshot 后的增量追赶
在线增量复制
ACK 与心跳
复制状态机
断线重连与部分同步
数据库锁和复制状态锁
ROLE 状态查询
主从集成测试
```

## 下一阶段：建立复制传输通道

下一阶段先只建立网络连接，不立即传输 Snapshot 或写命令。

Primary 需要同时维护两类端口：

```text
Primary
├── service_port：普通客户端连接
└── replication_port：Replica 复制连接
```

Replica 需要同时完成两类网络行为：

```text
Replica
├── 在 service_port 上接收只读客户端
└── 主动连接 primary_host:replication_port
```

以同一台机器上的两个进程为例：

```bash
./bin/kvstore primary 2000 2100
./bin/kvstore replica 2001 127.0.0.1 2100
```

目标连接关系是：

```text
客户端 ───────> Primary:2000
客户端 ───────> Replica:2001
Replica ──────> Primary:2100
```

建立连接后，操作系统中应当能够看到：

```text
Primary:2100 处于 LISTEN
Replica 临时端口到 Primary:2100 处于 ESTABLISHED
```

这一阶段首先需要检查现有 Reactor、NtyCo 和 Proactor 的启动结构。当前业务网络启动函数会持续运行，不能直接在它后面顺序启动第二个监听器。需要先确定复制通道是复用同一个事件循环，还是由独立线程运行，再为三种网络模型保留一致的上层复制接口。

连接建立只是通信基础，不等于主从同步已经完成。后续还需要在这条 TCP 连接上依次加入：

```text
握手
全量同步
增量追赶
在线复制
ACK 与心跳
断线恢复
```

## 第三阶段：先改造网络运行时，再承载复制连接

上一阶段已经明确了最终需要的三条网络路径：

```text
客户端 ───────> Primary:service_port
客户端 ───────> Replica:service_port
Replica ──────> Primary:replication_port
```

原来的网络层只服务于第一种场景：

```text
一个进程
    ↓
启动一个监听端口
    ↓
所有连接使用同一个 KV 命令处理函数
```

这对单机 KV 已经足够，但无法直接满足主从复制：

```text
Primary 需要同时监听两个端口
Replica 除了监听客户端，还要主动连接 Primary
不同端口收到的数据，需要交给不同协议处理器
复制连接断开时，还要通知复制管理器
```

因此这一阶段没有立刻开始发送 Snapshot，而是先把网络层从“单监听器启动函数”改造成可以组合监听器和连接器的运行时。

这部分工作量看起来较大，但它解决的是复制功能的承载问题。没有这层基础，后续的握手、全量同步和实时命令传播都会和某一个具体网络框架紧密耦合。

## 监听器是什么

监听器负责等待别人连接自己。它对应服务器端的典型调用流程：

```text
socket()
   ↓
bind()
   ↓
listen()
   ↓
accept()
```

例如 Primary 使用两个监听器：

```text
监听器 1
    port    = 19000
    handler = 客户端 KV 协议

监听器 2
    port    = 19100
    handler = 主从复制协议
```

同样是 TCP 连接，进入不同端口后会走不同的协议处理流程：

```text
HSET name jasper → 19000 → KV 命令处理器
PING             → 19100 → 复制协议处理器
```

这里不能继续只使用一个全局 `kvs_handler`，否则两个监听器无法分别绑定不同的处理函数。

## config 与 context 的区别

网络改造中出现了两类名字：

```text
config
context
```

`config` 表示启动任务的静态说明：

```text
在哪个端口监听
收到数据后调用哪个处理器
主动连接哪个主机和端口
连接建立或断开时调用哪个回调
```

`context` 表示网络框架真正运行时需要保存的数据：

```text
当前使用的 fd
当前监听器对应的 handler
连接上下文内存池
连接关闭回调
正在连接的目标地址
```

可以把它们理解为：

```text
config  = 任务说明书
context = 执行任务时背着的运行时背包
```

通用配置不包含 NtyCo 的内部实现细节。进入 NtyCo 后，再转换为 NtyCo 自己的 context：

```text
kvs_listener_config_t
        ↓
ntyco_listener_context_t

kvs_connector_config_t
        ↓
ntyco_connector_context_t
```

这样以后 Reactor 和 Proactor 可以使用同一份上层配置，但各自创建不同的运行时 context。

## 统一网络层消息处理函数

原来的 `msg_handler` 在 `kvstore.h`、Reactor、Proactor 和 NtyCo 中有多份重复定义。复制端口接入后，消息处理函数已经属于网络层与上层协议之间的公共契约，因此统一放入：

```text
include/server.h
```

最终接口增加了 `connection_fd`：

```c
typedef int (*msg_handler)(
    int connection_fd,
    char *msg,
    int length,
    char *response,
    int response_capacity,
    int *consumed_length
);
```

普通客户端协议暂时不关心具体连接，可以忽略：

```c
(void)connection_fd;
```

复制协议则必须知道是哪一条连接，因为后续需要为每台 Replica 保存：

```text
连接 fd
同步状态
确认 offset
待发送数据
```

因此 `connection_fd` 不是把 socket 逻辑塞进业务层，而是为有连接身份需求的上层协议提供最小必要信息。

## NtyCo 监听器上下文

通用监听器配置定义为：

```c
typedef struct {
    unsigned short port;
    msg_handler handler;
} kvs_listener_config_t;
```

NtyCo 将它转换为运行时上下文：

```c
typedef struct {
    unsigned short port;
    msg_handler handler;
    memory_pool_t connection_pool;
} ntyco_listener_context_t;
```

三项数据分别表示：

```text
port             当前监听器绑定的端口
handler          这个端口收到消息后使用的协议处理器
connection_pool  为 accept 出来的连接分配上下文
```

监听协程 `server()` 从 context 取出端口并创建监听 socket。每次 `accept()` 成功后，再为该连接创建独立的连接上下文。

## 为什么每条连接还需要 connection context

连接上下文定义为：

```c
typedef struct {
    int fd;
    msg_handler handler;
    kvs_connection_close_handler close_handler;
    memory_pool_t *pool;
} ntyco_connection_context_t;
```

它保存一条已经建立好的 TCP 连接运行时信息：

```text
fd             这条连接对应的 socket
handler        收到数据后调用谁
close_handler  断开时通知谁
pool           这个 context 应该归还到哪个内存池
```

最初代码把循环中的局部变量地址直接交给协程：

```c
int cli_fd = accept(...);
nty_coroutine_create(&read_co, server_reader, &cli_fd);
```

这里存在生命周期问题。`cli_fd` 是 `server()` 循环中的局部变量，下一轮 `accept()` 会继续修改它，而新建协程并不一定立刻读取该地址。多个协程可能最终看到同一个或已经变化的 fd，从而产生段错误或连接串线。

改造后，每次连接都从固定块内存池申请独立 context：

```c
ntyco_connection_context_t *connection =
    memory_pool_alloc(&context->connection_pool);

connection->fd = cli_fd;
connection->handler = context->handler;
connection->close_handler = NULL;
connection->pool = &context->connection_pool;
```

随后把 `connection` 交给读取协程。读取协程先复制其中需要长期使用的字段，再把 context 归还内存池：

```text
连接建立
    ↓
从内存池取出 connection context
    ↓
创建 server_reader 协程
    ↓
server_reader 保存 fd 和回调
    ↓
context 归还内存池
```

这个改造修复了实际测试中出现的 `Segmentation fault`。通过 Packet Sender 多次建立新连接并执行 `HSET`、`HGET` 后，服务不再崩溃。

## 为什么连接上下文使用专用内存池

连接上下文是固定大小、频繁申请和释放的对象，符合现有固定块内存池的使用条件：

```text
对象尺寸固定
生命周期短
申请释放频繁
不需要单独扩缩容
```

每个监听器持有自己的连接上下文池：

```text
listener context
    └── connection_pool
          ├── connection context
          ├── connection context
          └── ...
```

`server()` 负责从这个池申请，`server_reader()` 需要知道归还到哪个池，因此 pool 指针也被保存在 connection context 中。

当前每个 chunk 预分配 64 个连接上下文：

```c
#define NTYCO_CONNECTION_CONTEXT_BLOCKS_PER_CHUNK 64
```

这与 Hash、RBTree、SkipList 的节点池用途类似，但属于网络连接生命周期，不与存储引擎节点池混用。

## NtyCo 多监听器启动

Primary 需要同时启动业务监听器和复制监听器，因此增加：

```c
int ntyco_start_listeners(
    const kvs_listener_config_t *listeners,
    size_t listener_count
);
```

调用方传入监听器配置数组：

```c
kvs_listener_config_t listeners[] = {
    {
        .port = config.service_port,
        .handler = kvs_network_protocol
    },
    {
        .port = config.replication_port,
        .handler = kvs_replication_network_protocol
    }
};
```

NtyCo 为每份配置创建独立 listener context，再为每个 context 创建一个 `server` 协程：

```text
server 协程 1 → bind/listen service_port
server 协程 2 → bind/listen replication_port
                    ↓
             nty_schedule_run()
```

两个 `server()` 都包含长期运行的 `accept()` 循环，但使用 `nty_accept()` 后，等待连接的协程会主动让出执行权，不会阻塞另一个监听器。

这一点也解释了为什么 NtyCo 网络文件必须使用：

```text
nty_socket
nty_accept
nty_recv
nty_send
nty_close
```

如果直接使用系统的阻塞式 `accept()`，第一个监听协程可能会卡住整个调度线程，第二个端口就无法真正开始工作。

## 通用按行批处理器

客户端协议之前已经处理过 TCP 半包和粘包，但复制协议最初只处理一条 `PING`。如果一次 TCP 接收得到：

```text
PING\r\nPING\r\n
```

复制协议也应该返回：

```text
PONG\r\nPONG\r\n
```

如果只收到：

```text
PI
```

则不能提前报错，需要把数据保留到下一次 `recv()`。

因此将 CRLF 拆帧逻辑从具体 KV 协议中提取为通用网络缓冲能力：

```c
typedef int (*kvs_frame_handler)(
    int connection_fd,
    char *frame,
    int frame_length,
    char *response,
    int response_capacity
);
```

通用批处理器负责：

```text
查找 \r\n
识别完整帧
保留不完整半包
连续处理粘在一起的多帧
累加多个响应
计算 consumed_length
```

单帧处理器只负责：

```text
这条完整命令是什么意思
应该返回什么响应
```

于是客户端协议和复制协议都可以复用：

```text
客户端 TCP 数据
    ↓
kvs_line_batch_protocol()
    ↓
kvs_client_protocol()

复制 TCP 数据
    ↓
kvs_line_batch_protocol()
    ↓
kvs_replication_frame_protocol()
```

公共批处理器属于网络缓冲层，具体 KV 命令和复制命令仍然属于各自业务模块，因此解耦关系没有被破坏。

## 复制管理器

复制模块增加了进程级管理器：

```text
kvs_replication_manager_t
```

它不是网络连接本身，也不是启动参数，而是复制子系统运行期间的长期状态账本。

当前保存：

```text
role                    当前进程角色
replication_offset      当前复制进度
primary_host            Replica 的上游地址
primary_port            Replica 的上游复制端口
primary_connection_fd   当前上游 socket
state                   当前复制阶段
initialized             是否已经初始化
```

三类数据需要明确区分：

```text
kvs_server_config_t
    main() 启动时解析出的配置

kvs_replication_manager_t
    复制模块在整个进程生命周期中保存的业务状态

ntyco_connector_context_t
    NtyCo 执行 connect() 时使用的网络运行时数据
```

Replica 初始化时，复制管理器会复制 `primary_host`，而不是永久保存 `argv` 指针：

```text
config.primary_host
        ↓ memcpy，包含结尾 \0
replication_manager.primary_host
```

Standalone 和 Primary 没有上游节点，因此对应字段保持为空：

```text
primary_host = ""
primary_port = 0
primary_connection_fd = -1
```

复制 offset 初始值暂时来自当前 AOF 文件位置：

```c
kvs_aof_get_offset()
```

这为后续使用 Snapshot offset 和增量 AOF 打下基础，但仍然保留了复制 offset 与 AOF offset 的概念边界。

## 当前复制状态机

复制状态枚举已经定义为：

```text
DISCONNECTED  尚未建立复制连接
CONNECTING    Replica 正在连接 Primary
HANDSHAKE     已连接，正在交换同步信息
FULL_SYNC     正在传输并加载 Snapshot
CATCH_UP      正在补齐 Snapshot 之后的增量命令
ONLINE        已经追平，持续接收实时写命令
```

目前真正走通的状态范围是：

```text
DISCONNECTED
    ↓ connect()
HANDSHAKE
    ↓ PING / PONG
复制 TCP 连接保持建立
```

`FULL_SYNC`、`CATCH_UP` 和 `ONLINE` 已经有类型定义，但还没有实现对应动作。状态枚举不是完整状态机本身；只有当事件、状态判断和状态转换逻辑都接入后，才构成真正运行的状态机。

## Primary 复制端口的最小协议

为了先验证两端通信，在复制端口实现最小 `PING/PONG` 协议：

```text
Replica → Primary：PING\r\n
Primary → Replica：PONG\r\n
```

Primary 的复制端口使用：

```c
kvs_replication_network_protocol
```

它先调用通用按行批处理器，再将每条完整帧交给复制单帧处理器。未知复制命令返回：

```text
ERROR unknown replication command\r\n
```

这个 `PING/PONG` 不是 TCP 三次握手。TCP 三次握手由内核在 `connect()` 和 `accept()` 过程中完成；这里是建立 TCP 连接后的应用层协议探活，作用是确认：

```text
连接到的确实是复制协议端口
双方能够按照约定收发完整帧
通用批处理能够正确处理复制消息
```

## 监听器与连接器

监听器和连接器是方向相反的网络任务：

```text
listener
    等待别人连接自己
    bind + listen + accept

connector
    主动连接别人
    socket + connect
```

Primary 的复制端口属于 listener：

```text
Primary:19100 等待 Replica
```

Replica 到 Primary 的复制链路属于 connector：

```text
Replica 主动连接 127.0.0.1:19100
```

通用连接器配置定义了：

```text
host             目标主机
port             目标端口
open_handler     连接建立后做什么
message_handler  收到数据后交给谁
close_handler    连接断开后通知谁
```

复制管理器负责生成这份业务配置，NtyCo 只负责执行网络动作：

```text
复制模块
    决定连接谁、发送什么、如何解释响应
            ↓ connector config
NtyCo
    执行 socket、connect、send、recv
```

这样复制模块不需要直接依赖 NtyCo，网络层也不需要理解 `PING`、`PONG` 或复制状态。

## 为什么需要 open、message 和 close 三类回调

连接生命周期包含三个不同事件：

```text
连接建立
收到消息
连接断开
```

复制模块分别处理：

```text
open_handler
    保存 primary_connection_fd
    状态进入 HANDSHAKE
    生成 PING\r\n

message_handler
    通过通用批处理拆帧
    识别 Primary 返回的 PONG

close_handler
    将 primary_connection_fd 重置为 -1
    状态退回 DISCONNECTED
    保留 host、port 和 offset，为重连做准备
```

网络层只在正确时间调用回调，不直接修改复制管理器内部状态。

## 统一连接关闭路径

原来的 `server_reader()` 在多个分支中直接关闭 fd：

```text
输入缓冲区已满
协议处理失败
缓冲区消费失败
发送失败
对端主动断开
recv 发生错误
```

复制连接断开时还需要触发 `close_handler`。如果每个分支分别关闭和通知，很容易出现：

```text
重复 close
某条错误路径漏掉通知
回调调用两次
新增分支时忘记清理
```

因此读取协程改为统一退出路径：

```text
各类错误或断开
      ↓ break / goto
connection_closed
      ↓
nty_close(fd)
      ↓
close_handler(fd)
```

这里使用向下跳转到清理标签的 `goto`，目的是从“发送循环嵌套在接收循环”的位置一次跳出两层，并集中释放资源。它不是任意跳转业务流程，而是 C 项目中常见的统一错误清理方式。

## NtyCo 统一运行时

只有多监听器仍然不够，因为 Replica 同时需要：

```text
一个 listener：监听自己的客户端端口
一个 connector：主动连接 Primary
```

因此增加统一运行时入口：

```c
int ntyco_start_runtime(
    const kvs_listener_config_t *listeners,
    size_t listener_count,
    const kvs_connector_config_t *connectors,
    size_t connector_count
);
```

它的执行流程是：

```text
初始化所有 listener context
        ↓
初始化所有 connector context
        ↓
为 listener 创建 server 协程
        ↓
为 connector 创建主动连接协程
        ↓
nty_schedule_run()
```

当前限制为：

```c
#define NTYCO_MAX_LISTENER_COUNT 2
#define NTYCO_MAX_CONNECTOR_COUNT 1
```

也就是当前一个进程最多使用两个监听器和一个主动连接器。对现阶段的一主一从足够：

```text
Primary：2 listeners + 0 connector
Replica：1 listener  + 1 connector
Standalone：1 listener + 0 connector
```

`ntyco_start_listeners()` 被保留为兼容适配器：

```c
return ntyco_start_runtime(
    listeners,
    listener_count,
    NULL,
    0
);
```

它没有维护第二套实现，只是让原有只需要监听器的代码不必立刻改成四参数调用。

## Replica 主动连接 Primary

Replica 启动时，从复制管理器构造连接器配置：

```c
kvs_replication_build_connector_config(&connector);
```

然后把客户端 listener 和 Primary connector 一起交给 NtyCo：

```c
ntyco_start_runtime(
    &listener,
    1,
    &connector,
    1
);
```

连接器协程执行：

```text
解析 Primary IPv4 地址
        ↓
nty_socket()
        ↓
nty_connect(primary_host, primary_port)
        ↓
调用 open_handler 生成 PING
        ↓
处理 partial send，完整发送 PING
        ↓
复用 server_reader 接收后续数据
```

连接成功后构造的 connection context 不来自监听器内存池，因此：

```text
pool = NULL
```

`server_reader()` 已经支持这种情况：只有 pool 非空时才归还内存池。这样监听器接收的连接和连接器主动建立的连接可以复用同一套接收、拆包、协议分发和发送逻辑。

当前地址转换使用：

```c
inet_pton(AF_INET, primary_host, ...)
```

所以当前支持 IPv4 字面地址，例如：

```text
127.0.0.1
192.168.1.20
```

暂时不支持域名和 IPv6。后续如果需要支持：

```text
primary.example.com
::1
```

应将地址解析替换为 `getaddrinfo()`。

## 127.0.0.1 在测试中的意义

下面的 Replica 启动命令：

```bash
./bin/kvstore replica 19001 127.0.0.1 19100
```

参数含义是：

```text
19001       Replica 的客户端服务端口
127.0.0.1   Primary 所在主机
19100       Primary 的复制端口
```

`127.0.0.1` 表示当前机器自己。测试时 Primary 和 Replica 都运行在同一台 Ubuntu 虚拟机中，因此使用回环地址。

如果 Primary 位于另一台机器，例如 `192.168.1.20`，则 Replica 应使用：

```bash
./bin/kvstore replica 19001 192.168.1.20 19100
```

Primary 不需要填写自己的主机地址，因为它执行的是：

```text
bind(INADDR_ANY, replication_port)
listen()
```

Replica 是主动发起连接的一方，所以必须同时知道目标主机和目标端口。

## 为什么测试时使用两个独立目录

当前持久化文件使用相对路径：

```text
snapshot.db
appendonly.aof
```

如果在同一个项目目录中同时启动 Primary 和 Replica，两个进程会直接打开同一份文件：

```text
Primary ─┐
         ├── ~/dev/91kvstore/appendonly.aof
Replica ─┘
```

这不是主从同步，而是两个进程错误地共享本地文件，还可能造成内容交错和状态污染。

因此测试时分别创建临时工作目录，同时使用绝对路径运行同一个二进制文件。

Primary：

```bash
cd "$(mktemp -d /tmp/91kv-primary.XXXXXX)"
/home/jaspersao/dev/91kvstore/bin/kvstore primary 19000 19100
```

Replica：

```bash
cd "$(mktemp -d /tmp/91kv-replica.XXXXXX)"
/home/jaspersao/dev/91kvstore/bin/kvstore replica 19001 127.0.0.1 19100
```

由此形成：

```text
/tmp/91kv-primary.xxxxxx/
├── snapshot.db
└── appendonly.aof

/tmp/91kv-replica.xxxxxx/
├── snapshot.db
└── appendonly.aof
```

两个节点的数据是否一致，必须通过复制协议实现，而不能依赖共享文件。

## 自动连接测试

Primary 启动后输出：

```text
listen port : 19000
listen port : 19100
```

Replica 启动后输出：

```text
listen port : 19001
connect to primary 127.0.0.1:19100
```

使用 `ss` 检查得到三条监听：

```text
Primary pid=8282  LISTEN 19000
Primary pid=8282  LISTEN 19100
Replica pid=8371  LISTEN 19001
```

复制连接同时存在两个方向的 socket 视图：

```text
Replica 127.0.0.1:35810 → 127.0.0.1:19100 ESTAB
Primary 127.0.0.1:19100 → 127.0.0.1:35810 ESTAB
```

`35810` 是操作系统临时分配给 Replica 的本地端口，不需要写入启动参数。

这个结果证明：

```text
Primary 两个监听器能够并行工作
Replica listener 与 connector 能够并行工作
Replica 已主动连接 Primary 复制端口
PING/PONG 后连接保持建立
```

随后使用 Packet Sender 连接 Replica 的业务端口 `19001`，验证：

```text
读命令仍能正常处理
客户端写命令仍返回 READONLY
复制连接不会阻塞普通客户端连接
```

需要再次强调：此时只证明复制传输通道已经建立，并不代表数据已经完成同步。

## 第三阶段提交

这一阶段不是一次完成，而是按网络基础设施逐步提交。

NtyCo 监听器与连接上下文：

```text
020475e refactor(ntyco): 引入监听器与连接上下文
```

NtyCo 多监听器：

```text
f88c029 feat(network): 支持 NtyCo 多监听器启动
```

复制运行时基础设施：

```text
8dfbdb6 feat(replication): 建立复制运行时基础设施
```

复制端口与通用批处理：

```text
ea7311e feat(replication): 接入复制端口与通用批处理
```

Replica 主动连接 Primary：

```text
46805dd feat(replication): 实现 Replica 主动连接 Primary
```

这些提交分别解决：

```text
连接上下文生命周期与段错误
一个 NtyCo 调度器启动多个监听器
复制管理器、状态和 offset 基础
复制端口、半包、粘包与 PING/PONG
Replica 主动连接与断开通知
```

## 截至第三阶段的架构

目前主进程启动后的结构是：

```text
main()
  ├── 解析 kvs_server_config_t
  ├── 初始化 KV Engine
  ├── 加载 Snapshot
  ├── 重放 AOF
  ├── 打开 AOF
  ├── 初始化 replication manager
  └── 启动网络运行时
```

Primary 的网络结构：

```text
Primary
├── listener:19000
│     └── kvs_network_protocol
│           └── 客户端 KV 命令
│
└── listener:19100
      └── kvs_replication_network_protocol
            └── PING / PONG 与后续同步协议
```

Replica 的网络结构：

```text
Replica
├── listener:19001
│     └── kvs_network_protocol
│           └── 客户端读命令
│           └── 客户端写命令返回 READONLY
│
└── connector → Primary:19100
      ├── open_handler：发送 PING
      ├── message_handler：处理 PONG
      └── close_handler：状态退回 DISCONNECTED
```

网络通用层与业务协议层的边界是：

```text
NtyCo
    负责 socket、协程、收发、连接生命周期

buffer.c
    负责 CRLF 拆帧、半包、粘包和批量响应

kvstore.c
    负责客户端命令来源、权限和存储引擎执行

replication.c
    负责复制角色、状态、握手和后续同步决策
```

## 截至当前的完成比例

按照完整的一主一从目标估算，目前整体约完成 50%。

已经完成：

```text
三种启动角色
严格参数解析
命令来源模型
Replica 客户端只读
恢复命令不重复追加 AOF
Primary 客户端端口与复制端口
通用 CRLF 批处理
NtyCo 多监听器
NtyCo 主动连接器
复制管理器基础
Replica 自动连接 Primary
PING/PONG 应用层探活
连接断开通知
独立目录双进程测试
```

从不同角度看：

```text
网络与运行时基础：约 80%
真正的数据复制：约 10%
完整主从目标：约 50%
```

尚未实现：

```text
PSYNC 同步协商
Replication ID
Snapshot 全量传输
Replica 接收并加载 Snapshot
全量同步期间缓存新增写命令
Snapshot 后的 AOF 增量追赶
ONLINE 实时命令传播
ACK offset
断线自动重连
复制 backlog 与部分同步
Reactor / Proactor 的完整复制运行时
主从集成测试与异常测试
```

## 下一阶段：PSYNC 同步协商

现在已经有一条稳定的 TCP 通道：

```text
Replica ─────────────> Primary:replication_port
```

下一步不直接发送 Snapshot，而是先让双方协商同步方式。Replica 将发送类似：

```text
PSYNC ? -1\r\n
```

其中：

```text
?   表示 Replica 还不知道 Primary 的 replication ID
-1  表示 Replica 没有可继续使用的旧复制 offset
```

Primary 收到后，需要根据自己的 replication ID、当前 offset 和可保留的增量范围作出决定：

```text
无法部分同步
    ↓
FULLRESYNC <replication-id> <offset>
    ↓
进入 Snapshot 全量同步

可以部分同步
    ↓
CONTINUE <offset>
    ↓
从指定位置发送增量复制流
```

第一版会先走通 `PSYNC ? -1` 到 `FULLRESYNC`，也就是首次连接的全量同步协商。等 Snapshot 与增量追赶完成后，再补充断线重连时的部分同步。

下一阶段仍然按小步实现：

```text
1. 定义 replication ID
2. 定义 PSYNC 请求格式
3. Replica 在 PONG 后发送 PSYNC
4. Primary 解析 PSYNC
5. Primary 返回 FULLRESYNC
6. 状态从 HANDSHAKE 进入 FULL_SYNC
```

只有当这段协商稳定后，才开始设计 Snapshot 的文件头、长度、传输边界和加载流程。

## 第四阶段：首次 PSYNC 全量同步协商

上一阶段已经解决了“Replica 怎样主动找到并连接 Primary”的问题，但连接建立以后，双方仍然不知道应该从哪里开始同步。

因此这一阶段先完成同步协商，而不是立刻传输数据。

当前完整的应用层握手过程是：

```text
Replica                              Primary
   │                                    │
   │── PING ───────────────────────────>│
   │<──────────────────────────── PONG ─│
   │                                    │
   │── PSYNC ? -1 ─────────────────────>│
   │                                    │ 生成复制快照
   │<── FULLRESYNC <id> <offset> <size>─│
   │                                    │
   │ 进入 FULL_SYNC                     │
```

这里的 PING/PONG 只是应用层握手，不是 TCP 三次握手。

TCP 三次握手由操作系统和 `nty_connect()` 完成；复制层的 PING/PONG 用来确认：

```text
连接到的确实是 KVStore 的复制端口
双方能够理解当前复制协议
连接已经可以继续进行 PSYNC 协商
```

### 为什么首次同步发送 PSYNC ? -1

Replica 首次启动时没有保存过 Primary 的复制历史，因此发送：

```text
PSYNC ? -1

?   ：不知道 Primary 的 replication ID
-1  ：没有能够继续同步的旧 offset
```

这个请求实际上是在告诉 Primary：

```text
“我没有任何可复用的同步历史，请给我一次完整同步。”
```

Primary 收到该请求后，第一版直接选择 `FULLRESYNC`。将来实现复制 backlog 后，PSYNC 还可以携带旧 ID 和旧 offset，由 Primary 判断是否能够返回 `CONTINUE`。

## Replication ID

Primary 初始化复制管理器时，从系统随机源读取 20 个随机字节，再将每个字节转换为两个十六进制字符：

```text
20 个二进制字节
    ↓ 每字节转换为两个十六进制字符
40 个可打印字符
```

例如：

```text
8f39d585411b311d66ffa3dae4b43e9a08a4542a
```

使用 40 个十六进制字符的原因不是哈希计算更方便，而是：

```text
随机二进制数据可能包含 \0、换行和不可打印字符
十六进制字符串可以安全地放入文本协议和日志
20 字节仍然提供足够大的随机空间
```

当前 ID 的生命周期与 Primary 进程一致：

```text
Primary 启动
    → 生成新的 replication ID

Primary 正常运行
    → ID 保持不变

Primary 重启
    → 生成新的 ID
```

Primary 重启后 ID 改变，旧 Replica 暂时不能把原有 offset 当作同一条复制历史继续使用，因此会退回全量同步。这是当前没有 backlog 和持久化复制元数据时更安全的行为。

## 上游协议与下游协议

复制连接虽然只有一条，但两个方向收到的消息不同，因此分别使用两个单帧处理函数。

Primary 处理 Replica 发来的请求：

```text
kvs_replication_frame_protocol()
    ├── PING
    └── PSYNC ? -1
```

Replica 处理 Primary 发来的响应：

```text
kvs_replication_upstream_frame_protocol()
    ├── PONG
    └── FULLRESYNC ...
```

这里的 `frame` 不是状态，而是通用按行批处理器已经拆出来的一条完整协议消息。例如：

```text
TCP 输入：PONG\r\nFULLRESYNC ...\r\n

拆帧后：
frame 1 = PONG
frame 2 = FULLRESYNC ...
```

这样复制业务只处理完整消息，不需要重复处理 TCP 半包、粘包和 CRLF 边界。

## FULLRESYNC 的解析与状态变化

Primary 最初返回：

```text
FULLRESYNC <replication-id> <offset>\r\n
```

Replica 使用 `sscanf()` 按格式提取字段，并通过 `%n` 得到实际匹配到的位置：

```c
FULLRESYNC %40[0-9a-f] %lld%n
```

除了检查匹配字段数，还需要检查：

```text
解析是否完整消费了整条 frame
replication ID 是否恰好为 40 个十六进制字符
offset 是否非负
当前进程是否为 Replica
当前连接是否就是复制管理器记录的 Primary 连接
当前状态是否仍处于 HANDSHAKE
```

校验通过后，Replica 保存 Primary ID 和同步起点，并进入：

```text
KVS_REPLICATION_STATE_FULL_SYNC
```

这次状态变化只表示：

```text
全量同步协商已经成功
Replica 已经准备接收 Snapshot
```

它并不表示 Snapshot 已经加载完成。

这一阶段对应提交：

```text
1bc52f3 feat(replication): 实现首次 PSYNC 全量同步协商
```

## 第五阶段：为 Snapshot 文件传输准备元数据

完成 `FULLRESYNC <id> <offset>` 后，新的问题出现了：Primary 即使直接发送 Snapshot 文件，Replica 也不知道文件到哪里结束。

Snapshot 是原始文件负载，文件内部本身可以出现换行，不能继续使用 CRLF 判断文件边界。

因此协议扩展为：

```text
FULLRESYNC <replication-id> <aof-offset> <snapshot-size>\r\n
```

例如：

```text
FULLRESYNC 8f39...542a 580 2048\r\n
```

三个字段分别表示：

```text
8f39...542a ：Primary 的复制历史 ID
580         ：生成这份 Snapshot 时对应的 AOF 位置
2048        ：接下来 Snapshot 文件负载的总字节数
```

### offset 与 snapshot size 为什么不能混用

`aof_offset` 描述 AOF 文件中的逻辑进度：

```text
Snapshot 加载完以后
    → 从 AOF 的哪个位置继续发送增量命令
```

`file_size` 描述 Snapshot 文件的物理长度：

```text
Replica 需要从网络中准确接收多少原始字节
```

两个值可能完全不同。假设历史中多次修改同一个键：

```text
HSET name A
HMOD name B
HMOD name C
HMOD name D
```

AOF 会保留这些历史写命令，因此 offset 会持续增长；Snapshot 只需要保存最终状态：

```text
HSET name D
```

所以 Snapshot 文件可能很小，但对应的 AOF offset 已经很大。

全量同步需要同时拥有两个值：

```text
file_size
    → 决定 Snapshot 接收边界

aof_offset
    → 决定 Snapshot 之后的增量同步起点
```

## 扩展 Snapshot 保存接口

原来的持久化接口只有：

```c
int kvs_snapshot_save(const char *path);
```

它只能告诉调用者成功或失败，无法返回本次生成的 Snapshot 对应 offset 和文件大小。

现在定义：

```c
typedef struct {
    long long aof_offset;
    long long file_size;
} kvs_snapshot_metadata_t;
```

并将接口升级为：

```c
int kvs_snapshot_save(
    const char *path,
    kvs_snapshot_metadata_t *metadata
);
```

返回值仍然只负责表示：

```text
0  ：保存成功
-1 ：保存失败
```

可选输出参数 `metadata` 负责返回额外结果。

普通客户端执行 SAVE 时：

```c
kvs_snapshot_save("snapshot.db", NULL);
```

它只关心持久化是否成功，不需要复制元数据。

Primary 收到 PSYNC 时：

```c
kvs_snapshot_metadata_t snapshot_metadata;

kvs_snapshot_save(
    "replication.snapshot",
    &snapshot_metadata
);
```

它需要同时获得：

```text
snapshot_metadata.aof_offset
snapshot_metadata.file_size
```

两个场景不是由 `kvs_snapshot_save()` 自己判断的，而是由调用路径区分：

```text
业务端口收到 SAVE
    → 客户端协议
    → 保存 snapshot.db

复制端口收到 PSYNC ? -1
    → 复制协议
    → 保存 replication.snapshot
```

持久化模块只负责“把当前数据保存到指定路径”，不需要认识 Standalone、Primary、Replica、socket 或 NtyCo。这保持了模块边界。

## Snapshot 文件大小的取得时机

Snapshot 内容全部写完并执行 `fflush()`、`fsync()` 后，通过 `ftello()` 获取当前文件位置：

```text
写入 Snapshot 内容
    ↓
fflush
    ↓
fsync
    ↓
ftello
    ↓
得到完整文件字节数
```

这里先将结果保存在局部变量中，而不是立即填写调用者的 `metadata`。

原因是后面仍然可能发生：

```text
fclose 失败
rename 失败
```

只有所有步骤都成功以后，才把元数据交给调用者：

```text
临时文件完整写入
    ↓
关闭成功
    ↓
rename 成功
    ↓
提交 metadata
```

这与事务中的“最后提交”类似：函数返回失败时，调用者不会得到一组看起来有效、实际上却没有成功发布的元数据。

## rename 在项目中的原子替换

Snapshot 不直接写入正式文件，而是：

```text
先写 snapshot.db.tmp
        ↓
写完、刷新、同步并关闭
        ↓
rename(snapshot.db.tmp, snapshot.db)
```

在写新文件期间：

```text
snapshot.db     → 上一次完整快照
snapshot.db.tmp → 本次尚未完成的新快照
```

如果写入过程中进程崩溃，正式文件仍然是上一份完整快照。

同一文件系统内的 `rename()` 以原子方式替换目录项，因此其他读取者只能看到：

```text
旧的完整 snapshot.db
```

或者：

```text
新的完整 snapshot.db
```

不会看到一个“只替换了一半”的正式文件。

这里的原子性只解决文件发布问题，不代表整个数据库快照天然具备并发一致性，也不代表 Snapshot 已经通过网络发送。更严格的掉电持久性将来还可以在 rename 后同步父目录。

## Primary 生成复制快照的时机

Primary 在复制端口收到 `PSYNC ? -1` 后，先检查：

```text
复制管理器已经初始化
当前角色必须是 PRIMARY
replication ID 已经生成
```

只有全部成立，才执行：

```text
kvs_snapshot_save("replication.snapshot", &snapshot_metadata)
```

因此 Replica 不会误生成 Primary 的复制快照。

生成成功后，Primary 将同一次保存返回的两个值写入复制管理器：

```text
replication_offset  = snapshot_metadata.aof_offset
snapshot_file_size  = snapshot_metadata.file_size
```

随后才构造：

```text
FULLRESYNC <id> <offset> <snapshot-size>\r\n
```

这个顺序保证 FULLRESYNC 中的 offset 与即将传输的 Snapshot 属于同一次保存，而不是分别在两个时刻独立读取。

## Replica 保存 Snapshot 元数据

Replica 将 FULLRESYNC 的解析格式升级为三个字段：

```c
FULLRESYNC %40[0-9a-f] %lld %lld%n
```

解析并校验成功后保存：

```text
replication_id
replication_offset
snapshot_file_size
```

随后进入 `FULL_SYNC`。

复制管理器中的 `snapshot_file_size` 初始值为 `-1`：

```text
-1  ：当前没有有效的 Snapshot 大小
> 0 ：已经通过 FULLRESYNC 得到本次文件大小
```

在复制管理器初始化和销毁时都恢复为 `-1`，避免上一次连接留下的旧值被下一次同步误用。

## 元数据协商测试

Primary 使用独立目录启动：

```bash
cd "$(mktemp -d /tmp/91kv-primary.XXXXXX)"
/home/jaspersao/dev/91kvstore/bin/kvstore primary 19000 19100
```

Replica 使用另一个独立目录启动：

```bash
cd "$(mktemp -d /tmp/91kv-replica.XXXXXX)"
/home/jaspersao/dev/91kvstore/bin/kvstore replica 19001 127.0.0.1 19100
```

Primary 输出：

```text
replication: FULLRESYNC response ready
id=8f39...542a offset=0 snapshot_size=13 fd=8
```

Replica 输出：

```text
replication: enter FULL_SYNC
id=8f39...542a offset=0 snapshot_size=13 fd=7
```

两边的 ID、offset 和 snapshot size 完全一致，证明第三个字段已经正确生成、传输、解析并保存。

空库 Snapshot 大约只有：

```text
AOF_OFFSET 0\n
```

因此测试中的 `snapshot_size=13` 是合理结果。

这一阶段对应提交：

```text
49e0e12 feat(replication): 协商全量同步快照元数据
```

## 为什么不能直接扩大 msg_handler 缓冲区

当前 `msg_handler` 的语义是：

```text
网络层收到一次输入
    ↓
协议处理器生成一个小响应
    ↓
网络层发送该响应
```

这适合：

```text
PING → PONG
PSYNC → FULLRESYNC
HGET → value
```

但不适合大小不确定的 Snapshot 文件。

如果简单把 `BUFFER_LENGTH` 从 1024 提高到几十 MB，会产生三个问题。

第一，每条连接都会持有巨大的输入和输出缓冲区。当前缓冲区还是连接读取协程中的局部对象，过大的局部数组可能直接耗尽协程栈。

第二，Snapshot 没有固定最大值。它可能从几十字节增长到数 GB，永远无法选择一个保证足够、又不会浪费内存的固定缓冲区。

第三，`msg_handler` 只在网络层收到请求时调用。大文件需要在同一个请求之后连续读取并发送许多数据块，不能反复重新执行 PSYNC，否则会重复解析命令、重复生成 Snapshot。

因此正确模型不是准备一个特别大的桶，而是反复使用同一个小桶：

```text
读取 Snapshot 1024 字节
    ↓
发送这 1024 字节
    ↓
继续读取下一块
    ↓
直到累计发送 snapshot_file_size 字节
```

控制消息与文件流需要不同的调用语义：

```text
msg_handler
    → 解析控制命令
    → 生成 PONG、FULLRESYNC 等小响应

stream_handler
    → 连续生产 Snapshot 数据块
    → 每次只返回当前缓冲区能够容纳的字节
```

它们可以复用同一个小型网络输出缓冲区，但不应该混成一个“必须一次生成全部内容”的接口。

## 当前进度边界

截至提交 `49e0e12`，全量同步已经做到：

```text
✅ Replica 主动连接 Primary
✅ PING / PONG
✅ PSYNC ? -1
✅ Primary 生成 replication.snapshot
✅ Snapshot 保存后返回 offset 和 file size
✅ FULLRESYNC 携带 ID、offset 和 size
✅ Replica 校验并保存三项元数据
✅ Replica 进入 FULL_SYNC
```

尚未做到：

```text
❌ Primary 发送 Snapshot 原始字节
❌ Replica 按 snapshot_file_size 接收文件
❌ Replica 使用临时文件原子落盘
❌ Replica 清空旧数据并加载新 Snapshot
❌ 从 Snapshot offset 开始追赶 AOF 增量
❌ 追平后进入 ONLINE
```

所以日志中的：

```text
enter FULL_SYNC
```

当前含义是“准备开始全量同步”，而不是“全量同步已经完成”。

## 下一步：通用分块输出能力

下一步不会在 `replication.c` 中直接调用 `nty_send()`，因为那会让复制业务绑死在 NtyCo 后端。

目标边界仍然是：

```text
replication.c
    → 决定发送哪份 Snapshot
    → 提供下一块文件数据

network/ntyco.c
    → 负责 socket 与 partial send
    → 不理解 Snapshot 内容

persistence/snapshot.c
    → 负责 Snapshot 文件格式、生成与读取
```

计划在通用网络接口中加入流式输出回调：

```c
typedef int (*kvs_connection_stream_handler)(
    int connection_fd,
    char *output,
    int output_capacity
);
```

预期返回语义：

```text
> 0 ：本次生成的字节数，网络层发送后继续取下一块
= 0 ：流已经完成
< 0 ：读取失败，网络层关闭连接
```

最终发送路径应当是：

```text
Primary 发送 FULLRESYNC 控制行
    ↓
网络层发送控制行
    ↓
stream handler 分块读取 replication.snapshot
    ↓
网络层处理 partial send
    ↓
累计发送 snapshot_file_size 字节
    ↓
文件流结束
```

Replica 端则需要在 `FULL_SYNC` 状态下切换输入解释方式：

```text
HANDSHAKE 状态
    → 输入按 CRLF 控制帧解析

FULL_SYNC 状态
    → 输入按 snapshot_file_size 读取原始文件字节

Snapshot 收满
    → 原子发布并加载
    → 进入 CATCH_UP
```

这也是为什么复制状态机不只是几个名字：状态决定同一条 TCP 连接上的后续字节应该按“控制协议”还是“原始文件负载”解释。

## 第六阶段：Snapshot 分块传输与安装

前一阶段已经完成 Snapshot 元数据协商，但 Replica 只知道“接下来应该收到多少字节”，真正的文件内容还没有经过网络传输。

这一阶段需要补齐完整数据路径：

```text
Primary 的 replication.snapshot
    ↓ 分块读取
复制连接
    ↓ 分块接收
Replica 的临时文件
    ↓ 完整性确认与原子替换
Replica 的 snapshot.db
    ↓ 加载
Replica 内存 Engine
```

这里没有一次性申请与 Snapshot 等大的内存。Primary 和 Replica 都只复用固定大小的网络缓冲区，因此 Snapshot 即使继续增长，也不会让单条连接占用同等大小的内存。

## 为什么为端口绑定处理函数

Primary 同时对外提供两个端口：

```text
service_port      → 普通客户端协议
replication_port  → 主从复制协议
```

虽然两个端口最终都会执行 `accept()`、`recv()` 和 `send()`，但收到字节后的解释方式不同。

普通客户端端口需要处理：

```text
SET / GET / DEL / MOD / HSET / HGET ...
```

复制端口需要处理：

```text
PING / PSYNC / FULLRESYNC / Snapshot / AOF stream
```

因此监听器配置不仅保存端口，还保存与该端口对应的协议回调：

```c
typedef struct {
    unsigned short port;
    msg_handler handler;
    kvs_connection_stream_handler stream_handler;
} kvs_listener_config_t;
```

网络层只负责：

```text
在哪个端口建立连接
收到多少字节
把字节交给哪个回调
把回调产生的响应发送出去
```

它不需要写死“19000 是客户端端口”“19100 是复制端口”。这种做法让网络运行时保持通用，也让不同端口可以安装不同业务协议。

## stream handler 的职责

普通 `msg_handler` 是请求驱动的：

```text
收到一批输入
    ↓
解析命令
    ↓
产生一次响应
```

Snapshot 传输则需要在一条 `PSYNC` 请求之后连续输出多块数据。因此新增 `stream_handler`：

```c
typedef int (*kvs_connection_stream_handler)(
    int connection_fd,
    char *output,
    int output_capacity
);
```

它每次调用只负责生产下一块数据：

```text
> 0 ：output 中有这么多字节需要发送
= 0 ：当前流已经结束
< 0 ：产生数据失败
```

NtyCo 网络层拿到正数后调用 `send_all()`，保证即使一次 `send()` 只发送了部分数据，也会继续发送剩余内容。

因此职责边界变成：

```text
replication.c
    → 决定现在应该输出 Snapshot 还是 AOF

persistence.c
    → 从文件读取下一块数据

network/ntyco.c
    → 保证这一块数据完整写入 socket
```

## Snapshot reader

Primary 使用 `kvs_snapshot_reader_t` 保存一次分块读取任务的状态：

```c
typedef struct {
    int fd;
    long long remaining;
} kvs_snapshot_reader_t;
```

其中：

```text
fd         → 当前打开的 Snapshot 文件
remaining  → 还剩多少协商好的字节没有发送
```

打开 reader 时不仅打开文件，还通过 `fstat()` 校验实际文件大小必须等于 FULLRESYNC 中发送的 `snapshot_file_size`。

每次读取大小取两者中的较小值：

```text
min(output_capacity, remaining)
```

最后一块可能小于网络缓冲区。例如还剩 100 字节时，即使输出缓冲区能容纳 1024 字节，也只能读取 100 字节。

每次读取成功后执行：

```text
remaining -= result
```

当 `remaining == 0` 时，本次 Snapshot 的协商字节已经全部读完。

## Snapshot writer

Replica 使用对应的 `kvs_snapshot_writer_t`：

```c
typedef struct {
    int fd;
    long long remaining;
} kvs_snapshot_writer_t;
```

writer 的 `remaining` 表示还需要从网络中消费多少 Snapshot 字节。

这也是 `kvs_snapshot_writer_write()` 为什么返回“实际写入字节数”。一次 TCP 接收可能同时包含：

```text
Snapshot 最后 100 字节
+
后续 AOF 数据
```

如果 writer 还差 100 字节，它只能消费输入缓冲区的前 100 字节：

```text
written = 100
```

剩余的 AOF 数据必须继续留在网络输入缓冲区，交给下一个复制状态处理，不能被 Snapshot writer 一并吞掉。

## 临时文件、commit 与 abort

Replica 不会直接覆盖正式的 `snapshot.db`，而是先写入：

```text
snapshot.db.replication.tmp
```

打开临时文件时使用：

```c
O_WRONLY | O_CREAT | O_TRUNC
```

含义分别是：

```text
O_WRONLY  → 只写打开
O_CREAT   → 文件不存在时创建
O_TRUNC   → 文件已经存在时先清空
```

权限 `0600` 表示只有当前用户可以读写该临时文件。

只有 `remaining == 0`，也就是协商好的 Snapshot 已经一字节不少地接收完成，才允许执行 commit：

```text
fsync(fd)
    ↓
close(fd)
    ↓
rename(temp_path, final_path)
```

`fsync()` 保证文件内容已经交给底层存储；`rename()` 在同一文件系统内原子替换正式 Snapshot。其他线程或重启恢复过程不会看到只写了一半的新文件。

如果接收、落盘、关闭或重命名失败，则执行 abort：

```text
关闭临时文件
unlink(temp_path)
```

这里的 `unlink()` 表示删除临时文件的目录项，避免下一次同步误用半成品。

## 同一次 recv 跨越两个协议阶段

TCP 是字节流，不会保证一次 `recv()` 恰好对应一条命令或一个阶段。

例如 Replica 可能一次收到：

```text
Snapshot 最后 100 字节
+
AOF 第一条命令 25 字节
```

Snapshot writer 消费前 100 字节后，输入缓冲区仍然剩下 25 字节。此时状态已经从 `FULL_SYNC` 切换到 `CATCH_UP`，剩余字节应当立即按照 AOF 命令解释。

因此 `server_reader()` 在一次 `recv()` 后使用循环反复调用协议处理器：

```text
调用当前状态的 handler
    ↓
根据 consumed_length 移走已消费字节
    ↓
如果仍有输入并且刚才确实消费了数据
    ↓
再次调用 handler
```

循环条件类似：

```c
while (input.length > 0 && consumed_length > 0)
```

这让同一批 TCP 数据可以安全跨越：

```text
HANDSHAKE → FULL_SYNC
FULL_SYNC → CATCH_UP
CATCH_UP → ONLINE
```

同时 `consumed_length == 0` 时必须退出本轮处理并等待更多网络数据，否则会形成无限循环。

## NtyCo 对普通文件调用的 hook

第一次进行 Snapshot 传输时出现了一个隐蔽问题：

```text
Primary 已经生成 Snapshot
FULLRESYNC 元数据也正确
stream_handler 确实被调用
但 Replica 的临时文件始终是 0 字节
```

继续检查发现 NtyCo 在 `nty_socket.c` 中实现了同名的：

```c
read()
write()
close()
```

这些函数原本用于让 socket I/O 自动配合协程调度，但链接后也会截获 Snapshot 模块对普通文件描述符的调用。

结果是 Snapshot 文件读取被误当成协程 socket 读取，文件数据没有按照预期进入复制流。

解决方式是在持久化模块操作普通文件描述符时绕过 NtyCo 的 socket hook，直接使用系统调用层完成文件 `read`、`write` 和 `close`。

修复后的日志显示：

```text
snapshot_reader_read result=41 remaining=0
replication: Snapshot stream completed
```

Replica 随后能够收到 41 字节 Snapshot、完成原子安装并加载到内存。

这个问题说明：

```text
相同的 fd 类型都是 int
但 socket fd 与普通文件 fd 的 I/O 语义并不相同
```

对 libc 系统调用做全局 hook 时，必须明确它是否能够区分两类描述符。

## Snapshot 安装回调

复制模块知道“Snapshot 已经完整接收”，但它不应该直接操作 Array、RBTree、Hash 和 Skiplist。

因此初始化复制模块时安装一个回调：

```c
typedef int (*kvs_snapshot_install_handler)(
    const char *snapshot_path
);
```

`replication.c` 只在正确时机调用它：

```text
Snapshot 接收完成
    ↓
临时文件成功 commit
    ↓
调用 snapshot_install_handler(path)
```

真正的 Engine 操作仍然位于 `kvstore.c`：

```text
销毁旧 Engine
    ↓
初始化空 Engine
    ↓
加载新的 snapshot.db
```

如果加载中途失败，内存里可能已经恢复了一部分 key。因此失败路径再次销毁并初始化 Engine，确保 Replica 不会对外提供“半套数据”。

函数指针在这里实现的是控制反转：

```text
replication.c 知道何时安装
kvstore.c 知道如何安装
```

复制模块不需要包含各个 Engine 的实现细节，kvstore 模块也不需要了解 Snapshot 网络接收过程。

## 全量同步验证

Primary 先写入：

```text
HSET fullsync_test primary
```

然后启动一个空目录中的 Replica。日志依次出现：

```text
replication: enter FULL_SYNC
replication: Snapshot installed, enter CATCH_UP
```

在 Replica 查询：

```text
HGET fullsync_test
```

返回：

```text
primary
```

这证明数据经过了完整路径：

```text
Primary 内存
→ replication.snapshot
→ TCP 分块发送
→ Replica 临时文件
→ snapshot.db
→ Replica 内存 Engine
```

这一阶段对应提交：

```text
92c9947 feat(replication): 实现 Snapshot 全量传输与安装
```

## 第七阶段：AOF 增量追赶

Snapshot 安装完成只表示 Replica 恢复到了 Snapshot 对应的历史时刻。

假设生成 Snapshot 时 Primary AOF 末尾是：

```text
offset = 27
```

Snapshot 传输期间 Primary 又接受了新写入，AOF 增长到：

```text
offset = 80
```

那么 Replica 仍缺少区间：

```text
[27, 80)
```

因此 Snapshot 中保存的 AOF offset 不是“文件传输进度”，而是这个 Snapshot 对应的数据边界。

## 为什么 Replica 本地 AOF 需要对齐 offset

Replica 安装的是 Primary 在 offset 27 时的数据状态。如果 Replica 本地 AOF 仍然从 0 开始追加后续命令，它的字节坐标就会与 Primary 不一致：

```text
Primary 新命令位置：27 之后
Replica 新命令位置：0 之后
```

以后 Replica 报告“我处理到 offset 50”时，两边的 50 将不再指向相同位置。

因此安装 Snapshot 后执行：

```c
kvs_aof_reset_to_offset(snapshot_offset);
```

它通过 `ftruncate()` 把 Replica 本地 AOF 调整到相同长度：

```text
本地文件较短 → 扩展
本地文件较长 → 截断
```

然后把 stdio 写入位置移动到文件末尾。后续从 Primary 收到的增量命令，就从相同 offset 继续追加。

Replica 不需要复制 Primary offset 之前的完整 AOF，因为这部分历史效果已经包含在 Snapshot 中。对齐的是复制坐标，不是重复传输已有历史。

## 独立 AOF reader

Primary 仍然使用全局 `aof_fp` 接受新的客户端写入。增量发送不能通过修改这个 FILE 指针的位置来读取历史，否则会干扰正常追加。

因此新增独立 reader：

```c
typedef struct {
    int fd;
    long long offset;
    long long remaining;
} kvs_aof_reader_t;
```

打开时固定本轮发送区间：

```text
[start_offset, end_offset)
```

读取使用 `pread()`：

```text
从指定 offset 读取
但不修改文件描述符自身的共享位置
```

每次读取后：

```text
offset    += result
remaining -= result
```

这样 AOF 可以一边由客户端写入路径继续追加，一边由复制 reader 按固定区间读取。

## 为什么 AOF 使用按行协议

当前 AOF 内容本身就是可重放命令：

```text
SET key value\n
HSET key value\n
DEL key\n
```

它与复制握手控制行的区别是：

```text
网络控制协议 → 通常以 \r\n 结束
AOF 文件      → 以 \n 结束
```

原来的批处理器只识别 CRLF，直接发送 AOF 后，Replica 会一直等待不存在的 `\r`。

因此通用行解析器被扩展为同时识别：

```text
\r\n
\n
```

解析器除了返回行正文结束位置，还返回分隔符长度：

```text
CRLF → delimiter_length = 2
LF   → delimiter_length = 1
```

消费输入时执行：

```text
request_offset += frame_length + delimiter_length
```

这样握手控制消息和 AOF 命令可以复用同一个批量逐行处理框架。

## 复制命令执行回调

Replica 收到增量 AOF 后，需要真正执行 `SET`、`HSET`、`DEL` 等命令。

但 `replication.c` 不应该直接依赖所有 KV Engine，因此又安装一个函数指针：

```c
typedef int (*kvs_replication_command_handler)(
    char *msg,
    int length,
    char *response
);
```

调用关系是：

```text
replication.c
    → 从复制流拆出一条 AOF 命令
    → 调用 command_handler

kvstore.c
    → 标记来源为 REPLICATION
    → 执行真正命令
    → 更新 Replica 内存
    → 追加 Replica 本地 AOF
```

执行成功应返回：

```text
OK\r\n
```

复制帧处理器验证命令执行成功后，对 Primary 返回的网络响应长度仍然是 0：

```text
命令响应 OK\r\n → 只供内部判断执行是否成功
返回 0          → 不把 OK 再发送给 Primary
```

这里的 0 不是“没有执行命令”，而是“没有需要写回网络的响应”。

## 从 Snapshot 切换到 AOF

Primary 的流式输出函数首先读取 Snapshot：

```text
snapshot_reader.fd >= 0
    → 输出下一块 Snapshot
```

当 Snapshot reader 返回 0 时，说明文件已经发送完成。Primary 此时重新取得最新 AOF 末尾：

```text
catch_up_end = kvs_aof_get_offset()
```

然后打开固定增量区间：

```text
[snapshot_offset, catch_up_end)
```

这里不能立刻把 0 返回给网络层。对于 `stream_handler` 来说，返回 0 原本意味着整个流结束，网络层不会再次调用它。正确做法是在同一次流任务中从 Snapshot 阶段继续进入 AOF 阶段。

因此整体过程是：

```text
读取 Snapshot
    ↓ Snapshot 完成
打开第一轮 AOF reader
    ↓
读取并发送 AOF
```

## 追赶期间 Primary 继续写入怎么办

第一轮 AOF 发送期间，Primary 仍可能继续增长。

每一轮 reader 只读取打开时确定的固定区间。读完后记录：

```text
sent_offset
```

然后再次读取 Primary 最新 AOF 末尾：

```text
latest_offset
```

根据两者关系决定下一步：

```text
latest_offset < sent_offset
    → AOF 被异常截断，同步失败

latest_offset > sent_offset
    → 又产生了新命令
    → 打开 [sent_offset, latest_offset) 继续追赶

latest_offset == sent_offset
    → 已经追平
    → 发送 ONLINE <offset>
```

这形成一个有限的追赶循环。只要 Replica 的网络发送速度最终能够超过 Primary 的持续写入速度，它就能缩小差距并进入 ONLINE。

如果 Primary 写入速度始终高于 Replica 的接收与执行速度，Replica 就会一直处于 `CATCH_UP`。这不是状态机错误，而是系统吞吐能力不足。

## ONLINE 状态

`ONLINE` 表示：

```text
Snapshot 已安装
AOF 历史欠账已补齐
Replica 当前与 Primary 对齐
复制连接仍然保持
```

它不表示复制工作结束。进入 ONLINE 后，Primary 新产生的写命令仍然需要沿同一条连接持续发送。

Replica 收到：

```text
ONLINE <offset>\r\n
```

后保存新的复制 offset，并把状态从 `CATCH_UP` 切换为 `ONLINE`。

后续 AOF 命令仍复用增量命令处理器。它被允许在两种状态下工作：

```text
CATCH_UP
ONLINE
```

区别只在于：

```text
CATCH_UP → 正在偿还历史欠账
ONLINE   → 正在接收刚产生的实时写入
```

## 流结束与暂时没数据不是一回事

Snapshot 是有限文件，读完后确实结束；ONLINE 复制则是长期数据流。

当 Primary 暂时没有新 AOF 时，如果继续返回 0，网络层会理解为：

```text
整个 stream 已经结束
```

以后 Primary 即使出现新写入，也不会再次检查 AOF。

因此流式回调增加“暂时无数据”的状态，例如：

```c
#define KVS_STREAM_WAIT (-2)
```

新的返回语义是：

```text
> 0              → 有数据需要发送
KVS_STREAM_WAIT  → 当前没数据，但流仍然存活
0                → 流真正结束
-1               → 发生错误
```

NtyCo 网络层收到 `KVS_STREAM_WAIT` 后短暂休眠，再次调用 stream handler 检查是否出现新的 AOF。

当前选择 50ms 作为简单轮询间隔：

```text
延迟不会太高
也不会在无数据时持续占满 CPU
```

这属于定时轮询，不是 TTL，也不是心跳探活。它检查的是“是否出现新的复制数据”，没有用来判断对端是否存活。

## NtyCo 非零 sleep 的调度 bug

加入 ONLINE 轮询后出现了新的现象：

```text
Primary 和 Replica 已经进入 ONLINE
但客户端向 Primary 发送命令后没有任何响应
```

本地请求最终得到：

```text
timeout exit code = 124
```

问题不在 socket，也不在复制状态机，而在 NtyCo 的：

```c
nty_coroutine_sleep(uint64_t msecs)
```

原实现中，`msecs == 0` 的路径会调用：

```c
nty_coroutine_yield(co);
```

但非零路径只执行：

```c
nty_schedule_sched_sleepdown(co, msecs);
```

`nty_schedule_sched_sleepdown()` 只把协程插入 sleeping 红黑树并标记为 `SLEEPING`，函数末尾甚至留有：

```c
//yield
```

它没有真正切换回调度器。因此复制协程虽然登记了 50ms 休眠，实际上仍继续执行下一轮循环，形成持续空转，并饿死同一线程中的其他协程。

修复只需要一行：

```c
} else {
    nty_schedule_sched_sleepdown(co, msecs);
    nty_coroutine_yield(co);
}
```

修复后：

```text
复制协程进入 sleeping 队列
    ↓
主动让出执行权
    ↓
其他连接协程正常运行
    ↓
50ms 到期后复制协程重新被调度
```

该修复已经向 NtyCo 上游提交 PR：

```text
https://github.com/wangbojing/NtyCo/pull/18
```

PR 标题为：

```text
fix: 非零时长协程休眠后让出执行权
```

## ONLINE 实时同步验证

Primary 和 Replica 都进入 ONLINE 后，向 Primary 写入：

```text
HSET live_test online_value
```

Primary 正常返回：

```text
OK
```

Replica 随后收到并执行同一条命令。再向 Replica 查询：

```text
HGET live_test
```

返回：

```text
online_value
```

至此三条关键路径均已跑通：

```text
Primary 启动前已有数据
    → Snapshot 全量同步

Snapshot 之后产生的数据
    → AOF CATCH_UP

追平后产生的新数据
    → ONLINE 实时同步
```

这一阶段对应提交：

```text
97e0781 feat(replication): 实现 AOF 增量追赶与在线同步
```

## 当前复制架构

截至目前，NtyCo 后端上的完整路径是：

```text
Replica 连接 Primary
    ↓
PING / PONG
    ↓
PSYNC ? -1
    ↓
FULLRESYNC <id> <offset> <snapshot-size>
    ↓
Snapshot 分块传输与原子安装
    ↓
AOF [snapshot-offset, current-offset) 增量追赶
    ↓
ONLINE <offset>
    ↓
持续发送新追加的 AOF
```

模块职责保持为：

```text
kvstore.c
    → 命令执行、Engine 生命周期、Snapshot 安装

replication.c
    → 复制协议、状态机、发送阶段选择

persistence/snapshot.c
    → Snapshot 保存、reader、writer、原子提交

persistence/aof.c
    → AOF 追加、offset 对齐、固定区间 reader

network/buffer.c
    → LF / CRLF 帧拆分与输入消费

network/ntyco.c
    → 连接、收发、partial send、流式等待
```

## 当前完成边界

已经完成：

```text
✅ 三种服务器角色与启动参数
✅ Replica 客户端只读限制
✅ 独立复制端口
✅ Replica 主动连接 Primary
✅ PING / PONG 与 PSYNC 握手
✅ replication ID 与 FULLRESYNC
✅ Snapshot 元数据协商
✅ Snapshot 分块传输
✅ 临时文件落盘与原子安装
✅ Snapshot 加载进 Replica Engine
✅ Replica 本地 AOF offset 对齐
✅ AOF 固定区间增量追赶
✅ CATCH_UP → ONLINE
✅ ONLINE 后持续复制新写入
✅ NtyCo 环境下的一主一从端到端验证
```

尚未完成：

```text
❌ 自动化主从集成测试
❌ Replica 断线后的自动重连
❌ Primary 重启后的恢复策略
❌ 多 Replica 独立状态管理
❌ ACK 与已确认 offset
❌ replication backlog
❌ 根据 ID 与 offset 进行部分重同步
❌ Reactor 网络后端复制适配
❌ Proactor 网络后端复制适配
```

当前的复制管理器仍然只保存一组：

```text
snapshot_connection_fd
snapshot_reader
aof_reader
```

所以它本质上还是一主一从实现。支持多 Replica 时，这些状态需要从全局单例字段下沉到每条 Replica 连接自己的上下文中。

## 三个网络后端意味着什么

项目目前有：

```text
Reactor
Proactor
NtyCo
```

主从复制核心不需要重写三次。以下模块可以直接共享：

```text
复制状态机
Snapshot reader / writer
AOF reader
FULLRESYNC 协议
CATCH_UP 与 ONLINE 逻辑
Engine 安装与命令执行回调
```

另外两个网络后端需要补的是适配能力：

```text
同时监听客户端端口与复制端口
Replica 主动连接 Primary
安装 open / message / stream / close 回调
处理 partial send
处理 KVS_STREAM_WAIT
处理连接断开与资源清理
```

因此后续顺序确定为：

```text
1. 为 NtyCo 主从链路增加自动化集成测试
2. 使用同一套测试固定当前正确行为
3. 将复制网络接口适配到 Reactor
4. 使用同一套测试验证 Reactor
5. 最后适配 Proactor
6. 再实现断线重连、多 Replica 与部分重同步
```

当前项目已经越过“主从复制能不能工作”的阶段，进入“如何让它稳定、可重复验证，并覆盖全部网络后端”的阶段。

## 自动化主从集成测试

手工使用多个终端和 Packet Sender 可以帮助观察协议过程，但不适合作为长期回归手段。每次修改网络层后，如果都要手动完成下面的操作：

```text
启动 Primary
    ↓
写入同步前数据
    ↓
启动 Replica
    ↓
等待 FULLRESYNC
    ↓
检查 Snapshot 数据
    ↓
向 Primary 写入 ONLINE 数据
    ↓
再次查询 Replica
    ↓
停止两个进程并清理临时目录
```

不仅步骤多，而且很容易因为残留进程、端口占用或旧的持久化文件得到错误结论。

因此新增：

```text
tests/test_replication.sh
```

脚本会自动完成：

```text
创建相互隔离的 Primary / Replica 临时目录
启动两个 kvstore 进程
等待监听端口与复制状态就绪
验证全量 Snapshot 同步
验证 ONLINE 增量同步
检查命令返回结果
退出时停止后台进程
删除测试临时目录
```

日常回归命令简化为：

```bash
make
./tests/test_replication.sh
```

测试已经全部通过并提交。至此，“自动化主从集成测试”从上一节的未完成项中移除。

## 为什么 Reactor 仍然需要单独适配

复制状态机已经在 NtyCo 下跑通，并不等于 Reactor 能直接使用它。

共享的是业务能力：

```text
replication.c 决定现在应该处理握手、Snapshot 还是 AOF
persistence.c 负责生成或读取实际数据
kvstore.c 负责执行命令和安装 Snapshot
```

不同的是网络驱动方式。

NtyCo 可以在一个协程中写出接近顺序代码的逻辑：

```text
生成一块数据
    ↓
发送
    ↓
当前没有新数据就 sleep
    ↓
醒来后继续检查
```

Reactor 只有一个事件循环。它不能让某条连接停在函数中 sleep，否则同一个线程上的所有客户端都会被阻塞。它必须把一次完整复制拆成多个事件：

```text
EPOLLIN
    → 收到命令并生成控制响应

EPOLLOUT
    → 发送当前 output 缓冲区

output 发送完
    → 向 stream_handler 索取下一块数据

暂时没有新 AOF
    → 记录等待连接和重试时间
    → 返回 epoll 主循环
```

因此 Reactor 的适配不是重写主从复制，而是把已经完成的复制逻辑接入 epoll 的状态驱动模型。

## Reactor 多监听器改造

原来的 Reactor 启动接口是：

```c
int reactor_start(unsigned short port, msg_handler handler);
```

内部会从 `port` 开始创建一组连续端口，并且全部使用同一个全局 handler。这无法表达：

```text
service_port
    → kvs_network_protocol

replication_port
    → kvs_replication_network_protocol
```

因此引入统一的监听器配置：

```c
typedef struct {
    unsigned short port;
    msg_handler handler;
    kvs_connection_stream_handler stream_handler;
} kvs_listener_config_t;
```

新的 Reactor 入口接收配置数组：

```c
int reactor_start_listeners(
    const kvs_listener_config_t *listeners,
    size_t listener_count
);
```

Primary 传入两个监听器：

```c
kvs_listener_config_t listeners[] = {
    {
        .port = config.service_port,
        .handler = kvs_network_protocol,
        .stream_handler = NULL
    },
    {
        .port = config.replication_port,
        .handler = kvs_replication_network_protocol,
        .stream_handler = kvs_replication_stream
    }
};
```

这样配置表达的是两个明确的端口，而不是“从某个端口开始连续监听若干端口”。

## handler 为什么要从监听 fd 继承

epoll 返回的是发生事件的 fd，不会额外告诉程序：

```text
这个连接来自客户端端口
还是来自复制端口
```

因此监听 socket 创建时先保存自己的业务标签：

```c
conn_list[sockfd].handler = listeners[i].handler;
conn_list[sockfd].stream_handler = listeners[i].stream_handler;
```

`accept()` 得到新连接后，再继承监听 fd 的标签：

```c
conn_list[clientfd].handler = conn_list[fd].handler;
conn_list[clientfd].stream_handler =
    conn_list[fd].stream_handler;
```

例如：

```text
监听 fd 5：service_port
    handler = kvs_network_protocol
        ↓ accept
客户端 fd 8 继承 kvs_network_protocol

监听 fd 6：replication_port
    handler = kvs_replication_network_protocol
    stream_handler = kvs_replication_stream
        ↓ accept
Replica fd 9 继承两个复制回调
```

随后 `kvs_request()` 不再调用全局 handler，而是使用当前连接自己的：

```c
c->output.length = c->handler(
    c->fd,
    c->input.data,
    c->input.length,
    c->output.data,
    BUFFER_LENGTH,
    &consumed_length
);
```

网络层仍然不知道 `SET`、`PSYNC` 或 `FULLRESYNC` 的含义，它只根据连接上保存的函数指针把数据交给正确的业务协议。

## Reactor 接入流式输出

原来的 `send_cb()` 只支持普通请求响应：

```text
发送 output
    ↓
output 清空
    ↓
切回 EPOLLIN，等待下一条请求
```

复制连接不能在发送完 `FULLRESYNC` 后直接等待输入，因为 Replica 正在等 Primary 主动发送 Snapshot。双方都会等待对方，连接看起来正常，却不再产生任何数据。

因此在 output 发送完之后加入：

```c
reactor_prepare_stream_output(&conn_list[fd]);
```

这个辅助函数只负责向 `stream_handler` 索取下一块数据：

```text
返回值 > 0
    → 下一块 Snapshot/AOF 已放入 output
    → 继续关注 EPOLLOUT

返回值 = 0
    → 本轮数据流结束

返回值 = -1
    → 生成数据失败，关闭连接

返回值 = KVS_STREAM_WAIT
    → 已进入 ONLINE，但当前暂时没有新 AOF
```

因此 Primary 的发送过程变成：

```text
handler 生成 FULLRESYNC
    ↓
send_cb 发送 FULLRESYNC
    ↓
stream_handler 生成一块 Snapshot
    ↓
send_cb 发送 Snapshot
    ↓
继续生成下一块
    ↓
Snapshot → CATCH_UP AOF → ONLINE
```

## EPOLLOUT 的真实含义

`EPOLLOUT` 不是“对端已经收到数据”，也不是“业务层存在数据”。它只表示：

```text
当前 socket 的内核发送缓冲区还有空间，
可以尝试调用 send()。
```

执行：

```c
set_event(fd, EPOLLOUT, 0);
```

只是告诉 epoll 关注该 fd 的可写状态。真正的数据发送仍然发生在：

```c
send_cb(fd);
```

而且一次 `send()` 仍可能只发送部分字节，因此 Reactor 继续使用：

```c
output.offset += count;
```

记录尚未发送的区间。

socket 在大多数时候都是可写的。如果 ONLINE 状态没有新 AOF，却一直关注 `EPOLLOUT`，就会形成：

```text
epoll 报告可写
    ↓
stream_handler 发现没有数据
    ↓
再次关注 EPOLLOUT
    ↓
epoll 立即再次报告可写
```

这会造成 CPU 空转，所以 `KVS_STREAM_WAIT` 不能直接继续监听 `EPOLLOUT`。

## Reactor 中的 ONLINE 定时轮询

NtyCo 可以让复制协程 sleep，Reactor 则使用 `epoll_wait()` 的 timeout 实现同样的等待效果。

当前一主一从模型只维护一条复制流，因此记录：

```c
static int stream_waiting_fd = -1;
static long long stream_retry_at_ms = -1;
```

两个值分别表示：

```text
stream_waiting_fd
    → 哪一条复制连接正在等待

stream_retry_at_ms
    → 到什么时间才允许再次检查
```

例如：

```text
当前复制 socket 为 fd 8
当前时间为 10000ms

stream_waiting_fd = 8
stream_retry_at_ms = 10100
```

Reactor 主循环原来使用：

```c
epoll_wait(epfd, events, 1024, -1);
```

`-1` 表示没有网络事件就永远等待。存在 ONLINE 等待任务时，需要把 timeout 改成距离重试时间剩余的毫秒数：

```c
long long remaining_ms =
    stream_retry_at_ms - now_ms;

wait_timeout =
    remaining_ms > 0 ? (int)remaining_ms : 0;
```

这里使用绝对截止时间，而不是每一轮固定等待完整的 100ms。原因是普通客户端事件可能提前唤醒 `epoll_wait()`：

```text
计划在 1100ms 检查复制数据
    ↓
1050ms 时客户端 fd 先发生事件
    ↓
处理完成后只剩 50ms
    ↓
下一次 epoll_wait 应等待50ms，而不是重新等待100ms
```

到达重试时间后：

```c
int waiting_fd = stream_waiting_fd;

stream_waiting_fd = -1;
stream_retry_at_ms = -1;

set_event(waiting_fd, EPOLLOUT, 0);
```

旧闹钟必须先清除。如果再次调用 `stream_handler` 后仍然没有新 AOF，`send_cb()` 会设置一个新的 100ms 闹钟。

最终循环为：

```text
没有新 AOF
    ↓
记录 fd 与100ms后的截止时间
    ↓
epoll_wait 等网络事件或等待到期
    ↓
到期后将该 fd 切回 EPOLLOUT
    ↓
send_cb 再次检查 AOF
    ├── 有数据：发送数据
    └── 没数据：重新设置下一次闹钟
```

这仍然属于定时轮询，不是心跳，也不是 TTL。当前轮询间隔为 100ms，因此 ONLINE 复制属于近实时同步，最坏会引入约一个轮询周期的额外延迟。

## Reactor Primary 端到端验证

临时将：

```c
#define NETWORK_SELECT NETWORK_REACTOR
```

启动 Primary：

```bash
kvstore primary 19000 19100
```

两个监听器均成功启动：

```text
listen port : 19000
listen port : 19100
```

先向普通客户端端口写入：

```text
HSET reactor_test primary_value
```

返回：

```text
OK
```

再使用 Packet Sender 在复制端口发送：

```text
PSYNC ? -1
```

同一条持久 TCP 连接依次收到：

```text
FULLRESYNC cc1a88c488f7647f8cec7a2c5b16f3f171a6324f 32 46
AOF_OFFSET 32
HSET reactor_test primary_value
ONLINE 32
```

这证明 Reactor Primary 已经能够发送：

```text
FULLRESYNC 控制帧
Snapshot 内容
固定区间 AOF
ONLINE 控制帧
```

保持复制连接不关闭，再向客户端端口写入：

```text
HSET reactor_live after_online
```

复制连接在轮询周期内收到：

```text
HSET reactor_live after_online
```

因此 Reactor Primary 的 ONLINE 增量输出已经跑通。

## Reactor 当时的阶段性边界

在完成 Primary 发送侧、尚未改造 Replica 主动连接器时，Reactor 的阶段性进度是：

```text
✅ 多监听器配置
✅ 客户端端口与复制端口绑定不同 handler
✅ accept 后继承 handler
✅ accept 后继承 stream_handler
✅ FULLRESYNC 响应发送
✅ Snapshot 分块发送
✅ AOF CATCH_UP 发送
✅ ONLINE 定时轮询
✅ ONLINE 新增 AOF 实测发送成功
```

尚未完成的是 Replica 接收侧：

```text
❌ Reactor Replica 主动 connect Primary
❌ 连接成功后调用 open_handler 生成 PING
❌ 收到上游数据后调用 message_handler
❌ 断开后调用 close_handler
❌ Reactor 下真正的一主一从端到端验证
```

刚才的验证中，Packet Sender 暂时扮演了 Replica。下一阶段需要让 Reactor Replica 自己完成：

```text
创建 socket
    ↓
connect Primary
    ↓
调用复制 open_handler
    ↓
发送 PING / PSYNC
    ↓
通过 message_handler 接收 Snapshot 与 AOF
    ↓
断开时通知复制管理器清理状态
```

换句话说，复制协议本身已经不需要重写。接下来要补的是 Reactor 的主动连接器能力。

## Reactor Replica：主动连接 Primary

NtyCo 版本中，Replica 已经能够主动连接 Primary，但 Reactor 最初只有监听器，没有连接器。也就是说，它能够接受别人连接，却不能主动连接远端服务器。

Replica 同时需要完成两类网络行为：

```text
Replica
├── 被动监听 service_port
│   └── 接受客户端 HGET 等读请求
│
└── 主动连接 Primary replication_port
    └── 请求并持续接收 Snapshot/AOF
```

因此为 Reactor 增加了新的运行时入口：

```c
int reactor_start_runtime(
    const kvs_listener_config_t *listeners,
    size_t listener_count,
    const kvs_connector_config_t *connectors,
    size_t connector_count
);
```

这里有两组配置：

```text
listeners
    → 本进程需要监听哪些端口

connectors
    → 本进程需要主动连接哪些远端地址
```

对于 Reactor Replica，调用关系为：

```text
一个 listener
    → 监听 19001 客户端端口
    → handler = kvs_network_protocol

一个 connector
    → 主动连接 127.0.0.1:19100
    → open_handler    = 复制连接建立处理器
    → message_handler = Primary 消息处理器
    → close_handler   = 复制连接断开处理器
```

Standalone 不需要主动连接，因此旧接口仍然可以表示：

```c
reactor_run(
    listeners,
    listener_count,
    NULL,
    0
);
```

其中 `NULL, 0` 表示没有任何主动连接任务。

为了兼容旧调用者，原来的：

```c
reactor_start(port, handler);
```

也被保留下来。它只负责构造一个 listener，然后转调新的多监听器入口，不再维护第二套事件循环。

## 主动连接器的职责

Reactor 的连接器按照以下步骤启动：

```text
读取 connector 配置
    ↓
socket()
    ↓
inet_pton() 解析 Primary IPv4 地址
    ↓
connect() 连接复制端口
    ↓
event_register() 注册进 epoll
    ↓
保存 message_handler 和 close_handler
    ↓
调用 open_handler() 生成 PING
    ↓
切换到 EPOLLOUT
    ↓
复用 send_cb() 发送 PING
```

建立 TCP 连接的逻辑单独放进：

```c
reactor_open_connector_socket()
```

这个函数只负责：

```text
创建 socket → 解析地址 → connect → 返回 fd
```

它不理解复制状态，也不直接发送 `PING`。连接成功后的业务行为仍然由 connector 中的函数指针决定。

`open_handler()` 生成的 `PING` 不直接调用 `send()`，而是写入连接自身的 output 缓冲区：

```text
open_handler 生成 PING
    ↓
conn_list[fd].output 保存 PING
    ↓
关注 EPOLLOUT
    ↓
send_cb() 统一处理部分发送
```

这样主动连接和被动接受的连接共用相同的发送机制，不需要维护第二套 socket 输出代码。

## 每条连接保存自己的生命周期回调

为了让 Reactor 能够管理普通客户端、复制下游连接和 Replica 上游连接，`struct conn` 最终需要保存：

```c
msg_handler handler;
kvs_connection_stream_handler stream_handler;
kvs_connection_close_handler close_handler;
```

三者分别解决：

```text
handler
    → 收到数据后应该按什么协议解析

stream_handler
    → 普通响应发送完后，是否还要继续生成 Snapshot/AOF

close_handler
    → 连接关闭后，是否需要通知业务模块清理状态
```

普通客户端连接没有特殊的断开业务，因此：

```c
conn_list[clientfd].close_handler = NULL;
```

Replica 到 Primary 的主动连接则保存 replication 模块提供的 `close_handler`。当上游连接断开时，Reactor 负责释放网络资源，随后通知 replication 模块恢复复制状态。

所有已注册连接的关闭路径被收敛到：

```c
reactor_close_connection(fd);
```

该函数统一执行：

```text
从 epoll 删除 fd
    ↓
关闭 socket
    ↓
取消该连接的 ONLINE 等待闹钟
    ↓
清空 conn_list 中的连接状态
    ↓
最后调用保存下来的 close_handler
```

需要先保存 `close_handler`，再清空连接对象。否则清空：

```c
conn_list[fd].close_handler = NULL;
```

后就无法通知业务层。先清理、后回调也可以避免回调重入时看到一条仍处于半关闭状态的旧连接。

## Reactor Replica 联调时发现的 TCP 合包问题

第一次真正启动 Reactor Replica 时，日志停在：

```text
replication: enter FULL_SYNC ...
replication: Snapshot installed, enter CATCH_UP ...
```

却没有出现：

```text
replication: enter ONLINE ...
```

Primary 日志已经明确显示：

```text
replication: AOF catch-up completed, enter ONLINE offset=33 fd=7
```

因此问题不在 Primary，也不是 Snapshot/AOF 算法错误，而是 Replica 没有继续处理已经读入用户态缓冲区的数据。

TCP 是字节流，不保留应用层消息边界。Primary 连续发送：

```text
[Snapshot 最后一块][ONLINE 33\r\n]
```

Replica 的一次 `recv()` 完全可能同时得到两部分：

```text
input buffer:
[47 字节 Snapshot][ONLINE 33\r\n]
```

当时 Replica 处于 `FULL_SYNC`，所以第一次协议调用只会：

```text
消费47字节 Snapshot
    ↓
安装 Snapshot
    ↓
状态切换为 CATCH_UP
```

此时 input buffer 中仍然保存：

```text
ONLINE 33\r\n
```

旧版 Reactor 的 `recv_cb()` 每次 EPOLLIN 只调用一次 `kvs_request()`，随后退出。但 `ONLINE` 已经从内核 socket 进入程序自己的缓冲区，内核中没有新的未读数据，因此 epoll 不会再次报告 EPOLLIN。

结果相当于：

```text
快递员一次送来两个包裹
程序只拆了第一个
第二个已经放在屋里
程序却继续等待快递员再次敲门
```

修复方法是在一次 `recv()` 后，只要满足以下条件，就继续处理当前输入缓冲区：

```text
没有响应等待发送
并且输入缓冲区仍有数据
并且上一轮确实消费了数据
```

伪代码为：

```c
while (1) {
    int previous_input_length = input.length;

    process_current_buffer();

    if (output.length > 0) break;
    if (input.length == 0) break;
    if (input.length >= previous_input_length) break;
}
```

最后一个条件非常重要。如果剩余输入还不是完整消息，本轮长度不会减少，此时必须退出并等待下一次 `recv()`，否则会在用户态形成无限循环。

修复后的同一次 `recv()` 会经历：

```text
第一轮：FULL_SYNC 处理 Snapshot
    ↓
状态变为 CATCH_UP，缓冲区仍有 ONLINE
    ↓
第二轮：CATCH_UP 处理 ONLINE
    ↓
状态变为 ONLINE
```

这再次说明：TCP 只提供连续字节，不提供“一个 send 对应一个 recv”的消息边界。协议层必须显式维护输入缓冲区、消费长度和状态转换。

## Reactor 一主一从端到端验证

Primary 启动后监听：

```text
listen port : 19000
listen port : 19100
```

在 Replica 连接前，先写入：

```text
HSET reactor_full before_replica
```

Replica 启动后成功输出：

```text
listen port : 19001
connect to primary 127.0.0.1:19100
replication: enter FULL_SYNC ... offset=33 snapshot_size=47
replication: Snapshot installed, enter CATCH_UP ...
replication: enter ONLINE offset=33 ...
```

从 Replica 服务端口读取：

```text
HGET reactor_full
```

返回：

```text
before_replica
```

证明连接前已有数据已经通过 Snapshot 完成全量同步。

随后在 Primary 的 ONLINE 阶段写入：

```text
HSET reactor_live after_online
```

再从 Replica 读取：

```text
HGET reactor_live
```

返回：

```text
after_online
```

证明 Reactor 下的持续 AOF 增量同步也已经完成。

自动化复制测试最终全部通过：

```text
[PASS] Primary started
[PASS] Initial data written to Primary
[PASS] Replica entered ONLINE
[PASS] Snapshot full synchronization
[PASS] ONLINE incremental synchronization

replication integration test passed
```

恢复默认的 NtyCo 网络框架并重新编译后，同一套自动化测试仍然全部通过。因此本次 Reactor 改造没有破坏已经完成的 NtyCo 主从复制。

## 当前复制属于异步复制

当前写入路径是：

```text
客户端写入 Primary
    ↓
Primary 修改内存并写本地 AOF
    ↓
Primary 向客户端返回 OK
    ↓
Replica 稍后读取并执行新的 AOF 命令
```

Primary 返回 `OK` 前不会等待 Replica 确认，因此当前实现属于异步复制，而不是同步复制。

Reactor/NtyCo/Proactor 描述的是网络 I/O 模型；同步复制/异步复制描述的是 Primary 是否等待 Replica 的确认。二者不是同一个概念：

```text
Reactor + 异步复制
NtyCo   + 异步复制
```

都是合理组合。

异步复制存在一个明确窗口：

```text
Primary 已向客户端返回 OK
    ↓
数据尚未发送到 Replica
    ↓
Primary 此时宕机
```

故障切换后，这条已确认写入可能丢失。

同步复制则需要新增完整的确认链路：

```text
Primary 执行写命令
    ↓
发送给 Replica
    ↓
Replica 执行并持久化
    ↓
Replica 返回 ACK offset
    ↓
Primary 找到对应客户端请求
    ↓
Primary 才返回 OK
```

这还会引入超时、降级、多 Replica 确认策略、ACK 丢失和写入延迟等问题。因此当前阶段先完成异步复制是合理的工程顺序。

## 为什么解耦之后三个网络框架仍然需要大量改造

解耦并不意味着网络层不需要修改，而是复制业务逻辑只实现一次，各网络框架分别实现同一套网络能力。

原网络层主要面向：

```text
监听一个端口
    ↓
接收客户端命令
    ↓
调用一个全局 handler
    ↓
返回一次响应
```

主从同步要求的网络能力明显更多：

```text
一个进程监听多个端口
不同端口绑定不同协议
Replica 主动连接 Primary
一条连接持续流式发送 Snapshot/AOF
暂时无数据时等待并再次唤醒
连接断开时通知复制模块
一次 recv 中处理跨协议阶段的数据
```

这些不是 Snapshot 或 AOF 的业务算法，而是原网络抽象此前没有覆盖的通用能力。

当前解耦边界为：

```text
replication.c
├── 不知道 epoll
├── 不知道协程调度
├── 不知道 io_uring
└── 只提供 open/message/stream/close 回调

网络框架
├── 不知道 PSYNC 的业务含义
├── 不知道 Snapshot 文件格式
├── 不知道 AOF offset 规则
└── 只负责连接、等待、收发字节和调用回调
```

三个框架仍然必须分别改造，是因为等待和唤醒机制不同：

```text
NtyCo
    → 协程 sleep/yield

Reactor
    → epoll_wait timeout + EPOLLOUT

Proactor
    → io_uring 完成事件
```

无法复用的是“如何等待、如何被唤醒、如何提交和完成 I/O”；已经复用的是全部复制状态机与 Snapshot/AOF 业务规则。

这次改动较大，本质上是在支付一次网络基础设施升级的成本。多监听器、每连接 handler、主动 connector、流式输出、WAIT、close 回调和输入缓冲连续处理完成后，未来的数据迁移、订阅推送或节点通信都可以复用这些能力。

## Reactor 阶段完成

最终提交为：

```text
c3a88ec feat(network): 支持 Reactor 主从复制
```

当前总体进度：

```text
✅ Snapshot 全量同步
✅ AOF 增量追赶
✅ ONLINE 持续异步复制
✅ Replica 本地 AOF offset 对齐
✅ NtyCo 一主一从端到端验证
✅ Reactor 一主一从端到端验证
✅ 自动化复制回归测试
⬜ Proactor 网络框架适配
```

下一阶段不需要重写复制协议，而是让 Proactor 实现已经确定的通用网络契约：

```text
多监听器
每连接 handler
主动 connector
stream_handler
KVS_STREAM_WAIT
close_handler
输入缓冲连续消费
```

## 第八阶段：Proactor 主从复制适配

Reactor 完成后，复制业务协议已经稳定。Proactor 阶段的目标不是再实现一遍 PSYNC、Snapshot 或 AOF，而是让 io_uring 网络后端满足同一套通用网络契约。

改造前的 Proactor 只支持最简单的单端口请求响应模型：

```text
一个监听 fd
    ↓ accept
一个 client fd
    ↓ recv 完成
调用全局 kvs_handler
    ↓ send 完成
重新提交 recv
```

这种结构无法直接支持主从同步，原因包括：

```text
Primary 需要同时监听 service_port 和 replication_port
两个端口需要绑定不同的 handler
Replica 需要主动 connect Primary
复制连接需要连续发送 Snapshot/AOF
ONLINE 暂时无数据时不能把复制流判定为结束
连接断开时需要通知 replication.c
一次 recv 得到的数据可能跨越多个复制状态
```

因此 Proactor 的改造顺序仍然遵循此前确定的原则：

```text
先让连接拥有自己的协议
    ↓
再支持多个监听器
    ↓
再支持流式输出与定时重试
    ↓
最后支持 Replica 主动连接 Primary
```

## io_uring 中的提交与完成

Reactor 询问的是：

```text
“哪个 fd 现在可以读或可以写？”
```

Proactor 的思路则是：

```text
“我要对这个 fd 执行一次 accept/recv/send/timeout，
操作完成后再把结果告诉我。”
```

初始化代码：

```c
struct io_uring_params params;
memset(&params,0,sizeof(params));

struct io_uring ring;
io_uring_queue_init_params(
    ENTRIES_LENGTH,
    &ring,
    &params
);
```

可以把 `ring` 理解为用户态和内核态共同使用的一套异步任务队列：

```text
SQ（Submission Queue）
    程序向内核提交 accept/recv/send/timeout 请求

CQ（Completion Queue）
    内核把已经完成的请求结果返回给程序
```

一次事件循环的核心过程是：

```text
准备 SQE
    ↓
io_uring_submit
    ↓
内核完成 I/O
    ↓
生成 CQE
    ↓
程序根据 user_data 判断完成的是哪类操作
```

因此 `EVENT_ACCEPT`、`EVENT_READ`、`EVENT_WRITE` 并不是业务状态，而是用来标记 CQE 类型的网络调度信息。

## 为什么监听 fd 也需要保存 connection

多个监听端口出现后，全局 `kvs_handler` 已经不够用了：

```text
19000 service listener
    → kvs_network_protocol

19100 replication listener
    → kvs_replication_network_protocol
    → kvs_replication_stream
```

accept 的完成事件中，`result.fd` 是产生这次连接的监听 fd，`entries->res` 才是新建立的 client fd：

```text
result.fd   = listener fd
entries->res = accepted client fd
```

因此监听 fd 需要保存自己绑定的协议配置：

```c
listener_connection->fd = sockfd;
listener_connection->handler = listener->handler;
listener_connection->stream_handler =
    listener->stream_handler;
```

accept 完成后，新连接从监听连接继承这些处理器：

```c
connection->fd = connfd;
connection->handler = listener->handler;
connection->stream_handler =
    listener->stream_handler;
```

这里的 `listener_connection` 并不是新的结构体类型，它仍然是 `struct conn`。变量名只表示它当前保存的是监听 fd 的运行时状态。

真正关键的关系是：

```text
监听 fd 决定“这是什么端口”
    ↓
accepted fd 继承“这个端口使用什么协议”
```

这样事件循环只需要根据 fd 找到对应的 `struct conn`，不需要在每次读事件中判断端口号。

## Proactor 多监听器

提取 `proactor_register_listener()` 后，每个监听器的注册过程变成一个独立步骤：

```text
读取 listener config
    ↓
创建并 bind/listen socket
    ↓
创建 listener connection
    ↓
保存 handler/stream_handler
    ↓
提交 ACCEPT SQE
```

`proactor_run()` 只创建一个 io_uring，然后把所有监听器注册进去：

```c
for(size_t i=0;i<listener_count;i++){
    proactor_register_listener(
        &ring,
        &listeners[i]
    );
}
```

Primary 因此可以在同一个事件循环中同时处理两个监听端口：

```text
service_port
    → 普通客户端命令

replication_port
    → PSYNC / Snapshot / AOF
```

多个监听器不是多个 io_uring，也不是多个线程；它们共享同一个 ring 和同一个完成事件循环。

## start、start_listeners 与 start_runtime

Proactor 最终保留三层启动接口：

```text
proactor_start
    一个监听器
    兼容原来的 standalone 调用

proactor_start_listeners
    多个监听器
    Primary 使用

proactor_start_runtime
    多个监听器 + 多个主动连接器
    Replica 使用
```

这三个接口最后都进入同一个核心：

```text
proactor_run(
    listeners,
    listener_count,
    connectors,
    connector_count
)
```

`runtime` 并不是一种新的网络模型。它表达的是完整运行时启动需求：

```text
被动监听客户端
        +
主动连接上游 Primary
        +
进入同一个事件循环
```

Replica 必须在事件循环开始前同时完成这两类注册，因为事件循环进入后会长期等待 CQE，不会再返回启动代码继续创建上游连接。

## connector 与 connection 的区别

这两个名字很接近，但处在不同层次。

`kvs_connector_config_t` 是主动连接配置：

```text
host
port
open_handler
message_handler
close_handler
```

它描述“准备连接谁，以及连接生命周期各阶段调用谁”。

`struct conn` 是连接建立后的运行时对象：

```text
fd
input buffer
output buffer
handler
stream_handler
close_handler
```

它描述“这条已经存在的 TCP 连接现在处于什么状态”。

可以概括为：

```text
connector = 建立连接的蓝图
connection = 建立完成后的实例
```

Replica 启动时，`replication.c` 生成 connector config；Proactor 根据它创建并连接 socket，再把回调保存到 connection 中。

## Replica 主动连接 Primary

Proactor 的主动连接注册过程为：

```text
创建 socket
    ↓
connect Primary replication_port
    ↓
创建 struct conn
    ↓
安装 message_handler 和 close_handler
    ↓
调用 open_handler 生成首次 PING
    ↓
提交 SEND
    ↓
后续通过 READ/WRITE CQE 驱动复制状态机
```

当前 `connect()` 在启动阶段仍然是同步调用。它发生在事件循环开始之前，结构清晰，但如果 Primary 不可达，启动过程缺少异步连接、超时和自动重试能力。这属于后续可以继续演进的边界。

连接成功后，Proactor 不需要理解 `PING` 或 `PSYNC` 的含义：

```text
open_handler
    → replication.c 生成握手数据

message_handler
    → replication.c 消费 Primary 返回的数据

close_handler
    → replication.c 得知上游连接断开
```

## Proactor 中的流式输出

普通请求响应在一次 output 发送完成后就重新提交 RECV；复制连接则不同：

```text
FULLRESYNC 响应发送完成
    ↓
继续发送 Snapshot
    ↓
继续发送 AOF catch-up
    ↓
ONLINE 后继续等待新 AOF
```

因此 WRITE 完成后的逻辑变为：

```text
当前 output 是否全部发送完成？
    ├── 否：继续提交剩余 SEND
    └── 是：清空 output
             ↓
        调用 stream_handler
```

`proactor_schedule_stream_output()` 统一处理 stream handler 的四种返回值：

```text
> 0
    output 中已有下一块数据
    → 提交 SEND

0
    流已经结束
    → 重新提交 RECV

KVS_STREAM_WAIT (-2)
    流没有结束，只是暂时没有新 AOF
    → 提交 TIMEOUT

其他负数
    生成流数据失败
    → 关闭连接
```

这个辅助函数返回的是网络层调度结果：

```text
1  已经提交 SEND 或 TIMEOUT
0  当前没有流任务，可以回到 RECV
-1 调度失败
```

它把业务层的流语义转换成了 io_uring 可以执行的下一项操作。

## KVS_STREAM_WAIT 与 io_uring timeout

`KVS_STREAM_WAIT` 本身不属于 io_uring，它只是业务层和网络层之间的约定：

```text
复制尚未结束
但此刻没有新的 AOF 字节
```

如果收到 WAIT 后立刻反复调用 stream handler，会造成 CPU 空转。Proactor 因此向 ring 提交一个 timeout SQE：

```text
stream_handler 返回 WAIT
    ↓
提交 100ms timeout
    ↓
当前线程继续处理其他 CQE
    ↓
timeout 到期产生 EVENT_STREAM_RETRY
    ↓
重新调用 stream_handler
```

timeout 并没有脱离 io_uring。它与 ACCEPT、READ、WRITE 一样被提交到 SQ，完成后也从 CQ 返回。

区别只是它等待的不是 socket I/O，而是时间到期。

timeout 完成时 `entries->res` 通常是 `-ETIME`。对普通 I/O 来说负数通常表示失败，但对 timeout 来说 `-ETIME` 表示计时正常到期，因此不能把它当作连接错误。

如果重试后仍然没有 AOF：

```text
stream_handler 再次返回 WAIT
    ↓
再次提交一个新的 timeout
```

如果已经产生新 AOF：

```text
stream_handler 返回正数
    ↓
提交 SEND
```

这就形成 ONLINE 状态下的低频轮询。

## 为什么异步 I/O 仍然可能被 handler 阻塞

io_uring 使网络 I/O 异步，并不自动让业务处理并行。

当前每个 kvstore 进程仍然主要由一个事件循环线程处理 CQE：

```text
取出 CQE
    ↓
调用 connection->handler
    ↓
handler 返回
    ↓
处理下一个 CQE
```

因此如果某个 handler 执行时间很长：

```text
当前 CQE 长时间未处理完
    ↓
其他已经完成的 CQE 只能留在完成队列中等待
```

所以当前系统同时满足：

```text
网络 I/O 是异步的
复制协议是异步复制
业务 handler 不是多线程并行执行
```

“异步”必须明确它描述的是哪一层，不能简单等同于“所有代码同时运行”。

## 输入缓冲区为什么必须连续消费

Proactor Replica 联调时，同样遇到了 TCP 没有消息边界的问题。

一次 recv 可能同时得到：

```text
FULLRESYNC ...\r\n
AOF_OFFSET ...\r\n
Snapshot binary bytes
ONLINE ...\r\n
```

handler 只会消费当前状态能够解释的前缀，并通过 `consumed_length` 告诉网络层本次消费了多少字节。剩余数据仍然保存在用户态 input buffer 中。

错误做法是：

```text
调用一次 handler
    ↓
消费一部分 input
    ↓
直接提交下一次 RECV
```

如果内核接收缓冲区暂时没有新字节，下一次 READ 不会完成；而真正需要处理的 `ONLINE` 可能已经留在用户态 input buffer 里，于是状态机看起来像“忘了进入 ONLINE”。

正确做法是 `proactor_process_input()`：

```text
只要 input 中仍有数据
    ↓
调用当前 connection->handler
    ↓
根据 consumed_length 删除已消费前缀
    ↓
如果状态发生变化，下一轮自动使用新的状态解释剩余字节
```

循环停止条件包括：

```text
handler 生成了 output，需要先发送响应
consumed_length <= 0，说明当前数据还不完整
input 已经全部消费完
handler 返回错误
```

这里本质上涉及两层缓冲区：

```text
内核 socket receive buffer
    recv 完成后
用户态 connection->input buffer
```

READ CQE 只表示“内核已经把一批字节搬进用户缓冲区”，不表示“用户缓冲区中的所有完整协议帧都已经处理完”。

## 输出缓冲区与部分发送

输出也不能假设一次 SEND 就能发送全部字节。

```text
output.length
    本块数据总长度

output.offset
    已经成功发送的长度

remaining
    output.length - output.offset
```

WRITE CQE 返回正数后：

```text
offset += ret
```

如果 `offset < length`，只提交剩余部分；只有整块数据发送完成，才能清空 output 并向 stream handler 索取下一块。

因此输入和输出分别解决两个不同问题：

```text
input + consumed_length
    解决拆包、合包与跨状态消费

output + offset
    解决部分发送与流式续发
```

## Proactor 统一关闭路径

原 Proactor 在许多分支中重复执行：

```c
close(fd);
free(connection);
proactor_connections[fd] = NULL;
```

主从同步要求关闭连接时还必须调用业务层 `close_handler`。如果继续复制这些清理代码，很容易在某条错误路径中漏掉回调或重复释放。

因此提取统一的：

```text
proactor_close_connection(fd)
```

它负责：

```text
校验 fd 与 connection
保存 close_handler
清空 fd 映射
关闭 socket
释放 connection
最后通知业务层
```

先保存回调再释放 connection，是因为回调函数指针本身保存在即将释放的对象里。先清空映射则可以避免同一个 fd 被重复关闭。

网络层只负责资源生命周期；复制模块通过 close handler 更新复制状态。这个边界与 Reactor、NtyCo 保持一致。

## Proactor 一主一从端到端验证

Primary 使用两个监听端口启动：

```text
listen port : 19000
listen port : 19100
```

在 Replica 启动前，先向 Primary 写入：

```text
HSET proactor_full before_replica
```

Replica 随后主动连接 Primary：

```text
listen port : 19001
connect to primary 127.0.0.1:19100
replication: enter FULL_SYNC ...
replication: Snapshot installed, enter CATCH_UP ...
replication: enter ONLINE ...
```

Primary 侧对应输出：

```text
replication: FULLRESYNC response ready ...
replication: Snapshot stream completed ...
replication: AOF catch-up completed, enter ONLINE ...
```

从 Replica 查询 `proactor_full` 能够得到 `before_replica`，证明 Snapshot 全量同步成功。

在双方进入 ONLINE 后继续向 Primary 写入新键，再从 Replica 读取到该值，证明 io_uring timeout 能够重新唤醒复制流并发送新的 AOF 命令。

最终默认 NtyCo 后端下的自动化回归测试再次通过：

```text
[PASS] Primary started
[PASS] Initial data written to Primary
[PASS] Replica entered ONLINE
[PASS] Snapshot full synchronization
[PASS] ONLINE incremental synchronization

replication integration test passed
```

## 三种网络调度模型的最终对比

三种后端执行的是同一套复制状态机，但把 `KVS_STREAM_WAIT` 转换成不同的等待机制。

### NtyCo

```text
stream_handler 返回 WAIT
    ↓
当前连接协程 sleep/yield
    ↓
调度器运行其他协程
    ↓
协程醒来后重新检查 AOF
```

理解重点是协程的挂起和恢复。

### Reactor

```text
stream_handler 返回 WAIT
    ↓
记录 waiting fd 与 retry_at
    ↓
epoll_wait 使用剩余时间作为 timeout
    ↓
时间到达后重新关注 EPOLLOUT
    ↓
send callback 再次检查 AOF
```

理解重点是就绪通知、事件兴趣集合与定时唤醒。

### Proactor

```text
stream_handler 返回 WAIT
    ↓
提交 io_uring timeout SQE
    ↓
timeout 完成后产生 CQE
    ↓
EVENT_STREAM_RETRY 再次检查 AOF
```

理解重点是操作提交和完成通知。

统一部分为：

```text
handler
stream_handler
close_handler
KVS_STREAM_WAIT
输入缓冲区消费规则
输出缓冲区续发规则
```

差异部分为：

```text
连接如何等待
何时被唤醒
读写如何注册或提交
完成结果如何返回事件循环
```

这正是业务逻辑和 I/O 调度解耦后的边界。

## Proactor 阶段完成

最终提交为：

```text
505f937 feat(network): 支持 Proactor 主从复制
```

它完成了：

```text
✅ 多监听器共用一个 io_uring
✅ 每监听器和每连接独立 handler
✅ Primary Snapshot/AOF 流式输出
✅ ONLINE timeout 定时重试
✅ Replica 主动连接 Primary
✅ open/message/close 生命周期回调
✅ 用户态输入缓冲区连续消费
✅ 统一连接关闭路径
✅ Proactor 一主一从端到端验证
✅ 默认 NtyCo 自动化回归测试
```

## 主从同步功能最终合并

功能分支最终合并到 `main`：

```text
4c0bb7a merge: 集成主从同步
```

合并后执行了干净构建：

```bash
make clean
make
```

所有源文件在 `-Wall -Wextra` 下成功编译，没有 warning。

随后再次执行：

```bash
./tests/test_replication.sh
```

全量同步与 ONLINE 增量同步均通过，说明功能分支进入 `main` 后仍然保持正确。

README 也随之更新，不再把主从复制描述为 Roadmap，而是记录实际支持的启动方式、状态机、测试方法和当前边界。

## 当前主从同步完成度

如果以本阶段定义的目标衡量：

```text
“让三个网络后端都支持基础的一主一从异步复制，
完成首次全量同步、AOF 追赶和 ONLINE 持续同步。”
```

该目标已经完成。

已具备：

```text
✅ standalone / primary / replica 三种角色
✅ Primary 客户端端口与复制端口分离
✅ Replica 主动连接 Primary
✅ PING/PONG 与 PSYNC 握手
✅ FULLRESYNC 元数据
✅ Snapshot 分块传输与原子安装
✅ Snapshot AOF offset 边界
✅ AOF 增量追赶
✅ ONLINE 实时异步复制
✅ Replica 拒绝客户端写入
✅ NtyCo / Reactor / Proactor 三种网络后端
✅ 自动化全量与增量集成测试
```

但它仍然是学习型复制系统，而不是生产级高可用系统。

尚未具备：

```text
⬜ Replica 断线自动重连与退避
⬜ replication backlog 与部分重同步
⬜ Replica ACK 与已应用 offset 跟踪
⬜ 心跳和连接存活检测
⬜ 多 Replica 的独立连接状态管理
⬜ Primary 故障检测与 Replica 提升
⬜ 选主、仲裁与脑裂处理
⬜ Snapshot/AOF 后台生成，避免阻塞事件循环
⬜ 认证、TLS、协议版本和校验和
⬜ 三种网络后端的自动化测试矩阵与故障注入
```

所以更准确的评价是：

```text
学习项目的主从同步里程碑：完成度较高
生产级复制系统：仍处于早期阶段
```

## 去 AI 化复盘：必须能够自己讲清楚的四件事

这次功能代码很多，但真正需要掌握的不是每一行语法，而是四个核心模型。

### 1. 状态机

```text
CONNECTING
    建立 Replica 到 Primary 的 TCP 连接

HANDSHAKE
    PING/PONG、PSYNC 协商

FULL_SYNC
    接收并安装 Snapshot

CATCH_UP
    从 Snapshot offset 开始补齐 AOF

ONLINE
    持续接收新产生的 AOF 命令
```

状态决定同一批 TCP 字节当前应该由哪个协议逻辑解释。

### 2. offset

AOF offset 是字节位置，不是命令数量。

Snapshot 中记录：

```text
AOF_OFFSET N
```

表示 Snapshot 已经包含 AOF 区间 `[0, N)` 的执行结果。Replica 安装 Snapshot 后只应该继续应用 `[N, current_end)`，否则会重复执行已经包含在快照中的命令。

### 3. 缓冲区

```text
TCP 没有消息边界
一次 recv 可能得到半条、完整一条或多条消息
```

输入侧使用：

```text
input.length + consumed_length
```

保留未消费尾部，并在同一次 READ 后持续处理已经到达用户态的数据。

输出侧使用：

```text
output.length + output.offset
```

处理部分发送，并确保一块完整发送后才生成下一块 Snapshot/AOF。

### 4. 三种调度模型

```text
NtyCo
    连接协程主动让出执行权，之后恢复

Reactor
    epoll 告诉程序 fd 已经就绪

Proactor
    程序先提交 I/O 操作，io_uring 告诉程序操作已经完成
```

三者都调用同一套复制 handler；不同的是等待、唤醒与 I/O 完成方式。

如果能够不看代码讲清楚这四件事，就不是只会复制 AI 生成的代码，而是已经掌握了这次改造的核心设计。

## 面试版总结

可以用下面这段话概括整个功能：

> 我在一个 C 语言 KVStore 中实现了基于 Snapshot 和 AOF offset 的 Primary-Replica 异步复制。Replica 首次连接后通过 PSYNC 进入 FULL_SYNC，安装带有 AOF byte offset 的 Snapshot，再从该 offset 进行 CATCH_UP，追平后进入 ONLINE 持续接收增量命令。复制状态机集中在 replication 模块中，三种网络后端只实现统一的多监听器、主动连接器、每连接 handler、流式输出、定时唤醒和连接生命周期契约。NtyCo 使用协程 sleep/yield，Reactor 使用 epoll timeout，Proactor 使用 io_uring timeout CQE。为解决 TCP 拆包、合包和跨状态数据，我让输入缓冲区按 consumed_length 连续消费；输出则通过 length/offset 处理部分发送。当前实现已通过三种后端的端到端验证和自动化全量、增量同步测试，但自动重连、部分重同步、ACK、心跳和故障转移仍属于后续工作。

这段总结覆盖了：

```text
业务目标
复制状态机
offset 正确性
网络层解耦
三种调度差异
TCP 缓冲区问题
测试证据
当前工程边界
```

至此，本轮主从同步需求完成并合并到主分支。
