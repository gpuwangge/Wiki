
## Bandwidth（带宽） vs. Throughput（吞吐量）
在 GPU 架构设计中，**Bandwidth（带宽）** 和 **Throughput（吞吐量）** 是两个核心但截然不同的性能指标。简单来说：**带宽决定了数据搬运的“车道宽窄与车速”，而吞吐量决定了单位时间内实际完成的“有效工作量”。**

### 1. Bandwidth（带宽）

* **定义**：指在单位时间内，数据在两个硬件组件之间（如从显存VRAM到L2缓存、或者从内存到计算单元）最大可以传输的数据量。
* **常见单位**：GB/s（吉字节每秒）或 TB/s。
* **决定因素**：主要由硬件物理规格决定。
  $$\text{带宽} = \text{显存频率 (Clock Speed)} \times \text{位宽 (Bus Width)} \div 8$$
  * 例如：GDDR6/GDDR7 的引脚速率、HBM 的多通道堆叠。
  * 显存位宽（如 256-bit, 384-bit, 512-bit 等）。
* **设计直觉**：带宽就像是**公路的车道数和限速**。车道越宽（位宽越大）、车速越快（频率越高），单位时间内能通过的车辆（数据）就越多。

### 2. Throughput（吞吐量）

* **定义**：指系统在单位时间内**实际处理**完成的任务量或数据量。在GPU中通常分为两类：
  * **计算吞吐量 (Compute Throughput)**：如 TFLOPS（每秒万亿次浮点运算），衡量算力单元的输出能力。
  * **数据/像素吞吐量 (Data/Pixel Throughput)**：如 Gtexel/s（纹理）、Gpixel/s（像素）、或是实际测得的有效传输速率。
* **常见单位**：FLOPS（计算）、Ops/s、GB/s（实际传输）。
* **决定因素**：受限于计算单元（如 SM、CUDA Core、Tensor Core）的数量、频率、指令执行效率，以及**软件算法的实现**。
* **设计直觉**：吞吐量就像是**收费站实际通过的汽车数量**。即便公路（带宽）再宽，如果收费员（计算核心）动作很慢，或者车辆本身没那么多，实际吞吐量也上不去。

### 3. 两者的核心区别与联系

| 维度 | Bandwidth（带宽） | Throughput（吞吐量） |
| :--- | :--- | :--- |
| **本质属性** | **硬件能力上限**（静态指标，设计时决定） | **实际工作效率**（动态指标，受负载与软件影响） |
| **类比** | 高速公路的**最大通行能力**（车道与车速） | 收费站实际每天**通过的车辆总数** |
| **瓶颈表现** | 导致 **Memory-Bound**（数据搬运成为拖累） | 导致 **Compute-Bound** 或整体性能饱和 |

#### 在 GPU 设计中的权衡（Roofline 模型视角）
在设计 GPU 时，工程师必须平衡这两者：
1. **算力与带宽的配比**：如果 GPU 的计算吞吐量（如 TFLOPS）极高，但显存带宽跟不上，计算单元就会经常处于“饥饿”状态（等待数据加载），这被称为**带宽瓶颈（Memory-Bound）**。
2. **AI 与大模型时代**：像大语言模型推理这类任务，由于每一步计算需要从显存读取大量的权重参数，对带宽的要求极其苛刻。这也是为什么高端 AI GPU（如 NVIDIA H100/B200）不惜采用成本高昂的 **HBM（高带宽内存）** 来大幅拉高带宽，从而提升整体的有效吞吐量。

### 4. 为什么计算 Throughput 为什么要统计指令数目？

在计算 GPU 或处理器的 **Throughput（吞吐量）** 时统计指令数目（Instructions），主要是因为指令是度量**计算有效载荷（Workload）**和**微架构执行效率**最精确的尺度。单纯看运行时间或传输的数据量，无法准确反映硬件内部算力的利用率。

#### 1. 指令是“计算工作量”的基本度量单位
* **消除数据大小与类型的混淆**：如果只用“处理了多少 GB 数据”来衡量吞吐量，会忽略不同算法的复杂度。例如，加载 1KB 数据可能只需要做 1 次简单的加法，也可能需要做 1000 次复杂的超越函数运算。
* **直接映射硬件资源**：在 GPU 架构设计中，吞吐量通常用 **IPC (Instructions Per Cycle)** 或 **FLOPS (Floating-point Operations Per Second)** 来衡量。一条指令（如 MAD、FMA、Tensor Core 的矩阵乘加指令）代表了硬件发射和执行了一个确定的计算任务。

#### 2. 精准评估硬件利用率（Performance Bottleneck Analysis）
通过统计执行的指令数目，架构师可以结合时钟周期计算出关键性能指标：
$$\text{Throughput (Instructions/Cycle)} = \frac{\text{Total Instructions}}{\text{Total Cycles}}$$
* **暴露流水线停顿（Stalls）**：如果硬件理论上每个周期能发射 4 条指令，但实际统计发现每周期平均只有 0.5 条指令在执行（Throughput 极低），说明计算单元大部分时间处于**饥饿状态**（例如在等待显存返回数据，即 Memory Stall）。
* **量化算术强度（Arithmetic Intensity）**：通过统计指令中的浮点运算指令（FLOPs）与访存指令（Bytes）的比例，可以直接代入 Roofline 模型，判断当前负载是受限于算力（Compute-bound）还是受限于带宽（Memory-bound）。

#### 3. 指令类型不同，吞吐量权重不同
在 GPU 中，不同指令消耗的硬件资源和执行周期截然不同：
* **ALU 指令**（如标量/向量加减乘除）：吞吐量极高，每个周期可大量并发。
* **SFU 指令**（特殊函数单元，如求倒数、三角函数）：占用专属硬件，吞吐量较低。
* **Memory Load/Store 指令**：虽然也算指令，但它们的吞吐量受限于缓存命中率和显存带宽。

统计指令数目并将其按类型细分（Instruction Mix），能让设计人员清楚地知道：**是哪种指令占用了最多的周期？哪类硬件单元成为了性能瓶颈？**从而指导下一代 GPU 的架构优化（例如：该增加 FP32 ALU 的数量，还是优化 L1 缓存的命中率）。


## GPU 吞吐量计算算法解析 (Throughput Calculation Algorithms)
 GPU 性能模型中的两个核心吞吐量计算算法：**内存延迟转换为 GPU 周期** 与 **ALU 吞吐量计算**。  
 这两个公式是 GPU 硬件性能建模与抽象吞吐量计算中的核心模块，主要用于将硬件底层的物理指标转化为统一的性能评估指标。  

### 内存延迟转换为 GPU 周期 (Memory Latency to GPU Cycles)

#### 1. 公式与计算逻辑

该模块的核心逻辑是将外部内存（DDR）的传输延迟折算为 GPU 的时钟周期数，用于模拟带宽受限场景下的流水线停顿或传输开销。计算步骤如下：

*   **单拍字节数**：根据 AXI 总线位宽计算每拍传输的字节数。
    $${beats size bytes} = \frac{axi width}{8}$$
*   **总传输字节**：结合 DDR 拍数和 L2 缓存切片数计算总访存量。
    $${total bytes} = {ddr beats} \times {beats size bytes} \times {num l2s}$$
*   **传输时间**：利用标定后的 DDR 带宽计算实际传输耗时。
    $${transfer time sec} = \frac{total bytes}{bandwidth bps}$$
*   **周期换算**：将耗时乘以 GPU 顶峰运行频率（Top Frequency）得到对应的 GPU 周期数。
    $${gpu cycles} = {transfer time sec} \times {top freq hz}$$

#### 2. 核心作用与应用场景

*   **定量评估访存瓶颈**：通过将外部 DDR 访存的数据量、AXI 总线位宽和标定带宽转化为 ${gpu cycles}$，能够精确模拟当 GPU 发生缓存未命中（Cache Miss）或存在大量访存时，流水线需要等待的时钟周期数。
*   **硬件带宽约束建模**：算法中对带宽进行了硬编码上限设定（如最高限制在 55 GB/s），这用于模拟实际芯片设计中受限的内存通道带宽，避免理想化计算导致过高估计硬件性能。

### ALU 吞吐量计算 (ALU Throughput)

#### 1. 公式与计算逻辑

该模块的核心逻辑是基于硬件指令计数器（Instruction Counters）和不同功能单元的硬件开销权重，统计总体 ALU 计算吞吐量和资源占用。计算步骤如下：

*   **FMA 吞吐**：乘加指令权重为 $0.5$，分摊到各个子核（${num sc}$）并考虑异步发射比（${async ratio}$）。
    $$fma = \left( \frac{{EXEC INSTR FMA} \times 0.5}{num sc} \right) \times {async ratio}$$
*   **CVT 吞吐**：数据类型转换指令，引入架构特定的微架构因子（如 ${cvt pe}$）。
    $$cvt = \left( \frac{{EXEC INSTR CVT} \times {cvt pe}}{num sc} \right) \times {async ratio}$$
*   **MSG 吞吐**：消息/访存交互指令，权重为 $1.0$。
    $$msg = \left( \frac{{EXEC INSTR MSG} \times 1.0}{num sc} \right) \times {async ratio}$$
*   **SFU 吞吐**：特殊函数单元（如超越函数等）计算复杂度较高，权重设定为 $4.0$。
    $$sfu = \left( \frac{{EXEC INSTR SFU} \times 4.0}{num sc} \right) \times {async ratio}$$
*   **总 ALU 开销**：累加所有功能单元的归一化开销。
    $$alu\_total = fma + cvt + msg + sfu$$

#### 2. 核心作用与应用场景

*   **异构指令开销归一化**：GPU 执行的指令类型繁杂（如乘加、类型转换、消息交互、特殊函数），它们的硬件执行周期各不相同。该公式通过给不同指令赋予特定的权重因子，将复杂的指令计数器折算为一个可统一比对的总体吞吐量消耗。
*   **微架构差异适配**：通过引入架构特定的 PE 因子，该算法能够灵活适配不同代际 GPU 内部子核的硬件微架构吞吐差异，从而实现高层抽象模拟器对多种不同硬件配置的兼容。


## Little’s law
Little’s law是排队论中的一个关系：在稳定运行的系统里，平均未完成数量 = 平均吞吐率 × 平均停留时间。  
Little’s law 只统计请求待了多久，不要求请求一进来就开始处理。    

一个直观例子   
假设一家咖啡店： 平均每分钟进来 2 位顾客。  
每位顾客进店到离店的时间，平均5 分钟。这五分钟包括排队时间和实际处理时间。  
那么店里平均有： 2 人/min × 5 min = 10 人。  
这里的 10 人包括排队、点单、等待咖啡等所有尚未离店的人，不是“同时正在被服务的人”。这正对应 outstanding 包含等待与处理中的请求。  

套到内存读取   
假设每秒完成 10^9 笔读取。(单位时间进来的顾客数)  
每笔平均延迟为 100 ns。（顾客呆在店里的时间，包括了排队和处理）  
Outstanding Request: N = 10^9 笔/s × 100 × 10 ^(-9) s = 100 笔 (店里平均人数)  
也就是说，要维持这个完成速率，系统平均需要有 100 笔请求尚未完成。  



## Latency-Throughput-Bandwidth
1. bandwith是单位时间内最多能传的数据，例如 GB/s，而不是总数据量；
2. throughput是单位时间内实际传的数据；
3. latency是单笔请求从发出或被接受，到完成的耗时；

因为有latency的存在，所以会有outstanding request。请求从发出到完成期间就是 outstanding；多笔同时 outstanding 还需要系统允许重叠执行。  
系统里最大的outstanding request的数量是受硬件限制的。实际能维持多少，还受到程序依赖和独立请求数量的限制。  
所以如果latency很大的话，outstanding request达到最大值，这时候可能throughput达不到bandwidth。 有限并发可能不足以覆盖等待时间。  
要让throughput达到bandwidth，latency要小，或者要有足够并发来覆盖它。  

Latency-Throughput-Bandwidth三者的制约关系  
1. 比如Bandwidth是64GB/s
2. Latency是200ns，可维持的 outstanding 数=100, 每秒完成读取：100/200ns= 0.5G/s
3. Throughput计算：每笔请求 64 B, 每秒0.5G笔，总共32GB/s

这个例子中，Throughput32GB/s < Bandwidth64GB/s, 说明跑不满。如果要跑满，要么Lantency下降为100ns；或者Outstanding数提升为200.  


## 练习题1
某存储器接口每个 Beat(数据节拍)传输 8 Bytes(64-bit 总线)，主频与数据频率均为 2 GHz。发读命令至第一笔数据返回的延迟为 400 ns。读取 64 KB 数据时：  
1. 该读操作的 Latency 为多少？
2. 在等待 Latency 期间，内存控制器最多可维持多少个 Outstanding Requests？

解答
1. Latency = 400ns 题目已描述为“首笔数据返回所需时间”，即命令到首字节数据的通道延迟。  
但如果要计算至最后一个beat完成的时间，则  
总transaction数量为 64KB / 8B = 8K  
单次transaction传输时间 = 1/f = 1/2G=0.5ns  
全部传输完成需要时间为 8K * 0.5ns = 4096ns  
所以所有数据的读操作总耗时4496ns  

2. 最大 Outstanding Requests 无法由现有条件确定。800 只有在“一周期发一个独立请求”等额外假设下才成立；一个 Beat 不能直接等同于一个 Request。  
Max Outstanding = 400 ns / 0.5 ns per-issue = 800 个并发请求


## 练习题2
Latency / Bandwidth / Transfer Time 联动(进阶)  

内存控制器参数:  
f = 1 GHz, Beat SIZE = 8B, BURST = INCR, LEN = 7(8 Beats)。读操作延迟 = 400 ns。  
问:  
1. 单次 Transaction 的 Transfer Time 是多少？理论峰值带宽是多少？
2. 读取 64 KB 数据，若始终保持流水线满载(Outstanding 足够)，总耗时最少为多少？
3. 为掩盖 400 ns 延迟，最少需要多少个 Outstanding Transaction？
 
解析:  
1. 首先每周期传输时间为1/f=1/1G=1ns, 理论峰值带宽为： 8B/1ns=8GB/s  
Transfer有8 beats，则Transaction transfer time = 8ns

2. 假设每周期一个beat，每个beat 8B，每个transaction 8B，每个transaction数据量64B  
读取64KB数据，Transaction数量 = 64KB/64B=1K  
每个Transaction transer time = 8ns, 总传输时间为1k*8ns=8192ns  
因为latency=400ns，总耗时400+8192=8592ns  

3. 首先Latency=400ns，一个transaction transer time要8ns完成；也就是说第一个transaction发出第一个beat算起，要408ns。  
题目问的是outstanding transaction而不是outstanding request，所以  
最少需要408/8=51个outstanding transaction  


## Burst:
是在传输/片上互连协议(如 AXI、PCIe、DDR、UCIe 等)中的标准术语，  
意思是该接口是否允许在单条读写命令下，自动连续传输多个数据块(Beats)。  


### 什么是 Burst(突发传输)
非突发(Non-Burst / Single Transfer): 每传一个数据块(Beat)，主控都必须单独下达一次命令和地址。  
例如传 64 Bytes(8 个 Beat)，需要 8 次独立请求。  

突发模式(Burst Mode):  
主控只下发一条命令和起始地址，存储器/接口会自动按预设规则(递增、循环等)连续传送 N 个 Beat，无需主控重复发号。  
这个 N 就是 Burst Length(BL)。  

举例:  
若 Burst Length = 8，且每次传 8 Bytes，则：  
1 个 Transaction = 8 个 Beats = 64 Bytes  

### 为什么需要 Burst？  

| 维度 | 无 Burst（每次 1 Beat） | 有 Burst（如 BL=8） |
|---|---|---|
| **命令/地址开销** | 每次传数据时都要再发命令，总线很闲散/碎片化 | 命令开销被摊薄到整个 Burst，数据传输占比大幅提升 |
| **带宽利用率** | 理论带宽 = 8B × 2GHz = **16 GB/s**（假设每拍传输） | 理论带宽 = 8B × 2GHz = **16 GB/s**（持续满速传输） |
| **主控负载** | 需频繁发指令，CPU/控制器压力高 | 批量下发，调度更容易 |
| **典型应用场景** | 随机访问、延迟敏感型控制操作 | 顺序读取/写入、图像/视频、大模型权重搬运 |


对于相同的数据量，实际使用多 Beat burst 会减少 transaction 总数，Beat 总数不变。  
比如64KB数据和8B beat，最多发64KB/8=8K transaction  
如果burst len=7(8个beat)， 则transaction数量为1k  


