# NVidia GPU Architecture
## CUDA Core
CUDA Core是指一个执行基础运算的处理单元。  
CUDA Core数量通常是对应FP32计算单元的数量。  

## RT Core
在消费级显卡里处理光线追踪所添加的核心。  

## Tensor Core
用于机器学习加速的运算核心。  
可以把整个矩阵都载入寄存器中批量运算，实现十几倍的效率提升。  

## 典型产品
A100(2020): CUDA Core=6912, Ampere架构  
H100(2022): CUDA Core=16896, Hopper架构  
L40S(2023): CUDA Core=18176, RT Core=142(212 TFLOPS), Tensor Core=568, Ada Lovelace架构  
H200(2024): CUDA Core=?, Hopper架构  


