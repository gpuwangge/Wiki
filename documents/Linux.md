# Linux

## Linux发行版
Linux发行版(Distro)是一个由 Linux 内核、GNU 工具、附加软件和软件包管理器组成的操作系统，它也可能包括显示服务器和桌面环境，以用作常规的桌面操作系统。  
通常我们用Linux来指代Linux操作系统。但是实际上“Linux” 其实刚好是操作系统内核(Kernel)的名字，而 “Linux发行版”作为操作系统名字可以很好的区分彼此。这就是为什么Linux发行版有时也被称为基于Linux的操作系统的原因。  
Linux发行版是一个免费开源的操作系统。开源软件是源代码可以任意获取的计算机软件，任何人都能查看、修改和分发他们认为合适的代码。  
Linux发行版分为以下系列  

### Redhat系
该Linux发行版系列是商用系列，包括Redhat发行的RHEL(Redhat Enterprise Linux)。Redhat是一家以开源软件为核心的商业公司。  
但事实上，任何人都可以免费获得RHEL源代码，Readhat仅对服务收费，并对其负责。毕竟企业的操作系统是需要技术支持、安全更新和稳定性保证的。  
但也有CentOS这样的免费克隆版。其逻辑是把核心能力放在开源项目中，企业需要的高级特性集成到商业版中进行技术支持。  
RHEL和CentOS稳定性都很好，常见于商业公司服务器中。  
另外，Fedora也是基于RHEL的免费发行版，但集成了更多软件包，因此稳定性较差。  
Redhat系的特点是干净整洁，高手喜爱。  
Redhat系包管理工具使用rpm, yum, dnf  

### Debian系
该Linux发行版系列是免费系列，由社区运作。  
Debian发行版历史悠久，最早于1993年创建。结构精简稳定，deb包高度集中，依赖性问题很少。缺点是更新周期长，软件老旧。但其优点稳定并且占用硬盘和内存特别小，再服务器操作系统中也占有一席之地。  
Ubuntu是基于Debian的二次发行版。也是使用量最大的Linux发行版。历史也比较悠久，社区支持完善。但也有评论说稳定性差，效率不太高。因此，不太适合作为服务器操作系统。但是其图形界面优秀，适合作为个人用户使用。  
Mint也是Debian系的发行版，并且可以算是Ubuntu的衍生版。其特点是接近Windows用户的使用体验。集成了很多常用软件，开箱即用。但也有评论其比较臃肿。  
Debian系包管理工具采用dpkg(.deb), apt-get。 

### 其他系
Arch Linux发行版由自由和开源软件组成。稳定性不如Redhat/Debian系。优点是软件库庞大。包管理器叫pacman。  
Tails Linux发行版注重安全，有许多加密工具，收隐私爱好者喜爱。  


## Wiki Reference
https://github.com/gpuwangge/Wiki/blob/main/documents/WSL.md  



