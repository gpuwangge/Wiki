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

# BRDF（Bidirectional Reflectance Distribution Function，双向反射分布函数）
1. BRDF 方程
```c++
f_r(omega_i,omega_o)=
    dL_o(omega_o) / dE_i(omega_i)
```
其中：
- f_r：BRDF，描述表面的反射特性。
- omega_i：入射光方向（Incoming Light Direction）。
- omega_o：出射光 / 观察方向（Outgoing / View Direction）。
- dL_o：出射辐射亮度（Differential Radiance）。
- dE_i：入射辐照度（Differential Irradiance）。
简单来说：BRDF 描述光从某个方向照射到表面后，有多少光会被反射到另一个方向。  
BRDF 是渲染方程（Rendering Equation）中的核心组成部分  

2. BRDF 在渲染中的作用
不同材质具有不同的 BRDF，因此会产生不同的视觉效果：  
- Diffuse（漫反射）：光线向多个方向均匀散射，例如粗糙的粉笔表面。
- Specular（镜面反射）：光线集中反射到特定方向，例如抛光金属。
- Roughness（粗糙度）：影响镜面反射的高光范围；粗糙度越高，高光通常越宽、越模糊。

<p float="left">
  <img src="https://github.com/gpuwangge/Wiki/blob/main/images/brdf.png" alt="alt text">  
</p> 

3. PBR参数和BRDF的关系
Material 参数是描述材质的输入；BRDF LUT 则存储预计算的反射积分结果。  
常见的 PBR 材质参数如下：  

| 参数 | 含义 | 在 BRDF 中的作用 |
|---|---|---|
| Base Color | 基础颜色 | 决定漫反射颜色及金属的反射颜色 |
| Metallic | 金属度 | 决定使用金属还是非金属的反射模型 |
| Roughness | 粗糙度 | 控制微表面法线分布，影响高光形状 |
| Normal | 法线 | 决定表面朝向，影响光照计算 |
| AO | 环境光遮蔽 | 通常用于近似局部遮蔽，并非 BRDF 本身的参数 |

BRDF模型描述了Ray Tracing中直接光照的部分，但无法描述间接光照。间接光照需要使用Path Tracing算法。Path Tracing算法基于Monte Carlo 积分来模拟光线的路径。  

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

# Ray Query
Vulkan 的 Ray Query 可以理解为：在普通 compute / fragment shader 中，直接查询光线与场景的交点，不需要单独创建 ray tracing pipeline、SBT，也不需要 raygen / closest-hit / miss shader。  

简略实现步骤
1. 开启扩展和 feature
```
// Device extensions
VK_KHR_ACCELERATION_STRUCTURE_EXTENSION_NAME
VK_KHR_RAY_QUERY_EXTENSION_NAME
VK_KHR_DEFERRED_HOST_OPERATIONS_EXTENSION_NAME

// 对应 feature，通过 device 创建时的 pNext 链启用
bufferDeviceAddress = VK_TRUE;
accelerationStructure = VK_TRUE;
rayQuery = VK_TRUE;
```
2. 构建 BLAS：存放 mesh 几何
```
准备 vertex / index buffer，并获取 device address。
用 VkAccelerationStructureGeometryKHR 描述三角形数据。
调用 vkGetAccelerationStructureBuildSizesKHR() 查询所需空间。
分配 AS 存储 buffer 和 scratch buffer。
调用 vkCreateAccelerationStructureKHR() 创建对象。
调用 vkCmdBuildAccelerationStructuresKHR() 执行构建
```
3. 构建 TLAS：存放场景实例
```
instance.transform = ...;                    // 实例变换
instance.instanceCustomIndex = meshId;       // shader 用于查找自己的数据
instance.mask = 0xFF;
instance.accelerationStructureReference =
    blasDeviceAddress;                      // AS 地址，不是底层 buffer 地址
```
将 instance 数组作为输入构建 TLAS。整体关系是：
```
TLAS
 ├─ Instance A → BLAS(mesh A)
 ├─ Instance B → BLAS(mesh B)
 └─ Instance C → BLAS(mesh A)  // 可复用同一个 BLAS
```
BLAS 构建完成到 TLAS 构建之间，以及 TLAS 构建完成到 shader 查询之间，都要建立正确的同步依赖  
4. 将 TLAS 绑定给 shader
在 descriptor set 中添加：VK_DESCRIPTOR_TYPE_ACCELERATION_STRUCTURE_KHR  
通过 VkWriteDescriptorSetAccelerationStructureKHR 写入 TLAS。  
GLSL 对应声明：  
```
#extension GL_EXT_ray_query : require
layout(set = 0, binding = 0)
uniform accelerationStructureEXT sceneTLAS;
```
接下来仍然使用普通 graphics / compute pipeline。  
5. 在 shader 中发射并查询光线
下面是最简单的不透明三角形阴影测试
```c++
bool isOccluded(vec3 origin, vec3 direction, float tMin, float tMax){
    rayQueryEXT query;

    rayQueryInitializeEXT(
        query,
        sceneTLAS,
        gl_RayFlagsOpaqueEXT |
        gl_RayFlagsTerminateOnFirstHitEXT,
        0xFF,              // instance mask
        origin,
        tMin,
        direction,         // 调用方传入归一化方向
        tMax
    );

    //rayQueryProceedEXT() 做什么？
    //每次调用，它会继续遍历加速结构，直到出现以下一种情况：
    //返回 true：找到了需要 shader 处理的候选交点，暂停遍历，进入循环体。例如非 opaque 三角形，需要你做 alpha test，再决定是否确认命中。
    //返回 false：查询已经结束，退出循环。此时再读取最终的 committed intersection。
    //对于其中的不透明三角形，查询会自动确认命中；再加上 TerminateOnFirstHit，确认第一个命中后就结束，不需要你在循环体中处理。
    //保留 while 是更通用的写法，方便以后在循环体里增加 alpha-test 或程序化几何的候选处理；
    while (rayQueryProceedEXT(query)) {
        // 强制视为 opaque 的三角形，无需手动确认 candidate。
    }

    //gl_RayQueryCommittedIntersectionNoneEXT 表示“没有已确认的交点”。所以 != None 就表示存在已确认命中。
    return rayQueryGetIntersectionTypeEXT(query, true)
        != gl_RayQueryCommittedIntersectionNoneEXT;
}
```
在 fragment shader 的光照计算中调用：
```c++
vec3 toLight = lightPosition - worldPosition;
float distanceToLight = length(toLight);

//从“当前着色点指向光源”的 阴影光线（shadow ray）。
//它用来检查：这个点到光源之间有没有东西挡住。
//命中不透明遮挡物就意味着这条直接光照路径不通。
bool shadowed = isOccluded(
    worldPosition,
    toLight / distanceToLight,
    0.001,                       // 示例值，需按场景尺度调整
    distanceToLight
);

if (!shadowed) {
    // 累加该光源的直接光照
}
```
这里命中任意遮挡物就可以结束；如果要查找最近交点用于反射或完整渲染，则去掉 TerminateOnFirstHit，完成遍历后再读取 committed hit 的距离、primitive index、实例索引和重心坐标。  

不透明几何可以自动确认命中；alpha-test 几何需要在遍历循环中检查 candidate、采样 alpha，再按需调用 rayQueryConfirmIntersectionEXT()，不能像上例一样强制全部 opaque。  

Ray Query 只提供相交查询结果，不会自动执行材质着色。获取交点后，需要自己查 vertex / index / material 数据并计算光照。  

如果要进行二次光线的计算，要在第二段代码外层加一个循环：
```c++
for (int bounce = 0; bounce <= MAX_SECONDARY_BOUNCES; ++bounce)
{
    ...
}
```







