---
title: Go 项目反推：Feed 流系统实战——从 Docker 到开源贡献
slug: go-feed-system-reverse-engineering
description: 通过反推一个 Go + Vue 3 短视频 Feed 流系统，理解 Docker Compose、Gin 路由、Redis 缓存、RabbitMQ 消息队列与 Worker 进程模式，并走通一次完整的开源贡献流程
date: 2026-07-19T00:00:00+08:00
draft: false
image: cover.svg
tags:
  - Go
  - Docker
  - Gin
  - Redis
  - RabbitMQ
  - 开源贡献
categories:
  - 项目记录
---

承接 [Go 语言基础：从变量到指针]({{< ref "go-basics-variables-to-pointers" >}})、[Go 语言进阶：方法、接口与错误处理]({{< ref "go-methods-interfaces-errors" >}})、[Go 语言工程化：模块管理与项目结构]({{< ref "go-engineering-modules-project-structure" >}})、[Go 语言并发：从 Goroutine 到 Worker Pool]({{< ref "go-concurrency-goroutines-worker-pool" >}}) 和 [Go Web 开发：从 net-http 到 Gin]({{< ref "go-web-net-http-to-gin" >}})，这一篇跳出语法学习，反推一个真实的开源项目——[feedsystem_video_go](https://github.com/LeoninCS/feedsystem_video_go)（279 star，Go + Vue 3 短视频 Feed 流系统），从跑通项目到提交 PR，走一遍完整的工程实战。

## 项目概览

这个项目是一个短视频 Feed 流系统，技术栈：

- **后端**：Go + Gin + GORM + RabbitMQ + Redis
- **前端**：Vue 3 + TypeScript
- **基础设施**：Docker Compose 编排 MySQL、Redis、RabbitMQ

核心功能包括用户注册登录、视频发布、点赞评论、关注取关、Feed 流推送。架构上采用 API + Worker 的消息队列模式。

## Docker Compose：启动基础设施

项目的三个"帮手"——MySQL、Redis、RabbitMQ——通过 Docker Compose 一键启动：

```bash
cd ~/dev/projects/feedsystem_video_go
docker compose up -d mysql redis rabbitmq
```

- `docker compose up` — 根据 `docker-compose.yml` 启动服务
- `-d` — 后台运行（detach），不占终端窗口
- `mysql redis rabbitmq` — 只启动这三个，不启动 backend/worker/frontend

### 镜像 vs 容器

Docker 里有两个核心概念：

- **镜像** = 打包好的集装箱（里面有程序 + 配置），是静态文件
- **容器** = 集装箱正在运行的实例

类比：`.app` 文件是镜像，双击打开后正在运行就是容器。

### 端口映射

MySQL 默认端口 3306，但本机可能已经装了 MySQL 占用了 3306。所以 `docker-compose.yml` 里写的是：

```yaml
ports: "3307:3306"
```

意思是：外部访问 3307 → 转发到容器内部的 3306。就像你家门牌号是 3307，但内部房间编号是 3306。项目配置文件里写的也是 3307，Go 程序连接 MySQL 时用的就是 3307。

通过 `docker compose ps` 可以查看所有容器的运行状态：

```
NAME                              IMAGE          STATUS                  PORTS
feedsystem_video_go-mysql-1       mysql:8.0      Up 5 minutes (healthy)  0.0.0.0:3307->3306/tcp
feedsystem_video_go-redis-1       redis:7-alpine Up 5 minutes (healthy)  0.0.0.0:6379->6379/tcp
feedsystem_video_go-rabbitmq-1    rabbitmq:3     Up 5 minutes (healthy)  0.0.0.0:5672->5672/tcp
```

`0.0.0.0` 表示"接受任何来源的连接"，即对外部开放。

## 启动 Go 后端

基础设施就位后，启动后端：

```bash
cd backend
CONFIG_PATH=configs/config.compose-local.yaml go run ./cmd
```

- `CONFIG_PATH=...` — 告诉程序用哪个配置文件（里面写好了 MySQL 3307、Redis 6379、RabbitMQ 5672 的地址）
- `go run ./cmd` — 编译并运行 `cmd/main.go` 入口程序

### 日志解读

启动后会看到几段日志：

**第一段：依赖下载**

```
go: downloading github.com/gin-gonic/gin v1.11.0
go: downloading gorm.io/gorm v1.31.1
```

Go 在下载第三方库，类似 `npm install`，下载一次以后就不需要了。

**第二段：连接建立**

```
Loading config from configs/config.compose-local.yaml
Redis connected (cache enabled)
RabbitMQ connected
```

注意没有 "MySQL connected"，因为 GORM 连接 MySQL 是在后面 AutoMigrate 时才真正连接。

**第三段：Gin 路由注册**

```
[GIN-debug] GET /healthz
[GIN-debug] POST /account/register
[GIN-debug] POST /account/login
[GIN-debug] POST /video/publish
[GIN-debug] POST /like/like
[GIN-debug] POST /feed/listLatest
```

格式是 `HTTP方法 路由路径`，后面的 `(4 handlers)` 表示这条路由经过几个中间件（如限流 + JWT 鉴权 + 业务处理）。

**第四段：消息队列**

```
DLX ready: exchange=dlx.events queue=like.events.dlx
Timeline consumer 已启动, queue=video.timeline.update.queue
```

DLX = Dead Letter Exchange（死信队列），专门处理失败的消息。

**第五段：启动完成**

```
Server is running on port 8080
```

### 为什么不把 Go 也放进 Docker？

Docker 适合最终部署，不适合日常开发调试。本地直接 `go run` 的好处：

- 日志直接打印在终端，实时可见
- 改了代码立刻能跑，不用重新打包镜像
- 出错了能直接看到报错信息

### 日志不是中间件

日志是 Go 标准库 `log` 包写出来的，是程序员手动写的"程序进行到哪一步了"的提示。中间件是处理请求的"关卡"，两者没有关系。

## 核心概念：缓存与消息队列

### Redis 缓存

缓存 = 把经常用的数据放在"取用更快的地方"。

类比：冰箱里有水（MySQL，容量大但取用慢），你拿了一瓶放在桌上（Redis 缓存，取用极快），下次喝水直接从桌上拿，不用再去冰箱。

代码里 `cache enabled` 表示数据会先从 Redis 取，取不到再去 MySQL。

### RabbitMQ 消息队列

消息队列是 API 和 Worker 之间的"传话筒"。API 处理完核心逻辑后，把"后续要做的事"发到队列，Worker 从队列里取任务执行。

## 前后端分离架构

后端和前端是两个完全不同的程序：

- **后端**：Go 写的，负责业务逻辑、读写数据库，监听 8080 端口
- **前端**：Vue 写的，负责页面展示、用户交互，监听 5173 端口

它们通过 HTTP 请求通信：用户在浏览器点"登录" → 前端发请求到 `localhost:8080/account/login` → 后端处理 → 返回结果 → 前端展示。

就像餐厅里厨房（后端）和前厅（前端）是两个地方，前厅接到顾客点单传给厨房，厨房做完菜传给前厅。

前端用 npm（Node Package Manager）管理依赖，跟 Go 的 `go get`、Python 的 `pip install` 是同一个道理：

```bash
cd frontend
npm install    # 下载依赖
npm run dev    # 启动开发服务器
```

## Worker 进程模式

这是整个项目最核心的架构设计。Worker 是一个独立进程，和 API 分开启动：

```bash
cd backend
CONFIG_PATH=configs/config.compose-local.yaml go run ./cmd/worker
```

### 为什么要分开？

想象一个外卖平台：

**方案 A：一个人又要接单又要送餐** — 接单时有新订单没人接，送餐时又有新订单没人接，效率低。

**方案 B：两个人分工** — 前台专门接单（API），骑手专门送餐（Worker），前台接完单把订单放进"待送餐队列"（RabbitMQ），骑手从队列里取订单去送。

对应到项目：

| 角色 | 职责 |
|------|------|
| API 进程 | 接收用户请求（点赞、评论、发布视频），处理核心业务逻辑，把"后续要做的事"发到消息队列 |
| Worker 进程 | 监听消息队列，处理异步任务：更新点赞数、计算热度、推送通知 |

这个项目有四个 Worker：

- **LikeWorker** — 处理点赞/取消点赞事件
- **PopularityWorker** — 更新视频热度
- **SocialWorker** — 处理关注/取关事件
- **CommentWorker** — 处理评论事件

### 分开的好处

1. **性能**：用户点赞时，API 只需要返回"点赞成功"，不需要等热度计算完成
2. **独立伸缩**：请求量大时启动多个 API 进程，任务多时启动多个 Worker 进程
3. **故障隔离**：Worker 挂了 API 还能正常接收请求，API 挂了 Worker 还能把队列里的任务处理完

一句话总结：**API 是"接单的"，Worker 是"干活的"，通过消息队列解耦。**

## 开源贡献实战：修复 JWT 密钥缓存 Bug

跑通项目后，在实际使用中发现了一个真实 Bug。

### 问题复现

1. 不设置 `JWT_SECRET` 环境变量，启动后端
2. 注册一个账号
3. 登录成功（页面提示"登录成功"）
4. 但左侧栏仍然显示"未登录"
5. 浏览器控制台报错：`invalid or expired token (401)`

登录本身成功，但所有需要鉴权的接口全部返回 401。

### 根因定位

文件：`backend/internal/auth/jwt.go`

```go
func jwtSecret() []byte {
    secret := os.Getenv("JWT_SECRET")
    if secret == "" {
        b := make([]byte, 32)
        rand.Read(b)
        secret = hex.EncodeToString(b)
    }
    return []byte(secret)
}
```

问题出在这个函数**每次调用都生成一个新的随机密钥**：

- 登录时：`GenerateToken()` 调用 `jwtSecret()` → 拿到密钥 A → 用 A 签名 token
- 验证时：`ParseToken()` 调用 `jwtSecret()` → 拿到密钥 B → 用 B 验证
- A ≠ B → 签名不匹配 → 返回 401

### 修复方案

用包级别的变量缓存密钥，保证每个进程只生成一次：

```go
var cachedSecret []byte

func jwtSecret() []byte {
    if cachedSecret != nil {
        return cachedSecret
    }
    secret := os.Getenv("JWT_SECRET")
    if secret == "" {
        b := make([]byte, 32)
        if _, err := rand.Read(b); err != nil {
            log.Printf("FATAL: cannot generate JWT secret: %v", err)
            cachedSecret = []byte("fallback-unsafe-key-change-me")
            return cachedSecret
        }
        secret = hex.EncodeToString(b)
        log.Printf("WARNING: JWT_SECRET not set, generated random key.")
    }
    cachedSecret = []byte(secret)
    return cachedSecret
}
```

修复方案兼顾安全性（缓存保证签名一致）和灵活性（仍然允许随机密钥兜底）。

### 走通开源贡献流程

| 步骤 | 内容 |
|------|------|
| 1. 本地复现 | 按步骤触发 Bug |
| 2. 阅读源码 | 从前端 → API → JWT 中间件，定位根因 |
| 3. 提 Issue | 描述问题，附带复现步骤 → [#15](https://github.com/LeoninCS/feedsystem_video_go/issues/15) |
| 4. Fork + 修复分支 | `fix/jwt-secret-caching` |
| 5. 提 PR | 关联 Issue → [#16](https://github.com/LeoninCS/feedsystem_video_go/pull/16) |

### 面试话术

- 在一个 279 star 的开源项目中发现了真实 bug
- 从前端（Vue/TypeScript）追踪到 API（Gin/Go）再到 JWT 鉴权中间件，定位了根因
- 提出的修复方案兼顾安全性和灵活性
- 走通了完整的开源贡献流程

---

这一篇从"跑通项目"出发，反推了 Docker Compose 编排、Gin 路由注册、Redis 缓存、RabbitMQ 消息队列、Worker 进程模式等工程概念，最后通过一次真实的开源贡献把整条链路串了起来。相比纯语法学习，反推真实项目能更快建立工程直觉。
