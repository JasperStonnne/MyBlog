---
title: 91kvstore
description: 从网络模型起步，逐步补齐多存储引擎、持久化、主从复制与长度帧协议的 C 语言 KV 存储。
date: 2026-08-22T00:00:00+08:00
weight: 10
showComments: false
showReadingTime: false
showWordCount: false
showTableOfContents: true
---

`91kvstore` 是一条持续演进的系统编程主线。它从统一接入 Reactor、Proactor 与协程开始，随后扩展数组、红黑树、Hash、SkipList 等存储引擎，再向 AOF、Snapshot、主从复制和更可靠的命令协议推进。

- 语言：C
- 关键词：epoll、io_uring、协程、存储引擎、持久化、复制
- 源码：[JasperStonnne/91kvstore](https://github.com/JasperStonnne/91kvstore)

## 阅读路线

1. [网络层：统一接入 Reactor、Proactor 与协程]({{< ref "kv-store-network-layer" >}})
2. [协议层与数组存储]({{< ref "kv-store-protocol-array" >}})
3. [客户端测试、压力测试与多引擎扩展]({{< ref "kv-store-testing-multi-engine" >}})
4. [Hash 接入、多语言客户端与 SkipList 扩展]({{< ref "kv-store-hash-multilingual-clients-skiplist" >}})
5. [KV 进化（一）：接入跳表]({{< ref "kv-evolution-skiplist-engine" >}})
6. [KV 进化（二）：Snapshot + AOF]({{< ref "kv-evolution-snapshot-aof-persistence" >}})
7. [KV 进化（三）：内存管理与内存池]({{< ref "kv-evolution-memory-management" >}})
8. [KV 进化（四）：Batch Commands]({{< ref "kv-evolution-batch-commands" >}})
9. [KV 进化（五）：Primary / Replica 主从同步]({{< ref "kv-evolution-primary-replica-sync" >}})
10. [KV 进化（六）：命令协议改造]({{< ref "kv-evolution-length-prefixed-protocol" >}})
