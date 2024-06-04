# 版本控制简史
在软件开发史前时期，没有版本控制软件。不同程序员开发同一个项目的时候，往往为混乱的文件覆盖问题所苦恼。  
后来产生了一些简易的工具，比如diff，用来处理不同版本的文件的冲突。  

1985年，第一个版本控制软件Concurrent Version System诞生。CVS采用服务器/客户端架构，版本库位于服务器。每个文件的最初版本和差异文件被保存在服务器端。客户端仅保存最新文件。  
CVS也定义了最初的版本控制命令，比如commit, checkin, checkout, branch等。  
作为开源软件，CVS也存在一些问题，比如服务器端文件越多，速度越慢，合并和追踪困难，缺乏原子提交等。  

2001年出现了致力于取代CVS的商业开源版本控制软件Subversion(SVN)。SVN克服了很多CVS上的问题，是当时各个学校和企业进行版本控制的最佳选择之一。  
然而SVN和CVS一样是集中式版本控制系统，即一个项目只有唯一的一个版本库与之对应。如果中央服务器被破坏，那么所有的版本库都没有了。  
出于这个原因，Linus成为了CVS/SVN主要的反对者之一。  

2005年，Linus开发了分布式版本控制工具Git，Git有安全性强，全量提交等优点，从此逐渐成为开源社区版本控制系统的主流选择。  
在分布式版本控制系统中，每个程序员的代码提交到自己本地的版本库中。如果是solo开发，那么这样就足够了。    
如果多个程序员进行同一个项目的开发，那么还需要有个中央服务器(或称远端服务器)来负责交换各个程序员的本地版本库。  
区别在于，即使这个中央服务器被破坏也没关系，因为每个程序员的本地都有完整的版本库。  

2014年，Hex Swarm公司推出了集中式版本控制软件。当选择版本控制软件的时候，关于以Git为主的分布式和以Perforce为主的集中式管理的争议很多。它们有各自的优缺点。事实上，很多公司同时使用Git和Perforce。  

2008和2014年，分别诞生了GitHub和GitLab平台。  
GitHub和GitLab的相同点：它们都是版本控制系统的托管平台(用来管理仓库等)，并且都是基于Git这个系统的。  
GitHub和GitLab的区别：  
1、GitHub的免费仓库都是公有的，部署在商业服务器上。私有仓库需要付费。因此GitHub适合开源项目；  
2、GitLab可以免费搭建私有仓库，可以部署在自己的服务器上。所以GitLab适合学校、企业组建自己的项目；  

2009年，诞生了基于Git系统的代码审核工具Gerrit。  





