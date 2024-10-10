# OpenCL现状
- OpenCL是Kronos Group负责维护的：
> https://www.khronos.org/opencl/

- 2020年4月27日，Khronos Group 发布 OpenCL 3.0
- 现阶段谷歌排斥OpenCL、苹果也基本放弃OpenCL、微软也不可能去支持OpenCL。所以做操作系统的几家主流公司对OpenCL都不友好。
- OpenCL主要用于GPU的加速，目前也就看芯片公司的支持力度了，arm，qualcomm，amd，intel这几家对OpenCL的支持目前还是不错的，nVidia为了支持自家的cuda，目前支持的OpenCL（windows版本的）才到1.2。amd最近也在搞自家的软件生态，叫做rocm，对OpenCL也是一个挑战


# 背景知识
- OpenCL is a programming interface that allows users to take advantage of all the system resources  
- OpenCL is device agnostic （设备不可知）  
- Device are CPUs and GPUs, Multimedia chip, FPGA, Embedded processors  
- Data parallel computing: SIMD  
- OpenCL 和 OpenGL可以联动，特别是如果OpenCL使用GPU的时候可以incur little to no performance hit  
- 对比CUDA: CUDA不是device agnostic, vendor controlled; CUDA和OpenCL的Kernel长得差不多，只需要minor modification  
- Implementation: Mac OS X 10.6; NVIDIA; AMD; Any computer with a CPU  
- GPU的局限性： system to GPU bandwidth is limited; can't handle errors well; difficult to debug; need data arrangement  

# OpenCL的目标
- Compute devices: CPU, GPU; Device Group  
- Memory objects: Arrays(same as in C, has pointer, r/w on CPU are cached, r/w on GPU are usually not), Images(2D and 3D, elements are not accessed via pointers, data read use texture cache)  
- Executable objects: Compute kernel (a data-parallel function that is executed by the compute object, CPU or GPU), Compute program(A group of compute kernels and functions)  

# OpenCL基本概念
- work-item: a unit of work (等价于CUDA thread)  
- work-group: a grouped work items (等价于CUDA thread block)  
- NDRange Size: Global Size (global_id)  
- Work Group Size: Local Size (local_id) (In GPU, divide evenly; In CPU, always=1)  

|<---NDRange Size Gx---->|  
[][][][][][][][][][][][][][][][][][][][]  
|<--WG Sx--><--WG Sx-->|  

- GPU each instance of a kernel executing(work-item) is run as its own thread
- GPU can host thousands of threads
- Threads on GPU are extremely lightweight and are managed in hardware


# OpenCL Address Spaces
- 一个Compute Unit(就是一个work-group)里面有很多Thread。
- __private (CUDA local): Private Memory在Compute Unit里面，速度最快。每个thread有单独的private memory
- __local (CUDA shared): Local Memory 离 Compute Unit比较近，速度较快。同一个work-group的thread共享local memory。
- __constant (CUDA constant): 是个global cache, texture cache等，速度稍快
- __global (CUDA global): Compute Device Memory 通常说的内存就是指这个，速度中等

# OpenCL 运算的五个阶段
**`1 Initialization`**  
**`2 Allocate resources`**  
**`3 Creating programs/kernels`**  
**`4 Execution`**  
**`5 Tear down(release memory)`**  


# 对比OpenCL和CUDA
- 从很多方面来看，CUDA和OpenCL的关系都和DirectX与OpenGL的关系很相像。如同DirectX和OpenGL一样，CUDA和OpenCL中，前者是配备完整工具包、针对单一供应商(NVIDIA)的成熟的开发平台，后者是一个开放的标准。
- 虽然两者抱着相同的目标：通用并行计算。但是CUDA仅仅能够在NVIDIA的GPU硬件上运行，而OpenCL的目标是面向任何一种Massively Parallel Processor，期望能够对不同种类的硬件给出一个相同的编程模型。

# 对比OpenCL和Computer Shader
- OpenGL Compute Shader和OpenCL都是用于GPGPU计算（通用GPU计算）的API。它们都允许开发者利用GPU进行高性能计算。
- Compute Shader提供了在OpenGL/Vulkan中使用的通用计算方法。
- OpenCL本身就是直接用，不需要其他图形API。
- 总体来说，如果您需要在GPU上进行相对简单的计算，而且需要将计算结果直接渲染到屏幕上，那么使用OpenGL Compute Shader是不错的选择。如果您需要处理更加通用的计算任务，且需要在多个异构计算设备上运行，那么使用OpenCL会更加适合。但是，具体选择哪个API还需要根据具体应用场景来决定。

# 结论
- 桌面GPU计算：CUDA （仅N卡可用）
- 桌面或移动GPU计算: OpenCL
- 桌面或移动GPU计算、图形: OpenGL/Vulkan (Compute Shader)




