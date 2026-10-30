# 1 矩阵乘法中的 Workgroup、Invocation 和 Warp计算

本文使用两个简单例子说明：

- 如何估算 GEMM 的 FMA 和 FLOP 数
- Host 端如何设置 `vkCmdDispatch`
- Device 端如何设置 compute shader 的 `local_size`
- 如何计算 invocation 和 warp 数

假设使用最简单的映射：

```text
1 invocation 计算 C 矩阵中的 1 个元素。
```

并假设 NVIDIA 风格的：

```text
warp size = 32
```

> Vulkan 的通用术语是 subgroup。这里为便于讨论，假设 subgroup size 为 32，并称它为 warp。

---

## 基本规则

矩阵乘法：

```text
C[M][N] = A[M][K] × B[K][N]
```

每个输出元素：

```text
C[row][col] = sum(A[row][k] * B[k][col])
```

其中 `k` 从 `0` 循环到 `K - 1`。

因此：

```text
一个输出元素需要 K 次 FMA。
总 FMA = M × N × K。
```

如果使用：

```glsl
acc = fma(a, b, acc);
```

在 GPU 峰值算力和 GEMM 性能统计中，通常认为：

```text
1 FMA = 1 multiply + 1 add = 2 FLOPs
```

因此：

```text
总 FLOPs = 总 FMA × 2
```

注意：

```text
FLOP 是总运算量。
FLOPS 是每秒运算量。
```

只有用总 FLOP 除以 kernel 实际运行时间，才能得到 GFLOPS 或 TFLOPS。

---

## 例子 1：16×16 矩阵乘法

计算：

```text
C[16][16] = A[16][16] × B[16][16]
```

### 第一步：计算量估算

输出矩阵有：

```text
16 × 16 = 256 个输出元素
```

每个输出元素需要计算 16 次 FMA：

```text
C[row][col] = sum(A[row][k] * B[k][col])
k = 0 ... 15
```

所以：

```text
总 FMA = 16 × 16 × 16
        = 4096 FMA
```

按 `1 FMA = 2 FLOPs`：

```text
总 FLOPs = 4096 × 2
          = 8192 FLOPs
```

这表示一次矩阵乘法的总工作量。

### 第二步：Host 端设置 workgroup 数

一个 workgroup 计算完整的 `16×16` 输出矩阵：

```cpp
vkCmdDispatch(commandBuffer, 1, 1, 1);
```

所以总共有：

```text
1 × 1 × 1 = 1 个 workgroup
```

### 第三步：Device 端设置 workgroup size

```glsl
layout(local_size_x = 16,
       local_size_y = 16,
       local_size_z = 1) in;
```

一个 workgroup 中的 invocation 数量：

```text
16 × 16 × 1 = 256 个 invocations
```

每个 invocation 计算一个输出元素：

```text
1 invocation -> 1 个 C[row][col]
```

每个 invocation 做 16 次 FMA：

```text
256 invocations × 16 FMA/invocation
= 4096 FMA
```

和前面的计算量估算相同。

### Warp 视角

假设：

```text
warp size = 32
```

一个 workgroup 有：

```text
256 invocations / 32 lanes per warp
= 8 warps
```

也就是说：

```text
1 workgroup
= 256 invocations
= 8 warps
= 8 × 32 lanes
```

每个 lane 对应一个输出元素，并做 16 次 FMA。

每个 warp 的 FMA 数：

```text
32 lanes × 16 FMA/lane
= 512 FMA/warp
```

整个 workgroup 的 FMA 数：

```text
8 warps × 512 FMA/warp
= 4096 FMA
```

### 调度说明

- `16×16×1` 是一个 workgroup，不是一个 warp。
- 这个 workgroup 在 warp size 为 32 时包含 8 个 warp。
- 同一个 warp 中的 32 个 lanes 执行相同的指令流。
- SM 的 warp scheduler 会从可执行的 resident warps 中选择 warp 发射指令。
- 因此这 8 个 warp 逻辑上并行，但不保证每个时钟周期都由 8 个 warp 同时发射指令。

如果只执行：

```cpp
vkCmdDispatch(commandBuffer, 1, 1, 1);
```

那么整个 GPU 只有一个 workgroup 可做，通常只会用到一个 SM 的部分资源，GPU 利用率很低。

---

## 例子 2：1024×1024 矩阵乘法

计算：

```text
C[1024][1024] = A[1024][1024] × B[1024][1024]
```

仍使用相同的映射：

```text
1 invocation -> 计算 1 个输出元素
1 workgroup  -> 计算 1 个 16×16 输出 tile
```

### 第一步：计算量估算

输出矩阵有：

```text
1024 × 1024 = 1,048,576 个输出元素
```

每个输出元素需要计算 1024 次 FMA：

```text
C[row][col] = sum(A[row][k] * B[k][col])
k = 0 ... 1023
```

所以：

```text
总 FMA = 1024 × 1024 × 1024
        = 1,073,741,824 FMA
```

按 `1 FMA = 2 FLOPs`：

```text
总 FLOPs = 1,073,741,824 × 2
          = 2,147,483,648 FLOPs
          = 2.147483648 GFLOP
```

### 第二步：Host 端设置 workgroup 数

每个 workgroup 计算一个 `16×16` 输出 tile。

在 X 和 Y 方向上，需要的 workgroup 数为：

```text
1024 / 16 = 64
```

所以 host 端调用：

```cpp
vkCmdDispatch(commandBuffer, 64, 64, 1);
```

总 workgroup 数：

```text
64 × 64 = 4096 个 workgroups
```

### 第三步：Device 端设置 workgroup size

```glsl
layout(local_size_x = 16,
       local_size_y = 16,
       local_size_z = 1) in;
```

每个 workgroup 有：

```text
16 × 16 = 256 个 invocations
```

整个 dispatch 的 invocation 数量：

```text
64 × 64 × 16 × 16
= 1,048,576 invocations
```

这正好等于输出矩阵元素数：

```text
1024 × 1024 = 1,048,576
```

每个 invocation 计算一个 `C[row][col]`，并执行 1024 次 FMA：

```text
总 FMA = 64 × 64 × 16 × 16 × 1024
        = 1,073,741,824 FMA
```

这和第一步的计算量估算一致。

### Warp 视角

一个 `16×16` workgroup 包含：

```text
256 invocations / 32 lanes per warp
= 8 warps
```

整个 dispatch 有：

```text
4096 workgroups × 8 warps/workgroup
= 32,768 logical warps
```

也可以直接计算：

```text
1,048,576 total invocations / 32 lanes per warp
= 32,768 logical warps
```

注意：

```text
32,768 是 warp 数，不是 lane 数。
```

总 lane / invocation 数是：

```text
32,768 warps × 32 lanes/warp
= 1,048,576 lanes / invocations
```

每个 lane 做 1024 次 FMA，因此每个 warp 做：

```text
32 lanes × 1024 FMA/lane
= 32,768 FMA/warp
```

整个 dispatch 的工作量：

```text
32,768 warps × 32,768 FMA/warp
= 1,073,741,824 FMA
```

和前面的 `1024 × 1024 × 1024` 结果一致。

### Logical warp 和 resident warp

```text
32,768 logical warps
```

表示整个 dispatch 必须完成的 warp 总数，不表示 GPU 同时运行 32,768 个 warp。

例如假设：

```text
GPU 有 80 个 SM
每个 SM 对当前 kernel 可驻留 16 个 warp
```

那么 GPU 最多同时驻留：

```text
80 × 16 = 1280 resident warps
```

从 resident warp slot 的角度看：

```text
32,768 / 1280 = 25.6
```

所以需要大约 25.6 个“驻留容量批次”才能容纳所有 logical warps。

这不是精确的执行轮数或性能预测。实际运行时间还受以下因素影响：

- global memory load 和 store
- cache hit rate
- memory coalescing
- shared memory 和 barrier
- register pressure
- shared-memory 使用量
- 每个 SM 的 resident workgroup 上限
- warp scheduler 的指令发射能力
- 指令依赖和 memory latency
- 分支分歧

---

## Shared-memory tiling 的补充

如果使用 `16×16` 的 shared-memory tile：

```glsl
shared float As[16][16];
shared float Bs[16][16];
```

这里的 `16` 表示每轮处理的 K 维 tile 长度，不代表整个矩阵乘法只有 16 次 FMA。

对于 `1024×1024×1024` GEMM：

```text
K = 1024
K tile size = 16
1024 / 16 = 64 轮 K tile
```

每轮做 16 次 FMA：

```text
64 轮 × 16 FMA/轮
= 1024 FMA/invocation
```

因此 shared-memory tiling 会减少 global-memory 读取并提升数据复用，但不会改变 GEMM 的数学运算量：

```text
总 FMA = M × N × K
总 FLOPs = 2 × M × N × K
```

# 2 Memory and FIFO计算
一个 CPU 每个 cycle 可以产生 1 个 request，CPU 频率是 2 GHz。

CPU 和 Memory 之间有一个 FIFO：
- CPU：2 GHz
- CPU 每 cycle 产生 1 request
- Memory：每次处理 1 个 request
- Memory latency：100 ns
- Memory 可以连续处理 request，完成后每 cycle 可以接收一个新的 request
- FIFO 初始为空

问题：
1. 如果 FIFO 深度是 100 entries，能不能完全隐藏 memory latency？
2. 为了让 CPU 在 memory latency 期间不 stall，FIFO 至少需要多大？
3. 如果 FIFO 只有 50 entries，会发生什么？
4. 如果 memory bandwidth 只能达到 CPU request rate 的 80%，FIFO 多大才能解决这个问题？

思考：  
- FIFO是用来暂存还没被 Memory 处理的 request  
- 每cycle产生1个request，也可以处理1个request，这叫做速度匹配，这时候系统latency是可以隐藏的  
- 当看到latency的时候，就要同时想到，在latency的时间段会产生多少个outstanding request:  
```
2G * 100ns = 200 request
```
- 可以把 memory bandwidth 理解成 Memory 的“处理数据速度”  
- 当FIFO满了之后，CPU无法继续发request，这个状态叫做CPU Stall

解答：
1. FIFO 100深度不够，因为有200 outstanding request。在50ns之后，FIFO就满了，CPU会Stall
2. 200
3. CPU在50个cycle也就是50/2G=25ns之后CPU Stall
4. Memory每cycle产生0.8个request，每cycle积累0.2个request。FIFO多大都无法解决问题。  
一个cycle时间为1/2G=0.5ns。如果FIFO size=200，在200/0.2*0.5=500ns之后，FIFO就满了，CPU会Stall。


# 3 Latency相关计算
## 练习题1
某存储器接口每个 Beat(数据节拍)传输 8 Bytes(64-bit 总线)，主频与数据频率均为 2 GHz。发读命令至第一笔数据返回的延迟为 400 ns。读取 64 KB 数据时：  
1. 该读操作的 Latency 为多少？
2. 在等待 Latency 期间，内存控制器最多可维持多少个 Outstanding Requests？

解答
1. Latency = 400ns 题目已描述为“首笔数据返回所需时间”，即命令到首字节数据的通道延迟。  
但如果要计算至最后一个beat完成的时间，则  
总transaction数量为 64KB / 8B = 8K  
单次transaction传输时间 = 1/f = 1/2G=0.5ns  
全部传输完成需要时间为 8K * 0.5ns = 4096ns  
所以所有数据的读操作总耗时4496ns  

2. 最大 Outstanding Requests 无法由现有条件确定。800 只有在“一周期发一个独立请求”等额外假设下才成立；一个 Beat 不能直接等同于一个 Request。  
Max Outstanding = 400 ns / 0.5 ns per-issue = 800 个并发请求


## 练习题2
Latency / Bandwidth / Transfer Time 联动(进阶)  

内存控制器参数:  
f = 1 GHz, Beat SIZE = 8B, BURST = INCR, LEN = 7(8 Beats)。读操作延迟 = 400 ns。  
问:  
1. 单次 Transaction 的 Transfer Time 是多少？理论峰值带宽是多少？
2. 读取 64 KB 数据，若始终保持流水线满载(Outstanding 足够)，总耗时最少为多少？
3. 为掩盖 400 ns 延迟，最少需要多少个 Outstanding Transaction？
 
解析:  
1. 首先每周期传输时间为1/f=1/1G=1ns, 理论峰值带宽为： 8B/1ns=8GB/s  
Transfer有8 beats，则Transaction transfer time = 8ns

2. 假设每周期一个beat，每个beat 8B，每个transaction 8B，每个transaction数据量64B  
读取64KB数据，Transaction数量 = 64KB/64B=1K  
每个Transaction transer time = 8ns, 总传输时间为1k*8ns=8192ns  
因为latency=400ns，总耗时400+8192=8592ns  

3. 首先Latency=400ns，一个transaction transer time要8ns完成；也就是说第一个transaction发出第一个beat算起，要408ns。  
题目问的是outstanding transaction而不是outstanding request，所以  
最少需要408/8=51个outstanding transaction  

## 练习题3
A system has a 32-bit register.  
The register is set to 0. The register increments by a constant value every clock cycle. The clock frequency is 1 GHz.  
For questions 1-3 provide the formula and a numeric answer (not an estimate).  
(考察的是 整数溢出、时钟频率、位宽计算，以及 double 的整数精度)  

1. The increment value is 1. How many seconds until the register overflows?  
t=2^32/10^9=4.294967296s  
注意这里不要用2^32-1，因为问的是overflow  

2. The increment value is a 6-bit unsigned value. What is the minimal amount of time until it overflows?  
这里用了6bit而不是1bit数据，应该更快overflow所以  
4.294967296 / (2^6-1) =  4.294967296 / 63 = 0.06817408406s  
也可以先算需要多少个周期(2^32/63),再除以1GHz，结果一样  

3. If the increment value is a 6-bit unsigned value and we wanted to ensure the counter will not overflow in 1 day what would be the required width of the register?  
一天的时间是24x60x60s  
一天的总数据量是24x60x60x63G  
假设需要的宽度是x  
2^x = 24x60x60x63G + 1  
两边取log2  
x = 52.273, 往上取整，宽度至少需要53位  

4. Can the maximum value from Q3 be stored in a double precision floating point variable without losing precision?  
double能存很大的数，但是这里关键在于不能lose精度这个前提条件  
double能精确表示[-2^53~2^53]之内的每一个整数  
上一题的宽度是53, 所以不会lose精度  

进一步探索精度问题：  
这题不能用“double 占 64 bit，所以装得下 53 bit”来解释；真正决定整数是否精确的是有效数字精度，而不是总存储位数。  
IEEE 754 是目前计算机中使用最广泛的 浮点数标准。按 IEEE 754 的 32 位 float 和 64 位 double 来说：float 约有 7 位十进制有效数字，double 约有 16 位。这里的 精度 指有效数字总数，不是小数点后固定多少位。  
为什么2^32有十位十进制，但float 只约有 7 位十进制有效数字：因为并没有把所有的二进制位用于保存有效数字  
总共 32 位  
┌──────────┬──────────────┬───────────────────────┐
│ 符号 1 位 │ 指数 8 位    │ 小数部分 23 位         │  
└──────────┴──────────────┴───────────────────────┘
10^x = 2^24  
x = 7.22471989594  
同理，对于64位的double  
总共 64 位  
┌──────────┬──────────────┬───────────────────────┐
│ 符号 1 位 │ 指数 11 位   │ 小数部分 52 位         │  
└──────────┴──────────────┴───────────────────────┘
10^x=2^53  
x = 15.9545897702  
换句话说，double能精确表示[-2^53~2^53]之内的每一个整数，但超过了这个范围，相邻整数开始出现空缺。  
```
9,007,199,254,740,992     可以精确表示（2^53）
9,007,199,254,740,993     不可以精确表示
9,007,199,254,740,994     可以精确表示
```
在 [2^53,2^54) 内，相邻 double 的间距是 2，因此只能精确表示偶数整数    
如果不考虑精度的 IEEE 754 double，能表示的最大 有限数 是：  
- 有效数字略小于2  
- 指数部分有11位，其中一个符号位，最大指数是1023  
理论最大应该小于2^1024 = 1.79769313 x 10^308  


# 4 Cache计算
已知：cache data = 2KiB, line size=64B, data size=32b, address size=32b  
Direct mapping: 问题是如何寻址  
(注意知识点KB=1000B, KiB=1024B)  
tag-index-offset  

回答：  
1. 总共数据2048/64=32个cacheline，需要5b存储所有的cacheline, index需要5bits  
2. 接下来要选offset，也就是在64个Byte里面选4Byte，需要6bits  
为什么是6？我在猜测是2^6=64，后来问ai，其实不是？  
word offset: 选择64B里面的16个W,需要4bits  
byte offset: 选择word内的4个B之一，需要2bits  
完成line offset：4+2=6bits  
这个跟我原本的猜测一致？  
3. 剩余给tag的bits: 32-5-6=21bits  


# 5 Profiling计算
```
GPU frequency                 = 1 GHz
Shader Process                = 8
Shader ALU throughput         = 1024 ALU instructions / cycle / Shader Process

Texture Processor             = 16
Texture throughput            = 16 texture instructions / cycle / Texture Processor

Pixel Processor               = 4
Pixel throughput              = 16 pixels / cycle / Pixel Processor

Resolution                    = 3840 × 2160 = 8,294,400 pixels/frame
Shader ALU instructions       = 160 / pixel
Shader Texture instructions   = 6 / pixel

Measured performance           = 1000 FPS
```

合理吗?合理的话如何提高？不合理的话理论最好效能是什么？如何提高？  

profiling分析经验：  
hz：每秒多少次  
ghz：10^9 次。用这个做单位最好  
第一步：算HW Capacity  
第二步：算Workload  
第三步：算理论最大FPS，跟实测FPS对比  
第四步：找出瓶颈，优化瓶颈  

参考答案：  
- 先算每个 stage 能处理多少

Shader Processor  
有 8 个 Shader Processor，每个：  
1024 ALU instructions / cycle  

所以整个 GPU：  
8 × 1024 = 8192 ALU instructions / cycle  

GPU 是 1 GHz：  
8192 × 1 GHz = 8.192 × 10^12 ALU instructions / second  

每个 pixel 需要 160 条 ALU instruction：  
8.192 × 10^12 / 160 = 51.2 × 10^9 pixels / second  

转换成 FPS：  
51.2 × 10^9 / 8,294,400 ≈ 6173 FPS  

所以 Shader 理论上限约：  
6173 FPS  

- Texture Processor

16 个 Texture Processor，每个：  
16 texture instructions / cycle  

总吞吐：
16 × 16 = 256 texture instructions / cycle  
1 GHz：  
256 × 10^9 = 256 G texture instructions/s  

每 pixel 需要 6 条 texture instruction：  
256 × 10^9 / 6 ≈ 42.67 × 10^9 pixels/s  

换成 FPS：  
42.67 × 10^9 / 8,294,400 ≈ 5144 FPS  

所以 Texture 理论上限：  
≈5144 FPS  

- Pixel Processor

4 个 Pixel Processor，每个：  
16 pixels / cycle  

所以：  
4 × 16 = 64 pixels / cycle  
1 GHz：  
64 × 10^9 pixels/s  

转换成 FPS：  
64 × 10^9 / 8,294,400 ≈ 7716 FPS  

所以 Pixel Processor 理论上限：≈7716 FPS  

- 找 bottleneck

现在三个 stage：  
Stage	理论 FPS  
Shader	≈6173 FPS  
Texture	≈5144 FPS  
Pixel	≈7716 FPS  

所以：  
Texture < Shader < Pixel  
真正的 bottleneck 是 Texture Processor。  

因此在题目给出的简化模型下：  
理论最高性能 ≈ 5144 FPS  
而不是 6173 FPS。  

那么 1000 FPS 合理吗？  
合理。  

因为：  
Measured = 1000 FPS  
Theoretical = 5144 FPS  

实际性能只有：  
1000 / 5144 ≈ 19.4%  

也就是说 GPU 达到了这个简化理论模型大约 19% 的吞吐利用率。  
所以不能因为理论上能跑 5144 FPS，就认为 1000 FPS 不合理。  
真实 GPU 中还有大量没有在题目中体现的因素：  
```
Memory bandwidth
       ↓
Texture cache miss
       ↓
DRAM latency
       ↓
Shader occupancy
       ↓
Instruction dependency
       ↓
Pipeline bubbles
       ↓
Texture latency
       ↓
Synchronization
       ↓
Rasterization overhead
       ↓
Command / driver overhead
       ↓
Early-Z / blending / ROP behavior
       ↓
实际 FPS
```
因此 1000 FPS 完全可能。  

- 优化思路

第一优先级：优化 Texture  
提高 texture cache hit rate  
优化 sampling pattern：让相邻 pixel 尽可能访问空间局部的数据。  
使用合适的 mipmap：避免远距离纹理仍然访问过高分辨率 mip level。  
减少 texture bandwidth: 别的格式，compression  

第二步：优化 Shader  
减少 ALU instruction:比如减少不必要计算  
减少 divergence  

浓缩图：
```
                         1 GHz GPU
                            │
          ┌─────────────────┼─────────────────┐
          │                 │                 │
          ▼                 ▼                 ▼
      Shader             Texture            Pixel
      8 × 1024           16 × 16           4 × 16
      ALU/cycle          TEX/cycle          PIX/cycle
          │                 │                 │
          ▼                 ▼                 ▼
      6173 FPS           5144 FPS           7716 FPS
          │                 │                 │
          └─────────────────┼─────────────────┘
                            │
                            ▼
                     Bottleneck
                       Texture
                            │
                            ▼
                  Theoretical max
                     ≈ 5144 FPS
                            │
                            ▼
                   Measured = 1000 FPS
```