---
title: 机器学习基础：用梯度下降法训练模型
slug: coursera-ml-gradient-descent
description: 从导数、学习率与同步更新出发，推导一元线性回归的梯度公式，并用 Python 完成训练、收敛检查与预测
date: 2026-07-25T00:00:00+08:00
draft: false
image: cover.svg
tags:
  - 机器学习
  - 梯度下降
  - 线性回归
  - 学习率
  - Coursera
  - 学习笔记
categories:
  - 技术笔记
---

## 一、学习目标

1. 梯度下降如何更新参数 $w$ 和 $b$？
2. 导数项为什么能指引下降方向？
3. 学习率 $\alpha$ 应该如何理解和选择？
4. 如何用梯度下降训练一元线性回归模型？

---

## 二、梯度下降算法

梯度下降通过不断调整模型参数，使成本函数 $J(w,b)$ 尽可能小。

参数更新公式为：

$$w := w - \alpha \frac{\partial J(w,b)}{\partial w}$$

$$b := b - \alpha \frac{\partial J(w,b)}{\partial b}$$

不断重复以上更新，直到算法收敛。

### 符号含义

| 符号 | 含义 |
|------|------|
| $w, b$ | 模型参数 |
| $J(w,b)$ | 成本函数，衡量预测误差 |
| $\alpha$ | 学习率，控制每次更新的步长 |
| $\frac{\partial J}{\partial w}$ | 成本函数相对于 $w$ 的偏导数 |
| $\frac{\partial J}{\partial b}$ | 成本函数相对于 $b$ 的偏导数 |
| $:=$ | 赋值，用右侧结果更新左侧变量 |

### 赋值与相等的区别

$w := w - \alpha \frac{\partial J}{\partial w}$ 表示“计算右边的结果，然后把结果存入 $w$”，而不是断言左右两边在数学上永远相等。

在 Python 中，`=` 是赋值，`==` 是相等性判断。

---

## 三、为什么必须同时更新 $w$ 和 $b$

梯度下降要求两个参数基于同一组旧参数进行更新。

**正确做法**：

```python
temp_w = w - alpha * dj_dw    # dj_dw 和 dj_db 都根据更新前的 w, b 计算
temp_b = b - alpha * dj_db

w = temp_w
b = temp_b
```

**不正确的顺序更新**：

```python
temp_w = w - alpha * dj_dw
w = temp_w                     # w 已被更新

temp_b = b - alpha * dj_db     # 此时计算用的是新的 w，不是旧的
b = temp_b
```

问题在于：计算新 $b$ 时，$w$ 已经被更新，两个参数不再基于同一个旧状态进行计算。标准梯度下降所指的是**同步更新**。

---

## 四、导数项的直观含义

为了理解导数，暂时只考虑一个参数：

$$w := w - \alpha \frac{dJ(w)}{dw}$$

导数表示成本曲线在当前位置的斜率。

### 导数为正

$$\frac{dJ}{dw} > 0 \implies w_{\text{new}} = w - \text{正数}$$

$w$ 变小，在图像上向左移动。当位于最低点右侧时，向左移动能够降低成本。

### 导数为负

$$\frac{dJ}{dw} < 0 \implies w_{\text{new}} = w - \alpha(\text{负数}) = w + \text{正数}$$

$w$ 变大，在图像上向右移动。当位于最低点左侧时，向右移动能够降低成本。

### 导数为零

在局部最小值处，$\frac{dJ}{dw} = 0$，于是 $w_{\text{new}} = w$，参数保持不变。梯度下降到达最低点后会自然停止移动。

### 核心直觉

梯度下降之所以使用“减去导数”，是因为导数指向函数增长最快的方向，而导数的反方向能够降低成本。

---

## 五、学习率 $\alpha$

学习率控制每次参数更新的步长，通常取较小的正数，例如 $\alpha = 0.01$。

### 学习率太小

- 每次更新的步长很小
- 成本通常仍会下降，但需要大量迭代
- 训练速度非常慢

### 学习率太大

- 参数可能跨过最低点
- 成本可能反而增加
- 参数可能在最低点两侧来回震荡
- 算法可能无法收敛，甚至发散

### 为什么固定学习率也能收敛

当参数逐渐接近最低点时，成本函数的斜率会越来越小：$\left|\frac{dJ}{dw}\right| \to 0$

即使学习率 $\alpha$ 保持不变，实际更新量 $\alpha \frac{dJ}{dw}$ 也会自然变小。因此梯度下降通常表现为：

1. 离最低点较远时步长较大
2. 接近最低点时步长逐渐减小
3. 最终停留在最低点附近

---

## 六、一元线性回归模型

模型为：

$$f_{w,b}(x) = wx + b$$

其中 $x$ 是输入特征（如房屋面积），$f_{w,b}(x)$ 是预测值（如预测房价），$w$ 是斜率，$b$ 是截距。

---

## 七、平方误差成本函数

使用 $m$ 个训练样本时，成本函数为：

$$J(w,b) = \frac{1}{2m} \sum_{i=1}^{m} \left(f_{w,b}(x^{(i)}) - y^{(i)}\right)^2$$

其中 $x^{(i)}$ 是第 $i$ 个样本的输入，$y^{(i)}$ 是真实值，$m$ 是训练样本数量。

前面的 $\frac{1}{2}$ 是为了在求导时抵消平方项产生的系数 2，使梯度公式更简洁。

---

## 八、线性回归的梯度公式

成本函数相对于 $w$ 的偏导数：

$$\frac{\partial J(w,b)}{\partial w} = \frac{1}{m} \sum_{i=1}^{m} \left(f_{w,b}(x^{(i)}) - y^{(i)}\right) x^{(i)}$$

成本函数相对于 $b$ 的偏导数：

$$\frac{\partial J(w,b)}{\partial b} = \frac{1}{m} \sum_{i=1}^{m} \left(f_{w,b}(x^{(i)}) - y^{(i)}\right)$$

两者的区别：$w$ 的导数末尾需要乘 $x^{(i)}$，$b$ 的导数不需要。

---

## 九、完整的梯度下降算法

每轮迭代先计算：

$$d_w = \frac{1}{m} \sum_{i=1}^{m} \left(wx^{(i)} + b - y^{(i)}\right) x^{(i)}$$

$$d_b = \frac{1}{m} \sum_{i=1}^{m} \left(wx^{(i)} + b - y^{(i)}\right)$$

然后同步更新：

$$w := w - \alpha \, d_w$$

$$b := b - \alpha \, d_b$$

重复以上过程，直到成本不再明显下降，或者参数变化已经非常小。

---

## 十、收敛、局部最小值与全局最小值

### 收敛

梯度下降“收敛”通常表示：成本函数不再明显下降，$w$ 和 $b$ 的变化越来越小，参数已经接近某个最小值。

### 局部最小值 vs 全局最小值

- **局部最小值**：某个点比附近其他点都低，但不一定是整个函数的最低点
- **全局最小值**：在所有可能的参数取值中，成本函数值最低的点

### 线性回归的优势

线性回归的平方误差成本函数是**凸函数**（碗形函数），具有以下性质：

- 只有一个全局最小值
- 不存在多个不同的局部最小值
- 只要学习率选择合理，梯度下降就能收敛到全局最小值

---

## 十一、批量梯度下降

上述算法属于**批量梯度下降（Batch Gradient Descent）**。“批量”表示每次更新参数时都使用全部 $m$ 个训练样本（$\sum_{i=1}^{m}$），即每轮更新都会遍历整个训练集。

其他梯度下降方法可能每次只使用一个训练样本或一小批训练样本，但当前的一元线性回归使用的是完整训练集。

---

## 十二、Python 实现

### 计算梯度

```python
def compute_gradient(x, y, w, b):
    m = x.shape[0]
    dj_dw = 0
    dj_db = 0

    for i in range(m):
        f_wb = w * x[i] + b        # 预测值
        error = f_wb - y[i]         # 预测误差

        dj_dw += error * x[i]       # w 的梯度需要乘 x[i]
        dj_db += error              # b 的梯度不需要乘 x[i]

    dj_dw /= m
    dj_db /= m

    return dj_dw, dj_db
```

### 计算成本

```python
def compute_cost(x, y, w, b):
    m = x.shape[0]
    cost = 0

    for i in range(m):
        f_wb = w * x[i] + b
        cost += (f_wb - y[i]) ** 2

    return cost / (2 * m)
```

### 梯度下降主函数

```python
def gradient_descent(x, y, w_in, b_in, alpha, num_iters,
                     cost_function, gradient_function):
    J_history = []
    p_history = []

    w = w_in
    b = b_in

    for i in range(num_iters):
        # 先计算两个梯度（基于当前的 w, b）
        dj_dw, dj_db = gradient_function(x, y, w, b)

        # 同步更新参数
        w = w - alpha * dj_dw
        b = b - alpha * dj_db

        # 保存代价和参数（用于可视化）
        if i < 100000:
            J_history.append(cost_function(x, y, w, b))
            p_history.append([w, b])

        # 定期输出训练过程
        if i % math.ceil(num_iters / 10) == 0:
            print(f"Iteration {i:4}: Cost {J_history[-1]:0.2e}  "
                  f"w: {w:0.3e}, b: {b:0.5e}")

    return w, b, J_history, p_history
```

返回值：训练后的 `w`、`b`，每次迭代的代价 `J_history`，每次迭代的参数 `p_history`。

---

## 十三、运行实验

### 训练数据

```python
x_train = np.array([1.0, 2.0])       # 房屋面积（千平方英尺）
y_train = np.array([300.0, 500.0])    # 房屋售价（千美元）
```

### 训练参数

```python
w_init = 0
b_init = 0
iterations = 10000
alpha = 1.0e-2
```

### 运行

```python
w_final, b_final, J_hist, p_hist = gradient_descent(
    x_train, y_train, w_init, b_init, alpha, iterations,
    compute_cost, compute_gradient
)
```

最终结果约为 $w \approx 199.99$，$b \approx 100.01$，即模型为 $f(x) = 200x + 100$。

### 梯度下降的收敛特点

成功运行时可以观察到：

1. 代价在开始阶段迅速下降
2. `dj_dw` 和 `dj_db` 的绝对值逐渐减小
3. 越接近最低点，梯度越小
4. 梯度变小后，参数更新速度也会变慢
5. 代价应持续下降并逐渐稳定

虽然学习率 $\alpha$ 保持不变，但实际更新量 = $\alpha \times$ 梯度，梯度变小时更新幅度自然变小。

---

## 十四、可视化

### 代价变化曲线

```python
fig, (ax1, ax2) = plt.subplots(1, 2, constrained_layout=True, figsize=(12, 4))

ax1.plot(J_hist[:100])           # 训练初期：代价下降很快
ax2.plot(1000 + np.arange(len(J_hist[1000:])), J_hist[1000:])  # 训练后期：下降较慢

ax1.set_title("Cost vs. iteration (start)")
ax2.set_title("Cost vs. iteration (end)")
ax1.set_ylabel("Cost")
ax2.set_ylabel("Cost")
ax1.set_xlabel("Iteration step")
ax2.set_xlabel("Iteration step")
plt.show()
```

分开绘制是因为训练初期和后期下降速度差异大，用不同范围可以更清楚地观察变化。

### 梯度下降路径（等高线图）

```python
fig, ax = plt.subplots(1, 1, figsize=(12, 6))
plt_contour_wgrad(x_train, y_train, p_hist, ax)
plt.show()
```

等高线代表不同的代价值，箭头代表参数 $(w, b)$ 的更新路径。可以观察到：参数不断向最低点移动，开始时梯度大步幅也大，接近最低点时梯度变小步幅缩短。

---

## 十五、模型预测

训练完成后，使用 $\hat{y} = w_{\text{final}} x + b_{\text{final}}$ 进行预测：

```python
print(f"1000 sqft: {w_final * 1.0 + b_final:.1f} Thousand dollars")
print(f"1200 sqft: {w_final * 1.2 + b_final:.1f} Thousand dollars")
print(f"2000 sqft: {w_final * 2.0 + b_final:.1f} Thousand dollars")
```

| 房屋面积 | 预测售价 |
|----------|----------|
| 1000 平方英尺 | 300 千美元 |
| 1200 平方英尺 | 340 千美元 |
| 2000 平方英尺 | 500 千美元 |

虽然 1200 平方英尺不在训练数据中，模型仍可以根据学到的线性关系进行预测。

---

## 十六、学习率的影响

### 合适的学习率

- 代价持续下降，参数逐渐靠近最优值，算法最终收敛

### 学习率过小

- 更新步幅很小，算法能够稳定收敛，但训练速度较慢

### 学习率过大

- 参数可能跨过最低点，$w$ 和 $b$ 在正负之间振荡
- 梯度符号反复变化，参数绝对值越来越大
- 代价不断上升，算法最终发散

判断学习率过大的典型信号：

- 代价不降反升
- 参数绝对值越来越大
- 梯度不断改变符号
- 参数在最低点两侧大幅振荡

例如将学习率改为 0.8，只跑 10 次迭代就能观察到发散过程。

---

## 十七、常见错误与检查方法

1. **没有同步更新参数**：应先计算新 $w$ 和新 $b$，再统一赋值
2. **学习率过小**：成本下降但极其缓慢 → 适当增大 $\alpha$
3. **学习率过大**：成本上下震荡或越来越大 → 减小 $\alpha$
4. **导数公式漏乘 $x^{(i)}$**：计算 $w$ 的梯度时必须包含 $\text{error} \times x^{(i)}$，$b$ 的梯度只有误差项
5. **没有观察成本变化**：训练过程中应定期计算 $J(w,b)$，正常情况下成本应总体持续下降

---

## 十八、核心知识总结

1. 梯度下降的目标是通过调整参数来最小化成本函数
2. 学习率 $\alpha$ 控制每次更新的步长
3. 导数的符号决定参数移动方向，大小影响移动幅度
4. $w$ 和 $b$ 必须基于同一组旧值同步更新
5. 学习率太小导致收敛缓慢，太大导致震荡或发散
6. 接近最低点时导数自然变小，更新步长也会变小
7. 线性回归的平方误差成本函数是凸函数，只有一个全局最小值
8. 批量梯度下降每次更新都会使用全部训练样本
9. 训练完成后，使用 $f_{w,b}(x) = wx + b$ 对新数据进行预测
