# Motivation
要想获得真实感的渲染，要有光。  
光的产生过程：光源射出光，在物体表面反射，再射入摄像机。  
光的三个信息：光从哪个方向射过来；物体表面如何反射光；光有多少摄入摄像机。  
K_d\*入射光强度\*cos(theta)=反射光强度  
实际颜色=反射光强度\*色值  

## 关于K参数的讨论
以上K_d参数是使用在经典材质模型公式，真实的渲染方程则使用BRDF参数, BRDF可以看作描述各种材质的量化统一  
Physical-based Models, PBR模型: 使用BRDF的模型，其BRDF参数采用实验测算或理论推算，非常写实  

## 关于入射光的讨论
反射角度、材质的特性(颜色、粗糙度、反射后衰减、法线、透明度等)都会影响最终有多少光摄入摄像机。  
为了节省计算时间，实时的光照、阴影可以用预先渲染好的贴图来实现，而不用实时演算。  
以上叫做光栅化算法。光栅化算法与真实世界的原理是不一样的。比如光照不均匀，光线缺乏变化。  
真实光照原理：光源发出光，光经过反射进入眼睛。没进入眼睛的光仍然在空间中反复反射(直到光子被完全吸收)，照亮其他物体，最终也有一部分进入眼睛。这样光看起来更柔和(也更亮)。  
这个过程叫做全局光照(Global illumination, GI)  

# 解决GI的问题：RT Core(光线追踪核心)
GI计算量过于庞大，在NVidia RTX 20系显卡出现以前都没有实际应用。  
(RTX=Ray Tracing)  
RTX 20系显卡第一次(2018/9/20~2019/7/9)加入了光线追踪核心。  
所谓光线追踪，就是追踪3D世界中每一条光线的路径。实际应用中为了效率，反其道，从摄像机射出“射线”，看看经过数次反射后"射线"能不能进入光源。能进入光源的“射线”才是有用的光线。  
(实际上是从每个像素发出数根射线。射线的数量越多效果越逼真。要达到真实世界差不多的效果，每个像素要发射几百根)  
这样的好处是可以排除掉最终没有进入摄像机的光线。  
在射线数次反弹过程中会碰到各种物体，每一个被碰到的物体都会被照亮。其亮度会根据光源计算。  
摄像机会射出很多条射线，需要把所有射线的结果叠加计算，然后再跟材质颜色结合起来。  
RTX 20系显卡之前的传统显卡是针对光栅化设计的，无法支持如此运算量巨大的算法。  
光栅化计算的特点是由矩阵的乘法/加法组成的。传统显卡的计算单元由乘法/加法器和一些必要的逻辑线路组成，可以实现vertex/fragment shader的并行计算。  
但是光线追踪的路径非常复杂，并且存在不同光线的叠加，传统计算单元计算的效率并不高。  
NVidia曾经尝试过用软件设计算法(2009)，在传统计算单元上实现光线追踪，但效果并不理想。  

# RT Core(光线追踪核心)原理
设计思路是把光线追踪算法拆分成可以并行计算的部分，分别使用专门的硬件来计算。  
增加了五种新的Shader，用来处理光线产生、求交、碰撞、关闭碰撞和丢失的情况。  
RT Core里用硬件实现快速求交运算(包围盒)  
这样就可以实现可以比拟真实世界的全局光照效果。  
在实际应用中，会使用一些加速技术，比如把光栅化和光追结合起来的辅助光追。  

# Tensor Core原理
NVidia在RTX系列里加入光追核心的同时，也加入了被称为Tensor Core的AI计算核心。  
AI运算也是加法(结果相加)和乘法运算(计算权重)，因此天生适合AI计算。  
CUDA的原理就是用GPU的计算单元来进行AI计算。  
AI计算的指标：TOPS(Tera Operations Per Second)，即每秒钟的万亿操作次数。  
(Ex: RTX 40系显卡提供242~1177 AI TOPS的算力)
决定TOPS的三大要素：时钟频率，计算核心数量(每个计算核心都应该可以同时进行运算)，芯片架构
一次加乘的运算流程：
1、取出数据放到乘法器；  
2、把计算结果放回寄存器；  
3、从寄存器中取出数据放入加法器；  
4、把计算结果重新放入寄存器；  
在传统显卡的计算核心中，一次原子计算就是三个数字(A,B,C)的一次乘加(A*B+C)。  
而AI计算的核心是矩阵乘加。因此Tensor Core可以针对AI计算的特点进行进一步的优化，以减少数据存取过程。  
Tensor Core的底层架构未知。  
已知的是Google的TPU硬件架构：以减少矩阵计算的数据存取为目的。同时支持混合精度运算(Ex: 输入输出采用16位精度，中间计算用32位精度，这样可以提高数据存取速度)。  
同时，软件架构(与Tensor Core配套的软件架构叫Tensor RT)也很重要。  

# AI对Graphics应用(游戏)的帮助
AI可以极大提升游戏性能和画质。  
DLSS: 提高分辨率。(AI超分辨率)目前已经到第三代。  

# GI原理
对渲染方程的求解：理论上光反弹次数是无限次，因此渲染方程里面有积分。真实应用种会限制反弹次数，从而用有限的离散方程去逼近渲染方程的解。  
（每次新的反弹对最终结果的影响是递减的）  
对渲染方程的求解就是实现全局光照的过程。  

## Lumen
Lumen是虚幻引擎中实现实时全局光照的的方法。  
光线追踪(1986)虽然是实现全局光照的方法，但一直以来很难做到实时。  

### 距离场
判断每条管线被哪个三角形反弹一般流程是：  
1、求光线和三角形的交点；  
2、判断交点是否在三角形内(反复测算所有的三角形直到找到)；  
要想实时GI，需要很多次反弹(虽然效果最大的是第一次反弹)，但传统方法连每个像素发射一根射线都做不到。  
Lumen通过引入距离场(Distance Field)可以实现射线的无限次反弹。原理是计算第一次反弹(直接光, direct light)，然后使用Radiosity(1984)来生成其余的反弹(间接光, indirect lights)。  
距离场里的每一个点存一个常量，代表最近的三角形和这个点的距离。  
设定一个阈值比如0.01，如果距离场的值小于阈值，则表示这个点和三角形相交。  
在计算射线和三角形交点的时候，可以去查距离场表，而不需要真的计算交点。  
当然，当物体移动之后，其附近的距离场需要重新计算。但这也比求光线和三角形交点快多了。  

### 表面缓存和辐射度算法(Radiosity)
Lumen专门分配的缓存，用来存直接光。  
然后使用Radiosity来算出间接光。  
Radiosity原理：把场景分成很多面元，每个面元会向外辐射能量，也能接受其他面元的能量。  
因此，每个面元的能量就是把其余所有面元发来的能量加起来，再加上一个自发光项(非光源这项就是零)。  
面元的划分决定了模型的精细程度，面元划分越细小，视觉效果越好。  
Radiosity模型的求解可以转化成矩阵方程组。  
快速求解这个方程组的关键是复用：Lumen把上一帧接受的光照，作为这一帧向外辐射的能量。  
每一帧的光照，存进表面缓存作为下一帧使用。下一帧累加完毕后，更新表面缓存。  
另外，根据光线距离，Lumen采取了不同的计算方案。  

# Reference
https://www.bilibili.com/video/BV1yS421X7z6  
https://www.bilibili.com/video/BV1Hk4y1q7Rz



