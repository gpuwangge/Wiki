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
运行deqp-vk.exe或deqp-vksc.exe，结果写在当前目录下的TestResults.qpa里。  
运行参数：  
--deqp-caselist-file=<vulkancts>/external/vulkancts/mustpass/main/vk-default.txt (or vksc-default.txt for Vulkan SC implementations)  
--deqp-log-images=disable  
--deqp-log-shader-sources=disable  
举例:  
> deqp-vk --deqp-caselist-file=..\\..\\..\\..\..\..\external\vulkancts\mustpass\main\vk-default.txt --deqp-log-images=disable --deqp-log-shader-sources=disable


或  
> deqp-vksc --deqp-caselist-file=..\..\..\..\..\..\external\vulkancts\mustpass\main\vksc-default.txt --deqp-log-images=disable --deqp-log-shader-sources=disable

Issue: deqp-vksc只有begin session，没有end session  


# Build for Linux 64-bit Debug  
以下测试是在windows下
- Download source  
> git clone --recurse-submodules https://github.com/KhronosGroup/VK-GL-CTS.git  
> python3 external/fetch_sources.py  
> python3 -m pip install lxml  

> mkdir build  
> cd build  

> cmake <path to vulkancts> -DCMAKE_BUILD_TYPE=Debug -DCMAKE_C_FLAGS=-m64 -DCMAKE_CXX_FLAGS=-m64  
> make -j

试验  
cmake -G "MinGW Makefiles" -DCMAKE_BUILD_TYPE=Debug -DCMAKE_C_FLAGS=-m64 -DCMAKE_CXX_FLAGS=-m64 ..   
(error: glslang, mutex...)    

## Execute


# Reference
https://github.com/KhronosGroup/VK-GL-CTS  
https://github.com/KhronosGroup/VK-GL-CTS/blob/main/external/vulkancts/README.md  

