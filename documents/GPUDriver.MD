# GPU Driver对OpenGL API的工作流程  
1. OpenGL建立渲染画布，顶点变换，视点变换，场景贴图，Alpha渲染，场景渲染等  
2. App使用OpenGL建立流程后，最后使用提交命令  
3. (Driver)提交命令将OpenGL指令转换成对GPU操作的指令，提交给GPU  
4. (Driver)对指令中使用的数据，应使用DMA进行数据传递到GPU的显存。这些数据可能是顶点坐标，或是贴图等  
5. GPU执行指令，比如进行顶点矩阵变换，或者像素的alpha运算。GPU和CPU之间的DMA可能成为执行速度的瓶颈。DMA传输速度看PCIE的速度  

# Reference  
https://blog.csdn.net/m0_51737348/article/details/122657090  
