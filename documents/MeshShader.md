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






