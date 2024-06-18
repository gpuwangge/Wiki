# GPU设计原理
GPU和CPU设计上的区别：GPU设计目标是最大化吞吐量(Throughout), 关心并行度(Parallelism)。  
CPU更关心延迟(Latency)和并发(Concurrency)。  
并行：同时处理多个相同任务。  
并发：处理多个任务，但不是同时。 
显存：GPU里面独立的内存。HBM(High Bandwidth Memory)，通过PCIe与CPU内存通讯。  
GPU缓存Cache机制：目的是为了减少内存(显存，Latency=15x, B/W=1x)的时延。  
GPU寄存器：通常把GPU寄存器regs也当作缓存(L0，Latency=?, B/W=?)。regs距离SM非常近。因为SM是实际的计算单元，所以希望尽快的获取数据。    
同时有一些缓存(L2, Latency=5x, B/W=3x)希望离显存更近已方便读取内存的数据。  
SM本身也有专属的缓存(L1, Latency=1x, B/W=13x)。因此GPU实际会设计多级缓存机制。  
作为对比，GPU外部传输的PCIe速度为Latency=25x, B/W=0.02x，完全跟不上GPU内部的带宽传输速度，会拖累计算。  
计算强度：一个字节如果参与了8个时钟周期的计算，那么它的计算强度就是8。每个算法都有其计算强度。这个值越低就表示此算法越受制于带宽。L1 Cache适合计算强度8的算法，L2 Cache 39, HBM 100...  
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
Grid：所有的线程组成的任务系统。  
Block：Grid中的一些任务线程组成Block，Block中的线程都是独立执行的，可以通过本地数据共享同步交换数据(via local memory)。  
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
最外层使用PCIE 5.0接口，18条NVLink  
中间两个L2 Cache，链接8个GPC(GPU Process Cluster, GPU处理集群)  

### GPC架构
每个GPC内含有9个TPC(Texture Process Cluster, 纹理处理集群)  
每个TPC含有2个SM(Streaming Multiprocessor, 流式多处理器)  
每个SM含有128个FP32 CUDA Core和4个Tensor Core，同时还有64个INT32 CUDA Core和64个FP64 CUDA Core    
因此，每个GPU含有8x9x2x128=18432个CUDA Core  
每个GPU含有8x9x2x4=576个Tensor Core
H100是数据中心GPU，因此没有RT Core  

## NVidia硬件架构和CUDA的关系
### 基本概念
SM(Stream Multiprocessors): 如前所述，SM是GPU最小的计算单位。  
其核心组件有各种Core，还有共享内存，寄存器等。  
同样如前所述，一个Block上的线程是放在同一个SM。  
SM的硬件限制(主要是Cache的大小)也制约了每个Block的线程数量。  
Warp Scheduler: 调度线程用的。  
Dispatch Unit: 分发指令用的。因为线程其实是软件概念，最终要转化成指令发给运算单元(也就是各种Core)。  
理论上，分发到每个Block的线程都应该是并行处理的。但这仅仅是软件逻辑层面的假设。  
事实上，从硬件角度上说，即使分发到同一个Block的thread，也并不是能够同一时刻执行。  
真正保证thread并行的是一个Warp。比如一个Warp包含32个并行单元，就表示最多可以32个Thread并行运行。  
换句话说，这32个Thread执行于SIMT模式。即每个Thread使用各自的Data执行指令分支。  
Load/Store: 访问存储单元LD/ST，用来负责数据处理。  
Multi Level Cache  
在旧的架构里还有个SP(Stream Processor)的概念，后来它被CUDA Core取代了。  

### CUDA并行计算平台与CUDA线程层次结构
CUDA是一个并行计算构架和编程模型。  
CUDA有基于LLVM构建的CUDA编译器，方便开发者使用C进行开发。  
CUDA提供了C/C++和Python等语言的支持，并且提供OpenCL等API接口。  
CUDA TOOLKIT: CUDA Compiler, Developer Tools(Debugger/Profiler), CUDA C++ Core  
CUDA DRIVER: Memory Management, Windows & Graphics, Comms Libraries  
CUDA-X LIBRARIES: Machine Learning(cuDF, cuML, cuGRAPH), DL/HPC(cuDNN, CUTLASS, TENSORRT, CUDA Math Libraries)  
Host: CPU  
Device: GPU  
Host和Device交互执行，可以互相通讯。  
Kernel函数: 处理并行计算的函数。CUDA会把Kernel函数编译成GPU能执行的程序，运行在Device上。  
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

# ARM GPU Architecture
## 典型构架
Utgard(2007~2015)  
Midgard(2010~2016)  
Bifrost(2016~2018)  
Valhall(2019~2022)  
5thGen(2023)  

## Mali Bifrost架构
Shader Core: 相当于NVidia的SM。  
EE: Execution Engine(EE)相当于NVidia的SP(也就是后来的CUDA Core)。EE属于Shader Core的一部分，就如同CUDA Core是SM的一部分。  
Load/Store Unit  
Varying Unit: 进行attribute的插值运算。  
Texture Unit  
ZS & Blend Unit  


# Reference
https://developer.nvidia.com/zh-cn/blog/nvidia-hopper-architecture-in-depth/  
https://www.bilibili.com/video/BV1bm4y1m7Ki/?spm_id_from=333.880.my_history.page.click&vd_source=e9d9bc8892014008f20c4e4027b98036  
https://en.wikipedia.org/wiki/Mali_(processor)  
https://blog.csdn.net/FishSeeker/article/details/84844330  




