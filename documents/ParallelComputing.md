# GEMM
这个case是matrix mutiply  
```c++
C = A * B
A: MxN
B: NxK
C: MxK
```
分析：invocation的数量是MxK, 也就是说每个invocation需要计算的是C[m*K+k]  
m和k是invocation的属性  
n是本invocation的循环(0~N)  
每次循环累加如下：  
```c++
C[m][k] += A[m][n] * B[n][k]
```
或  
```c++
C[m*K+k] += A[m*N+n] * B[n*K+k]
```
在这个最基础的 GEMM 实现中，每个 invocation 独立计算一个输出元素，不需要 shader 内部的 barrier()。如果之后优化为 tiled GEMM，让多个 invocation 协作加载和复用 shared memory，就通常需要 Workgroup 内同步。  
每个 thread 独立求自己的局部和：不需要 barrier。  

# Parallel Reduction例：归约求和
Assume an assembly-like shader language in a compute shader with 1024 threads.  Write a program to sum all values in a buffer of size 256KB.  Each value is 32 bits(4B).  

分析：  
这题的核心是 归约（reduction）：先让每个线程计算一部分数据的和，再把 1024 个线程的局部和合并成一个结果。这样不需要每读一个数就对同一个全局地址做 atomic add。  
关于K这个单位：有可能指1000，也可能指1024，要看具体情况，这里是1024。  
(归约就表示需要barrier)
有几个value: 256k/4=64k个value  
有多少并行thread：1k  
每个thread要循环64次  
这里要做一个假设：这1k thread术语同一个workgroup，能使用shared memory和组内barrier。  

要点1：对于每个thread，有一个tid  
要点2：用一个share memory来记录中间结果(1024个局部和)  
要点3：这里的 barrier 非常重要：它保证所有 threads 都已经把自己的局部和写入 shared，Thread 0 才开始读取。  

有如下buffer  
```c++
int buffer[N]; // sizeof(buffer) = 256KB
```
shader实现如下：  
```c
uint sum = 0;
for (int i = 0; i < 64; i++)  sum += buffer[64 * tid + i]; //第一阶段

shared[tid] = sum;

barrier();

if (tid == 0) {
    uint final = 0;
    for (int i = 0; i < 1024; i++)  final +=  shared[i]; //第二阶段
    output[0] = final;
}
```
要点4：如果要进一步改进算法，可以用Tree reduction，改善第二阶段加法(把1024次加法并行执行)，通常更适合大规模并行归约  

要点5：从内存排布来看如何优化第一阶段的计算  
第一种读法：  
```c++
for (uint k = 0; k < 64; ++k) {
    sum += inputBuffer.values[tid + 1024 * k]; //优化的读法
}
```
thread 0:  0, 1024, 2048, ... , 64512  
thread 1:  1, 1025, 2049, ... , 64513  
thread 2:  2, 1026, 2050, ... , 64514  

或者叫 **跨步访问(Strided Access)**  
（Stride = 1024）  
**虽然同一个线程的两次读取相隔很远，但同一时刻不同线程的读取是连续的。这正是 GPU 合并内存访问所需要的模式。**   

这个技巧的核心概念是 **Coalesced Memory Access（合并内存访问）**  
(单词 coalesced 的读音为"科·欧·莱斯特") 

第二种读法：   
```c++
for (uint i = 0; i < 64; ++i) {
    sum += inputBuffer.values[64 * tid + i]; //原本的算法
}
```
thread 0:    0,   1,   2, ... ,  63  
thread 1:   64,  65,  66, ... , 127  
thread 2:  128, 129, 130, ... , 191  

这种读法叫 **Spatial Locality（空间局部性）**  
**访问某个地址后，附近地址也可能很快被访问**


为什么第一种通常更快？  
关键是竖着看“同一轮，各线程读什么”：第一种相邻线程地址相差 4 bytes(对于 4-byte 元素)，一次读取可以取出很多线程需要的数据。第二种相邻线程地址相差 256 bytes。  
而不是横着看“一个线程先后读什么”: 一个线程读连续的内存无法优化，因为warp是很多线程同时dispatch的。  
同一个 warp 的 load 会按各线程地址合并成尽可能少的内存事务。  

以 CUDA 的一个warp内 32 个线程为例：   
| 循环轮次 | 第一种（优化）：thread 0～31 | 第二种：thread 0～31 |
|---|---|---|
| 第 0 轮 | `0, 1, 2, …, 31` | `0, 64, 128, …, 1984` |
| 第 1 轮 | `1024, 1025, …, 1055` | `1, 65, 129, …, 1985` |
| 第 2 轮 | `2048, 2049, …, 2079` | `2, 66, 130, …, 1986` |

第一种：  
32 个线程读取连续的 128 bytes  
涉及 4 个 32-byte 区段（每次读32byte数据，要读4次）  

第二种：  
32 个线程读取相隔 256 bytes 的地址  
涉及 32 个不同的 32-byte 区段（每次读32byte数据，要读32次）  

可以把它记成一句话：GPU 上应优先让“一群线程这一轮一起读连续地址”，而不是只让“一个线程连续读地址”    

还有一个重要细节：Spatial Locality 和 Coalescing 并不是互斥的概念。 第一种方式在同一个 warp 的一次内存指令中具有很好的空间连续性；第二种方式则具有较好的单线程连续访问特征。实际性能还会受到缓存、内存事务粒度、线程块大小及 GPU 架构的影响。

总结一下：  
| 概念 | 含义 | 在这里的体现 |
|---|---|---|
| Spatial Locality（空间局部性） | 访问某个地址后，附近地址也可能很快被访问 | 第二种读法中，同一个线程连续读取附近的元素 |
| Coalesced Memory Access（合并内存访问） | GPU 将同一 warp 中多个线程的内存请求合并为较少的内存事务 | 第一种读法中，同一轮迭代的线程访问连续地址 |
| Strided Access（跨步访问） | 连续访问之间存在固定步长 | 第一种读法中，同一线程相邻两轮访问的步长为 1024 个元素 |


## CUDA版本的实现
```c
__global__ void sumKernel(const int* buffer, unsigned int* output){
    __shared__ unsigned int shared[1024];

    int tid = threadIdx.x;

    // 第一阶段：每个 thread 计算自己的局部和
    unsigned int sum = 0;
    for (int i = 0; i < 64; i++) {
        sum += buffer[64 * tid + i];
    }

    shared[tid] = sum;

    // 等待所有 threads 写入 shared memory
    __syncthreads();

    // 第二阶段：Thread 0 串行累加所有局部和
    if (tid == 0) {
        unsigned int final = 0;

        for (int i = 0; i < 1024; i++) {
            final += shared[i];
        }

        output[0] = final;
    }
}
```
调用方式：
```c++
sumKernel<<<1, 1024>>>(buffer, output);
```

## 按SM(streaming multiprocessor)分析如何配置归约求和
题目给了总thread数量，但没给workgroup number和size，就是默认1个workgroup。  
下面分析如何确定使用多少workgroup和设置其他参数。  

每个SM有32KB共享内存，64K寄存器  
根据这些资源，每个SM可以有若干workgroup驻留, 每个workgroup可以有若干个thread(NV架构是32个thread组成一个warp)，每个thread有若干个32-bit寄存器  
每个SM有若干个warp scheduler进行调度  

### Shared Memory限制SM上可以驻留的workgroup数量(32K/4K=8)
Shared memory是可以在同一个workgroup内共享的  
如果一个 SM 有 32 KiB 可用于 workgroup 的 **shared memory** ，每个 workgroup 使用 4 KiB，那么仅考虑 shared memory，最多可同时驻留 8 个 workgroup。  
这是 shared-memory 约束给出的上限，不等于最终一定能驻留这么多组。  

### Register限制了SM上总thread数量,进而限制workgroup数量(64K/64/128=8)
寄存器：通常按每线程用量计算，他的使用是一个线程内使用的私有资源  
首先区分单位：这里的 64K registers 通常指 65536 个 32-bit 寄存器，即 256 KiB 的寄存器存储  
每线程寄存器数如果是64，则限制了总thread数量是1024个  
假设每个workgroup有128个thread（即workgroup size=128），则限制了驻留的workgroup数量为1024 / 128 = 8个  

### Warp数量决定实际延迟
在 NVIDIA 上，一个 warp 是 32 个线程  
按照上述例子限制是8个workgroup，每个workgroup128个thread，所以一共32个warp  
按照一般经验，SM最好有8～32个workgroup，每个workgroup 128~256线程，总共512～1024线程，可以有效隐藏内存延迟

## 小知识：单个workgroup，thread的最大数量1024
一般来讲，一个workgroup默认最大的thread就是1024(2d 就是32x32)或512。为什么？  
原因：  
1. 每个线程需要有私有的register，一个计算单元sm中的总寄存器数量有限(比如64k个32bit regs，按1k thread数量看，每个thread分得64个), 如果thread不限制，则每个thread的reg太少，导致reg spill to local memory，性能下降。
2. 太多线程带来太多barrier压力，因为barrier要等太多线程
3. 单个workgroup size过大，会造成active warp减少，warp occupancy太低，降低hide latency能力
4. workgroup size过大，破坏l1 cache locality

当然workgroup size太小也不好，因为这样会导致线程间共享数据的能力弱。

## 综合分析：64K个数据，分8个workgroup，workgroup size=128，共1024个thread
对于64K个数据:  
1. 总thread数不变，每个thread还是处理64个数字
2. 跨 workgroup 合并与组内归约轮数  
普通 compute dispatch 中，没有可以让所有 workgroup 一起等待的全局执行 barrier，因此不能直接用组内 barrier() 完成跨组归约。  
可以选择：  
- 每组输出一个部分和，得到 8 个部分和，再由 host 或第二次 dispatch 合并。
- 每组完成组内归约后，由一个线程对同一个全局结果执行 atomicAdd，在一次 dispatch 中完成累加。结果需提前清零并正确同步；如果输入是 float，还需要支持并启用相应的浮点原子加功能。  

采用每轮将参与元素数减半的树形归约时： 
| 配置                        | 每组树形归约轮数                                    |
| ------------------------- | ------------------------------------------- |
| 1 个 workgroup，每组 1,024 线程 | log⁡2(1024)=10 |
| 4 个 workgroup，每组 256 线程   | log⁡2(256)=8      |
| 8 个 workgroup，每组 128 线程   | log⁡2(128)=7      |

3. 效率上，将一个大组拆成多个小组，可能增加跨 SM 分配的机会，提高整张 GPU 的利用率。
4. 共享内存占用上，如果只有一个workgroup需要1024x4B=4KB；如果有8个workgrouo每个只需要128x4B=512B
5. Register数量上，全部线程同时驻留在同一个 SM，64K/1K=64,即每个thread不能使用超过64个register


# Share Local Memory 和 Register对Occupancy的影响
在Vulkan Compute Shader里，Share Local Memory（SLM）使用shared关键字标记。表示同一个workgroup可以读写。  
SLM主要用于线程间交换数据。比如多个线程协作加载一份数据，之后反复从 shared 读取。  
跟Register对比：register 通常是单个 invocation 的私有值，由编译器自动分配。register访问成本比slm更低。  

SLM会限制可驻留workgroup数。  
REG会限制可驻留线程数。  
所以两者都会影响Occupancy。  

举例，下面是一个最小的组内交换例子：每个线程加载一个元素，然后读取组内下一个线程加载的数据。假设输入和输出是不同 buffer，且都至少包含 count 个元素。
```c
#version 450

// 每个 workgroup 有 256 个 invocation。执行到 barrier() 的 invocation 会等待：只有同组的 256 个 invocation 都到达这个同步点，才允许继续执行后面的代码。  
layout(local_size_x = 256) in;

layout(set = 0, binding = 0, std430) readonly buffer Input {
    float src[];
};

layout(set = 0, binding = 1, std430) writeonly buffer Output {
    float dst[];
};

layout(push_constant) uniform Params {
    uint count;
} pc;

shared float tile[256];

void main()
{
    uint lid = gl_LocalInvocationIndex; ////每个workitem在一个workgroup里的局部ID
    uint gid = gl_GlobalInvocationID.x; //每个workitem的全局ID

    // 越界线程仍参与初始化与同步。
    tile[lid] = gid < pc.count ? src[gid] : 0.0;

    barrier(); //barrier() 同步的是 同一个 workgroup 内的所有 invocation

    // 此时可以读取同组其他线程写入的数据。
    float value = tile[(lid + 1u) % 256u];

    if (gid < pc.count)
        dst[gid] = value;
}

```
可以理解为：  
```
同一个 workgroup：

invocation 0   写 tile[0]   ──┐
invocation 1   写 tile[1]   ──┤
...                           ├── barrier ── 然后各自读取 tile
invocation 255 写 tile[255] ──┘
```
这个程序具体输出什么？  
```
输入：src[0], src[1], src[2], ... , src[254], src[255]
输出：src[1], src[2], src[3], ... , src[255], src[0]
```
也就是每组内部循环左移一位。  

为什么用这个例子？只是为了展示三个步骤：
```
每个线程写自己的 shared 槽位
                 ↓
             barrier()
                 ↓
每个线程读其他线程写入的槽位
```
没有 barrier()，读取时不能保证邻居已经完成写入。

不过，这个操作本身完全可以直接计算源索引并读取 SSBO(Shader Storage Buffer Object)，不必绕 shared memory。我之前应该更明确地说明：这是同步机制的教学例子，不是展示 SLM 性能优势的好例子。  


