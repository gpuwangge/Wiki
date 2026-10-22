# Mesh Shader介绍
Mesh Shader是为了解决ras之前geometry管线的缺点：很多vertex被处理了之后却因为种种原因被丢弃。  

Task Shader: 扫描屏幕和相机，决定哪些物体该算，哪些该直接跳过。  
Mesh Shader：只处理挑中的物体，可以细分物体。  
Fragment Shader：接受精修过的三角形。  

输入mesh shader的是payload，而不是传统数据格式。  

mesh shader并不是全场景通用，而是适合特定场景。比如适合开放世界海量实体渲染。不适合简单场景。  

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






