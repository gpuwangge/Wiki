# Cubemap
Cubemap是一种特殊的texture，它由6个单独的2D textures组成。  
使用Cubemap的好处是它能贴在1x1x1的unit cube(该cube的中心在origin)上，并且用一个Direction vector(一条从origin发射的光线)来sample。  
当定义了Cubemap和Direction vector，图形API就能够计算出其相交的texels，并且返回正确的sampled texture value。  
如果使用如上的unit cube，cube本身的pos坐标其实就是sample的pos。因此，unit cube的vector position就是texture coordinates。  


# Skybox
Skybox是一个cube，它使用Cubemap(通常需要很大的分辨率)贴合，它的用途是在游戏中用于展示远山、星空等背景场景  
这种技术能让玩家觉得自己身处在一个巨大的宇宙场景中(尽管实际上是在一个小盒子里)  


# Environment Mapping
Environment Mapping是一项利用Cubemap来提高渲染效果的技术。其中最常见的两项是Reflection和Refraction  
Reflection: 物体反射环境光线  
Refraction：物体折射环境光线  
只要在渲染物体的时候，稍微修改fragment shader code，就可以把Cubemap的信息添加到物体的颜色中，使其出现Reflection和Refraction效果  

当然，对于一个物体，这样做的效果是很好的。但假如有多个物体，仅反射skybox就不行了。需要实时产生动态的Cubemap。这项技术叫做Dynamic Environment Mapping  
DEM的实现需要很多时间开销，一般来说实践中会利用pre-render Cubmap做优化  


# 在Vulkan里使用Cubemap
1. 准备用于cubemap的材质。可以准备6张方形材质，也可以把6张材质合并成一张。这里采取合并一张的做法，并且6张材质水平排列  
排列顺序一般为：front, back, up, down, right, and left  
举例：1024x1024的材质拼成6144x1024。假设每个texel由4个channel组成，并且每个channel由8个bits表示，材质总大小为6144x1024x4=25165824 bytes  
2. 在vkCreateImage的时候，设定imageCreateInfo.arrayLayers = 6和imageCreateInfo.flags = VK_IMAGE_CREATE_CUBE_COMPATIBLE_BIT  
（但imageType依旧保持VK_IMAGE_TYPE_2D）  
3. 在vkCreateImageView的时候，设定view.subresourceRange.layerCount = 6和view.viewType = VK_IMAGE_VIEW_TYPE_CUBE  
4. 将texture load进memory，上述6张方形材质称为一个layer。传的时候imagesize是所有6个layer的大小合起来计算；layersize就是一个layer的size，因此layersize=imagesize/6)  
5. 将memory里面的texel传进GPU之前需要改换layout(就像所有的材质一样)。但有一个区别是barrier.subresourceRange.layerCount需要设置成6  
6. 传输的时候需要分成6次拷贝。每个layer需要单独设定一个region，这个region需要正确的imageExtent和bufferOffset。示例如下：  
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
7. 正确创建Cubemap材质后，就可以把它如同普通材质一样贴在cube上。在贴的时候需要的修改：在vertex shader里，把pos坐标(而不是texture coordiante坐标)(pos是vec3类型)传给fragment shader  
8. 在fragment shader里使用samplerCube而不是sampler2D来采样：texture(samplerCube, pos)  
9. 其他要注意的：当场景里有很多物体的时候，skybox需要第一个draw，并且要关闭depth test，但是，使用early depth testing可以优化skybox  



# Reference
https://learnopengl.com/Advanced-OpenGL/Cubemaps  
https://github.com/SaschaWillems/Vulkan/blob/master/examples/texturecubemap/texturecubemap.cpp  
https://satellitnorden.wordpress.com/2018/01/23/vulkan-adventures-cube-map-tutorial/  
https://github.com/JerryYan97/Vulkan-Samples-Dictionary/tree/master/Samples/2-15_CubeMap/DebugSample/glsl  
