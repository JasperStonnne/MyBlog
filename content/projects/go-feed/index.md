---
title: Go Feed 流系统
description: 沿一条真实请求链路，反推认证、数据库、缓存、消息队列、Feed、SSE 与工程化设计。
date: 2026-07-21T00:00:00+08:00
weight: 20
showComments: false
showReadingTime: false
showWordCount: false
showTableOfContents: true
---

这组文章不是把框架 API 逐项抄一遍，而是从一个已有 Go 项目倒推设计：先建立整体地图，再沿请求生命周期进入各层，最后回到异步架构、Feed 流和工程化。

- 语言：Go
- 关键词：Gin、GORM、Redis、RabbitMQ、JWT、SSE

## 阅读路线

1. [从 Docker 到开源贡献：先建立项目全景]({{< ref "go-feed-system-reverse-engineering" >}})
2. [请求生命周期]({{< ref "go-feed-request-lifecycle" >}})
3. [认证体系]({{< ref "go-feed-authentication" >}})
4. [数据库与 GORM]({{< ref "go-feed-database-gorm" >}})
5. [Redis 缓存设计]({{< ref "go-feed-redis-cache-design" >}})
6. [消息队列与异步架构]({{< ref "go-feed-message-queue-async-architecture" >}})
7. [Feed 流设计]({{< ref "go-feed-design" >}})
8. [SSE 实时推送与分片上传]({{< ref "go-feed-sse-upload" >}})
9. [工程化]({{< ref "go-feed-engineering" >}})
