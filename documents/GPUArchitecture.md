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
## Cache Line
Cache Line（缓存行）是 CPU、GPU 等处理器中 Cache（高速缓存）与主内存（DDR/LPDDR）之间进行数据交换的最小基本单位。  
即便程序在代码里只读取或修改了一个 4 字节的整数（int），底层硬件也不会只从内存中搬运这 4 个字节，而是会把包含这 4 字节在内的一整行数据（通常为 64 字节或 128 字节）一次性加载进 Cache 中。  
- CPU：常见的 Cache Line 大小通常为 64 Bytes（如 x86、ARM 架构）。  
- GPU：为了适应大规模并行与高带宽需求，GPU 的 L2 Cache Line 或 Sector 通常更大，常见为 128 Bytes 或被划分为 32/64 Bytes 的子块（Sector）。  

为什么选择以 L2 Cache 作为缓存行（Cache Line）与带宽校验的核心：主要由 硬件架构角色、物理分布 以及 缓存一致性边界 决定。  
不选择 L1 的原因：太分散、噪音多、存在合并机制  
- 物理分布分散： L1 是各个计算核心（Shader Core / Compute Unit）私有的。GPU 内部可能有数十甚至上百个 L1 缓存，每个 L1 只能看到本核心的局部访问，无法提供全芯片统一的视图。
- 访问请求被合并（Coalescing）： 线程发起的频繁小粒度内存请求（如 4/16 字节），在经过 L1 阶段时会被合并或过滤。L1 看到的请求数无法直接映射为真实的物理内存 Line 数量。
- 单位粒度不匹配： L1 为了支持高效的线程并行，常采用按 Sector（如 32 Bytes）或更小粒度的管理机制，而系统级总线（AXI）是以完整 Cache Line（如 128 Bytes）为基本传输单位的。

不选择 L3 的原因：GPU 架构特性与物理边界  
- 许多 GPU 架构压根没有传统的 L3： 在经典 GPU 架构（如 NVIDIA、AMD 或移动端 GPU）中，通常只有两级缓存结构——L1（核心私有）和 L2（全芯片共享）。L2 往下就直接通过 Memory Controller 连接外存 DDR/LPDDR。
- 即使存在 L3，它也位于校验边界之外： 部分支持系统级缓存（SLC / System Level Cache）的架构将 SLC 称为 L3。但 SLC 通常位于 GPU 芯片外部或系统级总线上（由 CPU/GPU/NPU 共享）。
- L2 是 GPU 内部控制的物理极限： L2 Cache 是 GPU 芯片内部最后一个由 GPU 逻辑完全控制的缓存层。在 L2 界面进行数据校验，能最精准地划分“GPU 内部 Shader 请求流量”与“实际压入外存的物理流量”。  

| 缓存层级 | 物理属性 | 缺点/不适用的原因 |
| --- | --- | --- |
| **L1 Cache** | 核心私有 (Private) | 过于分散，包含大量未合并的局部小请求，缺乏全局视角 |
| **L2 Cache** | 全局共享 (Shared) | **最佳选择**：物理独立、全局统一，是连接 GPU 内部与外部总线的最终“网关” |
| **L3 / SLC** | 系统级共享 (System-wide) | 许多 GPU 无此层级；若有则受 CPU/其他外设流量干扰，无法独立校验 GPU 模型 |

空间局部性原理（Spatial Locality）：硬件采用 Cache Line 机制的主要依据是局部性原理——如果程序访问了内存地址 $A$，那么它极大概率很快就会访问地址 $A$ 附近的变量（例如遍历数组）。一次性拉取一整行数据，可以大幅提升后续内存访问的缓存命中率（Cache Hit）。  

对齐机制（Alignment）：Cache Line 在物理内存中是严格按其大小对齐的。例如在 64 字节 Cache Line 的系统中，内存地址 $0x00 \sim 0x3F$ 属于同一行，下一个 Cache Line 必然从 $0x40$ 开始。  

## Beat
GPU/CPU 硬件性能建模中，Beat（通常译为“拍”或“数据拍”）指的是在单个时钟周期（Clock Cycle）内，通过总线传输的一块数据单元（Data Transfer Unit）。  
简单来说，当系统需要传输一大块内存数据（例如一个 128 字节的 Cache Line）时，总线通常不会在一个时钟周期内把所有数据一次性送达，而是把这块数据拆分成多次突发传输（Burst Transfer），每一次传输的基本单位就叫作一个 Beat。

在实际芯片硬件或总线传输中，Cache Line 与数据传输的关系如下：  
- 假设 GPU 产生了一次 128 字节 Cache Line 的缺失（Miss），需要向外存发起读取；  
- 如果总线单次数据传输能力（Beat）为 16 字节（128-bit 总线）；(或者说一个beat的大小就是总线的宽度)  
- 那么传输这 1 个 Cache Line 的数据就需要占用 8 个 Beats（$128 / 16 = 8$ 拍突发传输）。  

## Cache Slice
Cache Slice（缓存切片）：是现代 CPU 和 GPU 为了解决高并发访问冲突与布线拥堵（Routing Congestion），将一个原本庞大的集中式大缓存（通常是 L2 或 L3 Cache）在物理和逻辑上拆分成的多个平行、独立的工作单元。每个独立的切片就被称为一个 Cache Slice。  

为什么需要 Cache Slice？  
- 在多核 CPU 或包含数十上百个 Shader Core 的 GPU 中，如果所有核心同时去读写同一个集中式的 L2 Cache，会产生两个严重的物理瓶颈：
- 端口竞争（Bank Conflict / Access Bottleneck）： 几百个线程同时发请求，单个 Cache 接口处理不过来，导致严重的等待延迟。
- 物理布线困难（Physical Layout）： 芯片面积很大，所有核心的信号线如果都挤向芯片中央的同一个 Cache 模块，会导致芯片内部布线极其拥堵。

Cache Slice 的工作机制  
- 分布式布局： 硬件设计时，将 L2 Cache 的总容量（比如 32MB）均匀切分成 8 个各 4MB 的 Slice，并把它们物理散布在芯片的不同位置，就近连接不同的计算核心（Shader Core / Compute Unit）。
- 哈希散列寻址（Address Hashing）： 内存地址在进入 L2 之前，硬件会通过一个交错的 Hash 函数对地址进行计算，决定这个地址的数据应该归属于哪一个 Slice。  
例如：地址 0x1000 映射到 Slice 0，地址 0x1040 映射到 Slice 1。  
- 独立并行处理： 每一个 Cache Slice 都拥有自己独立的控制逻辑、TAG 比较器和数据阵列（Data Array）。只要两个核心访问的数据被 Hash 到不同的 Slice，它们就能完全并行读写，互不干涉。

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

## DDR
DDR 是 Double Data Rate（双倍数据速率） 的缩写，在日常计算机与 GPU 硬件中，它通常指 DDR SDRAM（双倍速率同步动态随机存取存储器），即我们常说的主内存或系统显存/外存。  

在传统的单倍速率内存（SDR, Single Data Rate）中，内存只在时钟信号（Clock Signal）的上升沿（Rising Edge）传输一次数据。  
而 DDR 技术通过硬件优化，在时钟信号的上升沿和下降沿（Falling Edge）各传输一次数据。这意味着：在相同的时钟频率下，DDR 的数据传输速率提升了一倍。  

在计算机系统中，存储结构呈现金字塔形状（速度越快/容量越小/成本越高）：
- 寄存器（Registers）： 速度最快，容量极小（几个 KB）。
- L1 / L2 Cache： 内部缓存，位于芯片内部，速度极快（几十 KB ~ 几十 MB）。
- DDR（系统主存 / 显存）： 外部大容量存储，位于芯片外部，容量大（GB ~ TB 级别），但访问延迟显著高于片上 Cache。

在 GPU 架构中，当计算单元（Shader Core）所需的指令或数据没有在 L1/L2 Cache 中命中（Cache Miss）时，就必须通过内存控制器（Memory Controller）穿过物理总线，直接去 DDR 中拉取数据。  


## Bandwidth

## Cache



# Reference
https://developer.nvidia.com/zh-cn/blog/nvidia-hopper-architecture-in-depth/  
https://www.bilibili.com/video/BV1bm4y1m7Ki/?spm_id_from=333.880.my_history.page.click&vd_source=e9d9bc8892014008f20c4e4027b98036  
https://en.wikipedia.org/wiki/Mali_(processor)  
https://blog.csdn.net/FishSeeker/article/details/84844330  




