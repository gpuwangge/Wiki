# WSL Tutorial
Windows从Windows 10开始引入了Windows Subsystem for Linux(WSL)  
WSL本质上是个虚拟机  

## 安装教程
**`1.Open Windows PowerShell in administrator mode  `**
**`2.Run command:  `**
> wsl --install  

或  
> wsl --install -d Ubuntu  

(默认安装Ubuntu，但可以改)  
(It will show installing: Virtual Machine Platform; Installing Ubuntu)  
(This command will enable the features necessary to run WSL and install the Ubuntu distribution of Linux. )  
(The requested operation is successful. Changes will not be effective until the system is rebooted.)  
**`3.重新启动后，继续自动安装Ubuntu  `**
**`4.按要求创建具有管理员权限（sudo）的账号和密码  `**
> [username]  
> [password]  

**`5.更新package`**  
> sudo apt update && sudo apt upgrade  

(到这一步安装就完成了)  
(这时候pwd: /home/wangge)  

## Windows如何向WSL传输文件
如何用文件管理器打开/home/wangge：
> \\wsl$

或者
> \\wsl.localhost\Ubuntu\home\wangge

实际存储位置可能在
> user\AppData\Local\Packages\CanonicalGroupLimited.Ubuntu_79rhkp1fndgsc

**`在这里建议进入\\wsl$后，右键点击Ubuntu后，选择"Map network drive.."添加快捷方式`**



## Credits
https://learn.microsoft.com/en-us/windows/wsl/install  








