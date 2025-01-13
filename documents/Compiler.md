# 代码是如何使硬件工作的
通过C/C++语言生成的叫做高级代码。但是CPU能执行的只是二进制代码(即机器语言，全部由0或1组成)。  
需要通过编译来吧高级代码翻译成等效的二进制代码。  
编译或解释它们的操作方法不同，运行的效果就会不一样。(最终都是生成二进制代码)  
0或1的二进制代码，实际就是高或低的电平组合。  

有一些编译器把文件编译成了HEX文件，那么这个HEX文件是什么呢？  
HEX文件内部全部是十六进制代码。它跟二进制代码不等效。  
为什么不直接生成二进制代码，是因为HEX代码自带校验位。能为代码的传输存储带来便利。  
并且，相比较二进制代码，HEX代码也更加人类可读。  

CPU如何执行二进制代码：把CPU看作海量的开关(继电器)组合  
当输入不同的高低电平后，可以通过放大电路, A/D电路等，驱动屏幕、音响、电机等硬件工作  

# LLVM工具链介绍
LLVM是一个开源编译器框架，最早由Apple开发。  
Apple还开开发了Clang，作为编译器的前端(用来编译C/C++/OC)。  
Clang+LLVM的用途就相当于gcc。似乎在优化上比gcc强一些。  
Swift的编译器框架也是基于LLVM的。  
流程：  
**`1、Clang编译C/C++/OC成IR文件`**  
**`2、LLVM IR Linker链接IR文件`**  
**`3、LLVM backend 使用IR文件生成Assembly或Object Code`**  
(LLVM backend后端也称为LLVM核心)  

可以看出LLVM的中间文件是IR文件。  
IR文件有三种表示，它们完全等价：  
**`1、.ll：介于高等语言和汇编之间，人可以看懂`**  
**`2、.bc: bitcode，人不可以看懂`**  
**`3、内存格式，值保存在内存中，没有名字。不生成具体文件，所以这个方式最快`**  
IR文件因为是中间状态，默认是不生成的。但是可以用参数生成。  

.ll和.bc可以由llvm-as和llvm-dis做转换：  
llvm-as(llvm汇编器): .ll -> .bc  
llvm-dis(llvm反汇编器): .bc -> .ll   
举例：  
> llvm-dis out_fragmentShader.air -o test.ll

（在macOS里.air文件就是.bc文件）  

llc: 作为LLVM的后端，llc可以把bitcode转换为目标机器(本机器)的汇编码，需要指定架构。.bc -> asm  
举例：  
> llc -mtriple x86_64-pc-linux-gnu -o test.s out_fragmentShader.air  

(在macOS里.s就是asm文件)  

# 如何安装LLVM(llc,llvm-dis...)
进入WSL, 使用如下命令安装：
> sudo apt install llvm  

# MacOS环境下的Metal Shader编译举例
## 从.metal到.metallib
**`1、.metal -> .air`**  
air: 即bitcode文件(LLVM IR)  
工具：metal   
**`2、.air -> .metalar`**  
.metalair包含很多个.air文件  
工具：metal-ar  
**`3、.metalar -> .metallib`**  
工具：metallib  

## 从.metallib到asm
**`1、.metallib -> .air`**  
工具: unmetallib.py  
python3 unmetallib.py MyLibrary.metallib  
**`2、.air to asm`**  
工具：LLVM 6.0.0  
输入: .air文件  
Reference：
https://github.com/zhuowei/MetalShaderTools
https://worthdoingbadly.com/metalbitcode/

## 从.metal到asm
**`1、.metal -> .air`**  
需要macOS系统(xcrun is a tool in macOS)  
> xcrun metal -c xxx.metal -o xxx.air

如果遇到识别不了sqrt, sin等函数，在.metal里加上：  
#include<metal_stdlib>  
> xcrun -sdk iphoneos metal -c AAPLShaders.metal -o MyLibrary.air  

**`2、.air to asm`**  
需要WSL下面安装LLVM。  
（macOS也许也可以？待查证）  
llc -mtriple x86_64-pc-linux-gnu -o xxx.s xxx.air  
llc -mtriple arm64-none-linux-android -o xxx.s xxx.air  
(-mtriple参数指定LLVM target triple， 即arch, vendor and os)  

## 这里存在的问题
.air(bitcode)应该是平台无关  
llc生成的asm是平台有关的。尽管.air是从metal shader来的，但llc生成的instruction set是CPU的，因此不合适。  
解决办法是使用其他的LLVM backend，比如apple的内部工具，支持Apple GPU instruction的  


# Vulkan Shader编译相关
## Vulkan Shader格式(SPIR-V)和传统汇编语言的区别
Vulkan Shader采用了SPIR-V格式(为了GPU并行计算设计)，它使用跟传统汇编(为了运行CPU而设计)不同的指令集  
## Vulkan Shader编译使用的工具
## Vulkan Shader从高级语言生成SPIR-V汇编语言的流程










