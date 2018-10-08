---
title: "Git 初学者(二)"
subtitle: "Git 分支"
tags: [Git]
---
出于实用的考虑，本文略过 Git 实现分支的原理， 仅讨论日常用法。如有兴趣可以参考 [Pro Git](https://git-scm.com/book/zh/v2/Git-%E5%88%86%E6%94%AF-%E5%88%86%E6%94%AF%E7%AE%80%E4%BB%8B) 的描述。
<!--more-->

## 创建分支

如果还记得上篇文章 [Git基础](http://hanfu.space/learning/2015/08/26/git-tutorial/) 中提到的指针，那么就会知道，创建分支其实就是创建了一个新的指针。使用的指令是`git branch`：
{% highlight bash %}
$ git branch testing
{% endhighlight %}

这会在当前所在的提交对象上创建一个testing指针， 但是此时HEAD指针仍然指向原分支，所以准备在testing分支上工作前，需要切换分支。

![img](https://git-scm.com/book/en/v2/book/03-git-branching/images/head-to-master.png "head-to-master") 

## 切换分支
需要使用的指令是`git checkout`:
{% highlight bash %}
$ git checkout testing
{% endhighlight %}
这样HEAD就指向testing分支了，此时提交代码将会和master分离，产生分叉。

![img](https://git-scm.com/book/en/v2/book/03-git-branching/images/head-to-testing.png "head-to-testing") 

当然，需要将HEAD切换回master才能在master分支上提交。
![img](https://git-scm.com/book/en/v2/book/03-git-branching/images/advance-master.png "advance-master") 

以上两条命令可以合并成一句，用一个带有`-b`参数的`git checkout`命令：
{% highlight bash %}
$ git checkout -b testing
{% endhighlight %}

## 合并分支
是时候将分支的代码整合起来，需要用`git merge`命令合并分支。首先确定需要并入到哪一条分支，通常情况下会需要并入到master，所以先将HEAD切换到master：
{% highlight bash %}
$ git checkout master
Switched to branch 'master'
$ git merge iss53
Merge made by the 'recursive' strategy.
index.html |    1 +
1 file changed, 1 insertion(+)
{% endhighlight %}
![img](https://git-scm.com/book/en/v2/book/03-git-branching/images/basic-merging-1.png "basic-merging-1") 
Git 将此次合并的结果做了一个新的快照并且让master指向它。这里有一个特殊情况就是当master所指的快照是分支的父节点，那么merge操作将会直接移动master指针到分支指针的位置。
![img](https://git-scm.com/book/en/v2/book/03-git-branching/images/basic-merging-2.png "basic-merging-2") 
如果没有任何冲突，那么合并到这里就结束了。但大部分情况，代码往往不能简单的自动合并，比如两个分支均修改了同一个变量。任何因包含合并冲突而有待解决的文件，都会以未合并状态标识出来。Git 会在有冲突的文件中加入标准的冲突解决标记，这样你可以打开这些包含冲突的文件然后手动解决冲突。出现冲突的文件会包含一些特殊区段，看起来像下面这个样子：
{% highlight html %}
<<<<<<< HEAD:index.html
<div id="footer">contact : email.support@github.com</div>
=======
<div id="footer">
 please contact us at support@github.com
</div>
>>>>>>> iss53:index.html
{% endhighlight %}
此时只需要妥善处理冲突,确保已经把 `<<<<<<<` , `=======` , 和 `>>>>>>>` 这些行完全删除。在解决了所有文件里的冲突之后，对每个文件使用 git add 命令来将其标记为冲突已解决。一旦暂存这些原本有冲突的文件，Git 就会将它们标记为冲突已解决。

这就是 Git 分支的基本操作，工作中使用往往需要一定的管理策略，可以参考阮一峰的 [Git分支管理策略](http://www.ruanyifeng.com/blog/2012/07/git.html)。
