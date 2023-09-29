# WSL Tutorial
Windows从Windows 10开始引入了Windows Subsystem for Linux(WSL)  
WSL本质上是个虚拟机  

## 安装教程
**`1.Open Windows PowerShell in administrator mode`**  
**`2.Run command`**  
> wsl --install  

或  
> wsl --install -d Ubuntu  

(默认安装Ubuntu，但可以改)  
(It will show installing: Virtual Machine Platform; Installing Ubuntu)  
(This command will enable the features necessary to run WSL and install the Ubuntu distribution of Linux. )  
(The requested operation is successful. Changes will not be effective until the system is rebooted.)  
**`3.重新启动后，继续自动安装Ubuntu`**  
**`4.按要求创建具有管理员权限（sudo）的账号和密码`**  
> [username]  
> [password]  

**`5.更新package`**  
> sudo apt update && sudo apt upgrade  

(到这一步安装就完成了)  
(这时候pwd: /home/wangge)  

## Windows如何同WSL交换文件
### Windows下如何用文件管理器打开/home/wangge
> \\wsl$

或者
> \\wsl.localhost\Ubuntu\home\wangge

实际存储位置可能在
> user\AppData\Local\Packages\CanonicalGroupLimited.Ubuntu_79rhkp1fndgsc

**`在这里建议进入\\wsl$后，右键点击Ubuntu后，选择"Map network drive.."添加快捷方式`**  

### WSL下如何通过shell打开c、d...盘文件
> cd /mnt

在shell里试图运行.exe会先显示找不到xxx.dll的错误  
但是可以用vim打开文本文件，可以edit并保存  
windows也可以修改/home/wangge下面的文件  
但windows和wsl并不是同一个系统，所以如果要再wsl上运行的程序，还是要放在/home/wangge下面比较好，否则跨系统传输文件很没有效率  

**`如果删除了这个wsl2 Linux子系统,它里面的文件也就没了`**   

## 如何在WSL里编译Linux binary
以下指令可以在shell里打开vs code  
> code .  

但是光打开也没用，编译出来的还是windows .exe  

### VS Code使用WSL的正确方法：  
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

### gcc和安装的g++是啥架构？
x86_64是什么：INTEL的64位指令集，常常简称x64。AMD64和它是一样的。这个构架也兼容32bit的软件。  
AArch64是什么: ARM的64位指令集。不支持32bit。常常简称为ARM64 

使用如下指令：  
> g++ -v

wsl和SWRD上获得的信息：  
> Target: x86_64-linux-gnu

windows上获得的信息：  
> Target: x86_64-w64-mingw32

### 如何查看一个binary是什么架构？
(在Ubuntu系统下)  
> file filename

(会打印出elf信息)  

### 如何编译出AArch64架构的binary?
需要用Cross Compile技术:  
**`1、安装aarch64版gcc(g++)`**  
> sudo apt install gcc make gcc-aarch64-linux-gnu binutils-aarch64-linux-gnu



## Credits
https://learn.microsoft.com/en-us/windows/wsl/install  
https://jensd.be/1126/linux/cross-compiling-for-arm-or-aarch64-on-debian-or-ubuntu  








