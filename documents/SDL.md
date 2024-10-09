# Introduction
SDL = Simple DirectMedia Layer  
SDL是提供宏和函数的库，十分贴近底层  
SDL介于硬件层和软件层之间。SDL贴近硬件的那一部分是SDL_Renderer  
硬件的资源需要先加载到内存中，然后才能被软件访问  
不需要的资源应该及时关闭，避免内存泄漏  

# 使用方法
include files  
lib  
link lib  
copy SDL2.dll, SDL2_image.dll, SDL2_mixer.dll, SDL2_ttf.dll  
```
#include <SDL2/SDL.h>
#include <SDL2/SDL_image.h>
#include <SDL2/SDL_mixer.h>
#include <SDL2/SDL_ttf.h>
int main(int argc, char *argv[]){
  return 0;
}
```

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


# Reference
https://wiki.libsdl.org/SDL2/FrontPage  
https://www.bilibili.com/video/BV1gi4y1Y71f/?spm_id_from=333.337.search-card.all.click&vd_source=e9d9bc8892014008f20c4e4027b98036  




