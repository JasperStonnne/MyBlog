---
title: Go 项目反推：Feed 流系统实战——SSE 实时推送与分片上传
slug: go-feed-sse-upload
description: 拆解 Feed 流项目中 SSE 通知系统与分片上传、断点续传的核心设计
date: 2026-07-29T00:00:00+08:00
draft: false
image: cover.svg
tags:
  - Go
  - SSE
  - 分片上传
  - 断点续传
  - RabbitMQ
  - Feed 流
categories:
  - 项目记录
---

承接 [Go 项目反推：Feed 流系统实战——Feed 流设计]({{< ref "go-feed-design" >}})，这一篇继续拆解 `feedsystem_video_go` 的两个关键能力：用 SSE 实时推送通知，以及用分片上传和断点续传处理大文件。

# 阶段6：SSE实时推送

## SSE 是什么

SSE = Server-Sent Events，服务器主动给客户端推送消息。

就像外卖 APP 的订单状态：
```
你下单 → 服务器推送"商家已接单"
      → 服务器推送"骑手已取餐"
      → 服务器推送"骑手已送达"
```

客户端不用一直问"好了没？好了没？"，服务器有消息就主动推。

---

## SSE vs WebSocket

| | SSE | WebSocket |
|--|-----|-----------|
| 方向 | 服务器 → 客户端（单向） | 双向 |
| 协议 | HTTP | 独立协议 |
| 复杂度 | 简单 | 复杂 |
| 场景 | 通知、状态更新、新闻推送 | 聊天、游戏、实时协作 |

这个项目用 SSE 推送通知（点赞、评论、关注），因为只需要服务器→客户端，不需要客户端→服务器。

---

## 为什么用 SSE 不用轮询

| 方案 | 做法 | 问题 |
|------|------|------|
| 轮询 | 客户端每隔几秒问一次"有新消息吗？" | 浪费资源、实时性差、大部分请求返回空 |
| SSE | 服务器有消息就推，没有就等着 | 省资源、实时性高 |

---

## 项目里 SSE 用在哪

```
用户A给用户B的视频点赞
    ↓
LikeWorker处理点赞事件
    ↓
NotificationWorker收到MQ消息
    ↓
创建Notification记录，写入DB
    ↓
SSEHub.Push(userID, notification)
    ↓
用户B的浏览器收到推送，实时显示"xxx点赞了你的视频"
```

---

## SSEHub 数据结构

```go
type SSEHub struct {
    mu      sync.RWMutex
    clients map[uint][]chan *Notification  // userID -> 多个通知channel
    db      *gorm.DB
}
```

数据结构：
```
map[uint][]chan *Notification

用户1 → [channel1, channel2]  （可能多个设备）
用户2 → [channel3]
用户3 → [channel4, channel5, channel6]  （三个设备）
```

---

## 代码实现

### 相关文件

```
backend/internal/worker/ssehub.go           # SSE Hub，管理所有SSE连接
backend/internal/worker/notificationworker.go # 通知Worker，消费MQ消息并推送
```

### Notification 表结构

```go
type Notification struct {
    ID          uint
    RecipientID uint      // 接收者ID
    SenderID    uint      // 发送者ID
    Type        string    // 通知类型：like、comment、follow
    TargetID    uint      // 目标ID（视频ID、用户ID）
    Content     string    // 通知内容
    IsRead      bool      // 是否已读
    CreatedAt   time.Time
}
```

### SSEHub 核心方法

```go
// 订阅：用户连接时调用
func (h *SSEHub) Subscribe(userID uint) chan *Notification {
    ch := make(chan *Notification, 20)  // 缓冲区20
    h.mu.Lock()                         // 写锁
    h.clients[userID] = append(h.clients[userID], ch)
    h.mu.Unlock()
    return ch
}

// 取消订阅：用户断开时调用
func (h *SSEHub) Unsubscribe(userID uint, ch chan *Notification) {
    h.mu.Lock()                         // 写锁
    defer h.mu.Unlock()
    chs := h.clients[userID]
    for i, c := range chs {
        if c == ch {
            chs = append(chs[:i], chs[i+1:]...)
            if len(chs) == 0 {
                delete(h.clients, userID)
            } else {
                h.clients[userID] = chs
            }
            close(c)
            return
        }
    }
}

// 推送：有新通知时调用
func (h *SSEHub) Push(userID uint, n *Notification) {
    h.mu.RLock()                        // 读锁
    defer h.mu.RUnlock()
    chs, ok := h.clients[userID]
    if !ok {
        return
    }
    for _, ch := range chs {
        select {
        case ch <- n:   // 往channel里塞消息
        default:        // 塞不进去就跳过（防阻塞）
        }
    }
}
```

### SSEHandler - 处理 SSE 连接

```go
func (h *SSEHub) SSEHandler(c *gin.Context) {
    userID, ok := sseAccountID(c)
    if !ok {
        c.JSON(http.StatusUnauthorized, gin.H{"error": "invalid account"})
        return
    }

    // 设置SSE响应头
    c.Writer.Header().Set("Content-Type", "text/event-stream")
    c.Writer.Header().Set("Cache-Control", "no-cache")
    c.Writer.Header().Set("Connection", "keep-alive")
    c.Writer.WriteHeader(http.StatusOK)

    // 订阅通知
    ch := h.Subscribe(userID)
    defer h.Unsubscribe(userID, ch)

    ctx := c.Request.Context()
    flusher, _ := c.Writer.(http.Flusher)

    for {
        select {
        case <-ctx.Done():          // 客户端断开
            return
        case n, ok := <-ch:         // 收到通知
            if !ok {
                return
            }
            b, _ := json.Marshal(n)
            fmt.Fprintf(c.Writer, "data: %s\n\n", b)  // SSE格式
            if flusher != nil {
                flusher.Flush()     // 立刻发送给客户端
            }
        case <-time.After(30 * time.Second):  // 心跳
            fmt.Fprintf(c.Writer, ": keepalive\n\n")
            if flusher != nil {
                flusher.Flush()
            }
        }
    }
}
```

---

## Push 为什么用 RLock 不用 Lock

### 什么是读写锁（RWMutex）

普通锁（Mutex）：同一时刻只能一个人进去，不管是读还是写。

读写锁（RWMutex）：
- 多个人可以同时读（RLock）
- 写的时候只能一个人进去，其他人不能读也不能写（Lock）

### 对比

| 操作 | 做什么 | 用什么锁 |
|------|--------|----------|
| Push | 读 map，往 channel 塞消息 | RLock（读锁） |
| Subscribe | 往 map 里加一个 channel | Lock（写锁） |
| Unsubscribe | 从 map 里删一个 channel | Lock（写锁） |

### 为什么 Push 能用 RLock

Push 只做两件事：
1. 从 map 里找到用户的 channel（读）
2. 往 channel 里塞消息（不改 map）

map 结构没变，所以多个 Push 可以同时进行。

### 一句话：读用 RLock（可以并发），写用 Lock（必须独占）。

### 场景对比

```
场景：100个用户同时收到通知

用普通锁（Mutex）：
    Push1加锁 → 推送 → 解锁
    Push2加锁 → 推送 → 解锁
    ... 一个一个来，慢

用读写锁（RWMutex）：
    Push1加RLock → 推送
    Push2加RLock → 推送
    Push3加RLock → 推送
    ... 100个同时推，快
```

---

## 心跳机制

### 什么是心跳

SSE 连接可能因为网络问题断开，客户端不知道。

解决方案：每 30 秒发一次空消息（keepalive），告诉客户端"我还活着"。

```go
case <-time.After(30 * time.Second):
    fmt.Fprintf(c.Writer, ": keepalive\n\n")
    flusher.Flush()
```

### 为什么需要心跳

```
没有心跳：
    客户端连接服务器 → 网络抖动 → 连接断了
    客户端不知道断了 → 一直等 → 漏掉通知

有心跳：
    客户端连接服务器 → 每30秒收到keepalive
    30秒没收到 → 知道断了 → 重新连接
```

---

## select 里的 default 防阻塞

```go
for _, ch := range chs {
    select {
    case ch <- n:   // 往channel里塞消息
    default:        // 塞不进去就跳过
    }
}
```

channel 缓冲区满了（20条），塞不进去怎么办？

- 没有 default：阻塞等着，Push 卡住，其他用户的推送也卡住
- 有 default：直接跳过，不等，保证其他用户能正常收到推送

这是**防阻塞**设计：一个用户的消息堆积不影响其他用户。

---

## NotificationWorker 处理流程

```go
func (w *NotificationWorker) process(ctx context.Context, d amqp.Delivery) error {
    routingKey := d.RoutingKey

    var notif *Notification

    switch {
    case routingKey == "like.like":
        // 解析点赞事件
        // 查询视频作者ID
        // 如果是自己给自己点赞，不通知
        notif = &Notification{RecipientID: authorID, SenderID: evt.UserID, Type: "like", Content: "点赞了你的视频"}

    case routingKey == "comment.publish":
        // 解析评论事件
        // 查询视频作者ID
        // 如果是自己评论自己，不通知
        notif = &Notification{RecipientID: authorID, SenderID: evt.AuthorID, Type: "comment", Content: "评论了你的视频"}

    case routingKey == "social.follow":
        // 解析关注事件
        notif = &Notification{RecipientID: evt.VloggerID, SenderID: evt.FollowerID, Type: "follow", Content: "关注了你"}
    }

    if notif == nil {
        return nil
    }

    // 写入DB
    w.db.Create(notif)

    // 实时推送给用户
    w.hub.Push(notif.RecipientID, notif)

    return nil
}
```

---

## 接口设计

| 接口 | 方法 | 作用 |
|------|------|------|
| `/stream` | GET | SSE 连接，实时接收通知 |
| `/list` | POST | 查询通知列表（最近 50 条） |
| `/markRead` | POST | 标记已读 |
| `/unreadCount` | POST | 查询未读数量 |

---

# 阶段7：分片上传

## 为什么需要分片上传

视频文件可能很大（几十 MB 到 200MB），一次上传的问题：

```
一次上传100MB：
    网络抖动 → 上传到99%断了 → 重头再来
    用户体验极差
```

分片上传：
```
100MB文件，分成5MB一片，共20片
    第1片上传成功 ✓
    第2片上传成功 ✓
    ...
    第17片网络断了 ✗
    重新连接 → 只需要重新上传第17片，1-16不用重传
```

---

## 分片上传流程

### 整体流程

```
1. 客户端：计算文件MD5 → 调用Init接口 → 获得upload_id + 已上传分片列表
2. 客户端：逐片上传（跳过已上传的） → 每片带MD5校验
3. 客户端：全部上传完 → 调用Complete接口 → 服务器合并分片 → 返回视频URL
```

### 流程图

```
客户端                                    服务器
  |                                         |
  |--- Init(文件名, 大小, MD5) ----------->|
  |<-- upload_id + 已上传列表 -------------|
  |                                         |
  |--- UploadChunk(分片0, MD5) ----------->|  保存到临时目录
  |<-- chunk_index: 0 ----------------------|
  |                                         |
  |--- UploadChunk(分片1, MD5) ----------->|  保存到临时目录
  |<-- chunk_index: 1 ----------------------|
  |                                         |
  |    ...（中间可能断线重连）               |
  |                                         |
  |--- Complete(upload_id) --------------->|  合并所有分片
  |<-- url: 视频地址 -----------------------|
```

---

## 相关文件

```
backend/internal/video/chunk_entity.go     # 数据结构定义
backend/internal/video/chunk_handler.go    # 业务逻辑
```

---

## 核心数据结构

### ChunkUploadSession（上传会话）

```go
const ChunkSize = 5 << 20 // 5 MB

type ChunkUploadSession struct {
    UploadID     string   // 上传会话ID（随机生成）
    AccountID    uint     // 上传者ID
    Filename     string   // 原始文件名
    FileSize     int64    // 文件总大小
    ChunkSize    int64    // 每片大小（默认5MB）
    TotalChunks  int      // 总片数
    FileHash     string   // 整个文件的MD5
    UploadedBits []bool   // 每片是否已上传（位图）
}
```

### UploadedBits 是什么

```go
UploadedBits: []bool{true, true, true, false, false}
```

这个数组表示：
- 分片 0：已上传 ✓
- 分片 1：已上传 ✓
- 分片 2：已上传 ✓
- 分片 3：未上传
- 分片 4：未上传

用 bool 数组而不是存已上传的分片列表，好处是：
- 判断某片是否已上传：O(1)，直接 index
- 判断是否全部上传完：遍历一次，O(n)

### 两个辅助方法

```go
// 返回已上传的分片索引列表
func (s *ChunkUploadSession) UploadedChunks() []int {
    var indices []int
    for i, uploaded := range s.UploadedBits {
        if uploaded {
            indices = append(indices, i)
        }
    }
    return indices
}

// 判断是否全部上传完
func (s *ChunkUploadSession) IsComplete() bool {
    for _, b := range s.UploadedBits {
        if !b {
            return false
        }
    }
    return true
}
```

---

## 会话存在哪：Redis

上传会话存在 Redis，不是 MySQL，原因是：

| 存储 | 读写速度 | 持久化 | 适合存什么 |
|------|----------|--------|-----------|
| Redis | 微秒级 | 可配置过期 | 临时状态（上传进度） |
| MySQL | 毫秒级 | 永久 | 持久数据（用户信息、视频记录） |

上传会话是临时的（24 小时过期），用 Redis 更合适。

### Redis Key 设计

```go
// 上传会话
func (h *ChunkUploadHandler) sessionKey(uploadID string) string {
    return h.cache.Key("chunk_upload:%s", uploadID)
}

// 文件哈希 → upload_id 的映射（用于断点续传）
func (h *ChunkUploadHandler) hashKey(accountID uint, fileHash string) string {
    return h.cache.Key("chunk_upload_hash:%d:%s", accountID, fileHash)
}
```

两个 Key 的作用：
- `chunk_upload:<upload_id>`：存完整的上传会话
- `chunk_upload_hash:<account_id>:<file_hash>`：文件哈希到 upload_id 的映射，用于断点续传

---

## 四个接口详解

### 接口 1：InitChunkUpload（初始化上传）

```go
func (h *ChunkUploadHandler) InitChunkUpload(c *gin.Context)
```

**做了什么：**

1. 校验请求参数（文件名、大小、哈希、总片数）
2. 检查文件大小是否超过 200MB 限制
3. **断点续传检查**：用文件哈希查 Redis，如果找到之前的会话，直接返回
4. 创建新会话，存入 Redis
5. 返回 upload_id + 已上传列表（新上传为空）

**断点续传的实现：**

```go
// 检查是否有同文件的旧会话
hashKey := h.hashKey(accountID, req.FileHash)
existingID, err := h.cache.GetBytes(c.Request.Context(), hashKey)
if err == nil && len(existingID) > 0 {
    session, sessErr := h.getSession(c, string(existingID))
    if sessErr == nil {
        // 找到了！返回旧的upload_id和已上传列表
        c.JSON(http.StatusOK, gin.H{
            "upload_id":       session.UploadID,
            "uploaded_chunks": session.UploadedChunks(),
        })
        return
    }
}
```

**为什么用文件哈希做断点续传？**

```
用户上传 video.mp4（100MB，MD5: abc123）
    → Init，upload_id: xxx，开始上传
    → 上传了15片，浏览器关闭

用户重新选择同一个 video.mp4
    → Init，参数里带同样的 file_hash: abc123
    → 服务器查Redis：chunk_upload_hash:<uid>:abc123 → xxx
    → 找到旧会话！返回已上传的15片
    → 客户端跳过这15片，只上传剩下的5片
```

### 接口 2：UploadChunk（上传单片）

```go
func (h *ChunkUploadHandler) UploadChunk(c *gin.Context)
```

**做了什么：**

1. 从表单获取 upload_id、chunk_index、chunk_hash
2. 查 Redis 获取会话
3. 校验：会话存在、是本人上传、分片索引合法
4. **幂等检查**：如果这片已经上传过，直接返回成功（不重复处理）
5. 读取上传的文件内容，计算 MD5
6. **哈希校验**：对比客户端传的 chunk_hash 和实际计算的 MD5
7. 保存分片到临时目录：`.run/uploads/tmp/<upload_id>/<chunk_index>`
8. 更新 Redis 中的会话状态

**幂等设计：**

```go
if session.UploadedBits[req.ChunkIndex] {
    c.JSON(http.StatusOK, gin.H{"chunk_index": req.ChunkIndex})
    return
}
```

如果分片已上传过，直接返回成功，不重复处理。这保证了重试的安全性。

**哈希校验：**

```go
hash := md5.New()
io.Copy(hash, chunkFile)
actualHash := fmt.Sprintf("%x", hash.Sum(nil))

if actualHash != req.ChunkHash {
    c.JSON(http.StatusBadRequest, gin.H{"error": "chunk hash mismatch"})
    return
}
```

每个分片都有 MD5 校验，防止传输过程中数据损坏。

**分片存储位置：**

```
.run/uploads/tmp/
    └── <upload_id>/
        ├── 0      ← 第0片
        ├── 1      ← 第1片
        ├── 2      ← 第2片
        └── ...
```

### 接口 3：ChunkStatus（查询上传状态）

```go
func (h *ChunkUploadHandler) ChunkStatus(c *gin.Context)
```

返回：upload_id + 已上传分片列表 + 总片数。

用于客户端轮询或断线后查询进度。

### 接口 4：CompleteChunkUpload（完成上传）

```go
func (h *ChunkUploadHandler) CompleteChunkUpload(c *gin.Context)
```

**做了什么：**

1. 校验会话、校验是否本人
2. 检查是否所有分片都已上传（`IsComplete()`）
3. 如果有未上传的分片，返回错误和缺失信息
4. **合并分片**：按顺序读取所有分片，写入最终文件
5. 清理临时文件和 Redis 会话
6. 返回视频 URL

**合并过程：**

```go
tmpDir := filepath.Join(".run", "uploads", "tmp", req.UploadID)
for i := 0; i < session.TotalChunks; i++ {
    chunkPath := filepath.Join(tmpDir, fmt.Sprintf("%d", i))
    cf, _ := os.Open(chunkPath)
    io.Copy(finalFile, cf)   // 按顺序拼接
    cf.Close()
}
```

**最终文件位置：**

```
.run/uploads/
    └── videos/
        └── <account_id>/
            └── <date>/
                └── <random>.mp4
```

**清理：**

```go
// 删除临时分片
os.RemoveAll(tmpDir)

// 删除Redis会话
h.cache.Del(ctx, sessionKey)
h.cache.Del(ctx, hashKey)
```

---

## 安全设计

### 文件大小限制

```go
const maxSize = 200 << 20  // 200MB
if req.FileSize > maxSize {
    c.JSON(http.StatusBadRequest, gin.H{"error": "file size exceeds 200MB limit"})
    return
}
```

### 权限校验

每个接口都检查：会话存在 + 是本人操作。

```go
accountID, _ := jwt.GetAccountID(c)
if session.AccountID != accountID {
    c.JSON(http.StatusForbidden, gin.H{"error": "forbidden"})
    return
}
```

### 分片哈希校验

每个分片上传时都验证 MD5，防止数据损坏。

---

## 接口总结

| 接口 | 方法 | 作用 |
|------|------|------|
| `/chunk/init` | POST | 初始化上传，返回 upload_id |
| `/chunk/upload` | POST | 上传单个分片 |
| `/chunk/status` | POST | 查询上传状态 |
| `/chunk/complete` | POST | 完成上传，合并分片 |

---

## 面试要点总结

### Q: SSE 是什么？

A: Server-Sent Events，服务器主动推送消息给客户端，单向，基于 HTTP。场景：通知、状态更新、新闻推送。

### Q: SSE 和 WebSocket 怎么选？

A: 只要服务器推用 SSE（简单），需要双向通信用 WebSocket（复杂）。这个项目用 SSE 推送通知，因为只需要服务器→客户端。

### Q: 为什么用 SSE 不用轮询？

A: 轮询客户端每隔几秒问一次，浪费资源、实时性差。SSE 服务器有消息就推，省资源、实时性高。

### Q: 心跳机制是什么？

A: 每 30 秒发一次空消息（keepalive），告诉客户端"我还活着"。客户端 30 秒没收到，知道连接断了，重新连接。

### Q: Push 为什么用 RLock？

A: Push 只读 map 不改 map，用读锁可以多个 Push 并发。Subscribe/Unsubscribe 要改 map，用写锁必须独占。

### Q: select 里的 default 是干什么的？

A: 防阻塞。channel 满了塞不进去就跳过，保证一个用户的消息堆积不影响其他用户。

### Q: 为什么要分片上传？

A: 大文件一次上传，网络断了要重头来。分片后断点续传，只需重传失败的分片。

### Q: 断点续传怎么实现？

A: 用文件 MD5 做唯一标识，Init 时查 Redis 是否有同文件的旧会话。有就返回已上传列表，客户端跳过这些分片。

### Q: 上传会话存在哪？为什么？

A: 存 Redis，因为是临时状态（24 小时过期），读写快。MySQL 存持久数据。

### Q: 怎么保证分片数据没损坏？

A: 每片上传时计算 MD5，和客户端传的 chunk_hash 对比，不匹配就拒绝。

### Q: 分片重复上传怎么办？

A: 幂等设计——检查 UploadedBits，已上传的分片直接返回成功，不重复处理。

### Q: 会话里 UploadedBits 是什么？

A: bool 数组，每个元素对应一个分片是否已上传。长度等于总片数，直接用下标访问，O(1) 判断某片是否已上传。
