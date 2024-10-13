# 本章工具介绍
VS Code是文本编辑器，所以是不带编译器的，需要下载一个C++编译器，比如MinGW(x86_64-win32-seh)。  
同时，使用Git进行版本管理。因为VS Code和Git有很好的兼容性。  
 
# MinGW安装方法
MinGW下载地址：  
https://www.mingw-w64.org/downloads/  
页面底下 Sources这一栏，点击SourceForge。拉到底下，选择x86_64-win32-seh

下载后解压后剪切粘贴到c盘的根目录下。  
然后是配置环境变量——也就是告诉VS Code编译器在什么地方：  
把MinGW的bin路径(c:\mingw64\bin)粘贴到PATH环境变量里面。  
在command prompt里面输入  
> where gcc

就可以知道是否装好了。  
如果安装过Visual Studio，也会有编译器了： cl.exe。而且这个是Windows下VS Code的默认编译器   
(关于posix版本编译器：可以再下载x86_64-posix-seh，然后文件夹加上后缀_posix跟win32版放在一起，如果遇上p线程的问题可以换这个: 修改CMakeLists，使用posix版g++，但make还是一样的)  


# VS Code安装方法
VS Code下载地址：  
https://code.visualstudio.com/download  

安装时勾选允许添加入右键菜单,也就是勾选如下两个选项：  
Add "Open with code" action to Windows Explorer file context menu  
Add "Open with code" action to Windows Explorer directory context menu  


安装vscode后，再安装必要插件GitLens. 安装后，左侧出现GitLens和GitLens Inspect图标。  
建议安装如下Extensions:  
C/C++  
Better Comments (optional)  
Shader languages support

# Git安装方法
https://www.git-scm.com/downloads  
要先安装VS Code，再安装git  
在Choosing the default editor used by Git这个选项里选择  
Use Visual Studio Code as Git's default editor   
安装好之后，就可以在console里使用Git Clone等命令了  
在local git repo打开VS Code，可以发现左边的Source Control功能会自动开启git功能，表示安装成功，可以联动了  
如果要使用git来check in，别忘了设定name和email，比如：
> git config --global user.name "Connor"  
> git config --global user.email wxj995@gmail.com  

可以设置成任意值，这个是只为了区分此次修改是哪个id提交的  


# VS Code常用快捷键
Ctrl + P: 显示所有文件  
Ctrl + Shift + P 或 F1: 显示Command Editor  
Ctrl + Shift + B: Run Build Task  

# VS Code使用说明
打开VS Code后，点击Open Folder，选择一个你希望保存code的文件夹。  
新建一个名字为xxx.c的文件，就会弹窗问你是否需要安装C/C++ Extension Pack，安装之。  
安装好了之后会提示你选择Default Compiler（gcc or g++）。  
写一个hello world程序，点击运行，会提示选择gcc.exe作为build/debug工具。  
运行结果会显示在底下的TERMINAL窗口。  

这时候注意到左侧自动建立了.vscode文件夹，里面有一些配置文件，包含了编译器和编译参数等信息。可以自行修改该文件改变编译效果。  
使用VS Code的一个优点是使用intelliSenseMode来帮助定位代码，这里就需要这些配置文件了。  
配置文件有时候不会自动生成，这样就需要手动添加。  
主要的配置文件有如下几个  

## 配置文件：settings.json
这个文件设置VS Code的compiler path and IntelliSense settings。更新这个文件会自动更新c_cpp_properties.json。  
在建立工程后，这个文件似乎会自动产生。但其实不设置这个文件也没有什么关系。  

## 配置文件：tasks.json
跟编译有关的文件设置  
如果要添加tasks.json, 按Ctrl+Shift+P打开Command Editor，输入"Task"后会显示一系列跟Task有关的指令。  
选择"Tasks: Configure Default Build Task",然后选Task Build，就会生成默认的tasks.json文件了。  
(可以不要这个文件吗？？)  

## 配置文件：launch.json
跟运行有关的文件设置。 如果需要Debugger，就需要设置这个文件。  
这个Json会自动生成不需要修改。这个文件也不一定会出现。  

VS Code配置Run功能呢？
缺省情况下，VS Code支持打开terminal后通过控制台cmake, make, run。  
但是VS Code提供了更加快捷的可以不通过terminal来run的功能。这就需要配置额task.json了。  
在没有配置的时候，点击Run->Start Debugging或单击键盘上的F5会提示：
"You dong have an extension for debugging 'JSON with Comments'. Should we find a 'JSON with Comments' extension in the Marketplace?"

点击左侧“Run and Debug”按钮(图标是一个瓢虫趴在三角形上)，点击"create a launch.json file"，选择C++(GDB), 将生成launch.json文件。  
在json页面打开的情况下，点击右下角Add Configuration可添加配置。选择C/C++: (gdb) Launch  
(需要修改gdb定位，也就是之前MinGW的bin文件夹内)  
(多文件项目，其实就是建立不同的文件夹。可以新建更多工作区来存放不同代码，不同工作区有不同的.vscode编译和调试配置)  
第一次运行C++文件的时候，会提示选择g++编译器  
下面会看到"configurations"项目被自动填充了。  
program这一项设定要手动修改为需要debug的binary，比如
```  
"program": "${cwd}/bin/simpleMipmap.exe"  
```
另外需要选择debuger工具，一般可以使用gdb。可以通过如下指令测试是不是支持gdb：
```
where gdb
gdb
```
如果gdb可以用，则更改如下配置。
> "MIMode": "gdb"  
> "miDebuggerPath": "gdb"  

这时候F5就能正确执行binary了。并且程序里面也可以设置断点了。  


## 配置文件：c_cpp_properties.json
存有c/c++相关compiler的信息。如果需要intellisense，就需要设置这个文件。  
如果要添加c_cpp_properties.json，使用如下快捷键：Control+Shift+P，選擇C/C++: Edit Configurations (JSON)，这时候会生成c_cpp_properties.json。  
添加之后，会自动填充compilerPath, intelliSenseMode的信息。如下所示：  
```
"configurations": [
     {
         "compilerPath": "C:\\mingw64\\bin\\gcc.exe",
         "intelliSenseMode": "windows-gcc-x64"
     }
],
```
如果要让intelliSenseMode正常工作，还需要添加include信息：  
在includePath栏添加INCLUDE，如下所示  
```
"configurations": [
     {
         "includePath": [
             "${workspaceFolder}/**",
             "${INCLUDE}"
         ]
     } 
],
```
这里的INCLUDE是一个环境变量，我测试的时候包含了VulkanSDK, GLM, GLFW, SDL等头文件。  
如果不用环境变量INCLUDE，也可以直接展开，如下
```
"C_Cpp_Runner.includePaths": [
   "C:/VulkanSDK",
   "C:/VulkanSDK/GLM",
   "C:/VulkanSDK/GLFW/include",
   //"C:/VulkanSDK/1.2.176.1/Include"
   "${VULKAN_SDK}/Include"
],
```
如果cpp代码是c++17之后，需要手动把这个信息添加进这个文件。  
其实如果不使用IntelliSense，那不需要设置.vscode。因为可以自己用gcc/g++/cl来编译。  
但是vscode的IntelliSense可以自动发现一些错误。以下是设置IntelliSense的方法。  





