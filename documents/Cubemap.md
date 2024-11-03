# Cubemap介绍
Cubemap是一种特殊的texture，它由6个单独的2D textures组成。  
使用Cubemap的好处是它能贴在1x1x1的unit cube(该cube的中心在origin)上，并且用一个Direction vector(一条从origin发射的光线)来sample。  
当定义了Cubemap和Direction vector，图形API就能够计算出其相交的texels，并且返回正确的sampled texture value。  
如果使用如上的unit cube，cube本身的pos坐标其实就是sample的pos。因此，unit cube的vector position就是texture coordinates。  


# Vulkan和Cubemap



# 一个Vulkan的samplerCube例子
正确加载Cubemap后，在shader里，使用samplerCube而不是sampler2D来采样：  
texture(samplerCube, dir), while dir is a vec3  




# Reference
https://learnopengl.com/Advanced-OpenGL/Cubemaps  
https://satellitnorden.wordpress.com/2018/01/23/vulkan-adventures-cube-map-tutorial/  
https://github.com/JerryYan97/Vulkan-Samples-Dictionary/tree/master/Samples/2-15_CubeMap/DebugSample/glsl  
