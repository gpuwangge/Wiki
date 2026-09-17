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

## 图例
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


