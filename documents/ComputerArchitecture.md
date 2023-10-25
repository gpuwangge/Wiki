# Computer Architecture

Architecture，架构，也俗称平台。比如arm架构，也叫arm平台。  

首先，在Linux下要获得当前系统的构架信息，可以使用如下命令：  
> uname -m

结果可能是如下值
x86_64  (64 bit)
aarch64 (64 bit ARM)  
armv71 (32 bit ARM的一种)  

当编译完成后，在linux系统下也可以用file命令查看编译的可执行文件匹配哪一个架构  


## 架构和指令集(Instruction Set Architecture, ISA)的关系
程序被编译成CPU可识别代码的过程中，需要按照指令集ISA规范
ISA分为两种：
CISC: Complex Instruction Set Computer, x86_64使用这种  
RISC: Reduced Instruction Set Computer, arm是用这种。这种指令集的特点是每一条指令都足够简单，并且指令间互相没有功能重合。

## 架构和操作系统的关系
所谓操作系统，是指windows, linux等等   
任何操作系统，都可以运行在不同架构上。比如Windows即可以运行在x86架构上，也可以arm架构上  
arm架构上，也可以运行Windows或Linux操作系统  
但从市场占有率上看，大部分Windows系统（家用电脑）都运行在x86架构上；大部分嵌入式Linux系统（移动设备，比如android）都运行在arm架构上；大部分服务器Linux都运行在x86架构上。这是历史的原因造成的。  



## 32bit ARM(arm, armv6, armv7)
若要为此系统编译，需要的编译器通常有如下命名：  
gcc-arm-linux-gnu   



## 64bit ARM(aarch64)
若要为此系统编译，需要的编译器通常有如下命名：  
gcc-aarch64-linux-gnu  


## x86_64
x86一开始是32bit，后来扩展成x86_64就是64bit了。  
若要为此系统编译，需要的编译器通常有如下命名：  
gcc  



