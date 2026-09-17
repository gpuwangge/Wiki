<p float="left">
  <img src="https://github.com/gpuwangge/Wiki/blob/main/images/ShaderCore1.png" alt="alt text">  
</p> 

<p float="left">
  <img src="https://github.com/gpuwangge/Wiki/blob/main/images/ShaderCore2.png" alt="alt text">  
</p> 

# 各厂商Shader Processor

| 厂商 / GPU            | 名称                                                                                 | 更细粒度的执行单元                                             | 关键理解                                            |
| ------------------- | ---------------------------------------------------------------------------------------------------- | ----------------------------------------------------- | ----------------------------------------------- |
| Imagination PowerVR | USC / Unified Shading Cluster                                                                        | USCPD、ALU pipeline、ALU instance                       | USC 是执行 shader 的半自治集群，可共享较大的纹理资源                |
| Arm Mali            | Shader Core                                                                                          | Execution Engine、算术/纹理/Load-Store 管线                  | 一个 Shader Core 能执行 VS、FS、CS；不同代际内部组织差别显著        |
| Qualcomm Adreno     | 通常称 shader processor / unified shader architecture，公开资料更常讲 ALU 与 fetch resources，而不总给出等价“cluster”商标名 | Scalar ALU、纹理/访存 fetch、线程调度资源                         | 官方面向开发者的描述强调统一、标量 shader 架构；顶点、片元、计算共享同类计算/访存资源 |
| NVIDIA              | SM / Streaming Multiprocessor                                                                        | warp scheduler、CUDA cores、LD/ST、SFU、tensor/RT units 等 | 最适合用作公开资料丰富的学习参照，但不应把其实现细节直接投射到移动 GPU           |
| AMD                 | Compute Unit（CU） 或 Workgroup Processor（较新架构中）                                                        | SIMD、vector/scalar ALU、LDS、纹理/访存单元                    | 更偏 wavefront + SIMD 的术语体系                       |

# Shader Processor功能层

| 功能层     | 问题                                            | 示例                                       |
| ------- | -------------------------------------------------- | ------------------------------------------------ |
| 前端与工作分发 | 哪个 cluster 接到 draw/dispatch 工作？如何生成 thread / task？ | PDS 与 scheduler 向 USC 投递任务                       |
| 波前/线程组织 | 一次锁步执行多少个 fragment、vertex 或 compute invocation？    | SIMD 方式执行；文档把 ALU instances 分组，组内执行相同指令          |
| 指令发射与调度 | 遇到 texture/memory latency 时如何切换到其他工作？              | USC 负责任务调度与执行，内部有 bypass、store 与多阶段 ALU pipeline |
| 算术执行    | FP32、FP16、INT、FMA、特殊函数怎样占用执行管线？                    | USCPD 的 ALU pipeline 内含多类 ALU phase              |
| 访存和纹理   | Uniform、varying、SSBO、texture 分别走什么路径、缓存和带宽？        | Common Store、iterator、纹理处理资源及 Unified Store      |
| 图形后端    | 深度模板、混合、tile memory、写回 DRAM 在哪里发生？                 | USC 结果会交给 tiling / pixel back end 等后续模块          |

# Shader Processor设计原理
## 1 GPU 基础模型
### CPU 与 GPU 的根本差别
latency-oriented vs throughput-oriented

### Compute 流水线
dispatch、workgroup、subgroup/wave/warp、invocation、barrier、shared memory

### 统一着色器架构
VS、FS、CS 不再依赖不同的“专用 programmable core”，而是竞争同一类 shader 执行资源

### 并行术语的跨厂商对照
thread/invocation、warp、wavefront、subgroup、workgroup、SIMD lane、SIMT  

吞吐、延迟、occupancy、ILP（instruction-level parallelism）和 TLP（thread-level parallelism）  

## 2 Shader Core 的微架构
### Shader core / cluster 的职责边界

### 工作如何驻留
一个 core 同时保存多少个 workgroup、wave、warp 或 thread bundle 的状态

### Register file
每线程寄存器消耗为什么会降低并发驻留数。

### 指令缓存
fetch、decode、issue、dispatch。

### Scoreboard / dependency tracking
数据相关为何导致 stall

### 多线程切换隐藏 latency
当前 wave 等 texture 时，硬件为何可发射另一个 ready wave。

### ALU 类型
FP32/FP16、INT、FMA、conversion、special function。

### Load/store pipe、texture pipe、attribute/varying interpolation

### Shared memory / LDS / local data store / PowerVR Unified Store 的异同

### Cache hierarchy
纹理缓存、L1、L2、系统内存，以及 coherent/non-coherent 路径。

### Fixed-function 与 programmable block 的协作边界。

## 3 SIMD、SIMT 与控制流
SIMD：单条指令驱动多个 lane。  

SIMT：以线程编程模型表达，但硬件将一组线程锁步执行。  

Divergence：同一 wave/subgroup 中 if/else 路径不同会怎样。  

Reconvergence：分支结束后如何重新汇合。  

Predicate / execution mask：被屏蔽 lane 不等于没有占资源。  

Quad：fragment shader 中 2×2 pixel quad 的含义，及其与 derivative（dFdx / dFdy）和隐式 mip LOD 的关系。  

Subgroup：Vulkan subgroup operations 如何暴露部分硬件执行组织，而不承诺固定 lane 数。  

循环、动态索引、函数调用和复杂控制流对寄存器压力、指令数和 occupancy 的影响。  

### shader实用分析框架  

它主要是 ALU-bound、texture-bound，还是 bandwidth-bound？  

它有多少依赖链，能否用更多独立计算隐藏 latency？  

它是否有高 divergence？  

每 invocation 的寄存器和局部存储需求是否太高？  

相邻 lane / 相邻 pixel 是否有空间局部性？  

它是否被 early-Z、tile memory 或固定功能单元的行为限制？  


# Adreno Shader Processor
Unified shader model：顶点、片元、计算使用同一类 programmable resources。  

Scalar component architecture：把向量算术、swizzle 和分量利用率问题与编译器/硬件调度联系起来。  

四个 vertex 或 pixel 组成的处理组，以及 stall 时对其他工作进行调度的思路。  

GMEM / tile memory 概念及 render target load/store 成本。  

Texture access locality、格式、带宽压缩、MSAA 以及 framebuffer bandwidth。  

Adreno Profiler / Snapdragon Profiler 中的 shader、texture、memory、ALU 等计数器和 bottleneck 分类。  

# PowerVR USC
USC 的宏观职责：geometry、fragment 与 compute workload 共享的可编程处理单元。  

USC 的组成：Common Store、USC Pipeline Datapath、iterator、DMA output 和可选 F64 datapath。  

USCPD：Unified Store、bypass FIFO 和一条 ALU pipeline 的关系。  

任务类型：vertex task、pixel task、compute task 如何被调度到 USC。  

纹理资源共享：PowerVR 的某些公开 Series 6 描述中，每对 USC 共享 Texture Processing Unit；不要把具体比例泛化到所有代际。  

指令 co-issue、ALU phases 和 dependency/bypass 的意义；PowerVR 文档说明 USC pipeline datapath 可在一个周期内向不同 phase 发射多条指令，适合用来理解“shader 指令数”不直接等价于“周期数”  


# NVIDIA SM




