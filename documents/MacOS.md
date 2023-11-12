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

.bash_profile设置举例
```
export PATH=$PATH:/Users/mtk/32132/Downloads/nasm-2.16.01  
export PATH=$PATH:/Application/CMake.app/Contents/bin  
```

## MacOS安装软件举例
### NASM
https://www.nasm.us/  
点击Stable  
进入如下页面：  
https://www.nasm.us/pub/nasm/releasebuilds/2.16.01/macosx/  
下载  
https://nasm-2.16.01-macosx.zip/  
并解压缩，把地址添加进PATH  

这时候运行NASM，会出现如下错误：  
Cannot Be Opened Because the Developer Cannot be Verified  

解决办法：打开下载目录，按住control点击nasm  
点击Open  
这时候nasm会被加入白名单，以后就可以正常访问了  

### Ninja
https://ninja-build.org/  
点击download the Ninja binary  
根据系统选择文件binary下载  

### 安装特定版本的XCode
根据Geekbench 6的说明，需要XCode 14.2版(默认安装的是14.3)  
并且，它去找的目录是  
/Applications/Xcode-14.2.app/  
（默认的位置是/Applications/Xcode/）  

来到这个页面(需要输入苹果账号密码)  
https://developer.apple.com/download/all/  
搜索Xcode 14.2并下载xip文件  
双击xip文件会自动解压缩，默认名字就是Xcode  
更名为Xcode-14.2.app，然后拖入Applications文件夹下(control+点击 拖拽)  

### 安装Chevron
pip3 install chevron   

## XCode Swift使用技巧
可以建立一个带gui的app，或者控制台一样的项目  

binary位置：  
Users/userName/Library/Developer/Xcode/DerivedData/项目名字-随机生成的ID/Build/Products/Debug  

在控制台使用open命令可以快速进入这个位置  

如何切换Xcode编译Debug/Release版本：  
Product/Scheme/Edit Scheme.../Build Configuration  

## Windows如何向MacOS传输大文件
通过网盘  
1、windows上右键点击文件夹  
2、选择give access to  
3、选择Specific people...  
4、选择Everyone，按Add，点击share  
5、等一会儿，会生成一个NB打头的地址，记下来  
6、macOS上点击Finder  
7、点击Go->Connect to Server...  
8、在地址栏中输入smb://刚才copy的地址  
9、点击connect  


