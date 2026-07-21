---
title: Go 项目反推：Feed 流系统实战——请求生命周期
slug: go-feed-request-lifecycle
description: 以 POST /account/login 为例，追踪一个请求从 Gin 路由、中间件、Handler、Service、Repo 到 MySQL 和 Redis 的完整调用链
date: 2026-07-21T00:00:00+08:00
draft: false
image: cover.svg
tags:
  - Go
  - Gin
  - Redis
  - GORM
  - JWT
  - 请求生命周期
categories:
  - 项目记录
---

承接 [Go 项目反推：Feed 流系统实战——从 Docker 到开源贡献]({{< ref "go-feed-system-reverse-engineering" >}})，这一篇继续反推 `feedsystem_video_go` 的后端代码。这次不再从整体架构看项目，而是以 `POST /account/login` 为例，追踪一个请求从进来到返回的完整路径。

## 路由基础

### router.go 是什么

router.go = 路由表 = “地址簿”。它告诉服务器：什么方式 + 访问哪个路径 → 转发到哪个函数。

```text
POST /account/login      → accountHandler.Login
POST /account/register   → accountHandler.CreateAccount
GET  /healthz            → 返回 {"status":"ok"}
```

类比：router.go 是前台接待（“办 login 业务找张三”），handler.go 是张三的工位（“我来处理”）。

### 路由组

```go
accountGroup := r.Group("/account")
```

创建一个路由组，前缀是 `/account`。组里的所有路由，路径前面自动加上前缀：

```text
accountGroup.POST("/login", ...)    → 实际路径 POST /account/login
accountGroup.POST("/register", ...) → 实际路径 POST /account/register
```

类比：把用户相关接口都放在 `/account` 这个“文件夹”下。

### 路由注册

```go
accountGroup.POST("/login", loginLimiter, accountHandler.Login)
```

三个参数：

```text
"/login"                → 路径（拼上 /account，变成 /account/login）
loginLimiter            → 限流中间件（先过这一关）
accountHandler.Login    → 最终处理请求的函数（过了限流才到这里）
```

请求流程：

```text
POST /account/login 进来
    ↓
loginLimiter（限流中间件）
    ↓
accountHandler.Login（handler 处理）
```

## 函数引用 vs 函数调用

```go
accountGroup.POST("/login", loginLimiter, accountHandler.Login)
```

`accountHandler.Login` 是函数本身，不是调用。Go 里函数可以当变量传来传去，和 CS61A 的高阶函数一个意思：

```text
accountHandler.Login      → 函数本身（把函数“交给”路由器，等有人请求时再调用）
accountHandler.Login(c)   → 调用函数（立刻执行）
```

类比：给你一张纸条写上张三的电话号码（函数引用） vs 现在立刻拨打电话（函数调用）。

现在不执行，等有人发 `POST /account/login` 时，路由器才会调用 `Login(c)` 并传入请求信息。

## 三层架构

### 结构创建

router.go 第 40-42 行：

```go
accountRepository := account.NewAccountRepository(db)
accountService := account.NewAccountService(accountRepository, cache)
accountHandler := account.NewAccountHandler(accountService)
```

创建三个“工人”：

```text
Repository  → 操作数据库的人（查表、写表）
Service     → 处理业务逻辑的人（校验密码、生成 Token）
Handler     → 接收请求的人（解析参数、返回结果）
```

依赖关系：`db → Repository → Service → Handler → 路由`

类比：先招档案员（Repository）给数据库钥匙，再招业务员（Service）管档案员，再招前台（Handler）对接业务员，最后门口挂牌子（路由）：“办 login 业务找前台”。

### Handler 结构体

```go
type AccountHandler struct {
    accountService *AccountService
}

func NewAccountHandler(accountService *AccountService) *AccountHandler {
    return &AccountHandler{accountService: accountService}
}
```

`AccountHandler` 是一个结构体，里面有一个字段 `accountService`。`NewAccountHandler` 创建这个结构体并把 Service 存进去。

### 分层的意义

```text
handler 只管：解析请求 → 调用 service → 返回结果
service 处理：校验密码 → 生成 Token → 写数据库
repo 处理：   操作数据库
```

handler 不关心密码怎么校验、Token 怎么生成，它只管“调用 service”。

### 文件目录结构

```text
internal/http/router.go        → 路由目录，负责“找谁”
internal/account/handler.go    → 用户模块，负责“怎么干”
internal/account/service.go    → 用户模块，负责“核心逻辑”
internal/account/repo.go       → 用户模块，负责“操作数据库”
```

按模块分目录：account 目录下放所有跟用户相关的代码，video 目录下放所有跟视频相关的代码。

## Handler 层：Login 函数

handler.go 第 116-133 行：

```go
func (h *AccountHandler) Login(c *gin.Context) {
    var req LoginRequest                                    // 声明容器
    if err := c.ShouldBindJSON(&req); err != nil { ... }   // 解析 JSON
    account, err := h.accountService.FindByUsername(...)    // 查数据库存不存在
    accessToken, refreshToken, err := h.accountService.Login(...)  // 校验密码+生成Token
    c.JSON(200, LoginResponse{Token, RefreshToken, ...})    // 返回给客户端
}
```

完整流程：

```text
1. 解析客户端发的 JSON（用户名、密码）
2. 去数据库查这个用户存不存在
3. 存在的话，让 service 校验密码、生成 Token
4. 把 Token 返回给客户端
```

### ShouldBindJSON

```go
c.ShouldBindJSON(&req)
```

把客户端发的 JSON 解析到 `req` 变量里。客户端发 `{"username": "zhangsan", "password": "***"}`，解析后 `req.Username = "zhangsan"`，`req.Password = "123456"`。

**ShouldBindJSON vs json.NewDecoder**：两个干的事一样，都是把 JSON 解析到变量里。`c.ShouldBindJSON(&req)` 是 Gin 封装的一行搞定；`json.NewDecoder(r.Body).Decode(&req)` 是原生写法，要三行还要检查 err。类比：自动挡 vs 手动挡。

## Service 层：Login 函数

handler 调用 service 的 Login：

```go
accessToken, refreshToken, err := h.accountService.Login(c.Request.Context(), req.Username, req.Password)
```

service.go 的 Login 做了 6 件事：

```text
1. 查数据库拿用户信息（FindByUsername）
2. bcrypt 校验密码
3. 生成 accessToken 和 refreshToken
4. Token 存进 MySQL（调用 repo.Login）
5. Token 存进 Redis（3 个 key）
6. 返回两个 Token
```

### ctx（context）

`ctx` = 请求的“身份证”，带着两个信息：

- 超时时间：这个请求最多等多久
- 取消信号：客户端断开了，别再查了

每一层都要传 ctx，这样整条链路都能感知到“客户端还在不在”。

类比：你去饭店点菜，等了 5 分钟走了。没有 ctx → 厨师还在炒，炒完发现你走了，菜倒掉。有 ctx → 服务员告诉厨师“客人走了”，厨师立刻停手。

### bcrypt 校验密码

```go
bcrypt.CompareHashAndPassword([]byte(account.Password), []byte(password))
```

传入两个参数：数据库里存的加密密码（哈希）和用户这次输入的明文密码。bcrypt 把明文密码加密，和数据库里的对比，一样返回 nil（成功），不一样返回 error（失败）。

为什么不存明文？数据库被黑了，黑客拿到的是加密字符串，反推不出原始密码。

`[]byte()` = 把字符串转成字节数组，bcrypt 只接收这种格式。

### 双 Token（JWT）

登录成功返回两个 Token：

```text
accessToken   → 短期有效（15分钟），用来证明“我是谁”
refreshToken  → 长期有效（7天），用来续期 accessToken
```

为什么两个？只用一个 token 的问题：设太短 → 用户每 15 分钟重新登录，体验差；设太长 → 被偷了就一直能用，不安全。双 token 解决了这个问题：accessToken 短，被偷了也很快过期；refreshToken 长，用户不用频繁登录；accessToken 过期了，用 refreshToken 换新的。

类比：accessToken = 门禁卡（每天过期，重新刷卡续期），refreshToken = 身份证（长期有效，门禁卡丢了拿身份证补办）。

Token 根据用户信息生成（不是纯随机）：

```text
accessToken  → 包含用户ID + 用户名 + 过期时间（15分钟）
refreshToken → 只包含用户ID + 过期时间（7天）
用密钥加密生成一串乱码字符串
```

### 为什么有 ID 又有用户名

```text
ID（account.ID）  → 数据库主键，不可变，用来精确查找
用户名（username）→ 可以改，用来方便识别
```

类比：身份证号（ID）不可变，查账户用；姓名（username）可以改，写在凭证上让人看。生成 Token 时两个都存：ID 用来查人，用户名用来识别。

## Repo 层：Login 函数

service 调用 repo 把 Token 存进 MySQL：

```go
as.accountRepository.Login(ctx, account.ID, accessToken, refreshToken)
```

repo.go 里用 GORM 操作数据库：

```go
ar.db.WithContext(ctx).Model(&Account{}).Where("id = ?", id).Updates(
    map[string]interface{}{"token": token, "refresh_token": refreshToken})
```

翻译：在 account 表里，找到 id 等于 id 的那一行，更新 token 和 refresh_token。

等价 SQL：

```sql
UPDATE account SET token = 'xxx', refresh_token = 'yyy' WHERE id = 1;
```

GORM = 用 Go 代码描述“我要干什么”，GORM 帮你翻译成 SQL。

## Redis 缓存

登录成功后，Token 存两份：

```text
MySQL 存一份 → 保险，重启不丢，但慢
Redis 存一份 → 快，验证时直接查这里
```

Redis 存了 3 个 key：

```text
key: "account:1"           → accessToken    TTL 24小时
key: "account:1:refresh"   → refreshToken   TTL 7天
key: "refresh:xxxxx"       → accountID      TTL 7天
```

为什么存 3 个？第 1 个验证 Token 时查，第 2 个刷新 Token 时查，第 3 个用 refreshToken 反查用户 ID。

Redis 不可用时降级：跳过缓存，只靠 MySQL，功能不丢，只是慢一点。

## 中间件

### 怎么执行的

中间件 = 挡在 handler 前面的一道关卡。两个关键函数：

```text
c.Next()   → 放行，让请求继续走到下一个中间件或 handler
c.Abort()  → 拦截，不让请求继续走，handler 不会执行
```

类比：`c.Next()` = 安检通过，放你进去；`c.Abort()` = 安检不过，拦住你。

### 为什么是两层函数

```go
func Limit(cache, name string, ...) gin.HandlerFunc {   // 外层：传配置
    return func(c *gin.Context) {                        // 内层：处理请求
        c.Next()
    }
}
```

外层传配置，内层处理请求。外层函数执行一次，创建中间件；内层函数每次有人访问都执行一次。

类比：外层 = 装修奶茶店（执行一次，买机器、买原料）；内层 = 每天营业（执行无数次，客人来了做奶茶）。

`gin.HandlerFunc` = Gin 要求中间件必须是这种格式的函数。

## gin.Context 的数据传递

中间件把用户 ID 塞进 `c` 里：

```go
c.Set("accountID", accountID)   // 中间件：往 c 里存
```

handler 从 `c` 里取出来：

```go
value, exists := c.Get("accountID")   // handler：从 c 里取
```

`c` 就是一个“传话筒”，中间件往里塞东西，handler 从里拿东西。`c` 是临时对象，只活在一次请求里，请求结束就没了，不会存到 Redis。

类比：安检员检查完证件，在你手上盖个章“1号”（中间件 `c.Set`）；柜员看你手上的章，知道你是“1号”（handler `c.Get`）。

## 错误到 HTTP 状态码的映射

`ClassifyHTTPStatus` 就是一张对照表：

```text
error 类型                      → 状态码 → 含义
nil（没错误）                   → 200   → 成功
ErrUnauthorized（未登录）       → 401   → 未授权
ErrValidation（参数错误）       → 400   → 请求格式错
gorm.ErrRecordNotFound（查不到）→ 404   → 资源不存在
其他错误                        → 500   → 服务器内部错误
```

handler 里用法：

```go
c.JSON(apierror.ClassifyHTTPStatus(err), gin.H{"error": err.Error()})
```

翻译：看一眼 error 是什么类型，返回对应的状态码。

## Login 请求完整调用链

```text
1. 用户发 POST /account/login

2. router.go 找到这条路由，先过 loginLimiter（限流中间件）

3. 进入 handler.go 的 Login 函数：
   - 解析 JSON（拿到 username 和 password）
   - 调用 service.FindByUsername（查用户存不存在）
   - 调用 service.Login（校验密码 + 生成 Token）

4. 进入 service.go 的 Login 函数：
   - 再查一次数据库（拿到密码哈希）
   - bcrypt 校验密码
   - 生成 accessToken 和 refreshToken
   - 调用 repo.Login（把 Token 存 MySQL）
   - 把 Token 存进 Redis（3 个 key）

5. 进入 repo.go 的 Login 函数：
   - 用 GORM 把 Token 写进 account 表

6. Token 一层一层返回：repo → service → handler → 客户端
```

涉及 4 个文件：

```text
router.go   → 注册路由，把路径和函数连起来
handler.go  → 解析请求，调用 service
service.go  → 校验密码，生成 Token，存数据库和缓存
repo.go     → 操作 MySQL
```

涉及 2 个存储：

```text
MySQL  → 存用户信息和 Token（持久化，重启不丢）
Redis  → 缓存 Token（快，重启会丢）
```
