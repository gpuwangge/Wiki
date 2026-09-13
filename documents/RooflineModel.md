# Roofline Model（屋顶模型）

**Roofline Model** 是一种直观且强大的性能评估与分析模型，由 Samuel Williams 等人在 2009 年提出。它主要用于评估计算机体系结构（如 CPU、GPU、TPU 等）上给定内核或算法的性能上限，并指导代码优化方向。

该模型的核心思想非常直观：**算法在硬件上能达到的最大性能，受限于硬件的计算峰值能力以及内存带宽。** 它的图形化表现形式酷似屋顶，因此得名“屋顶模型”。

## 一、 核心概念与关键指标

在理解 Roofline Model 之前，必须先明确以下三个核心硬件与算法指标：

### 1. 浮点运算性能 (Performance)

* **定义**：单位时间内执行的浮点运算次数。

* **单位**：GFLOPS（每秒十亿次浮点运算）或 TFLOPS（每秒万亿次浮点运算）。

* **计算公式**：
  

  $$
  P = \frac{\text{Flops}}{\text{Time}}
  $$

### 2. 计算强度 (Arithmetic Intensity / Operational Intensity)

* **定义**：每个字节的内存访问所对应的浮点运算次数。它是连接计算与访存的桥梁。

* **单位**：FLOPs/Byte（每字节浮点运算次数）。

* **计算公式**：
  

  $$
  I = \frac{\text{Total Flops}}{\text{Total Memory Access (Bytes)}}
  $$

### 3. 硬件峰值指标

* **峰值计算性能 (**$P_{max}$**)**：硬件在理想状态下每秒能执行的最大浮点运算次数（通常由硬件的 ALUs 数量和频率决定）。

* **峰值内存带宽 (**$B_{max}$**)**：硬件主存（如 DRAM、HBM）与处理器之间每秒能传输的最大数据量。

## 二、 Roofline Model 的数学原理与公式

Roofline Model 的性能上限由一条分段函数（即“屋顶”形状的曲线）决定：

$$
\text{Attachable Performance} = \min \left( P_{max}, \quad I \times B_{max} \right)
$$

根据计算强度 $I$ 的不同，整个性能空间被划分为两个截然不同的区域：

### 1. 内存带宽瓶颈区 (Memory-Bound / Bandwidth-Bound)

* **条件**：当算术强度 $I$ 较小，即 $I < I_{crit}$ 时。

* **表现**：此时处理器的计算单元（ALU）经常处于“饥饿”状态，因为数据从内存传输到处理器的速度跟不上计算速度。

* **上限公式**：
  

  $$
  \text{Performance} = I \times B_{max}
  $$

* **优化目标**：提升计算强度 $I$（例如通过循环分块、算子融合等技术减少访存）。

### 2. 计算能力瓶颈区 (Compute-Bound)

* **条件**：当算术强度 $I$ 较大，即 $I \ge I_{crit}$ 时。

* **表现**：此时内存带宽已经能够满足数据供给，处理器的计算单元全负荷运转，达到了硬件的极限。

* **上限公式**：
  

  $$
  \text{Performance} = P_{max}
  $$

* **优化目标**：通过指令级并行、向量化、降低指令复杂性等手段提升硬件本身的 $P_{max}$。

### 3. 拐点 (Crest / Critical Intensity)

* **定义**：内存瓶颈区和计算瓶颈区的交界点，称为**临界计算强度 (**$I_{crit}$**)**。

* **计算公式**：
  

  $$
  I_{crit} = \frac{P_{max}}{B_{max}}
  $$

* **物理意义**：该值标志着硬件设计中“计算能力”与“访存带宽”的平衡点。

## 三、 Roofline Model 的可视化图表

在对数坐标系（Log-Log Scale）下，横轴为计算强度 $I$（$\log$ 尺度），纵轴为性能 $P$（$\log$ 尺度）。

```
Log(Performance)
  ^
  |                                        +---------------------------------- (Compute Peak: P_max)
  |                                       / 
  |                                      /  <- Compute-Bound Region
  |                                     /
  |                                    /  <-- 拐点 (Crest: I_crit = P_max / B_max)
  |                                   /
  |                                  /
  |   Memory-Bound Region           /
  |  (Slope = Bandwidth: B_max)    /
  |  /                            /
  | /                            /
  |/                            /
  +------------------------------------------------------------------------> Log(Arithmetic Intensity)

```

在这个图表中：

* 左侧斜线代表 $P = I \times B_{max}$。

* 右侧水平线代表 $P = P_{max}$。

* 实际内核（Kernel）在图上表现为一个点。如果点落在斜线上，说明受限于带宽；如果落在水平线上，说明受限于计算。

## 四、 深度学习与高性能计算中的应用

在现代深度学习模型和高性能计算（HPC）中，Roofline Model 是性能调优的金标准。

### 1. 常见深度学习算子的 Roofline 特征

| **算子类型** | **计算强度特征** | **主要瓶颈** | **典型优化手段** |
| :--- | :--- | :--- | :--- |
| **Pointwise (如 ReLU, Element-wise 加法)** | 极低 | 内存带宽 (Memory-Bound) | **算子融合 (Operator Fusion)**：减少中间结果的访存。 |
| **Pooling / Reduction** | 较低 | 内存带宽 (Memory-Bound) | 优化数据布局，提升缓存命中率。 |
| **Matrix Multiplication (GEMM / Conv2D)** | 很高 (随输入规模增大而增大) | 计算能力 (Compute-Bound) | 使用 Tensor Core / 硬件加速、分块 (Tiling)、循环展开。 |

### 2. 经典 Roofline 案例分析：大模型与矩阵乘法

在大语言模型（LLM）的训练和推理中：

* **训练阶段（Training）**：批量大小（Batch Size）通常较大，GEMM 占主导，计算强度高，系统通常处于**计算能力瓶颈区**，核心优化方向是利用硬件的矩阵张量核心（如 Tensor Cores、AMX）。

* **推理阶段（Inference - Batch Size = 1）**：生成阶段每生成一个 Token 都要从显存中加载整个模型的权重，导致计算强度极低，系统严重陷入**内存带宽瓶颈区**。这也是为什么大模型推理极其依赖高带宽内存（如 HBM）以及量化技术（Quantization，减少权重传输字节数）。

## 五、 进阶：Extended Roofline Model（扩展屋顶模型）

传统 Roofline 模型主要针对“主存带宽”，但在实际硬件中，存在多级存储层次（L1 Cache、L2 Cache、DRAM）。为了更精确地诊断问题，演化出了**扩展屋顶模型**。

* **多级屋顶**：考虑 L1/L2 缓存的极高带宽。如果算法的数据集能完全被缓存容纳，其可达性能上限会跃迁到一个更高的“屋顶”。

* **多精度 Roofline**：现代硬件（如 NVIDIA GPU）对不同数据精度（FP64, FP32, FP16, INT8, FP8）提供的峰值性能 $P_{max}$ 呈指数级倍增。扩展模型允许针对不同精度绘制多条水平屋顶线。

## 六、 总结与优化路线指导

通过 Roofline Model，开发者可以清晰地回答以下三个问题：

1. **我的代码性能有提升空间吗？**（对比当前的实际性能与屋顶上限）。

2. **瓶颈在哪里？**（在斜线上还是在水平线上）。

3. **下一步该怎么优化？**

   * 若处于**内存受限区**：不要盲目去优化算法的浮点计算逻辑，而应当专注访存优化（如数据重用、减少内存分配、内存对齐、算子融合）。

   * 若处于**计算受限区**：应当专注提高计算效率（如向量化指令、减少分支预测失败、利用专用硬件加速单元）。



# Roofline Model举例：Predicted GPU Active 
Predicted GPU Active Roofline模型核心逻辑基于：GPU 的整体性能上限由耗时最长的子系统（即最慢环节）决定。  

**模型公式**  
```c
predicted_gpu_active = max( max(tex, blend, alu, asn, quad, lsc, rtu), // Shader bound
                            max(l2, ddr), // Memory bound
                            tiler // Geometry bound
                          ) + csf
```

子系统分类与瓶颈分析
- Shader Bound (着色器瓶颈): max(tex, blend, alu, asn, quad, lsc, rtu)
评估着色器核心内部的计算与局部数据处理极限。涵盖了纹理映射 (tex)、混合 (blend)、算术逻辑单元 (alu)、加载/存储控制 (lsc)、光线追踪/渲染目标 (rtu) 以及其他核心级操作 (asn, quad)。
- Memory Bound (显存/内存瓶颈): max(l2, ddr)
确定存储层级的带宽限制，对比 L2 缓存 (l2) 传输限制与外部 DDR 内存 (ddr) 的带宽消耗，取其最大值。
- Geometry Bound (几何瓶颈): tiler
代表几何处理流水线中的约束，特别是基于分块渲染 (Tile-based rendering) 架构中的 Tiler 处理开销。
- CSF (命令流前端开销): + csf
Command Stream Frontend (命令流前端) 的开销独立于并行流水线的 max() 比较。它作为线性的命令调度与分发开销，直接叠加在底层硬件的并发瓶颈时间上。

架构评估逻辑  
- 模型首先在三个主要硬件域（Shader、Memory、Geometry）内部计算出最大执行时间或周期成本。
- 对比这三个域的最大值，找出全局并发执行时的绝对瓶颈（即重叠执行后暴露的最长关键路径）。
- 最后将 CSF 带来的前端串行指令调度开销附加到全局瓶颈之上，得出最终的 predicted_gpu_active 预测活跃周期。


什么是 Bottleneck？  
在 GPU 分析性能模型（A-Model）中，Bottleneck（瓶颈周期数） 指的是某个特定硬件子系统在处理完给定工作负载时，所需要消耗的理论最小时钟周期数（Cycles）。  
在 Roofline 性能模型中，模型假设各个硬件模块（如 ALU、Texture、L2 Cache、DDR 等）在理想状态下是完全并行重叠（Overlap）执行的。  
此时，整个系统或子系统的最终执行时间，取决于耗时最长的那个硬件模块。  
模型中计算出的每一个 ${Subsystem}$ 数值，代表该硬件单元“在吞吐量受限下独自完成工作所需的周期上限”。因此在代码和公式定义中，直接将这些模块算出来的周期数命名为该模块的 Bottleneck（瓶颈）。  

以 ALU 计算公式为例：  
$${ALU} = 0.5 \times {EXEC INSTR FMA} + 0.5 \times {EXEC INSTR CVT} + 1.0 \times {EXEC INSTR MSG} + 4.0 \times {EXEC INSTR SFU}$$  

把各类指令乘以各自系数后相加，本质上是在做硬件资源消耗的量纲转换与时间累加：

量纲统一（指令数 $\rightarrow$ 周期数）：
- EXEC_INSTR_x 的单位是指令数（Instruction Count）。
- 前面的系数（0.5, 1.0, 4.0）单位是指令周期倒数（Cycles / Instruction），代表硬件管线的发射/执行能力。
- 例如：FMA 硬件发射吞吐是 2 ops/cycle，因此 1 条 FMA 占用 $1 / 2 = 0.5$ 个周期；SFU 属于慢速超越函数（Transcendental Function）管线，1 条 SFU 指令需要占用 4 个周期。
- 指令数乘以系数后，消去了“指令”单位，统一变成了周期数（Cycles）。

ALU 硬件管线的时间累加
- 在 ALU 算术逻辑单元内部，各类指令在流水线上按发射吞吐依次消耗周期。将它们乘系数后的结果相加，算出的总和就是：ALU 硬件单元把这批指令全部执行完所需要的总时钟周期数。

参与 Roofline 的 Bottleneck 竞争
- 计算出的 ALU 周期总数，会被送入 Shader Core 的顶级选大器（MAX 函数）：  
$${Shader Core} = {Async Ratio} \times \max({ALU}, {Texture}, {Blend}, {RTU}, \dots)$$
- 如果算出来的 ALU 周期数高于 Texture 或 Blend，那么 ALU 的计算能力就成为了限制 Shader Core 性能的真实主导瓶颈（Dominant Bottleneck）；反之，若 Texture 周期更大，ALU 的周期数就只是一个潜在瓶颈指标。

因此，这里的 ALU 公式不是单纯在数指令，而是计算ALU 硬件单元的瓶颈执行周期

该模型主要通过从 Emulator/模拟器采集的硬件计数器（Hardware Counters）数据，预测 GPU 执行周期、识别系统性能瓶颈、评估 Cache 命中率以及计算帧率（FPS）。

## 1. 顶层性能预测模型（Main Performance Model）

A-Model 采用了基于 **Roofline** 的瓶颈分析范式。GPU 的总活跃周期（`Predicted_GPU_ACTIVE`）由微控制器（MCU）的串行开销与各并行处理单元中的**最大瓶颈周期**相加得到：

$${Predicted GPU ACTIVE} = {MCU ACTIVE} + \max ( {Shader Core Bottleneck}, {Tiler Bottleneck}, {L2 Cache Bottleneck}, {Memory Bottleneck})$$

* **${MCU ACTIVE}$**：前端微控制器/主机命令处理器的串行固定开销。
* **Pipeline Bottleneck Net**：主执行流水线遵循“木桶效应”（$\max$ 运算符），即整体性能由最慢的硬件资源瓶颈决定。

## 2. 核心子系统计算公式

### 2.1 着色器核心瓶颈（Shader Core Bottleneck）

Shader Core 的瓶颈周期由内部各子模块的最大周期决定，并通过 `Async_Ratio` 进行跨时钟域归一化：

$${Shader Core} = {Async Ratio} \times \max 
{Texture Bottleneck}, \\
{Blend Bottleneck}, \\
{Rasterizer Bottleneck}, \\
{ASN Bus Bottleneck}, \\
{ALU Bottleneck}, \\
{RTU Bottleneck}, \\
{LSC L1 Cache Bottleneck}
)$$

其中时钟频率异步比率（Async Ratio）公式为：

$${Async Ratio} = \frac{{CSF Freq}}{{SC Freq}}$$

* **${CSF Freq}$**：核心系统频率（Core System Frequency，MHz）。
* **${SC Freq}$**：着色器核心频率（Shader Core Frequency，MHz）。

### 2.2 ALU 计算瓶颈（ALU Bottleneck）

ALU 瓶颈由各类指令的执行次数乘以其对应的单指令周期系数（Issue Latency）累加得到：

$${ALU} = 0.5 \times {EXEC INSTR FMA} + 0.5 \times {EXEC INSTR CVT} + 1.0 \times {EXEC INSTR MSG} + 4.0 \times {EXEC INSTR SFU}$$

指令权重系数说明：  

| 指令类型 | 周期系数（Cycles/Inst） | 硬件含义与吞吐说明 |
| :--- | :---: | :--- |
| **FMA** (Fused Multiply-Add) | `0.5` | 融合乘加指令（等价于 2 ops/cycle 吞吐） |
| **CVT** (Conversion) | `0.5` | 数据类型转换指令 |
| **MSG** (Message) | `1.0` | 核心间通信与消息同步指令 |
| **SFU** (Special Function Unit) | `4.0` | 特殊功能单元指令（如 $\sin, \cos, \log, \sqrt{x}$ 等慢速超越函数） |

### 2.3 内存子系统瓶颈（Memory Bottleneck）

内存瓶颈综合评估了系统级缓存（SLC）的总线传输效率与外部 DRAM 带宽限制：

$${Memory Bottleneck} = \max({SLC Bottleneck}, {DDR Bottleneck})$$

- SLC 瓶颈计算公式  
$${SLC Bottleneck} = {Num L2} \times (\frac{10^{-9}}{150}) \times (\frac{{AXI Width}}{8}) \times ({CSF Freq} \times 10^6) \times ({L2 EXT READ BEATS} + {L2 EXT WRITE BEATS})$$

- DDR 瓶颈计算公式  
$${DDR Bottleneck} = ({CSF Freq} \times 10^6) \times (\frac{10^{-9}}{{DDR BW}}) \times ({DRAMC R BYTE} + {DRAMC W BYTE})$$

## 3. Cache 命中率计算（Cache Hit Rate Calculations）  

各级缓存的命中率评估指标如下表所示：

| 缓存类型 | 层级 / 目标 | 计算公式 | 说明 |
| :--- | :--- | :--- | :--- |
| **LSC (Load Store Cache)** | L1 Cache 命中率 | $\frac{{LSC READ HIT}}{{LSC READ HIT} + {LSC LINE FILL}}$ | L1 读命中数占总读与 Fill 次数的比例 |
| **LSC (Load Store Cache)** | L2 Cache 命中率 | $1 - \frac{{BEATS RD LSC EXT}}{{BEATS RD LSC}}$ | $1 - {外部总线读 Beat 占比}$ |
| **Texture Cache** | L1 纹理缓存命中率 | $1 - \frac{{TEX TPCH NUM PARKED MISS}}{{TEX TPCH NUM PARKED PASSES}}$ | $1 - {挂起 Miss 占总 Pass 的比例}$ |
| **Texture Cache** | L2 纹理缓存命中率 | $1 - \frac{{BEATS RD TEX EXT}}{{BEATS RD TEX}}$ | $1 - {纹理外部读 Beat 占比}$ |

## 4. 帧率（FPS）计算与误差分析  

根据系统时钟频率与 GPU 活跃周期，计算实际帧率（Golden FPS）、预测帧率（A-Model FPS）以及相对误差：

* **实际帧率 (Golden FPS)**：
  $${Golden FPS} = \frac{{CSF Freq} \times 10^6}{{GPU ACTIVE}}$$

* **预测帧率 (A-Model FPS)**：
  $${Model FPS} = \frac{{CSF Freq} \times 10^6}{\sum {Predicted GPU ACTIVE per segment}}$$

* **相对误差率 (Error Rate)**：
  $${Error} = \frac{|{Golden FPS} - {Model FPS}|}{{Golden FPS}} \times 100\%$$

## 5. 瓶颈自动识别算法（Bottleneck Identification）  

算法通过计算各硬件组件在 GPU 活跃时间中的占比（置信度），提取出排名前列的主导瓶颈：

```javascript
function identifyBottleneck(formula) {
    // 1. 提取各硬件组件的周期数值
    const bottlenecks = { mcu, tiler, l2, slc, ddr, tex, blend, alu, rtu, lsc };

    // 2. 计算各组件的置信度 (Confidence)，上限封顶为 0.99
    for (const [component, value] of Object.entries(bottlenecks)) {
        component.confidence = Math.min(0.99, value / gpu_active);
    }

    // 3. 按置信度降序排序，获取最高置信度值
    const sorted = Object.values(bottlenecks).sort((a, b) => b.confidence - a.confidence);
    const top_confidence = sorted[0].confidence;

    // 4. 筛选并返回所有达到最高置信度 75% 以上的主要瓶颈组件
    return sorted.filter(item => item.confidence >= 0.75 * top_confidence);
}
```

## 6. 总结与架构启发

1. **分层 Roofline 拓扑**：模型从最底层的存储/算力单元（SLC、DDR、ALU、TEX）到 Shader Core，再到顶层 GPU，均采用了多层级的 `MAX()` 取极大值逻辑，准确捕捉单点硬件瓶颈对系统吞吐的制约。
2. **异步时钟域解耦**：引入 `Async_Ratio` 参数，完美屏蔽了 CSF（系统时钟）和 SC（Shader Core 时钟）在 DVFS（动态频率缩放）下的频率差异。
3. **容错性瓶颈诊断**：瓶颈识别算法设置了 `0.75 * top_confidence` 的相对阈值，能够同时揭示主瓶颈及紧随其后的次要瓶颈，为性能优化提供更全面的指引。


