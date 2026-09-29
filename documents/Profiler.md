# Background Knowledge
Notation 1e9: 10^9  

## Precision 
Defined in IEEE 754  
- single-precision: 32 bits(1 sign bit + 23 fraction bits + 8 exponent bits)  
- half-precision: 16 bits(1 sign bit + 10 fraction bits + 5 exponent bits)  
- double-precision: 64 bits(1 sign bit + 52 fraction bits + 11 exponent bits)  
- int32: 32 bits, -2147483648 ~ 2147483647  
- int16: 16 bits, -32768 ~ 32767  
- int64: 64 bits, -9223372036854775808 ~ 9223372036854775807  

## Time
1 second = 1e3 millisecond = 1e6 microsecond = 1e9 nanasecond  

## FLOPS
FLOPS = floating-point operations per seconds  

kiloFLOPS = 1000  
megaFLOPS = 1000,000  
gigaFLOPS = 1000,000,000 (10^9, or 1e9)  
teraFLOPS = 1000,000,000,000 (10^12, or 1e12)  
petaFLOPS = 10^15  
exaFLOPS = 10^18  
zettaFLOPS = 10^21  
yottaFLOPS = 10^24  

Example  
numOps: total number of operations of FMA  
time: unit is second  
gflops = numOps * 2 / time / 1e9  

How to know if a fma test is running at peak  
减压法  
一段shader program计算fma，numOps总量是知道的  
运行完毕后统计时间time，就可以算出gflops_1  
这个值有可能是极限算力(peak)的结果，也可能不是(还有多余算力)  
现在把fma换成fmul，运算量降低一半，重新计算gflops_2  
如果之前是peak，那么time也降低一半，gflops_2==gflops_1  
如果本来就不是peak，time不变，gflops_2明显比gflops_1低  

# Concurrency and Parallelism  
两种都是并行策略。区别是Concurrency是交替进行，Parallelism是同时进行。  
Concurrency: Two or more tasks can start, run, and complete in overlapping time periods. It doesn't necessarily mean they'll ever both be running at the same instant. For example, multitasking on a single-core machine.  
Parallelism: Parallelism is when tasks literally run at the same time, e.g., on a multicore processor.  

## ILP and DLP
指令级并行（ILP, Instruction Level Parallelism）是指利用流水级并行和多指令发射等方式提高程序执行的并行度；   
数据级并行（DLP, Data Level Parallelism）是指处理器能够同时处理多条数据的并行方式，即SIMD。  

# RenderDoc
RenderDoc 是一款开源的图形 API 帧分析器（Graphics Frame Debugger），主要用于捕捉并分析单个渲染帧的 GPU 调用、管线状态、纹理和 Buffer 数据。  

## 核心面板与分析工作流
- Event Browser（事件浏览器）
按时间顺序列出当前帧调用的所有 Draw Call（绘制指令）。可以使用搜索框按名称或渲染 API 筛选特定 Draw Call。

- Texture Viewer（纹理查看器）
查看 Render Target、Depth Buffer 及 Bind 到管线上的纹理。支持切换 RGB/Alpha 通道、调节 Gamma/曝光度以及测量像素颜色值。

- Pipeline State（管线状态）
查看当前选中 Draw Call 在 GPU 管线各个阶段（IA, VS, RS, PS/FS, OM 等）的绑定的 Shader、Buffer 资源以及 Blend/Depth/Stencil 设置。

- Mesh Viewer（网格查看器）
可视化顶点着色器（VS）处理前后的网格几何结构，帮助排查顶点变幻、裁剪或法线计算错误。

- Resource Inspector（资源检查器）
列出当前帧用到的所有 Texture、Buffer、Sampler 及 Shader 资源，可快速跳转到对应的使用点。

## 高级调试功能
- Pixel History（像素历史）：在 Texture Viewer 中右键点击某个像素，选择 Pixel History，可排查该像素为何被覆盖、过度绘制（Overdraw）或被 Depth Test 裁掉。
- Shader Debugging（Shader 调试）：在 Pipeline State 中选择对应的 Shader，可对特定顶点或像素逐行单步调试（Debug Shader Execution）。
- Resource Inspection & Export：可将抓取的 3D 网格导出为 .obj 文件，或将 Render Target 导出为 .png/.exr 格式。

## RenderDoc图例
设定Executable Path和Working Directory，点击Launch运行程序  
<img src="https://github.com/gpuwangge/Wiki/blob/main/images/RenderDoc_Launch.PNG" alt="alt text">  
运行的时候点击Capture Frame(s) Immediately捕捉当前帧，然后可以关闭程序  
<img src="https://github.com/gpuwangge/Wiki/blob/main/images/RenderDoc_Capture.PNG" alt="alt text">  
点击捕捉的帧会列出详细信息，左侧是Event Browser，记录了API级别的commands，右侧面便可以查看Mesh  
<img src="https://github.com/gpuwangge/Wiki/blob/main/images/RenderDoc_Mesh.PNG" alt="alt text">  
在Texture View界面可以点开Inputs查看shadowmap  
<img src="https://github.com/gpuwangge/Wiki/blob/main/images/RenderDoc_Texture_Input.PNG" alt="alt text">  
也可以查看Outputs的渲染结果  
<img src="https://github.com/gpuwangge/Wiki/blob/main/images/RenderDoc_Texture_Output.PNG" alt="alt text">  
打开Pipeline State会列出Graphics/Compute Pipeline的所有阶段，点击获得详细信息
<img src="https://github.com/gpuwangge/Wiki/blob/main/images/RenderDoc_Pipeline.PNG" alt="alt text">  
打开Shader Module查看器可以看SPIR-V的代码  
<img src="https://github.com/gpuwangge/Wiki/blob/main/images/RenderDoc_SPIRV.PNG" alt="alt text">  

# Nsight
NVIDIA Nsight 是一套面向 GPU 加速程序的开发、调试和性能分析工具，用于定位 CPU、GPU、图形管线或 CUDA 内核中的性能瓶颈。它尤其适合 Vulkan、Direct3D、CUDA、OptiX 等工作负载的优化。  

Nsight 的 GPU 深度分析能力基本只针对 NVIDIA GPU。它依赖 NVIDIA 驱动、CUDA 运行时和 NVIDIA GPU 的硬件性能计数器，因此无法用来对 AMD Radeon、Intel Arc 或 Apple GPU 做同等级的 shader/kernel 性能剖析、帧调试或硬件计数器分析。  

核心工具
Nsight Graphics：用于图形应用的帧捕获、调试和性能分析，支持 Vulkan、Direct3D、OpenGL、OpenXR，以及光线追踪；可检查 draw/dispatch、资源、shader 与 GPU 管线利用情况。  
Nsight Systems：系统级时间线分析工具，观察 CPU 线程、GPU 队列、CUDA API、同步、内存传输和操作系统事件之间的关系，适合先找出“整帧/整程序为什么慢”。  
Nsight Compute：CUDA/OptiX kernel 级分析器，提供 occupancy、warp 调度、内存访问、缓存、指令吞吐等详细指标，适合对已定位的热点 kernel 做微观调优。  

| 工具              | 是否必须有 NVIDIA GPU | 主要用途                                                                                                                              |
| --------------- | ---------------- | --------------------------------------------------------------------------------------------------------------------------------- |
| Nsight Graphics | 基本是              | Vulkan / D3D / OpenGL 的帧捕获、图形调试、GPU 性能分析；官方支持的硬件列表是 GeForce/Quadro/RTX 等 NVIDIA GPU。                            |
| Nsight Compute  | 是                | CUDA 与 OptiX kernel 的详细微架构分析，包括 SM 利用率、warp 行为、内存吞吐、缓存命中和指令效率等；支持的架构列表也都是 NVIDIA 架构。                           |
| Nsight Systems  | GPU 追踪部分是        | 可采集 CPU 线程、OS 调度等系统级时间线；但 CUDA、GPU activity、NVIDIA Vulkan/图形追踪等关键能力面向 NVIDIA 平台，官方目标硬件要求为 Turing 或更新的 NVIDIA GPU。 |

对 Vulkan 渲染器而言，通常先用 Nsight Graphics 抓一帧，找出耗时最高的 pass、shader 或 barrier；若怀疑 CPU 提交、GPU 空转或异步计算重叠不足，则用 Nsight Systems 看 CPU–GPU 时间线。对于 CUDA 路径追踪/降噪 kernel，再进入 Nsight Compute 分析带宽、占用率和指令效率。  

Nsight vs RenderDoc  
| 维度 | RenderDoc | NVIDIA Nsight |
|---|---|---|
| 主要定位 | Frame-capture 图形调试器 | NVIDIA 平台的调试、帧分析与性能 profiling 工具集 |
| GPU 厂商 | 跨厂商：可用于 NVIDIA、AMD、Intel 等 Vulkan 驱动环境 | GPU 深度分析主要面向 NVIDIA GPU |
| Vulkan 支持 | 支持 Vulkan capture/replay、API 与资源检查；但跨硬件 replay 不保证兼容 | 对 Vulkan 有深度支持，并覆盖 RTX 与相关 NVIDIA 特性 |
| 典型问题 | “这个像素为什么是黑的？”“descriptor/buffer/layout/barrier 绑定对吗？” | “这一帧为什么 25 ms？”“哪个 pass 限制了吞吐？”“是 shader、带宽、RT 还是同步？” |
| 帧调试能力 | 逐事件查看调用、pipeline state、资源、纹理/Buffer、shader 输入输出、像素历史等；核心工作流简单直接 | 也可捕获并检查 API state、event、资源、pipeline 与 shader；对 Vulkan/D3D12 提供 ray-tracing acceleration structure 等专用检查能力 |
| 性能能力 | 有基本事件耗时和诊断价值，但不是硬件计数器级 profiler | Nsight Graphics GPU Trace、Nsight Systems、Nsight Compute 可提供 GPU 队列/CPU 时间线、性能计数器、吞吐与瓶颈归因等深度性能分析 |
| 可扩展性 | MIT 开源，内置 Python API，可批量解析 capture、做自定义检查或报告 | 闭源 NVIDIA 工具；与 NVIDIA 驱动、CUDA、Perf SDK 深度结合 |
| API 覆盖 | Vulkan、D3D11/12、OpenGL、OpenGL ES；支持 Windows、Linux、Android 等平台 | 除 Graphics 外还有 Systems、Compute，覆盖 CUDA、OptiX、图形 workload 和系统级时间线 |


下载地址：   
https://developer.nvidia.com/nsight-systems  
https://developer.nvidia.com/nsight-graphics  


## Nsight System图例
在新的project中设定command，然后点击start   
<img src="https://github.com/gpuwangge/Wiki/blob/main/images/NsightSystem1.png" alt="alt text">  
app会运行，运行结束后会生成一个.qdrep文件，双击打开即可查看性能分析结果。  
<img src="https://github.com/gpuwangge/Wiki/blob/main/images/NsightSystem2.png" alt="alt text">  
在左侧的timeline中，可以看到各个线程的执行情况，以及各个线程的执行  
<img src="https://github.com/gpuwangge/Wiki/blob/main/images/NsightSystem3.png" alt="alt text">  



