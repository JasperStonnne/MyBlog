---
title: Go 项目反推：Feed 流系统实战——消息队列与异步架构
slug: go-feed-message-queue-async-architecture
description: 梳理 Feed 流项目中 RabbitMQ 的核心设计，包括 Exchange、Queue、Routing Key、消息重试、指数退避、Outbox 模式与死信队列
date: 2026-07-29T00:00:00+08:00
draft: false
image: cover.svg
tags:
  - Go
  - RabbitMQ
  - 消息队列
  - 异步架构
  - Outbox
  - 死信队列
categories:
  - 项目记录
---

承接 [Go 项目反推：Feed 流系统实战——Redis 缓存设计]({{< ref "go-feed-redis-cache-design" >}})，这一篇继续反推 `feedsystem_video_go` 的消息队列与异步架构，梳理 RabbitMQ 的消息路由、消费重试、降级策略、Outbox 模式和死信队列。

## 什么是消息队列（MQ）

想象一个餐厅：
- 没有 MQ：顾客点餐 → 厨师做完一个才能做下一个 → 顾客等着
- 有 MQ：顾客点餐 → 写在小票上放窗口 → 厨师按顺序做 → 做好了喊号

消息队列就是那个"窗口"，消息就是"小票"。

---

## 为什么需要消息队列

用户点赞后要做的事：
```
用户点赞 → 更新Like表 → 更新Video点赞数 → 更新热度 → 推送通知 → 返回成功
```

没有 MQ 的问题：
1. 用户等太久（每个操作都要时间）
2. 任何一个失败，整个流程卡住
3. 高并发时 DB 扛不住

有了 MQ：
```
用户点赞 → 更新Like表 → 发一条"点赞消息"到MQ → 立刻返回成功
                                        ↓
                        Worker从MQ取消息 → 更新点赞数 → 更新热度 → 推送通知
```

好处：
1. 用户不用等所有操作完成，体验快
2. 点赞服务和热度计算服务解耦，改一个不影响另一个
3. 高并发时消息排队，DB 不会被打垮

---

## MQ 的三个核心概念

| 概念 | 比喻 | 作用 |
|------|------|------|
| **Exchange** | 邮局 | 接收消息，根据规则分发 |
| **Queue** | 邮箱 | 存储消息，等 Worker 来取 |
| **Routing Key** | 地址 | 消息的"目的地"，Exchange 根据它决定放哪个 Queue |

流程：
```
生产者（点赞服务） → 发消息到Exchange → Exchange根据Routing Key → 放到Queue → Worker从Queue取消息消费
```

---

## 三种 Exchange 类型

| 类型 | 规则 | 举例 |
|------|------|------|
| **Direct** | Routing Key 完全匹配 | 发到"like.like"，只有绑定了"like.like"的 Queue 才能收到 |
| **Topic** | 支持通配符（* 和 #） | 发到"like.*"，能匹配"like.like"、"like.unlike" |
| **Fanout** | 广播，忽略 Routing Key | 发到所有绑定的 Queue，每个 Queue 都能收到 |

这个项目用的是 Topic Exchange。

---

## 项目里的 6 种 MQ

| Exchange | Routing Key | 用途 |
|----------|-------------|------|
| like.events | like.like / like.unlike | 点赞/取消点赞 |
| comment.events | comment.publish / comment.delete | 评论/删除评论 |
| social.events | social.follow / social.unfollow | 关注/取关 |
| video.popularity.events | video.popularity.update | 热度更新 |
| video.timeline.events | video.timeline.publish | 视频发布 |
| dlx.events | # | 死信队列（重试/告警） |

---

## 代码实现

### MQ 相关文件

```
backend/internal/middleware/rabbitmq/
├── rabbitMQ.go        # MQ基础封装（连接、Channel、DeclareTopic、PublishJSON）
├── dlx.go             # 死信队列声明和重试计数
├── likeMQ.go          # 点赞MQ
├── commentMQ.go       # 评论MQ
├── socialMQ.go        # 关注MQ
├── popularityMQ.go    # 热度MQ
└── timelineMQ.go      # Feed流MQ（Outbox模式）
```

### Worker 相关文件

```
backend/internal/worker/
├── likeworker.go           # 点赞Worker
├── commentworker.go        # 评论Worker
├── socialworker.go         # 关注Worker
├── popularityworker.go     # 热度Worker
├── outboxworker.go         # Outbox定时任务
├── notificationworker.go   # 通知Worker
└── ssehub.go               # SSE推送Hub
```

---

## 消息生产（发消息）

### rabbitMQ.go - PublishJSON

```go
func PublishJSON(ctx context.Context, ch *amqp.Channel, exchange string, routingKey string, payload any) error {
    b, _ := json.Marshal(payload)           // 第1步：把数据变成JSON
    return ch.PublishWithContext(ctx, exchange, routingKey, false, false, amqp.Publishing{
        ContentType:  "application/json",    // 第2步：告诉MQ这是JSON格式
        DeliveryMode: amqp.Persistent,       // 第3步：消息持久化
        Body:         b,                     // 第4步：消息内容
    })
}
```

逐行解释：
- `json.Marshal(payload)`：把 Go 结构体变成 JSON 字节
- `ch.PublishWithContext(...)`：调用 MQ 客户端，发一条消息
- `exchange`：发到哪个 Exchange（邮局）
- `routingKey`：消息地址（like.like）
- `amqp.Persistent`：消息存到磁盘，MQ 重启不丢
- `Body: b`：消息内容

### 消息长什么样

```json
{
    "video_id": 123,
    "account_id": 456,
    "event_type": "like.like",
    "timestamp": "2026-07-28T14:30:00Z"
}
```

就是一个 JSON，告诉 Worker"用户 456 给视频 123 点了赞"。

---

## 消息消费（Worker 处理）

### likeworker.go - Run

```go
func (w *LikeWorker) Run(ctx context.Context) error {
    deliveries, err := w.ch.Consume(w.queue, "", false, false, false, false, nil)

    for {
        select {
        case <-ctx.Done():
            return ctx.Err()
        case d, ok := <-deliveries:
            if !ok {
                return errors.New("deliveries channel closed")
            }
            w.handleDelivery(ctx, d)
        }
    }
}
```

逐行解释：
- `w.ch.Consume(w.queue, ...)`：从 Queue 订阅消息，返回一个 channel
- `for { select { ... } }`：循环读消息，有消息来就处理
- `w.handleDelivery(ctx, d)`：调用处理函数，执行业务逻辑

### handleDelivery - 重试逻辑

```go
func (w *LikeWorker) handleDelivery(ctx context.Context, d amqp.Delivery) {
    const maxRetries = 3
    for i := 0; i <= maxRetries; i++ {
        if err := w.process(ctx, d.Body); err != nil {
            if i >= maxRetries {
                // 超过3次，放弃，Ack删除消息
                log.Printf("like worker: 重试 %d 次后仍失败, 丢弃", maxRetries)
                _ = d.Ack(false)
                return
            }
            // 等一下再重试（指数退避）
            wait := time.Duration(1<<uint(i)) * time.Second  // 1s, 2s, 4s
            log.Printf("like worker: 处理失败, %v 后重试 (%d/%d)", wait, i+1, maxRetries)
            time.Sleep(wait)
            continue
        }
        // 成功，Ack删除消息
        _ = d.Ack(false)
        return
    }
}
```

重试流程：
```
第1次处理 → 失败 → 等1秒
第2次处理 → 失败 → 等2秒
第3次处理 → 失败 → 等4秒
第4次处理 → 失败 → 放弃，Ack删除消息
```

### 指数退避（1s → 2s → 4s）

`1<<uint(i)` 是位运算：
- i=0: 1<<0 = 1秒
- i=1: 1<<1 = 2秒
- i=2: 1<<2 = 4秒

为什么要等这么久？避免疯狂重试打垮服务器。

### 为什么失败了还 Ack？

不 Ack 的话，消息会一直留在 Queue 里，Worker 会一直重试，卡住后面的消息。Ack 后消息从 Queue 删除，避免阻塞。

---

## MQ 发布失败的降级策略

场景：用户点赞成功，DB 写入了，但发 MQ 消息失败了。

如果直接忽略：热度不会更新，点赞数据和热度不一致。

### fallback 直写

```go
func (s *LikeService) publishLikeEvent(ctx context.Context, videoID, accountID uint, routingKey string) {
    event := LikeEvent{VideoID: videoID, AccountID: accountID}

    err := rabbitmq.PublishJSON(ctx, s.ch, "like.events", routingKey, event)
    if err != nil {
        // MQ发送失败，降级：直接更新热度
        go s.popularityWorker.HandleLikeEvent(event)
    }
}
```

意思：
- MQ 能用 → 走 MQ 异步处理
- MQ 挂了 → 直接调 Worker 的处理函数，同步处理

为什么不用重试？如果 MQ 挂了，重试大概率还是失败，用户要等很久。直接降级：虽然绕过了 MQ，但保证功能正常，用户体验不受影响。

---

## Outbox 模式

### 什么是 Outbox 模式

发视频时用的模式。

场景：用户发视频 → 更新 DB → 发 MQ 消息

问题：如果 DB 更新成功，MQ 发送失败，视频发布了但 Feed 流里没有。

为什么不能放在一个事务里？DB 和 MQ 是两个系统，没法用一个事务包起来。

### Outbox 的解法

把 MQ 消息先存到数据库里，用数据库事务保证一致性：

```
数据库事务（保证原子性）：
    INSERT INTO videos (title, play_url, ...) VALUES (...)
    INSERT INTO outbox_msgs (video_id, status) VALUES (xxx, 'pending')
    COMMIT
```

两步在同一个 DB 事务里，要么都成功，要么都失败。

然后用定时任务把消息发出去：

```
OutboxPoller（每秒运行一次）：

    查询：SELECT * FROM outbox_msgs WHERE status = 'pending'

    对于每条消息：
        发MQ消息
        更新：UPDATE outbox_msgs SET status = 'sent' WHERE id = xxx
```

### OutboxMsg 表结构

```go
type OutboxMsg struct {
    ID         uint
    VideoID    uint
    EventType  string
    CreateTime time.Time
    Status     string    // "pending" 或 "sent"
}
```

### 完整流程

```
00:00  用户发视频
       → 写Video表 + 写OutboxMsg表（同一个事务，保证原子）

00:01  OutboxPoller扫到这条消息
       → 发MQ消息成功
       → 标记为sent

00:02  Worker收到MQ消息
       → 更新Feed流
```

### 为什么这样能保证消息不丢？

- 消息先存在 DB 里，和业务数据在同一个事务
- 定时任务保证最终会发出去
- 即使 MQ 暂时挂了，消息还在 DB 里，等 MQ 恢复了再发

### 回滚只发生在 DB 层面

```
数据库事务：
    INSERT INTO videos ...     → 成功
    INSERT INTO outbox_msgs ... → 失败
    ROLLBACK                   → 两个都撤销
```

MQ 发送失败不回滚：
```
数据库事务：COMMIT（消息已经存到outbox_msgs表了）

OutboxPoller：
    发MQ消息 → 失败（MQ挂了）

    → 不回滚！消息还在outbox_msgs表里
    → 等MQ恢复了，下次扫描再发
```

### 为什么发视频用 Outbox，点赞不用？

| 场景 | 方案 | 原因 |
|------|------|------|
| 点赞 | 直接发 MQ + fallback | 点赞失败可以重试，用户能接受 |
| 发视频 | Outbox 模式 | 视频必须进 Feed 流，不能丢 |

视频发布是关键业务，不能丢消息，所以用 Outbox 保证"至少发一次"。

---

## 死信队列（DLX）

### 什么是死信队列

消息"死了"（消费失败），放到一个专门的队列里等处理。

就像快递派送失败：
```
正常流程：快递 → 送到你家 → 签收

派送失败：快递 → 送到你家 → 没人签收 → 退回快递站（死信队列）
                               → 重新派送（重试）
                               → 联系收件人（告警）
```

### 消息为什么会"死"

| 原因 | 举例 |
|------|------|
| 业务逻辑报错 | 热度计算除零了 |
| 消费超时 | Worker 处理太久，MQ 等不及了 |
| 拒绝消息 | 代码里主动 reject |

### 正常消费流程

```
Worker从Queue取消息
    ↓
处理成功 → msg.Ack() → MQ删除消息
```

### 消费失败了

```
Worker从Queue取消息
    ↓
处理失败 → msg.Nack() 或 超时
    ↓
MQ把消息放到死信队列（DLX）
    ↓
死信Worker处理：
    - 记录日志
    - 重试（最多N次）
    - 超过重试次数 → 告警/人工处理
```

### dlx.go - DeclareDLX

```go
const DLXExchange = "dlx.events"
const MaxRetryCount = 3

func DeclareDLX(ch *amqp.Channel, queueName string) error {
    // 1. 声明死信Exchange
    ch.ExchangeDeclare(DLXExchange, "topic", true, false, false, false, nil)

    // 2. 声明死信Queue（名字是 原Queue名 + ".dlx"）
    dlxQueue := queueName + ".dlx"
    ch.QueueDeclare(dlxQueue, true, false, false, false, nil)

    // 3. 绑定：# 匹配所有Routing Key
    ch.QueueBind(dlxQueue, "#", DLXExchange, false, nil)
}
```

- `like.events` 的死信队列叫 `like.events.dlx`
- `comment.events` 的死信队列叫 `comment.events.dlx`

### GetRetryCount - 获取重试次数

```go
func GetRetryCount(d amqp.Delivery) int {
    deaths, ok := d.Headers["x-death"].([]interface{})
    if !ok || len(deaths) == 0 {
        return 0
    }
    death, ok := deaths[0].(amqp.Table)
    if !ok {
        return 0
    }
    count, ok := death["count"].(int64)
    if !ok {
        return 0
    }
    return int(count)
}
```

MQ 会自动在消息头里记录"这条消息死了几次"，从 `x-death` 里取出来。

### 正常 Queue 声明时绑定 DLX

```go
func DeclareTopic(ch *amqp.Channel, exchange string, queue string, bindingKey string) error {
    // 声明Exchange
    ch.ExchangeDeclare(exchange, "topic", true, false, false, false, nil)

    // 声明Queue，绑定死信Exchange
    q, err := ch.QueueDeclare(queue, true, false, false, false,
        amqp.Table{"x-dead-letter-exchange": DLXExchange})

    // 绑定：Queue订阅Exchange的消息
    ch.QueueBind(q.Name, bindingKey, exchange, false, nil)
}
```

`x-dead-letter-exchange`：告诉 MQ，这个 Queue 的消息如果"死了"，发到 DLXExchange。

### 完整流程举例

```
正常流程：
    点赞 → like.events Queue → LikeWorker消费 → 成功 → Ack

失败流程：
    点赞 → like.events Queue → LikeWorker消费 → 失败 → Nack
                                                    ↓
                                        like.events.dlx Queue（死信队列）
                                                    ↓
                                        死信Worker处理（重试/告警）
```

---

## API 进程和 Worker 进程为什么要分开

```
cmd/
├── main.go          # API进程（处理HTTP请求）
└── worker/
    └── main.go      # Worker进程（消费MQ消息）
```

分开的原因：
1. **职责不同**：API 处理请求，Worker 处理后台任务
2. **独立扩展**：API 扛不住加 API 机器，Worker 扛不住加 Worker 机器
3. **故障隔离**：Worker 崩了不影响 API，API 崩了不影响 Worker
4. **资源分配**：API 需要低延迟，Worker 可以容忍高延迟

---

## ch.Qos(50, 0, false) - 预取数量控制

```go
ch.Qos(50, 0, false)
```

- 50：每次最多取 50 条消息
- 0：不限制消息大小
- false：只对当前 Channel 生效

为什么要控制预取数量？
- 太多：Worker 内存爆了
- 太少：Worker 闲着没事干
- 50：平衡点，Worker 处理完一批再取下一批

---

## 面试要点总结

### Q: 为什么用 MQ？

A: 削峰（高并发时消息排队）、解耦（点赞服务和热度计算独立）、异步（用户不用等所有操作完成）。

### Q: MQ 发布失败了怎么办？

A: 降级策略，直接调 Worker 的处理函数同步处理。点赞用 fallback 直写，发视频用 Outbox 模式保证消息不丢。

### Q: Outbox 模式是什么？

A: 把 MQ 消息先存到数据库里，用 DB 事务保证一致性。定时任务扫描 outbox_msgs 表，发 MQ 消息，标记已发送。

### Q: 死信队列怎么用的？

A: 消息消费失败时，MQ 自动把消息放到死信队列。Worker 里有重试逻辑，最多重试 3 次，指数退避（1s→2s→4s），超过 3 次放弃。

### Q: API 进程和 Worker 进程为什么要分开？

A: 职责不同、独立扩展、故障隔离、资源分配不同。

### Q: 什么是指数退避？

A: 重试间隔按 2 的幂次增长（1s→2s→4s），避免疯狂重试打垮服务器。

### Q: 三种 Exchange 的区别？

A: Direct 精确匹配，Topic 支持通配符，Fanout 广播到所有 Queue。

### Q: 为什么点赞用 fallback，发视频用 Outbox？

A: 点赞失败用户能接受重试，视频发布是关键业务不能丢消息。Outbox 保证"至少发一次"。
