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
latency-oriented(CPU) vs throughput-oriented(GPU)  
CPU 优先优化单个任务的响应时间，GPU 优先优化大量任务的整体处理速度。  

### Compute 流水线
#### Dispatch  
启动一次 Compute Shader 任务。  
CPU 通过 Dispatch 指令告诉 GPU：需要执行多少个 Workgroup。  
```
vkCmdDispatch(commandBuffer, 16, 16, 1);
```
表示启动：  
16 × 16 × 1 = 256 个 Workgroups  
Dispatch 本身不等于线程数量，而是定义 Workgroup 的数量。  

#### Workgroup  
一组协同执行的线程。  
每个 Workgroup 包含多个 Invocation，它们可以通过 Shared Memory 共享数据，并使用 Barrier 进行同步。  
[Workgroup详细介绍](https://github.com/gpuwangge/Wiki/blob/main/documents/VulkanComputeShader.md)  

#### Subgroup/wave/warp  

| 厂商 / GPU        | 对应概念                                 | 典型或公开宽度             | 实务结论                                                                                                               |
| --------------- | ------------------------------------ | ------------------- | ------------------------------------------------------------------------------------------------------------------ |
| NVIDIA          | Warp；Vulkan subgroup；HLSL wave       | 32                  | CUDA 生态下固定 32；在 Vulkan 中通常也是 subgroup size 32                                                                      |
| AMD GCN / CDNA  | Wavefront / wave                     | 64                  | 传统 GCN，以及面向计算的 CDNA，核心语义是 wave64                                                                                   |
| AMD RDNA 1–4    | Wavefront / wave                     | 32 或 64             | 原生/优化方向偏 wave32，但硬件与编译器可使用 wave64；不要在 Vulkan/DX12 中无查询地假设一种                                                        |
| Arm Mali        | Subgroup / warp-like execution group | 没有可跨代保证的单一数值        | 取决于 Mali 架构代际、shader core 和编译器；应以 Vulkan runtime subgroup 查询为准                                                     |
| Qualcomm Adreno | Subgroup / wave-like group           | 随系列、档位、编译器而变        | Qualcomm 明确建议使用设备特定的 optimal subgroup size；某些近期 Adreno 文档示例中 A8x 的普通 subgroup 为 32、ray-query optimal subgroup 为 64 |
| Apple GPU       | SIMD-group / thread execution width  | Apple Silicon 常见 32 | Metal 中通过 threadExecutionWidth 查询；Apple 明确要求不要硬编码，尤其跨 Apple / AMD / Intel Mac GPU 时                                |

Wave size的含义：NVIDIA 的 warp size 固定是 32 个线程，就是一个 warp 有 32 条 lane，硬件以 SIMT 方式调度和执行它们。  
比如一个workgroup size是64，那么在NVIDIA的对应的warp就是被调度成两个warp。

| Workgroup size | 拆成的 warp   | 结果                     |
| -------------- | ---------- | ---------------------- |
| 32             | 1 × warp32 | 每个 lane 都有效            |
| 64             | 2 × warp32 | 每个 warp 都完整            |
| 128            | 4 × warp32 | 每个 warp 都完整            |
| 256            | 8 × warp32 | 每个 warp 都完整            |
| 48             | 2 × warp32 | 第二个 warp 仅 16 个有效 lane |
| 100            | 4 × warp32 | 最后一个 warp 仅 4 个有效 lane |

#### Invocation  
一个 Compute Shader 的执行实例。  
可以简单理解为一个线程实例，负责处理一个独立的计算任务。例如图像处理里处理一个像素。  

#### Barrier  
同步线程执行进度，确保协作数据按要求可见。  
假设 256 个线程共同计算一个矩阵：  
```
Step 1: 每个线程加载数据到 Shared Memory
Step 2: Barrier
Step 3: 所有线程使用 Shared Memory 中的数据
```

#### Shared memory  
Workgroup 内线程共享的高速存储空间。  
在 Vulkan GLSL 中，常见对应存储类别是 shared。  
```
shared float data[256];
```

### 统一着色器架构
VS、FS、CS 不再依赖不同的“专用 programmable core”，而是竞争同一类 shader 执行资源  
为什么采用统一架构？提高硬件资源利用率。  

### 并行术语的跨厂商对照
#### thread/invocation
Thread	通用并行计算中的线程概念  
Invocation	Shader 的一次执行实例，常见于 Vulkan / GLSL  

#### SIMD lane
SIMD Lane = 向量化执行中的一个数据通道。  
例如一个 32-lane 的 SIMD 执行资源：  
```
SIMD Execution Unit
    ├── Lane 0
    ├── Lane 1
    ├── Lane 2
    ├── ...
    └── Lane 31
```
每个 Lane 可以处理一个数据元素。  
但需要注意：  
- SIMD 是一种数据级并行执行方式。
- 不同 GPU 对线程到 Lane 的映射方式可能不同。
- SIMD 宽度不一定等于软件层面 Subgroup 的大小。


#### SIMT  
SIMT（Single Instruction, Multiple Threads）= 单条指令驱动多个线程执行。  
常用于描述 GPU 的线程执行模型。例如：  
```
同一条指令
     │
 ┌───┼───┬───┐
 ▼   ▼   ▼   ▼
 T0  T1  T2  T3
```
多个线程可以执行相同指令，但各自拥有独立的线程状态和数据。  
SIMT 和 SIMD 有关联，但不是完全相同的抽象：  
- SIMD：强调数据级并行和向量化执行。
- SIMT：强调多个线程的执行模型。

#### 吞吐（Throughput）
单位时间内完成的工作量。  
吞吐量越高，表示单位时间完成的工作越多。  
常见指标：  
- TFLOPS
- GB/s
- Samples/s
- Instructions/cycle

实际性能还取决于工作负载和瓶颈  

#### 延迟（Latency）
完成一次操作所需要的时间。  
```
发起一次内存读取
        │
        ▼
等待数据返回
        │
        ▼
执行后续计算
```

#### Occupancy
硬件资源允许同时驻留的线程数量，相对于最大驻留能力的比例。  

#### ILP（instruction-level parallelism）
指令级并行。指同一个线程内部，存在多条可以重叠执行的独立指令。例如：
```
Instruction A ──┐
                 ├── 可以并行执行
Instruction B ──┘
```
#### TLP（thread-level parallelism）  
线程级并行。指多个线程同时处于执行状态，利用不同线程之间的独立工作来提高并行度。例如：  
```
Thread 0 → 计算任务 A
Thread 1 → 计算任务 B
Thread 2 → 计算任务 C
Thread 3 → 计算任务 D
```
GPU 通常依靠大量线程提高整体吞吐量，并在某些线程等待内存时执行其他就绪线程。  

## 2 Shader Core 的微架构
### Shader core / cluster 的职责边界
Shader Core 负责“执行”，Shader Cluster 负责“组织和管理多个执行单元”。  
```
Shader Core
 ├── ALU / Vector Arithmetic
 ├── Register File
 ├── Load / Store
 ├── Special Function
 └── Thread / Wave Execution
```
```
Shader Cluster
│
├── Shader Core 0
├── Shader Core 1
├── Shader Core 2
├── Shader Core 3
│
├── Shared Cache
├── Scheduler / Dispatch
└── Other Shared Resources
```

### 一个Shader Core的工作如何驻留
GPU 不一定只执行一个 Workgroup，而是会让多个 Workgroup / Wave 同时“驻留（resident）”在一个 Shader Core 上。  
这样，当某个 Wave 因为等待内存而停顿时，GPU 可以切换到其他已经驻留的 Wave，尽量保持计算单元忙碌。  
```
Shader Core
│
├── Workgroup 0
│    ├── Wave 0
│    ├── Wave 1
│    └── Wave 2
│
├── Workgroup 1
│    ├── Wave 3
│    ├── Wave 4
│    └── Wave 5
│
└── Workgroup 2
     ├── Wave 6
     └── Wave 7
```

### Register file
每线程寄存器消耗为什么会降低并发驻留数?  
核心原因非常简单：GPU 的 Register File 是有限的，而每个线程都要占用其中一部分空间。  
因此：每线程 Register 使用量越大 → 同一个 Shader Core 能同时保存的线程越少 → 能驻留的 Wave/Workgroup 越少 → Occupancy 可能下降。  
Register File 是每个 Shader Core 上有限的高速存储资源；每线程使用的 Register 越多，同一个 Core 能同时保存的线程/Wave 就越少，因此可能降低 Occupancy。  

### 指令缓存与指令流水线
Instruction Cache 负责“把指令准备好”，Fetch → Decode → Issue → Dispatch 负责“把指令送进执行单元”。  
Instruction Cache（I-Cache）= 缓存 Shader 程序的机器指令。  
Fetch = 取指令。  
Decode = 指令解码。  
Issue = 选择一条“现在可以执行”的指令，并把它发给对应执行资源。  
Dispatch = 将已经选择好的指令发送到具体执行单元。  

### Scoreboard / dependency tracking
数据相关为何导致 stall  
核心思想：GPU 不能随便执行有数据依赖的指令。  
Scoreboard 可以理解成一个“指令依赖登记表”，记录哪些寄存器/资源正在等待结果。  
Scoreboard 负责跟踪“谁产生了什么数据、数据什么时候 ready”；当一条指令依赖的数据还没准备好时，它不能被 Issue，于是产生 dependency stall。  
一个 Wave Stall，不代表整个 Core Stall；Scheduler 可以切换到其他 Ready Wave，用 TLP 把这个等待时间隐藏掉。  

### 多线程切换隐藏 latency
当前 wave 等 texture 时，硬件为何可发射另一个 ready wave。  
GPU Core 同时驻留多个 Wave，并且保存着它们各自的执行状态。因此一个 Wave 等待 Texture/Memory 时，Scheduler 可以直接选择另一个 Ready Wave 发射。  
GPU 通过在 Shader Core 上同时驻留多个 Wave，并保存它们的寄存器和执行状态，使 Scheduler 能在一个 Wave 因 Texture/Memory 等待而 Stall 时，立即选择另一个 Ready Wave 发射，从而用 TLP 隐藏长延迟。  

### ALU 类型
FP32/FP16、INT、FMA、conversion  
special function(SFU）:sin,cos,exp,log,sqrt,rsqrt...  

### Load/store pipe、texture pipe、attribute/varying interpolation
这三个可以理解成 Shader Core 周围的三类不同“数据供给路径”：  
ALU 负责算，Load/Store 负责搬数据，Texture Pipe 负责取纹理，Interpolation 负责给 Fragment Shader 准备插值后的输入。  
```
 Shader Core
                         │
          ┌──────────────┼──────────────┐
          ▼              ▼              ▼
      Load/Store      Texture       Interpolation
          │              │              │
      Buffer/Memory   Texture Cache   Attributes/
                                     Varyings
          │              │              │
          └──────────────┼──────────────┘
                         ▼
                        ALU
```

### Shared memory / LDS / local data store / PowerVR Unified Store 的异同
Shared Memory，也叫 LDS(Local Data Share）主要解决“同一个线程组里的线程如何快速共享数据”；  
一般是指位于 GPU 芯片内部、靠近计算单元的一小块高速 SRAM，供同一个 thread block / workgroup 中的线程协作读写。它不是显存 VRAM，也不是 Windows 任务管理器里显示的“共享 GPU 内存”。  
典型用途是线程协作、数据复用。  

PowerVR Unified Store 则是 PowerVR 特有的统一片上存储架构，概念范围更大  

Shared Memory 是编程模型概念；  
LDS / Local Data Store 是厂商或架构层面的实现概念；  
PowerVR Unified Store 则是 PowerVR 特有的更广义片上统一存储架构，不能简单等同于 Compute Shader 的 Shared Memory。  

### Texture Cache
Texture Cache 专门优化 Texture Sampling 的访问模式。  
Texture 访问和普通 Load 不完全一样，因为 Texture Unit 通常还需要处理：  
- Texture address
- LOD
- Mipmap
- Filtering
- Format conversion
- 多个 texel 的读取

### Fixed-function 与 programmable block 的协作边界。
Fixed-function 负责规则明确、重复度极高的工作；Programmable block 负责需要灵活编程的计算。  

## 3 SIMD、SIMT 与控制流
SIMD：单条指令驱动多个 lane。  

SIMT：以线程编程模型表达，但硬件将一组线程锁步执行。  

Divergence：同一 wave/subgroup 中 if/else 路径不同会怎样。  

Reconvergence：分支结束后如何重新汇合。  

Predicate / execution mask：被屏蔽 lane 不等于没有占资源。  

Quad：fragment shader 中 2×2 pixel quad 的含义，及其与 derivative（dFdx / dFdy）和隐式 mip LOD 的关系。  
在 GPU Fragment Shader 中，Quad 通常指一个 2 × 2 的相邻 Fragment/Pixel 组：  
```
       x →
      0     1
    ┌─────┬─────┐
 y0 │ F00 │ F10 │
    ├─────┼─────┤
 y1 │ F01 │ F11 │
    └─────┴─────┘
```
它非常重要，因为 Fragment Shader 中的：  
- dFdx()
- dFdy()
- 隐式 Texture LOD
- texture() 的 mipmap 选择

都与这种 局部 2×2 邻域密切相关。  
为什么 Texture Sampling 需要 derivative？GPU 怎么知道应该选择哪个 mip level?关键就是：看 UV 在屏幕上的变化速度。  

Subgroup：Vulkan subgroup operations 如何暴露部分硬件执行组织，而不承诺固定 lane 数。  

循环、动态索引、函数调用和复杂控制流对寄存器压力、指令数和 occupancy 的影响。  
循环、动态索引、函数调用和复杂控制流真正影响 GPU 性能的关键，不是语法本身，而是它们经过 Compiler 后形成多少指令、多少 live values、多少寄存器，以及多少 Wave divergence。  

### shader实用分析框架  
shader主要是 ALU-bound、texture-bound，还是 bandwidth-bound？  
shader有多少依赖链，能否用更多独立计算隐藏 latency？  
shader是否有高 divergence？  
每 invocation 的寄存器和局部存储需求是否太高？  
shader warp相邻 lane / 相邻 pixel 是否有空间局部性？  
shader是否被 early-Z、tile memory 或固定功能单元的行为限制？  
```
                  Shader
                            │
        ┌───────────────────┼───────────────────┐
        │                   │                   │
        ▼                   ▼                   ▼
   Compute / ALU        Memory / Cache      Parallelism
        │                   │                   │
   ALU-bound          Texture/             Occupancy
                      Bandwidth-bound      Dependency
                                            Divergence
        │                   │                   │
        └───────────────────┼───────────────────┘
                            │
                     Pipeline Interaction
                            │
             ┌──────────────┼──────────────┐
             ▼              ▼              ▼
          Early-Z        Tile Memory    Fixed Function
```
分析一个 Shader 时，不要只看 ALU utilization 或 instruction count。更实用的方法是从 计算、内存、并行度、控制流、数据局部性、固定功能 六个方面判断瓶颈。  
首先判断：ALU-bound、Texture-bound 还是 Bandwidth-bound？  
第二步：分析 Dependency Chain即使 ALU 很多，也不代表 GPU 能把 ALU 喂满。  
第三步：分析 Divergence  
对于 Warp/Wave：  
```
if (condition)
    A();
else
    B();
```
如果所有 lane 都走同一路：
```
Lane:
AAAAAAAAAAAAAAAAAAAAAAAA
```
效率通常较高。如果：
```
AAAAAAAAAAAABBBBBBBBBBBB
```
则需要处理不同 execution path。  
概念上：  
```
        Branch
        /    \
       A      B
       │      │
  active lanes
       │      │
       └──┬───┘
          ▼
        Merge
```
因此：Divergence 的核心成本是减少 SIMD/SIMT execution efficiency，而不是简单增加“branch instruction”数量。  
第四步：每 Invocation 的 Register / Local Storage. 这是判断 Occupancy 的关键。  
第五步：Warp Lane / Pixel 的空间局部性这对 Texture 和 Memory 非常重要。  
第六步：Early-Z / Tile Memory / Fixed Function  

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
NVIDIA 的 SM（Streaming Multiprocessor） 是 GPU 中执行 CUDA / Shader workload 的核心计算单元。可以把它理解为 NVIDIA GPU 中组织 Warp、寄存器、调度器和各种执行单元 的基本计算模块。  

| 特点                       | 简单理解                                      |
| ------------------------ | ----------------------------------------- |
| **Warp-based execution** | 通常以 **32 threads / Warp** 为基本 SIMT 执行组织   |
| **多个 Warp Scheduler**    | 同时管理多个 Warp，在 latency 出现时切换到其他 ready Warp |
| **大量执行单元**               | FP32、INT、SFU 等不同类型的计算资源                   |
| **Tensor Cores**         | 针对 Matrix / AI workload 的专用计算单元           |
| **大 Register File**      | 保存大量 resident Warp 的线程状态                  |
| **Shared Memory / L1**   | 提供低延迟的片上数据存储和缓存                           |
| **高并发 Residency**        | 一个 SM 可以同时驻留多个 Warp / Thread Block        |
| **Latency Hiding**       | 一个 Warp 等待 Memory 时，可以执行另一个 ready Warp    |

NVIDIA SM 是以 Warp 为基本执行组织、以 Scheduler 为调度核心、以 Register/Shared Memory 为片上状态和数据存储，并通过多种执行单元实现高吞吐和 latency hiding 的 GPU 核心计算模块。  

