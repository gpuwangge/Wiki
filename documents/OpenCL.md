# OpenCL

## 背景知识
- OpenCL is a programming interface that allows users to take advantage of all the system resources  
- OpenCL is device agnostic （设备不可知）  
- Device are CPUs and GPUs, Multimedia chip, FPGA, Embedded processors  
- Data parallel computing: SIMD  
- OpenCL 和 OpenGL可以联动，特别是如果OpenCL使用GPU的时候可以incur little to no performance hit  
- 对比CUDA: CUDA不是device agnostic, vendor controlled; CUDA和OpenCL的Kernel长得差不多，只需要minor modification  
- Implementation: Mac OS X 10.6; NVIDIA; AMD; Any computer with a CPU  
- GPU的局限性： system to GPU bandwidth is limited; can't handle errors well; difficult to debug; need data arrangement  

## OpenGL的目标
- Compute devices: CPU, GPU; Device Group  
- Memory objects: Arrays(same as in C, has pointer, r/w on CPU are cached, r/w on GPU are usually not), Images(2D and 3D, elements are not accessed via pointers, data read use texture cache)  
- Executable objects: Compute kernel (a data-parallel function that is executed by the compute object, CPU or GPU), Compute program(A group of compute kernels and functions)  
