<p float="left">
  <img src="https://github.com/gpuwangge/Wiki/blob/main/images/cache.jpg" alt="alt text">  
</p>  

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