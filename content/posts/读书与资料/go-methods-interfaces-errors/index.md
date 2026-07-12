---
title: Go 语言进阶：方法、接口与错误处理
slug: go-methods-interfaces-errors
description: Go 语言进阶语法——方法、接口、空接口与类型断言、错误处理、panic/recover、泛型，从基础到实战的第二步
date: 2026-07-07T00:00:00+08:00
draft: false
image: cover.svg
tags:
  - Go
  - 学习笔记
  - 方法
  - 接口
  - 错误处理
categories:
  - 读书与资料
---

承接 [Go 语言基础：从变量到指针](/p/go-basics-variables-to-pointers/)，这一篇覆盖 Go 的进阶核心概念：方法、接口、错误处理，以及泛型入门。

## 方法

```go
// 普通函数：参数是 Student
func greet(s Student) string {
    return "Hello, " + s.Name
}

// 方法：绑在 Student 上
func (s Student) greet() string {
    return "Hello, " + s.Name
}
```

区别就在于函数名前面多了一个 `(s Student)`，这叫**接收者（receiver）**。

调用方式不同：

```go
student := Student{"Jasper", 90}

// 函数调用
greet(student)

// 方法调用
student.greet()
```

返回一个字符串可以用 `fmt.Sprintf`。

### 值接收者 vs 指针接收者

在接收者的类型前加 `*` 就变成指针接收者。值接收者只改副本，指针接收者改原件。

```go
// 值接收者：改的是副本
func (s Student) setScore(score float64) {
    s.Grade = score  // 不影响原值
}

// 指针接收者：改的是原件
func (s *Student) setScore(score float64) {
    s.Grade = score  // 修改原值
}
```

## 接口

Go 中最重要的概念之一。

```go
// 定义接口：任何有 speak() 方法的类型，都算 Speaker
type Speaker interface {
    speak() string
}

// Dog 有 speak() 方法，Dog 实现了 Speaker
type Dog struct {
    Name string
}

func (d Dog) speak() string {
    return "Woof"
}

// 同理 Cat 亦如此
type Cat struct {
    Name string
}

func (c Cat) speak() string {
    return "Meow"
}

// 通用函数：接受任何 Speaker
func makeItSpeak(s Speaker) {
    fmt.Println(s.speak())
}

func main() {
    dog := Dog{"Buddy"}
    cat := Cat{"Kitty"}
    makeItSpeak(dog)  // Woof!
    makeItSpeak(cat)  // Meow!
}
```

三个要点：

- **接口定义**：`type Speaker interface { speak() string }` — 定义了一个"能力"，不关心谁实现
- **隐式实现**：Dog 和 Cat 都有 `speak()` 方法，所以它们自动"算"是 Speaker，不需要显式声明 `implements`
- **通用函数**：`makeItSpeak(s Speaker)` 接受任何 Speaker，不关心具体是 Dog 还是 Cat

### 嵌入接口

小接口可以组合成大接口：

```go
// 两个独立接口
type Speaker interface { speak() string }
type Walker interface { walk() string }

// 嵌入组合
type SpeakerWalker interface {
    Speaker  // 把 Speaker 的方法拿过来
    Walker   // 把 Walker 的方法拿过来
}
```

效果：`SpeakerWalker` 自动拥有 `speak()` 和 `walk()` 两个方法。

### 接口哲学：小接口 > 大接口

Go 的接口是隐式实现的：

```go
// C++ / Java：必须显式声明"我实现了这个接口"
class Person implements Speaker { ... }

// Go：只要你有方法，就自动满足接口
type Person struct { Name string }
func (p Person) speak() string { return "hi" }
// Person 自动满足 Speaker，不需要声明
```

标准库里最常见的接口：

```go
// io.Reader — 只有一个方法
type Reader interface {
    Read(p []byte) (n int, err error)
}

// io.Writer — 只有一个方法
type Writer interface {
    Write(p []byte) (n int, err error)
}
```

Go 标准库里很多接口只有 1~2 个方法，而不是把所有功能堆在一个大接口里。

为什么小接口好？

1. **容易实现** — 一个方法的接口，任何类型都能轻松满足
2. **容易组合** — 用嵌入把小接口组合成大接口
3. **灵活** — 你只关心你需要的能力，不关心具体类型

| 特性 | Go | Java/C++ |
|------|-----|----------|
| 实现方式 | 隐式（自动满足） | 显式（需要声明） |
| 接口大小 | 通常 1-2 个方法 | 通常很多方法 |
| 设计哲学 | 小接口，按需组合 | 大接口，一次定义 |

### io.Reader 与 io.Writer

这是 Go 标准库里最常用的两个接口，用来处理"数据流"。

先想一个问题：从哪里读数据？

- 从文件读
- 从网络读
- 从键盘读
- 从字符串读

这些地方都能读数据，但它们的实现完全不同。接口解决这个问题：

```go
// io.Reader：任何"能读出数据"的东西
type Reader interface {
    Read(p []byte) (n int, err error)
}
```

翻译成大白话：

> "你只要有 Read 方法，能往 p 里填数据，你就是个 Reader"

| 参数/返回 | 含义 |
|-----------|------|
| `p []byte` | 一个空的容器（桶），你把读到的数据放进去 |
| 返回 `n int` | 实际读了多少字节 |
| 返回 `err error` | 读取过程中有没有出错 |

为什么这个接口牛？因为它只有一个方法，所以实现简单、使用广泛——标准库里几百个函数都接受 Reader。

```go
// 这些类型都实现了 io.Reader
// - os.File          （文件）
// - strings.Reader   （字符串）
// - bytes.Buffer     （字节缓冲区）
// - net.Conn         （网络连接）

// 通用函数：不关心数据从哪来，只管读
func printAll(r io.Reader) {
    data := make([]byte, 100)
    n, _ := r.Read(data)
    fmt.Println(string(data[:n]))
}

// 从文件读
file, _ := os.Open("test.txt")
printAll(file)

// 从字符串读
str := strings.NewReader("hello")
printAll(str)
```

同一个函数，能处理不同类型的数据源——这就是接口的价值。

| 接口 | 方法数 | 作用 |
|------|--------|------|
| `io.Reader` | 1 | 能读 |
| `io.Writer` | 1 | 能写 |
| `io.Closer` | 1 | 能关闭 |
| `io.ReadWriter` | 2 | 能读能写（嵌入组合） |

Go 的设计哲学：把大能力拆成小接口，按需组合。

## 空接口 any 与类型断言

有时候需要处理"任意类型"，这时候用空接口：

```go
// any 是 interface{} 的别名
func printAnything(v any) {
    fmt.Println(v)
}
```

但是有个问题：拿到的是 `any`，不知道具体的类型怎么办？此时引出**类型断言**与 **type switch**。

### 类型断言

```go
func printAnything(v any) {
    if s, ok := v.(string); ok {
        fmt.Println("是字符串:", s)
    } else if n, ok := v.(int); ok {
        fmt.Println("是整数:", n)
    } else {
        fmt.Println("其他类型:", v)
    }
}
```

### Type Switch

更简洁的写法：

```go
func printAnything(v any) {
    switch val := v.(type) {
    case string:
        fmt.Println("是字符串:", val)
    case Student:
        fmt.Println("是学生:", val)
    }
}
```

`v.(type)` 是 Go 的特殊语法，**只能在 switch 里用**，意思是"告诉我 v 的实际类型是什么"。`val` 会自动变成对应分支的类型。

### Comma-Ok 模式详解

这是 Go 里处理"可能失败的操作"的通用模式，类型断言只是其中一个用法。

```go
s, ok := v.(string)
```

- 尝试把 `v` 转成 `string`
- 如果成功：`s` = 转换后的值，`ok` = `true`
- 如果失败：`s` = 空字符串零值，`ok` = `false`

关键点：如果不写 `ok`，断言失败会直接 **panic**（程序崩溃）。所以一般都用 `, ok` 安全地处理：

```go
// 不安全：失败会 panic
s := v.(string)

// 安全：失败只是 ok = false
s, ok := v.(string)
```

### 接口 vs Type Switch

| 方式 | 优点 | 缺点 |
|------|------|------|
| 接口 | 扩展性好，加新类型不用改函数 | 需要提前定义接口 |
| Type Switch | 直观，能处理任意类型 | 加新类型要改函数 |

## 错误处理

`strconv` 是 Go 标准库里专门处理字符串和数字之间转换的包。`strconv.Atoi` = ASCII to Integer。

Go 的错误处理模式：

```go
result, err := someFunction()
if err != nil {
    // 处理错误
    return ..., err  // 往上层传
}
// 正常使用 result
```

核心规则：

1. 错误是普通值，函数返回 `error` 接口
2. `nil` = 没错，非 `nil` = 有错（带错误信息）
3. 函数只返回错误，调用方决定怎么处理

### error 接口

`error` 是 Go 内置的接口，只有一个方法：

```go
type error interface {
    Error() string
}
```

任何实现了 `Error() string` 方法的类型，都算 `error`。

### 创建自己的错误

使用 `errors.New` 或 `fmt.Errorf`：

```go
import "errors"

func divide(a, b float64) (float64, error) {
    if b == 0 {
        return 0, errors.New("除数不能为零")
    }
    return a / b, nil
}
```

或者用 `fmt.Errorf` 带格式化：

```go
return 0, fmt.Errorf("除数不能为零：%f", b)
```

### 错误包装

有时候你想在错误上加点上下文，但是不想丢失原始错误：

```go
func readConfig(path string) (string, error) {
    data, err := os.ReadFile(path)
    if err != nil {
        // 包装原始错误，加上上下文
        return "", fmt.Errorf("读取配置文件失败：%w", err)
    }
    return string(data), nil
}
```

然后用 `errors.Is` 判断错误类型：

```go
content, err := readFile("不存在的文件.txt")
if err != nil {
    fmt.Println("错误:", err)
    if errors.Is(err, os.ErrNotExist) {
        fmt.Println("原因：文件不存在")
    }
}
```

### 设计理念：错误处理就像"层层汇报"

想象一个公司出了问题：

- **员工**：硬盘坏了（原始错误）
- **主管**：服务器出问题了（包装一层）
- **经理**：系统故障（再包装一层）

如果只传一句话：经理只知道"系统故障"，不知道根本原因，没法针对性解决。

如果层层包装但保留原始原因：经理收到 → 系统故障 → 服务器出问题 → 硬盘坏了。经理可以用 `errors.Is` 查问："是不是硬盘的问题？"——穿透所有包装，直接找到根本原因。

代码对应：

```go
// 底层：硬盘坏了
os.ReadFile("xxx")  // 返回 os.ErrNotExist（文件不存在）

// 中层：服务器出问题了
fmt.Errorf("读取文件失败: %w", err)  // 包装，但保留原始错误

// 顶层：经理
if errors.Is(err, os.ErrNotExist) {
    // 穿透包装，发现根本原因是"文件不存在"
    fmt.Println("原因：文件不存在")
}
```

三种方式对比：

```go
// 方式1：只传原始错误（不好）
return err  // main 只知道 "no such file or directory"，不知道是哪个文件

// 方式2：只传新错误（不好）
return errors.New("读取文件失败")  // main 不知道为什么失败

// 方式3：包装（正确）
return fmt.Errorf("读取文件失败: %w", err)  // main 既知道上下文，又能判断原因
```

### Sentinel Errors（哨兵错误）

就是预先定义好的错误值，让我们可以用 `errors.Is` 判断。`os.ErrNotExist` 就是一个 Sentinel Error。自己也可以定义：

```go
var ErrInvalidAge = errors.New("年龄不能为负数")

func setAge(age int) error {
    if age < 0 {
        return ErrInvalidAge
    }
    return nil
}

// 调用方
err := setAge(-1)
if errors.Is(err, ErrInvalidAge) {
    fmt.Println("年龄不合法")
}
```

## panic 与 recover

Go 中大部分用 `error` 处理错误，但有些情况下程序直接崩溃——`panic`。

- `panic`：程序崩溃
- `recover`：捕获 panic，让程序继续运行

### 什么时候用 panic

程序遇到无法继续运行的严重错误：

```go
func divide(a, b int) int {
    if b == 0 {
        panic("除数不能为零")  // 程序崩溃
    }
    return a / b
}
```

### 什么时候用 recover

在服务器程序里，你不想因为一个请求崩溃就让整个服务器挂掉：

```go
func safeDivide(a, b int) (result int, err error) {
    defer func() {
        if r := recover(); r != nil {
            err = fmt.Errorf("捕获 panic: %v", r)
        }
        // %v 是 Go 里最通用的格式化动词，意思是"用默认格式打印值"
    }()
    return divide(a, b), nil
}
```

几个要点：

1. **命名返回值**：`(result int, err error)` — Go 允许给返回值起名字，函数内部可以直接用这些名字，`return` 时可以省略返回值
2. **defer**：延迟执行，函数结束的时候才运行
3. **recover**：捕获 panic 的唯一方式，只能在 defer 里用

```go
// defer 里的匿名函数：定义后立即调用
defer func() {
    if r := recover(); r != nil {
        // r 是 panic 传出来的信息
        fmt.Println("捕获到 panic:", r)
    }
}()  // 定义完立刻调用
```

## 泛型（了解即可）

### 为什么需要泛型？

没有泛型时，同一个逻辑要写多份：

```go
func maxInt(a, b int) int {
    if a > b { return a }
    return b
}

func maxFloat(a, b float64) float64 {
    if a > b { return a }
    return b
}
```

逻辑完全一样，只是类型不同。泛型用一个函数搞定：

```go
func max[T int | float64 | string](a, b T) T {
    if a > b { return a }
    return b
}
```

### 泛型函数语法

```
func 函数名[类型参数 约束](参数) 返回类型
```

拆解 `func max[T int | float64 | string](a, b T) T`：

- `func max` → 函数名
- `[T int | float64 | string]` → T 可以是 int、float64 或 string
- `(a, b T)` → a 和 b 都是 T 类型
- `T` → 返回类型也是 T

调用时 Go 会自动推断 T 的类型：

```go
max(1, 2)           // T = int
max(1.5, 2.5)       // T = float64
max("hello", "world")  // T = string
```

### 类型约束

约束就是告诉编译器"T 必须是什么类型"。

常用约束：

- `[any]` → 任意类型
- `[comparable]` → 可以用 `==` 比较的类型
- `[constraints.Ordered]` → 可以用 `<` `>` 比较的类型（int、float64、string）

自定义约束：

```go
type Number interface {
    int | int32 | int64 | float32 | float64
}

func sum[T Number](nums []T) T {
    var total T
    for _, n := range nums {
        total += n
    }
    return total
}
```

### 泛型类型

泛型还能用在 struct 上：

```go
type Stack[T any] struct {
    items []T
}

func (s *Stack[T]) push(item T) {
    s.items = append(s.items, item)
}

func (s *Stack[T]) pop() T {
    item := s.items[len(s.items)-1]
    s.items = s.items[:len(s.items)-1]
    return item
}

// 使用
intStack := &Stack[int]{}
intStack.push(1)
intStack.push(2)
fmt.Println(intStack.pop())  // 2

strStack := &Stack[string]{}
strStack.push("hello")
fmt.Println(strStack.pop())  // hello
```

## 参考源

- raw/Go语言.md
