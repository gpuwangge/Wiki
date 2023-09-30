# Vulkan Compute Shader

## 基础知识
Workgroup的定义: 三维数组  
比如：3x4x6=72个work item，都可以并行执行  
但能并行多少个，要看硬件情况(通过硬件查询指令)  
可以往gpu传入不同的workgroup,比如w1,w2,w3。它们是不能并行的。有可能先执行w1,也可能w2或w3。  

## Device端代码
以下是Device(GPU)代码里面workgroup维度(size)的接口(这是compute shader专有写法，省略了变量名字)  
> layout (local_size_x = 4, local_size_y = 1, local_size_z = 1) in; 

## Host端代码
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

### Host渲染阶段   
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

## Device内存类型
在解释Host和Device之间数据交换之前，首先需要了解Device内存类型。

## Host和Device的数据交换
Host和Device的数据交换的介质是Storage Buffer。这是GPU可读写的一块内存空间。  
首先建立Storage Buffer(类型为Host Visible and Host Coherent)。  
同时声明一个mapped变量，并不需要为这个变量初始化内存，而是把它映射到Storage Buffer上
> vkMapMemory(CContext::GetHandle().GetLogicalDevice(), storageBuffers[i].deviceMemory, 0, storageBufferSize, 0, &storageBuffersMapped[i]);

这个mapped变量的用处是在Host上可以用memcpy()来拷贝。  
当mapped变量作为destination的时候，可以把host上的数据拷贝到Device的Storage Buffer上。  
> memcpy(storageBuffersMapped[currentFrame], &storageBufferObject, sizeof(storageBufferObject));

当mapped变量作为source的时候，可以把Storage Buffer的数据从Device拷贝到Host上。  
> memcpy(data, descriptor.storageBuffersMapped[renderer.currentFrame], sizeof(data));

如果对mapped memory的操作完毕后，可以通过unmap()来解除映射  
> vkUnmapMemory(CContext::GetHandle().GetLogicalDevice(), IN deviceMemory);

但事实上，mapped并不是真正的内存空间，即使不解除映射也可以。  



## Reference
https://www.khronos.org/opengl/wiki/Compute_Shader
https://www.bilibili.com/video/BV1PD4y1N7N2/?spm_id_from=333.788&vd_source=e9d9bc8892014008f20c4e4027b98036  

