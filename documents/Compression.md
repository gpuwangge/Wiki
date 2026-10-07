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



# GPU 领域常见压缩算法

如果限定在 **GPU 领域**，面试里说 `compression` 通常不是泛指 ZIP/Huffman，而更多是指 **显存、Cache、Framebuffer、Texture 等数据的压缩**。

> **GPU使用压缩的主要目的是减少 Memory Traffic 和 Bandwidth。GPU 每秒需要搬运大量数据，如果数据存在空间或时间上的相关性，就可以压缩后再写入 Memory，读取时再解压，从而减少 DRAM 带宽消耗，同时降低功耗、提高性能。**


可以把 GPU Compression 简单理解成：

```text
        Find Redundancy
              ↓
        Compress Data
              ↓
       Reduce Data Movement
              ↓
    Reduce Memory Bandwidth
              ↓
     Improve Performance
     and Power Efficiency
```

**GPU Compression 的核心价值不是“节省存储空间”这么简单，而是最重要地减少 Memory Traffic 和 DRAM Bandwidth。**

## 1. GPU 中常见的压缩类型

| 压缩类型                          | 核心思想                  | GPU 中典型用途                     |
| ----------------------------- | --------------------- | ----------------------------- |
| **Delta Compression**         | 不存绝对值，存相邻数据的差值        | GPU Memory / Cache            |
| **Frame Buffer Compression**  | 利用像素 / Tile 的相似性压缩    | Render Target / Framebuffer   |
| **Color Compression**         | 利用相邻 Pixel Color 的相关性 | Color Buffer / ROP            |
| **Depth Compression**         | 压缩相邻 Depth 值          | Depth Buffer / Z Buffer       |
| **Texture Compression**       | 固定大小 Block 压缩 Texture | Texture Memory                |
| **Sparse / Zero Compression** | 对大量 0 或无效数据进行压缩       | Memory / Tensor / Sparse Data |
| **Lossless Compression**      | 保证解压后数据完全一致           | Cache / Memory / Framebuffer  |
| **Lossy Compression**         | 允许一定精度损失              | Texture / AI/ML 数据            |


## 2. Delta Compression

### 核心思想

不直接存储数据本身，而是存储：

> 当前数据与参考数据之间的差值（Delta）。

例如原始数据：

```text
1000, 1001, 1002, 1003, 1004
```

可以转换成：

```text
1000, +1, +1, +1, +1
```

因为 Delta 通常数值很小，所以需要的 bit 数更少。

### GPU 中的意义

GPU Memory Compression 经常利用：

* Spatial Locality
* Data Correlation
* Temporal Correlation

来获得更高的压缩率。

### 面试关键词

```text
Delta
Reference Value
Data Correlation
Spatial Locality
Temporal Locality
Memory Bandwidth
Compression Ratio
```

## 3. Framebuffer / Color Compression

这是 GPU 中非常重要的一类 Compression。

假设一个 4×4 Tile 中所有 Pixel 都是相同颜色：

```text
Red  Red  Red  Red
Red  Red  Red  Red
Red  Red  Red  Red
Red  Red  Red  Red
```

没必要完整保存 16 个 Color。

可以保存类似：

```text
Color = Red
Compression Mode = Constant
```

读取时再恢复成：

```text
Red Red Red Red
Red Red Red Red
Red Red Red Red
Red Red Red Red
```

实际 GPU 中通常会有更加复杂的 Compression Mode，例如：

* Constant
* Delta
* Pattern
* Palette-like Encoding

### 主要目的

减少：

```text
GPU → DRAM
DRAM → GPU
```

之间的数据传输。

因此可以：

* 降低 DRAM Bandwidth
* 降低 Memory Traffic
* 降低功耗
* 提高 GPU Performance

## 4. Depth / Z Compression

Rendering 中的 Depth Buffer 非常大，因此 GPU 通常会对 Depth Data 进行 Compression。

例如相邻 Pixel：

```text
0.5000
0.5001
0.5002
0.5003
```

这些值非常接近，可以利用：

> Spatial Correlation

进行压缩。

## Depth Compression 和 Hi-Z 的区别

这两个概念经常容易混淆。

### Depth Compression

目标：

> **减少 Depth Buffer 的存储和 Memory Bandwidth。**

### Hi-Z / Hierarchical Z

目标：

> **快速判断一个 Primitive / Tile 是否可以被 Early-Z Reject。**

例如 Hi-Z 可以保存一个 Tile 的：

```text
Min Depth
Max Depth
```

从而快速判断：

```text
Primitive
    ↓
Hi-Z Test
    ↓
Can it be rejected?
```

所以：

> **Hi-Z 本身不是 Compression。**

但是它们经常在 GPU Rendering Pipeline 中一起出现。

## 5. Texture Compression

这是 GPU 中最经典的 Compression 之一。

常见格式包括：

* **BC1–BC7**
* **ETC2**
* **ASTC**
* **PVRTC**

例如 ASTC：

```text
4×4 Pixels
     ↓
Compressed Block
     ↓
Fixed-size Data
```

GPU 可以直接读取 Compressed Texture，然后在 Texture Unit 中进行 Decode。

因此不需要：

```text
Compressed Texture
        ↓
CPU / GPU 完整解压
        ↓
Uncompressed Texture
        ↓
DRAM
```

而是：

```text
Compressed Texture
        ↓
Texture Unit
        ↓
Decode
        ↓
Shader
```

### Texture Compression 的优势

主要减少：

* Texture Memory Footprint
* Memory Bandwidth
* DRAM Traffic

## 6. Sparse / Zero Compression

AI GPU 中越来越重要。

例如：

```text
0 0 0 0
0 5 0 0
0 0 0 0
0 0 7 0
```

如果数据非常 Sparse，就没有必要存储大量的 `0`。

可以只存储：

```text
(index, value)
```

例如：

```text
(5, 5)
(14, 7)
```

这种思想可以应用于：

* Sparse Matrix
* AI Accelerator
* Tensor Compression
* Memory Traffic Reduction


## 7. Lossless vs Lossy

## Lossless Compression

压缩之后：

```text
Original
   ↓
Compress
   ↓
Compressed
   ↓
Decompress
   ↓
Original
```

数据完全一致。

GPU 中常见于：

* Cache
* Memory
* Framebuffer
* Color Buffer
* Depth Buffer


## Lossy Compression

允许一定程度的数据损失：

```text
Original
   ↓
Compress
   ↓
Compressed
   ↓
Decompress
   ↓
Approximately Original
```

主要用于：

* Texture
* Image
* AI / ML Data

Texture Compression 中非常常见。

## 8. GPU Compression 为什么重要？

这是 GPU Architecture 面试非常常见的基础问题。

### 核心答案

> **The main goal is to reduce memory traffic and bandwidth consumption.**

GPU 每秒需要搬运大量数据：

```text
GPU
 ↓
Cache
 ↓
Memory Controller
 ↓
DRAM
```

如果数据存在：

* Spatial Correlation
* Temporal Correlation
* Repeated Values
* Zero Values

就可以进行 Compression。

例如：

```text
Original Data
     ↓
 Compression
     ↓
Less Data
     ↓
DRAM
```

读取时：

```text
DRAM
 ↓
Compressed Data
 ↓
Decompression
 ↓
GPU
```

因此可以减少：

* DRAM Bandwidth
* Memory Traffic
* Power Consumption

并可能提高：

* GPU Performance
* Effective Memory Bandwidth

## 9. GPU Compression 面试最应该掌握的 4 个

如果准备 **GPU Architecture / GPU Performance** 面试，建议优先掌握：

### ① Delta Compression

核心：

```text
Store Difference Instead of Absolute Value
```


### ② Color / Framebuffer Compression

核心：

```text
Exploit Spatial Similarity Between Pixels
```

### ③ Depth Compression

核心：

```text
Exploit Correlation Between Neighboring Depth Values
```

并理解：

```text
Depth Compression ≠ Hi-Z
```

### ④ Texture Compression

重点知道：

```text
BC1–BC7
ETC2
ASTC
PVRTC
```

以及：

```text
Compressed Texture
        ↓
Texture Unit
        ↓
Hardware Decode
        ↓
Shader
```



