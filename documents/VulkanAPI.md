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
**`VkImageView`**: 定义了这一段VkImage的哪一部分将被使用。  
如果打个比方，VkImage是一幅画，VkImageView是一个画框。你可以通过画框选择这幅画的呈现方式，但又不用改变画本身。  
每一个图片都必须先配一个画框，否则无法使用。  

Swapchain本身就是把一串VkImageView(以及相关联的VkImage)串起来。  
这些VkImageView/VkImage跟API使用者自己创建的VkImageView/VkImage本质上是一个东西。  
只不过Swapchain里面的image大小跟窗口大小一致。并且这些资源在创建完成后，使用权就交到Driver手里了。  
Driver负责查询哪一组VkImageView/VkImage处于空闲(可以被API使用者借用)，并在一定时间内把这个资源呈现出来。  

另外，通常我们不是仅仅想把一幅画显示出来，我们还需要对其做各种处理，这里就涉及到多幅画的融合。  
首先介绍如下概念：  
**`Attachment(附件)`**: 作为图像输出容器，比如Color Attachment, Depth/Stencil Attachment。每个Attachment都要绑定一个VkImageView。每个Attachment可以看作一种资源的描述。   
**`Framebuffer(帧缓存)`**: 把一些attachment资源，以及对应的描述渲染过程的RenderPass包在一起，让渲染器可以在渲染的时候使用的结构。  

Swapchain除了要创建本身的images和views，还要给这些attachment分别创建image和view。  

举例来说，一个最简单的情况，只需要把一张用户定义的图片画在窗口上。先准备一张窗口大小的容器(Color Attachment)，把画挪到容器里的某个位置。
然后创建RenderPass(稍后会解释，这是一个描述渲染过程的结构),里面当然只有这一个容器(Color Attachment)。   
Swapchain需要把RenderPass，以及对应的attachment资源打包成framebuffer。每个swapchain image/view对应一份framebuffer资源。  
渲染器渲染的时候会查看哪一个swapchain image当前available，如果确定了之后会获得swapchain id，于是就可以向swapchain索取相应id对应的framebuffer,从而开始渲染过程。  

实际情况比这个复杂一些，比如用户掏出两张图片，这其中就存在遮挡关系，后挪进容器的图片就会遮挡住第一张图片。  
显然我们不希望发生这样的结果，我们希望离镜头近的图片遮挡住离镜头远的图片。  
根据图片与镜头的距离，我们创建一张与窗口大小相同的深度图，并且也把它放进一个容器里(Depth Attachment)。  
现在，我们的RenderPass里就绑定了两个容器(or 附件Attachment)，同样把两者打包进一个framebuffer，这样渲染的时候就能够正确呈现两张图片的遮蔽关系了。  

因为交换链是与窗口系统和显示相关的组件，因此它依赖于surface的属性。  
因此，在创建了surface之后，我们可以立刻设置swapchain images/imageviews。尽管这时候还没有任何attachment/framebuffer资源。  

# Renderer
Renderer在Vulkan Platform里进行各种渲染的准备工作，包括准备vertex buffer，command buffer，获得swapchain image id
（就是acquire哪个swapchain image是available的）  
Renderer跟RenderPass的区别是：后者定义了渲染流程，前者使用这个定义好的流程，并且配合其他必要的资源（比如swapchain的framebuffer）进行渲染  
(在Vulkan Platform里，RenderPass和pipelines都在Renderprocess里创建)   
Renderer也处理compute buffer/command的内容，虽然严格来说这不是“渲染”  
Renderer也负责处理同步问题：fence, semaphore  
在Render过程中最重要的Record过程留给Sample实现，但Renderer会做一些前后准备工作，包括但不仅限于：  
- Begin Render Pass(需要renderPass本身以及available的那个swapchain view的framebuffer)  
- Bind Pipeline  
- Set Viewport  
- Set Scissor  
- Bind Descriptor Sets

Vulkan Platform定义了四种渲染模式  
## 1 RENDER_GRAPHICS_Mode
正常的图形渲染，走graphics pipeline  
## 2 RENDER_COMPUTE_Mode
仅计算，走compute pipeline  
## 3 RENDER_COMPUTE_SWAPCHAIN_Mode
主要走compute pipeline,但是最后画在swapchain上  
(Sample: textureCompute)  
这种情况不需要submit graphics command queue  
原理是通过compute shader把数据写到texture buffer上，再让graphics pipeline显示这个texture  
## 4 RENDER_COMPUTE_GRAPHICS_Mode
同时有graphics和compute pipeline，且各自都会提交command queue    
一般来讲会通过compute shader做一些并行计算，然后把结果通过graphics pipeline画出来  

# RenderPass
RenderPass描述了GPU如何进行渲染流程，它定义了渲染步骤和使用的资源。  
RenderPass通过subpass来组织这些资源。  
一个RenderPass里可以有多个subpass。  
每个subpass可以设置输入attachment和输出attachment。  
(这里的attachment其实就是swapchain里的attachment。所不同的是RenderPass里的不是真的attachment资源，仅仅是附件描述attachment description)  
每个subpass都必须有至少一个attachment作为输出。一个subpass的输出attachment可以成为另一个subpass的输入attachment。    
为了简单起见，也可以只建立1个subpass，然后添加多个attachments。  
比如，可以输出一个Color Attachment和一个Depth Attachment。如果开启MSAA，还可以加一个Resolve Attachment。  

因此，subpass需要控制执行的顺序。控制顺序的办法是Subpass Dependency。  
如果只有一个subpass，也只需要一个简单的dependency。  

通过构建不同的RenderPass，Vulkan定义了不同的渲染状态，允许开发者在渲染之前切换状态而不会导致性能下降。  

# Buffer

# Texture



# Shader

# Descriptor

# Pipeline




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
其中，Sample::initilize()要做如下事情：  
```cpp
initialize(){
  set camera
  Load Model
  create vertex buffer for renderer
  create command pool for renderer
  create command buffer for renderer
  load texture, create its image/imageview
  create msaa image/imageview
  create depth image/imageview
  create render pass for renderprocess
  load shaders
  create descriptors
  create pipelines for renderprocess
}
```
Application的执行顺序如下：  
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
Fence用于阻塞CPU直到Queue中的命令执行结束(GPU、CPU之间同步)。  
Semaphre用于不同的命令提交之间的同步(GPU、GPU之间同步)。  

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


