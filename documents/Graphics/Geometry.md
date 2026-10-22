# 判断 triangle 的 frontface/backface：几何上判断是否朝向相机  
设三角形三个顶点为p0​,p1​,p2​，相机位置为C
```
N=(p1−p0)×(p2−p0)
V=C−p0
```
再计算：
```
d=N⋅V
```

# G-buffer和Defered Rendering
Deferred Rendering 是一种渲染方法，G-Buffer 是这种方法用来暂存几何信息的一组缓冲区。  
传统的 Forward Rendering（前向渲染）通常在绘制物体时，就计算该物体受到的光照；Deferred Rendering 则把几何信息和光照计算拆成两个阶段。  
```
Deferred Rendering 流程
    Pass 1：Geometry Pass
    绘制场景几何体，记录每个像素的表面信息

    G-Buffer
    Position · Normal · Albedo · Material Properties

    Pass 2：Lighting Pass
    读取 G-Buffer，结合光源和 BRDF 计算光照

    Final Image
    最终像素颜色
```
为什么要这样做？  
假设一个场景有很多物体和光源。前向渲染可能需要在物体绘制时反复计算光照；延迟渲染先记录可见表面的信息，再对屏幕上的像素进行光照计算，避免对同一个可见表面重复执行大量光照计算。  
它特别适合具有大量动态光源的场景，但并不是所有场景都更快。  

## 什么是 G-Buffer？
G-Buffer 全称是 Geometry Buffer（几何缓冲区），通常由多个 Render Target / Texture 组成，用于保存 Geometry Pass 生成的表面数据。  
例如：
- Position（位置）：记录表面位置，通常使用世界空间或视图空间坐标。也可以通过深度和相机参数重建位置。
- Normal（法线）：记录表面法线，用于计算光线与表面的夹角，以及 BRDF 的反射方向关系。
- Albedo（颜色）：记录表面基础颜色，作为漫反射及金属反射颜色计算的输入。
- Material Buffer（金属度）：保存 Metallic、Roughness 等材质参数，供后续 BRDF 计算使用

这些只是常见的逻辑分类。实际引擎可以将多个参数打包到同一张纹理中，也可以不存储 Position，而是通过 Depth Buffer 重建位置，以节省显存带宽。  

## G-buffer/Defered Rendering跟TBDR的关系
TBDR是一种架构，对应的是TBDR vs IMR  
G-buffer/Defered Rendering是一种渲染方法，对应的是 Deferred vs Forward  

两者不冲突，设计上可以共存。  

Mobile GPU 可以采用 TBDR 架构，同时使用 Forward Rendering。
Desktop GPU 可以采用 IMR 架构，同时使用 Deferred Rendering。

但是，为什么不总是这样做？  
主要有三个原因：  
- 片上存储容量有限： G-Buffer 如果包含多个高精度 Render Target，会占用大量 Tile Memory。  
- 带宽与性能权衡： Deferred Shading 虽然可以降低重复光照计算，但 G-Buffer 的生成、存储和读取仍有成本。  
- 渲染需求不同： 透明物体、MSAA、复杂材质等场景可能更适合 Forward 或混合渲染方案。  

因此，实际引擎经常采用 Hybrid Rendering：例如不透明物体用 Deferred Shading，透明物体用 Forward Rendering。  


