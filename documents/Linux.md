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
## 种类  
- 永久的：通过修改配置文件实现  
> vim /etc/profile  

如下命令可以把/home加入到PATH变量的最前面（若想生效还需要source或重开shell）：  
```
export PATH=/home/fs: $PATH  
```
以上配置文件对所有用户生效  
也可以编辑对单独用户生效的文件  
> vim /home/user/.bash.profile  

- 临时的：使用export命令可以设置，但是关闭shell的时候失效  
> export 变量名=变量值  

## PATH格式


# Linux常用命令
用于查看binary的架构  
> file *  

-l means use a long listing format  
> ls -l

-a means show all files, include hidden files  
> ls -a

以上两个参数可以联合使用  
> ls -la  
> ls -al


Format介绍：  
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

ldd命令 用于打印程序或者库文件所依赖的共享库列表。  
> ldd *  

查看所有环境变量  
> env  

在环境变量中寻找含某个关键字的条目  
> env | grep wangge  

查看执行过的命令  
> history

显示指定的目录或文件所占用的磁盘空间。  
> du  

查找某个文件夹下名字带有某个字符串的所有文件  
> find . -maxdepth 1 -name "*string*" -print

(.和-print都是Default参数也可以省略)  

如果想把含有某个字符(":")的结果排除，可以用
> find . -maxdepth 1 -name "*string*" ! -name "*:*" -print

列出二进制文件中符号名字，比如函数名，变量名等  
> nm -DC xxx.so

查看当前用户  
>whoami

Linux df(全称: disk free)命令用于显示目前在Linux系统上的文件系统磁盘使用情况  
>df

使用人类可读的格式输出disk free  
>df -h

(h 是human readable的意思)  

查看自己的所有运行任务情况  
> bjobs



# Reference
https://github.com/gpuwangge/Wiki/blob/main/documents/WSL.md  



