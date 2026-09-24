# SIMT and SIMD
GPU 对程序员呈现的是 SIMT 模型，而底层大量算术执行通常具有 SIMD 的特征。 所以“GPU 既是 SIMD，又是 SIMT”在不同抽象层次上是对的，但不能把二者当成完全同义词。  

GPU 通常用 SIMD 风格的并行执行单元实现高吞吐计算，但把它暴露成 SIMT 线程编程模型。
所以，写 shader 时把它当作“很多线程”；做性能分析时把它当作“按 warp/wave 锁步运行的向量机器”，通常最准确。
```
GPU 对软件/编程模型通常是 SIMT；从执行资源和数据并行实现看，底层常可视为 SIMD-style。
```

| 概念 | 全称 | 视角 | 核心含义 |
|---|---|---|---|
| SIMD | Single Instruction, Multiple Data | 指令 / 硬件数据通路 | 一条向量指令同时作用于多个数据 lane |
| SIMT | Single Instruction, Multiple Threads | 编程模型 / 线程抽象 | 多个独立的 shader/CUDA thread 被成组调度，通常执行同一条指令 |

# Excution Unit
Execution Units（执行单元，简称 EU） 是 GPU 核心内部负责真正执行算术、逻辑及特定硬件加速指令的算术逻辑单元（ALU）集合。  

EU 是精密的“零件/计算单元”，而 Shader Core 是把多个 EU、调度器和缓存打包在一起的“工厂/复合核心”。

与 CPU 倾向于使用复杂分支预测和超大单核性能不同，GPU 的 EU 设计遵循 SIMT（Single Instruction, Multiple Threads，单指令多线程） 或 SIMD 模型，强调极致的吞吐量与并行计算能力。  

## EU核心组成与分类
- ALU / Core（通用算术逻辑单元）:FP32, INT32, FP64, FP16...
- SFU（Special Function Unit，特殊函数单元）: sin, cos, root...
- Tensor Core / AI Acceleration Unit（张量加速单元）: GEMM, MMA
- RT Core / Ray Tracing Unit（光线追踪单元）:BVH, Ray-Triangle Intersection
- LSU（Load/Store Unit，访存单元）: load/store from register file, L1 cache/shared memory

<p float="left">
  <img src="https://github.com/gpuwangge/Wiki/blob/main/images/warp.jpg" alt="alt text">  
</p> 

## EU执行机制：SIMT 与 Warp / Wavefront 调度
Warp / Wavefront 映射：GPU 调度器将 32 个（NVIDIA）或 32/64 个（AMD）线程打包为一个 Warp / Wavefront。  

Lockstep 执行：在一个时钟周期内，一个 Warp 中的所有线程在不同的 ALU 上执行同一条指令，但处理各自独立的数据（SIMD/SIMT）。  

分支分歧（Divergence）处理：如果 Warp 内的线程出现 if-else 分歧，硬件通常会通过掩码（Execution Mask）分两次串行执行两个分支，这会导致部分 ALU 在特定周期内处于闲置状态。  

延迟隐藏（Latency Hiding）：当当前 Warp 遇到访存（Load/Store）或高延迟计算时，Warp Scheduler 会立即无缝切换到另一个就绪的 Warp，从而保持 EU 处于满载状态。  

## EU关键设计指标
评估 GPU 执行单元效率时，主要关注以下几个维度：  
- Issue Width & Execution Latency：指令发射宽度与管线深度（Pipeline Stages）。
- FMA 吞吐量：融合乘加（Fused Multiply-Add, a x b + c）能够在单个周期内完成两次浮点操作（2 FLOPs/cycle/EU）。
- 寄存器文件（Register File）容量：EU 能否保持高占据率（Occupancy）很大程度上取决于每个线程分配到的寄存器数量。
- 结构性冲突（Structural Hazards）：不同管线（如 FP32 与 INT32，或 ALU 与 Tensor Core）之间对 Register File 读写端口的争用。


# Warp Schedular
在现代 GPU（如 NVIDIA 的 SM - Streaming Multiprocessor，或 AMD 的 CU - Compute Unit / WGP）中，硬件并不是按单线程（Thread）来调度的，而是将线程打包成固定的集合：  
- NVIDIA：32 个线程组成一个 Warp。
- AMD：32 或 64 个线程组成一个 Wavefront。

执行单元调度器（在 NVIDIA 术语中通常称为 Warp Scheduler）的核心任务，就是从当前 SM 中驻留的所有 Warp 里，挑选出准备就绪（Ready） 的 Warp，并将其指令送入硬件 Pipeline（执行流水线）。  

硬件调度的典型流程如下：  
```
+-------------------------------------------------------------------+
| SM Sub-core                                                       |
|                                                                   |
|   [ Warp Register Files / State Manager ] (例如驻留 8-16 个 Warps)  |
|                         │                                         |
|                         ▼                                         |
|             [ Instruction Buffer / Cache ]                        |
|                         │                                         |
|                         ▼                                         |
|              ┌──────────────────────┐                             |
|              │    Warp Scheduler    │ <--- 检查 Dependency/Scoreboard
|              └──────────┬───────────┘                             |
|                         │ (Issue Order / Priority Selection)      |
|                         ▼                                         |
|              ┌──────────────────────┐                             |
|              │ Instruction Dispatch │                             |
|              └──────────┬───────────┘                             |
|                         │                                         |
|        ┌────────────────┼────────────────┬────────────────┐       |
|        ▼                ▼                ▼                ▼       |
|  [ INT32/FP32 ]   [ FP64/Tensor ]   [ LD/ST Unit ]    [ SFU ]     |
+-------------------------------------------------------------------+
```

调度策略（Issue / Selection Policy）  
如果同时有多个 Warp 处于 Ready 状态，调度器会根据内嵌的硬件仲裁策略选择一个（或多个）：  
- Round-Robin（轮询调度）：在所有 Ready 的 Warp 之间公平轮换，平衡各 Warp 的推进进度。
- Greedy / LRU / Priority：优先调度最先就绪或阻塞时间最长的 Warp；在部分架构中，调度器会倾向于“连续发射同一个 Warp 的多条无依赖指令”（Instruction Level Parallelism, ILP），以最大化流水线利用率。

指令发射（Dispatch）: 选中 Ready 的 Warp 后，Dispatch Unit 会将指令广播给对应的 32 个 SIMD 通道（ALU），32 个线程在同一个周期执行同一条指令（SIMT 架构 - Single Instruction, Multiple Threads）。

## Warp Scheduler核心设计目标：隐藏延迟（Latency Hiding）
与 CPU 使用复杂的“乱序执行（Out-of-Order Execution）+ 大 Cache”来降低单线程延迟不同，GPU 依赖极高的并发量，通过 Warp Scheduler 频繁切换 Warp 来隐藏延迟。  

访存延迟隐藏（Memory Latency Hiding）：  
当 Warp A 发起一条全局显存读取指令（可能需要数百个 Clock Cycles）时，Scoreboard 会将 Warp A 标记为 Blocked。Warp Scheduler 会在下一个周期无缝零开销（Zero-Overhead Context Switch） 切换到 Warp B 执行计算，从而让 ALU 保持 100% 满载。  

零开销上下文切换的硬件基础：  
SM 内的所有驻留 Warp 的寄存器状态（Register File）在硬件上是物理分配好的。切换 Warp 只需要改变指令指针（Program Counter）和寄存器指针，不需要任何内存压栈/出栈操作，因此切换开销为 0 周期。  

# Texture Pipeline
由于纹理采样通常伴随着随机内存访问、高延迟以及庞大的数据量，GPU 架构师在硬件设计上采用了大量针对性的优化手段

## Texture内存布局与缓存架构优化（Layout & Cache Design）
- 内存布局与缓存架构优化（Layout & Cache Design）  
将 2D/3D 空间上邻近的 Texel 在 1D 物理内存中也保持邻近，使双线性和三线性采样时，所需的 Texel 极大概率落在同一个 Cache Line  

- 专用的多级纹理缓存与压缩（L1T / L2 & Compressed Cache Architecture）  
L1 Texture Cache（L1T）：为 TPU 专属定制  
带压缓存（Compressed Cache Line）：像 NVIDIA Decompress-on-Fly 技术，纹理即使在 L2 Cache 乃至 L1 Cache 中也保持压缩格式  

## Texture硬件解压缩与格式转换（Hardware Decompressor & Conversion）
- 硬件块解压缩（Block Decompression Engine）  
TPU 内置专用的硬核解压缩逻辑（如 Dedicated ASTC Decompressor）  

- 硬件 Formatting 与 Packing 单元  
将 UNORM8、SNORM16、FP16 等不同存储格式到 Shader 计算所需的 FP32 格式的转换，全部下放到 TPU 内部的专用数据转换电路（Data Convertor）中完成，完全不消耗 Shader 核心的 ALU  

## Texture采样与滤波流水线优化（Filtering & Sampling Hardware）
- Bilinear/Trilinear 滤波单元复用与并行化  
TPU 内部配备专门的线性插值算子（Lerp Unit）
三线性滤波需要跨 2 个 Mipmap 层级进行采样（共 8 个 Texel）  

- 各向异性过滤（Anisotropic Filtering）的硬件剪枝  
各向异性过滤可能需要高达 16x 的采样点（最大 16 个 Bilinear 采样）。现代 TPU 会根据屏幕空间梯度的椭圆形状，动态计算并剪除（Prune）对最终颜色贡献极低的采样点。  

## Texture延迟掩盖与并发调度（Latency Hiding & Dynamic Scheduling）
- 异步纹理指令与超深 Request FIFO（Async Texture Engine）  
非阻塞采样（Non-blocking Texture Fetch）  
超深线程并发（Massive Thread/Warp Occupancy）：当纹理未命中 L1/L2 发生高延迟 Memory Fetch（可能数百个 Cycle）时，GPU 的 Warp Scheduler 立即零成本切换到其他准备就绪的 Warp 运行，利用极高的 Thread/Warp 数量彻底掩盖纹理加载延迟  

- 纹理请求合并（Texture Request Coalescing）  
一个 Warp / Wavefront 内的 32 或 64 个 Thread 同时发出的纹理采样请求，TPU 前端的 Coalescer（合并单元） 会动态分析它们的 UV 坐标  


