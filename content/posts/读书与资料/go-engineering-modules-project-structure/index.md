---
title: Go 语言工程化：模块管理与项目结构
slug: go-engineering-modules-project-structure
description: Go 语言项目工程化实践：Go Modules、package 拆分、internal 访问控制、常见目录结构与第三方依赖管理
date: 2026-07-13T15:42:45+08:00
draft: false
image: cover.svg
tags:
  - Go
  - 工程化
  - Go Modules
  - 项目结构
  - 学习笔记
categories:
  - 技术笔记
---

承接 [Go 语言基础：从变量到指针]({{< relref "/posts/读书与资料/go-basics-variables-to-pointers" >}}) 和 [Go 语言进阶：方法、接口与错误处理]({{< relref "/posts/读书与资料/go-methods-interfaces-errors" >}})，这一篇覆盖 Go 项目的工程化核心：模块管理、包的组织、目录结构。

## Go Modules

之前写的代码都在单个 `.go` 文件里，使用 `package main`，直接 `go run xxx.go` 就能跑。但真实项目通常会有几十个 `.go` 文件、多个包，还会用到第三方库。这些东西怎么管理？答案就是 Go Modules。

### go.mod 是什么？

`go.mod` 就像项目的“身份证”，它主要记录三类信息：

- 模块路径是什么：`module example.com/demo`
- 项目使用哪个 Go 版本：`go 1.26.3`
- 项目依赖哪些第三方模块：由 `require` 指令记录

### 为什么要用 Go Modules？

假设你写了一个项目，用了第三方库。朋友拿到代码后，怎么知道需要哪些依赖和版本？

**没有 `go.mod`：**

> 朋友：你用了哪些库？
>
> 你：好像有 xxx、yyy，版本忘了……

**有 `go.mod`：**

> 朋友：看 `go.mod` 就知道了，执行 `go mod download` 或 `go mod tidy` 就能准备好依赖。

### 常用命令

#### `go mod init 模块路径`

在项目开始时创建 `go.mod`：

```bash
go mod init example.com/demo
```

生成的文件类似这样：

```go.mod
module example.com/demo

go 1.26.3
```

练习项目可以使用 `demo` 这类简单名称；准备发布的模块通常使用代码仓库地址，例如 `github.com/用户名/项目名`，这样别人才能通过稳定的路径导入它。

#### `go mod tidy`

`go mod tidy` 会根据源码中的 `import` 整理依赖：

1. 添加代码需要、但 `go.mod` 还没有记录的模块
2. 删除代码已经不再需要的模块
3. 补充或清理 `go.sum` 中的校验信息

简单说，它会让模块文件与当前代码保持一致。改完依赖后跑一次，通常是个好习惯。

#### `go get 模块路径`

给当前模块添加或升级依赖：

```bash
go get github.com/gin-gonic/gin
```

它会解析并下载依赖，同时更新 `go.mod` 和 `go.sum`。如果你要安装一个命令行工具，而不是给当前项目添加依赖，应使用带版本号的 `go install`：

```bash
go install example.com/tool@latest
```

### go.sum 是什么？

执行 `go mod tidy`、`go get` 或构建项目后，通常会出现 `go.sum`。它记录依赖模块内容的加密校验值：

- 项目使用某个模块的特定版本
- `go.sum` 保存这个版本及其 `go.mod` 文件的校验值
- 再次下载时，Go 会验证拿到的内容是否一致

`go.sum` 不是用来锁死所有依赖版本的锁文件；真正决定依赖版本的是 `go.mod` 和 Go 的版本选择规则。`go.mod` 与 `go.sum` 都应该提交到 Git，通常不需要手动编辑 `go.sum`。

## package 拆分

### 为什么要拆 package？

如果所有代码都堆在 `main.go` 里，用户管理、订单管理和数据库连接会逐渐混在一起。按职责拆包之后，结构会清晰很多：

```text
demo/
  go.mod
  main.go          # 程序入口
  user/
    user.go        # 用户相关功能
  order/
    order.go       # 订单相关功能
  db/
    db.go          # 数据库相关功能
```

### 怎么拆？

先创建 `user/user.go`，声明它属于 `user` 包：

```go
package user

import "fmt"

type User struct {
    Name string
    Age  int
}

func Print(u User) {
    fmt.Printf("%s, %d\n", u.Name, u.Age)
}
```

再在项目根目录的 `main.go` 中导入它：

```go
package main

import "example.com/demo/user"

func main() {
    u := user.User{Name: "Jasper", Age: 22}
    user.Print(u)
}
```

这里的导入路径由两部分组成：

```text
example.com/demo/user
└── 模块路径 ──┘ └包目录
```

也就是：**导入路径 = `go.mod` 中的模块路径 + 包所在的相对目录**。

### package 与可见性

同一目录下的普通 `.go` 文件必须属于同一个包。包名通常与目录名最后一段一致，这是一条强烈推荐的惯例，但不是“包名必须等于目录名”的语法规则。

标识符是否能被其他包使用，则由首字母大小写决定：

| 写法 | 可见范围 |
|---|---|
| `User`、`Print` | 首字母大写，可以被其他包访问 |
| `user`、`print` | 首字母小写，只能在当前包内访问 |

## internal 访问控制

### internal 是什么？

一句话：`internal` 是 Go 编译器真正认识的特殊目录，用来限制包的导入范围。

假设项目结构如下：

```text
myapp/
  go.mod              # module example.com/myapp
  cmd/api/main.go
  internal/db/db.go
```

`example.com/myapp/internal/db` 只能被 `example.com/myapp` 目录树中的代码导入：

```go
// myapp/cmd/api/main.go：可以导入
import "example.com/myapp/internal/db"
```

另一个项目则不能导入它：

```go
// otherapp/main.go：编译报错
import "example.com/myapp/internal/db"
```

更准确地说，规则不是“同一个模块才能用”，而是：**只有位于 `internal` 父目录树中的代码才能导入它**。大多数项目把 `internal` 放在模块根目录，所以看起来就像“只有本模块能用”。

| 目录 | 编译器是否特殊处理 | 用途 |
|---|---|---|
| `internal/` | 是，限制外部导入 | 不希望暴露给外部项目的实现 |
| `pkg/` | 否，只是社区惯例 | 明确准备对外复用的包，可选 |

## 常见项目结构

Go 官方没有规定一套所有项目都必须照搬的“标准目录结构”。下面是一种常见布局，适合已经出现多个入口和多个业务模块的项目：

```text
myproject/
  cmd/                    # 程序入口
    api/
      main.go             # 启动 HTTP 服务
    worker/
      main.go             # 启动后台任务
  internal/               # 不对模块外部开放的代码
    config/
      config.go           # 配置读取
    db/
      db.go               # 数据库连接
    user/
      user.go             # 用户相关逻辑
    handler/
      handler.go          # HTTP 处理函数
  pkg/                    # 确实需要对外复用时再添加
    utils/
      utils.go
  go.mod
  go.sum
```

三个常见目录的职责：

- `cmd/`：每个子目录对应一个可执行程序，包名为 `main`；通常只负责依赖组装、配置和启动
- `internal/`：放不希望被外部项目导入的实现代码，不代表所有业务逻辑都必须塞在这里
- `pkg/`：表达“这些包准备公开复用”的意图，没有编译器层面的特殊含义；小项目完全可以没有

目录不是越多越工程化。只有一个可执行程序的小项目，把 `main.go` 放在根目录也很正常；随着入口和职责增多，再引入 `cmd/`、`internal/` 即可。

### 为什么要多一层 cmd/？

一个项目可能提供多个启动方式：

```text
cmd/api/main.go      # 启动 HTTP 服务
cmd/worker/main.go   # 启动后台任务
```

它们是两个入口，但可以共享 `internal/` 里的业务代码。`cmd/` 下的程序应尽量轻量：

```go
// cmd/api/main.go
package main

import "example.com/myproject/internal/server"

func main() {
    server.Start()
}
```

`cmd/api` 中不一定只能有一个 `main.go`，也可以有多个属于 `package main` 的文件；重要的是入口层保持轻薄，不把大量业务逻辑堆在这里。

## 引入第三方包

标准库是 Go 自带的，例如 `fmt`、`net/http`。标准库之外，由其他项目提供的包就是第三方包，例如：

- `github.com/gin-gonic/gin`：Web 框架
- `github.com/jackc/pgx/v5`：PostgreSQL 驱动与工具包
- `github.com/spf13/cobra`：命令行应用框架

以 UUID 包为例，先添加依赖：

```bash
go get github.com/google/uuid
```

再在代码里导入并使用：

```go
package main

import (
    "fmt"

    "github.com/google/uuid"
)

func main() {
    id := uuid.New()
    fmt.Println(id)
}
```

此时 `go.mod` 可能包含：

```go.mod
module example.com/user-api

go 1.26.3

require github.com/google/uuid v1.6.0
```

改动依赖后执行：

```bash
go mod tidy
go test ./...
```

第一条命令整理模块文件，第二条命令确认整个模块仍能编译并通过测试。到这里，一个 Go 项目从单文件走向模块化工程的基本骨架就搭起来了。
