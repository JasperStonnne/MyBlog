---
title: Go 项目反推：Feed 流系统实战——工程化
slug: go-feed-engineering
description: 梳理 Feed 流项目的 Docker Compose、配置管理、pprof、CI 流程与启动脚本
date: 2026-07-29T00:00:00+08:00
draft: false
image: cover.svg
tags:
  - Go
  - Docker
  - CI/CD
  - DevOps
  - pprof
  - 配置管理
categories:
  - 项目记录
---

承接 [Go 项目反推：Feed 流系统实战——SSE 实时推送与分片上传]({{< ref "go-feed-sse-upload" >}})，这一篇回到工程全景，梳理 `feedsystem_video_go` 的容器编排、配置管理、性能分析、持续集成与本地启动流程。

## Docker Compose（容器编排）

### 什么是 Docker Compose

把所有服务打包成容器，一键启动。

没有 Docker Compose：
```
手动启动MySQL → 手动启动Redis → 手动启动RabbitMQ → 手动启动Backend → 手动启动Worker → 手动启动Frontend
每次都要敲6条命令，还要记住启动顺序
```

有 Docker Compose：
```bash
docker compose up -d
一键启动所有服务
```

### 这个项目的 Docker Compose

```yaml
services:
  mysql:      # 数据库
  redis:      # 缓存
  rabbitmq:   # 消息队列
  backend:    # API进程
  worker:     # Worker进程
  frontend:   # 前端
```

相关文件：`~/dev/projects/feedsystem_video_go/docker-compose.yml`

---

## depends_on + healthcheck（保证启动顺序）

### depends_on 是什么

告诉 Docker：某个服务依赖其他服务，要等依赖的服务启动后才能启动自己。

```yaml
backend:
  depends_on:
    mysql:
      condition: service_healthy   # MySQL健康了才启动backend
    redis:
      condition: service_healthy   # Redis健康了才启动backend
    rabbitmq:
      condition: service_healthy   # RabbitMQ健康了才启动backend
```

### healthcheck 是什么

判断服务是否"健康"（真正能接受连接）。

```yaml
mysql:
  healthcheck:
    test: ["CMD-SHELL", "mysqladmin ping -h 127.0.0.1 -uroot -p123456"]
    interval: 5s    # 每5秒检查一次
    timeout: 5s     # 超时5秒
    retries: 20     # 重试20次
```

MySQL 启动后，每 5 秒 ping 一次，连续 ping 成功才算"健康"。

### 为什么要 healthcheck

```
没有healthcheck：
    MySQL容器启动了 → backend立刻启动
    但MySQL还没初始化完 → backend连接失败 → 崩溃

有healthcheck：
    MySQL容器启动了 → 等MySQL真正能接受连接 → backend再启动
```

### 所有服务的 healthcheck

| 服务 | healthcheck 测试 | 作用 |
|------|-----------------|------|
| mysql | `mysqladmin ping` | 确认 MySQL 能接受连接 |
| redis | `redis-cli ping` | 确认 Redis 能接受连接 |
| rabbitmq | `rabbitmq-diagnostics ping` | 确认 RabbitMQ 能接受连接 |
| backend | `wget http://127.0.0.1:8080/healthz` | 确认 API 进程正常 |
| worker | `pgrep worker` | 确认 Worker 进程存在 |
| frontend | `wget http://127.0.0.1:80/` | 确认前端能访问 |

---

## 配置管理（优先级）

### 配置文件位置

```
~/dev/projects/feedsystem_video_go/backend/configs/
├── config.yaml               # 本地开发用（MySQL端口3306）
├── config.compose-local.yaml # Docker Compose本地用（MySQL端口3307）
└── config.docker.yaml        # Docker部署用（容器内部通信）
```

### 三个配置文件的区别

| 文件 | MySQL 端口 | 用途 |
|------|-----------|------|
| config.yaml | 3306 | 本地开发，MySQL 装在本机 |
| config.compose-local.yaml | 3307 | Docker Compose，MySQL 容器映射到 3307 |
| config.docker.yaml | 3306 | Docker 部署，容器内部通信 |

### 配置加载优先级

```
环境变量 > .env文件 > YAML文件
```

| 来源 | 举例 | 优先级 |
|------|------|--------|
| 环境变量 | `export MYSQL_DATABASE=feedsystem` | 最高 |
| .env 文件 | `MYSQL_DATABASE=feedsystem` | 中 |
| YAML 文件 | `database.dbname: feedsystem` | 最低 |

### 为什么要这样设计

```
开发环境：
    用YAML文件，配置写死在代码里，方便开发

Docker部署：
    用环境变量覆盖，不用改配置文件

    docker run -e MYSQL_DATABASE=feedsystem -e JWT_SECRET=*** ...
```

### 环境变量覆盖的好处

| 场景 | 做法 |
|------|------|
| 本地开发 | 用 YAML 文件的默认配置 |
| Docker 部署 | 用环境变量覆盖数据库地址、密码等 |
| 生产环境 | 用环境变量覆盖，敏感信息不写进代码 |

### 启动时指定配置文件

```bash
# 本地开发
cd backend && CONFIG_PATH=configs/config.yaml go run ./cmd

# Docker Compose本地
cd backend && CONFIG_PATH=configs/config.compose-local.yaml go run ./cmd
```

---

## API 进程和 Worker 进程为什么要分开

### 两个进程

```
cmd/
├── main.go          # API进程（处理HTTP请求）
└── worker/
    └── main.go      # Worker进程（消费MQ消息）
```

### 各自做什么

| | API 进程 | Worker 进程 |
|--|---------|------------|
| 职责 | 处理 HTTP 请求 | 消费 MQ 消息 |
| 入口 | `go run ./cmd` | `go run ./cmd/worker` |
| 启动方式 | 接收 HTTP 请求 | 订阅 MQ 队列 |

### 为什么要分开

**1. 职责不同**

```
API进程：用户发请求 → 处理 → 返回响应（要求快）
Worker进程：从MQ取消息 → 更新DB、推送通知（可以慢）
```

**2. 独立扩展**

```
场景：API扛不住了

没分开：只能整体加机器
分开后：只加API机器，Worker不动

场景：Worker扛不住了

分开后：只加Worker机器，API不动
```

**3. 故障隔离**

```
Worker崩了：不影响API，用户还能正常访问
API崩了：不影响Worker，后台任务还在处理
```

**4. 资源分配不同**

```
API进程：需要低延迟，分配更多CPU
Worker进程：可以容忍高延迟，分配更多内存
```

### Docker Compose 里的配置

```yaml
backend:
  build:
    dockerfile: backend/Dockerfile
    target: api        # 构建API镜像

worker:
  build:
    dockerfile: backend/Dockerfile
    target: worker     # 构建Worker镜像
```

同一个 Dockerfile，用不同的 target 构建两个镜像。

---

## pprof（性能分析）

### pprof 是什么

Go 内置的性能分析工具，帮你找出程序哪里慢、哪里吃内存。

### 能分析什么

| 类型 | 作用 |
|------|------|
| CPU | 哪个函数最耗 CPU |
| 内存 | 哪个函数最耗内存 |
| goroutine | 有没有 goroutine 泄漏 |
| 阻塞 | 哪里卡住了 |

### 配置

```yaml
observability:
  pprof:
    enabled: true
    api_addr: localhost:6060      # API进程的pprof端口
    worker_addr: localhost:6061   # Worker进程的pprof端口
```

启动后，访问 `http://localhost:6060/debug/pprof/` 就能看到性能数据。

### 常用命令

```bash
# CPU分析（30秒）
go tool pprof http://localhost:6060/debug/pprof/profile?seconds=30

# 内存分析
go tool pprof http://localhost:6060/debug/pprof/heap

# goroutine分析
go tool pprof http://localhost:6060/debug/pprof/goroutine
```

### 场景举例

```
用户反馈：API响应慢

1. 用pprof抓CPU分析
2. 发现某个函数占了80%的CPU时间
3. 优化这个函数
4. 响应变快了
```

---

## CI 流程（持续集成）

### CI 是什么

代码推送到 GitHub 后，自动运行检查，确保代码质量。

### 这个项目的 CI

```yaml
# .github/workflows/ci.yml
jobs:
  test:
    steps:
      - go vet ./...           # 静态检查
      - go test -race ./...    # 运行测试，检测数据竞争
```

### go vet 是什么

静态分析，找出语法错误和可疑代码，不需要运行程序。

```
举例：
- 变量声明了但没用
- 格式化字符串参数不对
- 死代码（永远不会执行的代码）
```

### go test -race 是什么

运行测试，同时检测数据竞争。

### 什么是数据竞争

多个 goroutine 同时读写同一个变量，没有加锁。

```go
// 数据竞争的例子
var count int

go func() { count++ }()  // goroutine1写
go func() { count++ }()  // goroutine2写

// 两个goroutine同时写count，结果不确定
```

### -race 怎么检测

```
go test -race ./...

如果发现数据竞争：
    WARNING: DATA RACE
    Write by goroutine 1
    Read by goroutine 2
    ...
    FAIL
```

### CI 流程

```
开发者推送代码到GitHub
    ↓
GitHub Actions自动触发
    ↓
go vet ./...           → 静态检查
    ↓
go test -race ./...    → 运行测试 + 检测数据竞争
    ↓
全部通过 → 绿色 ✅
有失败 → 红色 ❌，不允许合并
```

### 好处

| | 没有 CI | 有 CI |
|--|--------|------|
| 代码质量 | 靠人工 review | 自动检查 |
| 数据竞争 | 上线后才发现 | 提交时就发现 |
| 合并代码 | 随便合并 | 检查通过才能合并 |

---

## 启动脚本（start.sh）

### 一键启动所有服务

```bash
./start.sh
```

相关文件：`~/dev/projects/feedsystem_video_go/start.sh`

### 脚本是什么语言写的

Bash（Shell 脚本），文件扩展名 `.sh` = Shell 脚本。

```bash
#!/usr/bin/env bash    # 声明用bash解释器
set -euo pipefail      # 遇到错误立刻停止
```

Bash 是 Linux/macOS 终端的脚本语言，用来自动化命令行操作。

### 脚本做的事

```
1. 启动Redis（如果没有运行）
2. 启动RabbitMQ（通过Docker Compose）
3. 启动Backend（go run ./cmd）
4. 启动Worker（go run ./cmd/worker）
5. 启动Frontend（npm run dev）
6. Ctrl+C停止所有服务
```

### 核心逻辑

```bash
# 启动Redis
start_redis() {
    redis-server --bind 127.0.0.1 --port 6379
}

# 启动RabbitMQ（通过Docker Compose）
start_rabbitmq_compose() {
    docker compose up -d rabbitmq
}

# 启动Backend
start_backend_bg() {
    cd backend && go run ./cmd &
}

# 启动Worker
start_worker_bg() {
    cd backend && go run ./cmd/worker &
}

# 启动Frontend
start_frontend_bg() {
    cd frontend && npm run dev &
}

# Ctrl+C停止所有服务
trap cleanup INT TERM EXIT
cleanup() {
    kill $BACKEND_PID
    kill $WORKER_PID
    kill $FRONTEND_PID
    docker compose stop
}
```

### 环境变量控制

```bash
START_REDIS=1        # 是否启动Redis
START_RABBITMQ=1     # 是否启动RabbitMQ
START_BACKEND=1      # 是否启动Backend
START_WORKER=1       # 是否启动Worker
START_FRONTEND=1     # 是否启动Frontend
```

可以按需启动：

```bash
# 只启动Backend和Worker
START_FRONTEND=0 ./start.sh
```

---

## 面试要点总结

### Q: Docker Compose 怎么保证启动顺序？

A: depends_on + healthcheck，服务健康了才启动下一个。

### Q: 配置优先级？

A: 环境变量 > .env 文件 > YAML 文件。

### Q: API 和 Worker 为什么要分开？

A: 职责不同、独立扩展、故障隔离、资源分配不同。

### Q: pprof 能分析什么？

A: CPU、内存、goroutine、阻塞。

### Q: CI 做了什么？

A: go vet 静态检查 + go test -race 检测数据竞争。

### Q: 什么是数据竞争？

A: 多个 goroutine 同时读写同一个变量，没有加锁。用 go test -race 检测。

### Q: 为什么要用 healthcheck？

A: 确保服务真正能接受连接，而不是只启动了容器。
