---
title: Go 语言基础：从变量到 Map
slug: go-basics-variables-to-map
description: Go 语言核心语法笔记：变量声明、常量与 iota、数据类型、切片、Map，以及常见陷阱与例子
date: 2026-07-07T00:00:00+08:00
draft: false
image: cover.svg
tags:
  - Go
  - 编程基础
  - 学习笔记
  - Map
categories:
  - 读书与资料
---

一份 Go 入门语法梳理，覆盖变量、常量、类型、切片、Map 五块，重点标出踩坑点。

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
name := "Jasper"
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
const now = time.Now()  // 编译报错，time.Now() 是运行时才有的
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

所以实际开发用可变数组：**切片**。

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

> **扩容**：当 append 超过 cap 时，Go 会自动分配一个更大的底层数组，把旧数据拷过去。你不需要手动管，但要知道这个机制存在，它会影响性能。

### 切片操作

```go
nums := []int{10, 20, 30, 40, 50}
sub := nums[1:3]    // 左闭右开 -> [20, 30]
all := nums[:]      // 全部
first := nums[:2]   // [10, 20]
last := nums[2:]    // [30, 40, 50]
```

**陷阱**：切片出来的 `sub` 和原 slice **共享底层数组**，修改 `sub` 会影响 `nums`：

```go
sub[0] = 999
fmt.Println(nums)  // [10, 999, 30, 40, 50]，nums 也被改了
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

取一个不存在的 key 会怎么样？一般直接返回 value 类型的零值。但这会出问题：`0` 是真实分数，还是因为 key 不存在？所以提供 comma-ok 模式：

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
m["a"] = 1            // nil map 不能写入
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

## 参考源

- raw/go学习之路.md
