<p float="left">
  <img src="https://github.com/gpuwangge/Wiki/blob/main/images/GraphicsPipelineArchitecture2.png" alt="alt text">  
</p> 

# Graphics Pipeline 模块介绍

本文介绍 Graphics Pipeline 中各个模块的主要功能，以及它们在 GPU 渲染过程中的作用。

## 1. Geometry Data

**几何数据（顶点与索引）**

提供描述 3D 几何体所需的顶点和索引数据。

* 顶点位置、法线、纹理坐标等属性。
* 索引定义顶点之间的连接关系。
* 为后续顶点处理和图元组装提供输入。

## 2. Vertex Shader

**顶点着色器**

对顶点进行可编程处理，并计算顶点相关属性。

* 将顶点变换到裁剪空间。
* 计算并输出顶点属性。
* 执行蒙皮、变换等顶点级操作。

## 3. Primitive Assembly

**图元组装**

将处理后的顶点组装成可渲染的图元。

* 构建三角形、线段或点。
* 处理图元连接关系。
* 为后续光栅化阶段准备输入。

## 4. Hierarchy Tiling

**层次化分块（TBDR 相关）**

将图元划分到屏幕空间的不同 Tile 中，并执行相关的分块和剔除操作。

* 将图元分配到对应的 Tile（Binning）。
* 剔除不需要处理的图元或区域。
* 提高数据局部性，减少不必要的处理。

## 5. HSR

**隐藏表面消除（Hidden Surface Removal）**

利用可见性和深度信息，尽早移除不可见的表面或片段，以减少 Overdraw。

* 减少被其他表面遮挡的片段。
* 利用深度信息进行可见性判断。
* 常见于 TBDR 相关的渲染优化流程。

> HSR 的具体实现方式和执行位置取决于 GPU 架构。

## 6. Rasterization

**光栅化**

将几何图元转换为片段（Fragments）。

* 确定图元覆盖的像素区域。
* 对顶点属性进行插值。
* 生成供后续片段处理的候选片段。

## 7. LRZ

**低分辨率深度测试（Low Resolution Z，Qualcomm）**

利用低分辨率或粗粒度深度信息，提前剔除不可见的几何区域或片段。

* 执行粗粒度深度测试。
* 尽早拒绝被遮挡的区域。
* 减少后续渲染阶段的工作量。

## 8. Early Z

**早期深度测试**

在片段着色器执行之前，尽可能完成深度测试。

* 尽早剔除被遮挡的片段。
* 减少片段着色器的执行次数。
* 提高渲染效率。

## 9. FPK

**片段消除（Mali，FIFO-Based）**

利用 FIFO 机制追踪片段，并尝试消除不必要的 Overdraw。

* 追踪片段的处理顺序和可见性。
* 消除已确定不需要继续处理的片段。
* 减少不必要的片段着色工作。

> FPK 的具体机制需要结合 Mali GPU 的硬件实现进行分析。不同 GPU 架构的 Overdraw 消除策略可能存在差异。

## 10. Fragment Shader

**片段着色器**

对生成的片段执行可编程着色计算。

* 计算光照和材质效果。
* 执行纹理采样。
* 生成片段颜色及其他输出数据。

## 11. Late Z

**后期深度与模板测试**

在片段着色器之后（必要时）执行深度和模板测试。

* 检查深度和模板测试条件。
* 丢弃未通过测试的片段。
* 处理无法在 Early Z 阶段完成测试的情况。

## 12. Color Blending & Output

**颜色混合与输出**

将片段输出与现有帧缓冲数据进行混合，并写入渲染目标。

* 执行颜色混合操作。
* 处理渲染目标相关操作。
* 将最终颜色及深度/模板结果写入对应存储区域。

---

## Graphics Pipeline Overview

| 模块                      | 简要说明                         |
| ----------------------- | ---------------------------- |
| Geometry Data           | 提供顶点和索引数据，为几何处理提供输入。         |
| Vertex Shader           | 变换顶点并计算顶点属性。                 |
| Primitive Assembly      | 将顶点组装成三角形、线段或点。              |
| Hierarchy Tiling        | 将图元分配到 Tile，并执行层次化剔除。        |
| HSR                     | 消除不可见表面，减少 Overdraw。         |
| Rasterization           | 将图元转换为片段。                    |
| LRZ                     | 执行粗粒度深度剔除。                   |
| Early Z                 | 在片段着色前剔除不可见片段。               |
| FPK                     | 基于 FIFO 的片段消除机制，减少 Overdraw。 |
| Fragment Shader         | 计算光照、材质和片段输出。                |
| Late Z                  | 执行最终深度和模板测试。                 |
| Color Blending & Output | 执行颜色混合并写入渲染目标。               |

### 核心优化方向

> **通过可见性剔除、Overdraw Reduction 和并行着色执行，提高 GPU 渲染效率。**




