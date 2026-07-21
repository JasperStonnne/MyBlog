---
title: Go Web 开发：从 net-http 到 Gin
slug: go-web-net-http-to-gin
description: Go Web 开发全流程：从标准库 net/http 的 Handler、路由、JSON 处理，到 REST API 与 CRUD 实战，再到 Gin 的参数校验、中间件与路由分组
date: 2026-07-18T00:00:00+08:00
draft: false
image: cover.svg
tags:
  - Go
  - Web 开发
  - net/http
  - Gin
  - REST API
categories:
  - 技术笔记
---

承接 [Go 语言基础：从变量到指针]({{< ref "go-basics-variables-to-pointers" >}})、[Go 语言进阶：方法、接口与错误处理]({{< ref "go-methods-interfaces-errors" >}})、[Go 语言工程化：模块管理与项目结构]({{< ref "go-engineering-modules-project-structure" >}}) 和 [Go 语言并发：从 Goroutine 到 Worker Pool]({{< ref "go-concurrency-goroutines-worker-pool" >}})，这一篇进入 Web 开发实战：用 Go 写 HTTP 服务器。

## Handler：处理请求的函数

Go 的 HTTP 服务器一切从 Handler 开始。Handler 就是一个处理请求的函数，签名是固定的：

```go
func handler(w http.ResponseWriter, r *http.Request)
```

- `w` = 响应写入器（往里面写什么，客户端就收到什么）
- `r` = 请求信息（客户端发来的 URL、方法、头部等）

把 `w` 想象成"回复窗口"，`r` 想象成"来信内容"。

```go
package main

import (
    "fmt"
    "net/http"
)

func homeHandler(w http.ResponseWriter, r *http.Request) {
    fmt.Fprintf(w, "Hello, Go Web!\n")
}

func aboutHandler(w http.ResponseWriter, r *http.Request) {
    fmt.Fprintf(w, "About Page\n")
}
```

## 路由注册与启动服务器

告诉 Go：访问 `/` 调用 `homeHandler`，访问 `/about` 调用 `aboutHandler`。

```go
func main() {
    // 注册路由
    http.HandleFunc("/", homeHandler)
    http.HandleFunc("/about", aboutHandler)

    // 启动服务器，监听 8080 端口
    fmt.Println("Server starting on http://localhost:8080 ...")
    err := http.ListenAndServe(":8080", nil)
    if err != nil {
        fmt.Println("Server error:", err)
    }
}
```

### `ListenAndServe` 的含义

- **Listen** = 监听（竖起耳朵听有没有人敲门）
- **And** = 并且
- **Serve** = 提供服务（有人来了就接待）

这行代码一旦执行，程序就卡在这不走了，一直等着。下面的 `if err` 只有服务器出错才会执行。

> **端口被占了怎么杀**：`lsof -ti:8080 | xargs kill -9`

## HTTP 方法与 curl

`r.Method` → 请求方法，`r.URL.Path` → 请求路径。

curl 是在终端发 HTTP 请求的工具，相当于用命令代替浏览器：

```bash
curl http://localhost:8080/              # GET
curl -X POST http://localhost:8080/      # POST
curl -X PUT http://localhost:8080/       # PUT
curl -X DELETE http://localhost:8080/    # DELETE
```

HTTP 方法表示"我想干什么"：

| 方法 | 含义 | 类比（快递柜） |
|------|------|----------------|
| GET | 获取数据 | 查看 123 号格子 |
| POST | 提交数据 | 往里放新包裹 |
| PUT | 更新数据 | 换掉 123 号格子的东西 |
| DELETE | 删除数据 | 拿走 123 号格子的东西 |

## 返回 JSON 响应

JSON 是程序之间传递数据的标准格式。核心三步：

```go
import "encoding/json"

type User struct {
    Name string `json:"name"`
    Age  int    `json:"age"`
}

func userHandler(w http.ResponseWriter, r *http.Request) {
    user := User{Name: "Tom", Age: 20}

    // 第1步：把数据变成 JSON
    jsonBytes, err := json.Marshal(user)
    if err != nil {
        http.Error(w, "序列化失败", 500)
        return
    }

    // 第2步：告诉客户端"我返回的是 JSON"
    w.Header().Set("Content-Type", "application/json")

    // 第3步：把 JSON 写出去
    w.Write(jsonBytes)
}
```

### struct 的 json tag

```go
type User struct {
    Name string `json:"name"` // JSON 里字段叫 name（小写）
    Age  int    `json:"age"`
}
```

不写 tag 的话字段名就是 Go 里的大写 `Name`、`Age`。反引号里整个 `json:"xxx"` 是一个整体，冒号后面不能有空格。

### 响应头是什么

HTTP 响应 = 响应头（信封）+ 响应体（信的内容）：

- `w.Header()` = 拿信封
- `w.Header().Set("Content-Type", "application/json")` = 在信封上写"里面是 JSON"
- `w.Write()` = 往信里写内容

## 读取客户端 JSON

客户端 POST 发来 JSON，服务器解析到结构体：

```go
func createUserHandler(w http.ResponseWriter, r *http.Request) {
    // 检查方法
    if r.Method != "POST" {
        http.Error(w, "只支持 POST 方法", http.StatusMethodNotAllowed)
        return
    }

    // 从请求体解析 JSON 到 user
    var user User
    err := json.NewDecoder(r.Body).Decode(&user)
    if err != nil {
        http.Error(w, "JSON 解析失败", 400)
        return
    }

    // 返回给客户端
    jsonData, _ := json.Marshal(user)
    w.Header().Set("Content-Type", "application/json")
    w.Write(jsonData)
}
```

测试：

```bash
curl -X POST http://localhost:8080/user -d '{"name":"Jerry","age":25}'
```

### 序列化 vs 反序列化

| 操作 | 函数 | 用途 |
|------|------|------|
| 结构体 → JSON | `json.Marshal(data)` | 返回给客户端 |
| JSON → 结构体 | `json.NewDecoder(r.Body).Decode(&data)` | 读取客户端数据 |

关键点：

- `Decode` 的参数必须是 `&user`（指针），不是 `user`
- Marshal 和 Decode 都返回 `err`，必须检查
- 出错要 `http.Error` + `return`，不能光打印

## REST API 设计风格

REST 的核心思想：用 HTTP 方法 + 路径，表达"对资源做什么操作"。路径放名词（资源），HTTP 方法放动词（操作）。

本质就是 CRUD：

| 操作 | HTTP 方法 | 路径 | 含义 |
|------|-----------|------|------|
| Create | POST | `/user` | 新建 |
| Read | GET | `/user` 或 `/user/0` | 查列表 / 查单个 |
| Update | PUT | `/user/0` | 改单个 |
| Delete | DELETE | `/user/0` | 删单个 |

记忆：**GET=看，POST=增，PUT=改，DELETE=删**

设计规则：

- URL 用名词（`/user`），不用动词（`/getUser`）
- 用 HTTP 方法区分操作，而不是路径里写动词
- 同一个 URL，不同方法 = 不同操作

## 完整 CRUD 实战

### 四个 Handler 的套路

不管你写哪个 handler，步骤都是：

1. 检查 HTTP 方法对不对
2. 从 URL 拿 id（列表操作跳过）
3. 检查越界（同上）
4. 业务逻辑（创建/返回/替换/删除）
5. 序列化返回（JSON → 设 Header → 写响应）

区别只在第 4 步：

- **Create**：从请求体解析 → append 到 users
- **Read**：从 users 里取 → 序列化返回
- **Update**：从请求体解析 → 替换 `users[id]`
- **Delete**：从 users 删掉 → 返回文字消息

### 结构体与全局变量

```go
type User struct {
    Name string `json:"name"`
    Age  int    `json:"age"`
}

// 模拟数据库，服务器重启就清空
var users = []User{}
```

### 从 URL 提取 id

客户端访问 `GET /user/0`，服务器用 `TrimPrefix` 切掉前缀，再转数字：

```go
func getUserHandler(w http.ResponseWriter, r *http.Request) {
    if r.Method != "GET" {
        http.Error(w, "只支持 GET 方法", 405)
        return
    }

    // /user/0 → "0"
    idStr := strings.TrimPrefix(r.URL.Path, "/user/")

    // "0" → 0
    id, err := strconv.Atoi(idStr)
    if err != nil {
        http.Error(w, "无效的 id", 400)
        return
    }

    // 越界检查
    if id < 0 || id >= len(users) {
        http.Error(w, "越界", 404)
        return
    }

    user := users[id]
    jsonData, _ := json.Marshal(user)
    w.Header().Set("Content-Type", "application/json")
    w.Write(jsonData)
}
```

> 越界用 `>=` 不是 `>`：长度 2 的列表，有效下标是 0、1，id=2 要拦截。

### Update Handler（PUT）

套路：检查方法 → 拿 id → 越界检查 → 解析请求体新数据 → 替换 `users[id]` → 序列化返回。

```go
func updateUserHandler(w http.ResponseWriter, r *http.Request) {
    if r.Method != "PUT" {
        http.Error(w, "只支持 PUT 方法", 405)
        return
    }

    idStr := strings.TrimPrefix(r.URL.Path, "/user/")
    id, err := strconv.Atoi(idStr)
    if err != nil {
        http.Error(w, "无效的 id", 400)
        return
    }
    if id < 0 || id >= len(users) {
        http.Error(w, "越界", 404)
        return
    }

    var user User
    err = json.NewDecoder(r.Body).Decode(&user)
    if err != nil {
        http.Error(w, "解析失败", 400)
        return
    }

    users[id] = user

    jsonData, _ := json.Marshal(user)
    w.Header().Set("Content-Type", "application/json")
    w.Write(jsonData)
}
```

### Delete Handler（DELETE）

套路：检查方法 → 拿 id → 越界检查 → 从 slice 删元素 → 返回消息。

```go
func deleteUserHandler(w http.ResponseWriter, r *http.Request) {
    if r.Method != "DELETE" {
        http.Error(w, "方法错误", 405)
        return
    }

    idStr := strings.TrimPrefix(r.URL.Path, "/user/")
    id, err := strconv.Atoi(idStr)
    if err != nil {
        http.Error(w, "无效的 id", 400)
        return
    }
    if id < 0 || id >= len(users) {
        http.Error(w, "越界", 404)
        return
    }

    // 删 slice 元素
    users = append(users[:id], users[id+1:]...)
    fmt.Fprintf(w, "用户 %d 已删除\n", id)
}
```

### Slice 删除元素的原理

```go
users = append(users[:id], users[id+1:]...)
```

把 id 前面的人和 id 后面的人拼到一起，跳过 id 本身。假设 `users = [张三, 李四, 王五]`，删 id=1（李四）：

- `users[:1]` → `[张三]`
- `users[2:]` → `[王五]`
- append 拼起来 → `[张三, 王五]`

> 注意：删除后 users 长度变了，后面的 id 会前移。

### 路由分发（switch 重构）

当一个 URL 要根据 HTTP 方法分发到不同 handler 时：

```go
func main() {
    http.HandleFunc("/user/", func(w http.ResponseWriter, r *http.Request) {
        path := r.URL.Path

        if path == "/user" || path == "/user/" {
            // 列表和创建
            switch r.Method {
            case "GET":
                listUsersHandler(w, r)
            case "POST":
                createUserHandler(w, r)
            default:
                http.Error(w, "方法不支持", 405)
            }
        } else if strings.HasPrefix(path, "/user/") {
            // 单个用户操作
            switch r.Method {
            case "GET":
                getUserHandler(w, r)
            case "PUT":
                updateUserHandler(w, r)
            case "DELETE":
                deleteUserHandler(w, r)
            default:
                http.Error(w, "方法不支持", 405)
            }
        }
    })

    fmt.Println("Server starting on http://localhost:8080 ...")
    err := http.ListenAndServe(":8080", nil)
    if err != nil {
        fmt.Println("Server error:", err)
    }
}
```

### 测试

```bash
# 查列表（空的）
$ curl http://localhost:8080/user/
[]

# 创建两个用户
$ curl -X POST http://localhost:8080/user/ -d '{"name":"Jerry","age":25}'
{"name":"Jerry","age":25}

$ curl -X POST http://localhost:8080/user/ -d '{"name":"Tom","age":30}'
{"name":"Tom","age":30}

# 再查列表
$ curl http://localhost:8080/user/
[{"name":"Jerry","age":25},{"name":"Tom","age":30}]

# 查单个
$ curl http://localhost:8080/user/0
{"name":"Jerry","age":25}
```

### ServeMux 的斜杠重定向坑

注册 `/user/` 这种子树路由后，Go 默认路由器会把 `/user` 自动重定向到 `/user/`（301 Moved Permanently）。一些客户端跟随 301 重定向时会把 POST 改成 GET，导致请求体丢失；另一些客户端不会自动跟随重定向。

解决：访问列表/创建接口时直接使用带斜杠的 `/user/`，或者同时显式注册 `/user` 与 `/user/` 并自行处理。

## Middleware 中间件

### 一句话理解

在请求到达 handler 之前（或之后）执行的通用逻辑。

比喻：快递站的安检机——每个包裹都要过一遍，但安检机不是最终处理包裹的地方，安检完再交给真正处理的人。

实际用途：

- 日志记录：每个请求都打印一下
- 认证检查：检查有没有登录
- 请求计时：每个请求花了多少时间

### 核心代码

```go
func loggingMiddleware(next http.HandlerFunc) http.HandlerFunc {
    return func(w http.ResponseWriter, r *http.Request) {
        // 前置：请求进来时
        log.Println("收到请求:", r.URL.Path)

        // 调用真正的 handler
        next(w, r)

        // 后置：响应返回后
        log.Println("处理完成")
    }
}
```

### 使用方式

```go
// 原来
http.HandleFunc("/user", createUserHandler)

// 包一层中间件
http.HandleFunc("/user", loggingMiddleware(createUserHandler))
```

理解方式：

```
loggingMiddleware(createUserHandler)
    ↑                    ↑
  前台（先接客）      厨师（真正干活）

流程：请求进来 → 前台登记 → 转给厨师 → 做完 → 前台记录
```

### 多个中间件叠加

```go
http.HandleFunc("/user", authMiddleware(loggingMiddleware(createUserHandler)))
// 请求进来 → auth → logging → createUserHandler
```

本质：**函数当参数传 + 函数当返回值**。不用完全理解原理，记住套路就行，写多了自然就懂。

## Gin 框架

### Gin 是什么

Go 最流行的 Web 框架，帮你省掉 `net/http` 的重复代码。

```bash
go get github.com/gin-gonic/gin
```

### 创建服务器

```go
import "github.com/gin-gonic/gin"

r := gin.Default()   // 创建路由器，自带 Logger 中间件
r.Run(":8080")       // 启动服务器
```

### Gin vs net/http 对比

| net/http | Gin |
|----------|-----|
| `w http.ResponseWriter, r *http.Request` | `c *gin.Context` |
| `if r.Method != "GET" ...` | 不用写（`r.GET` 自动限定） |
| `json.NewDecoder(r.Body).Decode()` | `c.ShouldBindJSON(&user)` |
| `json.Marshal` + `w.Header` + `w.Write` | `c.JSON(200, data)` |
| `strings.TrimPrefix` + `Atoi` | `c.Param("id")` |
| `http.HandleFunc` + if/switch | `r.GET/POST/PUT/DELETE` 分开注册 |
| `http.ListenAndServe(":8080", nil)` | `r.Run(":8080")` |

### 路由注册

```go
r.GET("/user", listUsers)           // GET /user → listUsers
r.POST("/user", createUser)         // POST /user → createUser
r.GET("/user/:id", getUser)         // GET /user/0 → getUser
r.PUT("/user/:id", updateUser)      // PUT /user/0 → updateUser
r.DELETE("/user/:id", deleteUser)   // DELETE /user/0 → deleteUser
```

`:id` 是路由参数，`/user/0` 里的 `0` 会被捕获。

### Handler 签名

```go
func handler(c *gin.Context) {
    // c 把 w 和 r 合到一起了，用 c 就能完成所有事情
}
```

### 获取路由参数与查询参数

```go
// 路由参数
id := c.Param("id")  // /user/0 → "0"

// 查询参数
name := c.Query("name")                          // /user?name=张三 → "张三"
page := c.DefaultQuery("page", "1")              // 没传就用默认值 "1"
```

### 解析请求体 JSON

```go
var user User
if err := c.ShouldBindJSON(&user); err != nil {
    c.JSON(400, gin.H{"error": err.Error()})
    return
}
// 解析成功，user 里有数据了
```

`ShouldBindJSON` 一行代替 net/http 的三行（NewDecoder + Decode + 检查 err）。

### 返回 JSON 响应

```go
c.JSON(200, user)                           // 返回单个对象
c.JSON(200, users)                          // 返回列表
c.JSON(400, gin.H{"error": "解析失败"})       // 返回错误信息
```

`gin.H` 是 `map[string]interface{}` 的简写。`c.JSON` 自动帮你做：序列化 + 设 Content-Type + 写响应。

### 参数校验

在 struct tag 里加 `binding` 规则，Gin 自动校验：

```go
type User struct {
    Name string `json:"name" binding:"required,min=2,max=50"`
    Age  int    `json:"age" binding:"required,gt=0"`
}
```

常用规则：

- `required` → 必填
- `min=N` / `max=N` → 最小/最大长度（字符串）或值（数字）
- `gt=N` / `gte=N` → 大于 / 大于等于
- `email` → 必须是邮箱格式

校验在 `c.ShouldBindJSON` 时自动执行，不需要额外代码。

### Gin 中间件

```go
func Logger() gin.HandlerFunc {
    return func(c *gin.Context) {
        start := time.Now()

        log.Printf(">>> %s %s", c.Request.Method, c.Request.URL.Path)
        c.Next()  // 放行
        cost := time.Since(start)
        log.Printf("<<< 耗时: %v", cost)
    }
}
```

核心两个函数：

- `c.Next()` → 放行，让请求继续走到 handler
- `c.Abort()` → 拦截，不让请求继续走

### 认证中间件

```go
func AuthRequired() gin.HandlerFunc {
    return func(c *gin.Context) {
        token := c.GetHeader("Authorization")
        if token == "" {
            c.JSON(401, gin.H{"error": "未登录"})
            c.Abort()  // 拦截
            return
        }
        c.Next()  // 放行
    }
}
```

### 路由分组

用法1：单个路由加中间件

```go
r.GET("/user", Logger(), listUsers)
r.GET("/user/:id", Logger(), AuthRequired(), getUser)
```

用法2：路由分组（推荐）

```go
authorized := r.Group("/user", AuthRequired())
{
    authorized.GET("/:id", Logger(), getUser)
    authorized.PUT("/:id", Logger(), updateUser)
    authorized.DELETE("/:id", Logger(), deleteUser)
}
```

分组的好处：`AuthRequired` 写一次，这组路由全部生效。

### 完整示例（带中间件）

```go
func main() {
    r := gin.Default()

    // 不需要登录的接口
    r.GET("/user", listUsers)
    r.POST("/user", createUser)

    // 需要登录的接口
    auth := r.Group("/user", AuthRequired())
    {
        auth.GET("/:id", getUser)
        auth.PUT("/:id", updateUser)
        auth.DELETE("/:id", deleteUser)
    }

    r.Run(":8080")
}
```

### 常用函数速查

| 函数 | 作用 |
|------|------|
| `gin.Default()` | 创建路由器（自带 Logger） |
| `r.GET/POST/PUT/DELETE(path, fn)` | 注册路由 |
| `r.Group(path, middleware...)` | 路由分组 |
| `c.Param("id")` | 获取路由参数 |
| `c.Query("key")` | 获取查询参数 |
| `c.DefaultQuery("key", "default")` | 获取查询参数（带默认值） |
| `c.GetHeader("key")` | 获取请求头 |
| `c.ShouldBindJSON(&data)` | 解析请求体 JSON（含校验） |
| `c.JSON(status, data)` | 返回 JSON 响应 |
| `c.Next()` | 放行 |
| `c.Abort()` | 拦截 |
| `gin.H{...}` | `map[string]interface{}` 简写 |

## 踩坑汇总

- **中文输入法逗号**：`"错误"，405` → 要用英文逗号 `"错误", 405`
- **`:=` vs `=`**：全局变量用 `=` 赋值，用 `:=` 会创建局部变量
- **`fmt.Fprinf`** → `fmt.Fprintf`（p 和 r 反了）
- **`json.Decode` 拼写**：`Docode` → `Decode`
- **`application` 拼写**：`spplication` → `application`
- **`c.Abort` 漏括号** → `c.Abort()` 才是调用
- **Gin 里用 net/http 的写法**：`json.NewDecoder(r.Body)` 是 net/http 的，Gin 里用 `c.ShouldBindJSON`
- **net/http 中 POST `/user` 不带斜杠**：注册的是 `/user/` 时会被重定向，测试时直接用 `/user/`

## 参考源

- raw/GoWeb 后端开发.md
