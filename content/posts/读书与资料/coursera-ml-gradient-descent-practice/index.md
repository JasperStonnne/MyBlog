---
title: 梯度下降实践：从手写实现到 Scikit-Learn
slug: coursera-ml-gradient-descent-practice
description: 从手写代价函数与批量梯度下降出发，实践特征缩放、多项式回归与 Scikit-Learn，并总结收敛判断和常见调试方法
date: 2026-07-27T00:00:00+08:00
draft: false
image: cover.svg
tags:
  - 机器学习
  - 梯度下降
  - Python
  - Scikit-Learn
  - 特征缩放
  - Coursera
  - 学习笔记
categories:
  - 技术笔记
---

## 1. 学习目标

这组实验主要解决以下问题：

- 从零实现线性回归的代价函数和梯度；
- 使用批量梯度下降训练模型；
- 判断梯度下降是否正在收敛；
- 选择合适的学习率；
- 使用 Z-score 标准化加快收敛；
- 通过特征工程实现多项式回归；
- 使用 Scikit-Learn 完成标准化、训练与预测。

---

## 2. 梯度下降的核心流程

线性回归模型：

$$f_{\mathbf{w},b}(\mathbf{x}) = \mathbf{w}\cdot\mathbf{x}+b$$

整个训练过程可以概括为：

```
输入训练数据 X、y
        ↓
使用当前 w、b 计算预测
        ↓
预测值减去真实值得到误差
        ↓
根据全部误差计算梯度
        ↓
沿梯度反方向更新 w、b
        ↓
重复，直到代价不再明显下降
```

参数更新规则：

$$w_j \leftarrow w_j-\alpha\frac{\partial J}{\partial w_j}$$

$$b \leftarrow b-\alpha\frac{\partial J}{\partial b}$$

其中：

- $\alpha$：学习率，决定每一步走多远；
- 梯度：代价函数上升最快的方向；
- 减去梯度：向代价下降的方向移动。

---

## 3. 实践一：从零实现一元线性回归

实验使用城市人口预测餐厅利润：

- `x_train`：城市人口，单位为 10,000 人；
- `y_train`：餐厅月利润，单位为 10,000 美元；
- 数据集共有 97 个样本。

一元线性回归模型：

$$f_{w,b}(x^{(i)})=wx^{(i)}+b$$

### 3.1 实现代价函数

平方误差代价函数：

$$J(w,b) = \frac{1}{2m} \sum_{i=0}^{m-1} \left( f_{w,b}(x^{(i)})-y^{(i)} \right)^2$$

代码：

```python
def compute_cost(x, y, w, b):
    m = x.shape[0]
    total_cost = 0.0

    for i in range(m):
        prediction = w * x[i] + b
        error = prediction - y[i]
        total_cost += error ** 2

    return total_cost / (2 * m)
```

这里除以 $2m$：

- 除以 $m$：计算所有样本的平均损失；
- 除以 2：求导后平方项产生的 2 可以抵消，使梯度公式更简洁。

### 3.2 实现梯度

梯度公式：

$$\frac{\partial J}{\partial w} = \frac{1}{m} \sum_{i=0}^{m-1} \left( f_{w,b}(x^{(i)})-y^{(i)} \right)x^{(i)}$$

$$\frac{\partial J}{\partial b} = \frac{1}{m} \sum_{i=0}^{m-1} \left( f_{w,b}(x^{(i)})-y^{(i)} \right)$$

代码：

```python
def compute_gradient(x, y, w, b):
    m = x.shape[0]
    dj_dw = 0.0
    dj_db = 0.0

    for i in range(m):
        prediction = w * x[i] + b
        error = prediction - y[i]

        dj_dw += error * x[i]
        dj_db += error

    dj_dw /= m
    dj_db /= m

    return dj_dw, dj_db
```

`dj_dw` 多乘了一个 `x[i]`，因为 $w$ 是和输入 $x$ 相乘的参数；而 $b$ 只是直接相加，因此 `dj_db` 不需要乘 `x[i]`。

### 3.3 批量梯度下降

```python
import copy
import math

def gradient_descent(
    x, y,
    w_in, b_in,
    cost_function,
    gradient_function,
    alpha,
    num_iters
):
    w = copy.deepcopy(w_in)
    b = b_in
    J_history = []

    for i in range(num_iters):
        dj_dw, dj_db = gradient_function(x, y, w, b)

        # 必须使用同一次迭代计算出的梯度同步更新
        w = w - alpha * dj_dw
        b = b - alpha * dj_db

        if i < 100_000:
            J_history.append(cost_function(x, y, w, b))

        if i % math.ceil(num_iters / 10) == 0:
            print(
                f"Iteration {i:4d}: "
                f"Cost {J_history[-1]:8.2f}"
            )

    return w, b, J_history
```

实验参数：

```python
initial_w = 0.0
initial_b = 0.0
iterations = 1500
alpha = 0.01

w, b, J_history = gradient_descent(
    x_train,
    y_train,
    initial_w,
    initial_b,
    compute_cost,
    compute_gradient,
    alpha,
    iterations
)
```

实验结果大约为：

```
w = 1.1664
b = -3.6303
```

预测利润：

```python
profit_35k = (3.5 * w + b) * 10_000
profit_70k = (7.0 * w + b) * 10_000

print(profit_35k)  # 约 4519.77 美元
print(profit_70k)  # 约 45342.45 美元
```

这里的"批量"表示：

> 每轮使用全部 $m$ 个训练样本计算平均梯度，然后更新一次参数。

---

## 4. 多变量梯度下降

假设训练数据形状为：

```
X：(m, n)
y：(m,)
w：(n,)
b：标量
```

其中：

- $m$：训练样本数量；
- $n$：每个样本的特征数量。

预测：

```python
y_hat = X @ w + b
```

误差：

```python
errors = y_hat - y
```

向量化代价：

```python
def compute_cost(X, y, w, b):
    errors = X @ w + b - y
    return np.sum(errors ** 2) / (2 * X.shape[0])
```

向量化梯度：

```python
def compute_gradient(X, y, w, b):
    m = X.shape[0]
    errors = X @ w + b - y

    dj_dw = X.T @ errors / m
    dj_db = np.sum(errors) / m

    return dj_dw, dj_db
```

形状变化：

```
X.T             ：(n, m)
errors          ：(m,)
X.T @ errors    ：(n,)
dj_dw           ：(n,)
dj_db           ：标量
```

最终得到的 `dj_dw` 恰好包含 $n$ 个梯度，对应 $w$ 中的 $n$ 个参数。

---

## 5. 学习率实验

在未缩放的房价数据中，不同学习率会产生不同表现。

### $\alpha=9.9\times10^{-7}$

学习率过大：

- 参数更新越过最低点；
- 代价不断增加；
- 模型发散。

### $\alpha=9\times10^{-7}$

学习率稍小：

- 代价持续下降；
- 参数仍会在最低点两侧震荡；
- 最终能够收敛。

### $\alpha=1\times10^{-7}$

学习率更小：

- 参数较平稳地靠近最低点；
- 基本没有明显震荡；
- 但收敛速度更慢。

因此：

```
学习率太大 → 震荡或发散
学习率太小 → 稳定但很慢
学习率合适 → 快速且稳定下降
```

---

## 6. 为什么需要特征缩放？

房价数据包含以下特征：

```
面积：几百到几千
卧室：0 到 5
楼层：1 到 3
房龄：数十年
```

假设：

$$f(\mathbf{x})=w_1x_1+w_2x_2+b$$

如果 $x_1$ 是房屋面积，最大接近 2000；$x_2$ 是卧室数量，最大只有 5。

对于面积 2000、5 间卧室的房屋，一组合适的参数可能是：

$$w_1=0.1,\quad w_2=50,\quad b=50$$

预测为：

$$0.1\times2000+50\times5+50=500$$

单位是千美元，即 50 万美元。

可以看到：

- 数值范围大的特征，参数通常较小；
- 数值范围小的特征，参数通常较大。

这会让代价函数的等高线变得非常狭长：

```
未缩放：
梯度下降在狭长峡谷两侧反复震荡

缩放后：
等高线更接近圆形
梯度下降更直接地靠近最低点
```

特征缩放通常不会改变模型能够达到的最优结果，主要作用是让训练更快、更稳定。

---

## 7. 三种特征缩放方法

### 7.1 除以最大值

$$x_j' = \frac{x_j}{\max(x_j)}$$

例如：

```
面积范围：300～2000
缩放后：0.15～1

卧室范围：0～5
缩放后：0～1
```

### 7.2 均值归一化

$$x_j' = \frac{x_j-\mu_j} {\max(x_j)-\min(x_j)}$$

减去均值后，数据会以 0 为中心。

### 7.3 Z-score 标准化

$$x_j' = \frac{x_j-\mu_j}{\sigma_j}$$

其中：

$$\mu_j = \frac{1}{m} \sum_{i=0}^{m-1}x_j^{(i)}$$

$$\sigma_j = \sqrt{ \frac{1}{m} \sum_{i=0}^{m-1} \left(x_j^{(i)}-\mu_j\right)^2 }$$

处理后，每个特征通常具有：

- 均值接近 0；
- 标准差接近 1；
- 各特征具有相似的数值范围。

NumPy 实现：

```python
def zscore_normalize_features(X):
    mu = np.mean(X, axis=0)
    sigma = np.std(X, axis=0)
    X_norm = (X - mu) / sigma

    return X_norm, mu, sigma
```

使用：

```python
X_norm, X_mu, X_sigma = zscore_normalize_features(X_train)
```

实验中，未缩放数据只能使用约 `1e-7` 的学习率；经过 Z-score 标准化后，可以从 `0.1` 左右开始尝试，收敛速度明显加快。

---

## 8. 预测新数据时的关键规则

标准化训练数据后，必须保存训练集的均值和标准差。

预测面积 1200 平方英尺、3 间卧室、1 层、房龄 40 年的房屋：

```python
x_house = np.array([1200, 3, 1, 40])

x_house_norm = (
    x_house - X_mu
) / X_sigma

price = x_house_norm @ w_norm + b_norm
```

不能对新样本重新计算均值和标准差。

错误：

```python
# 不要这样做
new_mu = np.mean(x_house)
new_sigma = np.std(x_house)
```

正确原则：

```
训练集：计算并保存 mu、sigma
验证集：使用训练集的 mu、sigma
测试集：使用训练集的 mu、sigma
新数据：使用训练集的 mu、sigma
```

否则训练和预测使用的不是同一个坐标系。

---

## 9. 如何判断梯度下降是否收敛？

记录每次更新后的代价：

```python
J_history.append(
    compute_cost(X, y, w, b)
)
```

绘制学习曲线：

```python
plt.plot(J_history)
plt.xlabel("Iteration")
plt.ylabel("Cost J")
plt.show()
```

正常曲线：

```
J
│\
│ \
│  \____
│       ───
└──────────── 迭代次数
```

如果代价持续下降并逐渐变平，说明梯度下降正在收敛。

也可以使用自动收敛条件：

$$\left| J^{(t)}-J^{(t-1)} \right|<\epsilon$$

例如：

```python
if abs(J_history[-1] - J_history[-2]) < 1e-3:
    break
```

但实际学习中，直接观察学习曲线通常更直观。

---

## 10. 梯度下降调试方法

### 10.1 代价不断增加

可能原因：

- 学习率太大；
- 参数更新误写成了加号；
- 梯度公式实现错误。

错误：

```python
w = w + alpha * dj_dw
```

正确：

```python
w = w - alpha * dj_dw
b = b - alpha * dj_db
```

### 10.2 使用很小的学习率进行测试

把 `alpha` 设置得非常小。

如果代码正确，代价函数理论上应该缓慢下降。如果很小的学习率下代价仍然增加，通常说明代码存在错误。

需要注意：

> 很小的学习率适合调试，但不一定适合正式训练，因为它可能导致收敛非常慢。

### 10.3 尝试一系列学习率

可以按大约 3 倍的间隔尝试：

```
0.001
0.003
0.01
0.03
0.1
0.3
1
```

选择能够让代价快速、持续下降的较大学习率。

---

## 11. 特征工程

特征工程是利用业务知识，从已有特征构造更有意义的新特征。

例如：

- $x_1$：土地宽度；
- $x_2$：土地深度。

可以构造土地面积：

$$x_3=x_1x_2$$

代码：

```python
area = width * depth
```

模型变为：

$$f(\mathbf{x}) = w_1x_1+w_2x_2+w_3x_3+b$$

相比只使用宽度和深度，面积可能与房价具有更直接的关系。

---

## 12. 多项式回归

假设真实数据为：

$$y=1+x^2$$

如果只使用原始特征 $x$，线性回归只能拟合直线：

$$f(x)=wx+b$$

可以构造平方特征：

```python
x = np.arange(0, 20, 1)
y = 1 + x ** 2

X = (x ** 2).reshape(-1, 1)
```

模型变为：

$$f(x)=w_1x^2+b$$

虽然它对原始变量 $x$ 是非线性的，但对参数 $w_1,b$ 仍然是线性的，所以仍可以使用线性回归的训练方法。

不知道应该使用几次项时，可以构造多个候选特征：

```python
X = np.c_[
    x,
    x ** 2,
    x ** 3
]
```

对应模型：

$$f(x) = w_1x+w_2x^2+w_3x^3+b$$

### 多项式特征必须注意尺度

如果：

```
x：1～1000
```

那么：

```
x²：1～1,000,000
x³：1～1,000,000,000
```

尺度差异会非常大，因此应进行标准化：

```python
X = np.c_[x, x ** 2, x ** 3]
X_norm, mu, sigma = zscore_normalize_features(X)
```

特征缩放后，可以使用更大的学习率，训练速度会明显提高。

---

## 13. 使用 Scikit-Learn

Scikit-Learn 可以直接完成标准化和梯度下降回归。

```python
import numpy as np

from sklearn.linear_model import SGDRegressor
from sklearn.preprocessing import StandardScaler
```

### 13.1 标准化训练数据

```python
scaler = StandardScaler()
X_norm = scaler.fit_transform(X_train)
```

`fit_transform()` 包含两步：

```
fit       → 从训练集计算均值和标准差
transform → 使用这些统计量转换数据
```

### 13.2 创建并训练模型

```python
sgdr = SGDRegressor(
    max_iter=1000,
    random_state=0
)

sgdr.fit(X_norm, y_train)
```

查看训练信息：

```python
print(sgdr.n_iter_)  # 实际迭代轮数
print(sgdr.t_)       # 权重更新次数
```

查看参数：

```python
w_norm = sgdr.coef_
b_norm = sgdr.intercept_

print(w_norm)
print(b_norm)
```

这些参数对应的是标准化后的输入数据，不能直接按原始面积、卧室数量的单位解释。

### 13.3 进行预测

```python
y_pred_sgd = sgdr.predict(X_norm)
```

也可以手动计算：

```python
y_pred_manual = X_norm @ w_norm + b_norm
```

检查两种结果：

```python
np.allclose(
    y_pred_sgd,
    y_pred_manual
)
```

不要使用：

```python
(y_pred_sgd == y_pred_manual).all()
```

浮点数运算可能存在很小的舍入误差，应使用 `np.allclose()`。

### 13.4 预测新房屋

```python
x_house = np.array([
    [1200, 3, 1, 40]
])

x_house_norm = scaler.transform(x_house)
price = sgdr.predict(x_house_norm)

print(price)
```

对新数据只能调用：

```python
scaler.transform()
```

不能再次调用：

```python
scaler.fit_transform()
```

因为重新 `fit` 会改变标准化使用的均值和标准差。

### 13.5 SGD 与批量梯度下降的区别

手写实验使用的是批量梯度下降：

```
使用全部训练样本
→ 计算一次平均梯度
→ 更新一次参数
```

`SGDRegressor` 使用随机梯度下降：

```
依次使用单个样本或小批次
→ 更频繁地更新参数
```

两者都在尝试降低代价函数，但参数更新方式不同。

---

## 14. Notebook 中的 NameError

实验中出现：

```
NameError: name 'plt' is not defined
NameError: name 'load_house_data' is not defined
NameError: name 'run_gradient_descent' is not defined
```

这通常不是模型或公式的问题，而是依赖单元格没有运行。

Notebook 中的变量只存在于当前内核的内存里。重启内核、断线或者跳过前面的单元格，都会导致变量不存在。

正确运行顺序：

```
1. 运行 import 单元格
2. 运行辅助函数单元格
3. 加载训练数据
4. 标准化数据
5. 训练模型
6. 计算预测
7. 绘制图像
```

如果 Notebook 状态混乱，可以使用：

```
Restart Kernel and Run All
```

---

## 15. 高频易错点

1. **`w` 和 `b` 必须同步更新**
   同一轮参数更新必须使用更新前计算出的同一组梯度。

2. **更新公式使用减号**
   梯度指向代价上升方向，因此需要减去梯度。

3. **特征尺度差异会拖慢收敛**
   面积、卧室数量、楼层、房龄应进行缩放。

4. **预测新数据必须复用训练集统计量**
   不能为新样本重新计算均值和标准差。

5. **多项式特征更需要缩放**
   $x,x^2,x^3$ 的数值范围可能相差多个数量级。

6. **浮点数不要直接使用 `==` 比较**
   使用 `np.isclose()` 或 `np.allclose()`。

7. **权重较大不一定代表特征更重要**
   权重大小受到特征尺度影响，也不代表因果关系。

8. **SGDRegressor 不是批量梯度下降**
   它使用随机梯度下降，只是优化目标与线性回归相似。

---

## 16. 一页实践模板

```python
import numpy as np
from sklearn.preprocessing import StandardScaler
from sklearn.linear_model import SGDRegressor

# 1. 准备数据
X_train, y_train = load_house_data()

# 2. 标准化
scaler = StandardScaler()
X_norm = scaler.fit_transform(X_train)

# 3. 训练
model = SGDRegressor(
    max_iter=1000,
    random_state=0
)
model.fit(X_norm, y_train)

# 4. 训练集预测
y_pred = model.predict(X_norm)

# 5. 新数据预测
x_new = np.array([
    [1200, 3, 1, 40]
])
x_new_norm = scaler.transform(x_new)
prediction = model.predict(x_new_norm)

# 6. 检查手动计算
manual_prediction = (
    X_norm @ model.coef_
    + model.intercept_
)

print(np.allclose(
    y_pred,
    manual_prediction
))
```

整组实验最重要的主线是：

```
准备训练数据
→ 检查特征尺度
→ 使用训练集统计量标准化
→ 选择学习率
→ 运行梯度下降
→ 观察代价是否持续下降
→ 保存参数和标准化器
→ 用相同的转换处理新数据
→ 进行预测
```

一句话总结：

> 梯度下降负责寻找参数，特征缩放负责让寻找过程更快、更稳定，学习曲线负责判断训练是否正常，特征工程负责让线性模型表达更复杂的关系。
