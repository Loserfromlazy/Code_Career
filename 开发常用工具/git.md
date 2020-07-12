# Git学习笔记

同生活中的许多伟大事件一样，Git 诞生于一个极富纷争大举创新的年代。Linux 内核开源项目有着为数众广的参与者。绝大多数的 Linux 内核维护工作都花在了提交补丁和保存归档的繁琐事务上（1991－2002年间）。到 2002 年，整个项目组开始启用分布式版本控制系统 BitKeeper 来管理和维护代码。

到 2005 年的时候，开发 BitKeeper 的商业公司同 Linux 内核开源社区的合作关系结束，他们收回了免费使用 BitKeeper 的权力。这就迫使 Linux 开源社区（特别是 Linux的缔造者 Linus Torvalds ）不得不吸取教训，只有开发一套属于自己的版本控制系统才不至于重蹈覆辙。他们对新的系统订了若干目标：

• 速度

• 简单的设计

• 对非线性开发模式的强力支持（允许上千个并行开发的分支）

• 完全分布式

• 有能力高效管理类似 Linux 内核一样的超大规模项目（速度和数据量）

## 一. Git的安装

### 1.安装git for windows 

下一步即可

### 2.安装TortoiseGit

安装后默认选项下启动配置画面填写姓名邮箱（无影响）

### 3.搭建私有服务器

远程仓库实际上和本地仓库没啥不同，纯粹为了7x24小时开机并交换大家的修改。GitHub就是一个免费托管开源代码的远程仓库。但是对于某些视源代码如生命的商业公司来说，既不想公开源代码，又舍不得给GitHub交保护费，那就只能自己搭建一台Git服务器作为私有仓库使用。

1. 安装环境

   > yum -y install curl curl-devel zlib-devel openssl-devel perl cpio expat-devel gettext-devel gcc cc

2. 下载tar.gz文件传到linux中并解压缩   使用tar -zxvf xxx.tar.gz解压

3. 使用make prefix=/usr/local/git all编译   其中prefix中的是目录 

4. 使用make prefix=/usr/local/git/gitInstallDir install安装    其中prefix中的是目录 

5. 创建git用户adduser -r -d /home/git -m git    创建密码passwd git

6. 创建git仓库使用 git --bare init /home/git/repository

使用--bare参数，创建不带工作区的纯仓库

> 注意：如果不使用“--bare”参数，初始化仓库后，提交master分支时报错。这是由于git默认拒绝了push操作，需要.git/config添加如下代码：
>
> [receive]
>
> ​      denyCurrentBranch = ignore

## 二. 使用git管理文件版本

### 1.使用TortoiseGit右键操作即可

### 2.使用GitBash管理

创建仓库**$ git init**

添加文件 **$git add test.txt**

将暂存区更新到仓库**$git commit –m ‘日志信息 ’**

删除操作**$git rm file** 如果删除之前修改过并且已经放到暂存区域的话，则必须要用强制删除选项 **-f**

查看项目的当前状态**$git status** 加了 **-s** 参数，以获得简短的结果输出。如果没加该参数会详细输出内容

显示已写入缓存与已修改但尚未写入缓存的改动的区别**$git diff** :

> - 尚未缓存的改动：**git diff**
> - 查看已缓存的改动： **git diff --cached**
> - 查看已缓存的与未缓存的所有改动：**git diff HEAD**
> - 显示摘要而非整个 diff：**git diff --stat**

用于取消已缓存的内容**$git reset HEAD**

### 3.工作区和暂存区

Git和其他版本控制系统如SVN的一个不同之处就是有暂存区的概念。

什么是工作区（Working Directory）？

工作区就是你在电脑里能看到的目录，比如我的reporstory文件夹就是一个工作区。

有的同学可能会说repository不是版本库吗怎么是工作区了？其实repository目录是工作区，在这个目录中的“.git”隐藏文件夹才是版本库。这回概念清晰了吧。

Git的版本库里存了很多东西，其中最重要的就是称为stage（或者叫index）的暂存区，还有Git为我们自动创建的第一个分支master，以及指向master的一个指针叫HEAD。

我们把文件往Git版本库里添加的时候，是分两步执行的：

第一步是用git add把文件添加进去，实际上就是把文件修改添加到暂存区；

第二步是用git commit提交更改，实际上就是把暂存区的所有内容提交到当前分支。

 因为我们创建Git版本库时，Git自动为我们创建了唯一一个master分支，所以，现在，git commit就是往master分支上提交更改。你可以简单理解为，需要提交的文件修改通通放到暂存区，然后，一次性提交暂存区的所有修改。

## 三. 远程仓库

### 1.ssh协议

SSH 为 Secure Shell（安全外壳协议）的缩写，由 IETF 的网络小组（Network
Working Group）所制定。SSH 是目前较可靠，专为远程登录会话和其他网络服务提供安全性的协议。利用
SSH 协议可以有效防止远程管理过程中的信息泄露问题。

### 2.基于ssh密钥的安全验证

使用ssh协议通信时，推荐使用基于密钥的验证方式。你必须为自己创建一对密匙，并把公用密匙放在需要访问的服务器上。如果你要连接到SSH服务器上，客户端软件就会向服务器发出请求，请求用你的密匙进行安全验证。服务器收到请求之后，先在该服务器上你的主目录下寻找你的公用密匙，然后把它和你发送过来的公用密匙进行比较。如果两个密匙一致，服务器就用公用密匙加密“质询”（challenge）并把它发送给客户端软件。客户端软件收到“质询”之后就可以用你的私人密匙解密再把它发送给服务器。

### 3.ssh密钥生成

使用git bash 执行命令,生命公钥和私钥

> ssh-keygen -t rsa

在C:\Users\用户名\.ssh中生成公钥和私钥其中id_rsa是私钥id_rsa.pub是公钥

### 4.同步到GitHub

在GitHub的setting中添加自己的公钥（最好用notepad++等工具打开公钥）

然后再git bash中执行：

$git remote add origin git@github.com:自己的用户名/自己的仓库名.git 			——此为ssh方式连接GitHub（推荐）

$git remote add origin https://github.com/自己的用户名/自己的仓库名.git 		——此为http方式连接GitHub

$git push -u origin master

两种方式都可以但使用http会一直登录非常繁琐

> PS：如果出现remote origin already exists错误需要使用下面的命令
>
> git remote rm origin

$git push -f origin master//此命令会忽略版本不一致等问题，强制将本地库上传的远程库，-f会用本地库覆盖掉远程库，如果远程库上有重要更新，或者有其他同伴做的修改，也都会被覆盖

### 5.从远程仓库克隆

使用此命令即可$ git clone git@github.自己的用户名/自己的仓库名.git

### 6.连接自己的私有服务器

$ git remote add origin git@192.168.25.128:repo

或$ git remote add origin ssh://git@192.168.25.128/home/git/repo

## 四. 分支管理

​	在我们每次的提交，git都把他们串成一条时间线，这条时间线就是一条分支。截止到目前，只有一条时间线，在Git里，这个分支叫主分支，即master分支。HEAD指针严格来说不是指向提交，而是指向master，master才是指向提交的，所以，HEAD指向的就是当前分支。

​	每次提交，master分支都会向前移动一步，这样，随着你不断提交，master分支的线也越来越长。当我们创建新的分支，例如dev时，Git新建了一个指针叫dev，指向master相同的提交，再把HEAD指向dev，就表示当前分支在dev上。所以，Git创建一个分支很快，因为除了增加一个dev指针，改改HEAD的指向，工作区的文件都没有任何变化！

​	不过，从现在开始，对工作区的修改和提交就是针对dev分支了，比如新提交一次后，dev指针往前移动一步，而master指针不变。

​	假如我们在dev上的工作完成了，就可以把dev合并到master上。Git怎么合并呢？最简单的方法，就是直接把master指向dev的当前提交，就完成了合并。所以Git合并分支也很快！就改改指针，工作区内容也不变！

​	合并完分支后，甚至可以删除dev分支。删除dev分支就是把dev指针给删掉，删掉后，我们就剩下了一条master分支。

### 1.使用TortoiseGit实现分支管理

#### **创建分支**

在本地仓库文件夹中点击右键，然后从菜单中选择“创建分支”。

如果想创建完毕后直接切换到新分支可以勾选“切换到新分支”选项或者从菜单中选择“切换/检出”来切换分支：

![img](https://img-blog.csdn.net/20180702121702604?watermark/2/text/aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L0hlbGxvX1dvcmxkX1FXUA==/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70)

分支创建完成后，右键，发现git的提交指向了刚创建的branch分支了

下面就新增一个功能 DeploymentController.java ，及将Employee中的修改方法的返回值封装成一个工具类，如下图：

![img](https://img-blog.csdn.net/20180702143436613?watermark/2/text/aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L0hlbGxvX1dvcmxkX1FXUA==/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70)

提交到本地分支 branchOne ，并Push远程仓库。远程仓库的branchOne中的内容，如下图：

![img](https://img-blog.csdn.net/20180702144212238?watermark/2/text/aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L0hlbGxvX1dvcmxkX1FXUA==/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70)

假如假如新功能后，系统上线运行发生故障，短时间内无法处理，需要回到之前的代码，这儿就可以通过切回到主干master即可。点右键 -> TortoiseGit -> Switch/Checkout（切换检出） ，选择master后，点击 “ OK ” 即可

#### 合并分支

合并分支主要应用到项目的并行开发的情况，在两个项目小组或多个项目小组的开发工作完成并测试无误后，进行项目的合并工作，这一点就体现出了分支的最强大的地方。

将 branchOne 分支合并到主干 master，右键 -> TortoiseGit -> Merge（合并）

选中需要合并的分支，合并信息可选，然后点击 OK 即可，如下图：



![img](https://img-blog.csdn.net/20180702165407165?watermark/2/text/aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L0hlbGxvX1dvcmxkX1FXUA==/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70)

在日志中，可以常看分支合并的详细内容

![img](https://img-blog.csdn.net/20180702170123428?watermark/2/text/aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L0hlbGxvX1dvcmxkX1FXUA==/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70)

当然合并分支后，由于其他原因，有不想合并了，可以通过回退的方式，还原原来主干master的原代码状态，在日志中选中需要回退的版本右键 -> Reset "master" to this 即可。

#### 删除分支

如果分支不想用了，可以直接删除，这儿为了体现出效果（再次将分支进行合并），分支删除后，代码将还原到主干master，右键 ->  TortoiseGit -> Merge（合并） 点击 Branch 后面的选项，如下图：

![img](https://img-blog.csdn.net/20180702174124637?watermark/2/text/aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L0hlbGxvX1dvcmxkX1FXUA==/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70)

在弹出框中，选择需要删除的分支右键 -> Delete remote branch 即可

### 2.使用gitbash

**创建分支命令**：没有参数时，**git branch** 会列出你在本地的分支。

```
git branch (branchname)
```

**切换分支命令:**

```
git checkout (branchname)
```

**合并分支命令:**   git checkout -b (branchname) 命令来创建新分支并立即切换到该分支下

```
git merge 
```

**删除分支命令：**

```
git branch -d (branchname)
```

eg:

~~~
$ git branch testing
$ git branch
* master
  testing
我们也可以使用 git checkout -b (branchname) 命令来创建新分支并立即切换到该分支下
$ git checkout -b newtest
Switched to a new branch 'newtest'
------------------------------------
$ ls
README
$ echo 'lalala' > test.txt
$ git add .
$ git commit -m 'add test.txt'
[master 3e92c19] add test.txt
 1 file changed, 1 insertion(+)
 create mode 100644 test.txt
$ ls
README        test.txt
$ git checkout testing
Switched to branch 'testing'
$ ls
README
当我们切换到 testing 分支的时候，我们添加的新文件 test.txt 被移除了。切换回 master 分支的时候，它们有重新出现了。
-----------------------------------------------
$ git branch
* master
  testing
$ git branch -d testing
Deleted branch testing (was 85fc7e7).
$ git branch
* master
-------------------------------------------------
$ git branch
* master
  newtest
$ ls
README        test.txt
$ git merge newtest
将 newtest 分支合并到主分支去
~~~

### 3.冲突合并

**TortoiseGit**

例如在master分支中对mytest.txt进行编辑，然后提交到版本库。

切换到dev分支，对mytest.txt进行编辑，最后进行分支合并，例如将dev分支合并到master分支。需要先切换到master分支然后进行分支合并。

出现版本冲突。冲突需要手动解决。

在冲突文件上单机右键选择“解决冲突”菜单项。

把冲突解决完毕的文件提交到版本库就可以了。

**GitBash**

一个合并冲突就出现了，接下来我们需要手动去修改它。在 Git 中，我们可以用 git add 要告诉 Git 文件冲突已经解决。

## 五.在IDEA使用git

### **配置git**

安装好IntelliJ IDEA后，如果Git安装在默认路径下，那么idea会自动找到git的位置，如果更改了Git的安装位置则需要手动配置下Git的路径。

选择File→Settings打开设置窗口，找到Version Control下的git选项，选择git的安装目录后可以点击“Test”按钮测试是否正确配置。

### **设置GitHub账号**

接下来为github设置账号密码：
菜单->settings->Version Control->GitHub->Add account
设置好了之后，IDEA的git准备工作就做好了

### **用IDEA从github上pull一个现成的项目到本地（Clone）**

checkout
菜单->VCS->Chekout from Version Control->Github（或者Git），之后输入URL，点击test即可
这里的URL就是GitHub上的项目git地址，然后点击 Clone。

![在这里插入图片描述](https://img-blog.csdnimg.cn/20190608204028265.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9pbG92ZW1zcy5ibG9nLmNzZG4ubmV0,size_16,color_FFFFFF,t_70)

### **本地项目push到git仓库上（push）**

接着在本地创建一个项目helloworld，并且新建一个Java类

菜单->VCS->import into Version Control->Create Git Repository->e:\project\helloworld-OK

把项目加入到本地仓库的stage区暂存右键项目->Git->Add

将暂存的项目提交到本地仓库然后提交到远程仓库(IDEA里将这两步骤简化为一步 即Commit and Push)

右键项目->Git->Commit Directory之后弹出如图所示的窗口，在Commit Message 输入日志信息， 然后点击 Commit And Push



![在这里插入图片描述](https://img-blog.csdnimg.cn/20190608205652705.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9pbG92ZW1zcy5ibG9nLmNzZG4ubmV0,size_16,color_FFFFFF,t_70)

### 同步代码

如果需要从服务端同步代码可以使用工具条中的“update”按钮

### 将代码提交到Github

使用git bash将项目推到GitHub

## 六.在eclipse使用git

Eclipse从LUNA版本开始默认支持了GIT客户端，可以在导航菜单中`windows --> preferences`搜索git查看git相关配置。
Eclipse中对于git的操作基本都在右键菜单Team中。




