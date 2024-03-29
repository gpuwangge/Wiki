# Memory存储原理
一个内存地址可以存储1个字节，或8位(bit)的数据。  
原因是内存物理结构中的最小单元就是8 bit，因此系统会以此为依据进行管理。  
一个内存条上面有若干个内存颗粒(chip)，每个颗粒上又有若干bank，每个bank上又有电容组成的二维矩阵。矩阵的每个元素就是内存的一个最小单位，其含有8个小电容。  

# Memory地址的表示
通常用16进制的数字组合起来表示内存地址。  
比如0x0001表示某一个内存地址。(上面存储的内容大小为1个字节)  
此外，该内存地址的寻址使用了16 bit(带宽=16，或者说是CPU用了16根地址线)。则说明在此系统下最大寻址空间为2^16=65536(个地址)。或65536/1024=64kb个地址。  

在32位系统下，寻址空间为2^32=4194304kb=4096mb=4gb个地址。  
意味着该系统的CPU需要至少32根地址线。需要8个hex数字才能表示一个地址。但每个地址存储的数据依旧是1个字节。  
举例：0x00000001  

在64位系统下，寻址空间为2^64=17179869184gb个地址。(基本可以理解为用不完了)  
同样，该系统的CPU需要至少64根地址线。需要16个hex数字才能表示一个地址。但每个地址存储的数据依旧是1个字节。  
举例：0x 00000000 00000001  

在64位系统下，CPU同时有8，16，32和64位寄存器  
以32位寄存器寻址为例，因为其含有四个字节，所以每个内存地址的间隔为4：  
0x 00000000  
0x 00000004  
0x 00000008  
0x 0000000d  
0x 00000010  
...

在内存文件里，通常第一列是内存地址，以冒号区隔，冒号右边的是内存数据。  
以下表示在一个32位寻址系统中，每一行有16个内存地址，共存放16*8=128 bit的数据。  
这些数据以hex格式显示。一个hex需要4个bit存放，因此一共有128/4=32个hex数字，这些数字通常以8个(32位)为一组。  
04002000: 00000004 00000000 00000080 00000020  
04002010: 00000000 00000000 FFFFFFFF FFFFFFFF  
04002020: 00000600 00000000 00000000 00000000  
...

如果要往某个内存地址存数据，首先要获得内存地址的值，然后还有数据宽度(比如8, 16, 32或64位)。  

# Virtual Memory虚拟内存
## 虚拟内存原理
操作系统处理内存的时候，使用虚拟内存技术。采用这种技术时，每个进程仿佛自己独享一片2（N次方）字节的内存，其中N是机器位数。例如在64位CPU和64位操作系统下，每个进程的虚拟地址空间为2（64次方） Byte。  

这种虚拟地址空间的作用主要是简化程序的编写及方便操作系统对进程间内存的隔离管理，真实中的进程不太可能（也用不到）如此大的内存空间，实际能用到的内存取决于物理内存大小。  

由于在机器语言层面都是采用虚拟地址，当实际的机器码程序涉及到内存操作时，需要根据当前进程运行的实际上下文将虚拟地址转换为物理内存地址，才能实现对真实内存数据的操作。这个转换一般由一个叫MMU（Memory Management Unit）的硬件完成。  

## Page(页)
在现代操作系统中，不论是虚拟内存还是物理内存，都不是以字节为单位进行管理的，而是以页（Page）为单位。一个内存页是一段固定大小的连续内存地址的总称，具体到Linux中，典型的内存页大小为4096 Byte（4K）。4096是2的12次方。  

所以内存地址可以分为页号和页内偏移量。下面以64位机器，4G物理内存，4K页大小为例，虚拟内存地址和物理内存地址的组成如下：52bit的页号（高地址），跟12bit的页内偏移量（低地址）。  

## Page Fault
我们知道一般将内存看做磁盘的的缓存，有时MMU在工作时，会发现页表表明某个内存页不在物理内存中，此时会触发一个缺页异常（Page Fault），此时系统会到磁盘中相应的地方将磁盘页载入到内存中，然后重新执行由于缺页而失败的机器指令。这个机制把内存分为mapped和unmapped两部分。  

## 内存排布
以Linux 64位系统为例。理论上，64bit内存地址可用空间为0x0000000000000000 ~ 0xFFFFFFFFFFFFFFFF，如前所述这么大的空间其实根本用不完的，Linux实际上只用了其中一小部分（256T=2^8*2^40 =2^48）。(T=>G=>M=>K)   
Linux64位操作系统仅使用低47位，高17位做扩展（只能是全0或全1）。  
最高17位为全0的时候，低位2^47是128T大小用作User Space。  
最高17位为全1的时候，同样有2^47高位128T大小作为Kernel space。  
所以，实际用到的地址为空间为   
0x0000000000000000 ~ 0x00007FFFFFFFFFFF    
和    
0xFFFF800000000000 ~ 0xFFFFFFFFFFFFFFFF    
其中前面为用户空间（User Space），后者为内核空间（Kernel Space）。   

User Space内又分为(地址由高到低)  
Stack：栈  
Mapping Area：mmap系统调用相关的区域？  
Heap：堆  
BSS：未初始化过的全局变量  
Data：初始化过的全局变量  
Code: 编译之后的可执行机械代码指令  




