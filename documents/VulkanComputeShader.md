# Vulkan Compute Shader

## 基础知识
workitem: 基本计算单元, 3x4x6=72个work item，都可以并行执行(但能并行多少个，要看硬件情况(通过硬件查询指令)  
workgroup: 基本执行单元, workgroup的size就是三个数，叫做workGroupSize，也叫local_size  
workitem和workgroup能设置多少(也就是能并行多少)跟硬件有关，可以query找数字  

```
layout (local_size_x = 16, local_size_y = 16, local_size_z = 1) in; 
```
以上shader code描述了compute shader的一个workgroup结构为16x16x1=256个workitem  
```
vkCmdDispatch(1024/16,1024/16,1)
```
以上host code描述了compute shader的workgroup数量为64x64x1=4096个workgroup  
以上两段代码描述了总共1024x1024=256*4096=1048576个workitem  

这10485786个workitem可以组成很多(4096个)workgroup，叫做工作组集(可以看作由很多方块搭起来的三维方块矩阵，类似魔方)，但workgroup之间不能并行，执行顺序是乱序  
假如把workgroup index为w1,w2,w3...它们是不能并行的。有可能先执行w1,也可能w2或w3  

**`为什么要引入workgroup的概念，因为只有同一个workgroup里的workitem是保证并行的`**  

## 举例
这里简要举例，详细分析见最后  

例子1: 16x16的两个矩阵计算  
第一步：算力估算  
16x16的两个矩阵计算，有256个输出元素，每个输出元素计算16次fma。
整体算力：256x16=4096，or 4096 FMA  
在 GPU 峰值算力和 GEMM 性能语境中，1 次 FP32 FMA 通常计作： 
1 multiply + 1 add = 2 FLOP  
所以 flop = 4096x2 = 8192  

第二步：设置host端每个维度的workgroup number  
本例中直接设置成1x1x1就可以了  

第三步：设置device端一个workgroup size  
因为256个独立的输出元素，有256个invocation，可以设置为16x16x1  
即一个workgroup内完成全部运算  
总共invocation次数为16x16=256  
另外每个invocation=16个fma  
256x16=4096 fma，也吻合算力估计  

假设warp size=32，一个sm容纳16个warp，硬件是如何调度的。  
需要多少个warp：用总invocation数除以warpsize, 256/32=8  
(也就是有 8个warp，共256个lane，每个lane要算16fma )    
该 workgroup 被划分为 8 个 warp。每个 warp 的 32 个 active lanes 对应 32 个输出元素；每个 lane 完成 16 次 FMA。  
SM 的 warp scheduler 会从 ready 的 resident warps 中选择 warp 发射 load/FMA 指令，并在 warp 遇到 memory 或数据依赖延迟时切换到其他 ready warp。  
因此这些 warp 在逻辑上并行完成工作，但不保证 8 个 warp 在每个 cycle 同时发射或执行。  

例子2：若要计算1024x1024的矩阵乘法  
第一步：算力估算  
1024x1024个输出，每个输出需要计算1024次fma  
整体算力：1024x1024x1024=1073741824 fma or 2147483648 flops or 2.147483648 GFLOP  
第二步：host  
1024/16 x 1024/16 x 1 = 64x64x1  
第三步：device  
还是16x16x1  
这样的话invocation数量是: 64*64*16*16=1048576  
另外每个invocation=1024个fma  
总算力：64x64x16x16x1024=1073741824fma，也跟算力估算吻合  
需要多少个warp：1048576/32=32768   
(也就是有 32768 个warp， 1048567 个  lane，每个lane要算1024fma )  

32768 是总执行量，不是同时并发量  
假设某 GPU 有 80 个 SM，并且在这个 kernel 的寄存器/shared-memory 使用条件下，每个 SM 可同时驻留 16 个 warp，那么瞬时最多可有  80×16=1280 resident warps  
它们仍会分批完成总计 32768 个 logical warps。大约需要： 32768/1280 = 25.6个驻留 warp 批次  
实际执行时间还取决于每个 warp 的 1024 次 FMA、load/store、barrier、指令发射宽度、缓存命中率、memory coalescing 和寄存器压力等。  


## Device端代码
以下是Device(GPU)代码里面workgroup维度(size)的接口(这是compute shader专有写法，省略了变量名字)  
> layout (local_size_x = 4, local_size_y = 1, local_size_z = 1) in;

### Computer Shader内建变量
Compute Shader定义了如下五个常用变量：  
```glsl
in uvec3 gl_NumWorkGroups;               //workgroup的数量，这个值可以在Host Dispatch的时候指定
in uvec3 gl_WorkGroupID;                 //每个workgroup的维度(workgroup size)，这个值一般在shader的一开始的layout处设定
in uvec3 gl_LocalInvocationID;           //每个workitem在一个workgroup里的局部ID
in uvec3 gl_GlobalInvocationID;          //每个workitem的全局ID, 相当于局部ID加上一个Offset
in unit  gl_LocalInvocationIndex;        //其实就是展开成一维的workitem index
```
**`Compute Shder的本质，就是靠workitem的全局ID和局部ID来访问数据`**


## Host端代码
以下Host代码定义可读写buffer(storage buffer)  
如果需要只读的，可以加上readonly关键字  
```glsl
layout (set = 0, binding = 0) buffer Buffer{  
     uint value[4];
} buf;
```
注意在host端，createDevice的时候需要找支持Compute的queueFamily  
> VkQueueFlagBits requiredQueueFamilies = VK_QUEUE_Compute_BIT;  

另外，storage buffer是一个uniform，需要创建descripter  
注意的是，storage buffer跟sampler一样，poolsize type里面都不含uniform关键字， descriptor layout里面也不含uniform关键字  

### Host计算阶段   
**`1.首先创建command buffer`**  
**`2.gpu-cpu 同步(Fence)`**  
**`3.录制命令及Dispach`**  
```
record command buffer()
    beginCmd
    bindPipeline
    bindDescriptor
    dispatch
    endCmd
dispatch
```
**`4.Barrior同步`**  
之后等待运算完毕  
需要barrior同步，或者device.waitIdle()强行等待  
**`5.验证(可选)`**  
把数据从device写回来：  
> unsigned int data[4] = {0}  
> memcpy(data, storageBuffer->map, sizeof(data));  

## Device内存类型
在解释Host和Device之间数据交换之前，首先需要了解Device内存类型。
### 什么是Local Memory
GPU端显存，位于芯片内部，叫Local memory。Local memory可以导出一部分供CPU访问，叫做Host Visible Local Memory，对于其他部分的Local Memory则不能被CPU访问。  
### 什么是Host Memory
CPU端系统内存，这块存储即平常所说内存，叫Host Memory，通常来说Host Memory是能够映射到GPU的虚拟地址空间供GPU访问。
### 什么是Cache Coherency
GPU和CPU都有缓存系统，叫做GPUCache和CPUCache。
有Cache就存在Cache不一致问题（Cache Coherency）。  
### 解决CPU Cache Coherency的方法
1、分配不带CPUCache的Uncache host memory，CPU直接操作内存，CPU写后能保证GPU读到最新值；GPU写到Host Memory能保证CPU读到最新值。  
2、分配带CPUCache的host memory，并由软件管理CPUCache。当CPU写GPU读的时候，GPU需要flush CPUCache的API，保证Cache被刷到内存里；当GPU写CPU读的时候，GPU需要调用Invalidate CPUCache的API，保证CPU读到最新值。  
3、分配带CPUCache的host memory，并由硬件管理CPUCache。  
### 解决GPU Cache Coherency的方法
1、分配不带GPUCache的Uncache local memory, 保证CPU读到最新值。  
2、分配带GPUCache的local memory，CPU读之前flush GPUCache，CPU写完后Invalidate GPUCache。  
(所以flush(读之前)和Invalidate(写之后)是对对方cash进行的操作)  
3、分配带GPUCache的local memory，并由硬件管理GPUCache。（实现较复杂，目前可能没有厂商实现）  
### Vulkan规定的几种内存类型
在Vulkan里创建内存的时候，需要从以下类型里选择（复选）  
**`VK_MEMORY_PROPERTY_DEVICE_LOCAL_BIT "DeviceLocal"`**  
**`VK_MEMORY_PROPERTY_HOST_VISIBLE_BIT "HostVisible"`**  
**`VK_MEMORY_PROPERTY_HOST_COHERENT_BIT "HostCoherent"`**  
**`VK_MEMORY_PROPERTY_HOST_CACHED_BIT "HostCached"`**  
**`VK_MEMORY_PROPERTY_LAZILY_ALLOCATED_BIT "LazilyAllocated"`**  
如果只需要gpu访问，就DeviceLocal。  
如果需要cpu访问，就HostVisible。  
如果需要cpu/gpu协同，就用HostCoherent。  
总而言之，对于Uniform Buffer或动态Vertex/Index Buffer使用如下组合  
**`VK_MEMORY_PROPERTY_HOST_VISIBLE_BIT | VK_MEMORY_PROPERTY_HOST_COHERENT_BIT`**  
对于仅GPU访问，CPU不会读取和写入的情况，比如Color/Depth Attachment，使用如下类型  
**`VK_MEMORY_PROPERTY_DEVICE_LOCAL_BIT`**   
需要注意的是如果只有 Host Visible 而不指定 Coherent，当 CPU 更改数据时则需要手动调用 vkFlushMappedMemoryRanges 函数将写入的数据 Flush 到设备使设备可见，当 GPU完成后需要手动调用 vkInvalidateMappedMemoryRanges 使设备写入的数据使主机可见。这个标志对数据同步的可见性和可用性没有关系，即使指定了此标志，依然需要在同步时管理数据的可用性和可见性。另外，启用 Coherent 标志也就意味着数据是写合并的，这时指定 VK_MEMORY_PROPERTY_HOST_CACHED_BIT 可能会降低性能。  
### 内存类型的性能
一般来讲，GPU访问DeviceLocal会比其他类型更快一些。  
然而，如果使用DeviceLocal内存，CPU无法直接向其写入数据。  
解决办法是：先建立一块Host可见内存，CPU把数据写入；然后使用command buffer命令GPU自己把数据从Host可见区拷贝到DeviceLocal区。(copyBuffer)    
(使用了command buffer，就要考虑同步问题。如果不用同步的话就要用waitIdle)

## Host和Device的数据交换
Host和Device的数据交换的介质是Storage Buffer。这是GPU可读写的一块内存空间。  
首先建立Storage Buffer(类型为Host Visible and Host Coherent)。  
同时声明一个mapped变量，并不需要为这个变量初始化内存，而是把它映射到Storage Buffer上
> vkMapMemory(CContext::GetHandle().GetLogicalDevice(), storageBuffers[i].deviceMemory, 0, storageBufferSize, 0, &storageBuffersMapped[i]);

这个mapped变量的用处是在Host上可以用memcpy()来拷贝。  
当mapped变量作为destination的时候，可以把host上的数据拷贝到Device的Storage Buffer上。  
> memcpy(storageBuffersMapped[currentFrame], &storageBufferObject, sizeof(storageBufferObject));

当mapped变量作为source的时候，可以把Storage Buffer的数据从Device拷贝到Host上。  
> memcpy(data, descriptor.storageBuffersMapped[renderer.currentFrame], sizeof(data));

如果对mapped memory的操作完毕后，可以通过unmap()来解除映射  
> vkUnmapMemory(CContext::GetHandle().GetLogicalDevice(), IN deviceMemory);

但事实上，mapped并不是真正的内存空间，即使不解除映射也可以。  



# 矩阵乘法中的 Workgroup、Invocation 和 Warp

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




