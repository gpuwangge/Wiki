<p float="left">
  <img src="https://github.com/gpuwangge/Wiki/blob/main/images/NPUArchitecture1.png" alt="alt text">  
</p> 

# NPU Introduction

NPU（Neural Processing Unit，神经网络处理器）是一类专门用来高效执行 AI 推理或训练计算的处理器。它不像 CPU 那样擅长复杂控制逻辑，也不像 GPU 那样以通用并行为主要目标；NPU 会把神经网络中最常见的操作——尤其是大规模矩阵乘法、卷积、激活和量化——映射到专门的计算阵列与片上存储结构中。

这张图描述的是一个典型 NPU 的数据流：CPU 或运行时下发任务，DMA 搬运数据，片上 SRAM 暂存权重与激活值，Tensor Compute Engine 执行核心计算，最后通过后处理模块生成输出 Tensor。

***

## 总体数据流

```text
Host CPU / Runtime
        |
        v
NPU Command Processor
        |
        v
DMA / Tensor Data Mover <------> System Memory (LPDDR / DRAM)
        |
        v
On-chip SRAM Buffers
(Weights / Activations / Accumulators / L2)
        |
        v
NPU Compute Cluster
(Tensor Compute Engines + NoC)
        |
        v
Post-Processing
(Quantization / Pooling / Elementwise Ops)
        |
        v
Output Tensor
```

可以把它理解成一个高吞吐量的 AI 数据工厂：

- **CPU** 是任务发起者和调度者
- **DRAM** 是容量大但访问较慢的仓库
- **DMA** 是数据搬运系统
- **SRAM** 是靠近计算单元的小型高速缓存区
- **Tensor Compute Engine** 是大规模矩阵计算的核心生产线
- **NoC** 是片上高速互连网络
- **Post-Processing** 负责做输出整理、格式转换和轻量算子
- **Output Tensor** 是最终模型推理结果

***

## SoC 与主机侧模块

### Host CPU / Runtime

`Host CPU / Runtime` 表示运行神经网络软件栈的主机处理器及其运行时系统。

它通常负责：

- 加载模型、权重和输入数据
- 创建并管理 NPU 任务
- 将神经网络计算图拆分为可执行的硬件任务
- 配置缓冲区地址、张量形状和数据格式
- 向 NPU 提交命令
- 等待任务完成，并读取输出结果
- 处理 NPU 不支持或不适合运行的算子

这里的 `Runtime` 可以是厂商提供的 NPU Runtime、推理框架后端，或图编译器生成的执行层。例如，一个模型可能先由 ONNX、TensorFlow Lite、PyTorch Export 或自研图编译器转换，再由 Runtime 提交到 NPU。

可以把 CPU 看成是“项目经理”：它不一定亲自完成大量矩阵乘法，但负责决定计算什么、使用哪些数据、何时开始、结果交给谁。

***

### System Memory (LPDDR / DRAM)

`System Memory (LPDDR / DRAM)` 是系统主内存，通常保存模型运行所需的大部分数据。

其中可能包括：

- 神经网络权重，例如卷积核和全连接层参数
- 输入 Tensor，例如图像、音频、Token Embedding 或传感器数据
- 中间结果 Tensor
- 最终输出结果
- NPU 命令缓冲区和描述符
- 模型常量、查找表和量化参数

LPDDR/DRAM 的优点是容量大。例如，一个大模型的权重可能远超 NPU 片上 SRAM 容量，因此必须长期存放在外部 DRAM 中。

但 DRAM 的问题也很明显：

- 延迟通常比片上 SRAM 高
- 访问功耗较高
- 带宽有限且会与 CPU、GPU、ISP 等其他 SoC 模块竞争
- 重复读取权重或中间特征会显著拖慢推理性能

因此，NPU 的一个核心设计目标是：**尽量少访问 DRAM，尽量复用片上 SRAM 中的数据。**

***

## 命令与数据搬运

### NPU Command Processor

`NPU Command Processor` 是 NPU 内部的命令控制中心。

它接收来自 CPU Runtime 的任务描述，并将其转换成 NPU 硬件可以执行的操作。一个任务描述通常会包含：

- 输入和输出 Tensor 的内存地址
- Tensor 的维度和布局，例如 `NHWC`、`NCHW`、`BHWCN`
- 数据类型，例如 `FP32`、`FP16`、`BF16`、`INT8`、`INT4`
- 卷积、矩阵乘法或 Attention 的计算参数
- 权重地址与量化参数
- DMA 搬运命令
- 依赖关系和同步信号
- 要使用的计算 Cluster 或 Compute Tile

它的主要职责包括：

1. **解析命令流**  
   读取主机提交的命令、描述符或任务队列。

2. **启动数据搬运**  
   告诉 DMA 从哪里取数据、搬到哪里，以及搬运多大的数据块。

3. **配置计算单元**  
   设置 Tensor Compute Engine 的计算模式、数据精度和维度参数。

4. **管理依赖与同步**  
   确保权重、激活值等数据到位后，再启动对应计算。

5. **处理任务完成事件**  
   将完成状态写回内存，必要时向 CPU 发送中断或通知。

可以把它理解为 NPU 的“控制塔”：它不直接完成矩阵乘法，但协调数据和计算单元，使流水线能够连续运行。

***

### DMA / Tensor Data Mover

`DMA / Tensor Data Mover` 是 NPU 的数据搬运引擎。

DMA 的全称是 Direct Memory Access，即“直接内存访问”。它可以在不需要 CPU 逐字节参与的情况下，将数据从一个存储位置搬到另一个位置。

在 NPU 中，DMA 常见的数据搬运路径包括：

```text
DRAM -> Weight Buffer
DRAM -> Activation Buffer
DRAM -> L2 Shared SRAM
Local SRAM -> Accumulator Buffer
Accumulator Buffer -> DRAM
Output Buffer -> DRAM
```

DMA 通常不仅仅是简单复制数据，还可能负责：

- 张量分块（Tiling）
- 数据布局转换（Layout Transform）
- 数据转置（Transpose）
- 数据拼接或切分
- 对齐和填充（Padding）
- 数据类型转换
- 简单的数据重排
- 多播（Multicast）到多个计算 Tile
- 双缓冲切换（Double Buffering）

例如，在执行一个大型矩阵乘法时，完整矩阵可能放不进片上 SRAM。DMA 会把它切成多个 Tile：

```text
大矩阵 / 大特征图
        |
        v
切分为多个小块 Tile
        |
        v
逐块搬入片上 SRAM
        |
        v
Tensor Compute Engine 计算
        |
        v
写回结果或进入下一层
```

DMA 对性能非常重要。即使 MAC 阵列算力很强，如果数据不能及时送达，计算单元也会空转。这种情况常被称为 **memory-bound** 或 **data-starved**。

***

## 片上存储层次

### L2 Shared SRAM

`L2 Shared SRAM` 是多个 NPU Compute Tile 或 Compute Engine 可共享访问的片上高速存储。

SRAM（Static Random Access Memory）相比 DRAM：

- 访问延迟更低
- 带宽更高
- 功耗通常更低
- 面积成本更高，因此容量通常较小

L2 Shared SRAM 常用于保存可在多个计算单元之间复用的数据，例如：

- 多个 Tile 共享的权重块
- 多个计算阶段共享的中间特征
- Attention 中重复使用的 Key/Value Tile
- 后处理前暂存的输出数据
- 需要跨多个 Compute Engine 传递的 Tensor Tile

它可以降低重复从 DRAM 读取同一份数据的次数。

例如，如果一个卷积层的权重会被多个输入 Tile 使用，那么将权重先加载到 L2 SRAM，再分发给多个 Tensor Compute Engine，通常比每个 Engine 都从 DRAM 单独读取更高效。

***

### Weight Buffer

`Weight Buffer` 用于暂存神经网络模型权重。

权重是模型训练后固定下来的参数，例如：

- 卷积层中的 Kernel
- 全连接层中的矩阵
- Transformer 中的 `Q`、`K`、`V` 和输出投影矩阵
- MLP 中的升维和降维矩阵
- LayerNorm、RMSNorm、BatchNorm 参数
- 偏置（Bias）
- 量化 Scale 与 Zero Point

权重数据通常具有较高复用率。例如，一个卷积核会作用于多个空间位置，一个线性层权重会应用到多个 Token 或 Batch 元素。因此，NPU 会尽可能将权重保留在靠近计算单元的位置。

典型数据路径为：

```text
DRAM
  |
  v
Weight Buffer
  |
  v
Tensor Compute Engine
```

对于大模型来说，权重往往是带宽和容量的主要压力来源。降低权重精度，例如从 `FP16` 降到 `INT8` 或 `INT4`，能显著减少存储和 DRAM 传输需求。

***

### Activation Buffer

`Activation Buffer` 用于存放输入 Tensor 和中间激活值。

Activation（激活值）可以理解为网络每一层计算过程中产生的数据。例如：

```text
Input Image
    |
    v
Convolution Output
    |
    v
Activation Function Output
    |
    v
Next Layer Input
```

对于 Transformer，Activation 可能包括：

- Token Embedding
- Hidden States
- Query、Key、Value Tensor
- Attention Score
- MLP 中间结果
- Residual Add 的输入和输出

Activation Buffer 的职责是让计算单元能够以高带宽读取当前需要处理的数据块。

典型路径为：

```text
Input Tensor / Intermediate Tensor
            |
            v
     Activation Buffer
            |
            v
   Tensor Compute Engine
```

Activation 往往比权重更动态：权重通常在一次推理中不变，而 Activation 会随输入内容变化。因此，Activation Buffer 需要频繁装载和写回数据。

***

### Accumulator Buffer

`Accumulator Buffer` 保存计算过程中的部分和（Partial Sum），简称 `Psum`。

以矩阵乘法为例：

\[
C_{m,n} = \sum_k A_{m,k} \times B_{k,n}
\]

如果矩阵很大，NPU 不会一次完成完整的 \(k\) 维累加，而会将它分为多个 Tile：

\[
C_{m,n} =
\sum_{k \in Tile_0} A_{m,k}B_{k,n}
+
\sum_{k \in Tile_1} A_{m,k}B_{k,n}
+
\cdots
\]

每处理一个输入 Tile，都会产生一部分累加结果。这个结果不能立刻丢弃，需要保存下来，等所有 Tile 处理完成后才得到最终输出。

Accumulator Buffer 用于：

- 保存矩阵乘法的部分和
- 保存卷积输出的累加值
- 支持多次 K 维分块计算
- 保存融合算子之间的中间结果
- 在量化前保留较高精度结果

Accumulator 通常会使用比输入更高的精度。例如：

```text
INT8 × INT8 -> INT32 accumulation
FP16 × FP16 -> FP32 accumulation
```

这样可以减少长链累加过程中的精度损失和溢出风险。

***

### Local SRAM

`Local SRAM` 是每个 Tensor Compute Engine 或 Compute Tile 私有的高速片上存储。

它比 Shared L2 SRAM 更靠近计算阵列，通常用于保存当前计算 Tile 最频繁访问的数据：

- 当前输入 Activation Tile
- 当前 Weight Tile
- 当前输出或 Partial Sum Tile
- 小型查找表
- 局部计算状态
- Vector Unit 的临时数据

其典型特点是：

- 容量较小
- 访问速度极快
- 带宽极高
- 通常只服务于一个 Compute Tile
- 可以避免大量跨 Tile 数据传输

可以把它看作计算引擎旁边的“工作台”，而 L2 SRAM 是多个工作台共同使用的“近距离材料仓库”。

***

## 核心计算模块

### NPU Compute Cluster

`NPU Compute Cluster` 是 NPU 的主要算力区域。

一个 Cluster 往往由多个 Tensor Compute Engine 组成，并配套：

- 本地 SRAM
- 片上互连 NoC
- 调度和同步逻辑
- DMA 接口
- 性能监控单元
- 电源和时钟管理逻辑

现代 NPU 通常不会只使用一个巨型计算阵列，而是会采用多个可扩展的 Compute Tile。这样可以根据功耗、面积和模型规模，在不同产品中配置不同数量的 Cluster。

例如：

```text
Small NPU:
  1 Compute Cluster
  2 Tensor Compute Engines

Mobile NPU:
  Multiple Compute Clusters
  Multiple Tensor Compute Engines per Cluster

Datacenter AI Accelerator:
  Many Compute Clusters
  Large SRAM hierarchy
  High-bandwidth external memory
```

Cluster 化设计也有助于并行运行不同层、不同 Batch、不同 Token 或不同模型任务。

***

### Tensor Compute Engine

`Tensor Compute Engine` 是 NPU 内真正执行 AI 核心数学运算的模块。

它的主要目标是高效执行：

- 矩阵乘法（GEMM）
- 批量矩阵乘法（Batched GEMM）
- 卷积（Convolution）
- 深度可分离卷积（Depthwise Convolution）
- Attention 中的 `QK^T` 与 `Softmax(V)` 相关计算
- MLP / Fully Connected 层
- 向量乘加和张量变换

Tensor Compute Engine 往往具有固定的数据流结构，例如：

- Weight Stationary
- Output Stationary
- Activation Stationary
- Row Stationary
- Systolic Dataflow

不同数据流策略的核心目的相同：**最大化数据复用，最小化昂贵的数据移动。**

***

### MAC Array / Systolic Array

`MAC Array / Systolic Array` 是 NPU 的主要计算阵列。

MAC 是 Multiply-Accumulate 的缩写，表示乘加操作：

\[
acc = acc + a \times b
\]

神经网络中的卷积和矩阵乘法几乎都可以分解为大量 MAC 操作。

一个简化的矩阵乘法如下：

\[
C = A \times B
\]

其中每个输出元素满足：

\[
C_{i,j} = \sum_k A_{i,k}B_{k,j}
\]

MAC Array 会在很多 Processing Element（PE）中并行执行这些乘加操作。

### Systolic Array 的直观理解

Systolic Array 是一种规则的二维计算网格。数据会在阵列中以有节奏的方式流动，类似血液在心脏泵动下穿过血管，因此得名 “Systolic”。

```text
Activation Data  ---> ---> --->
                    PE  PE  PE
Weight Data       v     v   v
                  PE  PE  PE
                  v     v   v
                  PE  PE  PE
```

每个 PE 通常完成：

```text
Partial Sum += Activation × Weight
```

其优点包括：

- 高并行度
- 数据在阵列内部复用
- 规则的数据流，硬件实现效率高
- 能减少外部存储访问
- 很适合规则的矩阵乘法和卷积计算

它特别适合 `INT8`、`FP16`、`BF16`、`FP8` 等 AI 常用低精度格式。

***

### Vector / Activation Unit

`Vector / Activation Unit` 用于处理那些不完全适合大矩阵阵列的操作。

MAC Array 擅长密集线性代数，但完整神经网络还包含许多逐元素、逐向量或归约类运算，例如：

- ReLU
- GELU
- SiLU / Swish
- Sigmoid
- Tanh
- Softmax
- LayerNorm
- RMSNorm
- Elementwise Add
- Elementwise Multiply
- Clamp
- Reduce Sum
- Reduce Max
- Mean / Variance
- RoPE（Rotary Position Embedding）
- 位置编码与张量变形

这些任务通常由 Vector Unit 完成。

例如，一个 ReLU 操作可以写成：

\[
\mathrm{ReLU}(x) = \max(0, x)
\]

而 LayerNorm 会涉及均值、方差、归约和逐元素归一化：

\[
\mathrm{LayerNorm}(x) =
\gamma \cdot \frac{x-\mu}{\sqrt{\sigma^2+\epsilon}} + \beta
\]

Vector Unit 的存在让 NPU 不只能够完成矩阵乘法，还能高效运行完整的神经网络图。

***

## 片上互连

### NoC / Interconnect

`NoC / Interconnect` 是 Network-on-Chip 的缩写，即片上网络。

当一个 NPU 包含多个 Tensor Compute Engine、多个 SRAM Bank、DMA 和控制模块时，它们需要高效交换数据。NoC 就是连接这些模块的“片上高速网络”。

它负责在以下模块之间传输数据：

```text
DMA <-> L2 Shared SRAM
L2 Shared SRAM <-> Tensor Compute Engines
Tensor Compute Engine <-> Tensor Compute Engine
Compute Engine <-> Accumulator Buffer
Compute Cluster <-> Post-Processing
```

NoC 的常见拓扑包括：

- Ring（环形）
- Mesh（网格）
- Torus（环面）
- Crossbar（交叉开关）
- Tree（树状）
- Hierarchical Interconnect（分层互连）

NoC 的性能会直接影响 NPU 的可扩展性。如果计算阵列很强，但片上网络带宽不足，那么不同 Tile 会因为等待数据而闲置。

高质量的 NPU NoC 通常需要考虑：

- 带宽
- 延迟
- 多播能力
- 拥塞控制
- QoS
- 功耗
- 数据一致性或同步语义
- 不同类型流量的优先级

对于 AI 工作负载，多播尤其重要。例如，同一块权重数据可能需要发送给多个 Compute Tile；如果 NoC 支持硬件多播，就可以减少重复传输。

***

## 后处理与输出

### Post-Processing

`Post-Processing` 是主计算完成后的结果处理阶段。

许多模型算子并不只是简单的矩阵乘法。例如，一个卷积层的完整流程可能包括：

```text
Convolution
    |
    v
Bias Add
    |
    v
Activation
    |
    v
Quantization
    |
    v
Output Tensor
```

Post-Processing 会将这些轻量但必要的操作尽可能融合执行，以减少中间结果写回 DRAM 的次数。

常见的后处理内容包括：

- Quantization / Dequantization
- Pooling
- Elementwise Add
- Elementwise Multiply
- Bias Add
- Activation Functions
- Tensor Layout Conversion
- Clamp
- Requantization
- 输出格式转换

算子融合（Operator Fusion）非常重要。例如，如果 `GEMM -> Bias -> GELU` 可以在片上连续完成，就不需要将 GEMM 输出先写入 DRAM，再读回来做 Bias 和 GELU。

```text
较低效率：
GEMM -> DRAM -> Bias -> DRAM -> GELU -> DRAM

较高效率：
GEMM -> Bias -> GELU -> Output
```

后者能显著减少内存带宽和功耗。

***

### Quantization / Dequantization

`Quantization / Dequantization` 用于在不同数值精度之间转换数据。

神经网络模型常用的数据格式包括：

| 数据类型 | 常见用途 | 特点 |
|---|---|---|
| `FP32` | 训练、高精度计算 | 精度高，带宽和存储开销大 |
| `FP16` | 推理和训练 | 常见半精度格式 |
| `BF16` | 大模型训练与推理 | 动态范围接近 FP32 |
| `FP8` | 高吞吐 AI 计算 | 带宽和算力效率高 |
| `INT8` | 高效推理 | 常见于移动端和边缘设备 |
| `INT4` | 大模型权重量化 | 存储和带宽开销更低 |

量化的概念是将高精度数值映射到较低比特数表示。例如：

\[
x_{int8} = \mathrm{round}\left(\frac{x_{float}}{scale}\right) + zero\_point
\]

反量化则近似恢复为浮点数：

\[
x_{float} = (x_{int8} - zero\_point) \times scale
\]

量化的收益包括：

- 减少模型大小
- 降低 DRAM 带宽
- 提高 MAC 阵列吞吐量
- 降低功耗
- 提升边缘设备上的推理速度

但量化也可能损失精度，因此需要选择合适的量化策略，例如 per-tensor、per-channel、per-group 或 mixed precision。

***

### Pooling / Elementwise Ops

`Pooling / Elementwise Ops` 包含一类常见的非矩阵计算操作。

### Pooling

Pooling 常见于 CNN，用来降低空间分辨率或聚合局部特征。

例如 Max Pooling：

\[
y = \max(x_1, x_2, \dots, x_n)
\]

Average Pooling：

\[
y = \frac{1}{n}\sum_{i=1}^{n}x_i
\]

它可以减少后续层的计算量，并扩大感受野。

### Elementwise Operations

Elementwise Ops 是逐元素操作，例如：

```text
y = a + b
y = a * b
y = max(a, 0)
y = clamp(x, min, max)
```

在 Transformer 中，常见的逐元素操作包括：

- Residual Add
- Gating
- Bias Add
- Activation Function
- Masking
- Scale
- RoPE 相关变换

这些操作计算量不一定很大，但如果频繁写回 DRAM，会造成明显的带宽和功耗问题。因此它们通常适合与主计算算子融合。

***

### Output Tensor

`Output Tensor` 是整个 NPU 任务产生的最终结果。

它可能是：

- 图像分类模型的类别概率
- 目标检测模型的边界框和类别
- 语义分割模型的像素标签
- 人脸、语音或图像特征向量
- LLM 的下一 Token 概率或 Hidden State
- 推荐模型的排序分数
- 生成式模型的去噪结果
- 多模态模型的 Embedding

输出通常会经过以下路径：

```text
Tensor Compute Engine
        |
        v
Post-Processing
        |
        v
Output Tensor Buffer
        |
        v
System Memory (DRAM)
        |
        v
CPU / Runtime / Next Accelerator
```

输出不一定总要回到 CPU。有些 SoC 会允许 NPU 输出直接交给 GPU、ISP、DSP、显示引擎或其他硬件模块，从而减少 DRAM 往返。

***

## 控制、调度与功耗管理

### Compiler Graph

`Compiler Graph` 表示神经网络的计算图及其编译结果。

一个模型在高层框架中通常表示为计算图：

```text
Input
  |
  v
Conv -> ReLU -> Pooling
  |
  v
Conv -> ReLU
  |
  v
GEMM -> Softmax
  |
  v
Output
```

NPU 编译器会分析该图，并进行：

- 算子划分
- 算子融合
- Tensor 分块（Tiling）
- 内存规划
- 数据布局选择
- 精度选择
- 任务调度
- Buffer 分配
- DMA 命令生成
- Compute Engine 指令生成

编译器决定了模型如何映射到具体硬件，因此即使两颗 NPU 峰值 TOPS 相近，编译器和 Runtime 的质量也可能导致实际性能差异很大。

***

### Task Scheduler

`Task Scheduler` 负责安排不同计算任务的执行顺序和资源分配。

它需要协调：

- 哪个 Compute Engine 执行哪个 Tile
- 哪个 DMA 通道负责搬运数据
- 哪些任务可以并行
- 哪些任务必须等待前序结果
- L2 SRAM 中哪些数据需要保留
- 不同模型或不同应用之间的资源竞争
- 低延迟任务和高吞吐任务之间的优先级

一个好的调度策略会尽量实现计算和数据搬运的重叠：

```text
时间轴：

DMA 搬运 Tile 0  |====|
Compute Tile 0        |====|
DMA 搬运 Tile 1            |====|
Compute Tile 1                 |====|
```

这种方式通常称为 **double buffering** 或 **pipeline overlap**。目标是避免 Compute Engine 等待数据，也避免 DMA 因为没有空闲 Buffer 而停顿。

***

### Performance Counters

`Performance Counters` 是硬件性能监控计数器。

它们用于帮助开发者、驱动和 Runtime 分析 NPU 的真实运行情况。常见指标包括：

- MAC Array 利用率
- Tensor Compute Engine Busy Cycle
- DMA 传输字节数
- DRAM 读取和写入带宽
- SRAM 命中率
- NoC 带宽利用率
- NoC 拥塞周期
- Cache / SRAM Stall Cycle
- Compute Stall Cycle
- 任务执行延迟
- 每层算子耗时
- 功耗和温度相关事件

例如，如果 MAC Array 利用率很低，但 DRAM 带宽很高，通常说明模型受内存带宽限制，而不是算力限制。

```text
高 MAC 利用率：
说明计算单元持续有数据可处理

高 DMA Stall：
说明数据搬运或内存带宽可能是瓶颈

高 NoC Congestion：
说明片上互连可能无法满足多 Tile 通信需求
```

这些计数器对于模型优化、编译器调优、驱动开发和硬件架构评估都非常关键。

***

### Clock / Power Gating

`Clock / Power Gating` 是 NPU 的节能机制。

NPU 通常包含大量计算阵列、SRAM、DMA 和互连资源。如果所有模块始终全速运行，会产生大量静态和动态功耗。因此硬件会动态关闭不需要使用的部分。

### Clock Gating

Clock Gating 会停止某些空闲模块的时钟切换。

例如：

```text
某个 Tensor Compute Engine 当前没有任务
        |
        v
关闭该模块的时钟输入
        |
        v
减少动态功耗
```

### Power Gating

Power Gating 更进一步，会切断某些空闲电路区域的电源。

优点：

- 节能效果更明显
- 适合长时间闲置的模块

代价：

- 唤醒需要时间
- 可能需要保存和恢复状态
- 控制逻辑更复杂

对于手机、笔记本、边缘设备和车载 SoC，功耗管理通常与 NPU 性能同样重要。一个高效的 NPU 目标不只是高 TOPS，还包括高 TOPS/W。

***

## 一个推理示例

下面以一个简化的图像分类网络层为例，说明数据如何流动：

```text
Input Image
    |
    v
System Memory (DRAM)
    |
    v
DMA loads image tile into Activation Buffer
    |
    v
DMA loads convolution weights into Weight Buffer
    |
    v
Tensor Compute Engine performs convolution
    |
    v
Accumulator Buffer stores partial sums
    |
    v
Vector / Activation Unit applies bias + ReLU
    |
    v
Post-Processing quantizes output to INT8
    |
    v
Output Tensor is written to DRAM
    |
    v
Next layer or Host CPU consumes the result
```

如果模型层之间可以在片上直接衔接，数据就不需要每层都往返 DRAM：

```text
更优的数据流：

Layer 1 Output
    |
    v
On-chip SRAM
    |
    v
Layer 2 Input
```

这通常会带来更低的延迟、更高的吞吐量和更好的能效。

***

## 关键设计目标

一个高性能 NPU 的设计通常围绕以下目标展开：

- **提高计算密度**：在有限芯片面积内集成更多 MAC 单元
- **提高数据复用**：让权重、激活值和部分和尽可能多次被使用
- **减少 DRAM 访问**：避免片外内存成为性能和功耗瓶颈
- **提高片上带宽**：让 SRAM、NoC 和 Compute Engine 能够持续供数
- **支持混合精度**：在精度、吞吐量、功耗和模型大小之间平衡
- **优化算子融合**：减少中间 Tensor 的写回和重新读取
- **提高可编程性**：支持不断变化的 CNN、Transformer、多模态和生成式模型
- **提升能效**：以更低功耗完成更多 AI 运算
- **增强可扩展性**：支持更多 Compute Tile、更大模型和更高并发任务

***

## 模块速查表

| 模块 | 核心作用 | 通俗理解 |
|---|---|---|
| Host CPU / Runtime | 提交与管理 AI 任务 | 项目经理和任务入口 |
| System Memory (DRAM) | 保存大容量模型与 Tensor | 大型仓库 |
| NPU Command Processor | 解析并执行任务命令 | 控制塔 |
| DMA / Tensor Data Mover | 在 DRAM、SRAM 和计算单元间搬运数据 | 自动搬运系统 |
| L2 Shared SRAM | 多个 Compute Tile 共享的高速片上存储 | 共享近端仓库 |
| Weight Buffer | 暂存模型参数 | 权重材料区 |
| Activation Buffer | 暂存输入和中间特征 | 当前工作数据区 |
| Accumulator Buffer | 保存部分和与高精度中间结果 | 计算草稿区 |
| Local SRAM | Compute Tile 私有高速存储 | 每个计算单元旁的工作台 |
| NPU Compute Cluster | NPU 的主要计算区域 | AI 计算车间 |
| Tensor Compute Engine | 执行矩阵、卷积和张量计算 | 核心生产线 |
| MAC Array / Systolic Array | 并行完成海量乘加 | 大规模乘法阵列 |
| Vector / Activation Unit | 处理激活、归约和逐元素算子 | 灵活的辅助计算单元 |
| NoC / Interconnect | 连接片上模块并传输数据 | 片上高速道路 |
| Post-Processing | 执行输出整理与轻量算子 | 成品处理区 |
| Quantization / Dequantization | 转换数据精度 | 压缩与还原工序 |
| Pooling / Elementwise Ops | 执行池化和逐元素运算 | 轻量加工步骤 |
| Output Tensor | 模型最终结果 | 最终成品 |
| Compiler Graph | 将模型映射为硬件执行计划 | 工艺规划图 |
| Task Scheduler | 安排计算、DMA 与资源 | 生产排程系统 |
| Performance Counters | 记录硬件利用率和瓶颈 | 性能仪表盘 |
| Clock / Power Gating | 关闭空闲硬件以节省功耗 | 智能节能系统 |


