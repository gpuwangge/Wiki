# 不同标准的术语对比
| Vulkan                             | NVIDIA               | OpenCL     | 解释                                                                                                         |
| ---------------------------------- | -------------------- | ---------- | ---------------------------------------------------------------------------------------------------------- |
| 一次 compute dispatch 的全部 workgroups | grid                 | ND-range   | 一次计算任务的整体执行范围。严格来说，ND-range 描述全局 work-items 的 N 维索引空间；CUDA grid 则由多个 thread blocks 构成。   |
| workgroup / local workgroup        | thread block / block | work-group | 程序定义的协作组，包含多个执行实例；组内可以通过共享内存和组级同步机制协作。                                                   |
| subgroup                           | warp                 | sub-group  | 工作组内部更细粒度的分组，支持组内高效通信与集体操作。NVIDIA warp 为 32 个线程；Vulkan / OpenCL 的 subgroup 大小不能统一假设为 32。 |
| invocation / shader invocation     | thread               | work-item  | 最小的逻辑执行实例，即 shader / kernel 的一次执行；不是一个物理 GPU 核心。                                         |

```
Vulkan                    NVIDIA / CUDA          OpenCL
一次 compute dispatch     grid                   ND-range
└── workgroup             └── thread block       └── work-group
    └── subgroup              └── warp               └── sub-group
        └── invocation            └── thread             └── work-item
```

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
这里简要举例，详细分析见[Calculation](https://github.com/gpuwangge/Wiki/blob/main/documents/Calculation.md)    

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
还是16x16x1，表示每个workgroup size还是256  
这样的话总invocation数量是: 64x64x16x16=1048576  
另外每个invocation=1024个fma  
总算力：64x64x16x16x1024=1073741824fma，也跟算力估算吻合  
需要多少个warp(还是假设warp size=32)：1048576/32=32768   
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







