# Introduction
VulkanCTS is Build on dEQP framework.

# DrawElements Quality Program(DEQP)
DEQP is an Android Open Source toolkit Project for benchmarking the accuracy, precision, feature conformance and stability of OpenGL ES and OpenCL GPUs. 

# Requirement
- Git  
- Python 3.x  
- CMake 3.20.0 or newer  


# Build for Windows (Visual Studio 2022)
> git clone --recurse-submodules  https://github.com/KhronosGroup/VK-GL-CTS.git  
> python external/fetch_sources.py  
(会下载spirv-tools/src/)  

## Command
```
mkdir build  
cd build
cmake ..
```

## Execute
使用Visual Studio 2022打开dEQP-Core-default.sln  

Build(debug)  
Build: 205 succeed  
Binary位置：有很多个位置，比如：  
VK-GL-CTS/build/external/vulkancts/modules/vulkan/Debug/deqp-vk.exe  
运行deqp-vk.exe，结果写在当前目录下的TestResults.qpa里。  
运行时间？  


# Build for Linux 64-bit Debug  
未验证
- Download source  
> git clone --recurse-submodules https://github.com/KhronosGroup/VK-GL-CTS.git  
> python3 external/fetch_sources.py  
> python3 -m pip install lxml  

ERROR: Could not install packages due to an OSError: Could not find a suitable TLS CA certificate bundle, invalid path: C:\Users\mtk32132\Downloads\cacertmtk.pem

> cmake <path to vulkancts> -DCMAKE_BUILD_TYPE=Debug -DCMAKE_C_FLAGS=-m64 -DCMAKE_CXX_FLAGS=-m64  
> make -j  

## Execute


# Reference
https://github.com/KhronosGroup/VK-GL-CTS  
https://github.com/KhronosGroup/VK-GL-CTS/blob/main/external/vulkancts/README.md  

