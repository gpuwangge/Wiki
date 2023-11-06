# Scons
## 基本介绍
Scons是什么：Scons的工作跟make一样，但更简单，更容易。  
Scons特点：用Python脚本作为配置文件。支持C/C++。跨平台: Linux, Windows。  
Scons的结构：配置文件为SConstruct(使用Python脚本)，相当于Makefile。编译指令scons，相当于make指令。  

## Scons和CMake对比  
两者都是建造程序的工具，讷讷感自动检测文件之间的依赖关系，对于建造复杂程序很有帮助。  
CMake: 延续有比较长历史的传统工具链，虽然简化了，但仍旧生成Makefile, .sln或.xcodebuild文件，经由它们来调用编译器。CMake是一套专有的语言。  
Scons：放弃传统工具链，直接调用编译器。没有专门语言，直接使用Python。  

## 安装运行方法(Windows)：
1.Download SCons-4.5.2  
2.python setup.py install  
如何验证安装好了：  
> scons -v  
> scons -h  

3.写一个helloscons.c程序  
```
#include <stdio.h>
#include <stdlib.h>
int main(int argc, char* argv[]){
    printf("Hello, SCons!\n");
    return 0;
}
```
4.写一个SConstruct文件  
```
Program('helloscons.c')
```
5.运行如下程序  
scons  
6.生成的结果如下：  
.sconsign.dblite  
helloscons.obj  
helloscons.exe  

## 安装运行方法(Linux)：
pip3 install scons  
如何验证已经装好  
> scons -v  
> scons -h  



