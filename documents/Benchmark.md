# Manhattan、Aztec Ruins、Solar Bay 技术介绍

这三个 benchmark 可以理解为三代 GPU 图形负载：

- Manhattan：移动端复杂延迟渲染。
- Aztec Ruins：跨 API、subpass-based deferred rendering 和 compute 后处理。
- Solar Bay：光栅渲染与实时光追反射混合负载。

精确的 frame、render pass、draw call 统计必须绑定测试版本、API、画质、分辨率和场景时间点。以下区分公开实现信息、逻辑示意和需要实际抓帧确认的参数。

## 1. 历史与版本

| 测试 | 发布时间 | 所属套件 | 定位 |
|---|---|---|---|
| Manhattan 3.0 | 2014 年初 | GFXBench 3.0 | OpenGL ES 3.0 时代的复杂场景测试，使用 MRT 和延迟渲染 |
| Manhattan 3.1 | 2015 年 3 月 | GFXBench 3.1 | 延续 Manhattan 场景，但增强 compute 等负载，不是单纯换 API |
| Aztec Ruins | 2018 年 8 月正式发布 | GFXBench 5.0 | 跨 API 场景，强调现代移动端渲染 |
| Solar Bay | 2023 年 8 月 | 3DMark | 面向高端手机、轻薄电脑的跨平台实时光追测试 |
| Solar Bay Extreme | 2025 年 8 月 20 日 | 3DMark | 更重的独立光追测试，增加反射、玻璃和软阴影效果 |

### Manhattan

Manhattan 3.0 引入多渲染目标、延迟光照、大量灯光和后处理。公开介绍指出，其 diffuse/specular lighting 涉及超过 60 个灯光，并使用 cubemap reflection、emission、triplanar mapping 和 instanced mesh rendering。

Manhattan 3.1 说明：几何量不能直接代表测试复杂度。

发布报道指出，其几何资产经过优化，几何量比之前减少约 30%，但新增 compute 等效果后，报道中的测试帧率反而更低。这是当时报道中的版本比较，不是所有 GPU 都满足的性能规律。

### Aztec Ruins

Aztec 的关键不只是场景更复杂，而是把以下内容放到同一个游戏式 workload 中：

- 跨 API 实现。
- Subpass-based deferred rendering。
- 动态光照、实时阴影和动态 GI。
- Compute 实现的 HDR tone mapping、bloom、motion blur。

开发者说明提到，几何与光照阶段利用本地内存缓存。

注意：动态 GI 不等于硬件光追。不能因为测试支持 Vulkan，就认定它使用了 ray tracing。

### Solar Bay

Solar Bay 将光追反射加入原本的光栅引擎，并通过三个阶段增加光追负载。

它测的是混合渲染系统，不是只测 ray traversal 的微基准。

Solar Bay Extreme 是独立的更重测试，不能和原版混为一个测试，也不应认为它只是原版换了分辨率。

## 2. Frame 与测试配置

“Frame”需要分成两个问题：

1. 一次 benchmark 渲染多少帧？
2. 一帧内部执行哪些阶段？

### 配置与时间结构

| 项目 | Manhattan | Aztec Ruins | Solar Bay 原版 |
|---|---|---|---|
| 主要变体 | Manhattan 3.0、3.1 | Normal、High，以及不同 API | 常规 benchmark、stress test |
| API | 3.0 对应 GLES 3.0；3.1 对应 GLES 3.1；其他平台有相应实现 | Vulkan、OpenGL ES、Metal 等，具体可用测试取决于平台 | Android/Windows：Vulkan 1.1 加所需光追扩展；Apple：Metal |
| 常见标准分辨率 | Manhattan 3.0 offscreen：1920×1080 | Normal offscreen：1920×1080；High offscreen：2560×1440 | 内部渲染 2560×1440，再缩放到显示设备 |
| 时间结构 | 2014 年资料描述 Manhattan 为 62 秒；不能套用于所有后续版本 | 现有资料不足以确认各版本统一时长 | 常规约 1 分钟；stress test 20 分钟 |
| 场景负载变化 | 随镜头和内容变化 | 随镜头和内容变化 | 三段光追负载，后两段平均目标分别约为第一段的 2×、3× |

### 帧数与 FPS

固定时长不等于固定帧数：

```text
Average FPS = 完成帧数 / 统计时长
```

例如，统计区间恰好为 60 秒，完成 3,600 帧，对应 60 FPS。

这是计算示例，不是某个 benchmark 预先规定的帧数。

工程分析应记录场景时间点，而不仅是 frame index。不同设备跑到第 1,000 帧时，可能处于不同场景位置；同一镜头、同一 Solar Bay section 更适合对比。

### 单帧公开特征

| 维度 | Manhattan | Aztec Ruins | Solar Bay 原版 |
|---|---|---|---|
| 基础渲染 | Deferred + forward 混合 | Subpass-based deferred | PBR deferred + clustered lighting |
| 几何阶段 | MRT，支持 instancing | 几何与光照阶段利用本地缓存 | 临时 PBR G-buffer；fragment shader 计算 motion vectors |
| 光照 | 3.0 公开资料描述超过 60 个灯光 | 动态光照、实时阴影、动态 GI | Linear HDR，clustered lighting |
| 反射 | Cubemap reflection | 公开功能列表不将其定义为硬件光追测试 | 镜面类表面使用光追；其他表面使用局部 cubemap |
| 后处理 | Bloom、DOF；3.1 增加 compute 等增强 | Compute tone mapping、bloom、motion blur | TAA、XeGTAO、compute FFT bloom、DOF |
| 粒子与透明 | 精确实现需抓帧 | 精确实现需抓帧 | Compute 粒子模拟；weighted blended OIT |

Solar Bay 的体积 ray casting 不能直接等同于 RT 硬件执行场景 BVH traversal。

## 3. Render pass 与帧结构

### 先统一统计口径

| 概念 | 含义 | 注意点 |
|---|---|---|
| 逻辑渲染阶段 | G-buffer、lighting、shadow、bloom 等算法阶段 | 一个阶段可能拆成多个 API pass 或 dispatch |
| Vulkan render pass 实例 | 实际开始、结束的 render-pass 执行范围 | 与 compute、RT dispatch 分开记录 |
| Subpass | 一个 render pass 内部的子阶段 | Subpass 数不等于 render pass 数 |
| GPU 硬件 pass/job | 驱动与硬件实际执行的工作划分 | 不保证与应用 API 边界一一对应 |

不能从 deferred rendering 推导出“固定两个 render pass”。

也不能从使用 subpass 推导出“G-buffer 一定不写外部内存”。

### Manhattan 帧结构

下面是逻辑示意，不是实际抓帧得到的精确调用顺序：

```text
几何 / G-buffer
    ↓
延迟光照
    ↓
额外 forward rendering
    ↓
Bloom / DOF 等后处理
    ↓
最终输出
```

对于 GLES 版本，重点是 framebuffer、attachment 和阶段依赖，而不是寻找 VkRenderPass 对象。

抓帧时检查：

- G-buffer attachment 数量、格式和分辨率。
- 每个阶段是否读取之前的 render target。
- Forward 阶段的 overdraw 和 blending。
- 后处理是否使用降分辨率、中间纹理、多次过滤。

这些是检查项，不是已确认的固定实现参数。

### Aztec 帧结构

Aztec 使用 subpass-based deferred rendering，几何与光照阶段利用本地内存缓存。

概念示意：

```text
Deferred rendering 范围
    Geometry 阶段
        ↓
    Lighting 阶段
        ↓
Compute 后处理
    HDR tone mapping / bloom / motion blur
        ↓
最终输出
```

精确 render pass 数、subpass 数、attachment 格式，以及阴影/GI 的执行阶段，需要实际抓帧确认。

重点核对：

- Geometry 与 lighting 是否在同一个 render-pass 实例内。
- G-buffer 的生产者与消费者如何连接。
- Attachment load/store 行为。
- Compute 后处理读取哪些图像。
- 是否存在额外存储、布局转换或同步。
- 驱动是否将阶段映射为预期的硬件执行方式。

Subpass 优化和 compute 后处理成本应分别测量，不能只看整帧 FPS 判断是哪一个起作用。

### Solar Bay 帧结构

| 模块 | 公开实现 | 分析重点 |
|---|---|---|
| 不透明几何 | Deferred rendering，临时 PBR G-buffer | 几何吞吐、MRT 写入、材质 shader |
| 光照 | Clustered lighting，linear HDR | 光源列表、shading 成本 |
| 光追反射 | 反射面板、开场地板等使用光追 | Ray 工作量、反射 shader、场景复杂度 |
| 非光追反射 | 局部 cubemap，根据有效体积混合采样 | Texture sampling |
| 粒子 | Compute shader 模拟 | Dispatch、buffer 读写、同步 |
| 透明 | Weighted blended OIT，两个临时 RT，再合成 | Blending、overdraw、合成 |
| TAA | Motion vectors、历史 depth/illumination、variance clipping | 历史纹理读取、resolve |
| AO | XeGTAO | 屏幕空间采样 |
| Bloom | 降分辨率 compute FFT，利用 workgroup shared memory | Compute、shared memory、同步 |
| DOF | 半分辨率，过滤部分两遍执行 | 中间 RT、过滤 |

这些模块不能直接相加成为“10 个 render pass”。

其中有 compute、有过滤子阶段，也可能合并或拆分。

官方说明：Solar Bay 使用 primary command buffers，不使用 geometry shader 或 tessellation。

## 4. Draw call 与光追统计

### 是否有固定数量？

现有公开资料不足以确认这三个测试统一、固定的每帧 draw call / render pass 数。

因此，不应给出没有配置与抓帧依据的固定数字，例如：

```text
Manhattan 固定有 X 个 draw calls。
Aztec 固定有 Y 个 render passes。
```

2015 年关于未来 GFXBench 5.0 的报道提过“超过 6,000 draw calls”，但它描述的是当时的规划，不能直接作为 2018 年正式 Aztec Ruins 的实测每帧数量。

### 建议统计项目

| 统计项 | 反映什么 | 单独使用的问题 |
|---|---|---|
| Draw API 调用次数 | CPU/driver 提交粒度 | 不代表几何量、像素量、shader 成本 |
| Indirect 逻辑 draw 数 | 间接命令实际包含的绘制数量 | 一个 API 调用可包含多个 draw |
| Instance 数 | 实例化规模 | 同一 draw 的 GPU 工作量可差别很大 |
| Triangle / vertex 数 | 几何输入规模 | 不代表覆盖、overdraw、材质复杂度 |
| Dispatch 数 | Compute 提交次数 | 不代表 workgroup 数和执行成本 |
| Trace-rays 调用次数 | RT pipeline 工作提交次数 | 不代表 ray 数，也覆盖不了 ray query |
| AS build/update | 加速结构相关工作 | 应与 tracing 成本分开 |
| GPU 时间 | 阶段成本 | 还需 counters 解释瓶颈 |

### Solar Bay 的特殊点

Solar Bay 在 Android 使用 ray query，在 Windows 使用 RT pipeline。

因此：

```text
vkCmdTraceRaysKHR count = 0
```

不代表没有光追。

Ray query 在普通 shader 内执行，需要结合 shader/pipeline 和 GPU profiler 判断。

三个 section 的平均光追 workload 目标：

```text
Section 1 ≈ 1×
Section 2 ≈ 2×
Section 3 ≈ 3×
```

不是：

```text
1× → 2× → 4×
```

也不意味着整帧时间严格变成两倍、三倍，因为测试还包含几何、光照、透明和后处理。

## 5. 工程对比与抓帧模板

将结果拆成两层：

1. 固定配置的整段性能。
2. 同一场景时刻的单帧分析。

不要只比较排行榜上的一个 FPS。

### 抓帧记录模板

| Test/version | API | Resolution | Scene time/section | Draw API calls | Indirect logical draws | Dispatch calls | Trace-rays calls | AS build/update | Render-pass instances | Subpasses | GPU ms | Notes |
|---|---|---|---|---:|---:|---:|---:|---:|---:|---:|---:|---|
| 待填 | 待填 | 待填 | 待填 | — | — | — | — | — | — | — | — | 标明是否含 UI / present |

### 分析优先级

| 测试 | 优先检查 | 防止的错误结论 |
|---|---|---|
| Manhattan 3.0 | MRT、光照、forward overdraw、后处理 | 仅凭 draw 数判断负载大小 |
| Manhattan 3.1 | 与 3.0 的 compute/后处理差异、几何变化 | 当成完全相同 workload 的 API 对比 |
| Aztec Ruins | Subpass、attachment 流转、compute、同步 | 认为 subpass 必然消除外部内存访问 |
| Solar Bay | 分 section 的反射成本、ray query/pipeline、非 RT 阶段 | 把总分等同于纯 RT 单元吞吐 |
| Solar Bay stress | 随时间变化的性能、温度、频率 | 用短时成绩代替持续性能 |

Solar Bay 没有独立 CPU test，但不意味着不会 CPU-bound。

场景更新、可见性计算和命令录制采用多线程；CPU 提交速度不够也会限制成绩。

应先区分 CPU 提交时间与 GPU 执行时间，再讨论 shader、带宽或光追瓶颈。

## 资料链接

以下对应前面介绍使用的公开资料。

- [Kishonti 历史](https://kishonti.net/milestones)
- [Manhattan 3.0 技术介绍](https://www.tomshardware.com/reviews/gfxbench-3-graphics-performance,3743-2.html)
- [Manhattan 3.1 发布与 GFXBench 5.0 规划](https://www.tomshardware.com/news/gfxbench-3.1-low-level-gfxbench-5,28738.html)
- [Aztec Ruins 开发者功能说明](https://www.apkmirror.com/apk/kishonti-ltd/gfxbench-benchmark/gfxbench-benchmark-5-0-5-release/)
- [Solar Bay overview](https://support.benchmarks.ul.com/support/solutions/articles/44002466573-overview-of-3dmark-solar-bay-benchmark)
- [Solar Bay graphics test](https://support.benchmarks.ul.com/support/solutions/articles/44002466575-solar-bay-graphics-test)
- [Solar Bay engine](https://support.benchmarks.ul.com/support/solutions/articles/44002466580-solar-bay-engine)
- [Solar Bay Extreme 发布](https://benchmarks.ul.com/news/3dmark-solar-bay-extreme-is-available-now-)
