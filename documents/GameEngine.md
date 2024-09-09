# 简介
游戏引擎总是调用操作系统的API  
程序会编译生成资源文件，链接生成exe文件  
点击exe程序后，就会开始进程，也就是生成代码段，数据段，堆栈段，分配内存  
然后会找程序入口main函数  
exe就会变成数据加载入内存当中  
如果要看到图片，就要去内存中找到存图片的数据拿出来  
早期游戏开发需要操作硬件，比如驱动、控制显存，但现在不需要了  
举例：Unity的Input.GetKeyDown或UE的BindAxis本质都是使用windows的键盘消息WM_KEYDOWN和WM_KEYUP  
所以，只要能自己封装API(Windows, Vulkan)，就能自己做游戏引擎  
国内的游戏引擎程序员都是做系统这块的, 因此游戏开发也是系统级别的开发  

综上所述，要进行游戏开发有两个部分：系统API和图形API  
如果选择使用Windows系统，需要熟练掌握windows api的用法。包含windows.h头文件，使用int WINAPI WinMain(...)作为入口函数  
也可以使用glfw、sdl等包装好的库  
图形API一般可以选用跨平台的Vulkan。如果仅限于windows系统，也可以选directX  

# Windows编程小技巧
- Handle句柄：用处是找内存  
- 黑客注射玩法：首先不能用.exe方式，而应该生成.dll文件。入口函数要修改(或者不需要入口函数)。  
搜索一个叫"DLL注入工具"的工具，它可以自动把自己写的.dll注入到某个正在运行的进程里。  
这样就没办法在进程管理器里面关闭.dll的函数了。  
高端的玩法是自动注射，就是自己写注射工具  
- .dll, .lib, .exe本质都是一样的：存翻译好的机器代码和数据。其主要区别是用法不同。.exe可以直接启动, .dll不能单独启动，需要依附其他程序，其他程序会记录dll函数的地址。  
.lib存粹是代码库，在编译的时候就嵌入.exe了。  
- Windows内核(kernel)、用户(User)和GDI，是Windows内部三个主要.dll。
Kernel: 负责内存管理、文件输入输出和任务管理。ex: kernel32.dll  
User: 负责用户界面和窗口管理  
GDI: 负责图形设备接口或在打印机上显示文本和图形  


# Reference
- https://www.bilibili.com/video/BV1er4y1r7QK/  
- 毛星云<<逐梦旅程:Windows游戏编程之从零开始>>  
