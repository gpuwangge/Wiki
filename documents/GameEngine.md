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
如果选择使用系统API，需要熟练掌握windows api的用法。包含windows.h头文件，使用int WINAPI WinMain(...)作为入口函数  


# Reference
https://www.bilibili.com/video/BV1er4y1r7QK/  
毛星云<<逐梦旅程:Windows游戏编程之从零开始>>  
