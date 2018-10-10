---
title: "在 AWS 上搭建 Jekyll 博客"
subtitle: "利用 Github Post Receive Hook 同步"
header-img: "img/post-bg-jekyll.jpg"
tags: [Jekyll, AWS]
---
Jekyll是一个为生成静态博客非常快速且优雅的工具, [官方网站](http://jekyllrb.com/ "jekyll")资料详实, 有需要可以去学习一下, 本文会涉及它的基本应用, 但不会深入讨论日常用法. AWS是一个Amazon推出的强大的云服务, 可以被用作VPS. 本文主要讨论如何通过 Github Post Receive 实现本地与 AWS EC2 实例自动同步. 你需要确保已安装git和jekyll在你的机器上. 
<!--more-->

##创建博客

在想要存放博客的地方:

    jekyll new myblog
    
这样就创建了一个基本博客的文件系统, 其中生成了初始页面. 现在可以进入文件夹运行博客:

    cd myblog
    jekyll serve
    
现在可以通过访问`http://localhost:4000`, 查看博客. 

然后在博客文件夹里面新建一个git仓库:

    git init
    git add .
    git commit -m 'My new blog!'
    
##配置AWS EC2
除了AWS, 还有很多优秀的VPS提供商. 如果使用的是其他VPS可以跳过*申请开通*, 其余的根据系统相应调整. 

####申请开通  
创建一个AWS帐号, 按照默认设置初始化一个EC2实例. 我选择的是系统是Ubuntu, 因为比较熟悉. 不用多久, 将会获得一个Pem文件, 需要妥善保管, 这是登陆实例的唯一凭证. 还可以通过管理面板找到该实例的IP地址. 现在就可以通过SSH连接:

    ssh -i younameit.pem ubuntu@xxx.xxx.xxx.xxx


####配置Apache
```AWS默认只打开了ssh的22接口, 所以需要首先在AWS的控制面板中开放80接口.```

安装过程:

    sudo apt-get update
    sudo apt-get install tasksel
    sudo tasksel install lamp-server

这个时候访问你的公共IP就已經可以看到Apache的示例页面了. Apache默认的网站页面位置在`/var/www`, 在里面创建一个目录供博客网站使用:

    mkdir /var/www/mybog

```如果之后出现该文件夹权限不足的问题, 可以通过`sudo chmod -R 755 /var/www/myblog`解决.(感觉不是最好的办法, 但我水平不够, 望指教)```

现在要把Apache默认位置改到新建的文件夹上, 在默认配置的基础下创建自己的配置:

    sudo cp /etc/apache2/sites-available/000-default.conf /etc/apache2/sites-available/myblog.conf
    sudo vim /etc/apache2/sites-available/myblog.conf
    
现在把`/var/www`改为`/var/www/myblog`. 为测试可以先放一个`index.html`在myblog文件夹, 更换配置文件, 重启服务:

    sudo a2dissite 000-default && sudo a2ensite mysite
    sudo service apache2 reload

这时, 通过访问IP或者DNS看到刚刚放进去的`index.html`了.

####新建BARE仓库  
什么是bare仓库, 看[这里](http://git-scm.com/book/en/v2/Git-on-the-Server-Getting-Git-on-a-Server  "git-pro"). 此时在VPS里面:

    cd ~/
    mkdir myblog.git && cd myblog.git
    git init --bare
    

####创建Post-receive Hook  
接下来需要做的是创建Hook. 每次`push`到远程仓库时, Git都会执行一个叫作post-receive的shell脚本, 这里需要创建它:

    cd hooks
    touch post-receive
    nano post-receive

这时可以把以下脚本复制进来. `GIT_REPO`是刚刚创建的bare仓库, `TMP_GIT_CLONE`是临时放置clone下来的博客源码的地方, 在这里build代码后复制到大部分服务器默认的根目录`/var/www/myblog`, `PUBLIC_WWW`是最终网站代码的位置:

    #!/bin/bash -l
    GIT_REPO=$HOME/myblog.git
    TMP_GIT_CLONE=$HOME/tmp/git/myblog
    PUBLIC_WWW=/var/www/myblog

    git clone $GIT_REPO $TMP_GIT_CLONE
    jekyll build --source $TMP_GIT_CLONE --destination $PUBLIC_WWW
    rm -Rf $TMP_GIT_CLONE
    exit
    
有的地方建议修改添加文件可执行权限, 按需执行:

    chmod +x post-receive


####本地连接Remote
回到本地, 添加刚刚创建的bare远程仓库到remote, 为了方便命名为origin, 可以按需修改:

    git remote add origin ubuntu@[IP 或 DNS]:~/myblog.git
    
为了能够连接到AWS, 你需要把AWS给的证书写到本地安全钥里, 可以把`dsa`换成`rsa`:

    cat ~/.ssh/id_dsa.pub | ssh -i amazon-generated-key.pem ec2-user@amazon-instance-public-dns "cat >> .ssh/authorized_keys"
    
现在应该就可以`push`到AWS上了:

    git push origin master

##Tips
附加讲讲我在搭建这个环境是碰到的问题, 如果你已经顺利push并被自动build, 可以忽略这一节.

1. 如果出现不能识别gist liquid, 需要在AWS中安装jekyll-gist gem, 并在jekyll配置文件里注明.
    gems: [jekyll-gist]
    
2. 如果highlighter使用的是pygments, 需要在AWS中安装pygment.rb gem, 这个gem名字的后缀让我刚开始转了很多弯. 还要确保系统中安装了pygments:
    sudo apt-get install python-pygments
    
3. 注意Git分支!(这条是写给我自己看的, 可以忽略) 我之前用Github Page存管博客. 为了使用其他插件(Github只支持少量插件), 我采取的方法是将本地编译之后的代码上传到remote的master(Github 默认显示master). 使得我在本地的默认分支不是master, 而是我放置源代码的source分支. 当我在本地不小心使用了`git checkout`之后, 自己彻底凌乱了...
