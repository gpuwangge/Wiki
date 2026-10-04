# Full Frame Compression
在 GPU 体系结构设计中，“Full frame compression” 通常指代 **Framebuffer Compression (FBC)** 或 **Lossless Memory Compression**，其核心目的是在 GPU 内部各级缓存与外部物理内存（DRAM/LPDDR）之间，大幅降低数据传输带来的极高带宽开销与功耗。  

由于图形管线（光栅化、混合、纹理采样）要求极低延迟的**空间随机访问 (Spatial Random Access)**，GPU 硬件层面上无法采用字面意义上的“整帧单流压缩”（如将一帧图像作为连续数据流压缩成类似 PNG 的单体文件）。因此，GPU 设计中的全帧级显存压缩实质上是**基于分块 (Block-Based) 或缓存行 (Cache-Line Aligned) 的全局无损压缩**。  

### 核心硬件机制：Metadata 与 Payload
GPU 会在内存中为启用了压缩的 Framebuffer 维护两个物理隔离的区域：  

1. **Metadata (控制头信息)**：将全帧划分为固定大小的像素块（如 8x8、16x16 像素，或直接对齐到 128B/256B Cache Line）。每个 Block 在 Metadata 中对应少数几个状态位（通常占总显存开销的 1% 以下），指示该数据块当前的压缩状态。  
2. **Payload (压缩数据本身)**：实际的像素数据存储区。  

在渲染过程中，Texture Fetch Unit 和 Render Output Unit (ROP / CB) 读写内存时，硬件的内存控制器 (Memory Controller) 会拦截请求，读取 Metadata 并透明地执行硬件级实时压缩或解压。Metadata 常见的压缩状态机包括：  

* **Fast Clear**：当执行 Clear Color 操作时，GPU 仅在 Metadata 中标记该块被清空，并记录基础颜色值。此时完全没有针对 Payload 的内存写入发生，节省 100% 的写入带宽。
* **Delta Compressed**：以 Block 内的某个基准像素为参考，其余像素仅存储差值（Delta）。若像素变化平缓，整个 Block 占用的 Cache Line 数量会大幅减少。
* **Uncompressed**：当画面高频细节过多，无损压缩无法缩小体积时，退回到未压缩状态，原样写入显存。

### 业界主流架构的实现方案

不同 GPU 架构对全帧内存压缩有各自的专有命名，但硬件逻辑高度收敛：  
* **Qualcomm Adreno (UBWC - Universal Bandwidth Compression)**：在移动端 TBDR (Tile-Based Deferred Rendering) 架构中，Tile Buffer 中的数据在 Resolve 到 System Memory 的阶段会被封包为 UBWC 格式。这种格式实现了跨 IP 共享，例如 GPU 渲染完 UBWC 格式的画面后，Display Engine 可以直接读取上屏，全程跳过解压。  
CCU 在将颜色数据向外输送或内部回写时，采用的正是 UBWC 格式，从而让颜色缓存的数据以压缩态高效流转。  
* **ARM Mali (AFBC - ARM Frame Buffer Compression)**：采用 4x4 或 16x16 的宏块进行无损压缩，同样强调 GPU 与 DPU (Display Processing Unit) 乃至 VPU (Video Processing Unit) 之间的全链路无损流转。  
* **桌面级 GPU (NVIDIA DCC / AMD DCC)**：Delta Color Compression 在现代桌面 GPU 中不仅用于显存，还深入到了 L2 Cache 内部，覆盖 Color Target 和 Depth/Stencil Buffer，极大缓解了光栅化后期的带宽墙。  

### 渲染管线与 API 侧的影响
在图形 API (如 Vulkan) 的底层控制中，Full frame compression 的硬件行为与资源布局 (Image Layout) 和内存屏障 (Barrier) 深度绑定：  

* 当资源处于 `VK_IMAGE_LAYOUT_COLOR_ATTACHMENT_OPTIMAL` 等专用布局时，驱动底层才会启用 UBWC/AFBC 等厂商私有的内存压缩格式，以实现最优的光栅化读写效率。
* 如果 Compute Shader 需要对该纹理进行非标准读取（如转入 `VK_IMAGE_LAYOUT_GENERAL`），或者尝试将渲染结果导出给不支持该压缩格式的外部硬件模块时，GPU 管线必须触发一次高昂的 **Decompress Pass (Resolve)**，将压缩的 Payload 展平为连续的 Linear 数据。
* 在进行显存级 Profiling 时（例如通过 RenderDoc 捕捉 Frame 分析渲染 pass，或使用 PVRTune 分析硬件带宽瓶颈），非预期的布局转换导致的强制解压（Decompression），往往是导致总线带宽激增和掉帧的核心元凶。




