# NDC
在图形学里，NDC 指的是 Normalized Device Coordinates（规范化设备坐标），是图形管线中位于「裁剪空间」和「视口 / 屏幕空间」之间的一个中间空间：  
顶点经投影矩阵变换得到 clip space，再做透视除法，就进入 NDC。  
之后由 viewport transform 把 NDC 映射到 framebuffer 像素坐标。 

整个流程可以概括为：

$$
\text{Object Space}
\rightarrow
\text{World Space}
\rightarrow
\text{View Space}
\rightarrow
\text{Clip Space}
\rightarrow
\text{NDC}
\rightarrow
\text{Framebuffer Space}
$$

## NDC 的作用

NDC 的主要目标是将不同的相机参数、投影矩阵和屏幕分辨率，统一到一个固定范围内。

无论窗口是 800×600、1920×1080，还是 4K，顶点在 NDC 中都遵循相同的范围约定。硬件只需将 NDC 坐标映射到当前 viewport，即可完成对任意分辨率的光栅化。

NDC 也是裁剪后的标准空间，超出该范围的图元会被裁剪或丢弃。

## NDC 的坐标范围

OpenGL 的 NDC 范围：

$$
(x, y, z) \in [-1, 1]^3
$$

也就是：

$$
-1 \le x \le 1
$$

$$
-1 \le y \le 1
$$

$$
-1 \le z \le 1
$$

Vulkan 的 NDC 范围：

$$
(x, y, z) \in [-1, 1] \times [-1, 1] \times [0, 1]
$$

也就是：

$$
-1 \le x \le 1
$$

$$
-1 \le y \le 1
$$

$$
0 \le z \le 1
$$


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
