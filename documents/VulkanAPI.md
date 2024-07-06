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

Swapchain本身就是把一串VkImageView(以及相关联的VkImage)资源串起来，以供渲染器调用。  
这些VkImageView/VkImage跟API使用者自己创建的VkImageView/VkImage本质上是一个东西。  
这些image/view资源在其他API里通常叫做RenderTarget。  
Swapchain里面的image大小跟窗口大小一致。并且这些资源在创建完成后，使用权就交到Driver或API开发者手里了。  
Driver/API开发者负责查询哪一组VkImageView/VkImage处于空闲(可以被API使用者借用)，并在一定时间内把这个资源呈现出来。  

另外，通常我们不是仅仅想把一幅画显示出来，我们还需要对其做各种处理，这里就涉及到多幅画的融合。  
首先介绍如下概念：  
**`Attachment(附件)`**: 作为图像输出容器，比如Color Attachment, Depth/Stencil Attachment。每个Attachment都要绑定一个VkImageView。每个Attachment可以看作一种资源的描述。   
Swapchain除了要创建本身的images和views，还要给这些attachment分别创建image和view。  
**`Framebuffer(帧缓存)`**: 把一些attachment资源，以及对应的描述渲染过程的RenderPass包在一起，让渲染器可以在渲染的时候使用的结构。  
因为每一帧画面实际上只用到一个swapchain image，所以也只会用到一份framebuffer。因此swapchain image/view数量跟framebuffer数量也是相同的。  

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

<p float="left">
  <img src="https://github.com/gpuwangge/Wiki/blob/main/images/swapchain.png " alt="alt text" width="600" height="500">  
</p> 
 

(图片取自Learning Vulkan)  
可以看出来, swapchain 一开始创建的是color image和相应的imageview。  
随后如果用到depth image的时候需要手动allocate memory。  
相关创建代码：  
```vulkan
std::vector<VkImage> images;  
result = vkGetSwapchainImagesKHR(CContext::GetHandle().GetLogicalDevice(), handle, &imageSize, nullptr);
images.resize(imageSize);
result = vkGetSwapchainImagesKHR(CContext::GetHandle().GetLogicalDevice(), handle, &imageSize, images.data());
```
```vulkan
VkImageView imageView;
if (vkCreateImageView(CContext::GetHandle().GetLogicalDevice(), &viewInfo, nullptr, &imageView) != VK_SUCCESS) {
  throw std::runtime_error("failed to create texture image view!");
}
```


# Renderer
Renderer在Vulkan Platform里进行各种渲染的准备工作，包括准备vertex buffer，command buffer，获得swapchain image id
（就是acquire哪个swapchain image是available的）  
Renderer跟RenderPass的区别是：后者定义了渲染流程，前者使用这个定义好的流程，并且配合其他必要的资源（比如swapchain的framebuffer）进行渲染  
 
Renderer也处理compute buffer/command的内容，虽然严格来说这不是“渲染”  
Renderer也负责处理同步问题：fence, semaphore  
在Render过程中最重要的Record过程留给Sample实现，但Renderer会做一些前后准备工作，包括但不仅限于：  
- Begin Render Pass(需要renderPass本身以及available的那个swapchain view的framebuffer)  
- Bind Pipeline  
- Set Viewport  
- Set Scissor  
- Bind Descriptor Sets

在一些传统图形API里，渲染器选择哪一个framebuffer资源是由Driver控制的。但在Vulkan API里，driver的一些功能被移到API层，因此图形开发人员可以自由调配。  

Vulkan Platform定义了四种渲染模式  
- RENDER_GRAPHICS_Mode  
正常的图形渲染，走graphics pipeline  
- RENDER_COMPUTE_Mode  
仅计算，走compute pipeline  
- RENDER_COMPUTE_SWAPCHAIN_Mode  
主要走compute pipeline,但是最后画在swapchain上  
(Sample: textureCompute)  
这种情况不需要submit graphics command queue  
原理是通过compute shader把数据写到texture buffer上，再让graphics pipeline显示这个texture  
- RENDER_COMPUTE_GRAPHICS_Mode  
同时有graphics和compute pipeline，且各自都会提交command queue    
一般来讲会通过compute shader做一些并行计算，然后把结果通过graphics pipeline画出来  

# RenderPass
RenderPass描述了GPU如何进行渲染流程，它定义了渲染步骤和使用的资源描述。  
RenderPass通过subpass来组织这些资源。  
一个RenderPass里可以有多个subpass。  
每个subpass可以设置输入attachment和输出attachment。  
(这里的attachment其实就是swapchain里的attachment。所不同的是RenderPass里的不是真的attachment资源，仅仅是附件描述attachment description)  
每个subpass都必须有至少一个attachment作为输出。一个subpass的输出attachment可以成为另一个subpass的输入attachment。  
subpass设计的目的是为了实现TBR/TBDR，除此之外也没啥其他用处。  

为了简单起见，也可以只建立1个subpass，然后添加多个attachments。  
比如，可以输出一个Color Attachment和一个Depth Attachment。如果开启MSAA，还可以加一个Resolve Attachment。  

因此，subpass需要控制执行的顺序。控制顺序的办法是Subpass Dependency。  
如果只有一个subpass，也只需要一个简单的dependency。  

通过构建不同的RenderPass，Vulkan定义了不同的渲染状态，允许开发者在渲染之前切换状态而不会导致性能下降。  

(在Vulkan Platform里，RenderPass和pipelines都在Renderprocess里创建)  

# Shader
值得一提的是，Vulkan Shader Code使用的格式是SPIV-V，它需要使用单独的Shader Compiler编译   
这样做的好处是可以保护shader源代码，毕竟真正交付给用户的是spv文件。  

# Pipeline
Pipeline的本质就是各种shader组合在一起。  
每次走完一遍pipeline，就相当于一次renderPass。   
如果同一帧走了两次同一个pipeline，则说明这个renderPass里包含了两个subpass。(比如延迟着色)  

但是并不是所有的Pipeline都需要renderPass，比如光线追踪。  
如果是使用Compute Pipeline的情况，也不需要renderPass。  

(在Vulkan Platform里，RenderPass和pipelines都在Renderprocess里创建)  

# Vulkan资源概论
Vulkan里的资源分为三种：buffer, image和texture  

## Buffer
Buffer应用广泛。用在创建uniform, storage buffer, vertex data buffer或staging buffer等  
- Vertex Buffer: 专供vertex shader使用  
- Index Buffer: 专供vertex shader使用  
- Uniform Buffer：给所有shader公共的专门用于读取的数据。比如所有顶点都需要的颜色，或者transform matrix  
- Storage Buffer: 提供给compute shader使用的既可读又可写的数据。它的用法跟uniform差不多。区别是Storage Buffer可以提供非常大(~128 Mb)的空间，支持原子(Atomic)操作，支持可变存储(ex: int arr[])。缺点是Storage Buffer的访问会慢一些。  

Storage Buffer使用方法案例(Compute Shader)  
(注意本例中GPU并没有对其进行读写，读写指令通过Host端完成)  
(可以看出Storage Buffer在shader里实际是通过一个普通buffer和readonly buffer实现的)  
需要创建VK_DESCRIPTOR_TYPE_STORAGE_BUFFER类型的Descriptor  
```vulkan
layout (local_size_x = 4, local_size_y = 1, local_size_z = 1) in; //workgroup size
layout(std140, binding = 0) readonly buffer SBOIn {
   vec4 dataIn;
};
layout(std140, binding = 1) buffer SBOOut {
   vec4 dataOut;
};
void main(){
}
```

## Texture
与Buffer相似，texture也持有一段GPU上的内存。并且它还包含一些额外特征：Format, Sample Count, Flags。  
为了将图像映射到几何图形上，几何顶点上必须有纹理坐标(texture coordinate)  
为了跟顶点坐标xyz区分，纹理坐标一般表示为uvw。又因为texture一般是2d的，简写做uv  
另外，texture的尺寸跟屏幕的尺寸一般不一致，为了显示在屏幕上，需要使用采样技术(区别于显示几何所需的光栅化技术)  
既然要采样，就要定义采样器(Sampler): The (Texture) image is used as a descriptor interface and shared at the shader stage (fragment shader) in the form of samplers。
Texture使用方法案例(Fragment Shader)
```vulkan
layout(binding = 1) uniform sampler2D texSampler;
layout(location = 0) in vec3 fragColor;
layout(location = 1) in vec2 fragTexCoord;
layout(location = 0) out vec4 outColor;
void main() {
	outColor = texture(texSampler, fragTexCoord);
}
```

## Image
Image用在创建swapchain,以及创建attachment和texture  
Image与Texture的关系：texture是一类特别的image，为了方便描述，把texture单独列在上面。Image不需要Sampler  
Image与Buffer的关系：可以理解为Image是加了一层额外layer的buffer。因此，image和buffer之间的数据是可以互相拷贝的。  
Image可以脱离Graphics Pipeline，仅跟Compute Shader搭配使用。这时候Image仅作为Compute输入/输出的数据，并不需要sampler和一般image需要的哪些属性。这种Image一般称为Storage Image。这种用法还有个好处就是可以直接挂载在swapchain上，以实现不需要graphics pipeline，仅用compute shader就画出图像的效果。  
Image使用方法案例(Compute Shader)  
需要创建VK_DESCRIPTOR_TYPE_STORAGE_IMAGE类型的Descriptor  
```vulkan
layout (local_size_x = 2, local_size_y = 1, local_size_z = 1) in;
layout (binding = 0, rgba8 ) uniform writeonly image2D resultImage;
void main(){
    imageStore( resultImage,ivec2(gl_GlobalInvocationID.xy),vec4(1.0,0.0,1.0,1.0));
}
```
如果需要可读写的image2D，可以使用如下代码  
```vulkan
layout (binding = 0, rgba8) uniform readonly image2D inputImage; 
layout (binding = 1, rgba8) uniform writeonly image2D outputImage;
vec3 pixel = imageLoad(inputImage, ivec2(gl_GlobalInvocationID.xy)).rgb;
imageStore(outputImage, ivec2(gl_GlobalInvocationID.xy), pixel);
```

# Buffer的创建和使用
在Vulkan里，描述GPU内存的变量有两个：
- VkBuffer buffer: 保存内存的信息  
- VkDeviceMemory deviceMemory: 实际的GPU内存空间  

给buffer分配GPU内存分为如下几步：  
- 确定需要的内存大小size，确定使用的方式VkBufferUsageFlags  
- 根据以上信息，调用vkCreateBuffer()函数获得buffer  
- 调用vkGetBufferMemoryRequirements()函数, 结果被存到vmr(VkMemoryRequirements)里。因为内存对齐的缘故，vmr的size一般比需要的内存更大一些。  
这个过程也叫做fill vmr。  
- 分配真正的内存空间。使用vkAllocateMemory()函数，结果存到deviceMemory里。并且该buffer也绑定到了deviceMemory上。此时deviceMemory里面并没有数据。  
- 把数据从CPU内存数据(data)拷贝到devicememory里。  
首先API是run在CPU里的，并不能直接访问GPU内存，需要使用一些API函数来进行拷贝。  
第一步vkMapMemory()函数可以在deviceMemory上建立一个映射。映射名字叫什么无所谓。我就叫它pGpuMemory，是一个指向GPU内存的指针。  
第二步使用memcpy()函数把data拷贝到pGpuMemory上(就如同我们常常在CPU上的操作一样)  
第三步使用vkUnMapMemory()函数解除deviceMemory上的内存映射。但假如以后还需要从CPU访问这段内存，不解除也是可以的。  
这个过程也叫fill deviceMemory。

至此，完成了内存空间的初始化。  
别忘了在结束程序前，要使用vkDestroyBuffer(buffer)和vkFreeMemory(deviceMemory)清理内存资源。  

# Image Buffer
与GPU Buffer类似，在Vulkan里，描述GPU Image Buffer内存的变量也有两个：  
- VkImage image: 保存Image内存的信息  
- VkDeviceMemory deviceMemory: 实际的GPU Image内存空间  

另外，在创建image buffer的时候，一般也会创建imageview做资源配套  
- VkImageVIew view  

创建和销毁Image Buffer跟Buffer也差不多  
- vkCreateImage()  
- vkGetImageMemoryRequirements()  
- vkAllocateMemory()  
- vkBindImageMemory()  
- vkDestroyImage()  
- vkFreeMemory()  

# Texture的创建和使用
前面说过，texture是一类特殊的image。创建Texture的第一步，就是创建上述的Image Buffer。另外还需要制定一些其他参数，比如texture的长和高，图像格式，mipmap参数等。  
总的来说，texture有三个要素分别为：  
- Image: 保存一些创建memory需要的metadata。
- Image Layout：texture数据通过grid coordinate representation in image memory。根据texture用途的不同(color attachment, sparse texture等)设定不同的layout。  
- Image View: 跟普通的image view是一样的作用。Image View也作为操作texture的接口。  

在vulkan platform的实现里，当调用textureImageBuffer.createImage()的时候就会分配内存。  
```vulkan
CWxjImageBuffer textureImageBuffer;
```
创建image view也可以通过textureImageBuffer.createImageView()  

## 创建texture image的具体过程
### 1.创建Image Buffer
```vulkan
class CWxjImageBuffer final{
public:
	VkImage	image;
	VkDeviceMemory deviceMemory;
	VkDeviceSize size;
	VkImageView view;
}
void CWxjImageBuffer::createImage(...) {
    if (vkCreateImage(CContext::GetHandle().GetLogicalDevice(), &imageInfo, nullptr, &image) != VK_SUCCESS) {
        throw std::runtime_error("failed to create image!");
    }
    if (vkAllocateMemory(CContext::GetHandle().GetLogicalDevice(), &allocInfo, nullptr, &deviceMemory) != VK_SUCCESS) {
        throw std::runtime_error("failed to allocate image memory!");
    }
    vkBindImageMemory(CContext::GetHandle().GetLogicalDevice(), image, deviceMemory, 0);
}

VkImageView CWxjImageBuffer::createImageView(VkImage image, VkFormat format, VkImageAspectFlags aspectFlags, uint32_t mipLevels){
    VkImageView imageView;
    if (vkCreateImageView(CContext::GetHandle().GetLogicalDevice(), &viewInfo, nullptr, &imageView) != VK_SUCCESS) {
        throw std::runtime_error("failed to create texture image view!");
    }
    return imageView;
}
```
### 2.创建Texture
首先按照#1的做法准备好textureImageBuffer(就是上述结构和createImage和createImageView这两个函数)  
然后读取图像资源，一般会放入名字为texels的结构中。这个texel概念跟pixel很像，只不过前者是图片上的基本元素，后者是显示的基本元素。  
第三步就是真正创建textureImage了  
```vulkan
void CTextureImage::CreateTextureImage(void* texels, CWxjImageBuffer &imageBuffer) {
	VkDeviceSize imageSize = texWidth * texHeight * texChannels * texBptpc/8; 

	mipLevels = bEnableMipMap ? (static_cast<uint32_t>(std::floor(std::log2(std::max(texWidth, texHeight)))) + 1) : 1;

	//Step 1: prepare staging buffer with texture pixels inside
	CWxjBuffer stagingBuffer;
	VkResult result = stagingBuffer.init(imageSize, VK_BUFFER_USAGE_TRANSFER_SRC_BIT);
	stagingBuffer.fill(texels);

	stbi_image_free(texels);

	//Step 2: create(allocate) image buffer
	imageBuffer.createImage(texWidth, texHeight, mipLevels, VK_SAMPLE_COUNT_1_BIT, imageFormat, VK_IMAGE_TILING_OPTIMAL, usage, VK_MEMORY_PROPERTY_DEVICE_LOCAL_BIT);

	//Step 3: copy stagingBuffer(pixels) to imageBuffer(empty)
	//To perform the copy, need change imageBuffer's layout: undefined->transferDST->shader-read-only 
	//If the image is mipmap enabled, keep the transferDST layout (it will be mipmaped anyway)
	//(transfer image in non-DST layout is not optimal???)
	if(mipLevels == 1){
		transitionImageLayout(imageBuffer.image, VK_IMAGE_LAYOUT_UNDEFINED, VK_IMAGE_LAYOUT_TRANSFER_DST_OPTIMAL);
 		copyBufferToImage(stagingBuffer.buffer, imageBuffer.image, static_cast<uint32_t>(texWidth), static_cast<uint32_t>(texHeight));
      		transitionImageLayout(imageBuffer.image, VK_IMAGE_LAYOUT_TRANSFER_DST_OPTIMAL, VK_IMAGE_LAYOUT_GENERAL);//VK_IMAGE_LAYOUT_SHADER_READ_ONLY_OPTIMAL ///!!!!
	}else{
		transitionImageLayout(imageBuffer.image, VK_IMAGE_LAYOUT_UNDEFINED, VK_IMAGE_LAYOUT_TRANSFER_DST_OPTIMAL);
		copyBufferToImage(stagingBuffer.buffer, imageBuffer.image, static_cast<uint32_t>(texWidth), static_cast<uint32_t>(texHeight));
	}

	stagingBuffer.DestroyAndFree();
}
```
过程解释  
- stagingBuffer  
在Host/Device系统中，两者有各自的Memory。即使是Mobile平台，只有一块统一的物理memory，Host/Device也独立使用各自的部分。各自也看不见对方的memory。  
在任务运行的过程中，一般需要尽可能使用Device Memory以提高效率。  
这时候的一个问题就是数据常常是在Host段由CPU准备，所以它一开始只存在一块Host可见的memory区域，随后我们才会把它拷贝到Device Memory上。  
这一块仅Host可见，随后会被转移到device的memory区域成为Staging Buffer。  
在本例中，要转移的数据是texels，它被放到stagingBuffer里面，用途设置为TRANSFER_SRC。为接下来转移到Device做好准备。
  
- imageBuffer的Layout  
Layout是image(buffer)的一个属性，代表像素在内存中的组织形式  
在创建image的时候(vkCreateImage)，需要指定initialLayout，一般会指定为VK_IMAGE_LAYOUT_UNDEFINED  
之所以给image设定不同的layout，主要是因应不同的使用需求。使用合适的layout能提升性能  
(具体如何实现这些Layout对于使用者而言是黑盒，即不同的GPU有不同的实现形式)  
比如，一个作为color attachment使用的image，应当设置为VK_IMAGE_LAYOUT_COLOR_ATTACHMENT_OPTIMAL  
如果要作为传输使用，需要设置为VK_IMAGE_LAYOUT_TRANSFER_SRC_OPTIMAL或VK_IMAGE_LAYOUT_TRANSFER_DST_OPTIMAL  
其他常见的Layout还包括VK_IMAGE_LAYOUT_DEPTH_STENCIL_ATTACHMENT_OPTIMAL，VK_IMAGE_LAYOUT_SHADER_READ_ONLY_OPTIMAL等   

- transitionImageLayout函数  
此函数用于Layout Transition，这个动作会改变image的layout，该动作也会对内存进行读写  
回忆当我们CreateImage的时候，需要调用一系列vk开头的函数(vkCreateBuffer, vkGetBufferMemoryRequirements, vkAllocateMemory, vkBindBufferMemory)  
注意这些函数的作用是在device上开辟一块内存区域，然后用memcpy把数据从host拷贝到device memory里  
因此，当进行Layout Transition动作时，实际上是Host命令Device进行转换，这条命令是通过一个command queue来实现的  
再回忆Host和Device是异步工作的，对于Device上资源的操作，需要注意的是要防止资源读取乱序  
Barrier的作用是在同一个command queue里，在一堆command之间立一个barrier，以确立barrier之前的commands肯定比之后的commands先执行  
```vulkan
void CTextureImage::transitionImageLayout(VkImage image, VkImageLayout oldLayout, VkImageLayout newLayout) {
    VkCommandBuffer commandBuffer = beginSingleTimeCommands();

    VkImageMemoryBarrier barrier{};
    barrier.sType = VK_STRUCTURE_TYPE_IMAGE_MEMORY_BARRIER;
    barrier.oldLayout = oldLayout;
    barrier.newLayout = newLayout;
    barrier.srcQueueFamilyIndex = VK_QUEUE_FAMILY_IGNORED;
    barrier.dstQueueFamilyIndex = VK_QUEUE_FAMILY_IGNORED;
    barrier.image = image;
    barrier.subresourceRange.aspectMask = VK_IMAGE_ASPECT_COLOR_BIT;
    barrier.subresourceRange.baseMipLevel = 0;
    barrier.subresourceRange.levelCount = mipLevels;
    barrier.subresourceRange.baseArrayLayer = 0;
    barrier.subresourceRange.layerCount = 1;

    VkPipelineStageFlags sourceStage;
    VkPipelineStageFlags destinationStage;

    if (oldLayout == VK_IMAGE_LAYOUT_UNDEFINED && newLayout == VK_IMAGE_LAYOUT_TRANSFER_DST_OPTIMAL) {
        barrier.srcAccessMask = 0;
        barrier.dstAccessMask = VK_ACCESS_TRANSFER_WRITE_BIT;

        sourceStage = VK_PIPELINE_STAGE_TOP_OF_PIPE_BIT;
        destinationStage = VK_PIPELINE_STAGE_TRANSFER_BIT;
    }
    else if (oldLayout == VK_IMAGE_LAYOUT_TRANSFER_DST_OPTIMAL && newLayout == VK_IMAGE_LAYOUT_SHADER_READ_ONLY_OPTIMAL) {
        barrier.srcAccessMask = VK_ACCESS_TRANSFER_WRITE_BIT;
        barrier.dstAccessMask = VK_ACCESS_SHADER_READ_BIT;

        sourceStage = VK_PIPELINE_STAGE_TRANSFER_BIT;
        destinationStage = VK_PIPELINE_STAGE_FRAGMENT_SHADER_BIT;
    }
	else if (oldLayout == VK_IMAGE_LAYOUT_TRANSFER_DST_OPTIMAL && newLayout == VK_IMAGE_LAYOUT_GENERAL) {///!!!!
        barrier.srcAccessMask = VK_ACCESS_TRANSFER_WRITE_BIT;
        barrier.dstAccessMask = VK_ACCESS_SHADER_READ_BIT;

        sourceStage = VK_PIPELINE_STAGE_TRANSFER_BIT;
        destinationStage = VK_PIPELINE_STAGE_FRAGMENT_SHADER_BIT;
    }
    else {
        throw std::invalid_argument("unsupported layout transition!");
    }

    vkCmdPipelineBarrier(
        commandBuffer,
        sourceStage, destinationStage,
        0,
        0, nullptr,
        0, nullptr,
        1, &barrier
    );

    endSingleTimeCommands(commandBuffer);
}
```


- CopyBufferToImage函数  
该函数的作用就是把stagingBuffer里的数据从Host拷贝到Device Memory上  
同样因为是Host对Device下指令工作，需要使用command queue(但是不需要barrier)  
```vulkan
void CTextureImage::copyBufferToImage(VkBuffer buffer, VkImage image, uint32_t width, uint32_t height) {
    VkCommandBuffer commandBuffer = beginSingleTimeCommands();

    VkBufferImageCopy region{};
    region.bufferOffset = 0;
    region.bufferRowLength = 0;
    region.bufferImageHeight = 0;
    region.imageSubresource.aspectMask = VK_IMAGE_ASPECT_COLOR_BIT;
    region.imageSubresource.mipLevel = 0;
    region.imageSubresource.baseArrayLayer = 0;
    region.imageSubresource.layerCount = 1;
    region.imageOffset = { 0, 0, 0 };
    region.imageExtent = {
        width,
        height,
        1
    };

    vkCmdCopyBufferToImage(commandBuffer, buffer, image, VK_IMAGE_LAYOUT_TRANSFER_DST_OPTIMAL, 1, &region);

    endSingleTimeCommands(commandBuffer);
}
```


- beginSingleTimeCommands/endSingleTimeCommands函数  
第一个函数负责准备command buffer  
第二个函数负责submit queue  
这两个函数之间需要fill command buffer  
```vulkan
VkCommandBuffer CTextureImage::beginSingleTimeCommands() {
    VkCommandBufferAllocateInfo allocInfo{};
    allocInfo.sType = VK_STRUCTURE_TYPE_COMMAND_BUFFER_ALLOCATE_INFO;
    allocInfo.level = VK_COMMAND_BUFFER_LEVEL_PRIMARY;
    allocInfo.commandPool = *pCommandPool;
    allocInfo.commandBufferCount = 1;

    VkCommandBuffer commandBuffer;
    vkAllocateCommandBuffers(CContext::GetHandle().GetLogicalDevice(), &allocInfo, &commandBuffer);

    VkCommandBufferBeginInfo beginInfo{};
    beginInfo.sType = VK_STRUCTURE_TYPE_COMMAND_BUFFER_BEGIN_INFO;
    beginInfo.flags = VK_COMMAND_BUFFER_USAGE_ONE_TIME_SUBMIT_BIT;

    vkBeginCommandBuffer(commandBuffer, &beginInfo);

    return commandBuffer;
}

void CTextureImage::endSingleTimeCommands(VkCommandBuffer commandBuffer) {
    vkEndCommandBuffer(commandBuffer);

    VkSubmitInfo submitInfo{};
    submitInfo.sType = VK_STRUCTURE_TYPE_SUBMIT_INFO;
    submitInfo.commandBufferCount = 1;
    submitInfo.pCommandBuffers = &commandBuffer;

    vkQueueSubmit(CContext::GetHandle().GetGraphicsQueue(), 1, &submitInfo, VK_NULL_HANDLE);
    vkQueueWaitIdle(CContext::GetHandle().GetGraphicsQueue());

    vkFreeCommandBuffers(CContext::GetHandle().GetLogicalDevice(), *pCommandPool, 1, &commandBuffer);
}
```

# Descriptor
Descriptor是一类Shader变量，它用于描述Uniform变量的类型和Layout  
所谓Uniform变量，就是所有shader invocation共同可见的变量。需要为每一个Uniform创建其Descriptor，不然GPU会不知道如何使用  
常见的Uniform变量包括  
- MVP矩阵: 类型为VK_DESCRIPTOR_TYPE_UNIFORM_BUFFER
- Image Sampler: 类型为VK_DESCRIPTOR_TYPE_COMBINED_IMAGE_SAMPLER
- Storage Buffer: 类型为VK_DESCRIPTOR_TYPE_STORAGE_BUFFER
- Texture/Image: 类型都是VK_DESCRIPTOR_TYPE_STORAGE_IMAGE

创建Descriptor分为三个步骤  
- Create Descriptor Pool: Descriptor不能直接创建，它们必须从描述符池中分配(就如同Command的创建必须从命令缓冲池中分配一样)。在这里我们定义我们的程序需要哪些种类的Descriptor和它们的数量。  
- Create Descriptor Set Layout：对于每一个具体的Descritpor，需要指定一些参数，比如类型和这个uniform会在哪个shader里可见  
- Create Descriptor Sets: 在完成了以上两步后，我们有了descriptorPool和descriptorSetLayout，我们就可以据此分配真正的Descriptor了。  
这里面还有些需要注意的细节，比如需要为HOST端每一组资源创建一个单独的descritper set(MAX_FRAMES_IN_FLIGHT)  
再比如还要指出这个uniform所需要的资源(buffer)在哪里(叫什么名字)。  
ex1: 如果使用了MVP Uniform，需要为其allocate memory space，然后把这个信息update到descritor上面  
ex2: 如果使用的是texture，要把texture的imageView(不是image)挂到descriptor上  
ex3: 如果想让shader直接画到swapchain image上，这时候就要把swapchain ImageView挂上去  
总而言之，这里需要用什么就挂什么，最后GPU真正进行读写操作的就是这个区域  


# Vulkan Platform结构
施工中：需要更新最新版本  
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

Fence用于阻塞CPU直到Queue中的命令执行结束(GPU、CPU之间同步)。  
Semaphore用于不同的命令提交之间的同步(GPU、GPU之间同步)。 

在Vulkan中，资源读写的同步由app程序员负责。  
Vulkan在record command buffer后，需要调用vkQueueSubmit()函数(分别对graphics and compute queue调用)，这时候GPU才会开始执行传入的命令。  
Vulkan下GPU的执行顺序：  
同时提交的submit，先提交的先执行。  
同一个sumit内，如果有很多个command buffer(存在不同的VkSubmitInfo里),index小的先执行。  
同一个VkSubmitInfo内，如果有很多commands，也是index小的先执行。  
renderpass and subpass?  

## Fence
一开始Fence是未设置的（相当于Fence打开），CPU命令可以通向GPU  
vkWaitForFences会在Fence恢复之前阻塞CPU。这个函数可以理解成一个状态查询，它可能返回VK_TIMEOUT或VK_SUCCESS。第一次运行到这里都是未设置状态。 
vkQueueSubmit(Fence)这个指令会把Fence更改为被设置状态（就是相当于Fence关闭了）。  
并且，当GPU还没有处理完毕这个队列的时候，vkWaitForFence()总是会返回阻塞CPU的结果。  
只有当GPU完成了该次submit的命令队列的所有命令之后，才会把Fence更新为未设置。  
因此，Fence可以看成是以GPU为主导，阻塞CPU的一种消息机制。  
另外，在submit命令队列之前，一般会调用vkResetFence()把Fence手动重置一下(已确保它是未设置的)。       

## Semaphore
Semaphore(信号灯)跟Fence的一个区别是，Semaphore没办法用vkWaitForFences()这样的的函数来查询信号状态。  
Semaphore也有两种状态：singaled(激活)，unsignaled(未激活)  
Semaphore可以指向GPU运行的某个阶段，如果Semaphore被激活，则说明此阶段之前的所有GPU内存均已结束。  

实际操作中，一般采用两个信号灯配合使用(Binary Semaphore)  
Semaphore1(imageAvailableSemaphore)：指向一个预设的阶段，一般是VK_PIPELINE_STAGE_COLOR_ATTACHMENT_OUTPUT_BIT，这一步是GPU处理命令队列的几乎最后一步。表示当此signal激活的时候，命令队列做完了，图像也已经用完了。  
Semaphore2(renderFinishedSemaphore): 指向渲染完成的阶段。表示此时命令队列已经处理完毕。    
为什么要分这两个阶段，暂停两次呢？  

这里就要介绍GPU画每一帧都有三个步骤：  
1、从Swap chain获得可用的image资源；这里会检查Semaphore1，只有Semaphore1的信号被激活才能进行下一步。（vkAcquireNextImageKHR()也才能返回可用的image id）    
2、submit command buffer，执行命令队列，进行绘制工作。这里提交的时候要提供Semaphore1, Sempahore2，和fence；表示这一步完成之后会打开Fence和激活Semaphore2。  
3、将绘制完成的图像返回给Swap chain，并提交Present。提交的时候要设置Semaphore2，表示必须等待Semaphore2的信号被激活才能执行第三步。  
这三个步骤的每一步，都依赖于上一步的完成。  

## Submit Command Queue
从上述讨论我们也可以知道，在Render阶段，CPU要向GPU submit两次指令：  
vkQueueSubmit(CContext::GetHandle().GetGraphicsQueue(), 1, &submitInfo, inFlightFences[currentFrame])  
submitInfo信息包括：
- pCommandBuffers  
- pWaitSemaphores：这里给的是imageAvailableSemaphore(队列的指令会等待这些Semaphre被通知了之后才开始执行)  
- pWaitDstStageMask：一般是VK_PIPELINE_STAGE_COLOR_ATTACHMENT_OUTPUT_BIT这个阶段(表示提交的所有命令在执行到本指针指向的位置时停下。需要等待pWaitSemaphores所指向的所有Semaphore恢复)     
- pSignalSemaphores: 这里给的是renderFinishSemaphore（此次提交的命令全部接收后，本指针指向的所有Semapore都会被通知）  

可见vkQueueSubmit()需要提供1个fence和2个semaphore信息。  

vkQueuePresentKHR(CContext::GetHandle().GetPresentQueue(), &presentInfo)  
presentInfo信息包括：  
- pSwapchains  
- pImageIndices  
- pWaitSemaphores：这里给的是render finish semaphore  

vkQueuePresentKHR()只需要提供1个semaphore信息。
  
另外有Timeline Semaphore技巧，区别是增加了64-bit integer来指示payload。   

## Simple Triangle实例
在simple_triangle这个实例中，使用了2个frame(每个frame都对应于CPU内的一组控制资源)，Semaphore四个，Fence两个  
imageAvailableSemaphore x2 被pWaitSemaphores指向  
renderFinishedSemaphore x2 被pSignalSemaphores指向(两个合起来表示的逻辑就是render完成后，通知image可用了。)  
inFlightFence x2 赋值是VK_PIPELINE_STAGE_COLOR_ATTACHMENT_OUTPUT_BIT  
```vulkan
void drawFrame() {
    vkWaitForFences(device, 1, &inFlightFence, VK_TRUE, UINT64_MAX);
    vkResetFences(device, 1, &inFlightFence);

    uint32_t imageIndex;
    vkAcquireNextImageKHR(device, swapChain, UINT64_MAX, imageAvailableSemaphore, VK_NULL_HANDLE, &imageIndex);

    vkResetCommandBuffer(commandBuffer, /*VkCommandBufferResetFlagBits*/ 0);
    recordCommandBuffer(commandBuffer, imageIndex);

    VkSubmitInfo submitInfo{};
    submitInfo.sType = VK_STRUCTURE_TYPE_SUBMIT_INFO;

    VkSemaphore waitSemaphores[] = {imageAvailableSemaphore};
    VkPipelineStageFlags waitStages[] = {VK_PIPELINE_STAGE_COLOR_ATTACHMENT_OUTPUT_BIT};
    submitInfo.waitSemaphoreCount = 1;
    submitInfo.pWaitSemaphores = waitSemaphores;
    submitInfo.pWaitDstStageMask = waitStages;

    submitInfo.commandBufferCount = 1;
    submitInfo.pCommandBuffers = &commandBuffer;

    VkSemaphore signalSemaphores[] = {renderFinishedSemaphore};
    submitInfo.signalSemaphoreCount = 1;
    submitInfo.pSignalSemaphores = signalSemaphores;

    if (vkQueueSubmit(graphicsQueue, 1, &submitInfo, inFlightFence) != VK_SUCCESS) {
        throw std::runtime_error("failed to submit draw command buffer!");
    }

    VkPresentInfoKHR presentInfo{};
    presentInfo.sType = VK_STRUCTURE_TYPE_PRESENT_INFO_KHR;

    presentInfo.waitSemaphoreCount = 1;
    presentInfo.pWaitSemaphores = signalSemaphores;

    VkSwapchainKHR swapChains[] = {swapChain};
    presentInfo.swapchainCount = 1;
    presentInfo.pSwapchains = swapChains;

    presentInfo.pImageIndices = &imageIndex;

    vkQueuePresentKHR(presentQueue, &presentInfo);
}
```
解读：  
对于Fence  
一开始vkWaitForFences(fence)试图把CPU挡住，但这时候fence未设置（初始状态），所以挡不住。   
然后vkQueueSubmit(Fence)设置Fence。  
下一次运行到drawFrame()的相同frame index的时候，vkWaitForFences(fence)有可能挡住CPU(取决于GPU是否已经执行完毕了上一个command queue里的commands)  
对于Semaphore  
两个Semaphore合起来表示的逻辑就是render完成后，通知image可用了。  
vkAcquireNextImageKHR()的执行依赖image是否可用。  

## Compute Shader实例
如果创建Compute Pipeline，仅使用Computer Shader，那么是不使用Swapchain image的。  
换句话说，从Swapchain里提取image id的vkAcquireNextImageKHR()函数以及把image还回swapchain的函数vkQueuePresentKHR()都不需要使用。  
也不需要使用信号灯。当然也不需要RenderPass和FrameBuffer。  
```vulkan
vkWaitForFences(CContext::GetHandle().GetLogicalDevice(), 1, &computeInFlightFences[currentFrame], VK_TRUE, UINT64_MAX);
vkResetCommandBuffer(renderer.commandBuffers[renderer.computeCmdId][renderer.currentFrame], /*VkCommandBufferResetFlagBits*/ 0);

BeginCommandBuffer(computeCmdId);
BindPipeline(pipeline, VK_PIPELINE_BIND_POINT_COMPUTE, computeCmdId);
renderer.Dispatch(1, 1, 1);
BindDescriptorSets(pipelineLayout, descriptorSets, VK_PIPELINE_BIND_POINT_COMPUTE, computeCmdId);
EndCommandBuffer(computeCmdId);

VkSubmitInfo submitInfo{};
submitInfo.sType = VK_STRUCTURE_TYPE_SUBMIT_INFO;
submitInfo.commandBufferCount = 1;
submitInfo.pCommandBuffers = &commandBuffers[computeCmdId][currentFrame];
vkResetFences(CContext::GetHandle().GetLogicalDevice(), 1, &computeInFlightFences[currentFrame]);
if (vkQueueSubmit(CContext::GetHandle().GetComputeQueue(), 1, &submitInfo, computeInFlightFences[currentFrame]) != VK_SUCCESS) {
  throw std::runtime_error("failed to submit draw command buffer!");
}
```

## Compute+Graphics Pipeline实例
本例中使用两条pipeline进行运算，先计算compute pipeline的结果，然后把它作为输入放进graphics pipeline的vertex shader里。  
除了以上所描述的：graphics需要一组fence，两组semaphore；compute需要一组fence。   
还需要给compute准备一个新的semaphore(computeFinishedSemaphores)，并且把它加入Graphics pipeline的waitSemaphore里。waitStages设置为VK_PIPELINE_STAGE_VERTEX_INPUT_BIT。  
这样，画每一帧的时候CPU先submit compute command queue。GPU就会先调配compute的资源并运算。（此阶段不需要获取swap chain的资源）  
之后，CPU派发command buffer并获取swap chain资源。  
CPU submit graphics command queue，注意此时waitSemaphores里面有两个信号灯：computeFinishedSemaphores(阻塞直到compute计算完毕)和imageAvailableSemaphores(阻塞直到swapchain资源准备完毕)  
之后如正常一样submit present queue就可以了 (使用renderFinishedSemaphores做同步)  
代码片段如下：
```vulkan
        VkSubmitInfo submitInfo{};
        submitInfo.sType = VK_STRUCTURE_TYPE_SUBMIT_INFO;

        // Compute submission        
        vkWaitForFences(device, 1, &computeInFlightFences[currentFrame], VK_TRUE, UINT64_MAX);

        updateUniformBuffer(currentFrame);

        vkResetFences(device, 1, &computeInFlightFences[currentFrame]);

        vkResetCommandBuffer(computeCommandBuffers[currentFrame], /*VkCommandBufferResetFlagBits*/ 0);
        recordComputeCommandBuffer(computeCommandBuffers[currentFrame]);

        submitInfo.commandBufferCount = 1;
        submitInfo.pCommandBuffers = &computeCommandBuffers[currentFrame];
        submitInfo.signalSemaphoreCount = 1;
        submitInfo.pSignalSemaphores = &computeFinishedSemaphores[currentFrame];

        if (vkQueueSubmit(computeQueue, 1, &submitInfo, computeInFlightFences[currentFrame]) != VK_SUCCESS) {
            throw std::runtime_error("failed to submit compute command buffer!");
        };

        // Graphics submission
        vkWaitForFences(device, 1, &inFlightFences[currentFrame], VK_TRUE, UINT64_MAX);

        uint32_t imageIndex;
        VkResult result = vkAcquireNextImageKHR(device, swapChain, UINT64_MAX, imageAvailableSemaphores[currentFrame], VK_NULL_HANDLE, &imageIndex);

        if (result == VK_ERROR_OUT_OF_DATE_KHR) {
            recreateSwapChain();
            return;
        } else if (result != VK_SUCCESS && result != VK_SUBOPTIMAL_KHR) {
            throw std::runtime_error("failed to acquire swap chain image!");
        }

        vkResetFences(device, 1, &inFlightFences[currentFrame]);

        vkResetCommandBuffer(commandBuffers[currentFrame], /*VkCommandBufferResetFlagBits*/ 0);
        recordCommandBuffer(commandBuffers[currentFrame], imageIndex);

        VkSemaphore waitSemaphores[] = { computeFinishedSemaphores[currentFrame], imageAvailableSemaphores[currentFrame] };
        VkPipelineStageFlags waitStages[] = { VK_PIPELINE_STAGE_VERTEX_INPUT_BIT, VK_PIPELINE_STAGE_COLOR_ATTACHMENT_OUTPUT_BIT };
        submitInfo = {};
        submitInfo.sType = VK_STRUCTURE_TYPE_SUBMIT_INFO;

        submitInfo.waitSemaphoreCount = 2;
        submitInfo.pWaitSemaphores = waitSemaphores;
        submitInfo.pWaitDstStageMask = waitStages;
        submitInfo.commandBufferCount = 1;
        submitInfo.pCommandBuffers = &commandBuffers[currentFrame];
        submitInfo.signalSemaphoreCount = 1;
        submitInfo.pSignalSemaphores = &renderFinishedSemaphores[currentFrame];

        if (vkQueueSubmit(graphicsQueue, 1, &submitInfo, inFlightFences[currentFrame]) != VK_SUCCESS) {
            throw std::runtime_error("failed to submit draw command buffer!");
        }

        VkPresentInfoKHR presentInfo{};
        presentInfo.sType = VK_STRUCTURE_TYPE_PRESENT_INFO_KHR;

        presentInfo.waitSemaphoreCount = 1;
        presentInfo.pWaitSemaphores = &renderFinishedSemaphores[currentFrame];

        VkSwapchainKHR swapChains[] = {swapChain};
        presentInfo.swapchainCount = 1;
        presentInfo.pSwapchains = swapChains;

        presentInfo.pImageIndices = &imageIndex;

        result = vkQueuePresentKHR(presentQueue, &presentInfo);

        if (result == VK_ERROR_OUT_OF_DATE_KHR || result == VK_SUBOPTIMAL_KHR || framebufferResized) {
            framebufferResized = false;
            recreateSwapChain();
        } else if (result != VK_SUCCESS) {
            throw std::runtime_error("failed to present swap chain image!");
        }

        currentFrame = (currentFrame + 1) % MAX_FRAMES_IN_FLIGHT;
```

## 使用Compute计算，但是把结果画在Swapchain上  
既然Swapchain Image也是vk image，可以不需要走Graphics Pipeline直接画在swapchain image上。  
Swapchain Image是有独立的内存空间的，正式名称为Color Image。使用Vulkan规则API创建。  
虽然不用Graphics Pipeline了，但还是需要shader的。这里直接用compute shader画在swapchain的image上。  

代码(来自VulkanSingleExamples/VulkanComputeImageExample)：  
```vulkan
void drawFrame() {
        vkWaitForFences(device, 1, &inFlightFences[currentFrame], VK_TRUE, UINT64_MAX);

        uint32_t imageIndex;
        VkResult result = vkAcquireNextImageKHR(device, swapChain, UINT64_MAX, imageAvailableSemaphores[currentFrame], VK_NULL_HANDLE, &imageIndex);

        if (result == VK_ERROR_OUT_OF_DATE_KHR) {
            recreateSwapChain();
            return;
        }
        else if (result != VK_SUCCESS && result != VK_SUBOPTIMAL_KHR) {
            throw std::runtime_error("failed to acquire swap chain image!");
        }

        if (imagesInFlight[imageIndex] != VK_NULL_HANDLE) {
            vkWaitForFences(device, 1, &imagesInFlight[imageIndex], VK_TRUE, UINT64_MAX);
        }
        imagesInFlight[imageIndex] = inFlightFences[currentFrame];

        VkSubmitInfo submitInfo{};
        submitInfo.sType = VK_STRUCTURE_TYPE_SUBMIT_INFO;

        VkSemaphore waitSemaphores[] = { imageAvailableSemaphores[currentFrame] };
        VkPipelineStageFlags waitStages[] = { VK_PIPELINE_STAGE_COLOR_ATTACHMENT_OUTPUT_BIT };
        submitInfo.waitSemaphoreCount = 1;
        submitInfo.pWaitSemaphores = waitSemaphores;
        submitInfo.pWaitDstStageMask = waitStages;

        submitInfo.commandBufferCount = 1;
        submitInfo.pCommandBuffers = &commandBuffers[imageIndex];

        VkSemaphore signalSemaphores[] = { renderFinishedSemaphores[currentFrame] };
        submitInfo.signalSemaphoreCount = 1;
        submitInfo.pSignalSemaphores = signalSemaphores;

        vkResetFences(device, 1, &inFlightFences[currentFrame]);

        if (vkQueueSubmit(computeQueue, 1, &submitInfo, inFlightFences[currentFrame]) != VK_SUCCESS) {
            throw std::runtime_error("failed to submit draw command buffer!");
        }

        VkPresentInfoKHR presentInfo{};
        presentInfo.sType = VK_STRUCTURE_TYPE_PRESENT_INFO_KHR;

        presentInfo.waitSemaphoreCount = 1;
        presentInfo.pWaitSemaphores = signalSemaphores;

        VkSwapchainKHR swapChains[] = { swapChain };
        presentInfo.swapchainCount = 1;
        presentInfo.pSwapchains = swapChains;

        presentInfo.pImageIndices = &imageIndex;

        result = vkQueuePresentKHR(presentQueue, &presentInfo);

        if (result == VK_ERROR_OUT_OF_DATE_KHR || result == VK_SUBOPTIMAL_KHR || framebufferResized) {
            framebufferResized = false;
            recreateSwapChain();
        }
        else if (result != VK_SUCCESS) {
            throw std::runtime_error("failed to present swap chain image!");
        }

        currentFrame = (currentFrame + 1) % MAX_FRAMES_IN_FLIGHT;
    }

```
跟simple triangle的draw函数相比，额外有一个fence：  
vkWaitForFences(device, 1, &imagesInFlight[imageIndex], VK_TRUE, UINT64_MAX);  

imageInFlight: size = swapChainImages.size(). 也就是它的大小跟swapchain image的数量一致(而不是像其他的fence一样大小跟MAX_FLRAMES_IN_FLIGHT一致)  
```vulkan
std::vector<VkFence> imagesInFlight;
```
但他的作用跟其他的fence一样，也是阻塞CPU。  
重点是以下这一段(位于submit compute tree之前)：
```vulkan
if (imagesInFlight[imageIndex] != VK_NULL_HANDLE) {
    vkWaitForFences(device, 1, &imagesInFlight[imageIndex], VK_TRUE, UINT64_MAX);
}
imagesInFlight[imageIndex] = inFlightFences[currentFrame];
```
解读：imageIndex是从Swapchain申请的资源，currentFrame是CPU当前使用的资源组。  
每次提交compute queue之前，先把当前frame的InFlightFence[currentFrame]保存到imagesInFlight[imageIndex](一个临时fence容器)上。  
然后在下几帧，如果刚好又轮到同样的imageIndex,这时候之前那个currentFrame fence没打开的话，就会阻塞住。  
这种双重阻塞的目的是，既保证了GPU已经准备好接受compute command buffer(通过InFlightFences)，又保证了GPU上imageIndex对应的swapchain image是可用的(通过imagesInFlight)。  
如果不加上imagesInFlight，有可能发生的情况是，CPU acuire到一个"空闲"的swapchain image，但实际上之前的某一帧正在往上面写东西。  
在Simple Triangle这个例子里，因为我们不会直接向swapchain image写东西，CPU acuire到的就是真的空闲的swapchain image  

## Vulkan同步Reference
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


