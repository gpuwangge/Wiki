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

## 举例
Intel Exon 8280: Memory Bandwidth = 131 GB/sec, Memory Latency = 89ns, 则在89ns内理论可以传输的数据量是131*89=11659 bytes。  
实际在一个乘加运算里只移动了16 bytes，则Memory Efficency = 16/11659 = 0.14%  
NVIDIA H100的Memory Efficency更低。  
如果用并发展开，一次可以执行更多指令，但是有约束。  
如果使用并行展开，需要使用大量线程。  
GPU的DRAM Latency比CPU大数倍(数据搬运延迟更大)，但线程数比CPU多上百倍。 
结论：GPU的多线程架构是为大量任务并行设计的。  


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


# Reference
https://developer.nvidia.com/zh-cn/blog/nvidia-hopper-architecture-in-depth/  
https://www.bilibili.com/video/BV1bm4y1m7Ki/?spm_id_from=333.880.my_history.page.click&vd_source=e9d9bc8892014008f20c4e4027b98036  





