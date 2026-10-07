# Mesa3D 介绍
一个开源的“图形 API 实现 + GPU 用户态驱动（UMD）+ shader compiler + 一套图形软件基础设施”。  
它不是单纯的 API，也不是单纯的 driver,它是“真实 GPU 软件栈 ↔ GPU 硬件架构”之间的一个窗口。  

Mesa 最早是 Brian Paul 在 1993 年开始开发的 OpenGL 实现，1995 年发布 Mesa 1.0。  

后来从单纯的 OpenGL implementation，逐渐发展成支持 OpenGL、OpenGL ES、Vulkan、OpenCL、EGL、VA-API 等的完整开源 graphics stack。  

现在 Mesa 里面包含大量不同 GPU 的 driver，例如：

- AMD → RadeonSI / RADV
- Intel → Iris / ANV
- NVIDIA → NVK
- Qualcomm Adreno → Freedreno
- ARM Mali → Panfrost
- Broadcom → V3D / V3DV
- Vivante → Etnaviv

同时还有：
- Zink：OpenGL → Vulkan
- D3D12：OpenGL → D3D12
- LLVMpipe / Softpipe：CPU software rendering

```
Application
     │
     │ OpenGL / Vulkan / OpenGL ES
     ▼
+-----------------------+
|        Mesa            |
|                       |
| API implementation    |
| Gallium infrastructure|
| Shader compiler       |
| GPU user-mode driver  |
+-----------------------+
     │
     │ GPU commands / ISA
     ▼
+-----------------------+
| Linux Kernel Driver   |
|      (KMD)            |
+-----------------------+
     │
     ▼
+-----------------------+
|         GPU           |
| Shader / Texture /    |
| Raster / RT / Cache   |
+-----------------------+
```

## Mesa 和 API 的区别
API 是“规范/接口”  
例如 Vulkan：  
```
vkCreateBuffer(...)
vkCmdDraw(...)
vkCmdDispatch(...)
vkQueueSubmit(...)
```
它规定： Application 可以调用什么，以及这些 API 应该表现出什么语义。  
Mesa 是 API 的一种实现
例如：
```
Application
    │
    │ vkCmdDraw()
    ▼
Vulkan API
    │
    ▼
Mesa RADV
    │
    ▼
AMD GPU commands
```
RADV 就是 Mesa 中的 Vulkan driver，实现 Vulkan API 并把 API 操作转换成 AMD GPU 可以执行的命令。  
所以：  
Vulkan = specification/API  
RADV = Vulkan implementation  
Mesa = 包含 RADV 的整个开源 graphics stack  

## Mesa 和 Driver 的区别
Mesa 本身不是一个单独的 GPU driver。Mesa 里面包含很多 driver。  
```
                    Mesa
                      │
       ┌──────────────┼──────────────┐
       │              │              │
     RADV           Iris           NVK
       │              │              │
     AMD GPU       Intel GPU      NVIDIA GPU
```
另外现代 GPU stack 通常分成：
```
User Space
────────────────────────────

Application
     │
Vulkan / OpenGL
     │
Mesa UMD
     │
     │
────────────────────────────
Kernel Space
     │
Linux GPU KMD
     │
     ▼
    GPU
```
例如 AMD：
```
Application
     │
 Vulkan
     ▼
   RADV
     │
     ▼
 AMDGPU kernel driver
     │
     ▼
    AMD GPU
```
RADV 负责很多：
- command generation
- shader compilation
- GPU resource management
- pipeline setup
- GPU-specific programming

而 Linux kernel driver 更多负责：
- GPU scheduling
- memory management
- power management
- display
- command submission

## Mesa和操作系统
Linux：Mesa 最重要的平台  
Mesa 主要是在 Linux 上发展起来的，也是目前最重要的平台。  
```
Linux Application
       ↓
 Vulkan / OpenGL
       ↓
     Mesa
       ↓
   DRM / KMD
       ↓
      GPU
```

Windows: Mesa 不是主流 graphics stack。  

## Mesa 实际有什么用
用途 1：让 Linux 上的 GPU 跑 OpenGL/Vulkan  

用途 2：GPU driver development  
如果公司开发一个 GPU：
```
New GPU architecture
        ↓
New Mesa driver
        ↓
Linux
        ↓
Run real applications
```
Mesa 就可以成为非常重要的 software stack。  

用途 3：Software rendering
```
OpenGL
   ↓
Mesa
   ↓
LLVMpipe
   ↓
CPU
```
也就是说： 没有 GPU，也可以跑 graphics application。  
Mesa 官方把 LLVMpipe 描述为基于 LLVM JIT 的高性能 software renderer。  

用途 4：API translation
```
OpenGL
   ↓
Zink
   ↓
Vulkan
   ↓
GPU driver
```
Zink 本身不直接针对某一个 GPU，而是产生 Vulkan API calls  
```
OpenGL application
       ↓
      Zink
       ↓
     Vulkan
       ↓
     RADV
       ↓
    AMD GPU
```

## Mesa 对 GPU Model 开发有什么用
假设你正在设计一个新 GPU：
```
              New GPU
                 │
        ┌────────┼────────┐
        │        │        │
      Shader    Cache    Memory
      Core
```
你现在只有：
```
GPU Model / Simulator
```
但是你想知道： “真实 Vulkan application 到底会给我的 GPU 发什么东西？”
```
Real Application
       │
       │ Vulkan
       ▼
     Mesa
       │
       │ GPU commands
       ▼
   GPU Model
       │
       ▼
 Simulated GPU
```
这非常有价值。因为你可以：
- 用真实 workload,而不是自己手写draw/dispatch  

- 测试新的 GPU architecture
例如你设计：
```
L1 = 32 KB
L2 = 2 MB
Memory latency = 200 cycles
```
然后让真实 application 跑到 model 上：
```
Vulkan
 ↓
Mesa
 ↓
GPU Model
 ↓
Cache / Memory simulation
```
你就可以研究：
```
IPC
cache hit rate
memory bandwidth
shader utilization
stall cycles
occupancy
latency hiding
```

Mesa + GPU Model 最有价值的地方:
Mesa 提供 workload 和真实 software behavior。  
GPU Model 提供 hardware behavior。  
两者结合：  
可以在真实 graphics workload 上研究未来 GPU architecture。  





