# Tensorflow Lite

## 背景知识
TensorFlow Lite (TF Lite) 是谷歌推出的用于设备端推理的开源、跨平台深度学习框架，旨在为包括 Android 和 iOS 设备、嵌入式 Linux 和微控制器在内的多个平台提供支持。

TensorFlow Lite架構設計：  
第一階段：  
Trained TensorFlow Model (訓練模型：來源可以是預訓練的: Inception V3(圖像識別), MobileNet(視覺模型), 兩周都已經在ImageNet數據集上進行了訓練；也可以用TensorFlow Lite Model Maker創建)  
TensorFlow Lite Converter (把模型轉化成.tflite文件)  
TensorFlow Lite Model File(基於FlatBuffers針對最大速度和最小大小進行優化 )  
第二階段：（Android App / IOS App）  
Java API, C++ API / C++ API  
Interpreter [Kernels] 推斷器，可以推理  
Android Neural Networks API / NA  

TensorFlow Lite和TensorFlow的區別：  
Lite版重點是加載模型而不是訓練模型。後者可以訓練模型。  

TensorFlow Lite的應用領域：  
IOS, Android  
IoT 物聯網  
Raspberry PI, reTerminal 嵌入式Linux設備  
微控制器  

用Tensorflow Lite移動計算設備推理而不是其他桌面計算設備的原因  
把機器學習引入微型設備：可以降低硬件成本，無需穩定互聯網鏈接。  


## 编译方法
**`1. Download Python 3.9.5 Windows 64-bits exe`**  
When installing, add to PATH  
python -V //查看python版本  
其他版本比如3.11.4也行  
**`2. Follow this page to install dependencies:`**  
> https://www.tensorflow.org/install/source_windows  

> pip3 install -U six numpy wheel packaging  

有两个Warning可以忽略  
> pip3 install -U keras_preprocessing --no-deps  

**`3. Install Bazel`**  
這個tool是不用安裝的，只需要add this to PATH  
選擇Bazel版本：5.3.0  
**`4. Install MSYS2`**  
> https://www.msys2.org/  

選擇了使用最新的msys2-x86_64-20230718.exe  
這次需要點擊安裝了  
add bin folder to PATH:   
> E:\TensorflowLiteWindowsBuildTools\msys64\usr\bin  

then, in cmd prompt, run(在任意位置，一開始選Y):  
> pacman -S git patch unzip  

**`5. Install Visual C++ Build Tools 2019`**  
需要VS2019 Compiler  
可以安裝Visual Studio 2019 Community Edition, 注意勾選MSVC compiler  
如果已经安装了别的版本，比如2017，需要先卸載VS2017, 重裝2019(選擇Desktop Development with C++，就會包含MSVC v142)  
**`6. Download Tensorflow`**  
如何查看Tensorflow版本？在Github頁面右側，點擊release欄，直接下載，而不需要clone  
選擇哪個Tensorflow版本(2.12.0)  
TensorFlow 2.12.0，下面選擇Assets，Source code (zip)  
解壓縮  
**`7. Configure`**  
進入解壓縮後的tensorflow文件夾，打開cmd  
> python .\configure.py  

Set Python location(直接點擊回車，他就會顯示Found possible Python library paths)  
ROCm support: N  
CUDA support: N(不支持了，現在沒這個問題了。。。)  
arch:AVX: default  
Eigen strong inline: Y  
Android: N  
**`8. Bazel cmd`**  
> bazel build -c opt //tensorflow/lite:tensorflowlite  

result is in bazel-bin/tensorflow/lite directory  
tensorflowlite.dll  
tensorflowlite.dll.if.lib  
(-c opt的含義：https://bazel.build/docs/user-manual)  
(--compilation_mode: fastbuild, dbg or opt)  
(fastbuild: default, generate minimal debugging information)  
(dbg: build with debugging enabled (-g), so that you can use gdb (or another debugger))  
(opt: means build with optimization enabled and with assert() calls disabled (-O2 -DNDEBUG).)  

## 如何验证编译出来的dll/lib可用
参考这个repo:
> https://github.com/gpuwangge/TFLiteCheck  

## Reference
https://www.tensorflow.org/install/source_windows  
https://www.youtube.com/watch?v=1IhMISYvZG0  
github.com/karthickai/tflite





