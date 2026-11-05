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

# UL Solutions和3D Mark
UL： 一家提供安全科学、测试、认证和性能评估服务的公司。  
UL Solutions： UL 在2022年重组后使用的企业名称。  
3DMark： 专门用于测试 PC、笔记本电脑和移动设备图形性能的 Benchmark 软件。  
3DMark 最初由 Futuremark 开发，后来 Futuremark 被 UL 收购，因此现在由 UL Solutions 运营。  

## 一、UL 公司简介

- **全称**：UL Solutions Inc.，纽交所代码 ULS，总部在美国伊利诺伊州 Northbrook（[LinkedIn](https://www.linkedin.com/company/ulsolutions)）。
- **历史**：前身是 1894 年 William Henry Merrill 在芝加哥创办的 Underwriters' Electrical Bureau，20 世纪一直叫 Underwriters Laboratories（[Wikipedia](https://en.wikipedia.org/wiki/UL_(safety_organization))）。2024 年 4 月 IPO 上市（[Wikipedia](https://en.wikipedia.org/wiki/UL_(safety_organization))）。
- **主业**：测试、检验和认证（TIC），以及软件和咨询服务，客户遍布 110 多个国家。常见电器上的 "UL" 认证标志就是它的（[Nasdaq IPO 资料](https://www.nasdaq.com/market-activity/ipos/overview?dealId=1276415-108248)）。
- **和 3DMark 的关系**：3DMark、PCMark 原本是芬兰公司 Futuremark 的产品。[UL 收购了 Futuremark](https://www.ul.com/news/press-releases?dcr=ul-signals-strategic-expansion-into-performance-benchmarking-services-with-futuremark-acquisition)，之后 Futuremark 品牌[迁入 UL 旗下](https://benchmarks.ul.com/en-sc/news/futuremark-rebranding-to-ul-april-23)。现在由 UL Solutions 的 Benchmarks 部门维护 3DMark、PCMark、Procyon（AI 推理）等产品。

## 二、当前主要的 3DMark GPU Benchmark

| Benchmark | 定位 | API / 平台 | 分辨率 | 光追 | RT 实现方式 | 渲染类型 |
|---|---|---|---|---|---|---|
| Steel Nomad | 高端 PC，非光追（Time Spy 的继任者）（[UL](https://benchmarks.ul.com/news/introducing-3dmark-steel-nomad-the-successor-to-3dmark-time-spy)） | DX12 / Vulkan / Metal，跨平台 | 4K（[Tom's Hardware](https://www.tomshardware.com/pc-components/gpus/3dmarks-time-spy-successor-steel-nomad-makes-nvidias-rtx-4090-work-hard-without-ray-tracing)） | 否 | — | 纯光栅 |
| Steel Nomad Light（SNL） | 手机、平板、轻薄本，非光追（[UL](https://support.benchmarks.ul.com/support/solutions/articles/44002528070-overview-of-steel-nomad-light)） | DX12 FL12 / Vulkan 1.1 / Metal；Windows、WoA、Android、iOS、macOS | 2560×1440 | 否 | — | 纯光栅 |
| Solar Bay | 跨平台中等负载光追（[UL](https://support.benchmarks.ul.com/support/solutions/articles/44002466573-overview-of-3dmark-solar-bay-benchmark)） | Windows：Vulkan；Android：Vulkan；iOS/macOS：Metal | — | 是 | Windows 用 ray pipeline，Android 用 ray query，Apple 平台用 Metal | 混合：光栅 + 光追反射 |
| Port Royal | 最早的 DXR 光追 benchmark（[UL](https://benchmarks.ul.com/3dmark)） | DX12 DXR，只有 Windows | — | 是 | ray pipeline（raygen / hit / any-hit）（[Technical Guide](https://s3.amazonaws.com/download-aws.futuremark.com/3dmark-technical-guide.pdf)） | 混合：光栅 G-buffer + 光追反射、硬阴影 |
| Speed Way | 高端 PC，DX12 Ultimate（[UL](https://benchmarks.ul.com/3dmark)） | DX12 Ultimate，DXR Tier 1.1，只有 Windows | 2560×1440 | 是 | ray pipeline 加 ray query：玻璃反射在 pixel shader 里用 ray query（[UL](https://support.benchmarks.ul.com/support/solutions/articles/44002378662-speed-way-engine)） | 混合：mesh shader 光栅 + 光追反射 + lightmap 空间 path-traced GI |
| DXR feature test | 测峰值光线遍历性能（[Technical Guide](https://s3.amazonaws.com/download-aws.futuremark.com/3dmark-technical-guide.pdf)） | DX12，DXR Tier 1.1 | — | 是 | 在 compute shader 里追光线，推断是 inline（`TraceRayInline`），官方没有明说 | 纯光追：不用光栅，只追 primary ray |

### 补充说明

- **Port Royal**：反射 pass 只给粗糙度低于阈值的像素发反射光线。hit shader 只写击中信息，先查 cubemap 缓存，查不到才重新着色。阴影在 async compute 队列上跑，每个像素发一条 shadow ray，用 any-hit 判断遮挡（[Technical Guide](https://s3.amazonaws.com/download-aws.futuremark.com/3dmark-technical-guide.pdf)）。
- **Speed Way**：完全不依赖预计算光照。漫反射 GI 在 lightmap 纹理空间里每帧用 path tracing 算一次反弹，并用 SH 编码。直接光照是 compute shader 里的延迟渲染，AO 用屏幕空间方法（[UL](https://support.benchmarks.ul.com/support/solutions/articles/44002378662-speed-way-engine)）。
- **Solar Bay**：Android 端要求设备支持 Vulkan 1.1 ray query（[Google Play](https://play.google.com/store/apps/details?id=com.futuremark.dmandroid.application&hl=en)）。所以不同平台的分数测的不是同一条 RT 路径。
- **按负载对照**：很重：Speed Way（光追）/ Steel Nomad（非光追）；重：Port Royal；中：Solar Bay（光追）/ Steel Nomad Light（非光追）；轻：Wild Life Extreme（[TechSpot](https://www.techspot.com/downloads/5775-3dmark.html)）。

### 选测试的建议

| 想测什么 | 推荐测试 |
|---|---|
| RT core traversal 吞吐 | DXR feature test（只有 primary ray，几乎没有着色开销） |
| hit shader 调度、SBT 等 pipeline 开销 | Port Royal、Speed Way |
| 移动端 ray query 性能 | Android 上的 Solar Bay |
| 跨架构的纯光栅性能 | Steel Nomad / Steel Nomad Light |

## 三、下一代全路径追踪 3DMark Benchmark 计划

### 已公开的信息（截至11/05/2026）

| 项目 | 内容 |
|---|---|
| 公布时间 | 2026-06-03（[UL 公告](https://benchmarks.ul.com/en/news/announcing-our-next-generation-3dmark-gpu-benchmark-and-its-first-sponsor)） |
| 渲染方式 | fully path-traced，全路径追踪 |
| 分辨率 | 有 4K 原生模式 |
| 超分 / 帧生成 | 可以选不同厂商的 AI 超分和帧生成，横向比较各家方案 |
| 赞助商 | Thermal Grizzly（第一个），Seasonic（2026-07-24 加入，[UL 公告](https://benchmarks.ul.com/news/seasonic-joins-as-a-sponsor-of-the-next-3dmark-gpu-benchmark)） |
| 首次亮相 | Computex 2026，在 Thermal Grizzly 展台放了短 demo，之后发布了预告片（[Lowyat](https://www.lowyat.net/2026/395420/ul-releases-next-gen-path-tracing-3dmark-test-teaser/)） |
| 名称 / 发布日期 | 都还没公布 |

### 媒体报道（未经官方确认）

- 有媒体称它会支持 NVIDIA DLSS 4.5 和 AMD FSR 4.1（[GameGPU](https://en.gamegpu.com/news/zhelezo/novyj-3dmark-poluchit-trassirovku-puti-i-nativnoe-razreshenie-4k)）。UL 官方公告只说"支持多家厂商的方案"，没有写具体版本。

### 尚未公开的技术细节

- 图形 API：DX12 还是 Vulkan，会不会跨平台。
- RT 实现：ray pipeline 还是 ray query，有没有 SER（Shader Execution Reordering）、opacity micromap 等特性。
- 降噪方案：传统时空降噪，还是 AI 降噪（ray reconstruction 一类）。
- 按以往惯例，这些会在发布时写进 3DMark Technical Guide。

### 意义

- 现有的 Port Royal、Speed Way、Solar Bay 都是光栅加光追的混合渲染，测的是"光追只占一部分负载"时的性能。
- 全路径追踪会把瓶颈推到 traversal 吞吐、shader divergence、显存带宽和降噪上，更接近现在游戏开 path tracing 模式时的负载。
- 再加上超分和帧生成，它很可能成为对比下一代 RT 硬件和各家 AI 渲染方案的主力测试。


# 游戏自带Benchmark
| Benchmark                 | 光追内容                                                                                         | 特点                                                                                                               |
| ------------------------- | -------------------------------------------------------------------------------------------- | ---------------------------------------------------------------------------------------------------------------- |
| Quake II RTX              | 全路径追踪                                                                                        | Vulkan 下能切换 VK_KHR_ray_tracing_pipeline 和 VK_KHR_ray_query，Phoronix 的测试也把两者分开测（OpenBenchmarkingopenbenchmarking） |
| 《黑神话：悟空》Benchmark Tool    | Full Ray Tracing（路径追踪）：Low 是半分辨率 ReSTIR GI 加 RT 阴影，Very High 再加全分辨率反射和焦散（DSOGamingdsogaming） | Steam 上免费下载，独立于游戏本体（Steamstore.steampowered）                                                                     |
| 《赛博朋克 2077》RT Overdrive 等 | 路径追踪                                                                                         | 媒体评测最常用，悟空的 Full RT 就常被拿来和它比（HotHardwarehothardware）                                                             |

为什么手机光追都走 ray query
- ray query 可以直接在 fragment shader 或 compute shader 里发光线，不需要完整的 ray pipeline 和 SBT。这样省带宽也省功耗，更适合移动端的预算（Android Developers）。所以手机上测的是"混合渲染加少量光线"这种场景，和 PC 上的全路径追踪测试不能直接对比。

手游上的光追例子  
| 游戏                     | 光追用在哪     | 实现方式                                                                                    |
| ---------------------- | --------- | --------------------------------------------------------------------------------------- |
| 《暗黑破坏神：不朽》             | 实时反射      | Vulkan 硬件光追。Google 的案例文章特别强调 ray query 是移动端光追的核心技术（Android Developersdeveloper.android） |
| 《暗区突围》（Arena Breakout） | 反射、AO、软阴影 | Vulkan ray query（Lost on Arrivallost-on-arrival）                                        |
| War Thunder Mobile     | 阴影        | 硬件光追，Snapdragon 机型上可以开（Lost on Arrivallost-on-arrival）                                  |


# 离线渲染器Benchmark
这类渲染器的工作方式是：每个像素累积成百上千个采样，直到画面噪点收敛到可以接受。一帧要几秒到几分钟甚至几小时。  
实时光追则是每像素只追 1 条左右的光线，再靠降噪把噪点压下去，两者的目标不一样。  

| Benchmark                     | 开发方         | 光追后端                                                                                              | 计分方式                                  |
| ----------------------------- | ----------- | ------------------------------------------------------------------------------------------------- | ------------------------------------- |
| Blender Benchmark / Open Data | Blender 基金会 | Cycles：NVIDIA 用 OptiX，AMD 用 HIP RT，Intel 用 oneAPI，Apple 用 MetalRT，都调用硬件光追（Blender 手册docs.blender） | 每分钟渲染的采样数，有公开数据库，按后端分开统计pixelrankings |
| V-Ray 6 Benchmark             | Chaos       | 同一场景可以切换 RTX 和 CUDA 模式（Chaoschaos）                                                                | 两种模式的分数不能直接比（iRenderirendering）       |
| OctaneBench                   | OTOY        | RTX 加速可开可关                                                                                        | 以 GTX 980 为 100 分做基准（OTOYrender.otoy） |

为什么手机上一般不用离线渲染器 benchmark
- 算力和功耗差两个数量级。 手机 GPU 只有几瓦的功耗预算，持续高负载还会因为过热降频。桌面卡动辄几百瓦，渲染农场还会一台机器插 8 张卡。
- 显存装不下场景。 离线渲染的场景经常有几 GB 到几十 GB，几何和 BVH 通常必须整个放进显存。手机分给单个 app 的内存很有限。有 iPad 用户反映，Octane X 在大多数只有 2 到 4 GB 内存的 iPad 上根本跑不顺（Nomad Sculpt 论坛）。
- 软件生态不支持。 这几个渲染器的 GPU 后端都是 CUDA/OptiX、HIP、oneAPI 或 Metal。Android 上只有 Vulkan，它们都没有 Vulkan 后端，Blender 本身也不出 Android 和 iOS 版。
- 移动端的光追是为实时游戏设计的。 Android 上的光追主要通过 Vulkan ray query 暴露，3DMark Solar Bay 就要求设备支持 Vulkan 1.1 ray query（Google Play）。它的设计目标是混合渲染里少量光线、配合降噪，不是大批量的离线采样。
- 使用场景不同。 离线渲染是做影视和设计时的生产工具，手机用户真正需要的是实时游戏的光追效果，所以手机上的光追测试都是 Solar Bay 这种实时混合渲染 benchmark。

# Evolve和分项Benchmark
Evolve 是荷兰 Traverse Research 做的跨平台 GPU benchmark，对比的话可以这样概括：Solar Bay 给出一个"光追游戏性能"总分，Evolve 则把 GPU 的每类负载拆开计分，更接近一个分析工具。  

Evolve 是什么  
- 开发方：荷兰布雷达的 Traverse Research，2025 年 3 月发布（Evolve 新闻稿）。Android Authority 介绍的创始人是 Jasper Bekkers（Android Authority）。  
- 平台：Windows 和 Android，都要求 GPU 支持硬件光追（Evolve 官网、Microsoft Store）。  
- 引擎：自研的 Breda renderer，从一开始就围绕硬件光追设计（Evolve）。  
- 价格：免费版可以直接跑。专业版能导出图表和全部原始数据，价格没公开（Android Authority）。  
- "evolutionary benchmarking"：会持续加入新技术，不是一个固定不变的测试。比如 1.0.9 版默认打开了 AS compaction，这会改变 Acceleration Structure 分数（Evolve release notes）。所以不同版本的分数不能直接比。  

七项分数
| 分数                     | 测什么                                                                                                 |
| ---------------------- | --------------------------------------------------------------------------------------------------- |
| Ray Tracing            | 所有硬件光追 pass，细分为 ReSTIR DI、GI tracing、反射、阴影、透明/折射、水面、Wavefront Pathtracing                           |
| Acceleration Structure | GPU 端的 BLAS/TLAS build 和 BLAS refit                                                                 |
| Rasterization          | 硬件光栅器                                                                                               |
| Compute                | GPU compute 负载                                                                                      |
| Driver                 | CPU 端的驱动开销，目前主要是创建和更新 AS                                                                            |
| Energy                 | 通过 Android Power Stats HAL 读真实功耗。厂商不开放这个接口的机型就不出这项分，不会用电池 API 估算（Android Authorityandroidauthority） |
| Workgraphs             | DX12 work graphs，只在路径追踪模式下出分                                                                        |

- Mode：可以选 inline（ray query）或 pipeline，而且能按每个 render pass 分别设置（Evolve FAQ）。
- Renderer：可以选 Hybrid（混合渲染）或 Path Tracing（纯路径追踪，只跑 Wavefront Pathtracing 这一个 pass）（Technical Guide）。
- 官方实测：所有光追算法都分别用 inline 和 pipeline 实现了一遍。在 RTX 5070 Ti 和 Arc B580 上两者分数差不多，在 RX 9070 XT 上 pipeline 明显更快（Evolve 博客）。

手机实测数据
- Android Authority 用 Evolve 测了几款旗舰芯片（Android Authority）：
- Adreno 840 对比 Mali G1：光追快 27%，光栅快 40%，compute 领先 46%。
- Adreno 840 对比 Adreno 830：光追快 85%。
- Adreno 840 每帧 GPU 时间分布：光追 27.7%，光栅 17%，compute 45.7%。
- Mali-G925：每帧超过 50% 的时间耗在光追上，说明它的 RT 硬件是瓶颈。

## Evolve 和 Solar Bay 对比
相同点
- 都是游戏风格的实时光追 benchmark，都包含混合渲染（光栅加光追效果）。
- 都跨 Windows 和 Android，Android 端都走 Vulkan 光追。
- 都是免费版加付费专业版的模式，也都有在线排行榜。

不同点
| 维度                   | Solar Bay                                                             | Evolve                                           |
| -------------------- | --------------------------------------------------------------------- | ------------------------------------------------ |
| 开发方                  | UL（3DMark）                                                            | Traverse Research                                |
| 平台                   | Windows、Android、iOS、macOS                                             | Windows、Android，不支持 Apple 平台                     |
| API                  | Vulkan、Metal                                                          | Vulkan、DX12（DXR）                                 |
| ray query / pipeline | 由平台固定：Windows 用 pipeline，Android 用 ray query（ULsupport.benchmarks.ul） | 用户可以选，而且能按每个 pass 设置                             |
| 渲染模式                 | 只有混合渲染，光追主要用在反射上                                                      | 混合渲染或纯路径追踪，两种都有                                  |
| 光追效果                 | 主要是反射，分三段，光追负载逐段加大（Google Play）                                       | ReSTIR DI、GI、反射、阴影、折射、水面                         |
| 输出                   | 一个总分，加上稳定性测试                                                          | 七项分数，加上每个 pass 的 GPU 时间和功耗                       |
| AS build 是否单独计分      | 不单独计分                                                                 | 单独计分（BLAS/TLAS build、refit），CPU 端开销另算在 Driver 分里 |
| 功耗                   | 不测                                                                    | 读真实功耗                                            |
| 版本稳定性                | 版本固定，分数长期可比                                                           | 会持续更新技术，跨版本分数不一定可比                               |
| 业界地位                 | 媒体和厂商的标准对比项                                                           | 比较新，适合深入分析                                       |

怎么选
- 想要一个大家都认的"手机光追分"：用 Solar Bay。数据库最大，媒体评测也最常引用。
- 想知道瓶颈在哪：用 Evolve。它能拆开看是 BVH 构建、traversal 还是 shading 慢，还能在同一台设备上直接对比 inline 和 pipeline。这和你在 LuminError 里做 Ray Query 和 ray pipeline 对比的思路很接近。




