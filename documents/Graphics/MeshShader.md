# Keynotes
- Host 通过 vkCmdDrawMeshTasksEXT(cmd, T, 1, 1) 记录绘制命令。当 GPU 执行该命令时，如果当前图形管线包含 Task Shader，就启动 T 个 Task Shader Workgroup。每个 Task Shader Workgroup 通过 EmitMeshTasksEXT(M, 1, 1) 启动 M 个 Mesh Shader Workgroup，因此总共启动 T×M 个 Mesh Shader Workgroup。
- 在 Vulkan API 和 shader 执行模型中，Meshlet 不是一个原生的执行单位，而是应用自行定义的几何数据分块。常见的组织方式是让一个 Mesh Shader Workgroup 处理一个 Meshlet。因此，在每个 Workgroup 都处理一个对应 Meshlet 的前提下，这次绘制处理 T×M 个 Meshlet；这种一对一映射是应用约定，而不是 Vulkan 的强制要求。
- EmitMeshTasksEXT 是 workgroup 级的发射操作，而不是组内每个 invocation 各自追加一批 workgroup。它定义后续 Mesh Shader Workgroup 的网格大小，并终止当前 Task Shader Workgroup 的执行。

- 在典型的实现中，1 个 Task Shader Workgroup 并不只处理 1 个 Meshlet，而是负责评估一组（例如 t 个）候选 Meshlet。 总结 t 的作用：它是 Meshlet 级别的并行过滤器，避免用一整个 Workgroup 只算 1 个包围盒而造成的硬件算力浪费。
- 在典型的实现中，1 个 Mesh Shader 有 m 个 Invocation。m 个线程（例如 m = 32 或 m = 128）就在这里负责并行拆解并生成这个 Meshlet 的内部数据。并行处理顶点（替代传统 Vertex Shader）。

- Task Shader 可以概括为对后续几何处理工作的“增、减、改”：增，是将一个任务展开为多个 Mesh Shader Workgroup；减，是通过提前剔除避免不必要的 Mesh Shader 工作；改，是通过 LOD 等策略选择不同的几何任务。其本质是决定后续工作“做多少、做什么”。
- Mesh Shader 可以概括为对几何数据的“读、算、写”：读，是读取 Meshlet 的顶点、索引等输入数据；算，是进行顶点变换、属性计算及可选的图元剔除；写，是输出顶点属性和图元连接关系，交给后续光栅化阶段。其本质是决定几何“长什么样”。

- Mesh Shader 的主要代价是：用更高的实现复杂度和调优成本换取几何处理的灵活性；分块或资源配置不当时，性能甚至可能不如传统顶点管线。

# Mesh Shader介绍
Mesh Shader是为了解决ras之前geometry管线的缺点：很多vertex被处理了之后却因为种种原因被丢弃。  

Task Shader: 扫描屏幕和相机，决定哪些物体该算，哪些该直接跳过。  
Mesh Shader：只处理挑中的物体，可以细分物体。  
Fragment Shader：接受精修过的三角形。  

输入mesh shader的是payload，而不是传统数据格式。  

mesh shader并不是全场景通用，而是适合特定场景。比如适合开放世界海量实体渲染。不适合简单场景。  

# Mesh Shader计算举例(Vulkan)
对于一个mesh shader example，host端调用vkCmdDrawMeshTasksEXT(cmd, 100, 1, 1);  
taskshader: layout(local_size_x = 32) in;  
meshshader: layout(local_size_x = 64) in;  
问有多少个meshlet，task shader和mesh shader各执行多少次  

| 项目                             | 数量                                         |
| ------------------------------ | ------------------------------------------ |
| Task shader workgroup          | 100                                        |
| 每个 task workgroup 的 invocation | 32                                         |
| Task shader 总 invocation       | 100×32=3200100 \\times 32 = 3200100×32=3200 |
| Mesh shader workgroup          | 取决于 task shader 发射多少组                      |
| 每个 mesh workgroup 的 invocation | 64                                         |

假设每个 task workgroup 都执行
```
EmitMeshTasksEXT(1, 1, 1);
```
那么总共发射 100 个 mesh workgroup，mesh shader 总 invocation 为 100×64=6400。这里是每个 task workgroup 发射 1 个 mesh workgroup，不是其中 32 个 invocation 各自发射 1 个。  

如果每个 task workgroup 都执行：  
```
EmitMeshTasksEXT(32, 1, 1);
```
那么总共发射 100×32=3200 个 mesh workgroup，mesh shader 总 invocation 为 3200×64=204800。若不同 task workgroup 发射数量不同，就把各组的发射数量相加。  

Meshlet 数量与这些数字的关系  
常见设计是一个 mesh workgroup 处理一个 meshlet，但这是程序的数据组织约定，不是 local_size_x = 64 自动规定的；64 表示组内有 64 个 invocation 协作，并不表示有 64 个 meshlet，也不直接规定顶点数。  
所以，若采用“一组 mesh shader 处理一个 meshlet”的设计：  
| Host 调用     | 每个 task workgroup 发射的 mesh 组数 | 本次处理的 meshlet |
| ----------- | ----------------------------- | ------------- |
| (100, 0, 0) | 任意                            | 0             |
| (100, 1, 1) | 1                             | 100           |
| (100, 1, 1) | 32                            | 3200          |
后两行都需要相应的 task shader 发射逻辑，不能仅凭两个 local_size_x 推导出来。  

结果可以整理成：
| 项目                                            | 每个 task workgroup 发射 (1,1,1) | 每个 task workgroup 发射 (32,1,1) |
| --------------------------------------------- | ---------------------------- | ----------------------------- |
| Task workgroup 数                              | 100                          | 100                           |
| 每个 task workgroup 的 invocation 数              | 32                           | 32                            |
| Task 总 invocation 数                           | 3200                         | 3200                          |
| Mesh workgroup 数                              | 100                          | 3200                          |
| 每个 mesh workgroup 的 invocation 数              | 64                           | 64                            |
| Mesh 总 invocation 数                           | 6400                         | 204800                        |
| 处理的 meshlet 数（约定一 mesh workgroup 对应一 meshlet） | 100                          | 3200                          |
解读：
- 100 个 task workgroup，每个发射 (1,1,1) → 100 个 mesh workgroup。
- 100 个 task workgroup，每个发射 (32,1,1) → 3200 个 mesh workgroup。

Invocation 有什么用：
| 层级                                | 工作职责示例                         |
| --------------------------------- | ------------------------------ |
| 一个 task workgroup，32 个 invocation | 32 个线程并行检查一批 meshlet，决定哪些需要绘制  |
| 一个 mesh workgroup，64 个 invocation | 64 个线程协作处理一个 meshlet 的顶点、索引和图元 |

## 举例1
画一块 81 个顶点、128 个三角形的平面网格，把整个网格作为一个 meshlet。
```
// Task shader
layout(local_size_x = 32) in;

// Mesh shader
layout(local_size_x = 64) in;
```
Host 只启动一个 task workgroup，每个 task workgroup 发射 (1,1,1)：
```
vkCmdDrawMeshTasksEXT(cmd, 1, 1, 1);
```
整个过程就是：
```
1 个 task workgroup
    └── 32 个 task invocation
    └── 整组发射 (1,1,1)
            ↓
        1 个 mesh workgroup
            └── 64 个 mesh invocation
            └── 协作输出 81 个顶点、128 个三角形
```
- 32 个 task invocation：没有实际收益，因为它们没有做筛选或准备数据，只是发射一个 mesh workgroup。这个 task shader 是为了演示流程而保留的。
- 64 个 mesh invocation：有明确的分工作用，它们并行处理同一个 meshlet 内的 81 个顶点和 128 个三角形。但“有并行分工”不等于“这个微型例子一定更快”，实际性能还取决于硬件和工作负载。

## 举例2
32,000 个 meshlet。为了方便计数，每个 meshlet 都有：  
64 个顶点。  
96 个三角形。  
总共是 204.8 万条 meshlet 顶点记录、307.2 万个三角形。不同 meshlet 边界上的顶点可能重复存储，所以这里不是全模型去重后的顶点数。  
配置如下：  
```
// Task shader：每组处理一批 32 个候选 meshlet。
layout(local_size_x = 32) in;

// Mesh shader：每组处理一个可见 meshlet。
layout(local_size_x = 64) in;
```
Host 启动：
```
vkCmdDrawMeshTasksEXT(cmd, 1000, 1, 1);
```
本例不再坚持“每个 task 固定发射一个 mesh workgroup”，而是每组根据可见性发射 0～32 个。这正是 task shader 动态筛选工作的价值。  
Task 阶段：32 个 invocation 同时检查 32 个 meshlet  
组内每个 invocation 各检查一个：
```
Task workgroup 0
    invocation  0 → 检查 meshlet  0
    invocation  1 → 检查 meshlet  1
    ...
    invocation 31 → 检查 meshlet 31

Task workgroup 1
    invocation  0 → 检查 meshlet 32
    ...
    invocation 31 → 检查 meshlet 63
```
每次检查只读取少量元数据，例如包围球和法线锥：
- 包围球是否完全在视锥外？
- 对于适用的单面几何，法线锥是否能证明所有三角形都背向相机？

被剔除的 meshlet 不再启动 mesh shader，也不必在 mesh 阶段读取其完整几何数据。这比先处理全部顶点，再让后续阶段丢弃三角形更早。

假设第 0 组检查 32 个 meshlet，只有下面 6 个通过：
```
可见 meshlet ID：
[2, 5, 9, 17, 23, 30]
```

32 个 invocation 协作把可见 ID 压紧写入 payload，然后整组发射：
```
EmitMeshTasksEXT(6u, 1u, 1u);
```

子 mesh workgroup 通过组内编号索引 payload，获得各自负责的 meshlet：
```
子 mesh workgroup 0 → meshlet  2
子 mesh workgroup 1 → meshlet  5
子 mesh workgroup 2 → meshlet  9
子 mesh workgroup 3 → meshlet 17
子 mesh workgroup 4 → meshlet 23
子 mesh workgroup 5 → meshlet 30
```
“检查 → 压紧 ID → 发射可见数量 → mesh 阶段读取 payload”就是 task 与 mesh 之间的连接。  

Mesh 阶段：64 个 invocation 协作处理一个 meshlet
| Invocation 编号 | 顶点工作     | 三角形工作     |
| ------------- | -------- | --------- |
| 0～31          | 每个处理一个顶点 | 每个处理两个三角形 |
| 32～63         | 每个处理一个顶点 | 每个处理一个三角形 |
| 合计            | 64 个顶点   | 96 个三角形   |

```
uint tid = gl_LocalInvocationIndex;

// 每个 invocation 处理一个顶点。
ProcessVertex(tid);

// 前 64 个三角形：每个 invocation 处理一个。
ProcessTriangle(tid);

// 剩余 32 个三角形：前 32 个 invocation 各处理一个。
if (tid < 32u)
{
    ProcessTriangle(tid + 64u);
}
```
这里 ProcessVertex() 可以包含顶点解压、位置变换、法线变换等；ProcessTriangle() 读取局部索引并输出图元。Mesh shader 的收益在于组内协作处理一批几何，并允许自定义数据读取和解压流程，而不是一个 invocation 串行处理整个 meshlet。  

假设最终只有 25% 的 meshlet 可见  
32,000 个候选 meshlet 中，假设有 8,000 个通过剔除。各 task workgroup 的发射数量可以不同，只要总和为 8,000。  
与“不做 task 剔除、直接启动全部 mesh workgroup”的方案比较：  
| 项目              | 无 task，直接处理全部 meshlet | 加入 task 剔除 |
| --------------- | --------------------- | ---------- |
| Task workgroup  | 0                     | 1,000      |
| Task invocation | 0                     | 32,000     |
| Mesh workgroup  | 32,000                | 8,000      |
| Mesh invocation | 2,048,000             | 512,000    |
| Mesh 阶段处理的顶点记录  | 2,048,000             | 512,000    |
| Mesh 阶段处理的三角形   | 3,072,000             | 768,000    |

这是假想工作量的对比：增加 32,000 次轻量的 meshlet 元数据检查，换取少启动 24,000 个 mesh workgroup，并跳过这些 meshlet 的完整几何处理。实际收益取决于剔除比例、元数据访问成本，以及每个 meshlet 的处理成本。  
这里 mesh invocation 数减少了 75%，但不能因此断言帧率提高四倍：task 阶段有成本，可见物体的光栅化和 fragment shading 也仍然需要执行。  

两种 invocation 分别解决什么问题？
```
Task 的 32 个 invocation：
    并行检查 32 个候选 meshlet。
    回答“哪些 meshlet 值得继续处理？”

Mesh 的 64 个 invocation：
    并行处理一个已通过筛选的 meshlet 内部数据。
    回答“这个 meshlet 的顶点和三角形如何生成？”
```
因此，task 的优势是提前避免无效几何工作；mesh 的优势是并行、灵活地处理保留下来的几何。若几乎所有 meshlet 都可见、每个 meshlet 又非常便宜，task 的剔除成本就可能得不偿失；它不是无条件的加速阶段。  

# Task shader的具体作用
| 分类       | 更准确的表述                                                                      | 关键区别                                                               |
| -------- | --------------------------------------------------------------------------- | ------------------------------------------------------------------ |
| 增：工作放大   | 一个 Task Shader Workgroup 可以发射多个 Mesh Shader Workgroup，将一个上游任务展开成多个下游几何处理任务。 | 增加 M 只会增加 workgroup 数；还需要为这些 workgroup 分配有效的几何任务，才意味着处理更多 Meshlet。 |
| 减：提前剔除   | 在启动 Mesh Shader 之前进行 Meshlet 级别的可见性测试，只为保留下来的 Meshlet 发射 workgroup。         | 避免后续 Mesh Shader 的顶点读取、变换和图元生成，但仍有 Task Shader 自身的剔除成本。            |
| 改：选择工作内容 | 根据距离、屏幕空间误差等条件选择 LOD，并通过 payload 指定后续应处理的 Meshlet 或几何数据。                    | 通常是选择不同精度的数据，而不是 Task Shader 自己修改顶点、输出模型。                          |

# Mesh shader的缺点
开发更复杂、资源更敏感、性能不保底——Mesh Shader 将更多几何处理的控制权交给开发者，也把数据组织、并行分工和跨 GPU 调优的责任交给了开发者。  
- 开发与资产处理更复杂：采用 Meshlet 方案时，需要生成分块、局部索引及剔除信息，并自行设计顶点读取、线程分工和输出逻辑；不只是把 Vertex Shader 换一个名字。  
- 分块存在取舍：Meshlet 太小，会降低顶点复用率、增加跨块重复顶点处理；太大，又会让剔除粒度变粗，并提高单个 workgroup 的资源需求。
- 输出资源可能限制并发：输出顶点数、图元数、属性大小，以及寄存器和 shared memory 的使用都会影响 occupancy；尤其输出上限设得过大，可能因预留空间过多而降低并发。
- 不保证比传统管线快：几何处理较简单、可剔除工作较少时，收益可能不足以抵消额外成本。如果再加入 Task Shader，还会引入任务调度和 payload 开销——这部分不是 Mesh Shader 必须承担的。
- 跨 GPU 调优困难：不同架构偏好的 workgroup 大小、Meshlet 大小和输出方式不同，一套配置不一定处处高效。

# Mesh Shader对比Tessellation和Nanite
对比tessellation：tess是加了一个步骤，mesh shader是完全取代ras之前的步骤。tess效果不如mesh shader。  
对比nanite：nanite不是独立技术，它中间的一步可以用mesh shader实现。nanite是个大架构。这个引擎技术一般绑定UE5。nanite效果很好，电影级别高模，无lod突变。  

# 各厂商的mesh shader硬件支持  
apple：从a14起，metal原生完整支持  
qualcomm：从a6xxx起支持较广泛  
arm：mali g710后支持较广泛  
img：比较受限。意思是由硬件，但需要适配。img的mesh shader硬件实现跟tbdr高度耦合。  

# Tessellation简介
Hull Shader：处理CP(transform)和Patch，计算TF(跟LOD有关)  
Tessellator: 切拓扑结构，fixed function。架构师上要考虑cache。  
Domain Shader：根据HS和TESS的结果生成tess point.  

整体架构都是2 Pass结构。  
VS HS  
DS（GS）PS  

每条共享边上强制使用同一合法的TF。因为如果不一致，公共边上就会有裂缝。  

## 描述
Tessellation 是 GPU 中把一个比较粗的 mesh patch 动态细分成更多几何体的功能。主要有三个阶段：Hull Shader、Tessellator 和 Domain Shader。  
首先是 Hull Shader，也就是 HS。HS 根据输入的 patch，决定需要细分多少，并输出 Tessellation Factors，也就是 TF。  
TF 本质上控制 Tessellator 怎么细分 patch 的边和内部。TF 越大，细分得越多，最后产生的几何体也越多。  
然后是 Tessellator。它是一个 fixed-function stage。它根据 TF 生成细分后的 topology，以及新顶点在 patch 中的参数化坐标。比如 triangle patch，会生成 barycentric coordinates。  
最后是 Domain Shader。DS 拿到 Tessellator 生成的这些坐标，再结合原始 patch 的数据，计算每一个新顶点真正的 position。比如可以根据原始顶点做 interpolation，也可以通过 displacement map 对顶点进行进一步移动。DS 还可以计算 normal、texture coordinate 等其他属性。  
所以简单来说：HS 决定切多少，TESS 负责切并生成坐标，DS 根据这些坐标计算最终顶点的位置。  

Tessellation is a GPU feature that takes a coarse mesh patch and dynamically subdivides it into more detailed geometry. It has three main stages: Hull Shader, Tessellator, and Domain Shader.   
First, the Hull Shader, or HS, looks at the input patch and determines how much it should be subdivided. It outputs tessellation factors.  
The tessellation factors basically control how many pieces the Tessellator should divide each edge and the inside of the patch into. A higher factor means more subdivisions and more geometry.  
Then, the Tessellator is a fixed-function stage. It takes these factors and generates the topology and parametric coordinates of the new vertices. For example, for a triangle patch, it generates barycentric coordinates for the new points.  
Finally, the Domain Shader takes these coordinates and the original patch data, and calculates the actual position of each generated vertex. For example, it can interpolate the original vertex positions, or use a displacement map to move the vertex. It can also calculate other attributes such as normals and texture coordinates.  
So, in simple terms: HS decides how much to subdivide, TESS generates the subdivision and coordinates, and DS maps those coordinates to the final vertex positions.  


## Control Point（CP）、Tessellated Point、DS output vertex 三者关系
The input to the tessellation stage is a patch, which consists of a number of control points.  
The control points are basically the input geometry that defines the shape of the patch.  
The Hull Shader processes these control points and generates tessellation factors.  
The Tessellator then uses those factors to generate new parametric points inside or on the boundary of the patch.  
These points are not necessarily final vertices yet. They are essentially coordinates describing where we are on the patch.  
The Domain Shader is invoked for each tessellated point. It takes that point's parametric coordinates and the patch's control-point data, and calculates the final vertex position and other attributes.  

So the relationship is:  
Control Points → define the patch → Tessellator generates parametric points → Domain Shader turns each point into a final vertex.  

### Control Point（CP）
假设我们输入一个 triangle patch：CP0、CP1、CP2 = Control Points  
它们是 patch 的输入控制点，不是 Tessellator 新生成的点。  

### Tessellated Point
Tessellator 会根据 TF，生成很多 tessellated points。
这些点首先是参数化坐标（parametric coordinates），不是 DS 自己产生的。

### DS output vertex
```
DS(u, v, w)
```
DS 根据：CP0,CP1,CP2...以及：u, v, w   
计算真正的 vertex position：
```
P = u * CP0 + v * CP1 + w * CP2
```
当然实际 DS 可以做得更复杂，比如：
```
P = interpolated_position + displacement * normal
```
```
        Control Points
        CP0 CP1 CP2
             │
             ▼
        Hull Shader
             │
       Tessellation Factors
             │
             ▼
        Tessellator
             │
    parametric coordinates
       (u,v,w), (u',v',w')...
             │
             ▼
      Domain Shader
             │
             ▼
      Final Vertices
```
Control points define the patch.   
The Tessellator generates parametric coordinates based on the tessellation factors,   
and the Domain Shader converts each coordinate into a final vertex.  

## 无硬件支持的Tess是如何实现的
一些场上的GPU是不是对tess没有fixed function？它具体是怎样的？  

没有fixed-function tessellator的GPU，整个tessellation过程通过可编程shader完全软件模拟实现。  

具体实现方式  
将OpenGL ES的tessellation control shader (TCS)和tessellation evaluation shader (TES)作为通用shader管道执行，没有专用硬件生成domain点或索引。  
TCS负责计算tess levels和patch数据，TES则需手动实现细分逻辑（如基于levels的for循环生成顶点、计算barycentric坐标并插值），模拟桌面GPU的fixed tessellator行为。  
这种设计节省芯片面积，但增加shader复杂度和性能开销，高度依赖shader优化和底层硬件（如shader核心的warp执行）。  

架构影响  
利用IDVS（延迟顶点着色）和tiling架构，仅为可见patch执行TES，减少无效几何生成。  
或进一步线性化执行，但仍无fixed单元，用Offline Compiler优化TCS/TES二进制以最大化寄存器利用和线程并发。  

开发者需小心避免过度tessellation，以防带宽瓶颈；ARM推荐LOD-based固定levels而非动态计算。  

# 相关知识
## 重心坐标
Centroid Coordinate  
Barycentric Coordinate  
都叫重心坐标，但第一个是三角形三条中线交点，第二个是一种坐标表示方法：三角形上任何点，都可以用三个权重表示，对应三角形的顶点。  

在四边形上的一点P，用UV可以清楚知道四个点和P的对应关系，容易插值。  

而在三角形上的一点P，除非是特殊的三角形，不然用UV很难知道三个点和P的对应关系，不好插值，所以才需要重心坐标。  

如何计算三角形ABC上一点P的重心坐标  
```
P=alpha*A+beta*B+gamma*C  
alpha+beta+gamma=1  
```
三个未知数。  
如果是2D space，刚好有三个方程。  
如果是3D space，看起来好像有四个方程，但实际四个方程不是独立的。也就是其实少了一个自由度。原因是P必须是在ABC定义的平面上，否则方程组无解，因此三个分量方程并不是三个独立方程。  
换句话说，如果P位于三角形平面内，方程组有且仅有唯一解   

重心坐标的作用  
- 如果重心坐标都不是负数，P在三角形内
- 颜色差值
```
ColorP=alpha*ColorA+beta*ColorB+gamma*ColorC
```
- 纹理坐标差值

## 插值
Bezier插值跟普通差值（线性、拉格朗日等)的区别  
- 线性插值：简单，不光滑
- 拉格朗日插值：高阶多项式
- 三次样条插值：段段多项式连接，曲线光滑  
以上插值都是必过所有点，Bezier插值只通过首尾点，连续而且光滑性很高。而且通过调整cp，可控性很强。  
- B样条曲线：比Bezier曲线更复杂，甚至不需要经过任何控制点。

选Bezier还是普通插值：Bezier插值更易控制，更加光滑。Tess的特点是需要保持起始点，其余点决定形状，也适合bezier。普通插值要么不光滑，要么容易产生多项式震荡。  

选Bezier还是B样条：在OpenGL里普遍使用Bezier，适合实时渲染。复杂CAD建模可以使用B样条。  

Tess对于插值的实现是在ds里面让用户实现的。没有bezier插值的硬件实现。  

Bezier曲线的次数：如果有n+1个控制点，Bezier曲线就是n次的。  

如果不想用Bezier插值，只需要修改DS中用于插值的代码就可以了。  






