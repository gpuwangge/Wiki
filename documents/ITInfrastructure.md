# 启动无线网络的方法  
1、选择运营商，创建账号并Activate。同时获得Modern和网线。  
2、如果有带网络口的电脑，可以把网线接到电脑上，确保可以上网。如果不能访问internet，电话给运营商客服激活。  
3、购买Router  
4、找到屋内网线，连接到Moderm  
5、网线连接Moderm和Router  
6、Moderm和Router分别接上电源  
7、等待Moderm上Online灯闪烁并最终稳定，说明已经建立网络链接。如果Online灯不稳定亮起，联系运营商客服解决  
8、根据Router说明书，下载并打开Router App，可能需要注册账号。此时Router的internet灯也会亮起，但并不代表此时可以连接上网。  
9、根据Router说明书在App内设置，直到看见Internet灯亮起并稳定。如果无法获得稳定亮灯，联系Router客服解决  
10、需要连接的设备搜索Router名字和密码（一般印在Router背面）连接wifi  
11、有时候会出现Router找不到Moderm，这时候需要手动查询Moderm的MAC地址，然后拿一个电脑手动登录到Router上(IP一般是192.168.1.1，不要用公司电脑尝试)，手动更改MAC地址，之后Router就可以找到Moderm了。（建议这一步打电话给Router客服根据指示操作）  

# 开启Remote Desktop (Windows 10)  
1. 开启remote desk  
Start->Setting->Remote Desktop: Enable Remote Desktop  
2. 选择User  
默认是没有user的  
Select users that can remotely access this PC  
Click Add  
在下方方框中，输入本机的某个用户名A，点击Check Names，会显示A的完整名字  
点击OK就成功添加了。此时设置完成  
3. 同一局域网内如何连接  
在刚才的Select Users界面中  
From this location: 这个是远程连接的Computer名字  
(此名字即为[Device name]，可以通过PC About页面查看)  
(通过ipconfig，可以在本机查看IPv6 Address)  
(Remote端也可以通过ping [Device name]命令获得IPv6 Address)  
添加的用户名：远程连接的User name  
密码：就是User A的登陆密码  
4. 通过Remote PC如何连接  

# 如何链接无线麦克(Wireless Microphone)、音箱(Speaker)、合成器(Mixing Console)和投影仪
## 麦克
- 麦克放入电池
- 无线接收器需要充满电
## 音箱
- 链接主副两个音箱(使用配备的专用的链接线)
- 主音箱接电源
- 主音箱接两根音频数据线(左、右):2 x RCA，需要连接合成器获得音源
## 合成器
- 电源线接上
- 从音箱过来的两根音频数据线(左、右)需要连上
- 麦克风的无线接收器需要连接(?)
- 链接投影仪或其他音频输入，使用2 x RCA接口
## 投影仪
- 接电源
- 音频输出口连接合成器  

坚果N1有四个接口：
- HDMI(可以接电脑)
- HDMI eARC(增强型音频回传通道?好像也是用来输出音频数据的：HDMI to RCA Cable不知道能不能用，效果跟耳机口比起来如何？)
- USB(可以接移动硬盘)
- 耳机口(可以接耳机或合成器？可能需要耳机口转L、R的转接:3.5 Aux to 2 x RCA Adaptors)



