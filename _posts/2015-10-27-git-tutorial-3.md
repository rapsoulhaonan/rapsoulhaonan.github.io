---
title: "Git 初学者(三)"
subtitle: "Git 远程操作"
tags: [Git]
---
本文大量参考了 阮一峰 的 [Git远程操作详解](http://www.ruanyifeng.com/blog/2014/06/git_remote.html) 。

相比前文 [Git基本用法](http://hanfu.space/learning/2015/08/26/git-tutorial/) 提到的本地操作，远程操作仅仅多出一个工作区域，即远程仓库（Remote）。<!--more-->详见下图：
![img](http://image.beekka.com/blog/2014/bg2014061202.jpg "general-view") 


## git remote
一个本地仓库可以和多个远程仓库交流。为了便于管理，Git要求每个远程主机都必须指定一个主机名。`git remote`就用于管理主机名。
不带选项的时候，git remote命令列出所有远程主机。
{% highlight bash %}
$ git remote
    origin
{% endhighlight %}
使用-v选项，可以参看远程主机的网址。
{% highlight bash %}
$ git remote -v
    origin  git@github.com:jquery/jquery.git (fetch)
    origin  git@github.com:jquery/jquery.git (push)
{% endhighlight %}
上面命令表示，当前只有一台远程主机，叫做origin，以及它的网址。

克隆版本库的时候，所使用的远程主机自动被Git命名为origin。如果想用其他的主机名，需要用git clone命令的-o选项指定。
{% highlight bash %}
$ git clone -o jQuery https://github.com/jquery/jquery.git
$ git remote
    jQuery
{% endhighlight %}
上面命令表示，克隆的时候，指定远程主机叫做jQuery。

git remote show命令加上主机名，可以查看该主机的详细信息。
{% highlight bash %}
$ git remote show <主机名>
{% endhighlight %}
git remote add命令用于添加远程主机。
{% highlight bash %}
$ git remote add <主机名> <网址>
{% endhighlight %}
git remote rm命令用于删除远程主机。
{% highlight bash %}
$ git remote rm <主机名>
{% endhighlight %}
git remote rename命令用于远程主机的改名。
{% highlight bash %}
$ git remote rename <原主机名> <新主机名>
{% endhighlight %}

##取回代码
Git的远程操作基本上就是两类：取回代码和上传代码。

####git clone
如果新建了一个空白的本地仓库，通常要克隆远程仓库到本地，这时就要用到`git clone`命令。


    $ git clone <版本库的网址>

比如，克隆jQuery的版本库。


    $ git clone https://github.com/jquery/jquery.git

该命令会在本地主机生成一个目录，与远程主机的版本库同名。如果要指定不同的目录名，可以将目录名作为`git clone`命令的第二个参数。


    $ git clone <版本库的网址> <本地目录名>

`git clone`支持多种协议，除了HTTP(s)以外，还支持SSH、Git、本地文件协议等，下面是一些例子。


    $ git clone http[s]://example.com/path/to/repo.git/
    $ git clone ssh://example.com/path/to/repo.git/
    $ git clone git://example.com/path/to/repo.git/
    $ git clone /opt/git/project.git 
    $ git clone file:///opt/git/project.git
    $ git clone ftp[s]://example.com/path/to/repo.git/
    $ git clone rsync://example.com/path/to/repo.git/

SSH协议还有另一种写法。


    $ git clone [user@]example.com:path/to/repo.git/

通常来说，Git协议下载速度最快，SSH协议用于需要用户认证的场合。各种协议优劣的详细讨论请参考[官方文档](http://git-scm.com/book/en/Git-on-the-Server-The-Protocols)。

####git fetch
当远程仓库有更新需要取回本地时，就要用到`git fetch`命令。

    $ git fetch <远程主机名>

上面命令将某个远程主机的更新，全部取回本地。默认情况下，`git fetch`取回所有分支（branch）的更新。如果只想取回特定分支的更新，可以指定分支名。

    $ git fetch <远程主机名> <分支名>

比如，取回origin主机的master分支。

    $ git fetch origin master

所取回的更新，在本地主机上要用"远程主机名/分支名"的形式读取。比如origin主机的master，就要用origin/master读取。

`git branch`命令的-r选项，可以用来查看远程分支，-a选项查看所有分支。

    $ git branch -r
    origin/master

    $ git branch -a
    * master
      remotes/origin/master

上面命令表示，本地主机的当前分支是master，远程分支是origin/master。

取回远程主机的更新以后，可以在它的基础上，使用`git checkout`命令创建一个新的分支。

    $ git checkout -b newBrach origin/master

上面命令表示，在origin/master的基础上，创建一个新分支。

此外，也可以使用`git merge`命令或者`git rebase`命令，在本地分支上合并远程分支。

    $ git merge origin/master
    # 或者
    $ git rebase origin/master

上面命令表示在当前分支上，合并origin/master。

####git pull
`git pull`命令的作用是，取回远程主机某个分支的更新，再与本地的指定分支合并。

    $ git pull <远程主机名> <远程分支名>:<本地分支名>
    
实质上，这等同于先做`git fetch`，再做`git merge`。

    $ git fetch origin
    $ git merge origin/next
    
在某些场合，Git会自动在本地分支与远程分支之间，建立一种追踪关系（tracking）。比如，在`git clone`的时候，所有本地分支默认与远程主机的同名分支，建立追踪关系，也就是说，本地的master分支自动"追踪"origin/master分支。

Git也允许手动建立追踪关系。

    git branch --set-upstream master origin/next

上面命令指定master分支追踪origin/next分支。

如果当前分支与远程分支存在追踪关系，`git pull`就可以省略远程分支名。


    $ git pull origin

上面命令表示，本地的当前分支自动与对应的origin主机"追踪分支"（remote-tracking branch）进行合并。

如果当前分支只有一个追踪分支，连远程主机名都可以省略，当前分支自动与唯一一个追踪分支进行合并。

    $ git pull

如果合并需要采用rebase模式，可以使用--rebase选项。

    $ git pull --rebase <远程主机名> <远程分支名>:<本地分支名>

##上传代码
上传之前，为了防止远程仓库代码库发生冲突，一般都需要先`git pull`，以确保上传代码没有过旧。

####git push
git push命令用于将本地分支的更新，推送到远程主机。它的格式与git pull命令相仿。

    $ git push <远程主机名> <本地分支名>:<远程分支名>
    
如果当前分支与远程分支之间存在追踪关系，则本地分支和远程分支都可以省略。


    $ git push origin

上面命令表示，将当前分支推送到origin主机的对应分支。

如果当前分支只有一个追踪分支，那么主机名都可以省略。


    $ git push

如果当前分支与多个主机存在追踪关系，则可以使用-u选项指定一个默认主机，这样后面就可以不加任何参数使用`git push`。


    $ git push -u origin master


迄今，Git初学者系列结束了。该系列仅仅介绍日常工作学习中需要的 Git 操作，如果有更高的需求或其他疑问，请参见 [Pro Git](https://git-scm.com/book/en/v2)。
