# Surface
Vulkan是与平台无关API，使用surface以实现跟窗口系统隔离。  
```
VkSurfaceKHR surface;
```
在创建instance后，选择物理设备之前，应该创建surface。因为物理设备的创建依赖surface。  
在windows下，surface可以通过GLFW创建  
```
glfwCreateWindowSurface(instance->getHandle(), window, nullptr, &surface);
```
Android下可以通过如下方式创建  
```
vkCreateAndroidSurfaceKHR(sample.instance->getHandle(), &create_info,nullptr /* pAllocator */, &sample.surface)
```
程序运行结束之前，需要释放surface资源  
```
vkDestroySurfaceKHR(instance->getHandle(), surface, nullptr);
```

# Swapchain
在解释Swapchain概念之前，首先阐述Vulkan是如何处理图像的。  
**`VkImage`**：包含真正存储图像数据的内存结构VkDeviceMemory。  
**`VkImageView`**: 定义了这一段VkImage的那一部分将被使用。  
如果打个比方，VkImage是一幅画，VkImageView是一个画框。你可以通过画框选择这幅画的呈现方式，但又不用改变画本身。  
每一个图片都必须先配一个画框，否则无法使用。  

Swapchain本身就是把一串VkImageView(以及相关联的VkImage)串起来。  
这些VkImageView/VkImage跟API使用者自己创建的VkImageView/VkImage本质上是一个东西。  
只不过Swapchain里面的image大小跟窗口大小一致。并且这些资源在创建完成后，使用权就交到Driver手里了。  
Driver负责查询哪一组VkImageView/VkImage处于空闲(可以被API使用者借用)，并在一定时间内把这个资源呈现出来。  

另外，通常我们不是仅仅想把一幅画显示出来，我们还需要对其做各种处理，这里就涉及到多幅画的融合。  
首先介绍如下概念：  
**`Attachment(附件)`**: 作为图像输出容器，比如Color Attachment, Depth/Stencil Attachment。每个Attachment都要绑定一个VkImageView。  
**`Framebuffer(帧缓冲)`**：很多Attachments组合就成了Framebuffer。  

最简单的情况，需要把一张用户定义的图片画在窗口上。先准备一张窗口大小的容器(Color Attachment)，把画挪到容器里的某个位置。
然后创建Framebuffer,里面当然只有这一个容器(Color Attachment)。  
将Framebuffer交给Driver，看看哪个swapchain image/view是空闲的，挂上去就行了。  

实际情况比这个复杂一些，比如用户这时候掏出两张图片，这其中就存在遮挡关系，后挪进容器的图片就会遮挡住第一张图片。  
显然我们不希望发生这样的结果，我们希望离镜头近的图片遮挡住离镜头远的图片。  
根据图片与镜头的距离，我们创建一张与窗口大小相同的深度图，并且也把它放进一个容器里(Depth Attachment)。  
现在，我们的Framebuffer里就绑定了两个容器(or 附件Attachment)。  
Driver在把Framebuffer挂在Swapchain的时候，就能够正确呈现两张图片的遮蔽关系了。  

因为交换链是与窗口系统和显示相关的组件，因此它依赖于surface的属性。  

# RenderPass


# Vulkan Platform结构
## application.cpp
Sample需要实现以下函数：  
```cpp
initialize()  
update()  
recordGraphcisCommandBuffer()  
recordCoputeCommandBuffer()  
postUpdate()
```
这些函数的执行顺序如下：  
```cpp
CApplication::run(){
  create window
  create instance
  create surface
  find physical devices
  pick physical devices
  create swapchain images
  create swapchain image views
  initialize();
  loop{
    UpdateRecordRender(); //这个函数依次做update, record, render这三件事
  }
}

CApplication::initialize(){ //sample also implement this
  create sync objects
}

CApplication::UpdateRecordRender(){
  update()
  switch(renderMode){
    case RENDER_GRAPHICS_Mode
      renderer.WaitForGraphicsFence();
      recordGraphicsCommandBuffer();  //this function is reserved for sample to implement.
      renderer.AquireSwapchainImage(swapchain);
      renderer.SubmitGraphics();
      renderer.PresentSwapchainImage(swapchain); 
    case RENDER_COMPUTE_Mode
      renderer.WaitForComputeFence();
      recordComputeCommandBuffer(); //this function is reserved for sample to implement.
      renderer.SubmitCompute();
    case RENDER_COMPUTE_SWAPCHAIN_Mode
      renderer.WaitForComputeFence();
      recordComputeCommandBuffer();
      renderer.AquireSwapchainImage(swapchain);
      renderer.SubmitCompute();
      renderer.PresentSwapchainImage(swapchain); 
    case RENDER_COMPUTE_GRAPHICS_Mode
      renderer.WaitForComputeFence();
      recordComputeCommandBuffer();
      renderer.WaitForGraphicsFence();
      recordGraphicsCommandBuffer();
      renderer.AquireSwapchainImage(swapchain);
      renderer.SubmitCompute();
      renderer.SubmitGraphics();
      renderer.PresentSwapchainImage(swapchain); 
  }
  postUpdate() //this function is reserved for sample to implement.
  //postUpdate通常作用是在compute单步完成后打印此时的状态。在graphics中因为已经画出了图形，这个地方一般没啥用。
  renderer.Update() //just update currentFrame
}

CApplication::update(){ //sample also implement this
  update main camera
  for each descriptor{
    update MVP uniform buffer
    update VP uniform buffer
  }
}

```


# Vulkan多队列同步机制： Fences and Semaphores
同步的目的是什么：最大化使用CPU和GPU的资源，减少两者等待的时间。  

同步机制有两种  
1. 单队列(queue)内同步(采用barriers, events and subpass dependencies实现)  
2. 多队列同步机制(Fence, Semaphore)

Semaphore：用于同步GPU间的任务。比如有两个Queue都有command batch(同时submit的一些cmd组成一个batch)。  
Queue1 signals Semaphore，并且Queue2 waits on Semaphore。  
结果就是Queue1的command batch执行完毕后，Queue2的command才开始execute。  
Fence: 用于同步GPU-CPU之间的任务。比如如果需要用swap buffer展示下一帧，就可以用fence莱霍智什么时候去做swap操作。  

在Vulkan中，资源读写的同步由app程序员负责。  
Vulkan在record command buffer后，需要调用vkQueueSubmit()函数(分别对graphics and compute queue调用)，这时候GPU才会开始执行传入的命令。  
Vulkan下GPU的执行顺序：  
同时提交的submit，先提交的先执行。  
同一个sumit内，如果有很多个command buffer(存在不同的VkSubmitInfo里),index小的先执行。  
同一个VkSubmitInfo内，如果有很多commands，也是index小的先执行。  
renderpass and subpass?  

## Fence
Fence配合vkQueueSubmit(Fence)使用。这时候Fence处于被设置状态。  
vkResetFence会把Fence恢复。   
vkWaitForFences会在Fence恢复之前阻塞CPU。这个函数可以理解成一个状态查询，它可能返回VK_TIMEOUT或VK_SUCCESS。  

## Semaphore
Semaphore也是配合vkQueueSubmit(VksubmitInfo)使用，只不过它是在VksubmitInfo结构中。  
在VksubmitInfo结构中有三个参数：  
pWaitSemaphores: 指向一些Semaphore。队列的指令会等待这些Semaphre被通知了之后才开始执行。  
pWaitDstStageMask: 表示提交的所有命令在执行到本指针指向的位置时停下。需要等待pWaitSemaphores所指向的所有Semaphore恢复。   
pSignalSemaphores: 此次提交的命令全部接收后，本指针指向的所有Semapore都会被通知。  
首先，我们确定一个Stage，比如VK_PIPELINE_STAGE_COLOR_ATTACHMENT_OUTPUT_BIT，因为这一步是Graphics Queue的几乎是最后一步，到这一步就说明本frame的image render好了。  
然后，pSignalSemaphores指向renderFinishedSemaphores，表示render finish的信号灯变更成绿灯。  
最后，pWaitSemaphores指向的imageAvailableSemaphores收到了通知，表示Presentation Queue的image可以present了。  
以上Semaphore技巧也叫Binary Semaphore。  
另外有Timeline Semaphore技巧，区别是增加了64-bit integer来指示payload。   

## 结论
Fence用于阻塞CPU直到Queue中的命令执行结束(GPU、CPU之前同步)。  
Semaphre用于不同的命令提交之间的同步(GPU、GPU之前同步)。  

## 举例
在simple_triangle这个实例中，使用了2个frame，Semaphore四个，Fence两个  
imageAvailableSemaphore x2 被pWaitSemaphores指向  
renderFinishedSemaphore x2 被pSignalSemaphores指向(两个合起来表示的逻辑就是render完成后，通知image可用了。)  
inFlightFence x2 赋值是VK_PIPELINE_STAGE_COLOR_ATTACHMENT_OUTPUT_BIT  
在drawFrame()函数中的分配：   
```vulkan
VkResult result = vkAcquireNextImageKHR(logicalDevice, swapChain, UINT64_MAX, imageAvailableSemaphores[currentFrame], VK_NULL_HANDLE, &imageIndex);  
VkSemaphore waitSemaphores[] = { imageAvailableSemaphores[currentFrame] };  
VkPipelineStageFlags waitStages[] = { VK_PIPELINE_STAGE_COLOR_ATTACHMENT_OUTPUT_BIT };  
submitInfo.waitSemaphoreCount = 1;  
submitInfo.pWaitSemaphores = waitSemaphores;  
submitInfo.pWaitDstStageMask = waitStages;
```
和  
```vulkan
VkSemaphore signalSemaphores[] = { renderFinishedSemaphores[currentFrame] };  
submitInfo.signalSemaphoreCount = 1;  
submitInfo.pSignalSemaphores = signalSemaphores;
```
和  
```vulkan
vkWaitForFences(logicalDevice, 1, &inFlightFences[currentFrame], VK_TRUE, UINT64_MAX);  
vkResetFences(logicalDevice, 1, &inFlightFences[currentFrame]);
```
解读：  
对于Fence  
一开始vkWaitForFences(fence)试图把CPU挡住，但这时候fence没设置，所以挡不住。  
然后vkResetFence(Fence)会把Fence恢复。  
最后vkQueueSubmit(Fence)设置Fence。  
下一次运行到drawFrame()的相同frame index的时候，vkWaitForFences(fence)有可能挡住CPU(取决于GPU是否已经执行完毕了上一个command queue里的commands)  
对于Semaphore  
两个Semaphore合起来表示的逻辑就是render完成后，通知image可用了。  
vkAcquireNextImageKHR()的执行依赖image是否可用。  

## Links
https://www.youtube.com/watch?v=GiKbGWI4M-Y  
https://zhuanlan.zhihu.com/p/449222522  


# 如何添加Texture支持
贴图三要素：  
1. 图片Texture Image(需要Texture Image View支持-用来显示图像资源，信息打包进Sampler Descriptor)  
2. 采样方法Sampler(需要Sampler Descriptor支持)  
3. UV坐标TexCoord(需要改输入数据和Shaders)  

# 如何添加深度测试
Setup阶段：  
1. 创建render pass的时候，同时创建depth attachment。同时设定是否early-Z(在PS之前进行一次深度测试)  
2. 创建深度图createDepthResources()：depthImageBuffer and depthImageView  
3. 每个swapChainFramebuffers都需要指定views,把depthImageView添加上去   
4. 创建graphics pipeline的时候，添加pipelineInfo.pDepthStencilState组件

Run阶段  
1. begin renderpass(recordCommandBuffer())的时候，设定renderPassInfo的时候，设置depthStencil  

# 如何启用MSAA
1. 确认Sample Count  
(more samples lead to better results, however it is also more computationally expensive)  
定义  
VkSampleCountFlagBits msaaSamples = VK_SAMPLE_COUNT_1_BIT;  
简单起见，选择使用(硬件支持的)最大available的sample count：getMaxUsableSampleCount()  
(大部分GPU支持至少8 samples)  
VK_SAMPLE_COUNT_1_BIT相当于没开MSAA  

2. 设置Render Target (RT就是一个image buffer)  
原本的image buffer一个像素只有一个sample，所以需要额外的color buffer。  
MyImageBuffer colorImageBuffer_msaa;  
VkImageView colorImageView_msaa;  
使用以下两个函数分别新的buffer：  
createColorResources();  
MSAA必须开DepthBuffer。  
因为在model depthTest 也用到，因此不用重复  

3. 增添Attachment(colorAttachmentResolve到RenderPass)  
并设置color/depth attachment的msaaSamples数量  

4. createFramebuffers的时候swapChainFramebuffers带上colorImageView  

5. GraphicsPipeline创建的时候设置msaaSamples数量  


## Links
https://learnopengl-cn.github.io/04%20Advanced%20OpenGL/01%20Depth%20testing/  
https://developer.aliyun.com/article/325636  

# Todo  
multi object sample  
ray tracing    
android screen flicking(Mipmap, furMark)  
android controller   
resize window   
mesh shader


