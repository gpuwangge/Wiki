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
SPIR-V是IR(中间)文件  
- OpLoad: 从memory中加载data到reg  
- OpStore：从reg存data到memory  
作为对比，在传统汇编中，memory和reg的数据传输使用MOV实现(CISC, x86架构)或LOAD/STORE实现(RISC, ARM/MIPS架构)
```
%result = OpLoad %type %pointer
OpStore %pointer %value
```
上述代码中，%result是一个变量，用于存储从内存中加载的数据，相当于reg  
%type是数据类型  
%pointer是一个指向内存的指针  
通过OpLoad指令把内存的数据存到%result里了  
%value是要存储的数据  
通过OpStore把数据存到内存里了  
- OpIAdd, OpFAdd: 整数，浮点数加法，对应ADD  
- OpISub, OpFSub: 整数，浮点数减法，对应SUB   
- OpIMul, OpFMul: 整数，浮点数乘法，对应MUL  
- OpIDiv, OpFDiv: 整数，浮点数除法，对应DIV   
```
%result = OpIAdd %int %a %b
```
OpAccessChain用于指向数组中的一个指针
```
%result = OpAccessChain %type %base %index1 %index2 ...
```

## spir-v格式，反汇编后出来的opcode是汇编吗
算是汇编，但不是 GPU 的原生汇编。准确说，spirv-dis 输出的是 SPIR-V 的文本汇编表示；SPIR-V 本身是中间表示（IR），不是 GPU 硬件直接执行的机器指令。  
```
GLSL / HLSL                 高级着色语言
    ↓ 编译
SPIR-V 二进制 (.spv)        跨厂商的中间表示
    ↕ spirv-dis / spirv-as
SPIR-V 文本汇编 (.spvasm)   同一中间表示的可读形式
    ↓ GPU 驱动编译、优化
GPU 原生机器码/原生汇编      硬件实际执行的指令
```
spirv-dis 将二进制转换成人类可读、可解析的文本；spirv-as 则可以把文本重新组装成 SPIR-V 二进制。因此叫它“SPIR-V 汇编”是合理且准确的，但不能直接把它当成某款 GPU 的 ISA 汇编。  

## Vulkan Shader编译使用的工具链
安装好Vulkan SDK后，相关工具可以在安装目录Bin/下面找到  
glslc.exe可以把vulkan shader(glsl)转换成spir-v格式  
```
glslc.exe shader.comp -o shader.spv
```
spirv-dis.exe可以把spir-v转换成spir-v汇编语言  
```
spirv-dis.exe shader.spv -o shader.asm
```

## 如何获得GPU 原生机器码/原生汇编 
需要通过 GPU 驱动编译后的 pipeline 或厂商工具获取  
Vulkan 提供了查询编译产物的扩展，但是否能拿到最终机器码或原生汇编，取决于驱动支持和它愿意暴露哪些内容。  
VK_KHR_pipeline_executable_properties 可以暴露文本或二进制形式的内部表示，其中可能包括最终 shader 汇编、编译后的 shader 二进制或中间 IR；它不保证一定返回机器码。  

### 最方便：用图形工具
| 场景             | 工具                       | 能做什么                                                                     |
| -------------- | ------------------------ | ------------------------------------------------------------------------ |
| 通用 Vulkan 调试   | RenderDoc                | 驱动支持相应扩展时，可以检查 pipeline 的内部表示，例如原生汇编或驱动 IR。blogs.igalia                  |
| NVIDIA GPU     | Nsight Graphics          | Shader Profiler 支持 Vulkan，并提供底层 shader 汇编关联；具体显示粒度取决于版本和 Pro 功能权限。nvidia |
| AMD GPU，实际运行分析 | Radeon GPU Profiler（RGP） | 查看 shader ISA 及指令级时序，分析真实运行中的热点。gpuopen                                  |
| AMD GPU，离线分析   | Radeon GPU Analyzer（RGA） | 对 Vulkan、SPIR-V 等进行离线编译与性能分析。gpuopen                                     |

如果是 Imagination PowerVR GPU，官方有工具可以输出原生 USC 汇编，而且支持把 SPIR-V 作为输入。不过，要区分“离线编译得到的原生汇编”和“设备上 Vulkan 驱动实际生成的汇编”。

Imagination 官方文档列出了两类工具：
| 工具                          | 用途                               | 适用范围                                                                              |
| --------------------------- | -------------------------------- | --------------------------------------------------------------------------------- |
| PowerVR Profiling Compilers | 离线编译 shader，输出 USC 反汇编和逐行周期分析    | 文档列出 Rogue（Series 6、7、8、9）和 Volcanic（Series B、C），支持 SPIR-V 二进制、GLSL、OpenCL kernel |
| PVRShaderEditor             | GUI 中查看、分析编译后的 USC 汇编，包括 FP16 指令 | 文档明确提供 PowerVR Rogue 的 shader 反汇编                                                 |

对于任何gpu，有没有可能通过spir-v获得gpu原生asm后，修改一下，再让gpu运行？特定 GPU 和驱动上有可能，但没有适用于所有 GPU 的通用方法。  



