
# Setup (Windows, Visual Studio 2022)
> git clone --recurse-submodules  https://github.com/KhronosGroup/VK-GL-CTS.git  

python external/fetch_sources.py  
(会下载spirv-tools/src/)  

# Build
```
mkdir build  
cd build
cmake ..
```

# Use
使用Visual Studio 2022打开dEQP-Core-default.sln  

Build(debug)  
Build: 205 succeed  
Binary位置：有很多个位置，比如：  
VK-GL-CTS/build/external/vulkancts/modules/vulkan/Debug/deqp-vk.exe  
运行deqp-vk.exe，结果写在TestResults.qpa里。  

# Reference
https://github.com/KhronosGroup/VK-GL-CTS
