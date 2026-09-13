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


## GPU性能分析的四大组件
GPU 的性能瓶颈分析确实可以归纳为这四个核心部分：  

### 1 Shader Bound(着色器瓶颈)
100% GPU内部，tex, blend, alu, asn, quad, lsc, rtu  

### 2 Memory Bound(内存/显存瓶颈)
半内半外，L2在GPU封装内；DDR通过总线(如AMBA/PCIe)连接，物理位置在主板或SoC外围  
- 实践中，常常把L2和DDR(memory)分开来算
- SLC(System Level Cache, 系统级缓存) 既不是 DDR, 也不是传统意义上的 L2 Cache。它是现代
SoC(片上系统)架构中独立存在的一级片上大容量共享缓存，物理位置在 CPU/GPU 的 L2 缓存与外部 DDR 控制器之间。
- SLC是SRAM(静态随机存取存储器), 全集成在 SoC 硅片内部(On-die)
- 架构位置: 各核心 L1 -> 簇/引擎 L2 -> SLC -> 内存控制器 -> DDR
- 部分高端 SoC 的 Profiler 甚至会将这三者展开为: max(GPU L2, SLC Miss Overhead, External DDR)

### 3 Geometry Bound(几何瓶颈)
100% GPU内部，位于GPU内部几何管线，负责顶点处理与Tiled渲染分块

### 4 CSF Overhead(命令流前端开销)
"桥接域"，引擎虽在GPU内，但主要耗时往往来自CPU准备指令、总线传输延迟与命令缓冲区解析  

前面的 Shader, Memory, Geometry, CSF 四部分，其实是 GPU 中"决定渲染帧率最关键的性能路径"(Critical Path)。换句话说，它们是决定GPU"跑得快不快"的核心引擎。  
但一个完整的现代 GPU(尤其是集成在 SoC 中的移动GPU)内部就像一个微型城市，除了这四个"主要工厂"外，还有许多支撑性、辅助性或专用性的部件。  
它们虽然不一定直接出现在 Roofline 瓶颈公式中，但对 GPU的稳定运行、功耗控制和功能完整性至关重要。  

### 既然有这么多部件，为什么 Roofline 模型只关注那四个？  
这是因为 Roofline 模型的核心目的是“找出限制性能上限的短板”。  
- 并行度原则: Shader, Memory, Geometry 是大规模并行工作的，它们决定了吞吐量。
- 关键路径原则: CSF 是串行的前置依赖，决定了启动延迟。
- 其他部件的影响:
    - L1 缓存: 如果 miss 太高，会转化为 Memory 瓶颈。
    - NoC(Network on Chip): 如果拥堵，会转化为 Memory 或 Shader 的等待时间。
    - PMU: 如果降频，会影响所有部分的绝对速度，但不会改变瓶颈的相对比例。
    - 视频/显示: 它们通常是独立流水线，不占用 3D 渲染的核心资源(除非共享带宽)。

### 结论
GPU 内部确实还有许多其他部件，但 Shader, Memory, Geometry, CSF 是决定3D图形渲染性能的"四大金刚"。  
其他部件大多是为这四大金刚服务的(如缓存、互联)，或者是独立功能模块(如视频、显示)。  
在进行性能优化时，盯着这四个部分通常能解决 90% 以上的帧率问题；而其他部件更多涉及功耗、稳定性、安全或特定功能(如视频播放)的优化。  






# Reference
https://developer.nvidia.com/zh-cn/blog/nvidia-hopper-architecture-in-depth/  
https://www.bilibili.com/video/BV1bm4y1m7Ki/?spm_id_from=333.880.my_history.page.click&vd_source=e9d9bc8892014008f20c4e4027b98036  
https://en.wikipedia.org/wiki/Mali_(processor)  
https://blog.csdn.net/FishSeeker/article/details/84844330  




