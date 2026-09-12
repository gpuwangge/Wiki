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

空间局部性原理（Spatial Locality）：硬件采用 Cache Line 机制的主要依据是局部性原理——如果程序访问了内存地址 $A$，那么它极大概率很快就会访问地址 $A$ 附近的变量（例如遍历数组）。一次性拉取一整行数据，可以大幅提升后续内存访问的缓存命中率（Cache Hit）。  

对齐机制（Alignment）：Cache Line 在物理内存中是严格按其大小对齐的。例如在 64 字节 Cache Line 的系统中，内存地址 $0x00 \sim 0x3F$ 属于同一行，下一个 Cache Line 必然从 $0x40$ 开始。  

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