# Metal
## 简介
Metal是Apple公司为游戏开发者设计的图形API接口。  
Metal 开发者工具包括 Xcode 中的 Metal 调试器和 Instruments 中的 Metal 系统追踪。  
Xcode是IDE，它是专属MacOS上运行的开发工具，它不能在windows上运行。(但VS Code可以在MacOS上运行)  
如果要开发iOS程序，必须使用安装在MacOS上的Xcode。换言之，Xcode不跨平台。  
但是，可以在Windows虚拟机中安装MacOS，然后运行Xcode，~~或使用黑苹果机~~  

## Xcode安装
Xcode安装方式(10+Gb)：
1、App Store  
2、xip文件  

## Xcode基本用法
创建新项目的时候需要选择iOS或macOS  
organization id: 公司名字的倒写  
bundle-id: app ID  
swift UI/story(可选swift or oc)  
oc代码，启动页(ios)  

需要开发者账号，或apple id(wxj995@gmail.com, C2)  

M文件是用Objective-C编写的程序使用的类实现文件。 它始于@实现指令并初始化其他Objective-C源文件可以引用的变量和函数。 M个文件也可以引用标头（.H）文件。  

添加日志的方法：  
NSLog(@"hello")  
NSLog(@"%@", @"tt")  

打包：  
要求：花钱买的苹果开发者账号；app创建证书；Archive编译并导出ipa；上传App Store(通过Xcode或Transporter)  
苹果开发者账号怎么买：用apple id去开发者网站买。普通账号只能自己用来调试。  
编译的binary无法在另外的机器上运行，所以一般直接分发源代码，在目标机器上重新编译。  

更多信息：  
iOS应用程序打包成IPA文件是一项常见且重要的工作。在iOS开发中，为了将应用程序安装到真机上进行测试或者发布到App Store上，必须将代码打包成IPA文件。  

然而，有时候可能会遇到没有开发者账号的情况，下面我将为你详细介绍一下在没有开发者账号的情况下如何打包IPA文件的原理和步骤。首先，我们需要了解打包IPA文件的原理。  
打包成IPA文件主要依赖Xcode中的Archive功能，它会根据项目中的配置信息、代码和资源文件，生成一个包含了应用程序二进制文件和必要资源的归档文件。  
然后，通过Xcode中的Export功能，可以将归档文件导出为IPA文件。正常情况下，这个过程需要一个有效的开发者账号，因为只有通过Xcode登录有效的开发者账号，才能生成合法的IPA文件。  
但是，我们可以利用一些替代方法来绕过这个限制。  

https://www.yimenapp.com/kb-yimen/27018/  

总结一下，当没有开发者账号时，可以通过Xcode的Archive和Export功能来打包IPA文件。虽然这种方式有一些限制，但是对于进行个人测试或者企业内部分发应用程序是非常有用的。   

## XCFramework  
macOS的基础链接库是.a(静态库)或.tbd(动态库)  
.framework可作为动态库或静态库，需要使用lipo命令合成  
XCFramework(.xcframework)是为了取代.framework的  

## MoltenVK
MoltenVK is a layered implementation of Vulkan 1.2 graphics and compute functionality, that is built on Apple's Metal graphics and compute framework on macOS, iOS, tvOS, and visionOS.  
https://github.com/KhronosGroup/MoltenVK  



