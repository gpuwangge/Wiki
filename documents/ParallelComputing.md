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

# Parallel Reduction例：协作求和
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

