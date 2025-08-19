# Introduction
SDL = Simple DirectMedia Layer，提供了控制图像、声音、输入输出控制的函数。  
SDL用C语言写成，是提供宏和函数的库，十分贴近底层  
SDL介于硬件层和软件层之间。SDL贴近硬件的那一部分是SDL_Renderer  
SDL是跨平台(Linux, Windows, Mac)函数库  
硬件的资源需要先加载到内存中，然后才能被软件访问  
不需要的资源应该及时关闭，避免内存泄漏  
SDL第一版发布于1998年，代表作为英雄无敌3(1999)，无冬之夜(2002)，雷神之锤4(2005)等  
SDL第二版(SDL2.0.0)发布于2013年，SDL2不兼容SDL  
SDL第三版(SDL3)发布于2023年 

# SDL和OpenGL等图形API的关系
SDL提供了创建窗口功能，图形API提供了绘图功能  
所以可以用SDL创建窗口，再用图形API在该窗口中渲染  
默认情况下，SDL在windows系统中使用Win32 API进行视频显示，Direct3D进行图形渲染，使用DirectSound和XAudio2播放声音  
在Mac OS X中使用COcoa进行视频显示，用OpenGL进行图形渲染，用Core Audio播放声音  
在Linux中使用X11进行视频显示，OpenGL进行图形渲染，使用ALSA、OSS和PulseAudio API来处理声音  

# SDL官方使用说明
根据SDL3官方建议(https://wiki.libsdl.org/SDL3/README/dynapi), 使用动态链接库(Dynamic API)的方式来使用SDL  
也就是说，将SDL编译成动态Lib文件，然后外挂在游戏上，这样做是出于如下考虑：  
- SDL需要根据系统更新。而游戏发布一段时间之后作者就会停止更新，导致游戏无法在更新的系统中使用  
- 如果要在老游戏里面更新静态链接的SDL库是一项费时费力的活，还不一定能成功  
- Steam Runtime里面会包含最新的SDL，但是如果玩家选择绕开Steam启动游戏，会导致缺乏dependency失败  
- 如果开发者选择Steam以外的方式发布，比如说GOG，那么当然必须附带独立的SDL Lib  

因此，SDL3不建议使用静态链接库，或把SDL代码直接嵌入你的项目中(https://wiki.libsdl.org/SDL3/Installation)  

SDL3官方示范代码：  
https://github.com/libsdl-org/SDL/tree/main/examples  

# SDL3(3.13 for windows)安装方法
进入网站  
https://github.com/libsdl-org/SDL/releases  
选择一个版本(比如3.2.20)  
如果选择使用windows下mingw编译的话，可以下载:  
SDL3-devel-3.2.20-mingw.tar.gz  
解压后打开x86_64-w64-mingw32/这里面有安装需要的所有东西：   
- bin/SDL3.dll：是编译和运行程序都需要的动态链接库。当app发布的时候，要连这个一起附上  
- include/SDL3: 需要include到app源代码的头文件。可以放在常用的系统头文件目录下。我是简单扔进C:/VulkanSDK/底下，就不用额外设置环境变量了  
 

在CMakeLists.txt中添加  
```
include_directories($ENV{INCLUDE})  
link_directories(${PROJECT_SOURCE_DIR})
link_libraries(SDL3)
```
确保SDL3.dll存在${PROJECT_SOURCE_DIR}这个位置以供编译使用。  
并且SDL3.dll需要存在binary的路径才能正确执行。  

在源文件中添加需要的SDL头文件，并编写测试代码，比如： 
```c++
#include <SDL3/SDL.h>
#include <SDL3/SDL_main.h>

int main(int argc, char* argv[])
{
    SDL_Log("Hello, SDL3!");
    return 0;
}
```
看见console中出现"Hello, SDL3!"则表明系统配置成功。  

# SDL3配合Vulkan的使用方法
目前Vulkan官方SDK1.3.283.0/Tempaltes/并没有更新到SDL3，因此需要以SDL2为基础版本手动更改为SDL3的API  
安装完毕Vulkan后，在VulkanSDK/1.3.xxx.x/Templates/Visual Studio 2022下面有模板  
VulkanProgram.zip: 简易vulkan模板  
VulkanWindowedProgram.zip: 使用SDL2的vulkan模板  
参考：https://wiki.libsdl.org/SDL3/NewFeatures



# SDL系统函数
## SDL_CreateWindow(), SDL_DestroyWindow(window)
用于支持窗口显示  

## SDL_Delay()
用于提供延迟  

# SDL Render
SDL Render相当于画笔。这种渲染的方法有点像GDI。如果画个简单图形还行。画复杂物体效率就比较低了。  
需要SDL/SDL.h支持

## SDL_CreateRender(window), SDL_DestroyRenderer(renderer)
创建和销毁画笔

## SDL_RenderClear(renderer), SDL_RenderPresent(renderer)
把图形呈现出来  

## SDL_SetRenderDrawColor(renderer), SDL_RenderDrawLine(renderer)
画线  

## SDL_RenderDrawRect(renderer)
画矩形  

# SDL Texture
需要SDL2/SDL_image.h支持  
先把图片加载到surface  
再通过surface create texture  
还需要用SDL_RenderCopy把texture的内容传到渲染器(SDL Renderer)  

# SDL Text
首先需要SD2/SDL_ttf.h支持。SDL_TTF主页如下：  
https://github.com/libsdl-org/SDL_ttf  
下载页面如下：  
https://github.com/libsdl-org/SDL_ttf/releases  
选择一个版本(比如3.2.2)下载：  
SDL3_ttf-devel-3.2.2-mingw.tar.gz  
解压缩后打开文件夹x86_64-w64-mingw32  
bin/下面的SDL3_ttf.dll就是二进制文件，其头文件为  
include/SDL3_ttf/SDL_ttf.h  
将两者包含进项目工程里就可以了  

下一步准备ttf font类型的字体  
这里建议使用google的开源font: https://fonts.google.com/noto  
点击"Browse all Noto fonts"也可以查看所有font样式  
如果要下载font文件，进入github相关页面：https://github.com/notofonts/noto-cjk  

最后把字体加载成Surface  
再使用Texture来呈现  
示例代码如下：  
```
if (TTF_Init() == -1) 
   std::cout << "SDL_ttf could not initialize! SDL_Error: " << SDL_GetError() << std::endl;
m_font = TTF_OpenFont("../thirdparty/fonts/NotoSansCJK-VF.otf.ttc", 24);
if (!m_font) 
    std::cout << "Failed to load font! SDL_Error: " << SDL_GetError() << std::endl;

//test SDL text (Test only, for this is conflict with Vulkan)
SDL_Color white = {255, 255, 255, 255};
SDL_Surface* textSurface = TTF_RenderText_Blended(m_font, "hello你好", 11, white);
SDL_Renderer* renderer = SDL_CreateRenderer(window, NULL);
SDL_Texture* textTexture = SDL_CreateTextureFromSurface(renderer, textSurface);
SDL_FRect textRect = {20, 20, textSurface->w, textSurface->h};
SDL_SetRenderDrawColor(renderer, 0, 0, 0, 255);
SDL_RenderClear(renderer);
SDL_RenderTexture(renderer, textTexture, NULL, &textRect);
SDL_RenderPresent(renderer);
TTF_CloseFont(m_font);
TTF_Quit();
```

# SDL Event
用于用户交互  
```
while (SDL_PollEvent(&event)){
  ...
}

```

# SDL Sound
需要SDL2/SDL_mixer.h支持  
分为音效(chunk)和音乐(music)两种。区别是前者能同时播放多个。  
```
Mix_Chunk *sound = Mix_LoadWAV("./res/sound.mp3");
Mix_music *bgm = Mix_loadMUS("./res/bgm.mp3")

Mix_PlayChannel(-1, sound, 0); //channel(-1代表自动选择音轨)，chunk, loop
Mix_PlayMusic(bgm, 0); //music, loop

Mix_FreeChunk(sound);
Mix_FreeMusic(bgm);
```

# SDL Timer
用于计时，单位是millisecond  
```
unsigned int time = SDL_GetTicks();  
```

# SDL 错误处理
如果上述指针对象生成的时候出错，会返回一个空对象  


# Reference
https://zh.wikipedia.org/zh-hans/SDL  
https://www.bilibili.com/video/BV1gi4y1Y71f/?spm_id_from=333.337.search-card.all.click&vd_source=e9d9bc8892014008f20c4e4027b98036  
https://wiki.libsdl.org/SDL2/FrontPage  
https://wiki.libsdl.org/SDL3/FrontPage  
https://github.com/libsdl-org/SDL/releases/tag/preview-3.1.3  
https://www.cnblogs.com/xrunning/p/18241082  



