---
title: Go 项目反推：Feed 流系统实战——Redis 缓存设计
slug: go-feed-redis-cache-design
description: 梳理 Feed 流项目中 Redis 的 7 种用途，包括缓存读写、一致性策略、穿透与击穿防护、Lua 原子限流和 ZSET 热榜
date: 2026-07-28T00:00:00+08:00
draft: false
image: cover.svg
tags:
  - Go
  - Redis
  - 缓存
  - Lua
  - 限流
  - ZSET
categories:
  - 项目记录
---

承接 [Go 项目反推：Feed 流系统实战——数据库与 GORM]({{< ref "go-feed-database-gorm" >}})，这一篇继续反推 `feedsystem_video_go` 的 Redis 设计，梳理缓存读写、一致性与高并发防护，以及 Lua 限流和 ZSET 热榜的实现。

## 项目里 Redis 的 7 种用途

| #   | 用途              | 数据结构         | Key模式                                      |
| --- | --------------- | ------------ | ------------------------------------------ |
| 1   | JWT Token 缓存     | String       | `account:<id>`                             |
| 2   | Refresh Token 缓存 | String       | `account:<id>:refresh` / `refresh:<token>` |
| 3   | 视频详情缓存          | String(JSON) | `video:detail:id=<id>`                     |
| 4   | 视频实体缓存          | String(JSON) | `video:entity:<id>`                        |
| 5   | 热榜              | ZSET         | `hot:video:1m:<时间窗口>`                      |
| 6   | 分片上传会话          | String(JSON) | `chunk_upload:<upload_id>`                 |
| 7   | 接口限流            | String(计数器)  | `ratelimit:<prefix>:<subject>`             |

---

## 1. 缓存读取流程（video_service.go 的 GetDetail）

```go
func (vs *VideoService) GetDetail(ctx context.Context, id uint) (*Video, error) {
    cacheKey := vs.cache.Key("video:detail:id=%d", id)

    // 第一步：查缓存（50ms 超时）
    getCached := func() (*Video, bool) {
        opCtx, cancel := context.WithTimeout(ctx, 50*time.Millisecond)
        defer cancel()

        b, err := vs.cache.GetBytes(opCtx, cacheKey)
        if err != nil {
            return nil, false  // 缓存没命中
        }
        var cached Video
        if err := json.Unmarshal(b, &cached); err != nil {
            return nil, false  // 数据损坏
        }
        return &cached, true   // 缓存命中
    }

    // 尝试读缓存
    if v, ok := getCached(); ok {
        return v, nil  // 命中，直接返回
    }

    // 缓存没命中，查数据库...
}
```

### 为什么缓存读取要设 50ms 超时？

50ms 超时不是判断"没命中"，而是**防止 Redis 卡住**的降级策略：

- 缓存没命中：Redis 快速返回"key 不存在"，几毫秒就完成
- Redis 卡住：网络抖动、Redis 内部阻塞、连接池耗尽，可能卡几秒

没有超时：Redis 卡住 → 请求一直等着 → 用户体验很差
有 50ms 超时：Redis 超过 50ms 没响应 → 放弃缓存 → 直接查数据库

### 为什么不设更短的超时，比如 5ms？

Redis 正常操作本身需要几毫秒，设 5ms 正常情况都可能超时，缓存形同虚设。50ms 是平衡点：覆盖正常操作 + 轻微网络波动，同时不会让用户等太久。

---

## 2. 缓存写入

```go
setCached := func(video *Video) {
    b, err := json.Marshal(video)
    if err != nil {
        return
    }
    opCtx, cancel := context.WithTimeout(ctx, 50*time.Millisecond)
    defer cancel()
    _ = vs.cache.SetBytes(opCtx, cacheKey, b, vs.cacheTTL)
}
```

### 为什么缓存要设过期时间（TTL）？

TTL = Time To Live，缓存的"保质期"。

**主要原因：数据一致性**

```
09:00  缓存存了视频标题"今天天气真好"
09:05  作者把标题改成"今天下雨了"
       → 数据库更新了，但缓存还是旧的
09:06  用户查这个视频 → 从缓存读到"今天天气真好"
       → 看到的是过期数据
```

没有 TTL：缓存永远是旧数据，用户看到的和数据库不一致
有了 TTL（比如 10 分钟）：最多 10 分钟后缓存自动过期，下次请求加载最新数据

这是用**时间换一致性**：允许短暂不一致，但最终会一致。

**次要原因：缓存空间有限**

缓存满了需要淘汰旧数据，TTL 帮助自动清理。

---

## 3. 缓存删除（主动失效）

### 为什么不等 TTL 过期，要主动删缓存？

看 video_service.go 删视频时：

```go
func (vs *VideoService) DeleteVideo(ctx context.Context, id uint) error {
    // 先删数据库
    if err := vs.repo.DeleteVideo(ctx, id); err != nil {
        return err
    }
    // 再删缓存
    if vs.cache != nil {
        cacheKey := vs.cache.Key("video:detail:id=%d", id)
        _ = vs.cache.Del(context.Background(), cacheKey)
    }
    return nil
}
```

DB 更新后立刻删缓存，下次请求来了缓存没命中，从数据库加载最新数据。

---

## 4. 缓存和 DB 的一致性策略

### 两种方案对比

| 方案 | 做法 | 风险 |
|------|------|------|
| 先删缓存，再更新 DB | 删缓存 → 写 DB | 删完缓存、还没写 DB 时，另一个请求读 DB 旧数据写入缓存 |
| 先更新 DB，再删缓存 | 写 DB → 删缓存 | 写完 DB、还没删缓存时，另一个请求读到旧缓存 |

### 为什么"先更新 DB 再删缓存"更安全？

**方案 1：先删缓存，再更新 DB**

```
00:00  请求A：删缓存
00:01  请求B：查视频 → 缓存没命中 → 从DB读到旧数据 → 写入缓存
00:02  请求A：更新DB为新数据

结果：DB是新数据，缓存里是旧数据（脏数据）
```

发生概率不低，因为"删缓存"到"更新 DB"之间可能有几百毫秒（DB 写入比 Redis 慢）。

**方案 2：先更新 DB，再删缓存**

```
00:00  请求A：更新DB为新数据
00:01  请求B：查视频 → 从缓存读到旧数据（概率小）
00:02  请求A：删缓存

结果：即使请求B读到旧缓存，00:02缓存就被删了
```

出问题窗口极小：必须在"更新 DB"和"删缓存"之间，恰好有另一个请求来读。

### 共同风险点：删缓存失败了怎么办？

```
00:00  更新DB成功 → DB是新数据
00:01  删缓存失败（网络抖动、Redis挂了）
       → 缓存还是旧数据
```

**解决方案：靠 TTL 兜底**

代码里删缓存都用了 `_ =` 忽略错误：

```go
_ = vs.cache.Del(context.Background(), cacheKey)
```

即使删缓存失败，最多等 TTL 过期（比如 10 分钟），缓存自动失效，下次请求从 DB 加载新数据。

这是**最终一致性**：允许短暂不一致，但最终会一致。

---

## 5. 缓存穿透/击穿/雪崩

### 三个概念的区别

| 问题 | 现象 | 危害 |
|------|------|------|
| **缓存穿透** | 查询不存在的数据，缓存永远没命中 | 每次请求都打到 DB |
| **缓存击穿** | 热点 key 过期，大量请求同时打到 DB | DB 瞬间压力暴增 |
| **缓存雪崩** | 大量 key 同时过期 | DB 瞬间压力暴增 |

### 缓存击穿的解法（这个项目用的）

场景：一个热门视频的缓存过期了，100 个用户同时请求

```go
// 缓存没命中，尝试获取分布式锁
lockKey := "lock:" + cacheKey
token, locked, lockErr := vs.cache.Lock(lockCtx, lockKey, 2*time.Second)

if lockErr == nil && locked {
    // 拿到锁：去查DB，回填缓存
    defer func() { _ = vs.cache.Unlock(context.Background(), lockKey, token) }()

    video, err := vs.repo.GetByID(ctx, id)
    if err != nil {
        return nil, err
    }
    setCached(video)
    return video, nil
}

// 没拿到锁：等待别人回填缓存（最多100ms）
for i := 0; i < 5; i++ {
    time.Sleep(20 * time.Millisecond)
    if v, ok := getCached(); ok {
        return v, nil  // 别人已经回填了，直接用
    }
}

// 降级：直接查DB
video, err := vs.repo.GetByID(ctx, id)
```

**核心逻辑：让一个请求干活，其他请求等着用结果**

```
❌ 没有锁的击穿场景：
100个请求同时来 → 缓存没命中 → 100个都去查DB → DB压力x100

✅ 有锁的保护：
100个请求同时来 → 缓存没命中
    ↓
请求1：拿到锁 → 查DB → 写缓存 → 返回
请求2-100：没拿到锁 → 等20ms → 查缓存 → 命中 → 返回
```

**时序示例（DB 查询 15ms）：**

```
00:00   请求1拿到锁，开始查DB（耗时15ms）
        请求2-100没拿到锁，开始sleep 20ms
00:15   请求1查完DB，写入缓存
00:20   请求2-100醒来，查缓存 → 命中！
```

**降级策略（DB 查询很慢时）：**

请求 2-100 最多等 100ms（5次x20ms），如果请求 1 还没完成，降级去查 DB。

但这种情况很少见，DB 查询通常几毫秒到几十毫秒。

### 缓存穿透的解法（这个项目没做）

缓存穿透：查询不存在的数据，比如 `GET /video/999999999`，数据库里没有，每次请求都打到 DB。

| 方案 | 做法 |
|------|------|
| **缓存空值** | 查 DB 没结果，也写入缓存（value=""），设短 TTL |
| **布隆过滤器** | 在缓存前加一层，快速判断 key 是否存在 |

项目为什么没做：视频 ID 通过 Feed 流返回，不存在"查不存在 ID"的正常场景。接口有限流保护，攻击成本高。

### 缓存雪崩的解法（这个项目没做）

缓存雪崩：大量 key 同时过期，DB 瞬间压力暴增。

| 方案 | 做法 |
|------|------|
| **随机 TTL** | 给缓存 TTL 加随机值，避免同时过期 |
| **多级缓存** | L1 本地缓存 + L2 Redis 缓存 |
| **永不过期 + 异步更新** | 缓存不设 TTL，由后台任务定期更新 |

项目用的是固定 TTL，没做随机化，业务量可控。

---

## 6. 限流中间件（ratelimit.go）

### 限流是什么？

防止用户发太多请求打垮服务器。像银行取号机：每分钟最多服务 100 个人，第 101 个人来了告诉他"请稍后再来"。

### 限流中间件是什么？

Gin 中间件，所有请求都要经过它：

```
用户请求 → 限流中间件检查 → 通过 → handler处理 → 返回
                    ↓
                 超限 → 直接返回429 "too many requests"
```

### 限流实现原理

```go
func Limit(cache *rediscache.Client, keyPrefix string, maxRequests int64, window time.Duration, keyFunc KeyFunc) gin.HandlerFunc {
    return func(c *gin.Context) {
        subject, ok := keyFunc(c)
        key := buildKey(keyPrefix, subject)

        count, err := cache.IncrementWithExpire(c.Request.Context(), key, window)
        if err != nil {
            c.Next()
            return
        }
        if count > maxRequests {
            c.AbortWithStatusJSON(http.StatusTooManyRequests, gin.H{"error": "too many requests"})
            return
        }
        c.Next()
    }
}
```

### IncrementWithExpire 的 Lua 脚本

```lua
local count = redis.call("INCR", KEYS[1])
if count == 1 then
    redis.call("PEXPIRE", KEYS[1], ARGV[1])
end
return count
```

**限流流程（1 分钟内最多 100 次）：**

```
00:00.000  请求1来 → INCR → count=1
           因为count==1，是第一个请求
           → PEXPIRE设60000毫秒（60秒）过期

00:00.001  请求2来 → INCR → count=2
           count!=1，不设过期（沿用之前的）

...

00:00.500  请求100来 → INCR → count=100
           → 通过（100 <= 100）

00:00.600  请求101来 → INCR → count=101
           → 拒绝！（101 > 100）
           → 返回429 "too many requests"

01:00.000  key自动过期，Redis删除这个key
           计数器归零

01:00.001  请求来 → INCR → count=1（重新开始计数）
           → PEXPIRE重新设60秒
           → 通过
```

### 为什么用 Lua 脚本？

**原子执行 = 两条命令之间不会有其他命令插队**

**没有 Lua 脚本（可能被插队）：**

```
00:00  你的程序：INCR key → count=1
00:01  另一个程序：INCR key → count=2
00:02  你的程序：EXPIRE key 60秒
00:03  另一个程序：EXPIRE key 60秒
```

你的 INCR 和 EXPIRE 之间，被别人插队了。

**有 Lua 脚本（不会插队）：**

```
00:00  你的程序：开始执行Lua脚本
       INCR key → count=1
       EXPIRE key 60秒
00:01  Lua脚本执行完，其他程序才能执行
```

**注意：原子执行 ≠ 回滚，原子执行 = 不插队**

如果 INCR 成功了，EXPIRE 失败了，INCR 的结果不会撤销。

### 为什么需要原子执行？

如果分开写：

```go
count = redis.INCR(key)      // count=1
// 这里程序崩了，EXPIRE没执行
redis.EXPIRE(key, 60秒)      // 没执行到
```

结果：key 永远不过期，计数永远不会归零，限流永久生效。

Lua 脚本保证 INCR 和 EXPIRE 一起执行，不会出现"只执行了一半"的情况。

### EXPIRE 和 PEXPIRE

- `EXPIRE`：单位是秒
- `PEXPIRE`：单位是毫秒

### 限流的 key 设计

```go
func buildKey(keyPrefix, subject string) string {
    return fmt.Sprintf("feedsystem:ratelimit:%s:%s", keyPrefix, strings.TrimSpace(subject))
}
```

两种维度：
- `KeyByIP`：按 IP 限流，防止单 IP 攻击
- `KeyByAccount`：按用户 ID 限流，防止单用户刷接口

---

## 7. ZSET 热榜（popularity_cache.go）

### ZSET 是什么？

Redis 的有序集合，每个元素有一个分数，自动按分数排序。

```
ZSET "热榜"
┌─────────┬─────────┐
│  元素    │  分数   │
├─────────┼─────────┤
│ video_5 │  1000   │  ← 最热
│ video_2 │  800    │
│ video_8 │  500    │
│ video_1 │  100    │  ← 最冷
└─────────┴─────────┘
```

### 热榜更新代码

```go
func UpdatePopularityCache(ctx context.Context, cache *rediscache.Client, id uint, change int64) {
    // 删掉旧缓存
    _ = cache.Del(context.Background(), cache.Key("video:detail:id=%d", id))
    _ = cache.Del(context.Background(), cache.Key("video:entity:%d", id))

    // 写入热榜
    now := time.Now().UTC().Truncate(time.Minute)  // 当前时间，精确到分钟
    windowKey := cache.Key("hot:video:1m:%s", now.Format("200601021504"))  // 热榜key
    member := strconv.FormatUint(uint64(id), 10)  // 视频ID

    _ = cache.ZincrBy(opCtx, windowKey, member, float64(change))  // 热度+change
    _ = cache.Expire(opCtx, windowKey, 2*time.Hour)  // 2小时后过期
}
```

### 滑动窗口设计

热榜 key 是 `hot:video:1m:202607281430`，意思是"2026 年 7 月 28 日 14:30 这一分钟的热榜"。

```
00:00  video_5被点赞 → hot:video:1m:202607281430 → video_5分数+1
00:30  video_5又被点赞 → hot:video:1m:202607281430 → video_5分数+1
01:00  新的一分钟 → hot:video:1m:202607281431
```

每分钟一个 ZSET，2 小时后自动过期。

### 为什么要按分钟分窗口？

**如果用一个大的 ZSET（不分窗口）：**

```
所有点赞都写到同一个key "hot:video"

1月1日：video_5被点赞1000次，分数=1000，排第一
1月2日：video_8被点赞500次，分数=500
1月3日：video_5被点赞0次，但分数还是1000，还是排第一
```

问题：旧视频的热度永远不衰减，新视频永远追不上。

**按分钟分窗口：**

```
每分钟一个ZSET：

hot:video:1m:202607281430  →  14:30这一分钟的热度
hot:video:1m:202607281431  →  14:31这一分钟的热度
hot:video:1m:202607281432  →  14:32这一分钟的热度
```

查热榜时，把最近几个窗口合并：

```
14:35查热榜：
→ 合并 14:30、14:31、14:32、14:33、14:34、14:35 这6个窗口
→ 算总分，排序
```

好处：
- 14:30 的热度，到 14:35 时已经衰减了（只有 1/6 权重）
- 14:35 新产生的热度立刻反映
- 热榜是"最近几分钟"的热度，不是"历史累计"

### ZSET 常用操作

```go
// 分数增加
cache.ZincrBy(ctx, key, member, score)

// 按分数倒序取 top N
cache.ZRevRange(ctx, key, 0, 99)  // 取前100名

// 合并多个ZSET
cache.ZUnionStore(ctx, dst, keys, "SUM")

// 设置过期时间
cache.Expire(ctx, key, 2*time.Hour)
```

---

## 面试要点总结

### Q: 缓存和 DB 的一致性怎么保证？

A: 先更新 DB，再删缓存。删缓存失败靠 TTL 兜底，最终一致性。

### Q: 缓存穿透/击穿/雪崩的区别和解法？

A: 穿透是查不存在的数据（缓存空值/布隆过滤器），击穿是热点 key 过期（分布式锁 + 等待重试），雪崩是大量 key 同时过期（随机 TTL）。

### Q: 限流怎么实现的？

A: Redis 计数器，Lua 脚本原子执行 INCR + PEXPIRE，超限返回 429。

### Q: 为什么用 Lua 脚本？

A: 保证 INCR 和 EXPIRE 原子执行，不会被其他命令插队，避免"只执行了一半"的问题。

### Q: 热榜怎么实现的？

A: ZSET 按分钟分窗口，点赞时 ZincrBy 加分数，查热榜时合并最近几个窗口算总分。分窗口实现热度自动衰减。
