# Git状态区  
- **`工作区`**(Working Directory，VS Code里叫Changes区)-直接编辑的地方，比如记事本打开的文件，肉眼可见，直接操作。add可以把文件增添(Add)到暂存区。  
- **`暂存区`**(Stage/Index，VS Code里叫Stages Change区)-数据暂时存放的区域。暂存区的数据可以commit到版本区。  
- **`版本区`**(Commit History)-存放已经commit的数据的区域。push的时候就是把这里的数发到remote repo。   

# Git的基本概念  
- **`分支(Branch)`** 每个git仓库都有一个主代码库(master或main，也称作代码库的主分支。当第一次创建仓库的时候会选择master或main作为其默认名字)。从主代码库上可以分支，形成支线代码库。开发人员可以并行在不同的分支上工作，比如开发新功能、修复bug或进行试验。每个分支都代表一条独立开发线，并不会影响其他分支或主代码库。
- **`提交(Commit)`** 当在某个分支上工作，比如修改了代码之后， 需要创建提交Commit来记录其更改。一系列Commit记录就组成了一个分支的历史记录。每条Commit记录也称作一个Commit对象。每个Commit对象都有唯一的标识符。作者或其他人可以查看或回滚到不同的Commit版本。  
当代码提交之后，就不能用git status查看哪些文件修改了。但可以用git show来查看最新的commit改动了哪些地方。  
Commit的本质就是把本地的HEAD指针向前走一步。  
如果提交之后还没push的时候想反悔，可以git reset HEAD^1，也就是把HEAD指针往回走一步，这样就可以取消commit了。  
- **`推送(Push)`** 当完成了某一个提交Commit之后，可以通过推送Push来把修改应用到某一个分支Branch上。  
- **`拉取(Pull)`** 这个操作就是Push倒过来，它可以把某个分支的提交拉到本地。  
- **`HEAD`** 分支的本质其实就是个指向commit对象的指针。HEAD也是一个指针，一个指向工作区中本地分支的指针。HEAD指向的分支，就是你当前正在工作的分支。或者把HEAD理解为一个特殊的分支指针，它就是某个分支的别名。  
- **`Origin`** Origin是远程仓库，非本地仓库。Origin是个仓库的别名，它会被翻译成某个具体的仓库名字。比如，如下两个命令等价：  
> git push origin main  
> git push git@github.com:some-name/repo-name.git main  

可以看到，使用origin别名可以省下很多打字和记忆时间。  
除了本地有个HEAD指针，远程仓库Origin上也有一个HEAD指针，叫做origin/HEAD，它是远程默认分支的别名。  
当使用git branch -a的时候，可以显示当前HEAD和远程仓库分别指向哪个分支。  
> git branch -a

```
* develop
  remotes/origin/HEAD -> origin/develop
  remotes/origin/develop
  remotes/origin/master
```
上述结果解析：  
- *号表示HEAD指向本地的develop分支  
- 远端仓库有origin/develop和origin/master两个分支  
- origin/HEAD指向的是origin/develop  

如果要从本地(HEAD指向的develop)往远端(origin/HEAD指向的origin/develop)推送commit，可以用如下指令： 
- git push origin HEAD:/refs/for/develop  

当使用git log指令后，会显示本地commit历史记录，同时也会显示分支信息。比如：  
> git log

```
commit #commit_id0 (HEAD -> develop, origin/develop, origin/HEAD)
commit #commit_id1
commit #commit_id2
...
```
上述结果解析：  
- commit_id0是本地最新的commit  
- HEAD -> develop表示本地HEAD指向develop分支，它跟commit_id0一致  
- origin/develop是远程仓库的develop分支，它的最新版本也是commit_id0  
- origin/HEAD是远程仓库origin/develop的别名，它的版本也是commit_id0  

当使用git status命令查看状态的时候，会描述本地仓库的HEAD分支和远端仓库orign/HEAD分支的对比  
> git status

会获得如下结果  
```
On branch develop
Your branch is up to date with 'origin/develop'
```
上述结果解析：  
- 因为HEAD和origin/HEAD指向commit_id一致，所以develop和origin/develop一致  

假设本地工作区有了更改，重新run以上命令。git branch -a和git log的结果不改变。git status会显示"changes not staged for commit"。  
Add文件之后，git branch -a和git log的结果也不改变。git status会显示文件已经stage了，"Changes to be committed"。  
git commit之后，会生成新的commit_id。因此git branch -a不变。git log会显示HEAD->develop在最新的commit_id上；但origin/develop和origin/HEAD会指向另外的commit_id。git status显示"Your branch is ahead of 'origin/develop' by 1 commit"。这时候可以git push了。  
假如之前使用的origin版本不是最新的，这时候会有divergent，log里不再会显示origin相关指针。这里需要注意一下。git status显示"Your branch and 'origin/develop' have diverged, and have 1 and 1 different commits each"。总之是不太妙。  

总结就是HEAD是指向分支的分支别名，它代表了当前正在工作的分支。除了本地的HEAD指针，在远程仓库还存在一个叫做origin/HEAD的指针。  

- **`FETCH_HEAD`** 这个变量名字带一个HEAD，顾名思义也是一个指针。它的区别是它指向的是从remote repo上拉下来的某一个commit。  
FETCH_HEAD所指向的commit id可以通过查阅.git/底下的同名文件获得。  
当使用git fetch的时候，某一个commit的files会从remote repo上拉到本地(.git/FETCH_HEAD也会相应更新)。  
git pull的过程就是先git fetch，然后git merge。  

# Branch命令
## 切换Branch
Github上clone的项目默认是default branch，如果需要切换其他branch，应当如何操作?  
首先我们想知道项目一共有多少个branches：
> git branch -a

该指令列出了所有可能的branch，比如以下格式：  
remotes/origin/Lesson_0  
remotes/origin/Lesson2_2  
当前的Branch前面会带有一个*符号  

如果不加-a参数，仅使用
> git branch

就会仅列出本地branch，而不会列出remote branch

切换branch的指令如下：  
> git checkout -b Lesson_2_2  

或者
> git checkout develop


checkout的作用是切换到一个branch  
checkout -b的作用是创建一个新的branch，并且切换到那个branch  




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

如果要更新到某一个特定的commit，需要知道其commit id(一个40位hex)  
> git reset --hard xxxxxx...xxx  

举例：  
如果commit之後後悔了怎么回退：  
> git reset --soft HEAD^1  

reset还有一个--mixed参数，作用是除了修改HEAD，还修改了index。  
(index就是暂存区，add之后还没commit的东西就放在这里)  

一个例子详细说明--soft, --hard和--mixed参数的区别：  
要分清三者的区别，首先要复习一下git本地的三个分区：
基本原理：在修改了local repo后，change一开始是unstage的。checkin之前首先要add to stage(从工作区(work/change)移动到暂存区(stage/index))。然后再commit to commit区。  
之所有要专门分一个stage/index区，是希望在checkout remote repo的时候，有个区域可以暂时存放remote repo的资料，而仍旧保留本地work区的文件不被改变。  
HEAD指针在git里指向的是当前checkout的commit。HEAD指向的branch称为current branch。  

--soft和--mixed的区别是是否改变stage/index区的内容。  
比如，给定以下的commit链：  
\- A - B - C
HEAD指向C。当前Index/Stage的内容跟C保持一致。
>git reset --soft B

所做的事情是让HEAD指向B，但Index/Stage里的内容还是C。  
这时候git status会显示C diff的内容出现在Index/Stage区。  
(同时会提示your branch is behind 'xxxx' by 1 commit)  
这时候git commit就会生成一个新的commit内容到commit区，其跟C的内容完全相同。  

再看--mixed的情况。假如使用--mixed参数而不是--soft参数(或者不加任何参数，默认就是--mixed)。
结果就是不仅HEAD指向了B，连Index/Stage区的内容也更新为B了。  
这时候git status不会出现新内容，git commit也不会生成任何新的commit内容。  
(如果之前work区有改动的话，git status仍会标识这些为unstage内容)

最后是看--hard的情况。它不但修改了HEAD和Index/Stage区，还把Work区也改成B了。  
(注意如果work里面有untrack的新文件，需要手动删除)  
因此可以看出，如果你在本地做了任何修改，--hard将把它们立刻抹去，因此使用这个命令需要特别小心。  
在做--hard之前，建议先用git status检查有没有unstaged的改动。  

总结就是，--soft discard last commit。  
--mixed discard last commit and add。  
--hard discard last commit, last ladd and any changes in the work area。  

注意1: 当使用reset在各个commit版本之间切换的时候，如果是每个版本都有的文件都会变动；但若是该commit版本没有的文件，不会被删除，而是会被mark成"Untracked files"  
注意2：当使用reset切回旧的commit版本后，再pull的话，只会pull到该branch最新的版本。如果还没有merge的commit版本不能被pull到  

## reference
https://stackoverflow.com/questions/3528245/whats-the-difference-between-git-reset-mixed-soft-and-hard  


# Revert相关
git revert也有回退版本的作用。举例：  
以下指令回退到最近的第4个提交状态  
> git revert HEAD^3  

Revert和Reset的区别  
- git revert是用一次新的commit来回滚之前的commit，git reset是直接删除指定的commit  
- git reset 是把HEAD向后移动了一下，而git revert是HEAD继续前进，只是新的commit的内容和要revert的内容正好相反，能够抵消要被revert的内容  
- 如果回退分支的代码以后还需要的情况则使用git revert， 如果分支是提错了没用的并且不想让别人发现这些错误代码，则使用git reset  

# 其他有用的GitHub指令
## count
查看项目仓库大小可以使用命令:  
> git count-objects -vH   

如果.git/objects太大了：  
網上有些指令可以瘦身。但也可以重建一個repo（這樣會丟失所有的歷史記錄）  
如果僅僅是把.git/objects裏面的大文件刪除，則會造成無法commit的結果  


## log
> git log  

会把changelist的log记录打印出来  
log里每一条changelist记录包含一串40个hex字母组成的commit号码(也叫commit hash值，是此次提交的专门id. HEAD指针指向的就是commit hash值)  
同时括号里会表明当前HEAD指向哪一个commit hash值  
作者名字和邮箱  
日期  
文字说明  
Change-Id：一串41个hex字母组成的号码  
这些changelist会按日期降序排列(最顶上的是最新的changelist)  

> git reflog

显示HEAD变化的历史记录。会显示pull, reset等改变HEAD的行为的log。  


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

## rev-parse
前面说过，HEAD指向的其实是commit hash值，使用如下指令可以把HEAD的commit hash值打印出来：  
> git rev-parse HEAD

屏幕上会打印40位的commit hash值。  
也就是git show里面现实的那个commit hash值。  
也是git log里面第一个的那个commit hash值。  


## MD Color Solution
GitHub目前不支持颜色文字，下面是一个替代方案  
```diff
+ Green
- Red
! Orange
@@ Pink @@
# Gray
```



# 实战：从init开始(本地有一个工作中待上传的文件夹)  
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


# 实战：被要求github.com的Username和Password怎么办  
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


# 实战：从clone开始(使用已存在的Remote Repo的情況)
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

# 实战：MacOS Terminal的操作方法
与Windows VS Code类似。先建立网站repo，再打开Terminal, git clone。  
把文件拷贝入文件夹种，依照网站进行初始设置。  
之后提交新的修改均可以再terminal内操作：  
> git add -A  
> git commit -m "message"  
> git push  


# 实战：在网站上修改了，或者换了一台机器，如何同步文件呢
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

# 实战：Local Repo添加了一个新的文件，如何更新到Remote Repo呢 
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


# 实战：Local Repo修改了文件，Remote Repo也更新了，想扔掉Local文件更新到最新文件如何操作
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

# 实战：Local Repo修改了若干文件，Remote Repo也更新了。想把local更新到最新文件，然后继续local开发该如何操作
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
