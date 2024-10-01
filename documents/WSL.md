Windows从Windows 10开始引入了Windows Subsystem for Linux(WSL)  
WSL本质上是个虚拟机  

# 安装教程
**`1.Open Windows PowerShell in administrator mode`**  
**`2.Run command`**  
> wsl --install  

或  
> wsl --install -d Ubuntu  

(默认安装Ubuntu，但可以改)  
(It will show installing: Virtual Machine Platform; Installing Ubuntu)  
(This command will enable the features necessary to run WSL and install the Ubuntu distribution of Linux. )  
(The requested operation is successful. Changes will not be effective until the system is rebooted.)  
(卸载使用wsl --uninstall)  
**`3.重新启动后，继续自动安装Ubuntu`**  
**`4.按要求创建具有管理员权限（sudo）的账号(UNIX username)和密码`**  
> [username]  
> [password]  

(这时候pwd: /home/wangge)  
**`5.更新package`**  
> sudo apt update && sudo apt upgrade  

(到这一步安装就完成了)   
(左侧文件管理器会出现一个企鹅🐧图标的Linux，点击里面会发现Ubuntu文件夹)  

## 如何进入WSL
方法1：进入Powershell，键入WSL指令，就可以在蓝色背景的Powershell进行操作  
方法2：点击Start，点击WSL应用程序图标(是一个企鹅头)，会进入黑色背景的WSL Shell  

## Windows如何同WSL交换文件
### Windows下如何用文件管理器打开/home/wangge  
方法1：点击左侧企鹅，展开目录Linux->Ubuntu->home->wangge  
方法2：在地址栏输入  
> \\\wsl$

或者
> \\\wsl.localhost\Ubuntu\home\wangge

实际存储位置可能在
> user\AppData\Local\Packages\CanonicalGroupLimited.Ubuntu_79rhkp1fndgsc

**`如何创建某个盘(drive)的快捷方式：比如要创建\\wsl$，右键点击Ubuntu后，选择"Map network drive.."添加快捷方式`**  

## WSL下如何通过shell打开c、d...盘文件
> cd /mnt

在shell里试图运行.exe会先显示找不到xxx.dll的错误  
但是可以用vim打开文本文件，可以edit并保存  
windows也可以修改/home/wangge下面的文件  
但windows和wsl并不是同一个系统，所以如果要再wsl上运行的程序，还是要放在/home/wangge下面比较好，否则跨系统传输文件很没有效率  

**`如果删除了这个wsl2 Linux子系统,它里面的文件也就没了`**   

## 如何查看自己使用的wsl version是1还是2
在Windows PowerShell里面使用命令  
> wsl --list --verbose  

或  
> wsl -l -v

## 如何查看wsl默认安装设置
> wsl --status  

举例:
```
Default Distribution: Ubuntu
Default Version: 2
```

## 如何查看Ubuntu版本
wsl或ubuntu内  
> lsb_release -a

举例：
```
No LSB modules are available.
Distributor ID: Ubuntu
Description:    Ubuntu 22.04.3 LTS
Release:        22.04
Codename:       jammy
```
或
> cat /etc/os-release

## WSL升级
首先要处于Admin Group
> wsl --upgrade  


# 如何在WSL里编译Linux binary
以下指令可以在shell里打开vs code  
> code .  

但是光打开也没用，编译出来的还是windows .exe  

## VS Code使用WSL的正确方法：  
windows下打开VS Code，下载WSL插件。  
点击左下角链接WSL: Ubuntu. 选择Ubuntu里的文件夹。  
这个时候VS Code就正确运行在WSL里面了。  
但是这时候cmake是找不到的(因为ubuntu里面没安装吧)  
用如下指令安装cmake:  
> sudo snap install cmake 

(如果上面指令提示有安全问题，用下面的这个)
> sudo apt  install cmake  
 
查看cmake是否安装成功。
> cmake -version 

WSL Ubuntu自带gcc，但没有g++，使用如下指令安装g++:  
> sudo apt install g++

(这里编译出来的是默认的a.out文件)  
这时候使用cmake ..的话，会自动选择gcc和g++  
如果是从windows上移植过来的CMakeLists.txt，需要注释掉SET编译器的那两行。  
然后make指令可以生成没有任何后缀的可执行文件  
该可执行文件就是Linux可执行文件，可以在wsl shell里面运行。  

## 如何通过VS Code编辑器运行Linux binary
将希望运行的binary拷贝到WSL系统文件夹内，比如:  
> Z:\home\wangge\projects  

在windows下打开VS Code, 点击左下角链接WSL  
打开Terminal，运行binary  

如果遇到Permission denied问题,使用如下方法改变文件权限
> chmod u=rwx,g=r,o=r filename  

Here, each letter has a meaning:  
r gives read permissions  
w gives write permissions  
x gives execute permissions  

Linux/Unix的文件调用分为三级：依次为Owner(u), Group(g), Other Users(o)，各自有rwx属性：  
 u   g   o  
rwx-rwx-rwx  
只有文件owner和超级用户root可以修改文件或目录的权限  

常见权限列表：  
- 644 rw- r-- r--  
- 744 rwx r-- r--  
- 755 rwx r-x r-x  
- 777 rwx rwx rwx  

说明：r=4, w=2, x=1，所以7就是rwx; 同理6就是rw-; 5就是r-x; 4是r--  

其他使用方法举例  
以下指令把文件file.txt设置为所有人可读
> chmod ugo+r file.txt  
> chmod a+r file.txt

以下指令将目前目录下的所有文件与子目录皆设为任何人可读取
> chmod -R a+r *

以下指令将两个文件设置为ug科协，other user不可写  
> chmod ug+w,o-w file1.txt file2.txt  

以下指令将全部权限设置给ugo
> chmod 777 file.txt

以上指令也等价于以下指令  
> chmod a=rwx file.txt

常见的情况，某个文件我希望owner拥有rw权限, 其他人只能r，则设置成644  
> chmod 644 file.txt  

需要注意的是，chomod不但能该文件的permission，文件夹的也可以改。并且文件夹的permission也影响里面的文件。  
比如，文件有w权限，但文件夹没有w，那么这个文件也是无法删除的。  
以下命令可以把包括某个folder在内的的所有文件都改变权限：  
> chmod -R 777 folder/*  


## gcc和安装的g++是啥架构
x86_64：INTEL的64位指令集，常常简称x64。AMD64和它是一样的。这个构架也兼容32bit的软件。  
AArch64: ARM的64位指令集。不支持32bit。常常简称为ARM64 

使用如下指令：  
> g++ -v

wsl和SWRD上获得的信息：  
**`Target: x86_64-linux-gnu`**  

windows上获得的信息：  
**`Target: x86_64-w64-mingw32`**  

## 如何查看一个binary是什么架构
(在Ubuntu系统下)  
> file filename

(会打印出elf信息)  

## 如何编译出AArch64架构的binary
需要用Cross Compile技术   
**`1、安装aarch64版gcc(g++)`**  
> sudo apt install gcc make gcc-aarch64-linux-gnu binutils-aarch64-linux-gnu

或
> sudo apt-get install gcc-aarch64-linux-gnu  
> sudo apt-get install g++-aarch64-linux-gnu  

**`2、把以下增加到CMakeLists.txt里面`**  
set(CMAKE_C_COMPILER /usr/bin/aarch64-linux-gnu-gcc)  
set(CMAKE_CXX_COMPILER /usr/bin/aarch64-linux-gnu-g++)  

# 如何把代码从Windows上移植到WSL  
以OpenCL为例  
**`1.把整个项目拷贝到\home\wangge\coding\底下`**  
**`2.如上所述，把CMakeLists.txt里面SET编译器那两行去掉`**  
**`3.把\\改成/`**  
**`4.解决找不到CL/opencl.hpp的问题`**  
> sudo apt install opencl-headers ocl-icd-opencl-dev -y

源文件中加入如下代码：  
> #include <CL/opencl.hpp>

**`5.解决找不到OpenCL lib的问题`**  
cmake里面加上如下代码：  
> find_package(OpenCL REQUIRED)  
> link_libraries(OpenCL::OpenCL)  

## 如何得知当前系统是否支持OpenGL gpu platform
在Ubuntu系统下  
> sudo apt install clinfo  
> clinfo  

显示如下则表示找不到platform(在WSL和WSL 2中测试结果一样)  
**`Number of platforms`**      
(As far as I know OpenCL is not currently supported on WSL2)  





# References
https://learn.microsoft.com/en-us/windows/wsl/install  
https://jensd.be/1126/linux/cross-compiling-for-arm-or-aarch64-on-debian-or-ubuntu  
https://github.com/KhronosGroup/OpenCL-Guide/blob/main/chapters/getting_started_linux.md  
https://www.educative.io/answers/how-to-resolve-the-permission-denied-error-in-linux  








