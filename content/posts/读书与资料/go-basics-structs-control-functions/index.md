---
title: Go 语言基础：结构体、流程控制与函数
slug: go-basics-structs-control-functions
description: Go 语言进阶语法笔记：结构体与 JSON 互转、if/switch/for 流程控制、多返回值函数，以及常见写法与注意事项
date: 2026-07-07T00:00:00+08:00
draft: false
image: cover.svg
tags:
  - Go
  - 编程基础
  - 学习笔记
  - JSON
  - 函数
categories:
  - 读书与资料
---

接上一篇，继续梳理 Go 的核心语法。这次覆盖结构体、JSON 序列化、流程控制和函数。

## 结构体 Struct

结构体的知识在 C/C++ 中已经讲过很多，这里给个框架：

```go
type Car struct {
    Brand string
    Color string
    Money float64
}
```

创建实例：

```go
var c Car
c.Brand = "特斯拉"
fmt.Println(c.Brand)
```

### struct 与 map/slice 的区别

struct 是**值类型**，不同于 map、slice 底层数据共享。struct 可以直接赋值、修改元素，赋值时会**拷贝一份完整数据**。

### struct 与 JSON 互转

这是实际开发中的高频操作。

```go
import "encoding/json"

func main() {
    c := Car{Brand: "特斯拉", Color: "白色", Money: 25.99}

    // 转成 JSON
    jsonData, err := json.Marshal(c)
    if err != nil {
        fmt.Println("转换失败:", err)
        return
    }
    fmt.Println(string(jsonData))  // string() 将字节数组转为可读字符串
}
```

⚠️ **问题**：JSON 里的字段是大写的 `Brand`，但前端习惯小写。

**解决：struct tag**，用反引号包裹：

```go
type Car struct {
    Brand string  `json:"brand"`
    Color string  `json:"color"`
    Money float64 `json:"money"`
}
```

这样序列化出来的 JSON key 就是小写了。注意反引号 `` ` `` 里面整个 `json:"xxx"` 是一个整体，冒号后面不能有空格。

## 流程控制

三个核心：`if/else`、`switch`、`for`（Go 没有 `while`）。

### if/else

Go 的 `if` 可以带**初始化语句**，用 `;` 分隔：

```go
if age := 20; age >= 18 {
    fmt.Println("成年人")
}
// age 仅在 if 块内生效
```

⚠️ 如果用到了 `else`，Go 强制要求 `} else {` 必须在**同一行**，不然编译报错：

```go
if condition {
    // ...
} else {
    // ...
}
```

### switch

Go 的 switch 不需要写 `break`，每个 case 自动结束：

```go
day := "Monday"
switch day {
case "Monday":
    fmt.Println("星期一")
case "Tuesday":
    fmt.Println("星期二")
default:
    fmt.Println("其他")
}
```

### for 循环

Go 只有 `for`，但有四种写法：

**1. 经典写法**（和 C 一样）

```go
for i := 0; i < 5; i++ {
    // ...
}
```

**2. 当 while 用**（只有一个条件）

```go
i := 0
for i < 5 {
    i++
}
```

**3. 无限循环**（配合 break）

```go
for {
    fmt.Println("一直跑")
    break
}
```

**4. range 遍历**（实际开发中用得最多）

```go
// 遍历 slice
nums := []int{10, 20, 30}
for i, v := range nums {
    fmt.Println(i, v)
}

// 遍历 map
scores := map[string]int{"张三": 90, "李四": 85}
for k, v := range scores {
    fmt.Println(k, v)
}
```

range 后面可以只用一个变量接收值，用 `_` 跳过不需要的：

```go
for _, v := range nums {
    // 只要值，不要索引
}
```

> 注意：`for` 的三部分不用括号，但分号不能省。

## 函数

Go 用 `func` 定义函数，函数必须定义在函数外。格式正常，重点看**多返回值**——这是 Go 的特色。

```go
func divide(a, b float64) (float64, error) {
    if b == 0 {
        return 0, fmt.Errorf("除数不能为零")
    }
    return a / b, nil
}

func main() {
    result, err := divide(10, 3)
    if err != nil {
        fmt.Println("出错了:", err)
    } else {
        fmt.Println("结果:", result)
    }
}
```

几点注意：

- `error` 是 Go 内置的接口类型，表示错误
- 没出错时返回 `nil`
- `fmt.Errorf` 创建一个带错误信息的 error

Go 的错误处理模式就是 **返回值 + if 判断**，没有 try-catch。这在后面写项目时会反复见到。

## 参考源

- raw/go.md
