# Assembly Language

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

写读相关(write/read dependency)：对同一内存地址进行先写后读，那么读就必须等待写完成之后。所以必须尽量避免写读相关的出现。  

局部变量是使用寄存器的。(全局变量是使用内存的)因此，应该减少全局变量的访问次数，用局部变量取代之。  


## Reference
https://zhuanlan.zhihu.com/p/352698929  

