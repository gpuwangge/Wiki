# Linux发行版
Linux发行版(Distro)是一个由 Linux 内核、GNU 工具、附加软件和软件包管理器组成的操作系统，它也可能包括显示服务器和桌面环境，以用作常规的桌面操作系统。  
通常我们用Linux来指代Linux操作系统。但是实际上“Linux” 其实刚好是操作系统内核(Kernel)的名字，而 “Linux发行版”作为操作系统名字可以很好的区分彼此。这就是为什么Linux发行版有时也被称为基于Linux的操作系统的原因。  
Linux发行版是一个免费开源的操作系统。开源软件是源代码可以任意获取的计算机软件，任何人都能查看、修改和分发他们认为合适的代码。  
Linux发行版分为以下系列  

## Redhat系
该Linux发行版系列是商用系列，包括Redhat发行的RHEL(Redhat Enterprise Linux)。Redhat是一家以开源软件为核心的商业公司。  
但事实上，任何人都可以免费获得RHEL源代码，Readhat仅对服务收费，并对其负责。毕竟企业的操作系统是需要技术支持、安全更新和稳定性保证的。  
但也有CentOS这样的免费克隆版。其逻辑是把核心能力放在开源项目中，企业需要的高级特性集成到商业版中进行技术支持。  
RHEL和CentOS稳定性都很好，常见于商业公司服务器中。  
另外，Fedora也是基于RHEL的免费发行版，但集成了更多软件包，因此稳定性较差。  
Redhat系的特点是干净整洁，高手喜爱。  
Redhat系包管理工具使用rpm, yum, dnf  

## Debian系
该Linux发行版系列是免费系列，由社区运作。  
Debian发行版历史悠久，最早于1993年创建。结构精简稳定，deb包高度集中，依赖性问题很少。缺点是更新周期长，软件老旧。但其优点稳定并且占用硬盘和内存特别小，再服务器操作系统中也占有一席之地。  
Ubuntu是基于Debian的二次发行版。也是使用量最大的Linux发行版。历史也比较悠久，社区支持完善。但也有评论说稳定性差，效率不太高。因此，不太适合作为服务器操作系统。但是其图形界面优秀，适合作为个人用户使用。  
Mint也是Debian系的发行版，并且可以算是Ubuntu的衍生版。其特点是接近Windows用户的使用体验。集成了很多常用软件，开箱即用。但也有评论其比较臃肿。  
Debian系包管理工具采用dpkg(.deb), apt-get。 

## 其他系
Arch Linux发行版由自由和开源软件组成。稳定性不如Redhat/Debian系。优点是软件库庞大。包管理器叫pacman。  
Tails Linux发行版注重安全，有许多加密工具，收隐私爱好者喜爱。  

# Linux环境变量
## 如何查看某个环境变量
> echo $PATH  


## 如何查看所有环境变量  
> env  

在环境变量中寻找含某个关键字的条目  
> env | grep wangge

另外，printenv也有一样的效果  

## 种类  
- 永久的：通过修改配置文件实现  
> vim /etc/profile  

如下命令可以把/home加入到PATH变量的最前面（若想生效还需要source或重开shell）：  
```
export PATH=/home/fs:$PATH  
```
以上配置文件对所有用户生效  
也可以编辑对单独用户生效的文件  
> vim /home/user/.bash.profile  

- 临时的：使用export命令可以设置，但是关闭shell的时候失效  
> export 变量名=变量值  

## PATH格式
PATH=$PATH:\<PATH1\>:\<PATH2\>:\<PATH3\>:-----:\<PATHn\>  

# Linux Module
## 微内核体系结构(Micro Kernel)
在该结构下，操作系统的核心部分是一个很小的内核，实现一些最基本的服务，比如创建和删除进程，内存管理，中断管理等。  
其他功能都放在微内核外的用户空间进行，比如文件系统，网络协议等。  
微内核结构的有点事扩展性好，缺点是效率较低。  
采用微结构内核的典型系统是WindowsNT  

## 单一体系结构(Monolithic Kernel)
在该结构下整个操作系统内核是一个单独的大程序。优点是性能好，缺点是可扩展性和可维护性差。  
采用单一体系结构的典型系统是Linux  

## Module
为了克服单一体系结构的缺点，Linux采用了内核模块(Module)机制  
用户可以根据需要，在不对内核重新编译的情况下，动态地把模块从内核里装入或移出  

Module的特点：不能独立运行。可以在运行时链接到系统中作为内核的一部分。本质是一些函数和数据结构。  

查看所有模块  
> module av

查看某个模块  
> module av [moduleName]

查看已加载模块  
> module list

加载和卸载模块  
> module load [modulefile]  
> module unload [modulefile]  

切换模块  
> module switch [modulefile]

清空所有模块  
> module purge

显示模块内容  
> module show [modulefile]


# Linux常用命令
## file
用于查看binary的架构  
> file *  

## ls
-l means use a long listing format  
> ls -l

-a means show all files, include hidden files  
> ls -a

以上两个参数可以联合使用  
> ls -la  
> ls -al

## Format  
第一个字母肯定是-(代表regular file)或d(代表directory)  
从第二个字母开始，三个为一组，格式是rwx，分别代表read, write and executecd。如果是-表示没有这项权限  
(身份顺序为：owner、group、others)  
接下来一个数字表示链接数？  
接下来的两个word表示owner和owner group  
接下来表示size(byte)  
接下来表示最后修改时间  
最后表示文档名称。以.开始的是隐藏文件  
举例：  
-rw-r--r--.  1 root root   18 Dec 28  2013 .bash_logout  

## ldd
ldd命令 用于打印程序或者库文件所依赖的共享库列表。  
> ldd *  

## history
查看执行过的命令  
> history

## du
显示指定的目录或文件所占用的磁盘空间。  
> du  

## find
查找某个文件夹下名字带有某个字符串的所有文件  
> find . -maxdepth 1 -name "*string*" -print

(.和-print都是Default参数也可以省略)  

如果想把含有某个字符(":")的结果排除，可以用
> find . -maxdepth 1 -name "*string*" ! -name "*:*" -print

## nm
列出二进制文件中符号名字，比如函数名，变量名等  
> nm -DC xxx.so

-D: 打印动态符号。Display the dynamic symbols rather than the normal symbols. This is only meaningful for dynamic objects, such as certain types of shared libraries。总之，看见.so无脑加-D就行了  
-C: 输出demangle过了的符号名称。-C 总是适用于c++编译出来的对象文件。还记得c++中有重载么？为了区分重载函数，c++编译器会将函数返回值/参数等信息附加到函数名称中去形成一个mangle过的符号，那用这个选项列出符号的时候，做一个逆操作，输出那些原始的、我们可理解的符号名称。  
常见的符号类型:  
A 该符号的值在今后的链接中将不再改变；  
B 该符号放在BSS段中，通常是那些未初始化的全局变量；  
D 该符号放在普通的数据段中，通常是那些已经初始化的全局变量；  
T 该符号放在代码段中，通常是那些全局非静态函数；  
U 该符号未定义过，需要自其他对象文件中链接进来；  
W 未明确指定的弱链接符号；同链接的其他对象文件中有它的定义就用上，否则就用一个系统特别指定的默认值。  

## strip  
> strip filename

该命令会去除一个程序中的符号表，使其体积变小  
但是因为符号表在Debug中有重要作用，所以之应该在调试完毕，即将发布的时候使用strip命令  
file命令可以查看一个程序是否strip  
对strip程序使用nm命令，会得到"no symbols"的提示。  

 ## whoami
查看当前用户  
> whoami

## df
Linux df(全称: disk free)命令用于显示目前在Linux系统上的文件系统磁盘使用情况  
>df

使用人类可读的格式输出disk free  
>df -h

(h 是human readable的意思)  

## bjobs
查看自己的所有运行任务情况  
> bjobs

## tar
Linux下压缩和解压缩用    
> tar -xvf xxxx.tar  

-x: extract, 展开文件  
-v: verbose，详细显示处理的文件  
-f: file, 指定要处理的文件  

## ln 
ln命令的作用是link files，它为某一个文件在另一个位置建立一个同步的链接。
这样的好处是，当我们要在不同目录使用相同的文件时，不需要保存两次相同的文件。
我们只需要选一个位置，放上文件，然后在其他的目录建立同步链接。
链接有两种：硬链接(Hard link)和软链接(Symbolic link)
- 软连接：就是存一个路径，有点像windows系统的快捷方式。软连接里面就是存了个路径。如果删除了源文件，软连接依旧存在，只是指向一个不存在的内容，无法访问源文件了。创建命令为ln -s。
- 硬链接：在linux系统中，美俄该文件都有一个索引节点号(Inode Index)。可以有多个文件名指向同一个索引节点。比如，A是B的硬链接(A、B都是文件名)，它们的Inode Index都一样。A和B的地位相等，删除其中一个不会影响另一个。(软连接的Inode Index不一样，其中一个只有路径)
可以看出来，两种链接都只会保存一个副本，不会消耗额外的空间。使用硬链接更加复杂些，但好处是可以防止误删。对硬链接来说，只有所有相应的文件名被删除后(即所有关联的硬链接文件被删除)，其文件才会真正被删除。
举例：
> ln -s /proj/testfolder

该命令把/proj/testfoder在当前目录下创建了一个软连接

## cp



# Linux常用环境变量
如上所述，环境变量可以用env或printenv查看  
可以用env | grep xxxxx来查看含有某个关键字的环境变量  
如果知道变量名字，也可以直接用echo $xxxxx来查看其变量的值  
如果要更改变量的值，可以使用setenv xxxxx yyyyy命令，比如：  
> setenv DISPLAY abc12345:0  

## DISPLAY
如名字所暗示的，这个变量保存了有关显示设备的编号  
所有的图形程序都把结果显示在该编号的设备上  
当使用  
> echo $DISPLAY  

后，会显示如下格式的结果  
xxx:yyy.zzz  
其中xxx是host名字。有可能是一个名字，也可能是一个ip，也有可能缺失(表示host就是本机)。  
(host指Xserver所在的主机主机名或者ip地址, 空即代表Xserver运行于本机)  
yyy就是display的编号，默认是0  
zzz是显示器(screen)编号。比如一台机器连接了两台显示器，就会分给给编号。默认是0，该号码也可能缺失。  
合起来的意思是：图形程序把结果输入到xxx主机的yyy编号显示设备的zzz显示器上.  
使用如下命令可以测试图形被显示在哪里：  
> xclock




# Reference
https://zhuanlan.zhihu.com/p/431707034


