# MacOS

## MacOS环境变量
Mac OS系统的环境变量主要由下面几个文件和文件夹所决定，并且他们的加载顺序如下：  

/etc/profile  
/etc/paths  
/etc/bashrc  
~/.bash_profile 或 ~/.bashrc  
~/.bash_login  

其中，/etc/profile, /etc/paths, /etc/bashrc 是系统级别配置文件，系统启动就会加载。  

后面几个是当前用户级的环境变量，按照**从前往后的顺序**读取，如果~/.bash_profile文件存在，则后面的几个文件就会被忽略不读，如果~/.bash_profile文件不存在，才会以此类推读取后面的文件。  

~/.bashrc没有上述规则，它是bash shell打开的时候载入的。  

一般不建议修改/etc/profile和/etc/bashrc 文件，而应去修改/etc/paths文件。  

## MacOS添加PATH环境变量的两个方法
**`1. 全局修改`**  
不建议此方法  

**`2. 用户级别修改（单个用户生效）`**  
一般都是修改~/.bash_profile文件（Linux中是~/.bashrc，而Mac OS是~/.bash_profile）  

若bash shell是以login方式执行时，才会读取此文件。该文件仅仅执行一次!  

具体操作步骤：  
打开macOS控制台  
> vim ~/.bash_profile  
> export PATH=$PATH:地址  

那么如何获得地址呢？  
直接把文件拖拽到terminal之内，然后使用pwd命令  
地址举例：  
/Users/mtk/32132/Downloads/nasm-2.16.01  
(按住command再点击文件进行拖拽)  

修改完后使用如下命令查看PATH(需要重启后生效)  
> echo $PATH  

```.bash_profile
export PATH=$PATH:/Users/mtk/32132/Downloads/nasm-2.16.01  
export PATH=$PATH:/Application/CMake.app/Contents/bin  
```




