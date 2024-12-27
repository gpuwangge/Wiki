# Cubemap
Cubemap是一种特殊的texture，它由6个单独的2D textures组成。  
使用Cubemap的好处是它能贴在1x1x1的unit cube(该cube的中心在origin)上，并且用一个Direction vector(一条从origin发射的光线)来sample。  
当定义了Cubemap和Direction vector，图形API就能够计算出其相交的texels，并且返回正确的sampled texture value。  
如果使用如上的unit cube，cube本身的pos坐标其实就是sample的pos。因此，unit cube的vector position就是texture coordinates。  

# Skybox
Skybox是一个cube，它使用Cubemap(通常需要很大的分辨率)贴合，它的用途是在游戏中用于展示远山、星空等背景场景  
这种技术能让玩家觉得自己身处在一个巨大的宇宙场景中(尽管实际上是在一个小盒子里)  
Skybox和Cubemap的区别：后者是一种贴图(采样)方式，前者是一种应用方式。凡是skybox都需要用cubemap技术。但反过来不一定。比如之后会提到environment map应用的时候也是用了cubemap，但它不是skybox。  

# 在Vulkan里使用Cubemap
1. 准备用于cubemap的材质。可以准备6张方形材质，也可以把6张材质合并成一张。这里采取合并一张的做法，并且6张材质水平排列  
排列顺序一般为：front, back, up, down, right, and left  
举例：1024x1024的材质拼成6144x1024。假设每个texel由4个channel组成，并且每个channel由8个bits表示，材质总大小为6144x1024x4=25165824 bytes  
2. 在vkCreateImage的时候，设定imageCreateInfo.arrayLayers = 6和imageCreateInfo.flags = VK_IMAGE_CREATE_CUBE_COMPATIBLE_BIT  
另外，既然layer分成了6个，那么extent.width就需要设置成原来的六分之一了  
这时候image在内存里的模样就相当于六张方形煎饼叠在一起  
但imageType依旧保持VK_IMAGE_TYPE_2D  
3. 在vkCreateImageView的时候，设定view.subresourceRange.layerCount = 6和view.viewType = VK_IMAGE_VIEW_TYPE_CUBE  
4. 将texture load进memory，上述6张方形材质称为一个layer。传的时候imagesize是所有6个layer的大小合起来计算；layersize就是一个layer的size，因此layersize=imagesize/6)  
5. 将memory里面的texel传进GPU之前需要改换layout(就像所有的材质一样)。但有一个区别是barrier.subresourceRange.layerCount需要设置成6  
6. 传输的时候需要分成6次拷贝。每个layer需要单独设定一个region，这个region需要正确的imageExtent和bufferOffset。对于一行排开的六张图片，读取示例如下：  
```c++
void CTextureImage::copyBufferToImage_cubemap(VkBuffer buffer, VkImage image, uint32_t width, uint32_t height) {
  VkCommandBuffer commandBuffer = beginSingleTimeCommands();

  VkBufferImageCopy regions[6];
  memset(regions, 0, sizeof(regions));
  for(int i = 0; i < 6; i++){
    regions[i].bufferOffset = i * (width / 6) * 4;// is the offset in bytes from the start of the buffer object where the image data is copied from or to
    regions[i].bufferRowLength = width; //specify in texels a subregion of a larger two- or three-dimensional image in buffer
    regions[i].bufferImageHeight = height;
    regions[i].imageSubresource.aspectMask = VK_IMAGE_ASPECT_COLOR_BIT; //imageSubresource is a VkImageSubresourceLayers used to specify the specific image subresources of the image used for the source or destination image data.
    regions[i].imageSubresource.mipLevel = 0;
    regions[i].imageSubresource.baseArrayLayer = i;
    regions[i].imageSubresource.layerCount = 1;
    regions[i].imageOffset = { 0, 0, 0 }; //selects the initial x, y, z offsets in texels of the sub-region of the source or destination image data.
    regions[i].imageExtent = { //is the size in texels of the image to copy in width, height and depth.
        width/6,
        height,
        1
    };
	}

  vkCmdCopyBufferToImage(commandBuffer, buffer, image, VK_IMAGE_LAYOUT_TRANSFER_DST_OPTIMAL, 6, regions);

  endSingleTimeCommands(commandBuffer);
}
```
对于更加流行的一种排布方式，读取方法如下：  
```
/* Standard Skybox Format
*			up
*	left	front	right	back
*			down
*/
unsigned int extend_width = width / 4;
unsigned int extend_height = height / 3;
for(int i = 0; i < 6; i++){
	regions[i].bufferRowLength = width; //specify in texels a subregion of a larger two- or three-dimensional image in buffer
	regions[i].bufferImageHeight = height;
	regions[i].imageSubresource.aspectMask = VK_IMAGE_ASPECT_COLOR_BIT; //imageSubresource is a VkImageSubresourceLayers used to specify the specific image subresources of the image used for the source or destination image data.
	regions[i].imageSubresource.mipLevel = 0;
	regions[i].imageSubresource.baseArrayLayer = i;
	regions[i].imageSubresource.layerCount = 1;
	regions[i].imageOffset = { 0, 0, 0 }; //selects the initial x, y, z offsets in texels of the sub-region of the source or destination image data.
	regions[i].imageExtent = { //is the size in texels of the image to copy in width, height and depth.
		extend_width,
		extend_height,
		1
	};
}
int unit_width = extend_width * 4;
int unit_height = extend_height * 4;
regions[0].bufferOffset = 1 * unit_height * unit_width + 2 * unit_width; 	//right
regions[1].bufferOffset = 1 * unit_height * unit_width; 					//left
regions[2].bufferOffset = 0 * unit_height * unit_width + 1 * unit_width; 	//up
regions[3].bufferOffset = 2 * unit_height * unit_width + 1 * unit_width; 	//bottom
regions[4].bufferOffset = 1 * unit_height * unit_width + 1 * unit_width; 	//front
regions[5].bufferOffset = 1 * unit_height * unit_width + 3 * unit_width; 	//back
```
7. 正确创建Cubemap材质后，就可以把它如同普通材质一样贴在cube上。在贴的时候需要的修改：在vertex shader里，把pos坐标(而不是texture coordiante坐标)(pos是vec3类型)传给fragment shader  
8. 在fragment shader里使用samplerCube而不是sampler2D来采样：texture(samplerCube, pos)  

# 其他要注意的问题
1. 到底是先渲染cube，还是先渲染其它物体?  
因为cube是显示在背景里的，直觉上应该最先渲染cube。  
但是考虑到实际游戏中通常画面会被非背景物体填满，如果先渲染cube会很浪费fragment shader。  
因此实际上考虑到效率，应该最后渲染cube。  
而且需要开启Eealy depth testing。  

2. cube只有1x1x1大小，如果其他物体出现在这个范围之外，会被cube挡住因此看不见怎么办？  
具体做法是令cube的position.z分量等于position.w分量，这样进行透视除法的时候，z/w永远等于1.0(既认为cube有最大的深度值，无限远)  
也可以直接在fragment shader里指定z=1.0。  

3. 当摄像机转动的时候，view矩阵会改变，cube也要相应改变；但是摄像机移动的时候，希望view矩阵不变，否则cube也会跟着移动(背景理论上不应该随着摄像机的移动而变化)  
所以要在cube的view矩阵里去掉摄像机的translate分量。    

# Environment Mapping
Environment Mapping是一项利用Cubemap来提高渲染效果的技术。其中最常见的两项是Reflection和Refraction  
Reflection: 物体反射环境光线  
Refraction：物体折射环境光线  
只要在渲染物体的时候，稍微修改fragment shader code，就可以把Cubemap的信息添加到物体的颜色中，使其出现Reflection和Refraction效果  

当然，对于一个物体，这样做的效果是很好的。但假如有多个物体，仅反射skybox就不行了。需要实时产生动态的Cubemap。这项技术叫做Dynamic Environment Mapping  
DEM的实现需要很多时间开销，一般来说实践中会利用pre-render Cubmap做优化  

具体做法：(未验证)  
选择希望进行EM的物体，为它的shaders做出如下修改  
1. 在vertex shader里计算normal  
2. 在fragment shader里计算I(摄像机方向)；用结果计算R(反射方向)；用R作为输入变量进行samplerCube  
3. 在host(cpu)端，把cubebox的贴图贴在物体上  
这样贴出来的效果就是物体表面像镜面一样反射出背景。  
如果不想要镜子效果，可以添加反射贴图，造成不完全反射的效果(比如在金属部分制造反射，非金属部分禁止反射)。  
4. 在fragment shader里给R添加一个ratio，可以改变折射率，创造折射效果。  
运行结果就是更加像玻璃了。  
以上就是EM的实现方法。DEM需要每一帧创建六个角度的新贴图，因此需要很大的额外开销。其他做法是类似的。  



# Reference
https://learnopengl.com/Advanced-OpenGL/Cubemaps  
https://github.com/SaschaWillems/Vulkan/blob/master/examples/texturecubemap/texturecubemap.cpp  
https://satellitnorden.wordpress.com/2018/01/23/vulkan-adventures-cube-map-tutorial/  
https://github.com/JerryYan97/Vulkan-Samples-Dictionary/tree/master/Samples/2-15_CubeMap/DebugSample/glsl  
https://learnopengl-cn.github.io/04%20Advanced%20OpenGL/06%20Cubemaps/  

