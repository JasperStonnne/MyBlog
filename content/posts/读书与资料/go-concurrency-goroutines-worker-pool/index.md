---
title: Go 语言并发：从 Goroutine 到 Worker Pool
slug: go-concurrency-goroutines-worker-pool
description: Go 并发编程核心：Goroutine、Channel、Select、sync 包、Context、Race Detector 与 Worker Pool
date: 2026-07-14T00:00:00+08:00
draft: false
image: cover.svg
tags:
  - Go
  - 并发编程
  - Goroutine
  - Channel
  - Worker Pool
categories:
  - 技术笔记
---

承接 [Go 语言基础：从变量到指针]({{< ref "go-basics-variables-to-pointers" >}})、[Go 语言进阶：方法、接口与错误处理]({{< ref "go-methods-interfaces-errors" >}}) 和 [Go 语言工程化：模块管理与项目结构]({{< ref "go-engineering-modules-project-structure" >}})，这一篇覆盖 Go 最核心的特性：并发编程。

## 并发 vs 并行

先区分两个概念：

- **并行（Parallel）**：两个人同时吃两碗面
- **并发（Concurrency）**：一个人交替吃两碗面（这口吃这碗，下口吃那碗）

Go 的并发模型就是让你用很少的资源同时做很多事。

## Goroutine

一句话：轻量级线程，用 `go` 关键字启动。

```go
// 普通调用
fmt.Println("hello")

// 用 go 启动一个 goroutine
go fmt.Println("hello") // 加个 go 就行
```

加了 `go`，这个函数就会在后台跑，不阻塞当前代码。

### 先看个例子

```go
package main

import (
    "fmt"
    "time"
)

func sayHello() {
    fmt.Println("hello")
}

func main() {
    go sayHello()      // 后台跑
    fmt.Println("main") // 主线程继续
}
```

你猜输出什么？

答案：只输出 `main`。

为什么？因为 `main` 函数结束了，程序就退出了，goroutine 还没来得及跑。

这就像你叫了个外卖（`go sayHello`），但你立刻出门了（main 结束），外卖员来了发现没人。

### 怎么等 goroutine 跑完？

最简单的方式：加个 sleep（但这是**错误示范**）

```go
go sayHello()
time.Sleep(time.Second) // 等1秒，goroutine 有时间跑了
fmt.Println("main")
```

能工作，但不准、不优雅——你不知道 goroutine 到底要跑多久。

正确方式：用 channel（后面会详细讲）

```go
done := make(chan bool)

go func() {
    fmt.Println("hello")
    done <- true // 说一声：我做完了
}()

<-done // 等着，直到收到信号
fmt.Println("main")
```

现在输出：

```
hello
main
```

### 用比喻理解

- 普通函数调用：你叫同事做事，站在旁边等他做完
- `go` 函数调用：你叫同事做事，自己先去忙别的
- channel：同事做完了发消息通知你

## Channel

### 为什么需要 Channel？

goroutine 是独立运行的，它们之间怎么传数据？

```go
// 两个 goroutine，一个生产数据，一个消费数据
// 怎么把数据从 A 传到 B？
```

答案：用 channel，就像一根水管，一头进，一头出。

### 基本用法

```go
// 创建 channel
ch := make(chan int)    // 传 int 类型的 channel

// 发送数据（往管子里塞）
ch <- 42

// 接收数据（从管子里取）
x := <-ch
```

### Channel 的特性：阻塞

这是最关键的：

- 发送方：`ch <- 42` → 没人接收就等着（阻塞）
- 接收方：`x := <-ch` → 没人发送就等着（阻塞）

就像一个没有缓冲的传送带：

- 你放东西上去，必须有人在另一头接着，否则你就一直举着
- 你去拿东西，必须有人放了东西，否则你就一直等着

### 用阻塞特性来同步

这就是为什么 channel 能替代 sleep 来等 goroutine：

```go
done := make(chan bool)

go func() {
    fmt.Println("干活中...")
    done <- true // 干完了，发信号
}()

<-done // 等着，直到收到信号
fmt.Println("结束")
```

- goroutine 没发信号 → main 在 `<-done` 那行等着
- goroutine 发了信号 → main 继续往下跑

### 缓冲 Channel

普通 channel 没有缓冲，一发一收必须配对。可以加缓冲：

```go
ch := make(chan int, 3) // 缓冲区大小为3

ch <- 1  // 塞进去，不等（缓冲区没满）
ch <- 2  // 塞进去，不等
ch <- 3  // 塞进去，不等
ch <- 4  // 缓冲区满了，等着（阻塞）
```

比喻：

- 没缓冲：打电话，必须对方接了你才能说话
- 有缓冲：发微信，对方没看之前你也能发几条

### 关闭 Channel

发送方说"我不再发了"：

```go
close(ch)
```

接收方可以检测 channel 是否关闭：

```go
x, ok := <-ch
if !ok {
    fmt.Println("channel 已关闭")
}
```

### 概念总结

| 操作 | 含义 |
|------|------|
| `make(chan T)` | 创建 channel |
| `ch <- data` | 发送 |
| `x := <-ch` | 接收 |
| `close(ch)` | 关闭 |
| `make(chan T, n)` | 带缓冲的 channel |

## Select

### Select 是什么？

一句话：同时监听多个 channel，哪个先有数据就处理哪个。

### 为什么需要 Select？

假设你有两个 channel：

```go
ch1 := make(chan string)
ch2 := make(chan string)
```

用普通方式只能等一个：

```go
msg := <-ch1 // 只能等 ch1，ch2 来了也不知道
```

用 select 可以同时等：

```go
select {
case msg := <-ch1:
    fmt.Println(msg)
case msg := <-ch2:
    fmt.Println(msg)
}
```

哪个先有数据就执行哪个。

### 完整例子

```go
package main

import (
    "fmt"
    "time"
)

func main() {
    ch1 := make(chan string)
    ch2 := make(chan string)

    // 1秒后往 ch1 塞数据
    go func() {
        time.Sleep(1 * time.Second)
        ch1 <- "来自 ch1"
    }()

    // 2秒后往 ch2 塞数据
    go func() {
        time.Sleep(2 * time.Second)
        ch2 <- "来自 ch2"
    }()

    // 同时等 ch1 和 ch2
    select {
    case msg := <-ch1:
        fmt.Println(msg)
    case msg := <-ch2:
        fmt.Println(msg)
    }
}
```

输出：`来自 ch1`（因为 ch1 先到）

## sync.WaitGroup

### 为什么需要 WaitGroup？

你启动了多个 goroutine，想等它们全部做完再继续：

```go
go task1()
go task2()
go task3()
// 怎么知道三个都做完了？
```

之前用 channel 可以，但要创建多个 channel 很麻烦。WaitGroup 就是专门解决这个的。

### 怎么用？三步

```go
var wg sync.WaitGroup

wg.Add(1) // 第1步：告诉 wg 你要等几个任务

go func() {
    // 做事情...
    wg.Done() // 第3步：做完了，报告一下
}()

wg.Wait() // 第2步：等着，全部做完才继续
```

## sync.Mutex 互斥锁

解决的问题：多个 goroutine 同时读写同一个变量。

```go
var mu sync.Mutex

mu.Lock()
counter++
mu.Unlock()
```

## sync.Once

```go
var once sync.Once
once.Do(func() {
    // 这段代码只会执行一次
})
```

不管你调多少次 `once.Do()`，里面的函数只执行第一次。

什么时候用？初始化操作，比如数据库连接、配置加载，只做一次。

## Context

### Context 是什么？

一句话：控制 goroutine 的生命周期，告诉它"别干了，取消"。

### 为什么需要 context？

假设你发了一个 HTTP 请求，用户等不及关了页面：

```
用户 → 请求 → 后端启动 goroutine 处理 → 数据库查询中...
用户关了页面
goroutine 还在跑，浪费资源。
```

用 context 可以通知 goroutine："别查了，用户走了"。

### 先忘掉代码，想一个场景

你叫了一个外卖：

- 你：下单（启动 goroutine）
- 外卖员：开始送餐（goroutine 在跑）
- 情况1：你等不及了，打电话说"别送了"（手动取消）
- 情况2：你下单时就说"30分钟不到就取消"（超时取消）

context 就是那个电话。

### 代码里的角色

```go
ctx, cancel := context.WithTimeout(context.Background(), 2*time.Second)
```

- `ctx` → 外卖员手里的对讲机（能听到"取消"的信号）
- `cancel` → 你手里的电话（主动打过去说"取消"）
- `2*time.Second` → 30分钟超时（2秒后自动取消）

### goroutine 那边怎么听信号？

```go
select {
case <-ctx.Done(): // 对讲机响了，说"取消"
    fmt.Println("收到取消，退出")
    return
default: // 没响，继续干活
    fmt.Println("干活中...")
}
```

`ctx.Done()` 就是对讲机，context 取消时它就响（channel 关闭）。

### 整个流程

1. main：创建 context（设定2秒超时）
2. main：启动 worker，把对讲机给它
3. worker：干活...干活...
4. 2秒后：超时，context 自动取消
5. worker：听到对讲机响了，退出

### 两种取消方式

#### 1. 手动取消：WithCancel

```go
ctx, cancel := context.WithCancel(context.Background())
```

你自己决定什么时候取消：

```go
go worker(ctx)
// 过了一会儿，你决定取消
cancel() // 打电话说"别干了"
```

就像：你叫了外卖，等了5分钟不耐烦了，主动打电话取消。

#### 2. 超时取消：WithTimeout

```go
ctx, cancel := context.WithTimeout(context.Background(), 2*time.Second)
```

设个时间，到了自动取消：

```go
go worker(ctx)
// 不用管了，2秒后自动取消
// cancel() 也可以提前调，但不是必须的
```

## Race Detector

### 什么是数据竞争？

之前 Mutex 的例子，不加锁会出错：

```go
counter++ // 多个 goroutine 同时改，结果不对
```

这就是数据竞争，很难复现，偶尔出错，排查很痛苦。

### Race Detector

Go 自带一个工具，帮你检测数据竞争：

```bash
go run -race demo.go
go build -race -o app
go test -race ./...
```

加一个 `-race` 就行。

### 看个例子

```go
package main

import (
    "fmt"
    "sync"
)

func main() {
    counter := 0
    var wg sync.WaitGroup
    for i := 0; i < 1000; i++ {
        wg.Add(1)
        go func() {
            defer wg.Done()
            counter++
        }()
    }
    wg.Wait()
    fmt.Println("counter:", counter)
}
```

```bash
go run -race test.go
```

输出：

```
WARNING: DATA RACE
...
Found 2 data race(s)
```

### -race 告诉你什么

- 哪个 goroutine 在读（goroutine 12）
- 哪个 goroutine 在写（goroutine 8）
- 在第几行（第16行：`counter++`）

### 一句话总结

- `go run -race` → 跑代码 + 检测数据竞争
- `go test -race` → 跑测试 + 检测数据竞争（写项目时常用）

## Worker Pool 并发模式

一句话：固定几个 goroutine，从队列里取任务执行。

### 为什么需要？

假设你有1000个任务：

```go
// 方式1：每个任务一个 goroutine
for i := 0; i < 1000; i++ {
    go doTask(i)
}
```

启动1000个 goroutine，资源占用大，不好控制。

```go
// 方式2：Worker Pool，只启动3个 worker
for i := 0; i < 3; i++ {
    go worker(tasks)
}
// 3个 worker 从 tasks 里取任务，取完一个再取下一个
```

就像餐厅只有3个厨师，100个菜排队等他们做，做完一个做下一个。

### 模型

```
任务队列（channel）：[任务1, 任务2, 任务3, ...任务N]

Worker 1：从队列取任务 → 执行 → 再取下一个
Worker 2：从队列取任务 → 执行 → 再取下一个
Worker 3：从队列取任务 → 执行 → 再取下一个
```

### 完整代码

```go
package main

import (
    "fmt"
    "time"
)

// worker 函数：每个 worker 从 tasks 这个 channel 里不断取任务执行
// id：worker 的编号，用来区分是哪个 worker 在干活
// tasks：任务队列，只读 channel（<-chan 表示只能从里面取，不能塞）
func worker(id int, tasks <-chan int) {
    // range tasks：从 channel 里取任务，取到就执行
    // channel 关闭后，range 自动结束，for 循环退出
    for task := range tasks {
        fmt.Printf("worker %d 开始处理任务 %d\n", id, task)
        time.Sleep(time.Second) // 模拟干活耗时1秒
        fmt.Printf("worker %d 完成任务 %d\n", id, task)
    }
}

func main() {
    // 创建一个带缓冲的 channel，缓冲区大小10
    tasks := make(chan int, 10)

    // 启动3个 worker，用 goroutine 并发跑
    for i := 1; i <= 3; i++ {
        go worker(i, tasks)
    }

    // 往 tasks 里塞5个任务（编号1到5）
    for j := 1; j <= 5; j++ {
        tasks <- j
    }

    // 关闭 channel，告诉 worker："没有新任务了"
    close(tasks)

    // 等6秒，让 worker 有时间把任务做完
    time.Sleep(6 * time.Second)
    fmt.Println("全部完成")
}
```

### 执行流程

1. main：启动 worker 1、worker 2、worker 3
2. main：塞任务 1、2、3、4、5
3. worker 1：取到任务1，开始干
4. worker 2：取到任务2，开始干
5. worker 3：取到任务3，开始干
6. worker 1：任务1做完，取任务4
7. worker 2：任务2做完，取任务5
8. worker 3：任务3做完，队列空了，等着...
9. main：`close(tasks)` → worker 们收到关闭信号，全部退出
10. main：打印"全部完成"

### 到底怎么保证"干完了取下一个"还很有序？

关键：channel 的阻塞特性。

```go
for task := range tasks {
    // 干活...
}
```

`range tasks` 会自动做这件事：

- 有任务 → 取出来，执行
- 没任务 → 阻塞，等着
- channel 关闭 → 退出 for 循环

展开来看真实流程：

```
任务队列（channel）：[1, 2, 3, 4, 5]

worker 1：range tasks → 取到 1 → 开始干
worker 2：range tasks → 取到 2 → 开始干
worker 3：range tasks → 取到 3 → 开始干

// 三个 worker 都在干活，队列里还有 4 和 5

worker 1：干完 → range tasks → 取到 4 → 开始干
worker 2：干完 → range tasks → 取到 5 → 开始干
worker 3：干完 → range tasks → 队列空了 → 阻塞，等着

// 没有新任务了
main：close(tasks)

worker 3：检测到关闭 → 退出
worker 1：干完任务4 → 检测到关闭 → 退出
worker 2：干完任务5 → 检测到关闭 → 退出
```

### 为什么不需要锁？

因为 channel 天然安全：

- 同一个任务不会被两个 worker 取到
- channel 保证：取走一个就少一个

### 一句话总结

`range channel` → 没任务就阻塞，有任务就取一个，取完再取下一个。channel 关闭 → range 自动结束。

## 参考源

- raw/Go.md
