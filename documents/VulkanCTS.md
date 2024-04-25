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

(会下载spirv-tools/src/ 之类的tools)  

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
> deqp-vk --deqp-caselist-file=..\\..\\..\\..\\..\\..\external\vulkancts\mustpass\main\vk-default.txt --deqp-log-images=disable --deqp-log-shader-sources=disable


或  
> deqp-vksc --deqp-caselist-file=..\\..\\..\\..\\..\\..\external\vulkancts\mustpass\main\vksc-default.txt --deqp-log-images=disable --deqp-log-shader-sources=disable

Issue: deqp-vksc只有begin session，没有end session  


# Build for Linux 64-bit Debug  
以下测试是在windows下交叉编译,未进行WSL或Linux测试  

- Download source  
> git clone --recurse-submodules https://github.com/KhronosGroup/VK-GL-CTS.git

(2.92GB, 7663 Files)   

> cd VK-GL-CTS
> python3 external/fetch_sources.py  

(3.40GB, 15983 Files)  

> python3 -m pip install lxml  

> mkdir build  
> cd build  

> cmake <path to vulkancts> -DCMAKE_BUILD_TYPE=Debug -DCMAKE_C_FLAGS=-m64 -DCMAKE_CXX_FLAGS=-m64  
> make -j

试验  
cmake -G "MinGW Makefiles" -DCMAKE_BUILD_TYPE=Debug -DCMAKE_C_FLAGS=-m64 -DCMAKE_CXX_FLAGS=-m64 ..   
(error: glslang, mutex...)    

## Execute


# Test Lists  
## vk-default.txt  
```
vk-default/api.txt
vk-default/binding-model.txt
vk-default/clipping.txt
vk-default/compute.txt
vk-default/conditional-rendering.txt
vk-default/depth.txt
vk-default/descriptor-indexing.txt
vk-default/device-group.txt
vk-default/dgc.txt
vk-default/draw.txt
vk-default/drm-format-modifiers.txt
vk-default/dynamic-rendering.txt
vk-default/dynamic-state.txt
vk-default/fragment-operations.txt
vk-default/fragment-shader-interlock.txt
vk-default/fragment-shading-barycentric.txt
vk-default/fragment-shading-rate.txt
vk-default/geometry.txt
vk-default/glsl.txt
vk-default/graphicsfuzz.txt
vk-default/image/astc-decode-mode.txt
vk-default/image/atomic-operations.txt
vk-default/image/depth-stencil-descriptor.txt
vk-default/image/extend-operands-spirv1p4.txt
vk-default/image/extended-usage-bit.txt
vk-default/image/extended-usage-bit-compatibility.txt
vk-default/image/format-reinterpret.txt
vk-default/image/host-image-copy.txt
vk-default/image/image-size.txt
vk-default/image/load-store.txt
vk-default/image/load-store-lod.txt
vk-default/image/load-store-multisample.txt
vk-default/image/misaligned-cube.txt
vk-default/image/mismatched-formats.txt
vk-default/image/mismatched-write-op.txt
vk-default/image/mutable.txt
vk-default/image/nontemporal-operand.txt
vk-default/image/qualifiers.txt
vk-default/image/queue-transfer.txt
vk-default/image/sample-cubemap.txt
vk-default/image/sample-texture.txt
vk-default/image/store.txt
vk-default/image/subresource-layout.txt
vk-default/image/swapchain-mutable.txt
vk-default/image/texel-view-compatible.txt
vk-default/imageless-framebuffer.txt
vk-default/info.txt
vk-default/memory.txt
vk-default/memory-model.txt
vk-default/mesh-shader.txt
vk-default/multiview.txt
vk-default/pipeline/fast-linked-library.txt
vk-default/pipeline/monolithic.txt
vk-default/pipeline/pipeline-library.txt
vk-default/pipeline/shader-object-linked-binary.txt
vk-default/pipeline/shader-object-linked-spirv.txt
vk-default/pipeline/shader-object-unlinked-binary.txt
vk-default/pipeline/shader-object-unlinked-spirv.txt
vk-default/protected-memory.txt
vk-default/query-pool.txt
vk-default/rasterization.txt
vk-default/ray-query.txt
vk-default/ray-tracing-pipeline.txt
vk-default/reconvergence.txt
vk-default/renderpass.txt
vk-default/renderpass2.txt
vk-default/robustness.txt
vk-default/shader-object/api.txt
vk-default/shader-object/binary.txt
vk-default/shader-object/binding.txt
vk-default/shader-object/create.txt
vk-default/shader-object/link.txt
vk-default/shader-object/misc.txt
vk-default/shader-object/pipeline-interaction.txt
vk-default/shader-object/rendering.txt
vk-default/sparse-resources.txt
vk-default/spirv-assembly.txt
vk-default/ssbo.txt
vk-default/subgroups.txt
vk-default/synchronization.txt
vk-default/synchronization2.txt
vk-default/tessellation.txt
vk-default/texture.txt
vk-default/transform-feedback.txt
vk-default/ubo.txt
vk-default/video.txt
vk-default/wsi.txt
vk-default/ycbcr.txt
```

## vksc-default.txt  
Vulkan SC = Vulkan Safety Critical  
Vulkan SC 简化了开放的跨平台 Vulkan API ，用于确定性 GPU 图形和计算，在安全认证和实时嵌入式平台上实现高级应用程序和用例。
```
vksc-default/api.txt
vksc-default/binding-model.txt
vksc-default/clipping.txt
vksc-default/compute.txt
vksc-default/descriptor-indexing.txt
vksc-default/device-group.txt
vksc-default/draw.txt
vksc-default/dynamic-state.txt
vksc-default/fragment-operations.txt
vksc-default/fragment-shader-interlock.txt
vksc-default/fragment-shading-rate.txt
vksc-default/geometry.txt
vksc-default/glsl.txt
vksc-default/image/astc-decode-mode.txt
vksc-default/image/atomic-operations.txt
vksc-default/image/depth-stencil-descriptor.txt
vksc-default/image/extend-operands-spirv1p4.txt
vksc-default/image/extended-usage-bit.txt
vksc-default/image/extended-usage-bit-compatibility.txt
vksc-default/image/format-reinterpret.txt
vksc-default/image/image-size.txt
vksc-default/image/load-store.txt
vksc-default/image/load-store-lod.txt
vksc-default/image/load-store-multisample.txt
vksc-default/image/misaligned-cube.txt
vksc-default/image/mismatched-formats.txt
vksc-default/image/mismatched-write-op.txt
vksc-default/image/mutable.txt
vksc-default/image/qualifiers.txt
vksc-default/image/queue-transfer.txt
vksc-default/image/sample-cubemap.txt
vksc-default/image/sample-texture.txt
vksc-default/image/store.txt
vksc-default/image/subresource-layout.txt
vksc-default/image/swapchain-mutable.txt
vksc-default/image/texel-view-compatible.txt
vksc-default/imageless-framebuffer.txt
vksc-default/info.txt
vksc-default/memory.txt
vksc-default/memory-model.txt
vksc-default/multiview.txt
vksc-default/pipeline/monolithic.txt
vksc-default/protected-memory.txt
vksc-default/query-pool.txt
vksc-default/rasterization.txt
vksc-default/renderpass.txt
vksc-default/renderpass2.txt
vksc-default/robustness.txt
vksc-default/sc.txt
vksc-default/spirv-assembly.txt
vksc-default/ssbo.txt
vksc-default/subgroups.txt
vksc-default/synchronization.txt
vksc-default/synchronization2.txt
vksc-default/tessellation.txt
vksc-default/texture.txt
vksc-default/ubo.txt
vksc-default/ycbcr.txt
```


# Reference
https://github.com/KhronosGroup/VK-GL-CTS  
https://github.com/KhronosGroup/VK-GL-CTS/blob/main/external/vulkancts/README.md  
https://www.khronos.org/blog/vulkan-sc-overview/  
 

