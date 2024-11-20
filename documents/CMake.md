# 背景知识
## 编译流程
**`预处理器： 源代码(.cpp)->文本文件`**    
宏定义替换  
头文件展开  
条件编译展开  
删除注释  
以下编译参数可以生成预处理后的结果(.i or .ii)  
> gcc -E

预处理阶段不进行语法检查。或者说预处理结束后的代码才是真正的源代码   

**`编译器：文本文件(.ii)->汇编代码`**    
把高级语言翻译为汇编机器语言  
以下编译参数可以生成预编译后的结果(.s)  
> gcc -S

编译器识别和检查不同的高级语言，并产生通用的输出语言  

**`汇编器：汇编代码(.s)->目标代码`**    
进一步把汇编语言翻译成二进制机器语言  
以下编译参数可以生成预编译后的结果(.o)   
> gcc -C

**`链接器: 目标代码(.o)->可执行程序`**    
分为三种情况  
目标是可执行文件：所有的二进制文件链接成一个可执行程序  
目标是静态库：把库文件加入可执行文件中，生成的可执行文件较大  
目标是动态库：不把库文件加入可执行文件中，生成的可执行文件较小。可执行文件需要在程序运行的时候链接动态库  

## C++编译的特点
每个源文件都是独立的编译单元，编译优化只能基于本文件内容。  
如果要进行跨单元优化，只能通过链接器。
C++对于虚函数的编译处理：如果类有虚函数，编译器给该类创建一个虚函数地址表。并且，类的每个对象都增加一个8字节(64位系统)的地址(相当于一个指针)，指向虚函数地址表。  
虚函数地址表里记录了该类和所有继承类的虚函数地址。  
当对象调用虚函数的时候，程序通过查看这个虚函数地址表找到正确的函数地址并执行。  

## 编译优化
编译器参数通过O0(默认，不优化)~O3进行不同程度的优化。任何优化都将带来代码结构的改变，比如分支的合并和消除，load/store才做的替换和更改等，使得调试信息丢失。  

直接使用g++命令行编译，如果文件数量很多，那么即使只改动其中一个，也需要全部编译。  
VSCode自带多文件编译系统，也就是task.json，但是用起来不够灵活，无法满足需求。  
对于结构简单的多文件项目，可以使用Makefile。但是，Makefile是平台相关的(比如mac的makefile放到windows下就会有问题)。  
对于复杂一些的多文件项目，使用CMake。而且CMake是平台无关的。  
注意安装CMake的时候，选择下载解压包然后放到c盘下。如果选择安装包，似乎会自动指定MS的编译器。  

## GNU计划
20世纪八十年代流行的Unix是一套广泛使用的商业操作系统。  
1983年，一些科技活动家发起了自由软件集体写作计划，计划目标是建立一套完全自由的操作系统，称为GNU(GNU is Not Unix)。  
GNU计划发布一系列自由软件，并且它们可以实现Unix系统的标准接口。但后来很多依此计划发布软件也支持Windows/Mac OS等系统。  
1990年，GNU计划开发发布的著名软件包括文字编辑器Emacs，C语言编译器GCC等等。  
1991年，完全自由的操作系统Linux诞生，虽然其并不是GNU计划的一部分。  

## GCC编译器和MinGW
GCC(GNU Compiler Collection)是GNU计划的关键一部分，也是GNU工具链的主要组成部分。GCC也是迄今为止诞生的最大的自由软件。  
最早GCC只支持C语言(gcc)，后来也可以编译C++(g++)。  
MinGW(Minimalist GNU for Windows)是将GCC编译器移植到Windows平台的产物。  
MinGW编译的时候使用了Windows的C运行时库，因此其开发的程序可以直接在Windows下运行。  

# CMake使用步骤
目标： 运行一个基本的编译命令生成exe   
**`1. 安装CMake`**  
https://cmake.org/download/  
https://cmake-3.27.0-windows-x86_64.zip/  
下载后解压复制到C盘根目录下, 把C:\cmake-3.27.0-windows-x86_64\bin添加到Path环境变量中   
验证安装CMake的cmd命令(此cmd仅在windows prompt下生效):
> where cmake

编写基本的CMakeLists.txt(参考下面CMake编程入门章节)。  
建立build文件夹并进入。  
运行cmake的命令:  
> cmake ..

如果要指定编译器：  
> cmake -G "MinGW Makefiles" ..

(这一步会在build文件夹下生成makefile)  
安装编译器(编译器bin目录会被放入环境变量)  
验证安装了编译器的cmd命令： 
> where gcc

进入编译器bin目录，重命名mingw32-make.exe为make.exe  
验证make设置好的cmd命令：
> where make

验证make安装好的另一个cmd命令:   
> make -v

这时候在build目录下运行make，将编译生成exe文件。  
以及在一些目录下生成obj文件。并且对每一个cpp文件都生成单独的obj文件
自己写的.h文件不需要写在CMakeLists.txt里，也不必指定其目录  
如果只修改了某一些.cpp或.h文件，make就会只重新生成相关的.o文件，并连接成exe
记得在make之前先保存所有文件。

**`2. 编译`**  
如上所述，使用make命令就可以编译  
修改源文件后需要重新编译  
修改了cpp文件后，只要文件夹架构没有改变，就不用重新运行cmake命令，而可以直接make。  
在main.cpp中加入了 #include<stdio.h>，也不用cmake，就可以默认找到这些头文件。  

**`3. 使用外部include和lib`**    
CMakeLists.txt下面添加如下代码  
> include_directories(E:\\GitHubRepository\\somefoldername)  
> link_directories("/home/server/third/lib")  

**`4. 如何给make传递参数`**    
CMakeLists.txt code:
```cmake
if(SINGLE)  
    add_executable(simpleTriangle samples/simpleTriangle.cpp)  
else()  
    aux_source_directory(${PROJECT_SOURCE_DIR}/samples SRC)  
    foreach(sampleFile IN LISTS SRC)  
        get_filename_component(sampleName ${sampleFile} NAME_WE)  
        add_executable(${sampleName} ${sampleFile})  
    endforeach()  
endif()
```
Call CMakeLists.txt  
> cmake -G "MinGW Makefiles" -D SINGLE=true ..

**`5. 其他`**    
目前有个问题是，同一段CMakeLists，在laptop Windows环境下生成name.lib，在desktop Windows环境下上生成libname.a。在VSCode terminal和windows command prompt里面都试过。编译器都是MinGW。  
另外，不管名字是什么，  
> target_link_libraries(xxx name)

这种写法都能成功link到lib。并在test环境下都能成功生成exe。所以这个问题并不影响。  

CMake命令，比如target_link_libraries()的参数是存在依赖关系的。比如  
target_link_libraries(A,B,C)里：  
A依赖B+C, B依赖C。  
(A,C,B)会出现B链接失败的错误。  


# Makefile介绍
https://www.bilibili.com/video/BV188411L7d2/?spm_id_from=333.337.search-card.all.click&vd_source=e9d9bc8892014008f20c4e4027b98036  
## Makefile的作用  
假设一个项目工程有如下源文件  
functions.h //内含数学函数和打印函数的定义; #ifndef防止重复include  
factorial.cpp //#include "functions.h"。内含数学函数的实现  
printhello.cpp //#include "functions.h"。内含打印函数的实现  
main.cpp  //#include "functions.h"。main函数里调用数学和打印函数  
该如何编译呢:  
> g++ main.cpp factorial.cpp pringhello.cpp -o main  

现在问题是，在实际的项目中，往往不止三个源文件。使用这种原始的编译方法将导致编译命令无比坑长；并且如果只改动了其中一个源文件，重新编译的时候需要把所有源文件重新编译一次，浪费时间。  

### 解决办法A：逐个编译  
> g++ main.cpp -c  

这个命令只编译，不链接。结果是产生了main.o文件。同理：  
> g++ main.cpp -c

生成了三个.o文件，然后：  
> g++ *.o -o main  

这样就把所有的.o文件链接到了一起。  
这样带来的问题是，手动输入这些命令非常麻烦。  

### 解决办法B：用脚本实现逐个编译  
建立一个Makefile文件，写下如下脚本：(hello目标依赖于三个源文件)  
```Makefile
hello: main.cpp printhello.cpp factorial.cpp  
    g++ -o hello main.cpp printhello.cpp factorial.cpp
```
如何运行：  
在Makefile目录下，输入make加回车。  
这时候会寻找hello这个文件。如果它不存在，就编译生成它；如果hello存在，就比较hello生成的时间和依赖的三个源文件的生成时间：如果有任何源文件比hello更新，就重新编译hello，否则，啥也不做。  
这个方法的问题是编译时间依旧很长。  

### 解决办法C  
在方案B的基础上，如果源文件更新了，单独编译这个源文件，生成新的目标文件(.o)，然后再链接生成新的hello。  


# CMake编程入门
建立文件CMakeLists.txt  
一共三行：  
```cmake
cmake_minimum_required(VERSION 3.10)
project(hello)
add_executable(hello main.cpp factorial.cpp printhello.cpp)
```
如何运行CMake:  
> cmake .

运行结果, 生成了:  
CMakeFiles目录  
MakeFile  
结论：CMake帮我们生成了MakeFile  
这时候再用
> make

运行编译链接  
好处是CMakeLists.txt可以放到不同的操作系统上，自动生成对应系统和编译器的MakeFile(有跨平台特性)  
一个小窍门：  
一般而言运行cmake .会生成一大堆文件跟源文件混在一起。为了避免这种污染，一般手动建立一个空的build文件夹(想叫别的名字也可以)，然后执行：  
> cd build   
> cmake ..

(..就是往上一层目录里找文件)
现在，所有的中间文件都生成在build里面了


# VS Code环境下与CMake的配合开发环境
https://www.bilibili.com/video/BV13K411M78v?p=2&vd_source=e9d9bc8892014008f20c4e4027b98036   
在VS Code环境下编译项目有两种方法：1、不使用VS Code配置文件，就像上文所说，编写一个CMakeLists.txt，然后在VS Code terminal里面敲出CMake命令即可；2、使用VS Code配置文件，本章讨论这两种情况。  

## 方法一：使用CMake+VS Code
需要预先安装VS Code, mingW64(包含gcc/g++编译器)和CMake  
安装VS Code的Cmake插件：  
cmake (optional，cmake语言提示插件)   
cmake tools  (optional，cmake扩展支持)  

在VS Code下使用CMake手动编译多文件项目  
-编写一个简单的CMakeList.txt
```cmake
cmake_minimum_required(VERSION 3.10)
project(MyZoo)
add_executable(my_cmake_zoo main.cpp ClassAnimal.cpp)
```
Ctrl+Shift+p打开搜索，选择CMake: Configure, 再选择GCC 8.1.0  
(这时候会出现一个build文件夹。但其实这个build文件夹删除自己再建个空的也可以)  
> cd build  
> cmake ..

(生成Makefile)
> mingw32-make.exe      

(真正的编译源文件为obj，并链接成my_cmake_zoo.exe。以后每次更新了源文件，执行这个就可以了。)  
Issue：如果系统安装了Microsoft Visual Studio，则cmake ..有可能优先调用cl.exe (特点是在build文件夹里生成了MyZoo.sln项目文件)  
Solution: 使用
> cmake -G "MinGW Makefiles" ..

这个指令强制使用MingW的gcc编译器。只是第一次用这个命令指定，以后cmake ..即可  

## 方法二：不使用CMake，仅使用VS Code配置文件tasks.json
参考：https://github.com/gpuwangge/Wiki/edit/main/documents/VSCode.md    
不过一般不这么用，毕竟CMakeLists.txt还是比tasks.json功能强太多了  


# CMake参数
## --build
以下两条指令等价：
> cmake --build .

> make

make原理：代码变成可执行文件的过程叫做编译(compile)；编译的(顺序)安排叫做构建(Build)。make是构建的工具。构建的方法写在makefile里。  
前面说过，makefile是平台相关的文件，因此makeu也是平台相关的构建方法。  
那么，有没有平台无关的构建方法呢? cmake提供了--build参数作为跨平台的构建解决方案。  
举例来说，如果使用make来构建，语句如下：  
> make < targetName >  

如果用msbuild来构建：  
> msbuild /t:$< targetName >  

如果用xcodebuild来构建：  
> xcodebuild -target < targetName >  

多种构建方式用起来很麻烦，于是cmake统一成了如下格式：  
> cmake --build . --target < targetName >  

是不是很方便呢？当然如果确定使用make来编译，直接用make指令就好了。  

## -B
指定build的目录  
举例：  
> cmake -B build/release

## -DCMAKE_BUILD_TYPE
设置编译类型。具体类型跟编译target有关。比如Release, Debug, ReleaseWithDebInfo等等  
举例：  
> cmake -DCMAKE_BUILD_TYPE=Release  

## -DCMAKE_CXX_FLAGS
将额外的参数传递给编译器。比如-m32或-m64来实现32位和64位编译。如果未指定，将使用工具链原生默认参数。  
举例：  
> cmake -DCMAKE_BUILD_TYPE=-no-pie  

这里pie的意思是position independent executable。这个设置可以enable Address Space Layout Randomization(ASLR)。  
当ASLR被打开的的时候，kernel会把binary/dependencies load进随机的虚拟内存位置。  

### Reference
https://www.redhat.com/en/blog/position-independent-executables-pie  


# Make参数
## -j  
以下参数用于并行编译，一般用在服务器上发挥多核性能优势。该参数不需要把参数写死  
> make -j  

编译VulkanPlatform测试  
1、CPU: 11th Gen Intel Core i5-1145G7 @ 2.60GHz (x8核)  
make: 4'43"  
make -j: 1'32"  
2、CPU: Intel Core i7-8700K CPU @ 3.70GHz (x12核)  
make: 1'59"  
make -j: 24"  

## -C
指定一个目录。在读取Makefile之前，先进入到目录DIR中，用法如下：  
> -C DIR  

上述指令和以下两条指令分别等效：  
> -c DIR  
> -directory=DIR  

举例：  
> make -j -c build/release

## clean
清除上次的make命令所产生的object文件（后缀为“.o”的文件）及可执行文件  
举例：  
> make -C build clean  


## all
该参数用于build整个项目。  
一个项目可以包含很多源文件和库。all参数可以一次全部编译。  
在Makefile文件中定义target。all是其中一个target。通常它的形式如下：
···
all: target1 target2 target3
target1:
gcc -c target1.c
target2:
gcc -c target2.c
target3:
gcc -c target3.c
···

举例：  
> make all  






