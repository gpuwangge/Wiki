# 什么是Driver
Driver(驱动)是使硬件正常工作的程序。    
硬件在研发的时候本来就有配套的工作指令，这些指令往往保存在硬件上并可以由固件(Firmware)调用。  
作为软件开发者，直接使用这些底层指令是复杂和低效的。尤其是软件开发者极其依赖操作系统。  
因而软件开发者需要这样一种程序，它能令开发者在某个操作系统上调取硬件的信息，并且能够与该硬件通信，从而调动硬件上的底层指令令其正常工作。  
这个软件就是驱动程序。  
可见驱动程序是跟操作系统相关的。  
通常操作系统会默认自带“系统兼容驱动”，它集成一些简单驱动，这样即使不安装任何驱动，显卡、键盘等硬件也能工作。但是要获得完整体验，还是应该安装完整驱动才行。  
也有一些硬件，比如无线网卡，会把驱动集成到硬件里。  

# GPU Driver的作用是把GPU的复杂指令封装起来以供调用  
CPU的简单指令储存在主板的BIOS里  
CPU的复杂些的指令集成在OS上(为什么没有CPU Driver呢？因为集成在OS上了。CPU Driver也叫CPU核心，bootloader，中断管理，进程管理，内存管理，电源管理等一大堆功能)  
Compiler可以把指令编译成机器码让CPU运行  
(额外话：另一个不需要Driver的硬件是内存条)  

GPU的基本指令也存储在VGA BIOS里(调用这些指令也是通过CPU来完成)  
因此，即使没有安装Driver，GPU也是有基本绘图功能的(通过CPU调用)  

但是开发图形应用程序的时候，不会去关注GPU的基本指令，因为这些指令是制造商相关的。而图形API是有标准的：比如OpenGL或Vulkan  

GPU Driver是什么： 是一个运行在CPU上的软件  
GPU Driver要做的事情是：把一些稍微复杂的指令（比如给GPU发送一些命令和数据，取回一些数据或状态), 封装起来为上层提供更为抽象的应用，使上层不用关注硬件的细节。  
换句话说，如果App软件层清楚的知道硬件调用的细节，也能绕过Driver直接操作GPU实现复杂的指令  

为啥GPU Driver不集成到OS上，而是要单独装呢： 插在主板上的硬件种类太多了，GPU卡的制造商和种类也太多了，全部装上浪费资源，所以只在用的时候装需要的  


# 以OpenGL API的工作流程为例说明GPU Driver的作用  
1. OpenGL建立渲染画布，顶点变换，视点变换，场景贴图，Alpha渲染，场景渲染等  
2. App使用OpenGL建立流程后，最后使用提交命令  
3. (Driver)提交命令将OpenGL指令转换成对GPU操作的指令，提交给GPU  
4. (Driver)对指令中使用的数据，应使用DMA进行数据传递到GPU的显存。这些数据可能是顶点坐标，或是贴图等  
5. GPU执行指令，比如进行顶点矩阵变换，或者像素的alpha运算。GPU和CPU之间的DMA可能成为执行速度的瓶颈。DMA传输速度看PCIE的速度  


# GPU Driver(驱动)和GPU Firmware(固件)的关系
如上所述，GPU Driver是一段运行在CPU上的软件。以下简称"驱动"。  
GPU Firmware是一段运行在GPU上的软件(可以理解为显卡上有一个小的CPU组件)。以下简称"固件"。  
固件比驱动更加接近硬件操纵底层。对很多硬件来说，固件是最底层的软件。有的硬件没有非固件的软件部分。  
说句题外话，BIOS(Basic Input Output System)也可以看成是主板固件的一部分。BIOS由汇编写成，它的进阶版本为UEFI(Unified Extensible Firmware Interface)，由C语言写成。UEFI有更强大的功能，比如支持更强大的的图形显示，更安全的启动方式，和更大的内存寻址等。  
当然除了主板有固件，其他的网卡、硬盘等都有各自的固件。这一章我们讨论的“固件”特指显卡固件。  
早期的固件是在生产过程中写到硬件里的，成为产品后不能更改。但是后来设计上为了灵活性，固件也支持不频繁的更新了。而驱动的升级就频繁很多。  
以下以ARM GPU为例说明一块显卡的启动流程，以及驱动和固件在其中的作用：  
1. 用户运行图形API程序，CPU运行驱动程序准备叫显卡工作  
2. 驱动程序把GPU需要的数据准备到内存中。数据包含Pagetable、固件和GPU执行指令的Program、以及Program数据  
3. 驱动程序通过设置GPU MMU(Memory Mange Unit) Register的方式告知GPU Pagetable的位置(Pagetable的作用是为GPU使用内存空间做准备。GPU使用内存空间的两个部分是固件和GPU指令，因此可能存在两个Pagetable)  
4. 驱动程序通过设置GPU MMU Register的方式唤醒固件和GPU的其他部分。固件会开始去内存中读pagetable  
5. 驱动程序通过设置GPU MMU Register的方式命令固件去读固件program，也就是固件会运行的指令。之后固件会根据这些指令运行设置  
  （同时也包含驱动想告诉固件的一些setting参数，比如command queue size、command queue地址等，它们会被驱动提前准备在GPU的固件内存里） 
7. 驱动程序通过设置GPU MMU Register的方式命令GPU去读GPU program。这个program是真的GPU会做的事情，比如做一个draw call  
8. 驱动程序通过设置Doorbell让GPU开始工作，然后等待GPU工作完成返回的中断  
9. 驱动程序通过设置GPU MMU Register的方式令固件和GPU关闭(沉睡)  

从Driver的角度看，除了memory和data的配置操作，Driver对GPU的操作就是不停写和读一些register。真正让GPU干活的底层操作是由Firmware来调配的。但Firmware需要的program和data还是要由driver准备。  

另外，Driver和Firmware的沟通实际上是两套软件在dynamic地沟通。会存在某个系统透过memory等待另一个系统的情况。比如驱动程序叫固件去读pagetable之后，必须等待固件读取完毕才能进行下一步(读program)操作。这也是为什么driver需要读register的原因。  

# Reference  
https://blog.csdn.net/m0_51737348/article/details/122657090  
