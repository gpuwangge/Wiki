# Computer Architecture

Architecture，架构，也俗称平台。比如arm架构，也叫arm平台。  

首先，在Linux下要获得当前系统的构架信息，可以使用如下命令：  
> uname -m

结果可能是如下值  
x86_64  (32 and 64 bit)  
aarch64 (64 bit ARM)    
armv71 (32 bit ARM的一种)   

当编译完成后，在linux系统下也可以用file命令查看编译的可执行文件（或lib）匹配哪一个架构  


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
任何操作系统，都可以运行在不同架构上。比如Windows即可以运行在x86架构上，也可以运行在arm架构上  
arm架构上也可以运行Windows或Linux操作系统  
但从市场占有率上看，大部分Windows系统（家用电脑）都运行在x86架构上；大部分嵌入式Linux系统（移动设备，比如android）都运行在arm架构上；大部分服务器Linux都运行在x86架构上。这是历史的原因造成的。  

### Andoird操作系统是不是就是Linux?Android是否使用arm架构？
首先，linux系统(发行版)和linux内核(kernel)是不同的概念。linux发行版包含Linux内核以及一系列系统工具软件。  
linux系统(发行版)举例：Debian, Ubuntu, Mint, RedHat...  
通常，如果不考虑上下文，linux是指linux系统(发行版)。但是有时候也指内核，因为linux内核刚好也叫linux。具体语义要看上下文而定。  
Android是一个基于linux内核的操作系统。广义上也算是linux系统(发行版) 。Android的特点是linux内核+ART(Android Run Time)，android的app都是运行在ART上面的。换句话讲，非android的linux发行版，只要装了ART，就可以作为Android模拟器使用，运行android app。  
大多数android系统采用arm架构

### XBox和Playstation采用的是什么系统和架构？
XBox使用Windos NT内核   
Playstation采用FreeBSD内核  
以上都是采用x86_64架构的  
任天堂的主机switch则采用FreeBSD内核和arm架构  


## ARM架构
早期ARM架构ARMv7支持32位空间和32位运算，指令定长32位  
2011年，ARM发布了ARMv8-A架构添加了对64位空间和64位算术运算支持。ARMv8有两种执行模式(execution mode): AArch64和AArch32。具体处理器包括Cortex-A35, Cortex-A50系列, Cortex-A72, Cortex-A73  
2021年，ARM发布了ARMv9。  

### Apple A处理器架构
Apple A处理器均采用ARM架构，被用于iPhone, IPad, Apple TV等产品上  
A1 ~ A2: ARMv6  
A3 ~ A5: ARMv7  
A6: ARMv7s (2012)  
A7 ~ A9: ARMv8.0-A (2013 ~ 2015)  
A10: ARMv8.1-A (2016)  
A11: ARMv8.2-A (2017)  
A12: ARMv8.3-A (2018)  
A13: ARMv8.4-A (2019)  
A14 ~ A15: ARMv8.6-A (2020 ~ 2021)  
A16: ARMv8.6-A (September 7, 2022) (iPhone 14 Pro & 14 Pro Max, iPhone 15 & 15 Plus)  
A17 Pro: AArch64? (September 12, 2023) (iPhone 15 Pro & 15 Pro Max)

### Apple M处理器架构
2020年，Apple开展了把使用Intel x86-64处理器的Mac电脑换成ARM64的芯片架构迁移计划。这项计划的动机：一是不再需购买其他公司的昂贵CPU，二是可以在Mac上运行ARM架构的iOS应用程序。三是ARM架构一般功耗更低，有利于研发更加轻薄的Mac笔记本。  
Mac电脑在1994年前使用摩托罗拉68000架构，1994 ~ 2005年间使用PowerPC架构，2005 ~ 2020年间使用Intel x86架构。  
为了在新架构上兼容x86应用程序，Apple使用了动态二进制翻译技术。  
M1: (2020.11)  
M1 Pro; M1 Max: (2021.10)  
M1 Ultra: 应用于Mac Studio (2022.3)  
M2: 应用于Macbook Air (2022.6)  
M2 Pro: 应用于Mac Mini (2023.1), 应用于Macbook Air (2023.6)  
M2 Ultra: 应用于Macbook Pro (2023.6)  

### MTK处理器架构
Dimensity系列早期均采用ARMv8.2-A  
Dimensity 7200 (MT6886): ARMv9-A (2023)  
Dimensity 9000系列: ARMv9-A (2021~2023)  

### Qualcomm处理器架构
采用ARMv9-A的处理器包括：Qualcomm Snapdragon 7 Gen 1, 7+ Gen 2, 8(+)Gen 1, 8 Gen2等  

## 架构和编译器
若为不同架构编译程序，需要不同的编译器。  

### Native编译
一般来讲，开发的时候使用x86_64架构(无论是Linux或是Windows)，生成的应用程序也是x86_64架构的话，属于Native编译，通常编译器名字都很简洁，比如  
gcc  
编译完成的二进制文件在当前系统内可以立刻运行。  
值得提出的是x86一开始是32bit，后来扩展成x86_64就是64bit了。因此x86_64也是兼容32bit的。   

### Cross编译
Cross(交叉)编译，可以理解为在当前编译平台下，编译出来的的程序能运行在体系结构不同的另一种目标平台上，但是编译平台本身却不能运行该程序。    
交叉编译工具链，是为了编译跨平台体系结构的程序代码而形成的由多个子工具构成的一套完整的工具集。  

交叉编译的好处：目标平台运行速度比主机慢很多；编译过程非常消耗资源，目标平台往往没有足够的内存空间；  
就算目标平台资源足够，那在目标平台的第一个本地编译器也需要通过交叉编译获得；  
一个完整的编译需要很多工具支持，交叉编译可以避免把这些工具链都移植到目标平台上。  

具体来说，交叉编译涉及三个架构概念：  
Compiler Build(Build): Build编译器的平台  
Host: 运行编译器的平台  
Target: 运行编译器生成的应用程序的平台  
如果Build=Host=Target，称为Native Compile  
如果Build=Host!=Target，称为Cross Compile  
如果Build!=Host=Target，称为Cross Native Compile  
如果Build!=Host!=Target，称为Canadian Cross Compile  
如果Build=Target!=Host，称为Crossback Compile  

如上所述，如果要使Host=x86_64架构下编译的二进制文件运行在Target=arm架构，则意味它不能在原本的x86_64架构下运行。  
值得提出的是，如果目标平台是64位的，Host必须也是64位才行。  
不仅如此，编译器也要选择特殊版本才行。  
通常的交叉编译器工具商简单罗列如下  

#### Linaro
Linaro公司是2010年成立额非盈利公司，可免费提供如下交叉编译工具  
aarch64-linux-gnu-g++  
此工具能编译出运行在arm64架构linux系统下的c++程序。  
gnu表示库函数的规范，gnu=glibc+oabi, gnueabi=glibc+eabi  
glibc: 是GNU发布的libc库(c运行库)，包含linux系统最底层的api  
abi: application binary interface，嵌入式的二进制接口，是编译链接工具的基础规范，获知说是arm的系统调用方式   
oabi: old abi，旧标准的abi  
eabi: embedded abi，新标准的abi，效率比旧版的高  
除此之外，库函数规范名还包含hf后缀，意思是hard float，这种情况支持更快的浮点数计算，但是需要arm硬件有fpu才行    

若要安装该编译器，除了在Linaro官网下载外，还可以在Ubuntu下使用deb包管理器安装：  
> apt-get install g++-aarch64-linux-gnu

(apt-get命令是一条linux命令，适用于deb包管理式的操作系统(比如Ubuntu)，主要用于自动从互联网的软件仓库中搜索、安装、升级软件或操作系统)   
如果想看在软件仓库中有哪些aarch64编译器，可以用如下命令
> apt-cache search aarch64  

其它的Linaro编译器举例如下：  
arm-linux-gnueabihf-gcc   
编译arm32架构linux系统下的c程序，使用新版abi，开启hard float支持  



#### Codesourcery
Codesourcery是2005年成立的公司。已被Mentor收购，所以有的资料也称Mentor。其工具链同样免费，代表编译器如下：  
arm-none-linux-gnueabi-gcc  
同上，架构arm32位linux系统，glib+eabi, c程序  
none这里的意思是没有vendor(工具链提供商，比如Cortex之类的)  

如果连linux都省略了，那就是连操作系统都可以没有  
arm-none-eabi-gcc  
跟操作系统相关的函数都用不了了。这种arm架构称为bare metal arm(裸机arm系统)。它不能编译linux的application，除非你自己提供一些底层函数的实现，比如print()，内存管理等。  

#### ARM
ARM提供一些收费编译工具，比如  
armcc  
这个工具跟arm-none-eabi类似，可以编译裸机程序（u-boot, kernel），但不能编译Linux应用程序  

















