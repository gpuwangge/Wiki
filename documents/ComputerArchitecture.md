# Computer Architecture

Architecture，架构，也俗称平台。比如arm架构，也叫arm平台。  

首先，在Linux下要获得当前系统的构架信息，可以使用如下命令：  
> uname -m

结果可能是如下值  
x86_64  (32 and 64 bit)  
aarch64 (64 bit ARM)    
armv71 (32 bit ARM的一种)   

当编译完成后，在linux系统下也可以用file命令查看编译的可执行文件匹配哪一个架构  


## 架构和指令集(Instruction Set Architecture, ISA)的关系
程序被编译成CPU可识别代码的过程中，需要按照指令集ISA规范
ISA分为两种：
CISC: Complex Instruction Set Computer, x86_64架构使用CISC指令集。专利主要在Intel/AMD手上，CISC指令集需要Intel/AMD其商业授权。  
RISC: Reduced Instruction Set Computer, arm架构使用RISC指令集。专利主要在ARM手上，RISC指令集需要ARM其商业授权。这种指令集的特点是每一条指令都足够简单，并且指令间互相没有功能重合。

Intel发家史：推出x86指令集，跟微软一起拿下PC市场
ARM发家史：退出ARM指令集，灵活授权，跟德州仪器杀入手机功能机市场，转Android市场
苹果的选择：Intel没有答应苹果的定制要求，所以苹果选择了ARM指令集。
因此，ARM的软件是分裂成Android和Apple两部分的

另外，新晋ISA：RISC-V，是开源授权指令集。不像商业授权，RISC-V是可以免费使用的。RISC-V发源于UC Berkeley。发展路线跟ARM相似，都是从低端向高端发展。RISC-V的弱点是无法维持稳定的公版，导致软件和编译器兼容性不稳固。RISC-V的标准现在由RISC-V国际基金会这个组织来管理。虽说RISC-V的RISC-V授权是免费的，但如果使用其衍生品(比如阿里平头哥的设计)，还是要付费的。  




## 架构和操作系统的关系
所谓操作系统，是指windows, linux等等   
任何操作系统，都可以运行在不同架构上。比如Windows即可以运行在x86架构上，也可以arm架构上  
arm架构上，也可以运行Windows或Linux操作系统  
但从市场占有率上看，大部分Windows系统（家用电脑）都运行在x86架构上；大部分嵌入式Linux系统（移动设备，比如android）都运行在arm架构上；大部分服务器Linux都运行在x86架构上。这是历史的原因造成的。  



## 32bit ARM(arm, armv6, armv7)
若要为此系统编译，需要的编译器通常有如下命名：  
gcc-arm-linux-gnu   



## 64bit ARM(aarch64, arm64)
若要为此系统编译，需要的编译器通常有如下命名：  
gcc-aarch64-linux-gnu  


## x86_64
x86一开始是32bit，后来扩展成x86_64就是64bit了。因此x86_64也是兼容32bit的。  
若要为此系统编译，需要的编译器通常有如下命名：  
gcc  






