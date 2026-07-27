---
title: Go 项目反推：Feed 流系统实战——数据库与 GORM
slug: go-feed-database-gorm
description: 反推真实 Go Feed 流项目的数据库设计，梳理表结构、索引策略、GORM 标签、CRUD、事务、原子操作、关联查询与数据库规范化
date: 2026-07-27T00:00:00+08:00
draft: false
image: cover.svg
tags:
  - Go
  - GORM
  - MySQL
  - 数据库
  - 索引
  - 事务
categories:
  - 项目记录
---

承接 [Go 项目反推：Feed 流系统实战——认证体系]({{< ref "go-feed-authentication" >}})，这一篇继续反推 `feedsystem_video_go` 的数据库代码，理解表结构、索引设计和 GORM 的常见用法。

> 理解所有表结构、索引设计、GORM 用法。

---

## 1. entity.go 的两种结构体

entity.go 里有两种东西：

```text
1. 存数据库的（有 gorm 标签）   → Social struct、Like struct...
2. 不存数据库的（没有 gorm 标签）→ FeedVideoItem、Request、Response...
```

- `gorm:"xxx"` 标签 = 给 GORM 看的指令（怎么存数据库）
- `json:"xxx"` 标签 = 给 JSON 看的指令（怎么显示给客户端）
- 两者互不影响，各管各的

---

## 2. gorm 标签汇总

| 标签 | 含义 | 示例 |
|------|------|------|
| `gorm:"primaryKey"` | 主键（唯一标识，类似身份证号） | `ID uint gorm:"primaryKey"` |
| `gorm:"unique"` | 唯一（不能重复） | `Username string gorm:"unique"` |
| `gorm:"index"` | 普通索引（加快查询，可以重复） | `VideoID uint gorm:"index"` |
| `gorm:"uniqueIndex:idx_name"` | 唯一索引（不能重复） | `VideoID uint gorm:"uniqueIndex:idx_like_video_account"` |
| `gorm:"uniqueIndex:idx_name;not null"` | 联合唯一索引（多个字段组合唯一） | 见 Like 表 |
| `gorm:"index:idx_name,priority:1,sort:desc"` | 索引排序优先级 | 见 Video 表 |
| `gorm:"not null"` | 不能为空 | `Title string gorm:"not null"` |
| `gorm:"default:false"` | 默认值 | `IsRead bool gorm:"default:false"` |
| `gorm:"autoCreateTime"` | 自动填创建时间 | `CreateTime time.Time gorm:"autoCreateTime"` |
| `gorm:"type:varchar(255)"` | 字符串长度限制 | `Title string gorm:"type:varchar(255)"` |
| `gorm:"type:text"` | 长文本（无固定限制） | `Content string gorm:"type:text"` |
| `gorm:"column:likes_count"` | 指定数据库列名 | Go 驼峰 → DB 下划线 |
| `gorm:"-"` | 不存数据库 | 临时字段 |

---

## 3. json 标签

| 标签 | 含义 |
|------|------|
| `json:"id"` | JSON 里显示为 "id" |
| `json:"-"` | JSON 里不显示（密码、Token 等敏感信息） |
| `json:"description,omitempty"` | 空值时不显示这个字段 |

---

## 4. 表结构一览

### Account 表（用户表）

```text
ID           → 主键
Username     → 唯一（不能重复注册）
Password     → 不显示在 JSON 里
Token        → 不显示在 JSON 里
RefreshToken → 不显示在 JSON 里
AvatarURL    → 头像地址
Bio          → 个人简介
```

### Video 表（视频表）

```text
ID          → 主键
AuthorID    → 外键（关联 Account 表，存的是用户ID）
Username    → 作者名（冗余存储，方便查询）
Title       → 标题
Description → 描述
PlayURL     → 播放地址
CoverURL    → 封面地址
CreateTime  → 自动填创建时间
LikesCount  → 点赞数（统计字段）
Popularity  → 热度值（统计字段）
```

**Video 表的三个索引**：

| 索引名 | 排序规则 | 用途 |
|--------|----------|------|
| idx_videos_create_time | CreateTime DESC | 最新视频 |
| idx_videos_likes_count_id | LikesCount DESC | 最热视频（按点赞） |
| idx_videos_popularity_time_id | Popularity DESC, CreateTime DESC | 综合热度 |

priority = 优先级，数字越小越优先。

### Like 表（点赞表）

```text
ID        → 主键
VideoID   → 外键 + 联合唯一索引
AccountID → 外键 + 联合唯一索引
```

**联合唯一索引**：VideoID + AccountID 的组合不能重复

- 张三给视频 A 点赞 ✓
- 张三再给视频 A 点赞 ✗（违反唯一索引）
- 李四给视频 A 点赞 ✓（不同人）
- 张三给视频 B 点赞 ✓（不同视频）

**为什么需要 Like 表 + Video 表的 LikesCount？**

```text
Like 表 → 记录"关系"（谁点了谁）
LikesCount → 记录"总数"（点赞数）
两者配合：Like 管关系，LikesCount 管统计
```

### Comment 表（评论表）

```text
ID        → 主键
Username  → 普通索引
VideoID   → 外键 + 普通索引
AuthorID  → 外键 + 普通索引
Content   → 长文本（type:text）
CreatedAt → 自动填创建时间
```

**index vs uniqueIndex**：

- index = 普通索引（可以重复，一个视频可以有很多评论）
- uniqueIndex = 唯一索引（不能重复）

### Social 表（关注表）

```text
ID         → 主键
FollowerID → 外键 + 联合唯一索引（谁在关注）
VloggerID  → 外键 + 联合唯一索引（关注谁）
```

联合唯一索引：一个人不能关注同一个人两次。

### Tag 表 + VideoTag 表（标签表）

```text
Tag 表：
  ID   → 主键
  Name → 唯一索引（标签名不能重复）

VideoTag 表（中间表）：
  ID      → 主键
  VideoID → 外键 + 普通索引
  TagID   → 外键 + 普通索引
```

**多对多关系**：一个视频可以有多个标签，一个标签可以属于多个视频，需要中间表 VideoTag 来记录关系。

### Message 表（私信表）

```text
ID        → 主键
FromID    → 外键 + 普通索引（谁发的）
ToID      → 外键 + 普通索引（发给谁）
Content   → 长文本
IsRead    → 是否已读（默认 false）
CreatedAt → 自动填创建时间
```

### ChunkUploadSession（分片上传）

不是 MySQL 表，存 Redis 的。

```text
UploadID     → 上传会话ID
AccountID    → 谁在上传
UploadedBits → []bool 位图，记录每片上传状态
```

### FeedVideoItem（Feed 流响应格式）

不是数据库表，是 API 响应格式。

```text
Author  → FeedAuthor（包含 ID + Username，不是外键）
IsLiked → 动态计算（查 Like 表），不存在数据库里
```

---

## 5. db.go（数据库初始化）

三个函数：

```text
NewDB()       → 连接数据库（DSN 格式：用户名:密码@tcp(地址:端口)/数据库名）
AutoMigrate() → 根据 struct 自动建表/更新表
CloseDB()     → 关闭数据库连接
```

**AutoMigrate 的好处**：

- 手动建表：写 SQL，容易忘、容易错
- AutoMigrate：改 struct，重启程序，自动更新表

---

## 6. repo.go（数据库操作）

每个模块都有自己的 repo.go，包含增删改查操作。

### 基础套路

```go
// 创建
ar.db.WithContext(ctx).Create(&data)

// 查询（根据ID）
ar.db.WithContext(ctx).First(&result, id)

// 查询（根据条件）
ar.db.WithContext(ctx).Where("username = ?", name).First(&result)

// 更新（单字段）
ar.db.WithContext(ctx).Model(&Account{}).Where("id = ?", id).Update("username", newName)

// 更新（多字段）
ar.db.WithContext(ctx).Model(&Account{}).Where("id = ?", id).Updates(map[string]interface{}{"token": "", "refresh_token": ""})

// 删除
ar.db.WithContext(ctx).Delete(&Like{}).Where("video_id = ? AND account_id = ?", vID, aID)
```

### db.WithContext(ctx) 的作用

把 context 传进去，支持超时取消。

### 事务（db.Transaction）

```go
ar.db.WithContext(ctx).Transaction(func(tx *gorm.DB) error {
    // 操作1
    tx.Model(&Account{}).Where("id = ?", id).Update("username", newName)
    // 操作2
    tx.Model(&Account{}).Where("id = ?", id).Update("token", token)
    return nil
})
```

事务 = 要么全做，要么全不做。

类比：银行转账，A 扣钱与 B 加钱必须同时成功或同时失败。

### 原子操作（gorm.Expr）

```text
// 错误做法：先读再写，并发会出错
likes_count = 100
用户A读到100，用户B读到100
用户A写入101，用户B写入101
结果：只加了1 ❌

// 正确做法：让数据库直接加
gorm.Expr("likes_count + ?", 1)
数据库排队执行：100+1=101, 101+1=102
结果：正确加了2 ✓
```

**GREATEST(likes_count + ?, 0)**：防止点赞数变成负数。

### LikeIgnoreDuplicate（忽略重复点赞）

```go
err = r.db.WithContext(ctx).Create(like).Error
if err == nil {
    return true, nil  // 创建成功
}
// 如果是"重复主键"错误（1062），忽略
var mysqlErr *mysql.MySQLError
if errors.As(err, &mysqlErr) && mysqlErr.Number == 1062 {
    return false, nil  // 已经点过赞了，不报错
}
```

**为什么需要这个？**

- 联合唯一索引：保证数据正确（一人只能点一次）
- LikeIgnoreDuplicate：保证用户体验（重复点赞不报错）
- 两个配合：数据对 + 体验好

### gorm.ErrRecordNotFound（判断记录不存在）

```go
if err == gorm.ErrRecordNotFound {
    return false, nil  // 记录不存在
}
```

### 关联查询（Joins）

```go
// 查"张三点赞过的所有视频"
r.db.WithContext(ctx).
    Model(&Video{}).
    Joins("JOIN likes ON likes.video_id = videos.id").
    Where("likes.account_id = ?", accountID).
    Find(&videos)
```

### 分步查询（GetAllFollowers）

```sql
-- 第1步：从 Social 表查出 follower_id 列表
SELECT follower_id FROM socials WHERE vlogger_id = 1;
-- 得到 [2, 5, 8]

-- 第2步：用 ID 列表去 Account 表查用户信息
SELECT * FROM accounts WHERE id IN (2, 5, 8);
```

**为什么分两步？**

Social 表只存 ID（避免数据冗余），Account 表存用户信息。用户改名只改 Account 表，Social 表不用动。

---

## 7. 数据库规范化

同一份数据只存一个地方，其他地方只存 ID（外键）。

```text
反范例：Social 表存用户名
  用户改名 → 要更新 Account 表 + Social 表 + 其他表...
  牵一发动全身

正范例：Social 表只存 ID
  用户改名 → 只改 Account 表
  需要名字时，拿着 ID 去 Account 表查
```

---

## 8. 面试要点

### 1. Video 表为什么有三个索引？

因为有三种查询需求：

- `idx_videos_create_time` → 按时间排序（查最新视频）
- `idx_videos_likes_count_id` → 按点赞数排序（查最热视频）
- `idx_videos_popularity_time_id` → 按热度 + 时间排序（综合排名）

每种查询方式都需要自己的“目录”（索引），没有索引就要全表扫描，很慢。

### 2. 点赞表为什么使用联合唯一索引？

点赞表用 `uniqueIndex:idx_like_video_account`，保证 VideoID + AccountID 的组合唯一。一个人只能给一个视频点一次赞，如果重复点赞会违反唯一索引报错。配合 LikeIgnoreDuplicate 函数，重复点赞时静默忽略，不向用户报错。

### 3. gorm.Expr 是怎么解决并发问题的？

错误做法：先读 `likes_count=100`，再写 `100+1=101`。两个用户同时读到 100，都写入 101，结果只加了 1。

正确做法：用 `gorm.Expr("likes_count + ?", 1)`，让数据库直接执行 `likes_count = likes_count + 1`，数据库保证原子更新，结果正确。`GREATEST(likes_count + ?, 0)` 可以防止点赞数变成负数。

### 4. 改名 + 更新 Token 为什么要用事务？

因为改名后 Token 里的 Username 就过期了，必须同时更新。事务保证“要么全做，要么全不做”：改名成功但更新 Token 失败时回滚，改名也不生效。

### 5. 为什么 Social 表只存 ID，不存用户名？

如果存用户名，用户改名时要更新 Account 表、Social 表和其他表，牵一发动全身。只存 ID 时，用户改名只改 Account 表，Social 表不用动；需要名字时拿着 ID 去 Account 表查询。

### 6. Tag 和 Video 为什么需要中间表？

一个视频可以有多个标签，一个标签也可以属于多个视频，这是多对多关系。一个外键字段只能存一个值，因此需要中间表 VideoTag 存储 VideoID + TagID，记录“哪个视频有哪些标签”。
