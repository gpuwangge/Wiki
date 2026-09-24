# 三个核心维度 (Three Dimensions)

| 维度 | 定义 | 典型特征 | 优化方向 |
| :--- | :--- | :--- | :--- |
| **ALU-bound** | 计算能力受限 | 算力接近峰值，内存等待少 | 简化 Shader 算法，减少指令数 |
| **Memory-bound** | 带宽受限 | 内存吞吐量接近峰值，算力闲置 | 压缩 Texture，减少 Render Target，优化缓存命中率 |
| **Latency-bound (Occupancy)** | 并行度/延迟受限 | 活跃线程少，寄存器压力大，stalls 高 | 减少寄存器使用，优化分支分歧，增加并行 Warp，提高 occupancy/MLP |


## Latency bound
Latency bound 表示：  
GPU 正在等待某个结果回来，但没有足够的其他 independent work 可以切换执行，因此 latency 被暴露到关键路径上。  
它可能是：  
- Memory latency：等待 L2 / DRAM load 返回。
- Cache latency：即使 L2 hit，也可能 latency 太高。
- Instruction dependency latency：后一条指令依赖前一条结果。
- Texture latency：等待 texture fetch / filtering。
- Pipeline latency：等待 fixed-function stage 输出。
- Synchronization latency：barrier、fence、queue dependency。

最关键的判断是：  
Compute utilization 低，memory bandwidth 也低；不是资源的总吞吐被打满，而是 GPU 没有足够 ready work 来隐藏等待时间。  

Hide latency的两个方法：  
- Cache：靠的是避免延迟，把数据放在离核心近的地方，让线程秒回数据
- Occupancy: 允许主线程去主存慢悠悠读取数据，同时让硬件立马切换到另一个准备就绪的线程去执行

## Occupancy
```
Occupancy = (Resident Warps per SM) / (Maximum Resident Warps per SM)
```
Resident warp和active warp：已经被分配到一个 SM、占用其 register/shared memory 等资源的 warp  
​resident warp（驻留线程束） 和 active warp（活动线程束） 通常可以认为是一个意思，它们在概念上是等价的，都指已经被分配给流多处理器（SM）并且获得了所需硬件资源（如寄存器）的 warp。  
 
GPU 遇到 memory load、texture fetch、dependency 时，一个 warp 可能暂时不能 issue instruction。Scheduler 可以切换到另一个 ready warp，继续使用 ALU。  

因此较高 occupancy 通常提供更多候选工作来隐藏 latency。GPU 的 warp scheduler 会在 ready/eligible warps 中选择能发射指令的 warp；active warps 中也可能有大量因 dependency 或资源不可用而 stalled 的 warp。  

什么限制 Occupancy
- 通常由最先耗尽的资源决定：
- 每 thread register 使用量高。
- 每 threadgroup / block 的 shared memory 高。
- Threadgroup / block 太大或资源分配粒度不合适。
- 每个 core 的最大 threads、warps、blocks 上限。
- 某些架构上的 barrier、local memory、pipeline resource 限制。

高 occupancy 不等于高性能。
- 100% occupancy 但所有 wave 都在等 bandwidth → 仍然 memory bandwidth bound。
- 50% occupancy 但 ILP 很好、cache hit 高 → 可能已足够快。
- 强行减少 registers 以提高 occupancy，可能导致 register spilling，反而增加 local-memory traffic，性能变差。

Occupancy 的本质是可用于 latency hiding 的容量指标，不是性能分数。

### 决定 Occupancy 的“短板效应”示例
假设我们在 Ampere 架构 (Max Resident Warps = 64) 上运行一个 Kernel，
设置 Block Size = 256 Threads (即每个 Block 有 $256 / 32 = 8$ Warps)：
- 理想情况：如果资源足够，SM 最多可常驻 8 个 Blocks ($8 \times 8 = 64$ Warps)，此时 ${Occupancy} = 64 / 64 = 100\%$。
- 寄存器瓶颈：如果你的 Kernel 每个 Thread 使用了 64 个寄存器，而 Ampere 单个 SM 的 Register File 总量为 65536 个 32-bit 寄存器。
    - 每个 Block 需要：$256 \times 64 = 16,384$ 个寄存器。
    - 单 SM 最多只能放下 $65536 / 16384 = 4$ 个 Blocks。
    - 实际常驻 Warps：$4 { Blocks} \times 8 { Warps/Block} = 32 { Warps}$。
    - 此时：${Occupancy} = 32 / 64 = 50\%$。

## MLP(Memory-Level Parallelism（内存层级并行度)
MLP = 同时未完成的 memory operations 的数量。  
指的是处理器/GPU 在同一时间能发出并保持多个未完成的 memory requests，例如多个 cache miss、texture fetch 或 DRAM load 同时 outstanding。  

例如某个 shader/core 在某时刻：
```
Load A：正在等待 L2 / DRAM 返回
Load B：正在等待 L2 / DRAM 返回
Texture fetch C：正在等待
Load D：正在等待
```
如果这些 request 可并发处理，那么 MLP 大致为 4。MLP 常用于描述同时处理多个 cache miss / TLB miss / memory request 的能力。  

为什么它重要  
假设 DRAM latency 是 300 cycles：  
- 只有 1 个 request outstanding：
    - 发一个 request，等约 300 cycles，再发下一个。
    - 带宽非常低，latency 被完全暴露。
- 有很多 independent requests outstanding：
    - 先后发起 request A、B、C、D……
    - 300 cycles 后开始连续返回。
    - memory pipeline 可保持 busy，延迟被并行请求覆盖。

因此：
- MLP 是把高 memory latency 转换为持续 memory throughput 的关键。

MLP 的来源  
MLP 可以来自不同层次：  
- 同一 wave 内的 independent loads
    - 例如先发多个彼此无 dependency 的 load。
- 多个 wave 同时发出 load
    - 更多 resident warps 往往有机会提高 MLP。
- 多个 threadgroup / block 并发执行。
- GPU memory subsystem 本身支持更多 outstanding request：
    - load/store queue 深度。
    - MSHR 数量。
    - cache request queue。
    - NoC / memory controller queue。
    - 每 core / per-wave outstanding transaction 限制。

Occupancy 与 MLP 的关系  
| 项目               | Occupancy                                | MLP                                                            |
| ---------------- | ---------------------------------------- | -------------------------------------------------------------- |
| 衡量对象             | 同时 resident 的 wave/warp 数量               | 同时 outstanding 的 memory requests 数量                            |
| 本质               | 可切换工作的“库存”                               | memory 访问的实际并发度                                                |
| 解决的问题            | 给 scheduler 更多 wave，以隐藏 latency          | 让 memory system 同时处理更多请求，填满 latency 空洞                         |
| 主要限制             | Registers、shared memory、max warp/block 数 | Dependency、load queue、MSHR、cache/NoC/DRAM queue、access pattern |
| 高了是否必然更快         | 不一定                                      | 不一定                                                            |
| 与 latency hiding | 间接：更多 wave 可切换                           | 直接：更多 request overlap                                          |

### 一个具体例子
假设某 GPU core 最多驻留 16 个 wave。

Case A：高 Occupancy、低 MLP
```
16 个 resident wave
每个 wave：
  x = load(ptr)
  y = load(x)       // 依赖 x，不能提前发
  z = load(y)       // 依赖 y
```
Occupancy：高，可能 100%。  
MLP：仍可能低，因为 memory access 是 serial pointer chasing。  
现象：大量 wave 都在等 dependency；bandwidth 不一定高。  
结论：memory latency bound。  

Case B：中等 Occupancy、高 MLP
```
8 个 resident wave
每个 wave：
  a = load(A[i])
  b = load(B[i])
  c = load(C[i])
  d = load(D[i])
  result = a + b + c + d
```
Occupancy：50%。  
MLP：可能高，因为 A/B/C/D load independent，可同时发出。  
现象：memory system 可持续有 request，bandwidth 利用率高。  
结论：如果带宽接近峰值，可能变成 memory bandwidth bound。  


## 算术强度
即每搬运 1 byte，做了多少 ALU 运算。
```
Arithmetic Intensity = Operations/Bytes transferred
```
通过算术强度强度可以把peak计算指令数转成数据量，与memory bandwidth做对比，找到ALU bound还是memory bound。

# GPU不同模块的bound分析
- Vertex Shader: 常见ALU-bound, vertex fetch memory-bound
- Fragment Shader: ALu-bound, texture memory-bound
- Texture Unit: memory-bound
- L2 Cache / Memory Controller: 统一统计系统内存带宽压力
- ROP(Render Output): 后期混合，常受Framebuffer带宽影响

# Benchmark分析
## Segmentation
- 首先按照Frame拆分：整体性能FPS，判断是否掉帧，区分CPU/GPU bottleneck，全局内存占用与系统带宽消耗
- 再按(render)pass：区分不同的渲染场景，比如shadowmap, g-buffer, 光照和后期处理等设置的影响，隔离depth/stencil状态与early-z剔除效率，量化tbdr的tile开销
- 再按Drawcall分段：定位哪个具体mesh/material导致bottleneck

# Bound判断逻辑简析
- 判定ALU-bound: ALU usage > 80% 并且 memory stall低
- 判定Memory-bound: memory bandwidth接近硬件上线且memory stall cycle占比高
- 判定Latency-bound: occupancy < 50% 且 non memory stall 高(通常是register不足引起)


# 架构分析: Graphics Pipeline
找到vertex bound还是fragment bound  
TBDR用于解决fragment bound问题，但会带来paramater buffer explode问题  
再通过IDVS/DVS解决  

# 架构分析: Shader


