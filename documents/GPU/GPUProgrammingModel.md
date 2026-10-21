# CoopMat and CoopVec
在 GPU 领域，coopmat 是 Cooperative Matrix（协作矩阵），coopvec 是 Cooperative Vector（协作向量）。它们是让 shader 使用矩阵乘加加速硬件的两种编程抽象，而不是两个特定的硬件单元名称。最关键的区别是协作粒度：coopmat 让一组线程共同处理一个矩阵；coopvec 让每个线程逻辑上独立处理自己的向量，底层再把多个线程的计算协作起来。  

## CoopMat
它的核心操作通常是矩阵乘加：
```
D = AB + C
```
这里的 A、B、C、D 不是每个线程各自持有一份完整矩阵，而是在一个协作范围内——通常是 subgroup——分散存储、共同计算。具体哪些元素放在哪个线程的寄存器中，由实现决定，程序不能把它当成固定的线程到元素映射。

subgroup: 在 NVIDIA GPU 上，Vulkan 的 subgroup 通常对应一个 warp。但严格来说，subgroup 是跨厂商的抽象，warp 是 NVIDIA 的具体术语，不能在所有 GPU 上直接画等号。  
AMD GPU 上，subgroup 通常对应一个 wave/wavefront。  

```
一个 subgroup
    ├── thread 0：持有部分 matrix fragment
    ├── thread 1：持有部分 matrix fragment
    ├── ...
    └── thread N：持有部分 matrix fragment

整个 subgroup：
    协作加载 A、B、C
    协作计算 D = A × B + C
    协作存储 D
```

什么这样设计？  
矩阵乘法硬件适合处理一个矩阵块，而传统 shader 的编程单位是线程。Coopmat 在两者之间建立桥梁：用一个跨线程的矩阵类型，表达“这组线程共同完成一次矩阵块运算”。  

因此，它很自然地适合组织成矩阵 tile 的计算，例如 GEMM 和批量神经网络计算。程序员围绕 tile 来安排数据加载、共享内存复用和计算，而不是围绕“每个线程独立推理一个样本”来组织程序。  

## CoopVec
Coopvec 的核心使用方式是，每个 shader invocation 拥有自己的输入向量，执行类似：  
```
yi = Wxi +b
```
其中 i 表示不同 invocation。每个 invocation 可以独立完成一份小型神经网络推理，而不必在源码层面组织成一个 subgroup 共享的矩阵。
```
thread 0：x₀ → 神经网络 → y₀
thread 1：x₁ → 神经网络 → y₁
thread 2：x₂ → 神经网络 → y₂
...
```
它与普通向量类型的区别是：cooperative vector 可以有较大的、非传统 vec2/vec3/vec4 形式的长度，操作集合也更受限，重点是支持硬件加速的矩阵—向量运算和小型神经网络计算。  

| 维度         | Coopmat                 | Coopvec                          |
| ---------- | ----------------------- | -------------------------------- |
| 程序员看到的数据归属 | 一组线程共同拥有一个矩阵            | 每个线程逻辑上拥有自己的向量                   |
| 核心运算       | 矩阵—矩阵乘加                 | 矩阵—向量乘法，可带 bias                  |
| 显式协作       | 应用围绕 subgroup 等协作范围组织计算 | 应用保持逐线程编程，底层隐藏协作                 |
| 控制流约束      | 集体操作需要满足协作组的一致参与要求      | 不要求 subgroup 满员或统一控制流，但这些条件有利于性能 |
| 典型组织方式     | 矩阵 tile、批量计算            | 每个像素、采样或着色 invocation 独立运行小网络    |

## 放到 Vulkan 和渲染中理解
VK_KHR_cooperative_matrix：提供 cooperative matrix 类型，以及支持的矩阵形状和类型查询。  
VK_NV_cooperative_matrix2：在 cooperative matrix 基础上增加 workgroup scope、更灵活的矩阵尺寸、归约、逐元素操作和 tensor addressing 等能力。  
VK_NV_cooperative_vector：提供逻辑上属于单个 invocation 的 cooperative vector，以及矩阵布局转换等支持。  

## 各大硬件厂商对coopmat/coopvec的支持
截至目前（2026 年 10 月），可以概括为：coopmat 比 coopvec 更普及、更跨厂商  
| 厂商     | Vulkan coopmat                                                  | Vulkan coopvec              | DirectX coopvec                                           |
| ------ | --------------------------------------------------------------- | --------------------------- | --------------------------------------------------------- |
| NVIDIA | 支持；有早期 NV 扩展、KHR 基础扩展及 VK_NV_cooperative_matrix2 等增强            | 支持 VK_NV_cooperative_vector | 官方宣布所有 GeForce RTX GPU 可加速，具体取决于驱动与 SDK                   |
| AMD    | RDNA 3 / RDNA 4 在 Linux Mesa RADV 上支持 VK_KHR_cooperative_matrix | 本次资料未确认 AMD 驱动提供支持          | 已支持；AMD 2026 年文档明确要求开发者驱动，支持 Radeon RX 9000 系列            |
| Intel  | 已有支持；公开资料确认 Xe2 的 Vulkan 驱动支持，Windows Arc 驱动也已有实际使用记录           | 本次资料未确认 Intel 驱动提供支持        | 已提供开发者预览支持，微软明确列出 Arc B 系列及 Core Ultra Series 2 的 Arc GPU |

移动端也开始支持 coopmat。例如 Imagination 的 DDK 25.2 已宣布实现 VK_KHR_cooperative_matrix。  
对于 Arm、Qualcomm，本次查到的公开资料不足以给出可靠的“GPU 型号 × 驱动 × API”支持表。  




