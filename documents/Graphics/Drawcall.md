# Drawcall基础规则
在graphics pipeline里，一次只能绑定一个pipeline state object（PSO）。一个PSO就如同一支画笔。  

如果要在同一个graphics pipeline里面换不同的PSO，需要多个dracall，或一个drawcall切成多个renderpass。  

因此，只要使用同一个PSO，同一组descriptor，同一个顶点缓冲区内所有三角形(可以包含很多物体)，都可以用一个draw call完成。  

在Vulkan里，同一个command buffer里可以record多个vkCmdDraw(), 每次算一个独立drawcall。换句话说，如果场景里是个object每个调用一次vkCmdDraw，会产生10个drawcall。  

多次drawcall带来的问题：每次drawcall都需要读取指令，检查资源状态和同步，降低渲染效率。  
使用vkCmdDrawIndirect：相当于把多次drawcall和并成一个drawcall，显著降低CPU端的API调用，到了GPU再连续执行多条绘制指令。这样在有大量绘制对象的时候可以提升整体帧率。  
经验法则：每frame对象>200的时候，间接绘制能看到CPU段效率的显著提升。  

# 优化Drawcall
自己的渲染引擎，如果绘制对象很多，单独的vkCmdDraw做法会让CPU成为帧率瓶颈。提升的办法
- 上面提到的间接渲染
- 几何合并（适合共享管线和材质的情况）
- Instancing（适合同样模型但不同位置、颜色等属性）
- Push Constant合并（如果每个物体只有少量独立数据）

# Shadowmap和Drawcall
在shadow map算法里，因为要生成shadowmap和正常渲染，所以至少需要两个drawcall。  
为了避免跨帧依赖，采取两个renderpass的形式，每个renderpass里有一个drawcall。  

# 非Drawcall
Raytracing Pipeline因为不需要drawcall，所以drawcall数量为零。  

# Benchmark里的drawcall
一个真实的光栅化benchmark里面每帧可能有几百上千个drawcall，原因可能是：  
- 场景中有很多Mesh，每个mesh有自己的材质，并且材质不共享
- 透明物体需要单独draw，而且要排序
- 多个renderpass，比如shadowmap算法，g-buffer相关算法
- CPU端的LOD实现
- Particle System

现代GPU硬件每帧在4k解析度下可以处理5k-10k drawcall，只要CPU不被卡住。  

以Manhattan Benchmark为例，drawcall数量大致估算如下：  
- 2000个独立建筑，在不使用instancing的情况下产生8000-10000个submesh drawcall
- 路面、路肩、地下管线：300个mesh，合并为1-2个mesh，1-2个draw
- 街灯，路灯，交通标志：1200个单独材质需要单独的drawcall
- 车辆：300台，10个种类，用instancing之后10个drawcall
- 树木/植被：5000颗，用instancing之后1-2个drawcall
- 透明物体如雨滴、玻璃、雾气：800个，单独排序，800 drawcall
- 阴影：两个主光源，2 drawcalls
- 后处理：1-3 drawcalls

以上说两因为frustom的缘故会大幅被剔除，比如建筑可能最后能看见的也就剩30%，因此实际没那么多。  