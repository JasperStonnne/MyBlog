---
title: Go 语言基础：从变量到指针
slug: go-basics-variables-to-pointers
description: Go 语言入门语法全覆盖：变量、常量、数据类型、切片、Map、结构体、JSON、流程控制、函数、指针，附踩坑点与练习
date: 2026-07-07T00:00:00+08:00
draft: false
image: cover.svg
tags:
  - Go
  - 编程基础
  - 学习笔记
  - 指针
categories:
  - 技术笔记
---

一份 Go 入门语法梳理，从变量到指针，覆盖日常开发最常用的核心概念。

## 变量声明

两种方法。

### 1. `var` 标准形式

```go
var name type = value   // 标准形式
var pi = 22             // 可以自动推断类型，省略 type
var score int           // 可以不给 value
```

声明不赋值时，系统自动给**零值**。

### 2. `:=` 简短声明

```go
name := "草泥马"
```

比较常见。限制：

- 只能在函数里用（全局范围必须用 `var`）
- 不能只声明不赋值

## 常量与 iota

常量不可更改，直接用 `const` 声明：

```go
const Pi = 3.14
const name = "xxx"
```

常量必须在**编译期**确定，不能把运行时的值赋给常量：

```go
const now = time.Now()  // ✗ 编译报错，time.Now() 是运行时才有的
```

`iota` 是 Go 在 `const` 块里提供的**自动计数器**：

```go
const (
    Sunday    = iota  // 0
    Monday            // 1
    Tuesday           // 2
    Wednesday         // 3
)
```

比较简洁。想跳过某个值，用 `_ = iota` 占位就好。

> 注意：Go 没有 `enum` 关键字，做枚举需要用 `const` + `iota`。

## 数据类型

Go 是**强类型**语言，类型间不能自动转换（区别于 JS、Python），只能手动强制转换：

```go
var a = 10
var b = 3.14
// 要算 a + b，必须先转类型
float64(a) + b
```

### 基本类型一览

```go
var i int = 42          // 平台相关，64 位系统就是 64 位
var i8 int8 = 127       // 范围 -128 ~ 127
var u uint = 42         // 无符号，没有负数
var f32 float32 = 3.14
var f64 float64 = 3.14  // 一般都用 float64
var s string = "hello"
var b bool = true       // 只有 true/false，没有 1/0
```

### rune 与 byte 的区别

```go
var b byte = 'A'   // byte 就是 uint8，存储一个 ASCII 字符
var r rune = '中'  // rune 就是 int32，存储一个 Unicode 字符
```

为什么要这样？英文字母只占一个字节，而中文、emoji 占 3~4 个字节，一个 `byte` 装不下，需要 `rune`。

```go
s := "hello"
len(s)          // 5，len 对 string 返回字节数
len([]rune(s))  // 5，返回字符数
```

想拿**字符数**就用 `len([]rune(s))`。原理：`[]rune(s)` 把 string 转成 `[]rune` 切片。

### 字符串的两种写法

```go
s1 := "hello\nworld"   // 双引号，解释型：\n 会被解释成换行
s2 := `hello\nworld`   // 反引号，原始字符串：字面上的 \n 不会换行
```

## 切片 Slice

先说数组，但几乎不会用它：

```go
var arr [3]int = [3]int{1, 2, 3}  // 长度固定，声明时定死
```

所以实际开发用可变数组——**切片**。

### slice 声明

```go
nums := []int{1, 2, 3}       // 字面量
s := make([]int, 0, 10)      // make 指定长度 0，容量 10
var names []string           // nil，长度 0，后面 append
```

### len 与 cap

每个 slice 有两个属性：

- `len` 长度：里面实际有多少个元素
- `cap` 容量：底层数组最多能装多少个

```go
s := make([]int, 3, 10)
fmt.Println(len(s))  // 3
fmt.Println(cap(s))  // 10
```

### append 添加元素

```go
nums := []int{1, 2, 3}
nums = append(nums, 4)        // 末尾多了个 4
nums = append(nums, 5, 6, 7)  // now [1,2,3,4,5,6,7]
```

注意：`append` 返回的是**新的 slice**，必须赋值回去。

> **扩容**：当 append 超过 cap 时，Go 会自动分配一个更大的底层数组，把旧数据拷过去。你不需要手动管，但要知道这个机制存在——它会影响性能。

### 切片操作

```go
nums := []int{10, 20, 30, 40, 50}
sub := nums[1:3]    // 左闭右开 → [20, 30]
all := nums[:]      // 全部
first := nums[:2]   // [10, 20]
last := nums[2:]    // [30, 40, 50]
```

⚠️ **陷阱**：切片出来的 `sub` 和原 slice **共享底层数组**，修改 `sub` 会影响 `nums`：

```go
sub[0] = 999
fmt.Println(nums)  // [10, 999, 30, 40, 50] —— nums 也被改了！
```

要独立拷贝用 `copy`。

### 遍历

```go
fruits := []string{"apple", "banana", "cherry"}
for i, v := range fruits {
    fmt.Printf("[%d] %s\n", i, v)
}

// 不要索引就用 _
for _, v := range fruits {
    // ...
}
```

## Map

其实就是映射。

```go
// 创建一个 map，key 是 string，value 是 int
scores := map[string]int{
    "math":    90,
    "english": 85,
    "chinese": 92,
}

// 取值
fmt.Println(scores["math"])  // 90

// 添加/修改
scores["science"] = 88  // 新增
scores["math"] = 95     // 修改（key 已存在就覆盖）

// 删除
delete(scores, "english")
```

长度用 `len()`。

### Comma-ok 模式

取一个不存在的 key 会怎么样？一般直接返回 value 类型的零值。但这会出问题——`0` 是真实分数，还是因为 key 不存在？所以提供 comma-ok 模式：

```go
score, ok := scores["physics"]
if ok {
    fmt.Println("找到了：", score)
} else {
    fmt.Println("不存在")
}
// ok 是 bool 型变量
```

### Map 的零值

```go
var m map[string]int  // nil map
m["a"] = 1            // 🙅 nil map 不能写入
```

和 slice 一样，声明之后必须**初始化**才能使用：

```go
m := map[string]int{}         // 方式 1：空字面量
m := make(map[string]int)     // 方式 2：make
```

### 遍历

```go
scores := map[string]int{"math": 90, "english": 85}
for subject, score := range scores {  // 同时给 subject、score 赋值
    fmt.Printf("%s: %d\n", subject, score)
}
```

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

## 指针

核心概念：

| 概念 | 符号 | 作用 |
|------|------|------|
| 取地址 | `&a` | 获取变量 a 的内存地址 |
| 解引用 | `*p` | 通过地址，读取或修改那个地址里的值 |

指针类型：

| 普通类型 | 对应指针类型 | 读法 |
|----------|--------------|------|
| int | `*int` | 指向 int 的指针 |
| string | `*string` | 指向 string 的指针 |
| User | `*User` | 指向 User 的指针 |

指针的零值是 `nil`：

```go
var p *int  // p 的零值是 nil，没有指向任何东西
```

两个必须记住的事：

1. Go 函数传参永远是**复制**。想改原件，必须传指针。
2. `&` 是"取地址"，`*` 是"去地址找东西"。两个方向相反。

> 直觉：普通变量是盒子里装着值，指针是纸条上写着"盒子在哪"。

### 指针与 struct

```go
func birthday(u *User) {
    u.Age++  // Go 自动解引用，不用写 *u（语法糖）
}

func main() {
    user := User{"Jasper", 22}
    birthday(&user)             // 传地址
    fmt.Println(user.Age)       // 23
}
```

Go 中局部变量习惯用小驼峰 `maxStudent` 而不是大驼峰 `MaxStudent`。import 用小括号 `()` 而不是花括号 `{}`。

## 练习：学生成绩管理

一个综合 struct、slice、指针的小练习：

```go
package main

import "fmt"

type Student struct {
    Name  string
    Grade float64
}

// 找出成绩最高的学生
func findMaxGrade(students []Student) Student {
    maxStudent := students[0]
    for _, student := range students {
        if student.Grade > maxStudent.Grade {
            maxStudent = student
        }
    }
    return maxStudent
}

// 给学生加 5 分（通过指针修改原件，不需要返回值）
func plusFive(student *Student) {
    student.Grade += 5
}

func main() {
    students := []Student{
        {Name: "Alice", Grade: 85},
        {Name: "Bob", Grade: 90},
        {Name: "Charlie", Grade: 78},
    }
    maxStudent := findMaxGrade(students)
    plusFive(&maxStudent)
    fmt.Printf("最高分学生: %s, 成绩: %.2f\n", maxStudent.Name, maxStudent.Grade)
}
```

## 参考源

- raw/go学习之路.md
- raw/go.md
