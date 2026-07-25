---
title: Go 项目反推：Feed 流系统实战——认证体系
slug: go-feed-authentication
description: 从 JWT、Access Token、Refresh Token 与 Token 撤销出发，梳理 Feed 流系统的完整认证链路和软硬鉴权中间件
date: 2026-07-25T00:00:00+08:00
draft: false
image: cover.svg
tags:
  - Go
  - Gin
  - JWT
  - Redis
  - 认证
  - Token
categories:
  - 项目记录
---

承接 [Go 项目反推：Feed 流系统实战——请求生命周期]({{< ref "go-feed-request-lifecycle" >}})，这一篇继续反推 `feedsystem_video_go` 的认证代码，梳理 JWT、Refresh Token、Token 撤销，以及软硬鉴权中间件之间的关系。

## 一、jwt.go 文件结构

文件位置：`internal/auth/jwt.go`

```text
jwtSecret()              → 获取密钥
Claims                   → Token 里存什么
GenerateToken()          → 生成 accessToken（JWT 格式，15 分钟）
GenerateRefreshToken()   → 生成 refreshToken（随机字符串）
ParseToken()             → 验证 Token（签名 + 过期）
```

---

## 二、密钥与 Claims

### jwtSecret()：获取密钥

优先使用环境变量 `JWT_SECRET`，没有就随机生成：

```bash
JWT_SECRET=*** go run ./cmd
```

如果没有设置环境变量，每次服务器启动时密钥都会改变，之前签发的 Token 将全部失效，所有用户都要重新登录。因此正式环境**必须**设置稳定且足够随机的 `JWT_SECRET`。

### Claims：Token 里存什么

```go
type Claims struct {
    AccountID uint              // 用户 ID
    Username  string            // 用户名
    jwt.RegisteredClaims        // 标准字段（过期时间、签发时间等）
}
```

可以把 Claims 类比为一张有有效期的身份证，上面记录用户名、用户 ID、签发时间和过期时间。

需要注意：JWT Payload 只是经过 Base64URL 编码，并不是加密，客户端也能读取其中的内容。因此 Claims 里不能放密码、密钥等敏感数据。

---

## 三、生成 Token

### GenerateToken()：生成 accessToken

```text
1. 创建 Claims，写入用户 ID、用户名和 15 分钟后的过期时间
2. 使用 HS256 算法创建 Token 对象
3. 使用 JWT Secret 签名，生成 Token 字符串
```

结果是一串类似 `eyJhbG...xxxx` 的 JWT。它内部包含经过编码的 Header 和 Payload，以及用于防止内容被篡改的签名。

HS256 只负责签名和验签，不负责加密。其他人虽然可以读取 Payload，但没有 `JWT_SECRET` 就无法生成有效签名。

### GenerateRefreshToken()：生成 refreshToken

Refresh Token 和 Access Token 完全不同：

```text
accessToken  → JWT 格式，携带 Claims，并使用密钥签名
refreshToken → 随机字符串，本身不携带用户信息
```

项目使用 32 个随机字节，再转换为 64 位十六进制字符串。Refresh Token 不需要携带 Claims，它只需要成为一个难以猜中的长期凭证。

### 双 Token 时间设定

```text
accessToken  15 分钟：泄露后的可利用时间较短
refreshToken 7 天：   用于换取新的 Access Token，减少重复登录
```

Access Token 有效期短，降低泄露风险；Refresh Token 有效期长，保证使用体验。二者配合，在安全性和便利性之间取得平衡。

---

## 四、验证 Token

### ParseToken()：验证 Token

```text
输入：客户端发送的 Token 字符串
输出：Claims（用户 ID、用户名）或错误
```

流程：

```text
1. 解析 Token
2. 确认签名算法是 HS256
3. 使用 JWT Secret 验证签名
4. 检查 Token 是否过期
5. 全部通过 → 返回 Claims
6. 任一失败 → 返回错误
```

这里不是“解密 Token”，因为 JWT Payload 没有被加密。服务器做的是解析内容并验证签名。

还需要注意：`ParseToken()` 只检查格式、签名和过期时间，**不检查 Token 是否已被撤销**。

---

## 五、Refresh 流程

Access Token 过期后，可以使用 Refresh Token 换取新的 Access Token：

```text
客户端：Access Token 过期，这是 Refresh Token，请签发新的 Access Token

服务器：
  1. 使用 Refresh Token 查询 Redis 中的 refresh:<token>
  2. 查询成功后得到 accountID
  3. 查询数据库，确认 Refresh Token 与账户记录匹配
  4. 生成新的 Access Token
  5. 将新 Token 写入数据库和 Redis
  6. 返回给客户端
```

可以类比为：短期门禁卡过期后，使用长期凭证到前台换一张新卡。

关键点是：**使用 Refresh Token 换取 Access Token，而不是反过来。**

---

## 六、Token 撤销

JWT 的签名和过期时间一旦确定，单靠 `ParseToken()` 无法让一个尚未过期的 Token 提前失效。因此项目额外在 Redis 和 MySQL 中保存当前有效 Token。

登出时删除旧 Token：

```text
删除 Redis 中的三个 Key：
  account:<id>          → accessToken
  account:<id>:refresh  → refreshToken
  refresh:<token>       → accountID

清空 MySQL 中的字段：
  token
  refresh_token
```

删除后，即使旧 JWT 的签名正确且尚未过期，中间件也无法在 Redis 或 MySQL 中匹配到它，因此会拒绝请求。

修改密码时也可以使用相同思路：

```text
清空旧 Token
→ 签发新 Token
→ 让其他设备上的旧登录状态立即失效
```

---

## 七、JWTAuth 中间件

文件位置：`internal/middleware/jwt/jwt.go`

处理流程：

```text
1. 从请求头读取 Authorization
2. 检查格式是否为 Bearer <token>
3. 调用 auth.ParseToken() 验证格式、签名和过期时间
4. 调用 check() 检查 Token 是否仍是当前有效 Token
5. 验证通过后将用户信息写入 Gin Context
```

客户端发送：

```http
Authorization: Bearer ***
```

### check()：为什么同时查询 Redis 和 MySQL

```text
第一步：查询 Redis
  查到 → 对比请求 Token 是否为当前有效 Token
    相同 → 放行
    不同 → 拒绝，Token 已被替换或撤销

  Redis 未命中或不可用 → 查询 MySQL

第二步：查询 MySQL
  查到账户 → 对比数据库保存的 Token
    相同 → 放行，并回填 Redis
    不同 → 拒绝
```

Redis 是缓存，速度快但可能未命中或不可用；MySQL 是持久化的数据来源，可以作为兜底。MySQL 本身也可能发生故障，因此更准确的说法是“Redis 失败时降级查询 MySQL”，而不是“MySQL 永远可用”。

`check()` 主要完成两件事：

1. 将请求中的 Token 与服务器保存的当前 Token 对比，判断是否已被撤销
2. 将用户信息写入 `gin.Context`，方便后续 Handler 使用

---

## 八、两种鉴权

```text
JWTAuth（硬鉴权）：
  没有 Token 或 Token 无效 → 拒绝请求

SoftJWTAuth（软鉴权）：
  没有 Token → 继续放行
  有 Token且有效 → 识别用户身份后放行
```

项目中的使用示例：

```text
JWTAuth（必须登录）：
  /logout
  /rename
  /uploadAvatar

SoftJWTAuth（不强制登录）：
  /feed/listLatest
  /feed/listByPopularity
```

用户没有 Token 也能浏览 Feed；携带有效 Token 时，系统可以识别用户并提供个性化结果。

可以类比为：

```text
JWTAuth     → 会员俱乐部，没有会员卡不能进入
SoftJWTAuth → 商场，没有会员卡也能进入，有会员卡则能获得个性化服务
```

---

## 九、面试要点

1. **JWT 如何生成**：Claims + HS256 + JWT Secret 签名
2. **JWT 不等于加密**：Payload 可以读取，签名用于防篡改
3. **双 Token**：短期 Access Token + 长期 Refresh Token
4. **Refresh 流程**：使用 Refresh Token 换取新的 Access Token
5. **Token 撤销**：清理 Redis 与 MySQL 中保存的 Token，使旧 JWT 立即失效
6. **中间件 check()**：优先查询 Redis，未命中或不可用时查询 MySQL
7. **两种鉴权**：JWTAuth 强制登录，SoftJWTAuth 允许匿名访问

这一套设计实际上让 JWT 从“完全无状态”变成了“带服务端状态的认证方案”。它牺牲了一部分无状态带来的简单扩展能力，换取了主动登出、修改密码后强制失效和单会话控制等能力。
