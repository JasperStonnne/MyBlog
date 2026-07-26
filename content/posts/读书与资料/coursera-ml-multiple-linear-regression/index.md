---
title: 多元线性回归与 NumPy 向量化
slug: coursera-ml-multiple-linear-regression
description: 从多特征模型、矩阵表示与向量化出发，推导多元线性回归的代价函数和梯度下降，并梳理 NumPy 广播、形状与性能要点
date: 2026-07-26T00:00:00+08:00
draft: false
image: cover.svg
tags:
  - 机器学习
  - 多元线性回归
  - NumPy
  - 向量化
  - 梯度下降
  - Coursera
  - 学习笔记
categories:
  - 技术笔记
---

## 1. 多元线性回归

### 1.1 从单特征到多特征

单变量线性回归只使用一个特征：

$$
f_{w,b}(x)=wx+b
$$

多元线性回归同时使用多个特征。例如预测房价时，可以使用：

- $x_0$：房屋面积
- $x_1$：卧室数量
- $x_2$：楼层数
- $x_3$：房屋年龄

若共有 $n$ 个特征，则一个样本写成：

$$
\mathbf{x}^{(i)}
=
\left[x_0^{(i)},x_1^{(i)},\ldots,x_{n-1}^{(i)}\right]
$$

其中：

- $i$ 表示第几个训练样本；
- $j$ 表示第几个特征；
- $x_j^{(i)}$ 表示第 $i$ 个样本的第 $j$ 个特征。

> 数学教材常从 1 开始编号；Python 和本实验从 0 开始编号。

### 1.2 模型

多元线性回归模型为：

$$
f_{\mathbf{w},b}(\mathbf{x})
=w_0x_0+w_1x_1+\cdots+w_{n-1}x_{n-1}+b
$$

使用向量点积可以简写为：

$$
\boxed{f_{\mathbf{w},b}(\mathbf{x})=\mathbf{w}\cdot\mathbf{x}+b}
$$

其中：

- $\mathbf{w}=[w_0,w_1,\ldots,w_{n-1}]$ 是长度为 $n$ 的参数向量；
- $b$ 是标量偏置；
- $\mathbf{w}\cdot\mathbf{x}=\sum_{j=0}^{n-1}w_jx_j$。

例如：

$$
f(\mathbf{x})=0.1x_0+4x_1+10x_2-2x_3+80
$$

如果房价单位为千美元，可解释为：面积每增加 1 平方英尺，预测价格增加 0.1 千美元；房龄每增加 1 年，预测价格减少 2 千美元。参数表达的是“其他特征保持不变时”的线性影响。

> “多元线性回归”指多个输入特征。它不同于统计学中具有多个输出变量的“多变量回归”。

---

## 2. 训练数据的矩阵表示

若有 $m$ 个训练样本，每个样本有 $n$ 个特征，则输入数据组成矩阵：

$$
\mathbf{X}\in\mathbb{R}^{m\times n}
$$

- 每一行是一个训练样本；
- 每一列是一种特征；
- `X[i, j]` 是第 `i` 个样本的第 `j` 个特征；
- `X[i]` 或 `X[i, :]` 是第 `i` 个样本的特征向量。

实验数据：

```python
import numpy as np

X_train = np.array([
    [2104, 5, 1, 45],
    [1416, 3, 2, 40],
    [ 852, 2, 1, 35]
])
y_train = np.array([460, 232, 178])
```

其形状为：

```python
X_train.shape  # (3, 4)：3 个样本、4 个特征
y_train.shape  # (3,)：3 个目标值
```

参数示例：

```python
w_init = np.array([
    0.39133535,
    18.75376741,
    -53.36032453,
    -26.42131618
])
b_init = 785.1811367994083

w_init.shape  # (4,)
```

---

## 3. 用 NumPy 实现预测

### 3.1 循环版本

```python
def predict_single_loop(x, w, b):
    p = 0.0
    for j in range(x.shape[0]):
        p += x[j] * w[j]
    return p + b
```

### 3.2 向量化版本

```python
def predict(x, w, b):
    return np.dot(x, w) + b
```

使用方式：

```python
x_vec = X_train[0]
prediction = predict(x_vec, w_init, b_init)
```

`x_vec` 和 `w_init` 的形状都是 `(4,)`，`np.dot(x_vec, w_init)` 返回一个标量。

向量化版本的优势：

- 代码更短、更容易阅读；
- 底层使用经过优化的数值计算程序；
- 可利用 CPU 的并行指令及专门的线性代数库；
- 对大规模数据通常远快于 Python 层的显式循环。

> 向量化并不意味着“所有运算一定在一个时钟周期完成”，而是将循环交给优化后的底层实现，从而减少 Python 解释器开销并更充分地利用硬件。

---

## 4. 代价函数

单个样本的误差为：

$$
e^{(i)}
=f_{\mathbf{w},b}(\mathbf{x}^{(i)})-y^{(i)}
$$

平方误差代价函数：

$$
\boxed{
J(\mathbf{w},b)
=\frac{1}{2m}
\sum_{i=0}^{m-1}
\left(
f_{\mathbf{w},b}(\mathbf{x}^{(i)})-y^{(i)}
\right)^2
}
$$

Python 实现：

```python
def compute_cost(X, y, w, b):
    m = X.shape[0]
    cost = 0.0

    for i in range(m):
        prediction = np.dot(X[i], w) + b
        cost += (prediction - y[i]) ** 2

    return cost / (2 * m)
```

进一步向量化：

```python
def compute_cost_vectorized(X, y, w, b):
    errors = X @ w + b - y
    return np.sum(errors ** 2) / (2 * X.shape[0])
```

其中 `@` 在这里执行矩阵与向量的乘法：

```text
X       : (m, n)
w       : (n,)
X @ w   : (m,)
y       : (m,)
```

---

## 5. 多元线性回归的梯度

对每个参数 $w_j$：

$$
\boxed{
\frac{\partial J}{\partial w_j}
=\frac{1}{m}
\sum_{i=0}^{m-1}
\left(
f_{\mathbf{w},b}(\mathbf{x}^{(i)})-y^{(i)}
\right)x_j^{(i)}
}
$$

对偏置 $b$：

$$
\boxed{
\frac{\partial J}{\partial b}
=\frac{1}{m}
\sum_{i=0}^{m-1}
\left(
f_{\mathbf{w},b}(\mathbf{x}^{(i)})-y^{(i)}
\right)
}
$$

循环实现：

```python
def compute_gradient(X, y, w, b):
    m, n = X.shape
    dj_dw = np.zeros(n)
    dj_db = 0.0

    for i in range(m):
        error = np.dot(X[i], w) + b - y[i]
        for j in range(n):
            dj_dw[j] += error * X[i, j]
        dj_db += error

    return dj_db / m, dj_dw / m
```

完全向量化实现：

```python
def compute_gradient_vectorized(X, y, w, b):
    m = X.shape[0]
    errors = X @ w + b - y
    dj_dw = X.T @ errors / m
    dj_db = np.sum(errors) / m
    return dj_db, dj_dw
```

形状检查：

```text
errors          : (m,)
X.T             : (n, m)
X.T @ errors    : (n,)
dj_dw           : (n,)
dj_db           : scalar
```

---

## 6. 批量梯度下降

更新规则：

$$
w_j
\leftarrow
w_j-\alpha\frac{\partial J}{\partial w_j}
$$

$$
b
\leftarrow
b-\alpha\frac{\partial J}{\partial b}
$$

向量形式：

$$
\boxed{
\mathbf{w}
\leftarrow
\mathbf{w}-\alpha\nabla_{\mathbf{w}}J
}
$$

实现：

```python
import copy
import math

def gradient_descent(
    X, y, w_in, b_in,
    cost_function, gradient_function,
    alpha, num_iters
):
    w = copy.deepcopy(w_in)
    b = b_in
    J_history = []

    for i in range(num_iters):
        dj_db, dj_dw = gradient_function(X, y, w, b)

        # 使用同一次迭代计算出的梯度，同时更新所有参数
        w = w - alpha * dj_dw
        b = b - alpha * dj_db

        if i < 100_000:
            J_history.append(cost_function(X, y, w, b))

        if i % math.ceil(num_iters / 10) == 0:
            print(f"Iteration {i:4d}: Cost {J_history[-1]:8.2f}")

    return w, b, J_history
```

实验设置：

```python
initial_w = np.zeros_like(w_init)
initial_b = 0.0
iterations = 1000
alpha = 5.0e-7

w_final, b_final, J_hist = gradient_descent(
    X_train,
    y_train,
    initial_w,
    initial_b,
    compute_cost,
    compute_gradient,
    alpha,
    iterations
)
```

该实验的学习率很小，而且不同特征的数值范围差异大，因此 1000 次迭代后的预测仍不够准确。后续通常会通过特征缩放改善收敛速度。

---

## 7. NumPy 向量基础

### 7.1 一维数组

本课程用一维 NumPy 数组表示向量：

```python
a = np.array([1, 2, 3, 4])
a.shape  # (4,)
```

常用创建方式：

```python
np.zeros(4)                 # 4 个 0
np.random.random_sample(4) # 4 个 [0, 1) 随机数
np.arange(4.0)             # [0., 1., 2., 3.]
np.random.rand(4)          # 4 个 [0, 1) 随机数
```

`(4,)` 表示含 4 个元素的一维数组，不是二维行矩阵 `(1, 4)`，也不是二维列矩阵 `(4, 1)`。

### 7.2 索引

```python
a = np.arange(10)

a[2]   # 第 3 个元素
a[-1]  # 最后一个元素
```

访问一维数组的单个元素通常得到 NumPy 标量。

### 7.3 切片

格式：

```python
a[start:stop:step]
```

`stop` 不包含在结果中。

```python
a[2:7:1]  # 索引 2、3、4、5、6
a[2:7:2]  # 索引 2、4、6
a[3:]     # 从索引 3 到末尾
a[:3]     # 索引 0、1、2
a[:]      # 全部元素，得到一个视图
```

### 7.4 常用向量运算

```python
a = np.array([1, 2, 3, 4])

-a          # 逐元素取负
np.sum(a)   # 元素求和，返回标量
np.mean(a)  # 均值，返回标量
a ** 2      # 逐元素平方
5 * a       # 每个元素乘以 5
```

### 7.5 逐元素运算与点积

```python
a = np.array([ 1, 2, 3, 4])
b = np.array([-1, 4, 3, 2])

a * b         # 逐元素乘法，结果仍是向量
np.dot(a, b)  # 点积：逐元素相乘后求和，结果是标量
a @ b         # 对两个一维向量，同样是点积
```

手动实现点积：

```python
def my_dot(a, b):
    result = 0.0
    for i in range(a.shape[0]):
        result += a[i] * b[i]
    return result
```

两个向量做逐元素运算或点积时，形状必须兼容。

### 7.6 向量性能示例

```python
import time

size = 10_000_000
a = np.random.rand(size)
b = np.random.rand(size)

start = time.time()
vectorized_result = np.dot(a, b)
vectorized_time = time.time() - start

start = time.time()
loop_result = my_dot(a, b)
loop_time = time.time() - start
```

两种方法应给出近似相同的结果。浮点数运算顺序不同，可能造成极小的舍入误差，因此比较时宜使用：

```python
np.isclose(vectorized_result, loop_result)
```

---

## 8. NumPy 矩阵基础

### 8.1 创建矩阵

二维数组使用形状元组：

```python
np.zeros((1, 5))             # shape: (1, 5)
np.zeros((2, 1))             # shape: (2, 1)
np.random.random_sample((3, 4))
```

手动创建：

```python
a = np.array([
    [5],
    [4],
    [3]
])

a.shape  # (3, 1)
```

### 8.2 reshape

```python
a = np.arange(6).reshape(3, 2)
```

结果：

```text
[[0, 1],
 [2, 3],
 [4, 5]]
```

也可以写：

```python
a = np.arange(6).reshape(-1, 2)
```

`-1` 表示让 NumPy 根据元素总数和另一个维度自动推断该维度。

### 8.3 矩阵索引

```python
a[2, 0]  # 第 3 行第 1 列的标量
a[2]     # 第 3 行，返回 shape 为 (n,) 的一维数组
a[2, :]  # 同上
a[:, 0]  # 第 1 列，返回一维数组
```

### 8.4 矩阵切片

```python
a = np.arange(20).reshape(-1, 10)

a[0, 2:7]  # 第 1 行中索引 2 到 6
a[:, 2:7]  # 所有行、索引 2 到 6 的列
a[:, :]    # 全部行和列
a[1, :]    # 第 2 行，结果是一维数组
```

---

## 9. 广播（Broadcasting）

广播允许 NumPy 在形状兼容时，将较小的数组“扩展”到较大的数组上执行逐元素运算。

最简单的例子是标量与向量：

```python
a = np.array([1, 2, 3, 4])
a + 10
# [11, 12, 13, 14]
```

在线性模型中：

```python
X @ w + b
```

`X @ w` 的形状是 `(m,)`，标量 `b` 会广播到每个预测值上。

判断广播是否兼容时，从形状的最后一个维度向前比较；对应维度相等，或其中一个为 1，通常即可广播。

> 广播很方便，但也可能隐藏形状错误。机器学习代码中应经常打印或断言 `.shape`。

---

## 10. 正态方程与梯度下降

线性回归还可以通过线性代数方法直接求解参数，常称为正态方程法。

特点：

- 无需像梯度下降那样反复迭代；
- 主要用于线性回归，不能自然推广到逻辑回归、神经网络等模型；
- 当特征数很大时，直接求解可能较慢；
- 实际项目中通常交给成熟的数值计算或机器学习库处理，不应手动计算矩阵逆。

本课程重点学习梯度下降，因为它能推广到更多机器学习模型。

---

## 11. 高频易错点

1. **数学下标与 Python 下标不同**

   数学讲解可能从 1 开始；NumPy 从 0 开始。

2. **`(n,)` 不等于 `(n, 1)`**

   前者是一维数组，后者是二维列矩阵。

3. **`a * b` 不等于 `np.dot(a, b)`**

   前者逐元素相乘，后者对一维向量返回点积。

4. **切片右端不包含**

   `a[2:7]` 包含索引 2 到 6。

5. **更新梯度下降参数时要同步**

   同一轮中的 `w` 和 `b` 都应使用更新前的同一组梯度。

6. **特征尺度会影响梯度下降速度**

   面积可能是上千，楼层数可能只有 1～3；尺度差异大会使代价函数狭长，导致收敛缓慢。

7. **学习率过大或过小都不好**

   过大可能使代价震荡或发散；过小则收敛很慢。

8. **浮点数结果不要直接用 `==` 比较**

   使用 `np.isclose` 或 `np.allclose`。

9. **课程原文中的翻译错误**

   向量 `[1,2,3,4]` 的 shape `(4,)` 是一维，不是“二维向量”；向量的元素个数常称为长度或维数，而 NumPy 数组的 `ndim` 表示轴的数量。

---

## 12. 一页速记

```python
# 预测
y_hat = X @ w + b

# 误差
errors = y_hat - y

# 代价
J = np.sum(errors ** 2) / (2 * m)

# 梯度
dj_dw = X.T @ errors / m
dj_db = np.sum(errors) / m

# 更新
w = w - alpha * dj_dw
b = b - alpha * dj_db
```

对应的核心公式：

$$
\boxed{
\hat{\mathbf{y}}=\mathbf{X}\mathbf{w}+b
}
$$

$$
\boxed{
J(\mathbf{w},b)=\frac{1}{2m}\|\hat{\mathbf{y}}-\mathbf{y}\|_2^2
}
$$

$$
\boxed{
\nabla_{\mathbf{w}}J
=\frac{1}{m}\mathbf{X}^{T}(\hat{\mathbf{y}}-\mathbf{y})
}
$$

$$
\boxed{
\frac{\partial J}{\partial b}
=\frac{1}{m}\sum_{i=0}^{m-1}(\hat y^{(i)}-y^{(i)})
}
$$

掌握这一条主线即可：

```text
样本矩阵 X
→ 点积得到预测
→ 预测减目标得到误差
→ 由误差计算代价和梯度
→ 用梯度下降更新 w、b
```
