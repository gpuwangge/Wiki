# Vulkan Compute Shader

## 基础知识
Workgroup的定义: 三维数组  
比如：3x4x6=72个work item，都可以并行执行  
但能并行多少个，要看硬件情况(通过硬件查询指令)  
可以往gpu传入不同的workgroup,比如w1,w2,w3。它们是不能并行的。有可能先执行w1,也可能w2或w3。  

## Device端代码
以下是Device(GPU)代码里面workgroup维度(size)的接口(这是compute shader专有写法，省略了变量名字)  
> layout (local_size_x = 4, local_size_y = 1, local_size_z = 1) in; 

## Host 端代码
以下Host代码定义可读写buffer(storage buffer)  
如果需要只读的，可以加上readonly关键字  
```glsl
layout (set = 0, binding = 0) buffer Buffer{  
     uint value[4];
} buf;
```
注意在host端，createDevice的时候需要找支持Compute的queueFamily  
> VkQueueFlagBits requiredQueueFamilies = VK_QUEUE_Compute_BIT;  

另外，storage buffer是一个uniform，需要创建descripter  
注意的是，storage buffer跟sampler一样，poolsize type里面都不含uniform关键字， descriptor layout里面也不含uniform关键字  

Host渲染阶段： 
**`1.首先创建command buffer`**  
**`2.gpu-cpu 同步(Fence)`**  
**`3.录制命令及Dispach`**  
```
record command buffer()
    beginCmd
    bindPipeline
    bindDescriptor
    dispatch
    endCmd
dispatch
```
**`4.Barrior同步`**  
之后等待运算完毕  
需要barrior同步，或者device.waitIdle()强行等待  
**`5.验证(可选)`**  
把数据从device写回来：  
> unsigned int data[4] = {0}  
> memcpy(data, storageBuffer->map, sizeof(data));  


## Reference
https://www.khronos.org/opengl/wiki/Compute_Shader
https://www.bilibili.com/video/BV1PD4y1N7N2/?spm_id_from=333.788&vd_source=e9d9bc8892014008f20c4e4027b98036  

