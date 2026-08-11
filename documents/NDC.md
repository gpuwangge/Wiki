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

在这个立方体范围内的几何会被保留并送去光栅化，立方体外的部分在裁剪阶段被丢弃或裁剪重建。  
坐标已经从齐次 4D 变成 3D，方便后续做 viewport transform 和光栅化。  

## 从 Clip Space 到 NDC

Vertex Shader 最终输出的是齐次裁剪坐标：

```glsl
gl_Position = projection * view * model * vec4(position, 1.0);
```

可以表示为：

$$
p_{\text{clip}} =
\begin{pmatrix}
x_c \\
y_c \\
z_c \\
w_c
\end{pmatrix}
$$

然后，固定功能硬件会做透视除法：

$$
x_{\text{ndc}} = \frac{x_c}{w_c}
$$

$$
y_{\text{ndc}} = \frac{y_c}{w_c}
$$

$$
z_{\text{ndc}} = \frac{z_c}{w_c}
$$

因此：

$$
p_{\text{ndc}} =
\left(
\frac{x_c}{w_c},
\frac{y_c}{w_c},
\frac{z_c}{w_c}
\right)
$$

## 为什么需要透视除法

透视投影要实现「近大远小」。

假设两个点有相同的 X 偏移，但一个点离相机更远。经过投影矩阵后，远处点通常会有更大的 \(w_c\)。透视除法之后：

$$
x_{\text{ndc}} = \frac{x_c}{w_c}
$$

远处点的 \(x_{\text{ndc}}\) 更接近 0，于是它在屏幕上看起来更靠近中心、尺寸更小。

示例：

$$
p_{\text{clip}}^{\text{near}} = (2, 0, 0.5, 2)
$$

$$
p_{\text{ndc}}^{\text{near}} = (1, 0, 0.25)
$$

更远的点：

$$
p_{\text{clip}}^{\text{far}} = (2, 0, 5, 10)
$$

$$
p_{\text{ndc}}^{\text{far}} = (0.2, 0, 0.5)
$$

## Vulkan 的裁剪条件与 NDC

在透视除法之前，Vulkan 对 clip space 的可见范围可理解为：

$$
-w_c \le x_c \le w_c
$$

$$
-w_c \le y_c \le w_c
$$

$$
0 \le z_c \le w_c
$$

经过透视除法后，对应为：

$$
-1 \le \frac{x_c}{w_c} \le 1
$$

$$
-1 \le \frac{y_c}{w_c} \le 1
$$

$$
0 \le \frac{z_c}{w_c} \le 1
$$

也就是前面写的：

$$
(x, y, z) \in [-1, 1] \times [-1, 1] \times [0, 1]
$$

## NDC 到屏幕坐标（Viewport Transform）

NDC 不是最终的像素坐标。接下来，viewport transformation 会将它映射到 framebuffer。

设 viewport 参数为：

```cpp
VkViewport viewport{};
viewport.x        = x_v;
viewport.y        = y_v;
viewport.width    = W;
viewport.height   = H;
viewport.minDepth = d_min;
viewport.maxDepth = d_max;
```

X 坐标映射：

$$
x_{\text{fb}} = x_v + \frac{x_{\text{ndc}} + 1}{2} \, W
$$

Y 坐标映射：

$$
y_{\text{fb}} = y_v + \frac{y_{\text{ndc}} + 1}{2} \, H
$$

Vulkan 深度映射：

$$
z_{\text{fb}} = d_{\min} + z_{\text{ndc}} \, (d_{\max} - d_{\min})
$$

如果使用默认深度范围：

$$
d_{\min} = 0, \quad d_{\max} = 1
$$

那么：

$$
z_{\text{fb}} = z_{\text{ndc}}
$$

## 示例：1920×1080 Viewport

设 viewport 覆盖整个 1920×1080 framebuffer：

```cpp
VkViewport viewport{};
viewport.x = 0.0f;
viewport.y = 0.0f;
viewport.width  = 1920.0f;
viewport.height = 1080.0f;
viewport.minDepth = 0.0f;
viewport.maxDepth = 1.0f;
```

NDC 中心点：

$$
(0, 0)
$$

映射到 framebuffer 中心：

$$
x_{\text{fb}} = \frac{0 + 1}{2} \times 1920 = 960
$$

$$
y_{\text{fb}} = \frac{0 + 1}{2} \times 1080 = 540
$$

因此：

$$
(0, 0) \rightarrow (960, 540)
$$

再看一个完整 NDC 点：

$$
p_{\text{ndc}} = (-0.5, 0.25, 0.8)
$$

X 坐标：

$$
x_{\text{fb}} = \frac{-0.5 + 1}{2} \times 1920 = 480
$$

Y 坐标：

$$
y_{\text{fb}} = \frac{0.25 + 1}{2} \times 1080 = 675
$$

深度值（假设 \(d_{\min}=0, d_{\max}=1\)）：

$$
z_{\text{fb}} = 0.8
$$

最终映射结果：

$$
(-0.5, 0.25, 0.8) \rightarrow (480, 675, 0.8)
$$

## Vulkan 的 Y 轴翻转

Vulkan 支持通过负的 viewport height 翻转 Y 轴，例如：

```cpp
VkViewport viewport{};
viewport.x      = 0.0f;
viewport.y      = static_cast<float>(swapchainExtent.height);
viewport.width  = static_cast<float>(swapchainExtent.width);
viewport.height = -static_cast<float>(swapchainExtent.height);
viewport.minDepth = 0.0f;
viewport.maxDepth = 1.0f;
```

设 framebuffer 高度为 \(H\)，则：

$$
y_v = H, \quad \text{viewportHeight} = -H
$$

此时 Y 映射为：

$$
y_{\text{fb}} = H + \frac{y_{\text{ndc}} + 1}{2} \, (-H)
$$

于是：

$$
y_{\text{ndc}} = 1 \rightarrow y_{\text{fb}} = 0
$$

$$
y_{\text{ndc}} = -1 \rightarrow y_{\text{fb}} = H
$$

## NDC、Viewport 与 Scissor 的区别

- **NDC**：透视除法后的标准化坐标空间。
- **Viewport**：把 NDC 映射到 framebuffer 的某个矩形区域。
- **Scissor**：限制最终允许写入的像素区域。

比如 viewport 可以是整个 1920×1080，而 scissor 只允许 500×500 的子区域写入。

## 小结

从 clip space：

$$
p_{\text{clip}} =
\begin{pmatrix}
x_c \\
y_c \\
z_c \\
w_c
\end{pmatrix}
$$

到 NDC：

$$
p_{\text{ndc}} =
\left(
\frac{x_c}{w_c},
\frac{y_c}{w_c},
\frac{z_c}{w_c}
\right)
$$

在 Vulkan 中：

$$
(x, y, z) \in [-1, 1] \times [-1, 1] \times [0, 1]
$$

然后通过 viewport transform：

$$
x_{\text{fb}} = x_v + \frac{x_{\text{ndc}} + 1}{2} \, W
$$

$$
y_{\text{fb}} = y_v + \frac{y_{\text{ndc}} + 1}{2} \, H
$$

$$
z_{\text{fb}} = d_{\min} + z_{\text{ndc}} (d_{\max} - d_{\min})
$$



# ViewPort Transformation
由上述NDC分析可知，Viewport transformation（视口变换）是图形管线中把裁剪空间经透视除法得到的 NDC 坐标，映射到某个实际渲染区域的过程：也就是从标准化范围映射到 framebuffer 的像素坐标与深度范围。  
它发生在 vertex shader 输出 gl_Position 之后、光栅化之前。NDC 的 x,y 通常在 [−1,1]；视口变换决定它们具体落在屏幕的哪里、多大。  
视口本质上就是一次缩放加平移：把 NDC 正方形/立方体的一部分坐标范围，嵌入一块指定大小的渲染目标区域。  
最后，viewport 不是裁剪机制：三角形是否在可见范围，先由 clip space clipping 决定；viewport 负责把保留下来的 NDC 坐标映射至目标区域。scissor 则是在 rasterization 后限制 fragment 覆盖区域。  


