# GPU设计原理
**`GPU和CPU设计上的区别`**：GPU设计目标是最大化吞吐量(Throughout), 关心并行度(Parallelism)。  
CPU更关心延迟(Latency)和并发(Concurrency)。  
**`并行`**：同时处理多个相同任务。  
**`并发`**：处理多个任务，但不是同时。 
**`显存`**：GPU里面独立的内存。HBM(High Bandwidth Memory)，通过PCIe与CPU内存通讯。  
**`GPU缓存Cache机制`**：目的是为了减少内存(显存，Latency=15x, B/W=1x)的时延。  
**`GPU寄存器`**：通常把GPU寄存器regs也当作缓存(L0，Latency=?, B/W=?)。regs距离SM非常近。因为SM是实际的计算单元，所以希望尽快的获取数据。    
同时有一些缓存(L2, Latency=5x, B/W=3x)希望离显存更近已方便读取内存的数据。  
SM本身也有专属的缓存(L1, Latency=1x, B/W=13x)。因此GPU实际会设计多级缓存机制。  
作为对比，GPU外部传输的PCIe速度为Latency=25x, B/W=0.02x，完全跟不上GPU内部的带宽传输速度，会拖累计算。  
**`计算强度`**：一个字节如果参与了8个时钟周期的计算，那么它的计算强度就是8。每个算法都有其计算强度。这个值越低就表示此算法越受制于带宽。L1 Cache适合计算强度8的算法，L2 Cache 39, HBM 100...  
换句话说，L1 Cache很容易饱和从而提高利用率。PCIe的计算强度高达6240，因此很难进行饱和计算，计算利用率也就不高了。  
当GPU程序(kernel)被执行的时候，大量的线程被分配到不同的SM上，但是每同一个block的thread都只会被分配到同一个SM上。  
SM上有若干个CUDA Core，每个CUDA Core上又有若干Warp（线程束）。Warp是进行线程调度的基本单元。Warp的size一般为32。  
也因此，一般kernel代码里把block的size设计为32的倍数。  

## 举例
Intel Exon 8280: Memory Bandwidth = 131 GB/sec, Memory Latency = 89ns, 则在89ns内理论可以传输的数据量是131*89=11659 bytes。  
实际在一个乘加运算里只移动了16 bytes，则Memory Efficency = 16/11659 = 0.14%  
NVIDIA H100的Memory Efficency更低。  
如果用并发展开，一次可以执行更多指令，但是有约束。  
如果使用并行展开，需要使用大量线程。  
GPU的DRAM Latency比CPU大数倍(数据搬运延迟更大)，但线程数比CPU多上百倍。 
结论：GPU的多线程架构是为大量任务并行设计的。  

# AI计算原理
## Convolutional Computation
在图像处理中，每一个卷积核需要跟图片里对应的元素相乘再相加，最后获得特征图。  
实际计算的时候，会对图片元素进行重排，再把卷积核也进行重排，这样就得到了两个大矩阵。通过矩阵相乘计算的方式获得特征图矩阵。最后把特征图矩阵恢复成特征图。  
## GPU Thread Hierarchical
在AI计算中，并不是所有计算都是线程独立的。  
如果运算元素之间相互独立，那确实可以把所有的线程并行计算。  
但实际计算中某些元素的计算依赖于周边数据的配合，因此不能完全线程独立。  
**`Grid`**：所有的线程组成的任务系统。  
**`Block`**：Grid中的一些任务线程组成Block，Block中的线程都是独立执行的，可以通过本地数据共享同步交换数据(via local memory)。  
Grid/Block的本质是将线程进行分层。  
Memory/Cache的本质也是将内存进行分层。  
## Algorithmic Efficency
以矩阵乘法为例，矩阵A的某一行乘以矩阵B的某一列，其中涉及一系列元素的并行乘加操作，可以在同一个Block内实现。  
随着矩阵size的增加，此算法计算强度也会线性增加。  
GPU设计不仅仅关注算力。还需要关注算力和内存，带宽和时延的匹配。  

# NVidia GPU Architecture
## CUDA Core
CUDA Core是指一个执行基础运算的处理单元。  
CUDA Core数量通常是对应FP32计算单元的数量。  

## RT Core
在消费级显卡里处理光线追踪所添加的核心。  

## Tensor Core
用于机器学习加速的运算核心。  
可以把整个矩阵都载入寄存器中批量运算，实现十几倍的效率提升。  
Tensor Core虽然数量不多，但是每个都特别巨大  

## 典型产品
A100(2020): CUDA Core=6912, Ampere架构  
H100(2022): CUDA Core=18432, Tensor Core=576, Hopper架构  
L40S(2023): CUDA Core=18176, RT Core=142(212 TFLOPS), Tensor Core=568, Ada Lovelace架构  
H200(2024): CUDA Core=?, Hopper架构  

## Hopper架构(以H100为例)
### NVIDA Grace Hopper Superchip
NVDIA Hopper GPU 通过 NVLink链接NVIDIA Grace CPU,整个构架叫做NVIDA Grace Hopper Superchip  
(使得CPU-GPU通讯速度有了很大提升)  
不同GPU间也通过NVLink链接  
不同机器间通过PCIE 5.0接口链接。  
GPU内部含中间两个L2 Cache，链接8个GPC(GPU Process Cluster, GPU处理集群)  

### GPC结构
每个GPC内含有9个TPC(Texture Process Cluster, 纹理处理集群)  
每个TPC含有2个SM(Streaming Multiprocessor, 流式多处理器)  
每个SM含有128个FP32 CUDA Core和4个Tensor Core，同时还有64个INT32 CUDA Core和64个FP64 CUDA Core    
因此，每个GPU含有8x9x2x128=18432个CUDA Core  
每个GPU含有8x9x2x4=576个Tensor Core  
H100是数据中心GPU，因此没有RT Core  
另外，每个SM含四个Warp Scheduler  

## NVidia硬件架构和CUDA的关系
### 基本概念
**`SM(Stream Multiprocessors)`**: 如前所述，SM是GPU最小的计算单位。  
其核心组件有各种Core，还有共享内存，寄存器等。  
同样如前所述，一个Block上的线程是放在同一个SM。  
SM的硬件限制(主要是Cache的大小)也制约了每个Block的线程数量。  
**`Warp Scheduler`**: 调度线程用的。  
**`Dispatch Unit`**: 分发指令用的。因为线程其实是软件概念，最终要转化成指令发给运算单元(也就是各种Core)。  
理论上，分发到每个Block的线程都应该是并行处理的。但这仅仅是软件逻辑层面的假设。  
事实上，从硬件角度上说，即使分发到同一个Block的thread，也并不是能够同一时刻执行。  
真正保证thread并行的是一个Warp。比如一个Warp包含32个并行单元，就表示最多可以32个Thread并行运行。  
换句话说，这32个Thread执行于SIMT模式。即每个Thread使用各自的Data执行指令分支。  
**`Load/Store`**: 访问存储单元LD/ST，用来负责数据处理。  
**`Multi Level Cache`**  
在旧的架构里还有个SP(Stream Processor)的概念，后来(2010+)它被CUDA Core取代了。  
事实上，从2017年起，CUDA Core在硬件上也被拆分成了FPU和ALU，CUDA Core也仅仅成为软件上的概念。  
**`ALU(Arithmetic Logic Unit)`**: 做基本算数，比如加减乘除和逻辑运算(and, or, not, xor)  
**`FPU(Floating Point Unit)`**: 专门做浮点数运算，比ALU更加复杂。  

### CUDA并行计算平台与CUDA线程层次结构
CUDA是一个并行计算构架和编程模型。  
CUDA有基于LLVM构建的CUDA编译器，方便开发者使用C进行开发。  
CUDA提供了C/C++和Python等语言的支持，并且提供OpenCL等API接口。  
**`CUDA TOOLKIT`**: CUDA Compiler, Developer Tools(Debugger/Profiler), CUDA C++ Core  
**`CUDA DRIVER`**: Memory Management, Windows & Graphics, Comms Libraries  
**`CUDA-X LIBRARIES`**: Machine Learning(cuDF, cuML, cuGRAPH), DL/HPC(cuDNN, CUTLASS, TENSORRT, CUDA Math Libraries)  
**`Host`**: CPU  
**`Device`**: GPU  
Host和Device交互执行，可以互相通讯。  
**`Kernel函数`**: 处理并行计算的函数。CUDA会把Kernel函数编译成GPU能执行的程序，运行在Device上。  
Kernel函数写在.cu文件里(Host代码也写在.cu文件里)，用__global__符号声明，并且用<<<grid,block>>>来指定运行参数。  
每个grid包含很多block，block里面包含很多线程。  
每个block内部有共享内存(Shared Memory)供其内部线程共享。不同block之间的内存不能共享，也不能通信，也不保证并行。  

### CUDA架构和NVidia硬件架构的联系
CUDA概念：Thread，Block，Grid  
NVidia硬件架构概念：CUDA Core，SM，Device  
Thread线程是执行在CUDA Core里面的。  
Block线程块只在一个SM上通过Warp进行调度。  
一旦在SM上调起了Block线程块，就会一直保留到执行完Kernel。  
SM可以同时保存多个Block线程块，块间并行的执行。  

### 算力计算NVIDA Peak FLOPs
**`PeakFLOPS = F_clk * N_sm * F_req`**  
F_clk: GPU时钟周期内指令执行行数(FLOPS/Cycle)  
N_sm: SM数量  
F_req: 运行频率  

# ARM GPU(Mali) Architecture
## Mali各代架构
Utgard(2007\~2015)  
Midgard(2010\~2016)  
Bifrost(2016\~2018)  
Valhall(2019\~2022)  
5thGen(2023)  

## Mali架构组成
**`Shader Core`**: 相当于NVidia的SM。  
**`EE`**: Execution Engine(EE)相当于NVidia的SP(也就是后来的CUDA Core)。EE属于Shader Core的一部分，就如同CUDA Core是SM的一部分。  
SP内部含有上百个CUDA Core，但Shader Core里只有两个EE(Valhall)。这可能跟Mali的设计目标为移动设备有关。  
**`Load/Store Unit`**  
**`Attribute Unit`**    
**`Varying Unit`**: 进行attribute的插值运算。  
**`Texture Unit`**  
**`ZS & Blend Unit`**    

# GPU Performance分析
## MMU 页表查询(MMU Page Table Walk)
当内存管理单元 (MMU) 无法在本地缓存中完成转换时，**亲自访问多级页表数据结构，将虚拟内存地址 (Virtual Address, VA) 转换为物理内存地址 (Physical Address, PA)** 的过程。

### 地址转换流程
1. **TLB Hit (快表命中):** MMU 优先查询转换后备缓冲区 (TLB)。若命中，直接返回物理地址。
2. **TLB Miss (快表未命中):** 未命中时触发 **Page Table Walk**。
3. **多级页表逐级查询 (如 4 级页表):**
   * 读取基址寄存器，获取一级页表物理地址。
   * 利用虚拟地址的高位 Index 查找下级页表地址。
   * 重复查询直至在最后一级页表条目 (PTE) 中读取物理页帧号 (PFN)。
   * 将 PFN 与虚拟地址偏移量 (Offset) 拼接生成最终物理地址，并写入 TLB。

### 性能影响
* **访问延迟 (Latency):** 一次 TLB Miss 会引发多次额外的物理内存读访问，导致指令流水线停顿 (Stall)。
* **带宽开销 (Overhead Traffic):** 统计的 MMU 页表读取流量属于系统管理开销，不属于应用层有效数据流，计算 Core 有效带宽时需予以剔除。


## Beat
GPU/CPU 硬件性能建模中，Beat（通常译为“拍”或“数据拍”）指的是在单个时钟周期（Clock Cycle）内，通过总线传输的一块数据单元（Data Transfer Unit）。  
简单来说，当系统需要传输一大块内存数据（例如一个 128 字节的 Cache Line）时，总线通常不会在一个时钟周期内把所有数据一次性送达，而是把这块数据拆分成多次突发传输（Burst Transfer），每一次传输的基本单位就叫作一个 Beat。

在实际芯片硬件或总线传输中，Cache Line 与数据传输的关系如下：  
- 假设 GPU 产生了一次 128 字节 Cache Line 的缺失（Miss），需要向外存发起读取；  
- 如果总线单次数据传输能力（Beat）为 16 字节（128-bit 总线）；(或者说一个beat的大小就是总线的宽度)  
- 那么传输这 1 个 Cache Line 的数据就需要占用 8 个 Beats（$128 / 16 = 8$ 拍突发传输）。 


## Ground Truth and Actual
在硬件性能建模、仿真测试以及数据校验中，Ground Truth（底层基准值） 和 Actual（上层累加值） 是用来做交叉验证（Cross-Validation）的两个对比测量维度。  

由于 GPU/CPU 内部结构非常复杂，直接测量某个地方的数据往往容易出错（比如遗漏了某些请求，或者重复计算了某些流量）。因此，工程师会从物理底层和逻辑上层两个不同的视角去分别统计同一个物理过程，再将它们进行对比。  

Ground Truth（底层基准值 / 物理真实值）
- 定义： 从硬件的最外层、最底层的物理接口或数据总线（如 L2 Cache 到 DDR 内存控制器之间的 AXI 总线）直接观测到的“真实发生的物理流量”。
- 视角： 站在底层硬件（Bottom-up）视角。
- 准确性： 它代表了最终真正穿过物理总线的数据包，是物理世界发生的硬事实（Hard Facts），因此被用作参考标准（Ground Truth）。

Actual（上层累加值 / 逻辑推算值）
- 定义： 从硬件的各个上层逻辑功能单元（如 Shader Core 里的 Ray Tracing Unit、Texture Unit、Load-Store Cache 等）各自统计并加和汇总出来的“理论请求总流量”。
- 视角： 站在上层算法/模块（Top-down）视角。
- 推算逻辑： 把每个上层模块发出的内存读写请求数加在一起，乘以预估的传输大小，推算出“按逻辑来说，系统应该产生了多少流量”。

为什么要对比这两者？  
在理想状态下：Error = Ground Truth - Actual == 0  
然而在实际仿真或测试中，两者的计算往往会出现偏差（即公式中的 Error）
- 如果在模型仿真中：如果 $\text{Error} \neq 0$，说明性能模型写错了/漏算了某些模块。例如：某种 Shader 模块发出了内存请求，但模型忘记把它的统计项加到 Actual 里；或者总线的某种 Overhead 没在模型里正确映射。
- 如果在真实硬件测试（RTL / Silicon Validation）中：用来排查硬件 Bug。如果上层单元统计的写请求总和与底层总线收到的写请求不一致，说明中间的 Interconnect 或 Cache 逻辑可能存在丢包、重复发送请求或死锁预警。  

## GPU模块
### FTC
FTC（Filter & Texture Cache）
### TEX
TEX（Texture Processing Unit / Texture Engine）:负责执行具体的纹理采样逻辑。  
Shader Core 执行到纹理指令，将 UV 坐标交给 TEX 单元。然后TEX 单元 向 FTC（Cache） 查询该坐标对应的纹理数据。  
- 如果 FTC 命中（Hit），直接将数据返回给 TEX 进行过滤计算。
- 如果 FTC 缺失（Miss），FTC 会向外发送数据请求（产生 BEATS_RD_FTC 流量），将所需的纹理 Cache Line 从 L2 / DDR 中拉取过来。
### LSC
LSC（Load-Store Cache）：是 Shader Core 内专为普通内存读写（General Memory Access / Data Traffic）设计的统一数据缓存通道。它负责处理显卡/计算核心执行通用 C/C++、OpenCL、Vulkan 或 Compute Shader 时的 Load（读取变量）和 Store（写入变量/写回 Write-Back）操作。  
### RTU
RTU (Ray Tracing Unit)：RTU 的引入就是为了将这些高重复性、高算力消耗的任务从 Shader Core 中剥离出来，用专用硬件硬件电路去跑。  
比如：BVH 遍历加速（BVH Traversal），射线-三角面相交检验（Ray-Triangle Intersection Testing）。  

之所以选择 FTC、TEX、LSC、RTU，通常是因为这几个模块直接产生或消费大量可归因于数据访问的 cache/memory traffic。  
它们有一个共同点：它们的请求最终会进入 L2 / memory hierarchy，因此可以和 L2 侧的 counter 做 cross-check。  

为什么Shader以外的 Geometry Fixed Function 不参与计算？  
因为实际上隐含了一个假设：Shader-side external traffic ≈ L2 external traffic  
Geometry Fixed Function 不是因为“它不重要”而不统计，而是因为你必须确认它是否存在独立、可对应到 L2 的 memory traffic。  
如果存在，那么它理论上应该加入统计。  

## DDR
DDR 是 Double Data Rate（双倍数据速率） 的缩写，在日常计算机与 GPU 硬件中，它通常指 DDR SDRAM（双倍速率同步动态随机存取存储器），即我们常说的主内存或系统显存/外存。  

在传统的单倍速率内存（SDR, Single Data Rate）中，内存只在时钟信号（Clock Signal）的上升沿（Rising Edge）传输一次数据。  
而 DDR 技术通过硬件优化，在时钟信号的上升沿和下降沿（Falling Edge）各传输一次数据。这意味着：在相同的时钟频率下，DDR 的数据传输速率提升了一倍。  

在计算机系统中，存储结构呈现金字塔形状（速度越快/容量越小/成本越高）：
- 寄存器（Registers）： 速度最快，容量极小（几个 KB）。
- L1 / L2 Cache： 内部缓存，位于芯片内部，速度极快（几十 KB ~ 几十 MB）。
- DDR（系统主存 / 显存）： 外部大容量存储，位于芯片外部，容量大（GB ~ TB 级别），但访问延迟显著高于片上 Cache。

在 GPU 架构中，当计算单元（Shader Core）所需的指令或数据没有在 L1/L2 Cache 中命中（Cache Miss）时，就必须通过内存控制器（Memory Controller）穿过物理总线，直接去 DDR 中拉取数据。  

## Bandwidth Validation
带宽一致性校验（Bandwidth Validation）通过比较底层/硬件边缘计数（Ground Truth，基准值）与上层/着色器核心统计（Actual，实测估算值）之间的偏差，来校验数据流量建模或硬件监控（Hardware Counters）的准确性。  

| 符号              | 含义与说明                                                     |
| :-------------- | :-------------------------------------------------------- |
| `N_slice`       | L2 Cache 切片（Slice）总数                                      |
| `N_sc`          | Shader Core（着色器核心）总数                                      |
| `W_AXI`         | AXI 总线宽度（Bits 或 Bytes）                                    |
| `S_beat`        | 单次传输 Beat 的数据大小                                |
| `B_L2_EXT_RD`   | L2 观测到的外部读传输 Beat 数                                       |
| `B_L2_EXT_WR`   | L2 观测到的外部写传输 Beat 数                                       |
| `Σ B_SC_RD_EXT` | 各 Shader 单元（RTU, FTC, LSC, TEX）发起的外部读 Beat 总和             |
| `Σ B_SC_WR`     | 各 Shader 单元（LSC_OTHER, TIB, LSC_WB）发起的写传输 Beat 总和         |
| `M_L2_IN_TOTAL` | L2 接收到的内部请求消息总数                                           |
| `M_NON_DATA`    | 非数据/管理类开销消息（Eviction, Cache Coherency, MMU Table Reads 等） |
| `Σ B_L2_INT_RD` | 各 Shader 单元（FTC, LSC, TEX, OTHER）发起的 L2 内部读 Beat 总和       |

### 1. DDR 读带宽校验 (DDR Read Bandwidth Validation)
跨视角对比内存读带宽：将 **L2 Cache 观测到的外部内存读流量** 与 **Shader Core 各子单元发起的读请求量** 进行交叉校验，以捕获模型中的计数遗漏或接口不一致。
* **Ground Truth (基准值)**
```
BW_DDR_RD_GT = B_L2_EXT_RD × N_slice × W_AXI
```
* **Actual (测量估算值)**
```
BW_DDR_RD_ACT = (Σ B_SC_RD_EXT) × N_sc × S_beat
```
* **Relative Error (相对误差)**
```
Error_DDR_RD = (BW_DDR_RD_ACT - BW_DDR_RD_GT) / BW_DDR_RD_GT
```

### 2. DDR 写带宽校验 (DDR Write Bandwidth Validation)
验证 DDR 写带宽一致性：核对 **L2 写回外部存储的数据量** 与 **Shader Core（如 Tile Buffer/TIB, LSC Writeback 等）刷出的数据量** 是否匹配，确保写通路（Write Path）建模正确。
* **Ground Truth (基准值)**
```
BW_DDR_WR_GT = B_L2_EXT_WR × N_slice × W_AXI
```
* **Actual (测量估算值)**
```
BW_DDR_WR_ACT = (Σ B_SC_WR) × N_sc × S_beat
```
* **Relative Error (相对误差)**
```
Error_DDR_WR = (BW_DDR_WR_ACT - BW_DDR_WR_GT) / BW_DDR_WR_GT
```

### 3. L2 内部带宽校验 (L2 Internal Bandwidth Validation)
评估 L2 缓存内部有效数据吞吐率：排除 Cache 逐出（Eviction）、缓存一致性消息（Coherency）及 MMU 页表查询等非有效数据流量后，验证 **L2 内部真实数据读流量** 的准确性。
* **Ground Truth (基准值)**
```
BW_L2_INT_GT = (M_L2_IN_TOTAL - M_NON_DATA) × N_slice × 512
```
* **Actual (测量估算值)**
```
BW_L2_INT_ACT = (Σ B_L2_INT_RD) × N_sc × S_beat + BUS_READ × S_beat
```

## GPU 吞吐量计算算法解析 (Throughput Calculation Algorithms)

本文档综合分析了 GPU 性能模型中的两个核心吞吐量计算算法：**内存延迟转换为 GPU 周期** 与 **ALU 吞吐量计算**。这两个公式是 GPU 硬件性能建模与抽象吞吐量计算中的核心模块，主要用于将硬件底层的物理指标转化为统一的性能评估指标。

### 2.1 内存延迟转换为 GPU 周期 (Memory Latency to GPU Cycles)

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

*   **定量评估访存瓶颈**：通过将外部 DDR 访存的数据量、AXI 总线位宽和标定带宽转化为 $gpu\_cycles$，能够精确模拟当 GPU 发生缓存未命中（Cache Miss）或存在大量访存时，流水线需要等待的时钟周期数。
*   **硬件带宽约束建模**：算法中对带宽进行了硬编码上限设定（如最高限制在 55 GB/s），这用于模拟实际芯片设计中受限的内存通道带宽，避免理想化计算导致过高估计硬件性能。

### 2.2 ALU 吞吐量计算 (ALU Throughput)

#### 1. 公式与计算逻辑

该模块的核心逻辑是基于硬件指令计数器（Instruction Counters）和不同功能单元的硬件开销权重，统计总体 ALU 计算吞吐量和资源占用。计算步骤如下：

*   **FMA 吞吐**：乘加指令权重为 $0.5$，分摊到各个子核（$num\_sc$）并考虑异步发射比（$async\_ratio$）。
    $$fma = \left( \frac{{EXEC INSTR FMA} \times 0.5}{num\_sc} \right) \times async\_ratio$$
*   **CVT 吞吐**：数据类型转换指令，引入架构特定的微架构因子（如 $cvt\_pe$）。
    $$cvt = \left( \frac{{EXEC INSTR CVT} \times cvt\_pe}{num\_sc} \right) \times async\_ratio$$
*   **MSG 吞吐**：消息/访存交互指令，权重为 $1.0$。
    $$msg = \left( \frac{{EXEC INSTR MSG} \times 1.0}{num\_sc} \right) \times async\_ratio$$
*   **SFU 吞吐**：特殊函数单元（如超越函数等）计算复杂度较高，权重设定为 $4.0$。
    $$sfu = \left( \frac{{EXEC INSTR SFU} \times 4.0}{num\_sc} \right) \times async\_ratio$$
*   **总 ALU 开销**：累加所有功能单元的归一化开销。
    $$alu\_total = fma + cvt + msg + sfu$$

#### 2. 核心作用与应用场景

*   **异构指令开销归一化**：GPU 执行的指令类型繁杂（如乘加、类型转换、消息交互、特殊函数），它们的硬件执行周期各不相同。该公式通过给不同指令赋予特定的权重因子，将复杂的指令计数器折算为一个可统一比对的总体吞吐量消耗。
*   **微架构差异适配**：通过引入架构特定的 PE 因子（例如 Titan/Turse 与 Krake/Drage 的差异系数），该算法能够灵活适配不同代际 GPU 内部子核的硬件微架构吞吐差异，从而实现高层抽象模拟器对多种不同硬件配置的兼容。


### 2.3 Roofline Model (Predicted GPU Active) 分析报告

**核心概述**
该公式定义了用于识别GPU各子系统中主要性能瓶颈的基础 Roofline 模型。其核心逻辑基于：GPU 的整体性能上限由耗时最长的子系统（即最慢环节）决定。

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



## GPU Roofline 瓶颈模型分析举例
什么是 Bottleneck？  
在 GPU 分析性能模型（A-Model）中，Bottleneck（瓶颈周期数） 指的是某个特定硬件子系统在处理完给定工作负载时，所需要消耗的理论最小时钟周期数（Cycles）。  
在 Roofline 性能模型中，模型假设各个硬件模块（如 ALU、Texture、L2 Cache、DDR 等）在理想状态下是完全并行重叠（Overlap）执行的。  
此时，整个系统或子系统的最终执行时间，取决于耗时最长的那个硬件模块。  
模型中计算出的每一个 ${Subsystem}$ 数值，代表该硬件单元“在吞吐量受限下独自完成工作所需的周期上限”。因此在代码和公式定义中，直接将这些模块算出来的周期数命名为该模块的 Bottleneck（瓶颈）。  

以 ALU 计算公式为例：$${ALU} = 0.5 \times {EXEC INSTR FMA} + 0.5 \times {EXEC INSTR CVT} + 1.0 \times {EXEC INSTR MSG} + 4.0 \times {EXEC INSTR SFU}$$  

把各类指令乘以各自系数后相加，本质上是在做硬件资源消耗的量纲转换与时间累加：

量纲统一（指令数 $\rightarrow$ 周期数）：
- EXEC_INSTR_x 的单位是指令数（Instruction Count）。
- 前面的系数（0.5, 1.0, 4.0）单位是指令周期倒数（Cycles / Instruction），代表硬件管线的发射/执行能力。
- 例如：FMA 硬件发射吞吐是 2 ops/cycle，因此 1 条 FMA 占用 $1 / 2 = 0.5$ 个周期；SFU 属于慢速超越函数（Transcendental Function）管线，1 条 SFU 指令需要占用 4 个周期。
- 指令数乘以系数后，消去了“指令”单位，统一变成了周期数（Cycles）。

ALU 硬件管线的时间累加
- 在 ALU 算术逻辑单元内部，各类指令在流水线上按发射吞吐依次消耗周期。将它们乘系数后的结果相加，算出的总和就是：ALU 硬件单元把这批指令全部执行完所需要的总时钟周期数。

参与 Roofline 的 Bottleneck 竞争
- 计算出的 ALU 周期总数，会被送入 Shader Core 的顶级选大器（MAX 函数）：$$\text{Shader\_Core} = \text{Async\_Ratio} \times \max(\text{ALU}, \text{Texture}, \text{Blend}, \text{RTU}, \dots)$$
- 如果算出来的 ALU 周期数高于 Texture 或 Blend，那么 ALU 的计算能力就成为了限制 Shader Core 性能的真实主导瓶颈（Dominant Bottleneck）；反之，若 Texture 周期更大，ALU 的周期数就只是一个潜在瓶颈指标。

因此，这里的 ALU 公式不是单纯在数指令，而是计算ALU 硬件单元的瓶颈执行周期

该模型主要通过从 Emulator/模拟器采集的硬件计数器（Hardware Counters）数据，预测 GPU 执行周期、识别系统性能瓶颈、评估 Cache 命中率以及计算帧率（FPS）。

### 1. 顶层性能预测模型（Main Performance Model）

A-Model 采用了基于 **Roofline** 的瓶颈分析范式。GPU 的总活跃周期（`Predicted_GPU_ACTIVE`）由微控制器（MCU）的串行开销与各并行处理单元中的**最大瓶颈周期**相加得到：

$${Predicted GPU ACTIVE} = {MCU ACTIVE} + \max ( {Shader Core Bottleneck}, {Tiler Bottleneck}, {L2 Cache Bottleneck}, {Memory Bottleneck})$$

* **${MCU ACTIVE}$**：前端微控制器/主机命令处理器的串行固定开销。
* **Pipeline Bottleneck Net**：主执行流水线遵循“木桶效应”（$\max$ 运算符），即整体性能由最慢的硬件资源瓶颈决定。

### 2. 核心子系统计算公式

#### 2.1 着色器核心瓶颈（Shader Core Bottleneck）

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

#### 2.2 ALU 计算瓶颈（ALU Bottleneck）

ALU 瓶颈由各类指令的执行次数乘以其对应的单指令周期系数（Issue Latency）累加得到：

$${ALU} = 0.5 \times {EXEC INSTR FMA} + 0.5 \times {EXEC INSTR CVT} + 1.0 \times {EXEC INSTR MSG} + 4.0 \times {EXEC INSTR SFU}$$

##### 指令权重系数说明：

| 指令类型 | 周期系数（Cycles/Inst） | 硬件含义与吞吐说明 |
| :--- | :---: | :--- |
| **FMA** (Fused Multiply-Add) | `0.5` | 融合乘加指令（等价于 2 ops/cycle 吞吐） |
| **CVT** (Conversion) | `0.5` | 数据类型转换指令 |
| **MSG** (Message) | `1.0` | 核心间通信与消息同步指令 |
| **SFU** (Special Function Unit) | `4.0` | 特殊功能单元指令（如 $\sin, \cos, \log, \sqrt{x}$ 等慢速超越函数） |

#### 2.3 内存子系统瓶颈（Memory Bottleneck）

内存瓶颈综合评估了系统级缓存（SLC）的总线传输效率与外部 DRAM 带宽限制：

$${Memory Bottleneck} = \max({SLC Bottleneck}, {DDR Bottleneck})$$

##### 1. SLC 瓶颈计算公式
$${SLC Bottleneck} = {Num L2} \times (\frac{10^{-9}}{150}) \times (\frac{{AXI Width}}{8}) \times ({CSF Freq} \times 10^6) \times ({L2 EXT READ BEATS} + {L2 EXT WRITE BEATS})$$

##### 2. DDR 瓶颈计算公式
$${DDR Bottleneck} = ({CSF Freq} \times 10^6) \times (\frac{10^{-9}}{{DDR BW}}) \times ({DRAMC R BYTE} + {DRAMC W BYTE})$$

### 3. Cache 命中率计算（Cache Hit Rate Calculations）

各级缓存的命中率评估指标如下表所示：

| 缓存类型 | 层级 / 目标 | 计算公式 | 说明 |
| :--- | :--- | :--- | :--- |
| **LSC (Load Store Cache)** | L1 Cache 命中率 | $\frac{{LSC READ HIT}}{{LSC READ HIT} + {LSC LINE FILL}}$ | L1 读命中数占总读与 Fill 次数的比例 |
| **LSC (Load Store Cache)** | L2 Cache 命中率 | $1 - \frac{{BEATS RD LSC EXT}}{{BEATS RD LSC}}$ | $1 - {外部总线读 Beat 占比}$ |
| **Texture Cache** | L1 纹理缓存命中率 | $1 - \frac{{TEX TPCH NUM PARKED MISS}}{{TEX TPCH NUM PARKED PASSES}}$ | $1 - {挂起 Miss 占总 Pass 的比例}$ |
| **Texture Cache** | L2 纹理缓存命中率 | $1 - \frac{{BEATS RD TEX EXT}}{{BEATS RD TEX}}$ | $1 - {纹理外部读 Beat 占比}$ |

### 4. 帧率（FPS）计算与误差分析

根据系统时钟频率与 GPU 活跃周期，计算实际帧率（Golden FPS）、预测帧率（A-Model FPS）以及相对误差：

* **实际帧率 (Golden FPS)**：
  $${Golden FPS} = \frac{{CSF Freq} \times 10^6}{{GPU ACTIVE}}$$

* **预测帧率 (A-Model FPS)**：
  $${A Model FPS} = \frac{{CSF Freq} \times 10^6}{\sum {Predicted GPU ACTIVE per segment}}$$

* **相对误差率 (Error Rate)**：
  $${Error} = \frac{|{Golden FPS} - {A Model FPS}|}{{Golden FPS}} \times 100\%$$

### 5. 瓶颈自动识别算法（Bottleneck Identification）

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

---

## 6. 总结与架构启发

1. **分层 Roofline 拓扑**：模型从最底层的存储/算力单元（SLC、DDR、ALU、TEX）到 Shader Core，再到顶层 GPU，均采用了多层级的 `MAX()` 取极大值逻辑，准确捕捉单点硬件瓶颈对系统吞吐的制约。
2. **异步时钟域解耦**：引入 `Async_Ratio` 参数，完美屏蔽了 CSF（系统时钟）和 SC（Shader Core 时钟）在 DVFS（动态频率缩放）下的频率差异。
3. **容错性瓶颈诊断**：瓶颈识别算法设置了 `0.75 * top_confidence` 的相对阈值，能够同时揭示主瓶颈及紧随其后的次要瓶颈，为性能优化提供更全面的指引。


# Reference
https://developer.nvidia.com/zh-cn/blog/nvidia-hopper-architecture-in-depth/  
https://www.bilibili.com/video/BV1bm4y1m7Ki/?spm_id_from=333.880.my_history.page.click&vd_source=e9d9bc8892014008f20c4e4027b98036  
https://en.wikipedia.org/wiki/Mali_(processor)  
https://blog.csdn.net/FishSeeker/article/details/84844330  




