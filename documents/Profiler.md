# Background Knowledge
Notation 1e9: 10^9  

## Precision 
Defined in IEEE 754  
- single-precision: 32 bits(1 sign bit + 23 fraction bits + 8 exponent bits)  
- half-precision: 16 bits(1 sign bit + 10 fraction bits + 5 exponent bits)  
- double-precision: 64 bits(1 sign bit + 52 fraction bits + 11 exponent bits)  
- int32: 32 bits, -2147483648 ~ 2147483647  
- int16: 16 bits, -32768 ~ 32767  
- int64: 64 bits, -9223372036854775808 ~ 9223372036854775807  

## Time
1 second = 1e3 millisecond = 1e6 microsecond = 1e9 nanasecond  

## FLOPS
FLOPS = floating-point operations per seconds  

kiloFLOPS = 1000  
megaFLOPS = 1000,000  
gigaFLOPS = 1000,000,000 (10^9, or 1e9)  
teraFLOPS = 1000,000,000,000 (10^12, or 1e12)  
petaFLOPS = 10^15  
exaFLOPS = 10^18  
zettaFLOPS = 10^21  
yottaFLOPS = 10^24  

Example  
numOps: total number of operations of FMA  
time: unit is second  
gflops = numOps * 2 / time / 1e9  

How to know if a fma test is running at peak  
减压法  
一段shader program计算fma，numOps总量是知道的  
运行完毕后统计时间time，就可以算出gflops_1  
这个值有可能是极限算力(peak)的结果，也可能不是(还有多余算力)  
现在把fma换成fmul，运算量降低一半，重新计算gflops_2  
如果之前是peak，那么time也降低一半，gflops_2==gflops_1  
如果本来就不是peak，time不变，gflops_2明显比gflops_1低  




