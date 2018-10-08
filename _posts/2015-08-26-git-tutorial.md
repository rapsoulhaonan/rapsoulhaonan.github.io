---
title: "Git 初学者(一)"
subtitle: "Git 基础与基本用法"
tags: [Git]
---
这篇文章要说的是我在工作中需要的git的用法。 如果你稍微理解git的工作原理，这篇文章能够让你理解的更透彻。  
<!--more-->

## Git基础  

### 记录快照
Git 保存数据是对文件系统的一组快照。 每次你提交更新时，它主要对当时的全部文件制作一个**快照**。如果文件没有修改，Git 只保留一个链接指向之前存储的文件。  

![img](https://git-scm.com/book/en/v2/book/01-introduction/images/snapshots.png "title")  

### 三个工作区域
Git 有三个区域：工作目录（Working Directory），暂存目录（Stage or Index），仓库（Git directory）。
基本的 Git 工作流程如下：
 
1. 在工作目录中修改文件。  
2. 暂存文件，将文件的快照放入暂存区域。  
3. 提交更新，找到暂存区域的文件，将快照永久性存储到 Git 仓库目录。  

工作目录就是你正在耕耘的项目文件夹，当你向暂存目录提交时，会以快照的形式被存放其中。提交后若修改了工作目录中的文件，再次提交会生成新的快照覆盖到暂存目录。  
仓库，是 Git 中最重要的部分，由图中绿色块组成，数据结构看起来像一个单向链表，但后面会提到其实是树。每一个节点都是印有哈希值的快照组，也就是项目的一个版本。仓库中有一个或多个指针，代表分支，指向某一个节点。只有一个 HEAD 指针指向正在工作的分支。  
![img](http://marklodato.github.io/visual-git-guide/conventions.svg "title")  

当你从暂存目录向仓库提交时，Git 用暂存区域的文件创建一个新的提交，并把此时的节点设为父节点。然后把当前分支指向新的提交节点。下一图中，当前分支是master。 在运行命令之前，master指向ed489，提交后，master指向新的节点f0cec并以ed489作为父节点。下二图中，在master分支的祖父节点maint分支进行一次提交，生成了1800b。  
![img](http://marklodato.github.io/visual-git-guide/commit-master.svg "title")  
![img](http://marklodato.github.io/visual-git-guide/commit-maint.svg "title")   

## 基本用法[^1]

[^1]: 引自[图解Git](https://marklodato.github.io/visual-git-guide/index-zh-cn.html "图解Git").  

```
目前为止，文章还未涉及命令行的操作，请尽量理解前文的逻辑再往下细读。  
```  

![img](https://marklodato.github.io/visual-git-guide/basic-usage.svg "title")  
上面的四条命令在工作目录、暂存目录(也叫做索引)和仓库之间复制文件。

*  `git add files` 把当前文件放入暂存区域。  
*  `git commit` 给暂存区域生成快照并提交。
*  `git reset -- files` 用来撤销最后一次`git add files`，你也可以用`git reset` 撤销所有暂存区域文件。
*  `git checkout -- files` 把文件从暂存区域复制到工作目录，用来丢弃本地修改。  

你可以用 `git reset -p`, `git checkout -p`, or `git add -p`进入交互模式。

也可以跳过暂存区域直接从仓库取出文件或者直接提交代码。  
![img](https://marklodato.github.io/visual-git-guide/basic-usage-2.svg "title")
   
*  `git commit -a` 相当于运行 `git add` 把所有当前目录下的文件加入暂存区域再运行。  
*  `git commit files` 进行一次包含最后一次提交加上工作目录中文件快照的提交。并且文件被添加到暂存区域。  
*  `git checkout HEAD -- files` 回滚到复制最后一次提交。  

### diff  
![img](https://marklodato.github.io/visual-git-guide/diff.svg "title")

### checkout
`checkout`命令用于从历史提交（或者暂存区域）中拷贝文件到工作目录，也可用于切换分支。

当给定某个文件名（或者打开-p选项，或者文件名和-p选项同时打开）时，git会从指定的提交中拷贝文件到暂存区域和工作目录。比如，git checkout HEAD~ foo.c会将提交节点HEAD~(即当前提交节点的父节点)中的foo.c复制到工作目录并且加到暂存区域中。（如果命令中没有指定提交节点，则会从暂存区域中拷贝内容。）注意当前分支不会发生变化。  
![img](https://marklodato.github.io/visual-git-guide/checkout-files.svg "title")  
当不指定文件名，而是给出一个（本地）分支时，那么HEAD标识会移动到那个分支（也就是说，我们“切换”到那个分支了），然后暂存区域和工作目录中的内容会和HEAD对应的提交节点一致。新提交节点（下图中的a47c3）中的所有文件都会被复制（到暂存区域和工作目录中）；只存在于老的提交节点（ed489）中的文件会被删除；不属于上述两者的文件会被忽略，不受影响。  
![img](https://marklodato.github.io/visual-git-guide/checkout-branch.svg "title")  
**_该做法尽量避免。_**如果既没有指定文件名，也没有指定分支名，而是一个标签、远程分支、SHA-1值或者是像master~3类似的东西，就得到一个匿名分支，称作detached HEAD（被分离的HEAD标识）。这样可以很方便地在历史版本之间互相切换。比如说你想要编译1.6.6.1版本的git，你可以运行git checkout v1.6.6.1（这是一个标签，而非分支名），编译，安装，然后切换回另一个分支，比如说git checkout master。  
![img](https://marklodato.github.io/visual-git-guide/checkout-detached.svg "title")  

到这里，你已经掌握了基本的本地Git的日常用法。但是现在的工作在多数情况下，处理[分支（branch）](http://hanfu.space/learning/2015/10/26/git-tutorial-2/)和[远程 Git 仓库](http://hanfu.space/learning/2015/10/27/git-tutorial-3/)的情况是不可或缺的，将在接下来的文章中详细介绍。
