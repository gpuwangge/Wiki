# Cache的结构
Cache的结构分为物理层和逻辑层  
逻辑层：CPU/GPU是如何寻找cache数据的  
物理层：数据存在cacheline里，cacheline在物理上放在某个cachebank里，某个cachebank放在某个cacheslice里  

## Cache物理层
<p float="left">
  <img src="https://github.com/gpuwangge/Wiki/blob/main/images/cache_physical.jpg" alt="alt text">  
</p>  

cache物理层的重要概念：分区交错(Partitioned & Interleaved)  
A. 分区(Partitioned) - 物理切割  
* 目的：解决大面积 SRAM 的时序问题。  
* 物理体现：如果 L2 Cache 有 4MB，做成一个整体会太大，信号传递太慢。硬体工程师将其物理切割成 4 个 1MB 的 slice，分散在晶片不同位置（例如围绕在 CPU Core 周围）。  
* 逻辑视角：逻辑上它们仍然组成一个完整的 4MB Cache。  

B. 交错(Interleaved) - 地址映射规则
* 目的：提升频宽，避免冲突。
* 物理体现：通过硬体解码器，将连续的地址单元(Address Bits)路由到不同的物理 Bank。  
地址 0x00 -> Bank 0  
地址 0x40 -> Bank 1  
地址 0x80 -> Bank 0  
* 逻辑视角：逻辑上地址仍然是连续的 0x00, 0x40, 0x80，程式不会感知到它们实际上存在不同的物理电路块中。

### 深入解析:Bank Conflict 避免与存储体映射优化
在高性能计算(如 GPU, DSP, NPU)中, Bank Conflict(存储体冲突) 是影响并行访存效率的关键瓶颈。  

#### 核心概念:什么是 Memory Bank？
高速片上存储器(如 GPU Shared Memory, DSP SRAM)并不是单一的大块内存, 而是被物理划分为多个独立的存储体(Banks)。  
并行访问能力:每个 Bank 可以独立读写。如果多个线程访问不同的 Bank, 操作可以并行完成。  
带宽聚合:总带宽 = 单个 Bank 带宽 * Bank 数量。  
典型配置:例如 NVIDIA GPU 的 Shared Memory 通常有 32 个 Bank, 每个 Bank 每周期可提供 4 字节(32-bit)的数据。  

#### 什么是 Bank Conflict？
当同一个时钟周期内, 多个线程请求访问不同的地址, 但这些地址映射到同一个 Bank 时, 就会发生 Bank Conflict。  
后果:硬件无法并行处理, 必须将访问序列化(Serializes)。  
性能损失:如果有 32 个线程冲突, 原本 1 个周期完成的访问可能需要 32 个周期, 性能下降高达 32 倍。  
无冲突情况:多个线程访问同一地址(Broadcast)通常不会冲突, 或者访问不同 Bank 完全并行。  

以下是图片 "serialization.jpg" 中的完整文字内容提取：

### 访问序列化 (Access Serialization)
在高性能计算和硬件架构中，访问序列化(Access Serialization)是指原本可以并行(Parallel)发生的多个内存访问请求，由于硬件资源冲突或限制，被迫变成串行(Sequential)依次执行的现象。  
简单来说，就是“排队”：大家本来可以同时进门，结果因为门太窄或只有一个柜台，必须一个接一个地进。  

**为什么会发生序列化？**  
硬件资源(如内存 Bank、总线通道、缓存端口)是有限的。当多个计算单元(如 GPU 线程、CPU 核心)在同一时钟周期内请求访问同一个物理资源单元的不同地址时，硬件仲裁器(Arbiter)必须强制将这些请求排队。

**常见场景：**  
* GPU Shared Memory Bank Conflict(最典型)
* DRAM 行缓冲区冲突
* 多核 CPU 争抢同一内存通道
* DMA 通道争用

**详细案例：GPU Bank Conflict 中的序列化**  
32×32 矩阵转置，这是理解访问序列化的最佳场景。

**硬件设定**  
* 线程数：32 个线程(一个 Warp/Wavefront)
* Bank 数量：32 个 Bank
* 理想情况：32 个线程访问 32 个不同的 Bank -> 并行。
* 冲突情况：32 个线程访问同一个 Bank 的不同地址 -> 序列化。

**性能影响量化(吞吐量计算)**  
* 假设硬件理论带宽为 1 TB/s。
* 无序列化：实际带宽 ≈ 1 TB/s
* 32 路序列化：实际带宽 ≈ 1 TB/s ÷ 32 = 31.25 GB/s

**延迟增加**
* 单次访问延迟：不变(例如 10ns)
* 批量访问延迟：从 10ns 变为 320ns(对于 32 个请求)

**对矩阵转置的影响**
在 32×32 矩阵转置内核中：  
* 优化前(有序列化)：Kernel 执行时间 100 μs
* 优化后(无序列化)：Kernel 执行时间 3 μs
* 加速比：约 33 倍


### 仲裁器 (Arbiter)
在硬件系统(如 SoC、CPU、GPU)中，仲裁器(Arbiter)是一个至关重要的控制逻辑模块。它的核心职责是管理多个请求者对共享资源的竞争访问，确保系统有序、安全、高效地运行。  
简单来说，仲裁器就是硬件世界的“交通警察”或“排队管理员”。  

资源竞争场景  
| 请求者 (Masters) | 共享资源 (Slaves) | 冲突原因 |
| --- | --- | --- |
| CPU 核心 0 | 内存控制器 (DDR) | 同时需要读写数据 |
| CPU 核心 1 | 内存控制器 (DDR) | 同时需要读写数据 |
| GPU | 显存 / Shared Memory | 多个线程同时访问 |
| DMA 控制器 | 外设寄存器 / 内存 | 数据传输争抢总线 |
| NPU | 片上 SRAM | 神经网络权重加载 冲突 |

问题: 共享资源(如内存总线、存储体)在同一时刻通常只能服务一个请求。如果多个请求同时到达，没有仲裁器会导致:  
- 数据冲突: 信号打架，数据损坏。  
- 死锁: 双方互不相让，系统挂起。  
- 不可预测: 无法确定谁先访问，导致程序行为不一致。  

解决方案: 引入仲裁器，决定谁先谁后。  

工作步骤
- 请求(Request): 多个模块同时发出访问请求信号。
- 决策(Decision): 仲裁器根据预设策略(算法)选择一个获胜者。
- 授权(Grant): 向获胜者发送授权信号，允许其访问资源。
- 访问(Access): 获胜者进行数据传输，其他请求者等待。
- 释放(Release): 访问完成后，资源释放，仲裁器处理下一个请求。

常见的仲裁策略(算法)
- Fixed Priority 给每个请求者分配固定等级，高优先级永远优先
- Round-Robin 按顺序轮流服务，人人有机会
- FIFO / Age-based 谁先来谁先服务
- Dynamic Priority 根据等待时间动态提升优先级

仲裁器与“访问序列化”的关系  
冲突发生时的仲裁  
当 32 个 GPU 线程同时访问同一个 Memory Bank 时:  
- 物理限制: Bank 只有一个读写端口。
- 仲裁器动作: Bank 内部的仲裁器检测到 32 个请求冲突。
- 序列化执行: 仲裁器将这 32 个请求排队(例如按线程 ID 顺序)，每个周期只允许一个请求通过。

结果: 原本并行的操作变成了串行(Serialized)。  

性能瓶颈  
- 仲裁延迟: 仲裁器本身做决策需要时间(通常 1 个周期)。
- 排队延迟: 等待轮到自己是主要延迟来源。
- 优化目标: 最好的优化是避免冲突(如使用 Padding)，让仲裁器无需排队，直接并行分发到不同 Bank。



## Cache的逻辑层
<p float="left">
  <img src="https://github.com/gpuwangge/Wiki/blob/main/images/cache_logical.jpg" alt="alt text">  
</p>  

Cacheline：基本cache单元。有时又叫一个cacheslot。  
Cacheset：由很多个cacheline组成的集合。  
Cacheway：每个cacheset里面cacheline的数量。  

为什么要设计这种系统？  
假如每个地址对应唯一的cacheline(单way cache)，那么如果两个常用数据刚好对应同一个地址，就会产生conflict  
如果把多个cacheline组成set(多way cache)，每个地址对应一个cacheset，也就是对应多个cacheline，可以减少conflict  

具体例子：  
```
cache_size = 1 * 1024 * 1024 # 1 MB
cache_line_size = 64          # 64 Bytes, 1024 * 1024 / 64 = 16384个Cacheline
ways = 16                     # 16-way, 16个cacheline组成一个set，16384 / 16 = 1024个Cacheset
```

以下是一个完整的CPU发出的cache address的结构：
| Tag (標籤) | Index (組索引) | Offset (線內偏移) |
|---|---|---|
| 用來比對是否命中 | 決定去哪個 Set (組) | 決定在 Line 內哪 Byte |

Offset: 因為 Line 是 64B, 所以最低 6 bits (2^6=64) 用來找 Line 內的位置。  
Index: 因為有 1024 個 Set, 所以中間 10 bits (2^10=1024) 用來決定去第幾組。  
Tag: 剩下的高位元, 用來比對儲存的內容是否正確。只有Tag是真正存儲在Cache裡面的地址，也叫“显式地址”。  

注意的是，cache address里面并没有出现“数据应该存储在哪个cacheline”这条信息。数据放在哪个cacheline里是由cache自行决定的，cpu并不知道。  
当cache收到cpu来的cache address存数据的时候，它可以把数据放在任意一个cacheline里(同时也把tag存在这里)  
当cache收到cpu来的cache address读数据的时候，它在每一way使用一个并行的比较器对比tag，看tag hit到哪一个路，就去那一路读数据。  
这样的好处是CPU不需要知道或管理cacheway的信息。  

实际上，cpu地址里面的index通常是集合了物理层cache的slice/bank信息的。  
换句话说，cpu在使用cache的时候，不但不需要知道数据在哪个cacheline，也不需要知道具体该去看哪个cacheslice和cachebank，这些都会自动完成。  

对于CPU来说，只需要知道如下实际的逻辑顺序：  
步骤 | 使用的地址栏位 | 硬体动作
1 | Index | 找到数据在哪个Cache Set
2 | Tag | 从该 Set 的所有 Way 读出 Tag，并与地址 Tag 并行比对
3 | Hit Signal | 多工器(MUX)选通命中中的 Way 的数据，确认命中后才开启数据通路
4 | Offset | 透过移位器/多工器选取 Line 內的特定 byte

## Associative
Associative (關聯度) 是cache邏輯層的概念，用來描述「一個memory區塊可以存放在 Cache 的哪些位置」的規則。  
簡單來說: Associativity 就是 Cache 的「自由度」。  

DirectMapped: 把具体cacheline的index放在地址里，比对方便，但实际最慢。只在早期微控制器和指令快取片段使用。  
Fully Associative: cache地址可以放在cache的任一cacheline。每次存取都需要对比所有的cacheline。在TLB或小型SRAM里面使用。  
Set Associative: cache地址先定位到一个Set，再在该Set的N Way里任选。这是上面两种方案的折中方案。现代CPU Cache配置 (L1/L2/L3)的首选。  

Set Assciative的way的数量如何确定：  
Way高的时候: 较少发生conflict，但面积大，功耗大，LRU实现复杂  
Way低的时候: 容易conflict，但面积小，电路简单，寻址快  
L1 Cache: 一般8路way  
L2 Cache: 一般48路way  
L3 Cache: 一般1216路way (对命中率要求最高)  

1-way 一般用在特殊SRAM里，简单化，冲突miss率高  

实际应该根据PPA的目标选用不同的way值。  

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

即使是读取uniform和vertex buffer，GPU的访问请求在物理上都会经过L2 Cache.即使在某些模式下可以让L2不保存数据，仍必须走它的路由和仲裁单元。  

| 缓存层级 | 物理属性 | 缺点/不适用的原因 |
| --- | --- | --- |
| **L1 Cache** | 核心私有 (Private) | 过于分散，包含大量未合并的局部小请求，缺乏全局视角 |
| **L2 Cache** | 全局共享 (Shared) | **最佳选择**：物理独立、全局统一，是连接 GPU 内部与外部总线的最终“网关” |
| **L3 / SLC** | 系统级共享 (System-wide) | 许多 GPU 无此层级；若有则受 CPU/其他外设流量干扰，无法独立校验 GPU 模型 |

所有L1/L2/L3都是SRAM材料做的。速度分别为, 几个cycle，十几cycle~几十cycle，?cycle。  
另外，有L0 cache：最靠近 SIMD / 执行单元的小缓存
Register File也是SRAM，速度最快1cycle。  
顺便说一句TBDR的Tile Buffer(or imageblock)也是SRAM，但是它不是Cache。  
GMEM 的物理本质是 DRAM（GDDR / HBM / LPDDR），在架构和编程模型上它是全局可寻址的虚拟内存空间，读写会经过 SRAM 构建的 L1/L2 Cache 硬件层级。  

在高通 Adreno GPU 的 TBR/TBDR（Tile-Based [Deferred] Rendering）分块渲染架构 中，GMEM 指的是紧挨着 GPU 核心的一块高速片上 SRAM 缓存（On-Chip SRAM），其实就是Tile Buffer。  

Apple的统一内存（Unified Memory）：统一数据池：CPU、GPU、NPU（Neural Engine）、Media Engine 共同共享同一块物理 DRAM。  
- Apple内存的零拷贝（Zero-Copy）：数据在内存中只有一份，CPU 处理完后，GPU 可以直接使用该内存地址读写，完全消除了 PCIe 总线的搬运延迟和重复占用。  
- Apple内存的极高总线位宽与带宽：传统 PC 的双通道内存位宽通常只有 128-bit，而 Apple 的 M 系列芯片（尤其是 Pro / Max / Ultra 版本）采用了极宽的封装位宽（如 512-bit 到 2048-bit），提供惊人的内存带宽（M 系列芯片带宽可达 150GB/s 到 1.2TB/s 以上），直接达到了普通显存（VRAM）的吞吐水平。  

Apple的大缓存SLC（System Level Cache，系统级缓存）：为了让 CPU 和 GPU 能顺畅地抢着用同一块主存，而不发生严重的“抢道”和高延迟，Apple 在芯片内部设计了非常激进且庞大的 Cache 结构：位于芯片中央、服务于所有计算核心（CPU, GPU, NPU, ISP）的公共超大 Cache。    
- APPLE的GPU的L1大小适中，但L2也是超大  

空间局部性原理（Spatial Locality）：硬件采用 Cache Line 机制的主要依据是局部性原理——如果程序访问了内存地址 $A$，那么它极大概率很快就会访问地址 $A$ 附近的变量（例如遍历数组）。一次性拉取一整行数据，可以大幅提升后续内存访问的缓存命中率（Cache Hit）。  

对齐机制（Alignment）：Cache Line 在物理内存中是严格按其大小对齐的。例如在 64 字节 Cache Line 的系统中，内存地址 $0x00 \sim 0x3F$ 属于同一行，下一个 Cache Line 必然从 $0x40$ 开始。  

### cacheline 64bit和128bit有什么优缺点
| 特性 | 64-Byte Cacheline | 128-Byte Cacheline |
| :--- | :--- | :--- |
| **空间局部性利用** | 良好，适合通用计算 | 极佳，适合连续流式读取（如图形渲染、张量计算） |
| **Tag RAM 面积开销** | 较高（相同的 Cache 容量下需要两倍的 Tag 数量） | 较低（元数据开销减半） |
| **False Sharing (伪共享)** | 易发生，但影响相对受控 | 极易发生，且带来的 Cache 颠簸惩罚更严重 |
| **随机访存带宽利用率** | 较高（Overfetch 浪费较少） | 极低（为读取极少数据而搬走大量无用数据） |
| **总线突发 (Burst) 契合度** | 适合标准 DDR 传输 | 更契合 GDDR/HBM 的宽总线与长突发传输 |

#### 128B 的优势（即 64B 的劣势）
- 硬件元数据（Tag RAM）开销减半： Cache SRAM 不仅存储数据，还需要大量面积存储 Tag（标签地址）和状态位（如 MESI 协议状态）。Cacheline 从 64B 增大到 128B，意味着在总 Cache 容量不变的情况下，Cacheline 数量减半，Tag 占用的芯片面积和功耗也随之大幅降低。

- 极致的空间局部性收益： 对于具有极高空间局部性的 Workload（例如处理大片连续内存、纹理采样或矩阵乘法），128B 能够通过一次 Fetch 动作预取更多相邻数据，显著降低 Cache Miss Rate，减少内存请求次数。

- 提升高带宽内存效率： 现代高带宽内存（如 GDDR6/HBM）的设计倾向于单次传输大量数据（Long Burst Length）以掩盖内部核心的访问延迟。128B Cacheline 能完美填满这些宽总线单次事务的数据包，使内存控制器达到最高吞吐量。

#### 128B 的劣势（即 64B 的优势）
- 致命的带宽浪费 (Overfetch) 与 Cache 污染： 如果代码的访存模式非常离散（如复杂的树状/图结构遍历、随机指针跳转），为了读取 4 字节的有效数据，硬件必须将整个 128B 搬运到 Cache 中。这不仅浪费了 90% 以上的总线带宽，还会将 Cache 中真正有用的数据挤出，造成严重的 Cache 污染。

- 更高的伪共享 (False Sharing) 惩罚： 在多线程并行中，如果两个独立的线程频繁修改同一个 Cacheline 内的不同变量，会导致该行在不同核心的 L1 间不断发生 Invalidate 和同步重载。128B 涵盖的变量范围是 64B 的两倍，将毫无逻辑关联的变量硬凑到同一行的概率大幅增加，多核扩展性容易受挫。

- 传输延迟 (Latency) 增加： 在总线位宽固定的情况下，串行化传输 128B 数据所需的时钟周期是 64B 的两倍，这会增加关键路径上数据到达寄存器的等待时间，对延迟敏感的控制流极为不利。

#### 架构层面的路线分化
这种权衡导致了不同处理器的底层设计分歧：  

CPU 通常坚守 64B：  
CPU 面向低延迟、复杂的控制流和密集的并发线程操作，极力避免 False Sharing 和随机访存时的延迟。64B 是兼顾局部性和避免伪共享的“甜点（Sweet Spot）”。  

GPU 倾向于 128B 及 Sectored Cache 机制：  
GPU 面向超高吞吐量和海量数据并行，对延迟具有极强的掩盖能力。因此底层（如 L2 Cache）往往采用 128B 以降低庞大 Cache 带来的 Tag 开销并配合 GDDR/HBM 带宽。  
为了缓解离散访存带来的 Overfetch，现代 GPU（如许多基于 Vulkan 或底层 API 开发时可见的架构特性）通常引入 Sectored Cache（分段缓存）：  
一个 128B 的 Cacheline 被划分为 4 个 32B 的 Sector。  
Tag 匹配以 128B 为单位，但有效位（Valid bit）和总线数据请求以 32B 为单位进行。  
如果 Warp/Wavefront 发出离散的内存请求，内存控制器只会抓取 128B 中实际被需要的那个 32B Sector，从而完美兼顾了低 Tag 开销与高带宽利用效率。  

## Cache Eviction
Cache 逐出 (Eviction) 是指当缓存（Cache）空间已满，而 CPU/GPU 又需要载入新的数据时，缓存控制器强制将某一条已存在的缓存数据（Cache Line）移除或写回主存，从而为新数据腾出空间的机制。  

工作原理  
命中（Cache Hit）： 请求的数据已在 Cache 中，直接快速读取。  
未命中（Cache Miss）与替换： 请求的数据不在 Cache 中，需要从更慢的下一级存储（如 L3 Cache 或 DDR/内存）读取。如果此时 Cache 没有空余位置，就必须触发 Eviction。  
Dirty / Clean 状态处理：  
- Clean Line（未修改数据）： 该缓存数据与下一级内存中的内容一致，逐出时直接丢弃/覆盖（Discard），不产生额外的写回开销。
- Dirty Line（已修改数据）： 该缓存数据被处理器写过，与内存不一致。逐出时必须先将其写回（Write-back）到下一级存储，这会产生额外的内存总线流量（写开销）。  

### 常见的替换策略（Eviction Policies）  
决定“哪一条数据该被逐出”由硬件/软件的算法控制
- LRU (Least Recently Used)： 逐出最近最少使用的数据，假设越久没用过的未来越不可能用（最常用的算法）。
- FIFO (First-In, First-Out)： 逐出最先载入的数据，不考虑后续使用频率。
- LFU (Least Frequently Used)： 逐出使用频率最低的数据。
- Random（随机）： 随机挑选一条数据逐出，实现成本低，常用于某些硬件硬件简化的缓存结构。

对系统性能的影响
- Cache Thrashing（缓存抖动）： 如果频繁触发 Eviction（例如程序循环访问的数据量大于 Cache 容量），会导致数据不断被“载入-逐出-写回-再载入”，引发大量的内存带宽开销，显著拖慢系统性能。
- 带宽开销： 在硬件性能建模（如 GPU 仿真）中，Eviction 产生的 Dirty Line 写回属于非有效数据请求（Overhead Traffic），通常需要在计算核心有效数据吞吐率时予以剔除。

## Cache Coherency
缓存一致性消息（Cache Coherency Messages） 是多核处理器或 GPU 多 Slot/Core 架构中，各个 Cache 控制器之间为了保证不同缓存中同一份数据完全一致而发送的控制信号或数据数据包。  

核心痛点：为什么需要 Consistency/Coherency？  
在多核系统（例如 GPU 的多个 L2 Cache Slice 或 Shader Core）中，主存（DDR）中的同一个内存地址 $A$ 可能会同时被复制并缓存到多个核心的私有/局部 Cache 中：  
1. Core 0 读取了地址 $A$（值 = 10），并在本地缓存。
2. Core 1 也读取了地址 $A$（值 = 10），并在本地缓存。
3. Core 0 将地址 $A$ 修改为 20。此时，Core 0 的 Cache 里 $A=20$，但 Core 1 的 Cache 里依然是过期的旧值 $10$。
4. 如果 Core 1 再次读取地址 $A$，就会读到脏数据（Stale Data）。

为了解决这个冲突，系统必须在硬件层面引入缓存一致性协议。  

常见的 Coherency 消息类型为了维护数据的一致状态（如 MESI、MOESI 协议），各 Cache 节点之间会频繁广播或点对点发送以下控制消息：
- Invalidate（失效消息）： 当某个核心写数据时，向其他所有持有该数据 Cache Line 的核心发送“失效通知”，强制它们将本地副本标记为无效（Invalid）。
- Read Shared / Read Exclusive（读请求消息）： 核心申请以“只读”或“独占/准备写入”的状态获取数据。
- Writeback / Probe Response（写回与响应消息）： 当核心 $A$ 申请最新的数据，而最新的数据刚好在核心 $B$ 的 Dirty 状态 Cache 里时，核心 $B$ 会响应并将最新数据发送给核心 $A$ 或写回下一级 Cache。
- Snoop Request / Probe（总线嗅探/探测消息）： 检查其他核心的 Cache 中是否包含指定内存地址的副本。

对系统与性能建模的影响
- 总线带宽开销（Control Overhead）： 一致性消息本身通常不包含完整的 64B/128B 用户数据，而是简短的控制命令包（Header/Address/State）。但在频繁进行跨核数据共享和并发写入时，这些消息会占用相当一部分片上网络（NoC）或总线带宽。
- 性能模型剔除：这类计数器统计的就是与 Cache Coherency / Compute Unit 协议通信相关的非数据消息。在计算真正的有效数据传输带宽时，需要将这些一致性控制消息占用的流量剔除。

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

## Cache Bank
Cache Bank（缓存库） 是位于单个 Slice 内部的物理 SRAM 阵列划分。  
Cache Bank是SRAM 阵列的物理切分。一个 Bank 内部包含成千上万个 Cache Line。  
目的：提供伪多端口能力。单端口 SRAM 同一周期只能响应一个读或写请求。通过将 Cache 切分为多个 Bank（如 16 个或 32 个），只要同一周期到达的多个访存请求指向不同的 Bank，硬件就可以并行处理它们，从而提供高吞吐量。  
交错映射（Interleaving）：为了最大化并发概率，物理地址通常会做 Bank 交错。例如，地址空间中相邻的 Cache Line 会被物理映射到连续的 Bank 0, Bank 1, Bank 2 中。  

## L2 Cache的内部和外部读写
对 GPU 的 L2 Cache 来说，“内部”与“外部”是以 L2 Cache 本身所在的层级（或 GPU 核心边界）来划分的：  

内部读写（Internal Access）  
- 主体： GPU 内部的计算单元（如着色器核心 / SM / WGP）、纹理采样单元、光栅化引擎等。
- 内部读： 当着色器或纹理单元需要读取数据（如顶点数据、纹理贴图、常量缓冲区、本地共享内存溢出数据）时，首先会向 L2 Cache 发起读请求。如果命中（Hit），这就是一次内部读。
- 内部写： 着色器完成计算后，将渲染结果、片元颜色或 UAV（Unordered Access View）写入缓存。

外部读写（External Access）
- 主体： L2 Cache 与 GPU 芯片外部的物理显存（如 VRAM / GDDR6 / HBM）之间的交互。
- 外部读（L2 Miss / Fill）： 当内部单元请求的数据在 L2 Cache 中未命中（L2 Miss）时，L2 Cache 必须通过内存控制器向外部显存发起读取请求，把数据加载到 L2 中。公式中的 BW_DDR_RD（或 B_L2_EXT_RD）指的就是这部分流量。
- 外部写（Write-back / Evict）： 当 L2 Cache 中的脏数据（Dirty Data）因为缓存替换（Eviction）策略需要被清理腾出空间时，或者遇到非 缓存一致性直写（Write-through）操作时，L2 会把数据写回到外部显存中。

内部读写是内部的block读写L2；外部读写是L2读写外部的memory。  

<p float="left">
  <img src="https://github.com/gpuwangge/Wiki/blob/main/images/GPU_Memory_Hierachy.jpg" alt="alt text">  
</p> 

## CCU(Cache & Compression Unit)
CCU（Cache & Compression Unit，缓存与压缩单元） 是现代 GPU（特别是移动端 GPU 如 Arm Mali、Qualcomm Adreno，以及部分桌面级/嵌入式图形 IP）管线中的关键硬件模块。  

高通 GPU 采用的是特殊的分片/平铺渲染（Tile-based Rendering）架构。CCU 的工作是从高通独有的片上高速图形内存（GMEM）中切出一块空间，作为专用的颜色缓存（Color Cache）和深度缓存（Depth Cache）。  

当 GPU 在进行 2D 图像块复制（Blit）、系统内存渲染目标访问、或者把片上 GMEM 的渲染结果最终输出（Resolve）到手机系统内存（主内存）时，全部都要通过 CCU 来进行高速缓存、合并与数据压缩，以此极大地节省手机的内存带宽和功耗。  

L1.5：CCU 内部的 Cache 既不完全等同于传统的通用 L1，也不属于全局共享的 L2，而是一个紧贴着 ROP(Raster Operations Unit) / Tile Buffer 的“专用级缓存”（Specialized/Dedicated Cache）。  

| 厂商/架构 | CCU (Compression Control Unit) 代表性压缩技术 | 说明 |
| :--- | :--- | :--- |
| **Arm Mali** | **AFBC** (Arm Frame Buffer Compression) / **AFRC** | 业界广泛使用的无损/可控损帧缓冲压缩，支持 Color/Depth 并延伸至 Display 接口。 |
| **Qualcomm Adreno** | **UBWC** (Universal Bandwidth Compression) | 高通 Adreno 架构中的通用带宽压缩技术，覆盖 GPU 渲染、Camera、Video Decoder 与 Display DPU。 |
| **NVIDIA** | **DCC** (Delta Color Compression) | NVIDIA GPU 片上 ROP/L2 层的无损增量颜色压缩技术。 |
| **AMD** | **DCC** (Delta Color Compression) | AMD GCN/RDNA 架构中的片上无损增量颜色压缩电路。 |

### CCU在TBDR中的位置
在 **TBDR（Tile-Based Deferred Rendering）** 架构（如 Qualcomm Adreno、Arm Mali 等移动端 GPU）中，**CCU（Cache & Compression Unit）** 与 **Tile** 有着极其紧密的物理与逻辑绑定关系。

简单来说：**CCU 是专门服务于单个或一组 Tile 的片上后端缓存与压缩处理单元。**

#### CCU 与 Tile 的具体绑定关系

在 TBDR 架构中，屏幕被分割为固定大小的小方块（Tile，如 $32\times32$ 或 $64\times64$ 像素）：

##### 1. 逻辑绑定：以 Tile 内的 Block 为单位处理
* TBDR 的核心思想是 **“在片上（On-Chip）把一个 Tile 彻底画完，再写回主存”**。
* CCU 内部的无损压缩引擎（如 UBWC/AFBC）**必须以 Tile 内的微型像素块（Block / Micro-Tile，如 $4\times4$ 或 $8\times8$ 像素）为基本单位**进行压缩。它不能跨 Tile 处理，也不能按单像素处理。

##### 2. 物理数据流：Tile 渲染的最后一个出口
在 Tile 渲染的完整生命周期中，CCU 扮演的角色如下：

* **In-Tile 渲染阶段（Tile Buffer 内）**
  当 Fragment Shader 在绘制当前 Tile 时，所有的深度测试、Color 输出、Alpha 混合都在片上 SRAM（如 Adreno 的 **GMEM / Tile Buffer**）中高速完成。此时 CCU 的 Cache 在旁随时准备接收和汇总 ROP 输出的数据。

* **Tile Resolve / Store 阶段（写回 DRAM 时）**
  当当前 Tile 渲染完毕，要执行 **Resolve / Store**（将 Tile 结果存入片外 DRAM 帧缓冲区）时，数据**必须经过 CCU**：
  1. **CCU 抓取 Tile 缓存中的数据**。
  2. **CCU 的 Compression 硬件对该 Tile 进行实时无损压缩**。
  3. **压缩后的 Tile 数据被一次性写入 DRAM**。



#### 概念区分：Tile Buffer (GMEM) vs CCU

在高通等 TBDR 架构中，Tile Buffer 与 CCU 的分工非常明确：

| 模块 | 物理本质 | 在 TBDR 中的角色 | 运作范围 |
| :--- | :--- | :--- | :--- |
| **Tile Buffer (GMEM)** | 片上大容量 SRAM | **“画板”**<br>存放当前正在绘制的 Tile 的全量像素/深度数据。 | **Tile 内部**<br>（Pixel 级频繁读写与 Blend） |
| **CCU (Cache & Compression Unit)** | 专用 Cache + 硬件压缩电路 | **“打包出库通道”**<br>把画板上画好的 Tile 数据压缩打包发送给 DRAM；或者把之前压缩过的 Tile 解压读取回来。 | **Tile 出入口**<br>（Block 级的压缩/解压与 Burst 传输） |


在 TBDR 架构里，**CCU 是 Tile 数据出入片上 SRAM 的“关卡”**：

$$
\text{Tile Buffer (GMEM)} \xrightarrow[\text{Block 组装}]{\text{Tile 结束}} \text{CCU (Cache)} \xrightarrow[\text{实时无损压缩}]{\text{Compression}} \text{L2 Cache / System DRAM}
$$

只要 GPU 在以 TBDR 方式逐块渲染，**CCU 就是以 Tile（或 Tile 内部的 Block）为单位**在片上完成数据收集，并在刷入显存的前一刻完成硬件压缩。


### CCU 的两大核心功能
1. Cache（缓存功能）
Pixel/Attachment 缓存：CCU 负责暂存 ROP（Render Output Unit）写入的颜色（Color Target）、深度（Depth/Stencil Buffer）数据。  
合并与解耦（Merging & Decoupling）：当像素着色器（Fragment Shader）计算出大量的 Render Target 输出时，CCU 作为中间缓冲区，将分散的小块像素读写合并为大块的连贯 Burst 传输，极大地改善了内存访问效率。  
2. Compression（无损帧缓冲压缩）
这是 CCU 最核心的物理价值所在。图形渲染中大量的数据（如 Render Target、MSAA 采样点、Depth Buffer）存在极高的空间局部性与冗余度。CCU 内置了专用硬件算法电路，提供实时无损压缩/解压缩（Lossless Framebuffer Compression）：  
写入路径（Write Path）：ROP 输出像素数据 $\rightarrow$ CCU 硬件算法进行块压缩（Block-based Compression） $\rightarrow$ 较小的体积写入 DRAM/L2。  
读取路径（Read Path）：后期 Pass（如 Post-Processing、Blend、Texture Sampling 或 Display Controller）读取数据 $\rightarrow$ CCU/纹理单元硬件解压 $\rightarrow$ 还原为原始像素。  

### CCU 的关键设计收益
1. 大幅降低移动端功耗：
在移动 GPU（TBR/TBDR 架构）中，访问片外 DRAM 的功耗远高于片内计算。CCU 的硬件压缩通常能带来 20% ~ 50% 的内存带宽节省，直接显著延长设备续航并降低发热。  

2. 加速 MSAA（多采样抗锯齿）：
MSAA 会导致 Depth/Color 缓冲区大小翻倍（如 4x MSAA）。CCU 可以通过专门的深/模板压缩算法（如 Delta/Clear Compression），将未被边缘覆盖的覆盖块压缩到极小尺寸。  

3. 跨系统组件的“零解压”流转：
在现代 SoC（如 Snapdragon 或 Dimensity）中，CCU 压缩的数据不仅 GPU 自己看懂，甚至可以直接传给 DPU（Display Processing Unit，显示控制器） 或 VPU（Video Engine）。显示硬件直接读取 UBWC/AFBC 压缩流并解码显示，彻底消除了跨 Chiplet/IP 间的无谓解压开销。  

### 为什么 Cache 要和 Compression（压缩）合并在一起？
将缓存和硬件无损压缩电路打包成一个统一的 CCU 模块，是图形芯片设计中非常经典且高效的硬件协同设计（Co-design），原因主要有以下三点：  
1. 算法逻辑：无损压缩必须以“Block（数据块）”为单位操作  
2. 数据流的最佳流水线位置（Pipe Pipeline Bottleneck）  
3. 内存元数据（Metadata / Header）的协同管理  

### 压缩效果
统计数据表明，在现代 3D 游戏或 UI 渲染中：约 30%~50% 的像素块是纯色或极高一致性的（压缩率 > 80%）；  
约 40% 的像素块是平滑渐变或线性深度的（压缩率 50%~75%）；  
只有不到 10%~20% 的复杂纹理边缘块是难以压缩的（压缩率 0%）。  
因此，对 $4\times4$ 这 64 字节的小块进行压缩，平均能为整个场景节省 20%~50% 的带宽，而在 Depth Buffer 和背景区域，轻松突破 80% 甚至 90%。  

### 为什么 GPU 的压缩 Block 偏偏选用 $4\times4$（64 字节）等微型尺寸？
虽然直觉上数据块越大，整体找到冗余并进行高倍率压缩的潜力越大，但在硬件层面，**$4\times4$ 或 $8\times8$ 这种微型 Block 是经过严密权衡后得出的最佳工程解**。原因主要包含以下三个核心维度：  

1. 对齐 Cache Line（缓存行物理匹配）  

* **硬件传输单位**：GPU 片外 DDR 内存或片上 L2 Cache 的基本传输单位（Cache Line）通常就是 **64 字节** 或 **128 字节**。
* **1:1 完美映射**：一个 $4\times4$ 的 RGBA8 像素块正好是 $16 \times 4 \text{ Bytes} = 64 \text{ Bytes}$，能**完美 1:1 映射到一个 Cache Line**。
* **粒度适配**：如果解压或写回的单位过大（如 1KB），哪怕上层仅读取了其中一个像素，硬件也不得不从显存调取整个庞大的数据块，这反而会引发严重的数据搬运浪费（Bus Amplification）。

2. 空间局部性（Spatial Locality）的最佳平衡  

* **高一致性保证**：$4\times4$ 的物理尺寸极其微小，在绝大多数情况下，这 16 个像素都会完全处于**同一个三角形或者同一块材质内部**，像素间的颜色/深度关联性极强（极易进行 Delta 增量压缩）。
* **跨越边缘风险**：如果 Block 尺寸扩大（例如 $32\times32$），一个块内极易跨越物体的几何边缘（如一部分是高对比度的树叶，一部分是背景墙面）。颜色一旦发生剧烈跳变，高阶压缩算法就会失效，导致**完全压不动**。

3. 硬件电路成本与延迟（Hardware Area & Latency）  

* **零延迟流水线**：CCU 必须在每个 Clock Cycle（时钟周期）内对数据进行**实时、无缝的无损压缩与解压**，不能阻塞渲染管线。
* **晶体管开销控制**：处理 16 个像素（$4\times4$）的增量计算与编码，其并行逻辑电路非常轻量；若扩大到 256 个像素（$16\times16$），硬件压缩/解压电路的逻辑复杂度、晶体管占用面积以及计算延迟将呈指数级上升，这在芯片设计中是不可接受的。


## Set-associative（组相联）
Set-associative（组相联）通常指 CPU/GPU 的一种 Cache 组织和映射方式。  
它把 Cache 划分为许多 set（组），每组包含若干条 way（路/缓存行）：一个内存块只能映射到某个固定的组，但在该组内可以放进任意一路。它是直接映射（direct-mapped）与全相联（fully associative）之间的折中。  

核心结构:假设一个 Cache 有
64 个 set  
每个 set 有 4 个 way  
每个 cache line 为 64 B  
这就是一个 4-way set-associative cache，总容量为：
```
64 sets×4 ways/set×64 B/line=16 KB
```
其中：
Set：由同一个 index 选中的一组 cache line  
Way：一个 set 内的一条候选 cache line  
Associativity（相联度）：每个 set 中 way 的数目；4-way 就是 4 路组相联  
Cache line / block：Cache 和下级内存之间搬运数据的最小块，常见大小是 64 B  

CPU 访问一个地址时，通常把地址概念上拆成：
```
[Tag∣Set Index∣Block Offset]
```
Block offset：定位到一个 cache line 内的具体字节。  
Set index：决定去哪个 set 找。  
Tag：在该 set 的所有 way 中确认“是不是我要的那个内存块”。  

例如，一个地址的 set index 指向 set 12：
硬件选中 set 12。  
并行读取这个 set 的全部 4 个 way 的 tag。  
将地址 tag 与 4 个 tag 比较。  
任意一路匹配且 valid，就发生 cache hit。  
全部不匹配，就是 cache miss；数据从更低层 cache 或内存取回，并在这个 set 的某一路中填入  

因此，4-way 的含义并不是“一个地址可随机放到 Cache 的任意四个位置”，而是：  
它先被 index 限制到一个固定 set；随后可位于该 set 内四个 way 的任意一个。  

可以将它理解为停车场：
Direct-mapped：你的车牌号决定唯一车位；找车快，但车位被占就没办法。  
Fully associative：可以停任意车位；最灵活，但找车时必须查遍全部车位。  
Set-associative：车牌号先决定停车区域（set），在该区域里的几个车位（ways）任选一个；这是工程上最常用的折中。  

组相联主要解决 conflict miss（冲突未命中）。
设两个频繁访问的数据块 𝐴和 B 恰好映射到同一个 set：  
对 direct-mapped cache，它们只能争夺同一条 line：  
```
A→B→A→B→⋯
```
会不断互相驱逐，形成 cache thrashing。  
对 2-way cache，如果该 set 有两条 line，A 可放在 way 0、B 可放在 way 1；之后两者都能命中。  
如果同一组中长期活跃的数据块多于相联度，比如 4-way 中有 5 个同组热点块，则仍可能发生冲突与替换。  


## Cache性能分析
### 什么是 temporal locality 和 spatial locality？GPU 上各自如何优化？
- Spatial locality（空间局部性）：访问一个地址后，很快访问其邻近地址。  
优化：连续数组、紧凑数据布局、线程 ID 映射到连续元素、mipmapping、tile/block 处理。  

- Temporal locality（时间局部性）：同一个地址或 cache line 在较短时间内被重复访问。  
优化：blocking/tiling、复用中间结果、persistent data、将热点数据放在更适合的只读或显式共享路径。  

优化方向：人为建立可预测且高重用的 temporal locality。当同一 block/workgroup 对一组数据有高复用时，显式 tiling 往往比依赖硬件 cache 更稳健。

### 什么时候 shared memory/LDS 比 L1 cache 更好？什么时候反而不值得？
适合显式 shared memory/LDS 的情况：  
- workgroup 内数据复用高，而且复用模式可预测。
- 需要重排数据，例如矩阵转置、卷积 tile、histogram、reduction。
- 需要跨线程通信与同步。
- 需要将 global 的不规则或低效访问变成片上连续访问。
- 硬件 cache 难以自动捕获该复用，例如访问跨度大、工作集竞争严重、复用时序过长。

不一定值得的情况：  
- 每项数据只用一次，搬入 shared memory 只是多一次 load/store 与同步。
- 工作集本身已很好地命中 L1/L2。
- shared memory 的分配降低 occupancy，得不偿失。
- 访问产生严重 bank conflict。
- 算法本身 compute-bound，内存优化对总时间影响很小。

### 为什么“使用更多 shared memory”可能让性能下降？
常见原因有五类：  
- 降低 occupancy  
每个 block/workgroup 分配更多 shared memory，单个 SM/CU 能并发驻留的 block 数减少，可用于隐藏内存延迟的 active warps/waves 变少。  

- Bank conflict  
多个 lanes 访问映射到同一 bank 的不同地址，会被序列化或拆分为多次访问。  

- 额外搬运成本  
global → shared 的 load 与 shared → register 的读取需要指令；若复用不够高，成本无法摊销。  

- 同步成本  
tile 填充后通常需要 barrier；如果 workgroup 内工作不均或 barrier 很多，会削弱收益。  

- 挤占 L1/片上资源  
在一些架构上 shared memory 与 L1 容量或资源调度存在关系；即使物理上完全独立，二者也可能共同影响 SM 的资源压力。  

### 为什么 ray tracing 往往是 cache-unfriendly？如何改善？
ray tracing 的 cache 难点来自：  
- 光线经过反射、折射、随机采样后迅速失去空间相干性。
- 每条 ray 的 BVH traversal path 不同，导致 control-flow divergence。
- BVH node、triangle、material、texture、ray state 的访问交错且随机。
- hit/miss、不同材质、不同 bounce 导致 shader execution reordering 不足或受限。
- 较大的 ray payload、attribute、stack/state 会增加寄存器压力、spill 和 cache 竞争。

常见改善策略：  
- Ray sorting / ray reordering：按方向、origin cell、material、hit type、shader group 等重排，提高 traversal 或 shading coherence。
- BVH layout 优化：压缩 node、使用紧凑的 child bounds、宽 BVH、减少 pointer chasing、按 traversal 访问顺序布局。
- 减少 payload/attribute 压力：缩小 payload，避免不必要的 live variables，减少 register spill。
- 分阶段 pipeline：将 traversal 与 shading 分离，按 material 或 hit 类型 batch。
- 纹理和材质布局优化：压缩材质参数、减少随机 descriptor/resource indirection。
- 控制 bounce 与随机性：在允许的质量范围内使用更相干的采样策略，或在合适阶段进行 path compaction。
- 用 profile 验证：判断瓶颈究竟是 RT core traversal、shader execution、L2 miss、VRAM bandwidth、occupancy 还是 divergence。

### 如何判断一个 kernel/shader 是 cache-bound、bandwidth-bound 还是 latency-bound？
bandwidth-bound 是单位时间搬运的字节数接近瓶颈；latency-bound 是单次访问或依赖链太长且并发不足以掩盖它；cache-bound 常表示 cache miss、cache throughput、cache contention 或 working-set thrashing 限制了性能。三者经常共存，必须以计数器和对照实验区分。  

### 如果一个 shader 的 L2 hit rate 很高，但性能仍不好，可能是什么原因？
可能原因包括：  
- L2 本身带宽饱和：命中不等于免费；大量 L2 hit 仍可能压满 L2 fabric/port。
- 请求数量过多：stride、未合并访问、过小粒度 load 导致大量 transaction。
- L1 miss / L2 latency 仍无法隐藏：occupancy 低、寄存器压力高、波前数不足。
- load-use dependency chain 很长：同一线程必须等本次 load 才能发出下一次关键指令。
- warp/wave divergence：有效 lanes 少，吞吐被控制流稀释。
- shared-memory bank conflict 或同步：真正瓶颈不在 global memory。
- compute-bound 或 special-function-bound：例如 ALU、RT traversal、texture filtering、寄存器文件端口、指令派发受限。
- cache thrashing：平均 hit rate 高，但关键数据的 reuse 仍被短时间驱逐。
- L2 与其他工作负载竞争：例如并发 queue、graphics + compute 或其他 kernel 共享 L2。


## Cache有价值的总结
- GPU cache 的主要价值是减少下层流量；GPU 更依赖大量并发 warp/wave 隐藏延迟。

- L1 通常更局部，L2 通常跨 SM 共享；精确结构因厂商和代际而异。

- shared memory/LDS 是显式 scratchpad，不是自动 cache；适用于可预测的 workgroup 级高复用。

- cache hit rate 不等于性能；还必须看 transaction、line utilization、带宽、stall、occupancy。

- warp/wave 内连续、对齐、紧凑的访问通常能减少 transaction，这就是 coalescing。

- SoA(Structure of Arrays，数组结构体)/AoS(Array of Structures，结构体数组) 应按 wave 的实际访问字段和复用模式选择，不能教条化。

- shared memory 优化会受到 bank conflict、barrier 和 occupancy 降低的制约。

- shared L2 不等于自动正确同步；coherence 与 consistency/ordering 是不同问题。

- atomic 的关键问题是 contention；用 warp/block 局部聚合降低全局热点原子次数。

- 优化要以 profiler 和对照实验闭环，而不是只依赖架构直觉。


## Dynamic Cache
Dynamic Cache（动态缓存 / 动态显存分配） 通常指的是 GPU 硬件能够根据实际工作负载的需要，在运行时（Runtime）实时、按需动态分配硬件缓存/显存资源的技术，而不是在编译或任务启动前静态预分配固定大小的资源。  
传统静态分配 (Static Allocation)： 在传统 GPU 架构中，当 Shader（着色器）准备执行时，硬件或编译器会根据该任务可能需要的最大资源上限（Worst-case），提前为每个线程/Task 划定固定大小的 Register（寄存器）和 Local Memory/Cache。  
- 缺点： 大多数线程在实际运行中根本用不满预留的最高资源，导致极大的缓存/显存浪费，限制了同时并行的线程数量（降低了 Occupancy / 占用率）。

动态分配 (Dynamic Cache / Caching)： 硬件内置微架构级别的分配引擎，能够实时监控每个 Execution Thread / SIMD Lane 的真实需求。用多少，给多少。  
- 优点： 释放了被闲置的 Cache/SRAM 空间，使 GPU 能够同时调度和并行处理多得多的线程，大幅提升 GPU 核心占用率与吞吐效率。  

主要应用场景
- 光线追踪 (Ray Tracing) 与复杂计算： 光线追踪或物理模拟任务具有极强的不确定性和分支发散性（Divergence），不同光线需要的计算资源差异极大。Dynamic Cache 能极其显著地改善这类负载下的 GPU 资源利用率。
- 移动端 / 低功耗芯片： 在 SRAM（片上缓存）面积和功耗受限的芯片设计中，Dynamic Cache 可以在不增加物理 SRAM 面积的前提下，等效提升缓存利用率并降低对外部 DRAM 带宽的依赖。

## Atomic
Atomic（原子操作） 指的是不可分割的、一步完成的内存读写操作。  
为什么移动端 GPU 需要 Atomic？  
移动端 GPU（如 Arm Mali、Qualcomm Adreno、Imagination PowerVR、Apple GPU）采用大规模并行架构，成百千个 Core/ALU 会同时运行 Shader。  
常见的 Atomic 指令类型  
- atomicAdd（原子加）、atomicSub（原子减）、atomicMin / atomicMax（原子取极值）

移动端 GPU 设计中 Atomic 的核心痛点与挑战  
- 写回机制（Write-Through vs Write-Back）： 传统的移动端 L1/L2 Cache 往往追求低功耗，采用简化的一致性协议。如果 Shader 在 L1 Cache 中执行 Atomic 操作，如何通知其他 Core 的 L1 Cache？  

Write-through：写 Cache 的同时，立即写 Memory。  
Cache 和 Memory 始终保持同步。  
优点：简单,Memory 中的数据比较新,cache eviction 时不需要额外 write-back  
缺点：每次 write 都可能产生 memory traffic, 写很多数据时，memory bandwidth 压力大,通常会配合 write buffer，避免 CPU 每次写都被 memory latency 卡住。  

Write-back：先只写 Cache，等这个 cache line 被 eviction 时，再写回 Memory。  
Memory 可能暂时保存旧数据，Cache 保存最新数据。  
优点：大幅减少 memory write traffic,对大量连续写入尤其有效,更节省 memory bandwidth  
缺点：需要 dirty bit,cache eviction 时需要 write-back,cache coherence / consistency 设计更复杂  

- 硬件硬件硬件原子单元（Atomics Engine）： 现代移动 GPU 倾向于把 Atomic 操作下放到 L2 Cache 硬件层（L2 Atomic Unit）甚至 System Interconnect（如 AXI5 / CHI 协议中的 Bus Atomics）执行。线程发起 Atomic 请求后直接路由到 L2，不经过 L1 缓存，从而避免复杂的 L1 缓存一致性维护。  

性能损耗（Performance Penalty & Contention）  
- 管道阻塞（Serialization）： 当成百上千个线程对同一个 Hotspot（热点地址）执行 Atomic 操作时，原本高并行的 GPU 会退化成单线程串行执行，造成严重的 Stall（流水线停顿）。  
- Tile-Based Deferred Rendering (TBDR) 的冲突： 移动端广泛采用 TBDR 架构，局部数据存在 Tile Memory (On-chip SRAM) 中。如果 Atomic 发生在 Tile Memory 内（如 Order-Independent Transparency 顺序无关透明度、自定义 Blend），速度极快；但如果 Atomic 发生在 Global Memory，则会频繁占用外部 DRAM 带宽，导致功耗骤升。  



## 32x32 矩阵转置存储的硬件优化方案
假设矩阵里面存的是pixel数据：一个pixel有4byte (32bits) : RGBA  
每行32个pixel，也就是128Byte  
将矩阵以转置形式存储是提升硬件访存效率的重要手段，尤其适用于矩阵乘法、卷积、神经网络等计算密集型任务。  

转置存储的基本概念  
原始矩阵按行主序存储：  
```
A[i][j] 存储在 memory[i * 32 + j]
```
转置后存储为：
```
A_T[j][i] 存储在 memory[j * 32 + i]
```
即行列互换后存储，后续访问 A[i][j] 实际上读取的是 A_T[j][i]。  

为何采用转置存储（性能优势）  
1. 提升内存访问连续性
许多算法（如矩阵乘法 C = A * B）中，矩阵 B 常按列访问。若 B 按转置形式存储，则列访问变为连续内存访问，显著提升缓存命中率与带宽利用率。  
2. 避免运行时转置开销
预转置矩阵可消除运行时转置操作，节省计算资源与内存带宽。  
3. 适配硬件内存结构
如 GPU 的 Shared Memory, Tensor Core, SIMD 单元等，对特定访存模式有优化，转置存储可最大化利用这些特性。  

硬件优化策略
1. 内存布局优化  

| 技术 | 描述 | 优势 |
| --- | --- | --- |
| 行转列存储 | 提前转置矩阵 B | 提高列访问效率 |
| 分块存储(Tiling) | 将 32x32 分为 8x8 子块 | 提升缓存局部性 |
| 向量化对齐 | 地址按 128-bit 对齐 | 支持 SIMD 加载 |
| Bank 冲突避免 | 映射到不同存储体 | 提升并行访问效率 |

2. 访存模式优化
使用连续内存访问代替跨行/跨列跳转  
配合 DMA 预取机制减少访存延迟  
利用寄存器文件缓存热点数据  

3. 并行计算优化
利用多线程/多核心并行处理子块  
应用 SIMD/向量指令进行数据并行  
使用 Tensor Core 等专用硬件加速矩阵运算  


## 32×32 矩阵转置中的冲突
以 32×32 的 float 矩阵转置为例，这是最典型的 Bank Conflict 场景。  

**冲突场景演示**  
假设 Shared Memory 定义为 `float tile[32][32]`，共有 32 个 Bank，每个 Bank 处理 4 字节。

**行访问(无冲突):**  
* 线程 0 访问 `tile[0][0]` (地址 0) -> Bank 0
* 线程 1 访问 `tile[0][1]` (地址 4) -> Bank 1
* ...
* 线程 31 访问 `tile[0][31]` (地址 124) -> Bank 31
* 结果:32 个线程访问 32 个不同 Bank，完全并行。

**列访问(严重冲突):**  
在转置写入时，线程需要写入 `tile[threadIdx.x][threadIdx.y]`。假设 `threadIdx.y` 固定为 0，`threadIdx.x` 从 0 变到 31。
* 线程 0 访问 `tile[0][0]` (地址 0) -> Bank 0
* 线程 1 访问 `tile[1][0]` (地址 128) -> 128/4 = 32 -> 32 % 32 = Bank 0
* 线程 2 访问 `tile[2][0]` (地址 256) -> 256/4 = 64 -> 64 % 32 = Bank 0
* ...
* 结果:32 个线程全部访问 Bank 0，发生 32-way Bank Conflict，性能急剧下降。

**优化方案: Padding(填充)技术**  
```
// 存在 Bank Conflict 的版本
`__shared__ float tile[32][32];`
// 行宽 32 是 32 的倍数，导致列访问时地址步长是 Bank 数的倍数
```
```
// 优化后的版本 (Padding)
`__shared__ float tile[32][33];`
// 行宽改为 33，破坏了对齐关系，列访问时地址分散到不同 Bank
```

**其他避免 Bank Conflict 的技巧**  
除了 Padding，还有以下策略:  
1. **数据洗牌(Shuffling):** 在写入共享内存前，通过寄存器交换数据，使得写入地址自然分散。
2. **改变线程映射:** 调整线程块(Thread Block)的维度，使得线程 ID 与内存地址的映射关系错开。
3. **使用向量化访问:** 如果硬件支持，使用 `float4` 等宽向量类型，减少访问次数，但需注意对齐。
4. **编译器优化:** 现代编译器(如 LLVM/GCC for DSP)有时能自动检测并插入 Padding，但手动控制更可靠。

## 硬件怎么识别专属的cache
在多核处理器（如 CPU 或 GPU）架构中，硬件核心并不需要通过复杂的软件逻辑去寻找或“识别”自己的专属 Cache（如 L1 或 L2）。这种归属关系在芯片的物理设计与硬连线（Hardwiring）阶段就已经被固定下来，随后通过硬件状态机和地址路由来管理。  

具体而言，硬件通过以下几个层次的机制来识别和管理专属 Cache：  

1. 物理层：硬连线与拓扑结构 (Physical Hardwiring)  
专属 Cache（通常为 L1 指令/数据 Cache 和私有 L2 Cache）在硅片的物理布局上紧贴着对应的执行核心。  
核心内的存取单元（Load/Store Unit, LSU）通过专属的内部数据总线直接硬连线到这块 SRAM。当核心发出内存访问请求时，电信号在物理层面上只能首先传输到这块直接相连的 Cache 阵列，它在物理上根本“看”不到其他核心的私有 L1/L2 Cache。这种物理隔离是专属身份的最基础保障。  

2. 路由层：硬件核心 ID 与片上网络 (Hardware IDs & Routing)  
当专属 Cache 发生未命中（Cache Miss），需要向下一级共享 Cache（如 L3）或主存发起请求时，系统依靠硬件 ID 来区分数据的主人。  

3. 寻址层：地址映射与 Tag RAM (Addressing & Tagging)  
核心在自己专属的 Cache 内部识别某块数据是否存在，依赖于地址切片匹配机制。  
- 当核心（通过 MMU/TLB 转换后）输出一个物理内存地址时，Cache Controller 会将该地址截断为 Tag（标签）、Index（索引）和 Offset（偏移量）。
- Index 用于快速定位数据应该存放在专属 Cache 的哪一个组（Set）中。
- 硬件会并行对比该组内所有的 Tag RAM。如果发现 Tag 完全匹配，即确认为 Cache Hit（缓存命中）。这套识别逻辑由核心专属的 Cache Controller 独立闭环完成。

4. 协议层：缓存一致性管理 (Cache Coherence)  
在多核系统中，同一个内存地址的数据可能被复制到多个不同核心的专属 Cache 中。硬件通过缓存一致性协议（如 MESI 或 MOESI 协议）来识别数据的“所有权”和“有效性”。  
- 每个专属 Cache 行（Cache Line）除了存储数据，还包含几个隐藏的状态位（如 Modified 已修改、Exclusive 独占、Shared 共享、Invalid 无效）。
- 如果核心 A 修改了自己专属 Cache 中的数据，它的 Cache Controller 会通过 Snoop Filter（监听过滤器）或 Directory（目录）向全系统广播。
- 其他核心的 Cache Controller 监听到后，会检索自己的专属 Cache，如果发现相同地址的副本，就会将其状态位标记为 Invalid（无效）。




