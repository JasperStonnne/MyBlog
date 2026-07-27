---
title: AI Infra 讲座：从并行计算到 CUDA、Triton 与 TileLang
slug: ai-infra-parallel-computing-cuda-triton-tilelang
description: 未名 HPC Training Camp Session 1 学习笔记，涵盖并行计算基础、CUDA 编程模型、Triton 与 TileLang 的核心概念
date: 2026-07-25T00:00:00+08:00
draft: false
image: cover.svg
tags:
  - AI Infra
  - GPU
  - CUDA
  - Triton
  - TileLang
  - 并行计算
categories:
  - 技术笔记
---

> 面向刚进入 Infra / AI Infra 领域的初学者
> 课程时间：2026-07-25
> 整理依据：96 页课件 + 约 3 小时逐字稿
> 说明：本文不是逐页复述，而是按知识依赖关系重构。逐字稿中的语音识别错误（如把 warp 写成 work、grid 写成 grade、Triton 写成 ten、TileLang 写成 tell long）均已按上下文纠正。

---

## 0. 先给出整场讲座的主线

这场讲座其实在回答一个连续的问题：

1. **为什么 AI 需要并行计算？**
   大模型中的矩阵乘法、逐元素运算、通信和流水线，都含有大量可以同时完成的工作。
2. **为什么 GPU 适合做这些计算？**
   GPU 用大量计算单元和高带宽内存换取高吞吐量，并通过并发执行许多 warp 来隐藏单个任务的延迟。
3. **程序员怎样把计算映射到 GPU？**
   CUDA 用 `grid → block/CTA → warp → thread` 描述并行任务，并要求程序员显式处理索引、内存、同步和边界。
4. **为什么还需要 Triton / TileLang？**
   手写每个线程过于繁琐。Tile-level DSL 让程序员先描述"一块数据如何搬运和计算"，再由编译器完成更细的线程映射。
5. **这与 AI Infra 有什么关系？**
   算子、编译器、分布式训练、集群通信、推理服务和强化学习系统，本质上都在解决：
   **怎样切分计算、放置数据、协调执行，并让昂贵硬件尽可能持续地做有效工作。**

可以把全文压缩成一句话：

> AI Infra 的核心不是"GPU 比 CPU 快"，而是识别并行性，并在计算、内存和通信之间设计高效的数据流。

---

## 1. 活动定位与课程地图

本次暑期活动的目标是"学习和实践 AI Infra"，除课程外还计划提供算力、练习、评测排行、交流讨论和业界连接。整体内容分为四个主题：

| 主题 | 主要内容 |
|---|---|
| Kernel 与 ML Compiler | CUDA、GPU 编程、DSL、数据布局、流水线、编译器栈、性能上限分析 |
| Interconnect 与 Communication | Scale-up / Scale-out、网络互联、P2P、集合通信、MoE/EP、通信计算重叠、存储协同 |
| LLM Serving 与 Inference | KV Cache、vLLM/SGLang、算子与框架、并行策略、Prefill-Decode 分离、投机解码 |
| Distributed RL Systems | RL 算法、Rollout、训推一致性、分布式 RL、Agentic RL |

Session 1 是后续所有主题的"地基层"：先建立并行计算、GPU 执行模型和 Kernel 编程的共同语言。

---

# 第一部分：从计算到并行计算

## 2. 什么是计算

课程给出的朴素定义是：

> 计算是把输入值按照某种规则变换为输出值的过程。

这个定义可以拆成两件事：

- **数据及其搬运**：输入在哪里？输出写到哪里？中间数据怎样流动？
- **计算规则及其执行单元**：加法、乘法、比较、矩阵乘法由什么硬件完成？

这是一个很重要的 Infra 视角。应用开发常把"算法"当作核心，但性能工程会同时追问：

- 算了多少次？
- 每次计算需要读写多少字节？
- 数据位于寄存器、缓存、显存，还是另一张卡？
- 计算单元是在工作，还是在等数据？

很多程序"算得慢"并不是算术运算慢，而是数据没有及时到达计算单元。

## 3. 指令、时钟周期、流水线与 IPC

一个经典处理器执行指令大致经过：

`取指 → 译码 → 执行 → 访存 → 写回`

如果把每条指令完整做完才开始下一条，硬件的大量部件会闲置。流水线让不同指令同时处于不同阶段。例如：

- 指令 1 正在写回；
- 指令 2 正在访存；
- 指令 3 正在执行；
- 指令 4 正在译码；
- 指令 5 正在取指。

**IPC（Instructions Per Cycle）** 表示平均每周期完成多少条指令。需要注意：

- IPC 不是频率；频率表示一秒有多少周期。
- IPC 也不是程序性能的全部；不同指令的工作量、延迟和吞吐不同。
- 超标量处理器能在同一周期发射或完成多条互不依赖的指令，因此 IPC 可能大于 1。

这已经是一种并行：它发生在单个处理器内部，称为**指令级并行（ILP）**。

## 4. 延迟与吞吐量

这是全场最重要的一组区分。

### 延迟（Latency）

完成一个任务要等多久。

例：一个请求从进入推理服务到收到第一个 token 的时间。

### 吞吐量（Throughput）

单位时间完成多少任务或多少计算。

例：一个推理集群每秒处理多少 token。

GPU 的优势主要是吞吐量，而不是每个单独运算的最低延迟。它的策略是：

> 当一个任务在等待数据时，迅速执行另一个已就绪的任务，用大量并发工作隐藏等待。

因此，"GPU 快"必须补全为：

> 对具有足够并行度、规则控制流和合适访存模式的工作负载，GPU 往往能提供很高的总吞吐量。

## 5. 什么样的计算能够并行

### 5.1 数据依赖

```text
c = a * b
d = 3 * c
```

`d` 必须等待 `c`，二者存在真数据依赖，不能直接同时执行。

```text
c = a * b
d = 3 * b
e = a + b
```

三条计算只读相同输入、写不同输出，可以并行。

更系统地说，判断能否并行要看读写集合是否冲突。常见依赖包括：

- **RAW（Read After Write）**：后者要读前者写出的值，是真依赖。
- **WAR（Write After Read）**：后者写入的位置仍需被前者读取。
- **WAW（Write After Write）**：两个任务写同一位置，最终结果依赖顺序。

### 5.2 竞争、互斥和锁

多个执行单元同时读写同一资源，可能产生 race condition。锁、原子操作、屏障可以恢复正确性，但会带来等待和串行化。

所以，并行化不只是"把任务分成 N 份"，还要保证：

- 每份任务的读写范围清楚；
- 共享状态受到正确保护；
- 同步频率不过高；
- 结果与执行顺序无关，或执行顺序被明确控制。

### 5.3 通信与同步

任务即使能拆开，也常常需要交换中间结果。例如：

- 多卡训练中聚合梯度；
- 求所有设备上的最大值或总和；
- 流水线后一阶段等待前一阶段输出；
- MoE 中把 token 路由到不同专家。

并行收益最终要减去：

- 通信时间；
- 同步等待；
- 数据重新布局；
- 任务调度；
- 负载不均衡。

这解释了为什么"设备数翻倍"通常不等于"性能翻倍"。

## 6. 两类基本拆分方法

### 6.1 Domain Decomposition：按数据域拆分

把一个大数据集切成若干块，让不同计算单元处理不同区域。

常见切法：

- 连续分块（block）；
- 循环分配（cyclic）；
- block-cyclic；
- 二维或三维网格切分。

选择切法时要考虑：

- 各块计算量是否均衡；
- 相邻数据是否频繁交互；
- 数据是否连续，能否高效访问；
- 每块能否放入更快的存储层。

矩阵乘法是典型例子。输出矩阵 `C = A × B` 的不同块可以独立计算，因此可以先按 `M`、`N` 维切分输出，再沿归约维 `K` 分块累加。

### 6.2 Functional Decomposition：按功能或阶段拆分

把不同类型的计算放到不同执行单元，或让不同设备负责流水线的不同阶段。

例：

- CPU 负责调度和控制，GPU 负责密集计算；
- 流水线并行中，不同 GPU 负责模型的不同层；
- 推理系统中，Prefill 和 Decode 由不同资源池承担。

### 6.3 大模型训练中的典型映射

| 并行方式 | 拆什么 | 主要代价 |
|---|---|---|
| 数据并行 DP | 不同设备处理不同 batch，模型副本相同 | 梯度 All-Reduce |
| 张量并行 TP | 把单层矩阵按行或列拆到多卡 | 层内频繁通信 |
| 流水线并行 PP | 不同设备负责不同层 | 流水线气泡、激活传输 |
| 专家并行 EP | 不同设备放不同 MoE 专家 | All-to-All、负载不均 |
| 序列/上下文并行 | 按序列维切分 | 注意力相关通信 |

现实系统经常组合多种并行方式，形成多维并行拓扑。

## 7. Flynn 分类：从指令流与数据流看硬件

| 类型 | 含义 | 直觉 |
|---|---|---|
| SISD | 单指令流、单数据流 | 经典串行处理 |
| SIMD | 单指令流、多数据流 | 同一指令同时作用于多个数据元素 |
| MISD | 多指令流、单数据流 | 很少有严格对应的通用硬件 |
| MIMD | 多指令流、多数据流 | 多核 CPU、集群等通用并行系统 |

这个分类是抽象模型，不应把复杂现代处理器硬塞进唯一格子。GPU 在不同层级上会同时呈现不同特征：

- 不同 block 可以执行不同任务，整体具有 MIMD 味道；
- 单个 warp 通常以相同指令推进，具有 SIMD/SIMT 味道。

## 8. 并行编程模型：程序员怎样描述计算

课程提到的主要模型包括：

- **共享内存**：执行单元通过同一地址空间读写数据。
- **线程模型**：创建、同步和管理多个线程。
- **分布式内存 / 消息传递**：不同进程拥有独立内存，通过 send/receive、broadcast、reduce 等显式通信；MPI 是代表。
- **PGAS**：逻辑上提供全局地址空间，但数据具有物理归属。
- **SPMD**：多个执行单元运行同一程序，但依据自身编号处理不同数据。
- **MPMD**：不同执行单元可以运行不同程序。
- **混合模型**：例如节点间 MPI、节点内线程、GPU 上 CUDA。

不要把 **SPMD** 和"数据并行训练"混为一谈：

- SPMD 描述"程序是否相同"；
- 数据并行描述"模型和数据如何切分"。

---

# 第二部分：CUDA 编程模型

## 9. 为什么 GPU 适合 AI/HPC

课程用"三胜"概括 GPU：

### 9.1 更多晶体管用于吞吐型计算

CPU 要做好分支预测、乱序执行、低延迟缓存和复杂控制；GPU 把更多面积用于大量相对简单的计算单元。

这不是"GPU 每个核心都比 CPU 核心强"，而是：

- CPU 强于低延迟、复杂控制和串行任务；
- GPU 强于规则、密集、批量、可并行的任务。

### 9.2 用并发隐藏延迟

GPU 的单次操作或显存访问可能并不低延迟，但 SM 保存大量 warp 的执行状态。一个 warp 等待内存时，调度器可以选择另一个就绪 warp。

### 9.3 高显存带宽

训练和推理频繁搬运权重、激活和 KV Cache，高带宽显存非常关键。但标称带宽只有在访问足够连续、事务利用率高时才可能接近。

课堂中的具体延迟、吞吐和"HBM 比 DDR 快多少倍"是用于建立直觉的示例，不是跨所有 CPU/GPU/内存配置都成立的常数。

## 10. CUDA 的软件层级与硬件层级

### 10.1 软件视角

```text
Kernel launch
└── Grid
    ├── Block / CTA
    │   ├── Warp（NVIDIA 当前通常为 32 threads）
    │   │   ├── Thread
    │   │   └── ...
    │   └── ...
    └── ...
```

- **thread**：CUDA 编程中最细的逻辑执行实例。
- **warp**：硬件调度与执行的一组线程。
- **block / CTA**：线程合作、共享 shared memory 和执行块级同步的范围。
- **grid**：一次 kernel launch 创建的所有 block。

### 10.2 硬件视角

```text
GPU
└── GPC 等更高层组织
    └── 多个 SM
        ├── warp schedulers
        ├── register file
        ├── shared memory / L1
        ├── CUDA cores / ALUs
        ├── Tensor Cores
        └── SFU 等专用单元
```

一个 block 会被调度到某个 SM，并通常在该 SM 上运行至结束。block 数可以远多于 SM 数，剩余 block 等待调度。

### 10.3 自动可扩展性的来源

一般 CUDA kernel 应让不同 block 相互独立，因为：

- block 的执行顺序没有保证；
- 设备可能有几十、几百或更多 SM；
- 同一程序无需根据 SM 数重写。

这是一种软件与硬件规模解耦。例外包括 cooperative launch、thread block cluster 等有额外约束的机制，但初学阶段先遵守"block 独立"最安全。

## 11. CUDA 程序的基本生命周期

典型流程：

1. CPU（host）准备输入；
2. 分配 GPU（device）内存；
3. 把数据从 host memory 复制到 device memory；
4. 使用 `<<<gridDim, blockDim, ...>>>` 启动 kernel；
5. GPU 并行执行；
6. 必要时同步；
7. 将结果复制回 CPU；
8. 释放资源并检查错误。

CUDA C++ 中常见执行空间修饰符：

- `__host__`：在 CPU 上执行；
- `__device__`：在 GPU 上执行，由 device 代码调用；
- `__global__`：声明 kernel，通常由 host 发起并在 device 上执行，返回类型为 `void`。

编译链可粗略理解为：

```text
CUDA C++ (.cu)
  ├── host code → host compiler → CPU code
  └── device code → nvcc toolchain → PTX / device binary
```

- **PTX** 是虚拟指令集/中间表示，便于在不同代际 GPU 上进一步生成机器代码。
- **CUBIN** 包含面向具体 GPU 架构的机器代码。

## 12. 从线程编号映射到数据

### 12.1 一维向量

```cpp
int i = blockIdx.x * blockDim.x + threadIdx.x;
if (i < N) {
    c[i] = a[i] + b[i];
}
```

其中：

- `threadIdx.x`：线程在 block 内的局部编号；
- `blockIdx.x`：block 在 grid 内的编号；
- `blockDim.x`：每个 block 的线程数；
- 边界判断处理 `N` 不是 block 大小整数倍的情况。

### 12.2 二维矩阵

```cpp
int row = blockIdx.y * blockDim.y + threadIdx.y;
int col = blockIdx.x * blockDim.x + threadIdx.x;
if (row < M && col < N) {
    c[row * N + col] = a[row * N + col] + b[row * N + col];
}
```

二维数组在内存中仍然是一段线性地址。以行主序为例：

```text
address(i, j) = base + i * stride_row + j * stride_col
```

对连续张量，通常 `stride_col = 1`，`stride_row = 列数`。理解 stride 是之后看 Triton 矩阵乘法指针计算的关键。

## 13. Warp、SIMT 与分支发散

### 13.1 SIMT

SIMT 是 Single Instruction, Multiple Threads。程序员看到独立线程，但硬件通常以 warp 为单位发射指令。

SIMT 与 SIMD 的直觉差异：

- SIMD 直接表达"一条向量指令作用于多个数据 lane"；
- SIMT 表达"许多逻辑线程运行同一 kernel"。

在 NVIDIA GPU 上，每个线程具有独立执行状态，但同一 warp 中的活跃线程仍以共同指令高效推进。

### 13.2 Warp divergence

若同一 warp 内部分线程走 `if`，另一部分走 `else`，硬件可能依次执行不同路径，并 mask 掉当前路径不参与的线程。

```cpp
if (threadIdx.x % 2 == 0) {
    expensive_a();
} else {
    expensive_b();
}
```

这会让每个 warp 都经历两条路径。优化方向是让相同分支的数据按 warp 边界聚集，使一个 warp 尽量只执行一条路径。

但不要把"任何 if 都很慢"当成规则：

- 所有线程条件一致时没有发散；
- 很短的分支可能被编译为 predication；
- 边界判断通常只影响少量 warp；
- 真正是否成为瓶颈应由 profiler 判断。

### 13.3 Warp-level primitives

warp 内线程可以通过 shuffle 等原语交换寄存器数据。例如 reduction 可在 `log2(32)` 轮中逐步求和，而不必经过 shared memory。

## 14. Block 内协作与同步

同一 block 内的线程可以：

- 访问 shared memory；
- 使用原子操作；
- 使用内存屏障；
- 使用 `__syncthreads()` 进行块级同步。

关键规则：

> `__syncthreads()` 必须由 block 中所有相关线程以一致的控制流到达，否则可能死锁或产生未定义行为。

不能仅仅因为"每个线程最终都会调用一次"就认为安全；它们必须在相同的同步阶段到达。

Compute Capability 9.0 起，CUDA 还提供可选的 **thread block cluster**，使一组 block 在受约束的硬件范围内协作，并可使用 distributed shared memory。它是进阶机制，不改变初学者优先保证普通 block 独立的原则。

## 15. GPU 内存层次

| 层次 | 可见范围 | 容量/速度直觉 | 主要用途 |
|---|---|---|---|
| Register | 单个 thread | 最小、最快 | 当前值、累加器 |
| Shared memory | 单个 block/CTA | 小、低延迟、显式管理 | 数据复用、线程协作 |
| L1 cache | 通常 SM 层级 | 硬件管理 | 缓存局部访问 |
| L2 cache | 全 GPU | 更大、更慢 | 跨 SM 数据缓存 |
| Global memory / HBM | 全 GPU | 最大、高延迟、高总带宽 | 权重、激活、输入输出 |
| Local memory | 逻辑上 thread 私有，物理上通常在 device memory | 可能很慢 | 寄存器不足时的 spill 等 |
| Constant memory | 全局只读且有缓存 | 适合 warp 内相同地址读取 | 常量参数 |

重要纠偏：

- "local"描述可见性，不保证物理位置靠近线程，也不保证快。
- shared memory 与 L1 常共享片上资源或可配置容量，但具体组织取决于架构。
- 寄存器不是无限的；每线程用得太多，会减少同一 SM 可驻留的 warp/block。

## 16. 合并访存（Coalescing）

一个 warp 的线程访问 global memory 时，硬件会把地址请求合并为尽量少的内存事务。

理想模式：

```text
lane 0 → A[k]
lane 1 → A[k+1]
lane 2 → A[k+2]
...
```

不理想模式：

- 大步长跨越；
- 随机地址；
- 每个事务只使用很少字节；
- 未对齐访问导致额外事务。

这就是为什么行主序矩阵中，常把 `threadIdx.x` 映射到连续列：同一 warp 内连续线程访问连续地址。

## 17. Tiling 与矩阵乘法

朴素矩阵乘法：

```text
C[m, n] = Σ_k A[m, k] * B[k, n]
```

一个高性能分块思路：

1. 每个 block 负责 `C` 的一个 `BLOCK_M × BLOCK_N` 输出块；
2. 沿 `K` 维以 `BLOCK_K` 为步长循环；
3. 每轮把 `A`、`B` 的对应 tile 从 global memory 搬到 shared memory；
4. block 内线程复用这些数据并进行乘加；
5. 结果在寄存器中累加；
6. 最后写回 global memory。

核心收益不是减少数学运算，而是增加数据复用：

- `A` 的一个元素可参与同一输出块的多列计算；
- `B` 的一个元素可参与同一输出块的多行计算；
- 从 HBM 读取一次后，在 shared memory / register 中多次使用。

这正是后续 Triton 与 TileLang 都以 tile 为核心抽象的原因。

## 18. Warp 调度、延迟隐藏与 Occupancy

### 18.1 延迟隐藏

当 warp A 等待 global memory 时，调度器可以发射 warp B。要做到这一点，SM 中必须有足够多的可运行 warp。

### 18.2 Occupancy

常用定义：

```text
Occupancy = SM 上实际驻留的 active warps / 该 SM 支持的最大 active warps
```

限制驻留数量的资源包括：

- 每 block 的线程数；
- 每线程的寄存器数；
- 每 block 的 shared memory；
- 架构对 block、warp 的数量上限。

### 18.3 高 Occupancy 不等于最高性能

更高 occupancy 通常有利于隐藏延迟，但可能与其他优化冲突：

- 更多寄存器可减少 spill 和重复访存；
- 更大的 shared-memory tile 可提高数据复用；
- 但它们会降低可同时驻留的 block/warp 数。

所以目标不是机械追求 100%，而是找到足以隐藏主要延迟、同时保留高数据复用和低指令开销的配置。

## 19. CUDA 优化检查表

课程总结的四个方向可以扩展成：

1. **暴露足够并行度**：grid 足够大，能覆盖全部 SM。
2. **提高数据复用**：用 register/shared memory 减少重复 HBM 访问。
3. **改善访存模式**：合并访问、对齐、减少无效事务。
4. **减少控制流浪费**：降低 warp divergence。
5. **控制资源使用**：在寄存器、shared memory、occupancy 之间权衡。
6. **使用专用单元**：适合时使用 Tensor Core、SFU 等。
7. **让搬运与计算重叠**：异步拷贝、pipeline、多 stream。
8. **基于测量优化**：先验证正确性，再 benchmark，再 profiler。

### API 选讲的准确理解

- **CUDA events**：适合测量 GPU 时间；不要只用 CPU 墙钟且忘记异步执行。
- **Streams**：同一 stream 内有序；不同 stream 的工作"有机会"并发，但是否真正重叠取决于依赖、资源、硬件能力和默认 stream 语义。
- **Unified Memory**：简化地址与迁移管理，但数据迁移仍有成本；性能敏感场景需要关注预取、驻留和缺页。
- **P2P**：在支持的拓扑与配置下允许 GPU 间直接访问或复制；不是所有 GPU 对都支持，性能也取决于 PCIe/NVLink 等互联。

---

# 第三部分：从 Thread-level 到 Tile-level

## 20. 为什么需要 Tile-level Programming

CUDA 的控制力强，但程序员要显式完成：

- 每个 thread 负责什么；
- thread/block 的索引如何映射到数据；
- global/shared/register 之间怎样搬运；
- 如何同步；
- 如何处理边界；
- 如何避免 bank conflict、发散和低效访存。

但很多 AI Kernel 的自然表述是：

> 取一块张量 → 做一次块级计算 → 把结果写回。

因此 Tile-level DSL 把 block 内的大量线程协作抽象成 tile 操作：

1. 确定当前 tile；
2. 加载 tile；
3. 计算；
4. 写回；
5. 用 mask 处理边界。

程序员仍决定关键分块，编译器负责把 tile 内计算映射到 thread/warp。

## 21. Triton 编程模型

### 21.1 基本视角

- CUDA 从 **thread** 出发；
- Triton 从 **program instance** 出发；
- 一个 Triton program 通常负责一个输出 tile；
- `tl.program_id(axis=...)` 类似取得当前 program 在 launch grid 中的位置；
- thread ID 通常不显式出现。

"一个 Triton program 约等于一个 CUDA block/CTA"是有用的初学者类比，但不是所有情况下都应理解成严格的一对一源码映射。

### 21.2 向量加法

核心结构：

```python
pid = tl.program_id(axis=0)
offsets = pid * BLOCK_SIZE + tl.arange(0, BLOCK_SIZE)
mask = offsets < n_elements

x = tl.load(x_ptr + offsets, mask=mask, other=0.0)
y = tl.load(y_ptr + offsets, mask=mask, other=0.0)
z = x + y
tl.store(z_ptr + offsets, z, mask=mask)
```

理解重点：

- `offsets` 是一组下标，不是单个线程的一个下标；
- `mask` 防止最后一个 tile 越界；
- `other=0.0` 为越界 load 提供填充值；
- 写法表达的是"整个 tile 的向量操作"。

### 21.3 矩阵乘法

每个 program 负责 `C` 的一个输出块：

```text
acc = zeros(BLOCK_M, BLOCK_N)
for k0 in range(0, K, BLOCK_K):
    a_tile = load A[m:m+BLOCK_M, k0:k0+BLOCK_K]
    b_tile = load B[k0:k0+BLOCK_K, n:n+BLOCK_N]
    acc += dot(a_tile, b_tile)
store C tile
```

这里有三个分块尺度：

- `BLOCK_M`：输出 tile 的行数；
- `BLOCK_N`：输出 tile 的列数；
- `BLOCK_K`：归约维每轮处理的长度。

指针计算依赖 stride：

```text
A[i,j] 地址 = A_base + i * stride_am + j * stride_ak
B[i,j] 地址 = B_base + i * stride_bk + j * stride_bn
```

性能还受 program 执行顺序影响，因为相邻 program 是否复用同一批 A/B 数据会改变 L2 命中率。

### 21.4 Triton 的真实优缺点

优点：

- Python 风格、代码短；
- 保留 tile shape、program 排列等重要决策；
- 适合 elementwise、reduction、matmul、attention 和融合算子；
- 支持 autotune；
- 可显著降低手写 CUDA 的工程成本。

代价：

- 一些线程级布局、shared memory 组织和流水线由编译器决定；
- 当性能瓶颈恰好位于这些细节时，调优空间较受限；
- 复杂硬件特化和某些 pipeline 表达可能更困难。

课堂把编译器称为"黑盒"、说 PTX/SASS 不透明，是强调控制粒度的说法，不应理解为"完全无法查看生成代码"。实践中可以 dump IR/PTX、结合 profiler 分析；只是源码与最终机器行为之间的映射比手写 CUDA 更间接。

## 22. TileLang 编程模型

TileLang 的目标位置可以理解为：

> 保留 tile-level 的简洁，同时把影响性能的数据流和内存层次更明确地写出来。

课程中的关键层次：

- 函数参数 `T.Tensor`：global memory 中的输入输出；
- `T.alloc_shared`：CTA 内共享的 tile；
- `T.alloc_fragment`：寄存器中的局部累加结果；
- `T.copy`：不同内存层次之间搬运；
- `T.gemm`：tile 级矩阵乘加；
- `T.Pipelined`：沿分块循环表达搬运与计算重叠；
- `T.Kernel(...)`：定义 launch grid / CTA 级程序实例。

一个矩阵乘法的数据流可以写成：

```text
Global A/B
   │ T.copy
   ▼
Shared-memory tiles
   │ T.gemm
   ▼
Register fragment / accumulator
   │ T.copy
   ▼
Global C
```

其价值在于，读代码时能直接看到：

- 数据在哪一层；
- 什么数据会被复用；
- 沿哪个维度循环；
- 搬运和计算是否流水化。

## 23. CUDA、Triton、TileLang 怎样选择

| 维度 | CUDA C++ | Triton | TileLang |
|---|---|---|---|
| 主要抽象 | thread / warp / block | program / tensor tile | tile + 显式内存层次和数据流 |
| 控制粒度 | 最细 | 较高层 | 介于二者之间 |
| 开发效率 | 较低 | 高 | 中等 |
| 硬件细节负担 | 高 | 较低 | 中等 |
| 典型场景 | 极端调优、特殊机制、库级实现 | 快速开发与融合常见算子 | 需要显式数据流、pipeline 和可读结构的高性能 kernel |

选择不是"谁永远更快"，而要问：

- 算子是否规则？
- 瓶颈是计算、访存、同步还是 launch overhead？
- 是否需要特殊硬件指令或复杂流水线？
- 是否需要跨硬件后端？
- 团队能承担多少开发和维护成本？

## 24. `torch.compile` 与手写 Kernel 的关系

课堂问答认为二者大体正交，这个方向是对的，但机制需要更精确地区分：

- TorchDynamo 负责从 Python 程序捕获可编译的计算图；
- TorchInductor 等后端进行融合、布局、调度和代码生成，GPU 上常生成 Triton kernel；
- CUDA Graphs 可用于减少 CPU launch overhead，但它不是 `torch.compile` 捕获 Python 计算图的同义词；
- 手写 Triton/TileLang/CUDA 仍有价值，因为通用编译器不一定能生成最适合特殊算子、数据布局或硬件特性的实现。

因此可以同时使用：

```text
模型/图级优化：torch.compile
          +
关键热点算子：手写或定制 Kernel
```

---

# 第四部分：把这些知识连回 AI Infra

## 25. AI Infra 的共同问题

| 层级 | 典型对象 | 核心问题 |
|---|---|---|
| 单条指令/单个 warp | SIMD/SIMT、分支 | lane 是否都做有效工作 |
| 单个 block/CTA | tiling、shared memory | 数据能否在片上复用 |
| 单个 GPU | SM 调度、HBM、stream | 是否计算饱和、带宽饱和、延迟被隐藏 |
| 多 GPU | TP/PP/DP/EP、collectives | 计算与通信如何重叠 |
| 单机推理服务 | batching、KV Cache、调度 | 吞吐、首 token 延迟、显存容量如何权衡 |
| 集群 | 网络拓扑、容错、资源调度 | 如何扩展并保持利用率 |

同一个思想不断重复：

1. 找到可并行部分；
2. 选择分块；
3. 把块映射到硬件；
4. 把数据放到合适存储层；
5. 安排通信和同步；
6. 用并发隐藏不可避免的延迟；
7. 测量瓶颈并迭代。

## 26. 为什么矩阵乘法贯穿整个课程

Transformer 的大量计算最终落在 GEMM 或 GEMM-like Kernel 上，包括：

- Q/K/V 线性投影；
- attention 中的 `QKᵀ` 和 `PV`；
- MLP 的上投影、门控、下投影；
- 训练中的梯度计算；
- 张量并行下的分片矩阵乘。

矩阵乘法同时包含：

- 大量独立输出元素；
- 沿 K 维的归约；
- 明确的数据复用；
- 对布局、带宽、缓存和专用矩阵单元的强依赖。

因此它是学习并行化、tiling、memory hierarchy 和 pipeline 的最佳样例。

---

# 第五部分：课程之外必须补上的性能模型

## 27. Amdahl 定律：串行部分决定强扩展上限

若程序中可并行比例为 `p`，使用 `N` 个处理单元，理想加速比为：

```text
Speedup(N) = 1 / ((1 - p) + p / N)
```

例如 95% 可并行，即使设备无限多，最大加速也只有 `1 / 0.05 = 20` 倍。

现实还要加上通信、同步和调度开销，所以实际更低。它解释了为什么优化最后一点串行瓶颈可能比继续加卡更重要。

## 28. Strong Scaling 与 Weak Scaling

- **Strong scaling**：总问题规模不变，增加设备，看运行时间能否下降。
- **Weak scaling**：每台设备的工作量不变，设备与总问题规模一起增长，看总时间能否保持。

训练一个固定模型更接近 strong scaling；随着 GPU 数增加同时增大 batch 或模型规模，则包含 weak scaling 的味道。

## 29. Arithmetic Intensity 与 Roofline

算术强度：

```text
Arithmetic Intensity = 运算次数 FLOPs / 从慢速内存搬运的字节数 Bytes
```

粗略性能上限：

```text
Performance ≤ min(Peak Compute, Memory Bandwidth × Arithmetic Intensity)
```

- 算术强度低：通常 **memory-bound**；
- 算术强度高：可能 **compute-bound**。

向量加法每个元素只做一次加法，却要读两个输入、写一个输出，通常是 memory-bound。分块矩阵乘法可让载入的数据被多次复用，算术强度高得多，更容易接近计算峰值。

这比"看到 GPU 峰值 FLOPS 就估算程序速度"可靠得多。

## 30. Little 定律式的延迟隐藏直觉

要维持高吞吐，系统中需要足够多的在途工作：

```text
并发在途量 ≈ 吞吐量 × 单次延迟
```

显存延迟很高时，需要许多 warp 在途；推理服务中单请求生成有等待时，需要 continuous batching 等调度策略维持设备利用率。底层 GPU 调度和上层服务调度，在这里有相似的结构。

## 31. 数值精度不是附属问题

GPU 的不同单元和编译选项可能在速度与精度之间取舍：

- FP32、TF32、FP16、BF16、FP8 等格式吞吐不同；
- 累加精度可能高于输入精度；
- `--use_fast_math` 等选项可能使用更快但精度/语义不同的近似；
- 并行 reduction 改变求和顺序，浮点结果可能与串行结果略有不同。

性能优化必须同时定义正确性容忍度：绝对误差、相对误差、ULP、任务指标或训练稳定性。

---

# 第六部分：容易误解的课堂表述

## 32. 纠偏清单

1. **"GPU 比 CPU 快"**
   只对适合高吞吐并行、具有足够工作量和良好数据访问的任务成立。

2. **"有 N 个核心就快 N 倍"**
   这是无依赖、无通信、无调度开销、负载完全均衡时的理想上限。

3. **"一个 warp 的 32 个线程必须一直执行相同指令"**
   它们可以发生分支，但不同路径通常会被分阶段执行，造成发散开销。

4. **"block 之间绝对不能通信"**
   普通 kernel 的可扩展模型要求 block 独立；现代 CUDA 有 cooperative groups、cluster 等受约束机制，但不能默认依赖任意 block 的调度顺序。

5. **"Shared memory 一定比 global memory 快"**
   通常低延迟且可控，但 bank conflict、额外同步、低复用或过高资源占用都可能抵消收益。

6. **"Occupancy 越高越好"**
   需要"足够高"来隐藏延迟，不必盲目追求 100%。

7. **"多个 stream 就一定并发"**
   stream 只提供并发机会；真实重叠取决于依赖和硬件资源。

8. **"Unified Memory 不需要考虑拷贝"**
   API 简化了，但迁移、缺页和驻留成本仍然存在。

9. **"Triton 看不到生成代码"**
   可检查编译中间表示和生成代码；真正的限制是控制较间接、调优与定位成本可能较高。

10. **"torch.compile 就是 CUDA Graph"**
    计算图捕获、编译优化、代码生成和 CUDA Graphs 是不同环节，可能组合使用。

11. **"TileLang 一定跨平台且性能都好"**
    可移植抽象能减少迁移成本，但新后端仍需要编译器支持、调度适配和性能验证。

---

# 第七部分：术语速查

| 术语 | 一句话解释 |
|---|---|
| HPC | High Performance Computing，高性能计算 |
| AI Infra | 支撑 AI 训练、推理和部署的软硬件系统 |
| Kernel | 在加速器上执行的一段计算函数 |
| Host / Device | 通常分别指 CPU 侧 / GPU 侧 |
| Grid | 一次 CUDA kernel launch 的全部 block |
| Block / CTA | 可在同一 SM 上合作、共享 shared memory 的线程组 |
| Warp | NVIDIA GPU 的线程调度/执行分组，通常含 32 个线程 |
| SM | Streaming Multiprocessor，执行 block/warp 的硬件单元 |
| SIMT | Single Instruction, Multiple Threads |
| SIMD | Single Instruction, Multiple Data |
| SPMD | Single Program, Multiple Data |
| Tile | 数据张量的一个分块 |
| Tiling | 把大计算分成适合并行和存储层次的小块 |
| Coalescing | 合并同一 warp 的连续内存请求 |
| Divergence | 同一 warp 的线程走不同控制路径 |
| Occupancy | SM 实际驻留 warp 与架构上限之比 |
| Register pressure | 每线程寄存器需求对驻留并发度造成的压力 |
| Spill | 寄存器不足，变量被放到更慢的 local/device memory |
| Reduction | 把多个值按加、最大值等操作聚合 |
| Collective | 多进程/多设备共同参与的通信操作 |
| DSL | Domain-Specific Language，面向特定领域的语言 |
| PTX / SASS | NVIDIA 的虚拟指令集 / 更接近实际 GPU 机器指令的表示 |
| HBM | High Bandwidth Memory，高带宽显存 |
| Arithmetic Intensity | 每搬运一字节数据能完成多少运算 |

---

# 第八部分：推荐的权威延伸资料

1. [NVIDIA CUDA Programming Guide：Programming Model](https://docs.nvidia.com/cuda/cuda-programming-guide/01-introduction/programming-model.html)
   优先读 thread block、grid、SM、block 独立性和内存层次。

2. [NVIDIA CUDA Programming Guide：Introduction to CUDA C++](https://docs.nvidia.com/cuda/cuda-programming-guide/02-basics/intro-to-cuda-cpp.html)
   对照课堂中的 kernel、内置索引和 thread block cluster。

3. [Triton 官方教程总览](https://triton-lang.org/main/getting-started/tutorials/)
   推荐依次做 vector add、fused softmax、matrix multiplication。

4. [Triton Vector Addition](https://triton-lang.org/main/getting-started/tutorials/01-vector-add.html)
   用于掌握 `program_id → offsets → mask → load → compute → store`。

5. [Triton Matrix Multiplication](https://triton-lang.org/main/getting-started/tutorials/03-matrix-multiplication.html)
   重点看三维分块、pointer arithmetic、L2-friendly program ordering 和 autotune。

6. [TileLang TileLibrary Reference](https://tilelang.com/language_ref/tilelibrary.html)
   查阅 `T.alloc_shared`、`T.Pipelined` 等原语。

7. [PyTorch `torch.compile` 文档](https://docs.pytorch.org/docs/stable/generated/torch.compile.html)
   用于区分图编译、Inductor/Triton 代码生成和 CUDA Graphs。

---

## 结语：应该带走的不是术语，而是一个分析模板

以后看到任何 AI Infra 性能问题，可以按这个顺序问：

1. **计算是什么？** 输入、输出、规则分别是什么？
2. **并行机会在哪里？** 哪些任务没有依赖？
3. **怎样分块？** 按数据、功能、模型层、序列还是专家？
4. **映射到哪里？** thread、warp、CTA、SM、GPU、节点分别承担什么？
5. **数据在哪里？** register、shared、HBM 还是远端 GPU？
6. **瓶颈是什么？** 计算、带宽、延迟、同步、通信还是 launch overhead？
7. **能否重叠？** 搬运与计算、通信与计算、不同请求是否可流水化？
8. **怎样证明？** 正确性测试、benchmark 和 profiler 是否支持你的判断？

当你能够沿这八个问题分析一个 Kernel 或一个分布式系统时，就已经不再只是"知道几个 GPU 术语"，而是在用 Infra 的方式思考。
