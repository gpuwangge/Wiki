ARM=Acorn RISC Machine  
ARM是32位计算机，采用寄存器-寄存器型体系结构，使用Load/Store指令（间接寻址，不支持直接寻址）在存储器和寄存器之间移动数据。  

所有操作数都是32位宽除了几条乘法指令会产生64位结果并保存在两个32位寄存器中。  
16个可见寄存器，4位地址寻址。  
14个通用寄存器r0-r13。  
r13被用来保留用作栈指针。  
r14是特殊寄存器，存放子程序返回地址。  
r15是特殊寄存器，为程序计数器PC。  

## 常用ARM ISA List
```
Instruction               ARM Mnemonic      Definition
Addition                  ADD r0,r1,r2      [r0] ← [r1] + [r2]
Subtraction               SUB r0,r1,r2      [r0] ← [r1] - [r2]
AND                       AND r0,r1,r2      [r0] ← [r1] · [r2]
OR                        ORR r0,r1,r2      [r0] ← [r1] + [r2]
Exclusive                 OR EOR r0,r1,r2   [r0] ← [r1] ⊕ [r2]
Multiply                  MUL r0,r1,r2      [r0] ← [r1] × [r2]
Register-to-register move MOV r0,r1         [r0] ← [r1]
Compare                   CMP r1,r2         [r1] - [r2]
Branch on zero to label   BEQ label         [PC] ← label (jump to label)
MLA r0, r2, r1, r0                          ;乘累加 r0=r2*r1+r0
```

当前指令后面带s的时候，才会自动CCR。

反汇编：就是将汇编器产生的代码转换回汇编语言源程序。

## 汇编伪指令
汇编伪指令不是ISA的一部分，但程序员可以用。  
伪指令是一种便捷的形式，汇编器会把它转成实际指令，将程序员从一些事务性的工作中解脱出来。  

比如：
```
AREA       ARMtest, CODE, READONLY
ENTRY
...真实代码...
END
```

汇编伪指令EQU  
```
Tuesday      EQU     2      ;把字符串Tuesday绑定到数值2上
```
如果有指令ADD r1,r2,#Tuesday，则编译器视作ADD r1,r2,#2  

汇编伪指令DCD  
在程序运行前将数据提前载入存储空间，为常量和变量预留存储空间。  
DCB：一个8位字节  
DCW：16位的半字  

可以使用符号"="把字符串保存在存储器中。字符串后面还可以带有以逗号分开的其他字节值  
```
Mess1 = "This is message 1", 0
Mess2 = "This is message 2", 0
ALIGN                   ;伪指令ALIGN告诉汇编器下面不管是什么，都必须按照字边界对齐（32位）
```

