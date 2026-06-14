---
title: CS61A Lab 练习理解
slug: cs61a-lab-notes
description: CS61A 课后 lab 习题的 Q&A 理解笔记，涵盖高阶函数、闭包、列表推导式、递归等
date: 2026-05-20T00:00:00+08:00
draft: false
tags:
  - CS61A
  - Python
  - 函数式编程
  - 递归
categories:
  - 读书与资料
---

## composite_identity

**原题**：判断对于某个 `x`，`f(g(x))` 和 `g(f(x))` 是否相等。

```python
def composite_identity(f, g):
    return lambda x: f(g(x)) == g(f(x))
```

- `lambda` 是 `def` 内部函数的简写，适合逻辑简单、一行能写完的情况。
- 第一次调用 `composite_identity(f, g)` 只是“打包”，返回函数本身；`b1(0)` 才真正执行计算。
- `return` 遇到就执行，和定义顺序无关；内层的 `return` 要等主动调用才会触发。

关联概念：高阶函数、闭包。

## count_cond

**原题**：返回一个函数，统计 `1` 到 `N` 中满足 `condition(N, i)` 的数字个数。

```python
def count_cond(condition):
    def count(n):
        total = 0
        for x in range(1, n + 1):
            if condition(n, x):
                total += 1
        return total
    return count
```

- `total = 0` 必须放在 `for` 循环外，否则每次循环都会重置。
- `condition` 被闭包捕获，外层函数返回后内层仍能访问。
- 这不是递归，外层返回的是另一个函数（闭包），不是调用自己。

关联概念：闭包、高阶函数。

## close & close_list

**原题**：统计列表中“值与下标之差的绝对值小于等于 `k`”的元素个数或列表。

```python
def close(s, k):
    count = 0
    for i in range(len(s)):
        if abs(s[i] - i) <= k:
            count += 1
    return count

def close_list(s, k):
    return [s[i] for i in range(len(s)) if abs(s[i] - i) <= k]
```

- 要用 `abs()`，因为差值可能为负。
- `close_list` 用列表推导式：`[放什么 for 怎么遍历 if 什么条件]`。

关联概念：列表推导式。

## cycle

**原题**：依次循环应用 `f1`、`f2`、`f3`，共应用 `n` 次。

```python
def cycle(f1, f2, f3):
    def gf(n):
        def gff(x):
            for i in range(1, n + 1):
                if i % 3 == 1:
                    x = f1(x)
                if i % 3 == 2:
                    x = f2(x)
                if i % 3 == 0:
                    x = f3(x)
            return x
        return gff
    return gf
```

- 用 `i % 3` 判断第 `i` 次用哪个函数。
- 循环变量不能叫 `x`，否则会覆盖传入参数。

关联概念：高阶函数、闭包。

## squares

**原题**：返回列表中所有完全平方数的平方根。

```python
def squares(s):
    return [int(n**0.5) for n in s if int(n**0.5) ** 2 == n]
```

完全平方数判断：`int(n**0.5) ** 2 == n`。`int()` 截断小数后平方回去，如果等于原数，就是完全平方数。

关联概念：列表推导式。

## double_eights

**原题**：判断 `n` 中是否存在连续两位都是 `8`，不能用循环。

```python
def double_eights(n):
    if n < 10:
        return False
    if n % 10 == 8 and (n // 10) % 10 == 8:
        return True
    return double_eights(n // 10)
```

- `n % 10`：最后一位。
- `(n // 10) % 10`：倒数第二位。
- 每次去掉最后一位，向 base case（`n < 10`）靠近。

关联概念：递归。

## make_onion

**原题**：从 `x` 出发，每次用 `f` 或 `g` 变换，`limit` 步内能否到达 `y`。

```python
def make_onion(f, g):
    def can_reach(x, y, limit):
        if limit < 0:
            return False
        elif x == y:
            return True
        else:
            return can_reach(f(x), y, limit - 1) or can_reach(g(x), y, limit - 1)
    return can_reach
```

- 每次只做一步，`or` 表示两条路任一可达即返回 `True`。
- 这里结合了闭包（`can_reach` 捕获 `f`、`g`）和递归。

关联概念：递归、闭包。
