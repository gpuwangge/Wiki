# 汇编基础 
计算机只会做加法：乘法x*y就是y个x相加，除法本质是减法  
机器语言是位运算，都是电路实现的，是计算机最底层的根本。  
通过机器语言来实现加法器，设计电路。（设置开关）  
通过指令来代替二进制编码，就是汇编语言。需要编译器来吧汇编语言转成二进制代码。  
(学C的正确方式：Linux, vim, gcc)  
学汇编之前，先学汇编环境  
32位和64位汇编架构没区别，区别是寄存器和寻址能力。32位系统最大寻址是4gb空间。  
学习汇编的三个内容  
**`1、通用寄存器`**  
**`2、内存`**  
**`3、汇编指令`**  

# 通用寄存器
电脑存储数据的方法：CPU（有寄存器，8,16,32位或8,16,32,64位） > 内存 > 硬盘  
32位通用寄存器只有八个：EAX, ECX, EDX, EBX, ESP, EBP, ESI, EDI, 存值范围0~FFFFFFFF  
16位通用寄存器名字就是把E去掉，所以是八个: AX, CX, DX, BX, SP, BP, SI, DI, 16位寄存器操作的时候是只能对32位寄存器的低16位进行操作  
8位通用寄存器八个： AL, CL, DL, BL, AH, CH, DH, BH， 16位寄存器操作的时候也是对32位寄存器的低16位进行操作，但只能操作AX, CX, DX, BX  
计算机如何往寄存器里存值：二进制直接修改值。  
汇编怎么存寄存器：mov指令  
mov 存的地址(寄存器名字)，要存的数  
mov 存的地址(寄存器名字)，要存的数的地址(寄存器名字)  

非通用寄存器，比如EIP，你不能改它的值。除了通用寄存器，其他寄存器都有自己特定的功能。  

# 内存
寄存器很小，不够用，所以要把数据放到内存中。  
每个32位应用程序都有4gb内存空间(是虚拟内存)  

虚拟内存可以链路到物理内存  
程序真正运行的时候才会用到物理内存  
1B = 8bit  
1KB = 1024B  
1MB = 1024KB  
1GB = 1024MB  
4G内存 = 4096MB  

## 内存地址
存一个数：占用的大小，数据宽度，存到哪里？  
计算机内存地址很多，空间很大，每个空间分配一个地址，名字  
这些给内存起的编号，就是内存地址  
32位：寻址能力是4gb  
32位是8个16进制的值，最大的值：FFFFFFFF+1=100000000  
100000000个内存地址，100000000*8=800000000bit，换成10进制，除以8=4294967296B  
4294967296/1024/1024/1024 = 4GB  
而64位绰绰有余。  

每个内存地址都有一个编号，可以通过编号往里面存值。  

## 内存如何存值
需要两个信息  
内存宽度：byte word dword  
地址位置：比如0xFFFFFFFF  
不是任意的地址都可以写东西，应用程序需要先申请使用。  
以下指令往内存地址写了1  
> mov byte ptr ds:[0019FF70],1  
> mov dword ptr ds:[0x0019FF74], 1  

语法：  
> mov byte/word/dword/qword, ptr ds:[0x19FF70], 0x1  


## 内存地址的多种写法
**`内存地址偏移`**  
ds:[0x19FF70+4]  
**`寄存器`**  
ds:[eax]  
**`寄存器偏移`**  
ds:[eax+4]  
**`数组[]`**  
ds:[reg+reg*{1,2,4,8}]  
ds:[reg+reg*{1,2,4,8}+4]  


# 汇编指令
## LSC (Load Store Cache)
我们所有计算机程序的本质，是指令的执行过程，从根本上说，也就是处理器(CPU)的寄存器(Register)到内存之间的数据交互过程。  

这个交互，无非两个步骤:  
读内存，也就是加载操作(load), 从内存读到寄存器(CPU)  
写内存，就就是存储操作(store)，从寄存器(CPU)写入到内存  
（总结: Load/Store是以register(CPU)作为主体的操作）  

CPU从register读数据，只要0(?)个cycle(CPU周期)；而从内存读，需要几百个cycle。  
为了解决这个问题，引入了高速缓存(Cache)，避免CPU和内存直接交换数据。  
Cache分为很多层级。  

### LSC的优化
为了减少load/store的次数，尽量只使用register，避免使用内存。  

Store Unit: 含有数据和它的机械地址。进行Store/Load操作之前，直接去看Store Unit，如果有，就不必进行Store/直接进行Load，从而避免访问内存了。  
(这个Store Unit就是Cache)  

Cache分为L1, L2, L3等等级。CPU读取的时候依次从L1, L2, L3中获取，找不到采取内存读取。  
其实cache不一定非要三级，但是现代CPU架构一般采取三级，L1在CPU内部供每个CPU核心使用，L2也在CPU内部供所有CPU核心使用，L3在CPU外部但接近内存。  
(也有L1和L2同时供CPU核心使用的设计)  
从L3到L1，速度变快，成本也变高。  
寄存器则是位于CPU核心和L1之间。  

写读相关(write/read dependency)：对同一内存地址进行先写后读，那么读就必须等待写完成之后。所以必须尽量避免写读相关的出现。  

局部变量是使用寄存器的。(全局变量是使用内存的)因此，应该减少全局变量的访问次数，用局部变量取代之。  

通常我们写代码的时候说的“变量a保存在内存”，严格来说不是真的内存memory，而是缓存Cache。  
内存是DRam，缓存是SRam(结构更复杂)。  
内存条如果严格命名，应该叫"外存"，因为其位于CPU的外部。  

硬板读写速度约4Gb/s，内存可以做到40Gb/s，缓存比内存还要快，L1可达4000Gb/s, L2 1000Gb/s，L3也有200~300Gb/s。  
内存的size一般为Gb级，而L1只有约48Kb，L2是512Kb, L3也只有区区16MB。  

# 指令的执行
指令的执行有四个周期
- Fetch Cycle：从内存中去除下一条要执行的指令(从程序计数器PC找到下一条指令，并存到指令寄存器IR中，然后更新PC指向下一条指令)
- Decode Cycle：对取出的指令进行译码，确定指令的操作类型和操作数(IR中的指令送到控制单元。如果需要操作数，计算操作数的地址)
- Execute Cycle：根据译码结果执行指令的操作(使用ALU执行操作)
- Store Cycle：将执行结果存储到指定的存储位置(结果写入目标寄存器或内存)


# Reference
https://zhuanlan.zhihu.com/p/352698929  

