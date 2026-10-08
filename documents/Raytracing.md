# 光线追踪（Ray Tracing）

## 核心思想
从摄像机（眼睛）向屏幕每个像素发射一条光线，计算光线与场景中物体的交点，根据交点处的材质、法线以及光源信息计算最终颜色；遇到反射/折射材质时，继续递归发射次级光线。
```
Ray(t) = O + t*D, t > 0
```

## 三种主流实现方式对比

### 1. Fragment Shader（像素着色器）

* **核心机制**：把全屏渲染当作一个绘制全屏矩形（Full-screen Quad）的过程，为每个像素（Fragment）运行一次 Shader。
* **实现方式**：
  1. 根据当前像素坐标 $(x, y)$ 生成摄像机射线（Ray）。
  2. 在 Shader 内用**数学解析式**（如球体 $x^2 + y^2 + z^2 = r^2$）或 **SDF（Signed Distance Field，有向距离场/光步进 Raymarching）** 手动计算光线与几何体的相交。
  3. 若要处理复杂网格，需将 BVH 结构和顶点数据打平存入 Texture 或 SSBO（Shader Storage Buffer Object）手动遍历。
* **特点**：纯软件/算法模拟，无需硬件光追支持。灵活性高，适合 ShaderToy 式的数学曲面或简单 SDF 渲染，但处理海量多边形网格时效率极低。

### 2. Compute Shader（计算着色器）

* **核心机制**：脱离管线渲染流程，利用 GPU 的通用并行计算能力（GPGPU），按 2D/3D 线程组（Thread Group）直接并行计算像素。
* **实现方式**：
  1. 分派线程组（如 `Dispatch(width / 8, height / 8, 1)`），每个线程映射一个像素。
  2. 在 Compute Shader 中直接写出结果到 **Storage Texture / RWTexture2D**（如 `imageStore` / `RWTexture2D::operator[]`）。
  3. 解耦了传统渲染管线，更容易在 Shader 内实现**栈结构（Stackless BVH 遍历）**、路径追踪（Path Tracing）的多次反弹循环以及累加平滑（Accumulation Buffer）。
* **特点**：结构比 Fragment Shader 更自由、更容易做蒙特卡洛积分和路径追踪，但光线与复杂网格相交依然依靠软件层面（Shader 代码）遍历 BVH。

### 3. HW 光追（硬件加速：DXR / Vulkan Ray Tracing）

* **核心机制**：将 BVH 构建与“射线-三角面”相交计算彻底交由 GPU 专用的硬件单元（如 RTX 中的 RT Core）处理。
* **实现方式**：
  1. **构建加速结构**：CPU/API 端创建底层的 **BLAS**（Bottom-Level Acceleration Structure，包含 Mesh 顶点）和顶层的 **TLAS**（Top-Level Acceleration Structure，包含 Instance 变换）。
  2. **管线拆分**：引入全新的光追着色器阶段：
     * **Ray Generation Shader**：发射主射线。
     * **Closest Hit Shader**：最近交点处执行材质与光照计算。
     * **Miss Shader**：光线未命中任何物体时（如渲染天空盒）。
     * **Any Hit / Intersection Shader**（可选）：处理透明贴图裁剪或自定义几何体。
  3. 在 RayGen 中直接调用硬件指令 `TraceRay()` / `traceRayEXT()`，硬件自动高效完成 TLAS/BLAS 遍历与求交，并回调对应的 Hit/Miss 着色器。
* **特点**：性能极高，能轻松处理百万级多边形场景的实时光追，大幅简化了开发者手写 BVH 遍历的代码复杂度。

## 汇总总结

| 特性 | Fragment Shader | Compute Shader | HW Acceleration |
| :--- | :--- | :--- | :--- |
| **硬件要求** | 普通 GPU | 支持 CS 的普通 GPU | 支持 DXR/VK RT 的硬件 |
| **几何相交** | 手写 (SDF/解析解) | 手写 (Software BVH) | 硬件加速 (RT Core) |
| **管线集成** | 传统 Rasterize | 通用 Compute | 专用 Ray Tracing Pipeline |
| **适用场景** | ShaderToy / 简单场景 | 自研 Path Tracer / 试验项目 | 工业级 / 实时游戏光追 |



# Raytracing Context 包含什么

| 类别 | RCE Context 内容 |
|---|---|
| 射线参数 | `Origin`、`Direction`、`tMin/tMax`、`Ray Flags`、`Hit Mask` |
| 追踪状态机 | 当前命中/未命中阶段、`Hit/Miss Shader Index`、`Payload`、缓冲区引用、`Ray Query` 状态（`Proceed/Committed`） |
| 加速结构引用 | `TLAS/BLAS` 引用 ID、`SBT`（Shader Binding Table）偏移、`Instance/Geometry ID` |
| 遍历栈与常数 | 递归深度、`BVH` 遍历栈指针、`Push Constants` / `Tunable Parameters` |
| 线程组管理 | 线程 ID 分发状态、屏蔽掩码（`Enable Mask`）、预取队列指针 |

# 为什么Path Tracing用monte carlo积分而不是普通积分
因为光追的rendering equation维度大，不连续  
普通积分追求精确，无法解  
MC积分不追求精确，对维度连续性无要求，而且可以并行  
```
I=1/N sum_1…N f(xi)  
```
其中xi随机分布，每个点独立。xi对应一条”光路径”  
N就是SPP 或 (SPP x 像素的数量)，看是local还是global  
收敛分析：对一个pixel，误差随sqrt(SPP)缩小  


