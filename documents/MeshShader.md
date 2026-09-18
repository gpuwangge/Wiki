# Mesh Shader介绍
Mesh Shader是为了解决ras之前geometry管线的缺点：很多vertex被处理了之后却因为种种原因被丢弃。  

Task Shader: 扫描屏幕和相机，决定哪些物体该算，哪些该直接跳过。  
Mesh Shader：只处理挑中的物体，可以细分物体。  
Fragment Shader：接受精修过的三角形。  

输入mesh shader的是payload，而不是传统数据格式。  

mesh shader并不是全场景通用，而是适合特定场景。比如适合开放世界海量实体渲染。不适合简单场景。  

# Mesh Shader对比Tessellation和Nanite
对比tessellation：tess是加了一个步骤，mesh shader是完全取代ras之前的步骤。tess效果不如mesh shader。  
对比nanite：nanite不是独立技术，它中间的一步可以用mesh shader实现。nanite是个大架构。这个引擎技术一般绑定UE5。nanite效果很好，电影级别高模，无lod突变。  

# 各厂商的mesh shader硬件支持  
apple：从a14起，metal原生完整支持  
qualcomm：从a6xxx起支持较广泛  
arm：mali g710后支持较广泛  
img：比较受限。意思是由硬件，但需要适配。img的mesh shader硬件实现跟tbdr高度耦合。  



