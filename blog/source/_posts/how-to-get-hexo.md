---
title: GitHub + Hexo搭建个人博客
date: 2017-07-25 13:32:54
tags:
---
<img src="http://i.imgur.com/cyX1bBo.jpg" style="zoom:50%"/>
<!--more-->
# 前言

很早就有搭建博客的想法，最近终于有时间，去网上去搜相关教程，结果各种坑，决定写一篇。

<img src="http://i.imgur.com/lgL0TVv.jpg" style="zoom:10%"/>

假设你已经拥有了GitHub以及ssh认证等，去gitHub新建一个仓库，仓库名必须为  `yourusername.github.io`

如：


![](http://i.imgur.com/1bvOqTh.png)

创建完了之后先不用管他。


## 正式安装Hexo
在安装Hexo之前需要安装Node.js，直接去官网download无脑安装即可。这里我们主要说Hexo。

首先创建一个文件夹存放hexo配置，如Blog。

**记住在执行hexo指令前，必须是在刚才存放hexo配置的文件目录下，否则命令无效，所依我们一路cd 到刚才的Blog文件夹下**

执行如下命令安装Hexo：

    sudo npm install -g hexo

初始化然后，执行init命令初始化hexo,命令：

    hexo init

hexo的指令非常少而且简单，现在我们来测下hexo是否成功安装

    hexo g
本地启动
我们来预览调试
    hexo s
浏览器输入 `http://localhost:4000`

#### 现在回到我们的github
1. 找到你的仓库，`yourusername.github.io`
2. 点击仓库的`SETTING`选项卡
3. 下拉到`GitHub Pages`选项中，点击如下图按钮`Choose theme`


![](http://i.imgur.com/xDRJvHT.png)

点击之后随便选个theme，随后关闭我们的GitHub.

## 配置Hexo
进入我们的Blog文件夹，这里的根目录下有个`_config.yml`文件，这个叫做**站点配置文件**可以理解成全局的配置文件。

现在来设置这个配置文件

这里重点修改内容是整个配置文件的最下面的这么一段：

    deploy:

     type: git

     repo: git@github.com:fay77/fay77.github.io.git

     branch: master

<img src="http://i.imgur.com/FtgFnxj.jpg" style="zoom:%"/>

**需要注意的是所有的冒号后面都要加个空格，否则无效，repo后面跟的是ssh地址，http地址会有问题**

然后执行命令：

    npm install hexo-deployer-git --save

到此为止基本配置都结束了，我们开始部署到gitHub上，执行命令：

    hexo deploy


然后再浏览器中输入[http://fay77.github.io/](http://fay77.github.io/) 就行了，我的github的账户叫fay77,把这个改成你github的账户名就行了

### Notes
每次部署都按以下三步来进行


- `hexo clean`
- `hexo g`
- `hexo d`

## 域名绑定
以上步骤生成的博客地址均以 `xxx.github.io`为地址的，你也可以更改这个地址，绑定你的域名。

### 购买域名
我用的是dnspod，进入搜索你想要的域名，然后支付。这里很简单不多说。


点击  **解析** 进入

![](http://i.imgur.com/ez4ypru.png)

点击  添加记录 选A

![](http://i.imgur.com/LCTN8zm.png)

这里的ping需要打开cmd执行命令 
   ` ping -t fay77github.io`
得到一串ip地址，复制并且粘贴到记录值一栏。 点击确定。

再次点击添加记录 选CNAME 
记录值这次填你的github.io那个地址就可以了。


## GitHubPage绑定你的域名
在blog文件下
![](http://i.imgur.com/R6M0kTM.png)
新建CNAME，里面填入你的域名。 然后执行命令
    `hexo g`
    `hexo d`


<img src="http://i.imgur.com/MVzNTim.jpg" style="zoom:50%"/>

大功告成。 访问你的域名就可以了。


----------
## 多台电脑同时编写博客
上面都完成之后你应该正在愉快的编写博客了，当你回家想写点什么，fuck博客的资料全在单位电脑咋办
![](http://i.imgur.com/2ZJJxv2.jpg)


### 利用分支解决多电脑同步博客的问题
首先我们进入到已经`git clone`了`xxx.github.io`仓库了的文件夹中，命令行敲起来，墙裂推荐cmder命令行，wins下的cmd简直跟屎一样。


二话不说就是一个`git pull`更新master分支上的内容，随后创建本地分支`hexo` ，这个名字你可以随便取

`git checkout -b hexo`创建并且切换到我们本地的`hexo`分支下，我们要在这个分支下管理所有的博客资源文件，将我们上面，不对是很上面创建的Blog文件整个复制到`hexo`分支所在的文件夹目录下

执行`git status` 会发现blog文件下的东西显示未添加到git， 这里有个小坑

**如果git提示某些文件无法add，我们只要进入那个文件下找到 `.git` 文件并且删除即可**
然后`commit`这些都是git常用指令了，不多说。

接下来就是提交blog整个文件到远端了

执行`git push origin hexo:hexo` 这里我将远端的分支也命名为`hexo`了，所以看起来有点变扭，建议命名为`remote_hexo`用以区分远端和本地。

到这一步基本就都ok了。  然后我们来看看有分支下的情况下如何更新博客的：

1. 进入到`git clone`了的文件目录下，找到`.md`也就是你的博客文章，然后修改。
2. `git add` `git commit` `git push origin hexo`将修改的博文推送到远端
![](http://i.imgur.com/ppGSYAf.png)
你会发现修改的博客文章状态改变了


![](http://i.imgur.com/WffikCl.png)

3. 执行`hexo g` `hexo d`,hexo会自动将`.md`文章转成html等推送到远端的`master`分支。

建议去github将`hexo`分支设置为默认分支，这样我们只需要手动管理`hexo`分支即可，`master`就交给了`hexo`去自动管理了。





    

