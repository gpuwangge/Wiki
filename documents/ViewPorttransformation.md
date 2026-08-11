# NDC
在图形学里，NDC 指的是 Normalized Device Coordinates（规范化设备坐标），是图形管线中位于「裁剪空间」和「视口 / 屏幕空间」之间的一个中间空间：  
顶点经投影矩阵变换得到 clip space，再做透视除法，就进入 NDC。  
之后由 viewport transform 把 NDC 映射到 framebuffer 像素坐标。 

整个流程可以概括为：

Object Space
→
World Space
→
View Space
→
Clip Space
→
NDC
→
Framebuffer Space
Object Space→World Space→View Space→Clip Space→NDC→Framebuffer Space

NDC 是一个固定范围的立方体：

OpenGL 的 NDC 范围：

$$
(x, y, z) \in [-1, 1]^3
$$

Vulkan 的 NDC 范围：

$$
(x, y, z) \in [-1, 1] \times [-1, 1] \times [0, 1]
$$

在这个立方体范围内的几何会被保留并送去光栅化，立方体外的部分在裁剪阶段被丢弃或裁剪重建。

坐标已经从齐次 4D 变成 3D，方便后续做 viewport transform 和光栅化。


# ViewPort Transformation
