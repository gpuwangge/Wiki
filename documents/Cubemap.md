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


# Vulkan和Cubemap



# 一个Vulkan的samplerCube例子
正确加载Cubemap后，在vertex shader里，把pos坐标(不需要texture coordiante坐标)传给fragment shader  
在fragment shader里使用samplerCube而不是sampler2D来采样：  
texture(samplerCube, dir), while dir is a vec3  

当场景里有很多物体的时候，skybox需要第一个draw，并且要关闭depth test  
但是，使用early depth testing可以优化skybox  



# Reference
https://learnopengl.com/Advanced-OpenGL/Cubemaps  
https://github.com/SaschaWillems/Vulkan/blob/master/examples/texturecubemap/texturecubemap.cpp  
https://satellitnorden.wordpress.com/2018/01/23/vulkan-adventures-cube-map-tutorial/  
https://github.com/JerryYan97/Vulkan-Samples-Dictionary/tree/master/Samples/2-15_CubeMap/DebugSample/glsl  
