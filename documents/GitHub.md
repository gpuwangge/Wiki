# 基础知识
GitHub有三个状态区  
- **`工作区`**(Working Directory，VS Code里叫Changes区)-直接编辑的地方，比如记事本打开的文件，肉眼可见，直接操作。add可以把文件增添到暂存区。  
- **`暂存区`**(Stage/Index，VS Code里叫Stages Change区)-数据暂时存放的区域。暂存区的数据可以commit到版本区。  
- **`版本区`**(Commit History)-存放已经commit的数据的区域。push的时候就是把这里的数发到remote repo。   

# 从init开始(本地有一个工作中待上传的文件夹)  
**`1.网站上新建一个new remote repo`**  
**`2.在本地文件夾打开VS Code, 进入terminal，使用如下指令`**   
> git init

(這一步完成後，全部文件轉Changes區)  
**`3.Add所有文件`**  
> git add -A

(這一步完成後，全部文件轉入Staged Changes區)  
可以用如下命令查看有哪些文件被加入了Staged Changes区
> git status

如果返回了，可以用如下命令清空Staged Changes区
> git reset

**`4.Commit所有文件`**    
> git commit -m "first commit"

(`git log --stat` or `git status`可以查看branch name)  
(這一步完成後，staged區域清空)  
**`5.按照GitHub网站的操作做第一次Push，例如： `**   
> git remote add origin url

> git branch -M main

(如果新建的remote repo不是空的就需要这一步: `git pull --rebase origin main`)  
(這一步完成後，remote repo上的readme.MD或LICENSE就會被同步到本地了)  
> git push -u origin main

如果之前没配置过name和email，则要通过如下指令配置：  
> git config --global user.name "Your Name"

> git config --global user.email you@example.com

(左侧的sync按钮，其实就是sync = pull & push)  
(pull的时候会产生conflict)   
(如果上傳文件大於50mb，是不推薦的。目前來看69.59 MB的文件還是能成功上傳。只是過程中有個warning)  


# 被要求github.com的Username和Password怎么办  
在push的时候会被要求提供authenticate。这是因为push的时候需要access github的API。  
首先要提供用户名，这个就是github.com/后面的那个自定义名字。  
密码并不是网站密码，而是在网站上设置的Token。设置方法如下：  
1.go to github.com, click user icon  
2.click Settings  
3.click Developer settings  
4.click Personal access tokens  
5.put something in the Note, Set Expiration, choose scope(repo, workflow, gist, user)  
6.click Generate token  
7.save token in some save place  
8.push again, when asked username to github.com, use gpuwangge  
9.when asked password to xxx@github.com, paste the token  


# 从clone开始(使用已存在的Remote Repo的情況)
登录GitHub账号，并建立一个repo，或选择一个repo。总之，准备好url。  
(新建立的remote repo默认有一个main(而不是master)branch)  
> git clone url
  
(第一次只能clone不能pull url)  
> cd folderName  

> git status  

(这时候会显示nothing to commit, working tree clean)  
> vim fileName  

(在vim里修改了文件)  
> git status  

(显示Changes not staged for commit: modified: <filename>)  
> git commit <filename>  

(会提示输入comments)  
> git status  

(这时候又会显示nothing to commit, working tree clean。但网站并没有更新)  
> git push

(网站上会看到结果)  

# MacOS Terminal的操作方法
与Windows VS Code类似。先建立网站repo，再打开Terminal, git clone。  
把文件拷贝入文件夹种，依照网站进行初始设置。  
之后提交新的修改均可以再terminal内操作：  
> git add -A  
> git commit -m "message"  
> git push  


# 在网站上修改了，或者换了一台机器，如何同步文件呢
> git pull  

(如果remote repo跟local repo一致，会显示Already up to date)  
(以下两条指令联合使用等同于git pull)  
> git fetch  

> git merge  

(如果local和remote repo是synced，那么git fetch不显示任何结果;否则会显示download信息)  
(以下指令可以Fetch difference到临时branch)  
> git fetch origin main:temp  

> git diff temp  

(这时候会显示remote repo上与local repo上的不同的文件列表)  
> git branch  

(这时候会显示temp branch被创建了，但*还是指向main)  
> git merge temp  

(local repo会看到更新的结果)  

# Local Repo添加了一个新的文件，如何更新到Remote Repo呢 
> git status  

(这时候显示untracked files: newfile.txt)  
(这时候git commit是提示有错误的)  
> git add <filename>  

> git status  

(这时候显示changes to be committed: new file: <filename>)  
(这个命令把文件从workspace推到stage)  
> git commit  

(会提示输入comments)  
> git push  


# Local Repo修改了文件，Remote Repo也更新了，想扔掉Local文件更新到最新文件如何操作
这时候git pull会出错  
> git checkout -- filename.xxx

然后git pull就可以了  

另外，如果修改了local文件，但是不想要了，手动delete。这时候git pull仍然显示uptodate，但实际上该文件并没有出现在local文件夹下，编译会出错。   
这时候git status可以发现delete的文件。  
如果要把文件复原(为remote repo上的原始文件)，可以用如下指令：  
> git restore .  

restore的作用是丢掉工作区的改动。但是文件本身的添加或删除不会变化。  

要完全还原到某个工作区，使用如下指令：  
> git checkout HEAD :/  

# Local Repo修改了若干文件，Remote Repo也更新了。想把local更新到最新文件，然后继续local开发该如何操作
直接git pull肯定不行  
(可能的出错提示是"cannot pull with rebase: You have unstaged changes. Please commit or stash them")   
(或者这个错误："Please move or remove them before you merge. Aborting")
首先用status命令可以查看目前有哪些修改  
> git status

使用git stash可以保存当前的修改  
> git stash

这时候如果用status命令会发现修改都不见了，但是可以用stash show命令查看保存的修改  
> git stash show

这里显示的修改跟之前status命令显示的修改应该是一样的  

还可以用stash list命令查看stash列表  
> git stash list

接下来就可以pull了  
> git pull  

pull成功后，把stash的修改内容恢复，就可以继续开发了  
> git stash pop  

使用如下命令丢掉存储  
> git stash drop  

使用如下命令删除所有存储
> git stash clear :  

如果使用VSCode UI  
在vs code面板左侧，STASHES目录下选择Apply Stash或Pop Stash。两者区别是前者会保留stash，后者会删除stash。选后者就可以了。  
这时候会打开Merge Changes面板，对比两个版本。绿色是remote pull下来的版本，蓝色是local修改的版本。  
点击Resolve in Merge Editor按钮处理conflict。  

# Branch相关
## 切换Branch
Github上clone的项目默认是default branch，如果需要切换其他branch，应当如何操作?  
首先我们想知道项目一共有多少个branches：
> git branch -a

该指令列出了所有可能的branch，比如一下格式：  
remotes/origin/Lesson_0  
remotes/origin/Lesson2_2  
当前的Branch前面会带有一个*符号  
切换branch的指令如下：  
> git checkout -b Lesson_2_2  

## Branch的合并方法
### 没有冲突的合并
分为快进合并和非快进合并。  
因为没有冲突，说明用户在分支上开发完成的时候，主分支上没有其他更新。这种情况就比较简单。  
快进合并会把每个分支的提交记录都移到主分支上。这是Git的默认合并方式。  
非快进合并是把分支的所有提交记录集中生成一个记录，并移到主分支上。  

### 有冲突的合并
如果有冲突，则需要进行Merge或Rebase。  
在如下开发例子中，在C2的时候出现了两个不同分支的更新。其中C3在master上由其它作者提交，用户则是在C2基础上更新了C4。  
```
C0 <- C1 <- C2 <- C3  
             | <- C4  
```
#### 合并方法一：Merge
Merge命令会把C3和C4，以及两者共同的祖先C2进行三方合并(同时要处理可能的冲突)，最终生成C5提交
```
C0 <- C1 <- C2 <- C3 <- C5  
             | <- C4 |  
```

#### 合并方法二: Rebase
Rebase可以使某一个分支版本的修改重新应用到另一个分支版本上(相当于重新播放一次)。  
比如，用户进行了C4更新后，将C4的更新Rebase到C3上，生成C4'(此时处理冲突)
```
C0 <- C1 <- C2 <- C3 <- C4'  
             | <- C4 |  
```
用户再回到master分支(这时候是C3)。  
这时候的情况相当于用户在C3上更新了C4'，进行快速合并就可以了。  
使用Rebase的时候，如果观察历史记录，就好像C0 <- C1 <- C2 <- C3 <- C4'是串行开发，C4不存在一样。提交历史会比较整洁。  

#### 使用Rebase来合并有什么好处  
设想如下场景：有一个原作者正在积极维护的开源项目，某位用户希望参与开发。  
该用户首先下载分支，进行开发。  
当开发完成后，用户需要把代码Merge到原作者的主分支(通常这时候主分支也更新了好几个版本)。  
出于安全考虑，用户只能提交修改，Merge的工作只能由原作者完成，这时候原作者很可能需要处理很多冲突(三方合并)。  
用户也可以把自己的更新Rebase到origin/master上最新的版本上，并进行适当的整合工作(处理冲突)，然后提交。  
原作者就可以省去处理冲突的步骤，直接快进合并。  


### Reference
https://git-scm.com/book/zh/v2/Git-%E5%88%86%E6%94%AF-%E5%8F%98%E5%9F%BA  


# 子模块(submodule)相关
如何克隆含有子模块的项目。  
## 方法1，使用两条git指令  
> git submodule init  
> git submodule update

## 方法2，在clone原项目的时候假如recurse参数
> git clone --recurse-submodules xxx.git

## Reference
https://git-scm.com/book/zh/v2/Git-%E5%B7%A5%E5%85%B7-%E5%AD%90%E6%A8%A1%E5%9D%97  

# Reset相关
Reset是比较强的命令，它可以强制让本地repo更新为某一个remote分支版本   
> git reset --hard origin/some_branch  

该指令不仅更改了HEAD，还把本地工作目录一起修改了(index区也改了)，彻底还原到以前的某次提交，并且无法找回。因此使用上要特别小心。  

如果只需要修改HEAD，可以用soft参数  
> git reset --soft origin/some_branch

举例：  
如果commit之後後悔了怎么回退：  
> git reset --soft HEAD^1  

reset还有一个--mixed参数，作用是除了修改HEAD，还修改了index。  
(index就是暂存区，add之后还没commit的东西就放在这里)  

# Revert相关
git revert也有回退版本的作用。举例：  
以下指令回退到最近的第4个提交状态  
> git revert HEAD~3  


# 其他有用的GitHub指令
## count
查看项目仓库大小可以使用命令:  
> git count-objects -vH   

如果.git/objects太大了：  
網上有些指令可以瘦身。但也可以重建一個repo（這樣會丟失所有的歷史記錄）  
如果僅僅是把.git/objects裏面的大文件刪除，則會造成無法commit的結果  


## log
>git log  

会把changelist的log记录打印出来  
log里每一条changelist记录包含一串40个hex字母组成的commit号码(也叫commit hash值，是此次提交的专门id. HEAD指针指向的就是commit hash值)  
作者名字和邮箱  
日期  
文字说明  
Change-Id：一串41个hex字母组成的号码  
这些changelist会按日期降序排列(最顶上的是最新的changelist)  

## show
> git show HEAD

查看当前HEAD指向的提交记录的详细信息。该命令会显示提交的commit hash值、作者、提交日期和提交的变更内容等信息。  

如果不指定参数，默认是show HEAD，因此如下指令等价：  

> git show  



## symbolic-ref
> git symbolic-ref HEAD

显示HEAD的引用名称。默认情况下，HEAD引用的是当前所在的分支。  
比如可能获得的结果是：  
```
refs/heads/develop
```
如果打开./git/HEAD文件，可以看到一样的引用：  
```
ref: refs/heads/develop
```




## MD Color Solution
GitHub目前不支持颜色文字，下面是一个替代方案  
```diff
+ Green
- Red
! Orange
@@ Pink @@
# Gray
```
