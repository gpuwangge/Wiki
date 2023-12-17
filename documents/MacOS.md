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

## XCode Instruments使用技巧
在右上方spotlight里面搜索instruments可以打开  
(instruments似乎是xcode的一个组件)  

打开instruments软件后，在上方地址栏处，选择要record trace的文件夹和目标文件  
点击edit，可以按+号添加argument  
File/Record Trace可以开始执行trace的录制  

## XCode基本使用说明  

### 如何知道XCode生成的binary生成在哪里
默认的build目录在File -> Project Settings 中有显示  

在Xcode界面下，点击左上角：  
Xcode -> Settings... -> Locations  
可以发现有一个default文件夹，名字是../Xcode/DerivedData  
点击Advanced，可以发现更多选项。  
默认选项是Unique。代表对于每一个Project，在DerivedData文件夹下会生成一个独特的名字，一般是Project名字加上一些随机数字和字母  

如何进入这个文件夹：在Finder界面下，点击左上角：  
Go -> Go to Folder...  
在这里黏贴地址  

binary位置举例：  
Users/userName/Library/Developer/Xcode/DerivedData/项目名字-随机生成的ID/Build/Products/Debug  

或者在控制台使用open命令也可以快速进入这个位置  

### 如何执行XCode编译的binary
有以下三种方法： 
- 在XCode里面点击三角形符号(build and run)  
- 用Finder打开binary位置的文件夹，鼠标双击运行
- 在terminal里打开binary位置的文件夹，使用Open命令运行
(这里会发现编译好的binary并不是一个二进制文件，而是一个后缀为.app的文件夹。使用./并不能运行它)  

### 如何切换Xcode编译Debug/Release版本  
Product/Scheme/Edit Scheme.../Build Configuration  

### XCFramework相关设置
有三种Build For模式：Running, Testing, Profiling  
- Running is for running your app (on the Mac for Mac OS X, in the simulator or on the device for iOS).  
- Profiling is for running your app with Instruments (for finding memory leaks, bottlenecks etc.).  
- Testing is for running unit tests.

Build .xcframework的时候，如果选择Running，则build release的结果  
Build .xcframework的时候，如果选择Testing，则build debug的结果  

界面上的三角形按钮写着Run，猜测可能是Running模式。实测Run按钮除了编译，还会执行。Build for Running则只会build，不会执行。  
界面上的菱形中间有个三角形的按钮写着Test，但是实测可以Testing的项目，使用这个Test按钮也无法build，猜测功能也不尽相同。Test应该包含了Build和执行的功能。  
点击Product->Build For可以选择其他模式  

