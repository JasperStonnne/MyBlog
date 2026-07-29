---
title: Go 项目反推：Feed 流系统实战——Feed 流设计
slug: go-feed-design
description: 拆解 Feed 流项目的推拉模型、游标分页、热榜、关注流、话题流、三级缓存与 singleflight
date: 2026-07-29T00:00:00+08:00
draft: false
image: cover.svg
tags:
  - Go
  - Feed 流
  - 游标分页
  - Redis
  - 缓存
  - Singleflight
categories:
  - 项目记录
---

承接 [Go 项目反推：Feed 流系统实战——数据库与 GORM]({{< ref "go-feed-database-gorm" >}})，这一篇继续拆解 `feedsystem_video_go` 的核心 Feed 流：从推拉模型到游标分页，再到热榜、关注流、话题流与缓存策略。

## 什么是 Feed 流

打开抖音/微博，往下刷，不断出来新内容，这就是 Feed 流。

---

## 这个项目的 4 种 Feed 流

| 类型 | 说明 | 排序方式 |
|------|------|----------|
| 最新视频 | 全站最新发布的视频 | 按时间倒序 |
| 热榜 | 最热门的视频 | 按热度排序 |
| 关注流 | 只看关注的人发的视频 | 按时间倒序 |
| 话题流 | 某个话题下的视频 | 按时间倒序 |

---

## 相关文件

```
backend/internal/feed/
├── entity.go      # 请求/响应结构体
├── handler.go     # HTTP处理
├── service.go     # 业务逻辑（缓存、singleflight）
└── repo.go        # 数据库查询
```

---

## 推模型 vs 拉模型

### 拉模型（这个项目用的）

```
发视频：只写Video表（不做额外操作）
刷视频：实时查询Video表（拉取数据）
```

数据流动：用户刷的时候才从 DB 拉。

### 推模型

```
发视频：写Video表 + 把视频ID推送到所有粉丝的Feed列表里
刷视频：直接从自己的Feed列表读（不用查DB）
```

数据流动：发的时候就推给粉丝了，粉丝刷的时候直接读。

### 推拉结合

```
普通用户发视频 → 推送给粉丝（粉丝少，推送成本低）
大V发视频 → 不推送，粉丝读的时候实时拉取
```

### 对比

| | 拉模型 | 推模型 | 推拉结合 |
|--|--------|--------|----------|
| 发视频成本 | 低 | 高（大V要推给几百万粉丝） | 中 |
| 刷视频成本 | 高（每次都要查DB） | 低（直接读列表） | 中 |
| 实时性 | 实时 | 略有延迟 | 中 |
| 复杂度 | 简单 | 复杂 | 最复杂 |
| 场景 | 用户量小 | 用户量大 | 抖音/微博 |

这个项目用拉模型，因为是学习项目，用户量小，拉模型简单够用。

---

## 游标分页 vs 页码分页

### 页码分页

```
第1页：GET /feed?page=1&size=10
第2页：GET /feed?page=2&size=10
```

对应 SQL：
```sql
SELECT * FROM videos ORDER BY create_time DESC LIMIT 5 OFFSET 0  -- 第1页
SELECT * FROM videos ORDER BY create_time DESC LIMIT 5 OFFSET 5  -- 第2页
```

**问题：数据变化时可能重复或漏数据**

```
数据库：[100, 90, 80, 70, 60, 50, 40, 30]

第1页（page=1, size=3）：跳过0条，取3条 → [100, 90, 80]

插入新视频110，数据库变成：[110, 100, 90, 80, 70, 60, 50, 40, 30]

第2页（page=2, size=3）：跳过3条，取3条 → [80, 70, 60]

视频80重复出现了！
```

为什么？因为 OFFSET 是基于"位置"的，不是基于"数据"的。新视频插入后，原来第 4 个位置的视频 80 变成了第 5 个，但 OFFSET 还是 3，所以又取到了。

### 游标分页（这个项目用的）

```
第1页：GET /feed?limit=10
       返回：video_list + next_time=1000

第2页：GET /feed?limit=10&latest_time=1000
       返回：video_list + next_time=900
```

对应 SQL：
```sql
SELECT * FROM videos ORDER BY create_time DESC LIMIT 10
SELECT * FROM videos WHERE create_time < 1000 ORDER BY create_time DESC LIMIT 10
```

**为什么游标分页不会重复？**

```
第1页：WHERE create_time < now() LIMIT 3
       返回 [100, 90, 80]
       next_time = 80的时间

插入110

第2页：WHERE create_time < 80的时间 LIMIT 3
       返回 [70, 60, 50]

没有重复！
```

游标是基于数据的值（时间），不是位置。新视频插入不影响查询条件。

### 对比

| | 页码分页 | 游标分页 |
|--|----------|----------|
| 请求方式 | page=2 | latest_time=1000 |
| 数据变化时 | 可能重复或漏数据 | 不会重复 |
| SQL | OFFSET 跳过 | WHERE 条件过滤 |
| 场景 | 后台管理系统 | Feed 流、时间线 |

---

## 游标分页的实现

### 服务器代码

```go
func (repo *FeedRepository) ListLatest(ctx context.Context, limit int, latestBefore time.Time) ([]*video.Video, error) {
    query := repo.db.WithContext(ctx).Model(&video.Video{}).Order("create_time DESC")

    // 关键：如果有游标，加上WHERE条件
    if !latestBefore.IsZero() {
        query = query.Where("create_time < ?", latestBefore)
    }

    query.Limit(limit).Find(&videos)
    return videos, nil
}
```

### 游标怎么生成的

```go
// 取本页最后一条视频的时间作为游标
var nextTime int64
if len(baseVideos) > 0 {
    nextTime = baseVideos[len(baseVideos)-1].CreateTime.UnixMilli()
}
```

本页最后一条视频的时间 = 下一页的起点。

### 响应结构

```go
type ListLatestResponse struct {
    VideoList []FeedVideoItem `json:"video_list"`
    NextTime  int64           `json:"next_time"`   // 游标
    HasMore   bool            `json:"has_more"`     // 还有没有更多
}
```

### 完整流程

```
客户端：GET /feed?limit=10
    ↓
服务器：SELECT * FROM videos ORDER BY create_time DESC LIMIT 10
    ↓
返回：video_list + next_time = 本页最后一条的时间
    ↓
客户端：GET /feed?limit=10&latest_time=next_time
    ↓
服务器：SELECT * FROM videos WHERE create_time < next_time ORDER BY create_time DESC LIMIT 10
    ↓
返回：video_list + next_time = 新的游标
    ↓
...循环，直到 has_more = false
```

---

## 热榜实现

### 热榜排序算法

```go
query := repo.db.WithContext(ctx).Model(&video.Video{}).
    Order("popularity DESC, create_time DESC, id DESC")
```

排序规则：热度高的优先，热度相同按时间倒序，时间也相同按 ID 倒序。

### 热度分数怎么算的

```go
func UpdatePopularityCache(ctx context.Context, cache *rediscache.Client, id uint, change int64) {
    now := time.Now().UTC().Truncate(time.Minute)
    windowKey := cache.Key("hot:video:1m:%s", now.Format("200601021504"))
    member := strconv.FormatUint(uint64(id), 10)

    cache.ZincrBy(opCtx, windowKey, member, float64(change))  // 热度+change
    cache.Expire(opCtx, windowKey, 2*time.Hour)
}
```

每次点赞/评论/关注，热度+1。

### 热榜查询流程（冷热分离）

```go
func (f *FeedService) ListByPopularity(ctx context.Context, limit int, reqAsOf int64, offset int, ...) {
    // 1. 确定时间窗口（最近60分钟）
    asOf := time.Now().UTC().Truncate(time.Minute)

    // 2. 生成60个ZSET的key
    keys := make([]string, 0, 60)
    for i := 0; i < 60; i++ {
        keys = append(keys, f.rediscache.Key("hot:video:1m:%s", asOf.Add(-time.Duration(i)*time.Minute).Format("200601021504")))
    }

    // 3. 合并成一个ZSET（快照）
    dest := f.rediscache.Key("hot:video:merge:1m:%s", asOf.Format("200601021504"))
    f.rediscache.ZUnionStore(opCtx, dest, keys, "SUM")

    // 4. 从合并后的ZSET取top N
    members, _ := f.rediscache.ZRevRange(opCtx, dest, start, stop)

    // 5. 根据ID去查视频详情
    videos, _ := f.repo.GetByIDs(ctx, ids)

    return videos
}
```

### 快照 key 是什么

快照 key 就是"合并结果的缓存"。

```
没有快照key：
    用户A请求 → 合并60个ZSET → 返回
    用户B请求 → 合并60个ZSET → 返回
    100个用户请求 → 合并100次 → 浪费！

有快照key：
    用户A请求 → 合并60个ZSET → 存到快照key → 返回
    用户B请求 → 发现快照key已存在 → 直接用 → 返回
    100个用户请求 → 只合并1次 → 省资源！
```

代码：
```go
dest := f.rediscache.Key("hot:video:merge:1m:%s", asOf.Format("200601021504"))
exists, _ := f.rediscache.Exists(opCtx, dest)
if !exists {
    f.rediscache.ZUnionStore(opCtx, dest, keys, "SUM")
    f.rediscache.Expire(opCtx, dest, 2*time.Minute)
}
members, _ := f.rediscache.ZRevRange(opCtx, dest, start, stop)
```

快照 key 2 分钟后过期，下次请求重新合并，拿到最新数据。

---

## 关注流实现

### 核心代码

```go
func (repo *FeedRepository) ListByFollowing(ctx context.Context, limit int, viewerAccountID uint, latestBefore time.Time) ([]*video.Video, error) {
    var videos []*video.Video
    query := repo.db.WithContext(ctx).Model(&video.Video{}).Order("create_time DESC")

    if viewerAccountID > 0 {
        // 子查询：找到我关注的所有人
        followingSubQuery := repo.db.WithContext(ctx).
            Model(&social.Social{}).
            Select("vlogger_id").
            Where("follower_id = ?", viewerAccountID)

        // 只查这些人发的视频
        query = query.Where("author_id IN (?)", followingSubQuery)
    }

    if !latestBefore.IsZero() {
        query = query.Where("create_time < ?", latestBefore)
    }

    query.Limit(limit).Find(&videos)
    return videos, nil
}
```

### 等价 SQL

```sql
SELECT * FROM videos
WHERE author_id IN (
    SELECT vlogger_id FROM socials WHERE follower_id = 123  -- 我关注的人
)
AND create_time < 1000  -- 游标
ORDER BY create_time DESC
LIMIT 10
```

### 子查询是什么

子查询就是"查询里的查询"，拆成两步理解：

```
第1步（子查询）：
    SELECT vlogger_id FROM socials WHERE follower_id = 123
    → 结果：[456, 789]（用户A关注的人）

第2步（主查询）：
    SELECT * FROM videos WHERE author_id IN (456, 789)
    → 结果：视频100（B发的）、视频200（C发的）
```

子查询是一次 SQL 搞定，数据库内部优化，比两条 SQL 更快。

### 流程

```
用户A关注了B和C
    ↓
子查询：从social表找到 [456, 789]
    ↓
主查询：从video表找 author_id 是 456 或 789 的视频
    ↓
返回：视频100（B发的）、视频200（C发的）
    ↓
用户D的视频300不会出现（A没关注D）
```

---

## 话题流实现

### 核心代码

```go
func (repo *FeedRepository) ListByTag(ctx context.Context, tagName string, limit int) ([]*video.Video, error) {
    var videos []*video.Video
    err := repo.db.WithContext(ctx).Model(&video.Video{}).Table("videos").
        Joins("JOIN video_tags ON video_tags.video_id = videos.id").
        Joins("JOIN tags ON tags.id = video_tags.tag_id").
        Where("tags.name = ?", tagName).
        Order("videos.create_time desc").
        Limit(limit).
        Find(&videos).Error
    return videos, err
}
```

### 等价 SQL

```sql
SELECT videos.* FROM videos
JOIN video_tags ON video_tags.video_id = videos.id
JOIN tags ON tags.id = video_tags.tag_id
WHERE tags.name = 'Go语言'
ORDER BY videos.create_time DESC
LIMIT 10
```

### 三张表的关系

```
tags表：
┌────┬──────────┐
│ id │ name     │
├────┼──────────┤
│ 1  │ Go语言   │
│ 2  │ Python   │
└────┴──────────┘

video_tags表（多对多关联）：
┌──────────┬─────────┐
│ video_id │ tag_id  │
├──────────┼─────────┤
│ 100      │ 1       │  ← 视频100有"Go语言"标签
│ 100      │ 2       │  ← 视频100也有"Python"标签
│ 200      │ 1       │  ← 视频200有"Go语言"标签
└──────────┴─────────┘

videos表：
┌────┬──────────┐
│ id │ title    │
├────┼──────────┤
│ 100 │ Go入门  │
│ 200 │ 并发编程 │
└────┴──────────┘
```

### 查询流程

```
1. 从tags表找到"Go语言"的id = 1
2. 从video_tags表找到tag_id = 1的video_id = [100, 200]
3. 从videos表找到id = [100, 200]的视频

JOIN一次搞定。
```

---

## 三级缓存架构

这个项目在 Feed 查询里用了三级缓存：

```
请求 → L1本地缓存 → L2 Redis → L3 MySQL
         ↓ 快           ↓ 较快              ↓ 慢
        纳秒级          毫秒级              毫秒级
```

### 代码

```go
type FeedService struct {
    localcache   *cache.Cache      // L1：本地缓存（进程内存，3秒过期）
    rediscache   *rediscache.Client // L2：Redis缓存
    repo         *FeedRepository    // L3：MySQL
}

// L1：本地缓存
localcache: cache.New(3*time.Second, 5*time.Second)

// L2：Redis缓存（50ms超时）
cacheCtx, cancel := context.WithTimeout(ctx, 50*time.Millisecond)

// L3：MySQL
videos, err := f.repo.GetByIDs(ctx, ids)
```

### 查询流程

```
请求来了
    ↓
查L1本地缓存（进程内存）
    ├── 命中 → 直接返回（最快）
    └── 没命中
            ↓
        查L2 Redis（50ms超时）
            ├── 命中 → 写回L1 → 返回
            └── 没命中
                    ↓
                查L3 MySQL
                    → 写回L2 Redis → 写回L1 → 返回
```

---

## singleflight 防并发

### 什么是 singleflight

Go 内置的工具，保证同一个请求只执行一次，其他并发请求等着用结果。

### 场景

```
100个用户同时请求热榜
    ↓
没有singleflight：
    用户1：查DB → 返回
    用户2：查DB → 返回
    ...
    用户100：查DB → 返回
    DB被查了100次！

有singleflight：
    用户1：查DB → 返回
    用户2-100：等着 → 用户1查完后直接用结果
    DB只被查了1次！
```

### 代码

```go
type FeedService struct {
    requestGroup singleflight.Group
}

v, err, _ := f.requestGroup.Do(sfKey, func() (interface{}, error) {
    return f.repo.GetByIDs(ctx, []uint{videoID})
})
```

- 第 1 个请求：执行 func，查 DB
- 其他请求：同一个 sfKey，等着，拿到同样的结果

### sfKey 是什么

请求的唯一标识：

```go
sfKey := f.rediscache.Key("sf:entity:%d", videoID)
// 结果："sf:entity:123"
```

同一个视频 ID 的请求，sfKey 相同，会共享结果。

### 和分布式锁的对比

| | 分布式锁 | singleflight |
|--|----------|--------------|
| 作用范围 | 多台机器 | 单机内 |
| 实现 | Redis | Go 内存 |
| 复杂度 | 高 | 低 |
| 场景 | 分布式系统 | 单机内防并发 |

---

## 面试要点总结

### Q: 推模型和拉模型的区别？

A: 推是发视频时推送给粉丝，拉是刷视频时实时查 DB。推的读快写慢，拉的读慢写快。抖音用推拉结合。

### Q: 游标分页和页码分页的区别？

A: 游标用 WHERE 条件过滤，不会重复；页码用 OFFSET 跳过，数据变化时可能重复。Feed 流用游标分页。

### Q: 热榜怎么实现的？

A: ZSET 按分钟分窗口，点赞时热度+1，查询时合并 60 个窗口算总分，快照 key 缓存合并结果。

### Q: 关注流怎么实现的？

A: 子查询找关注列表，再查这些人发的视频。一次 SQL 搞定。

### Q: 话题流怎么实现的？

A: 三表 JOIN（videos + video_tags + tags），查某个标签下的视频。

### Q: 什么是 singleflight？

A: Go 内置工具，保证同一个请求只执行一次，其他并发请求等着用结果。和分布式锁类似，但作用于单机内。

### Q: 三级缓存是什么？

A: L1 本地缓存（进程内存）→ L2 Redis → L3 MySQL。先查快的，没命中再查慢的。
