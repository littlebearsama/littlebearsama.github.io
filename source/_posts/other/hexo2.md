---
title: hexo教程2
subtitle: hexo教程2
author: 小熊
date: 2020-07-19 0:23:46
tags: hexo
categories: [hexo]
---

在两台电脑上面更新hexo博客

<!--more-->

## 在两台电脑上面更新hexo博客

亲测成功：

https://blog.csdn.net/qq_30105599/article/details/118302086

在利用Hexo+Github Pages写我们的博客的时候，真正的原始Hexo文件在我们的电脑本地，而GitHub上传的只是Hexo生成的静态网页，即public文件夹里面的内容。

那么假如我们有两台电脑工作，Hexo最开始搭建在其中一台电脑上，而我们需要在另外一台电脑上同时更新我们的博客，该怎么做呢？

也就是说，我们需要实现多台电脑间博客项目的迁移与同步。为实现这一点，我们可以利用Git的分支。

### 1. 创建分支

博客搭建好后(博客搭建教程见——利用Hexo框架从零开始搭建个人博客 - 江客 (jettsblog.top))，我们在Github上创建分支

* 创建一个名为hexo的分支

* 设置hexo分支为默认分支
  将博客项目仓库的Settings->Branches->Default branch修改为hexo

### 2. 将创建的分支（hexo）的远程仓库克隆到本地

* 删去除.git文件夹以外的所有你内容
* 在克隆的仓库下分别执行以下命令更新删除操作到远程

> git add -A
> git commit -m "--"
> git push origin hexo

* 将分支克隆到本地的仓库中的.git文件夹复制到博客文件夹中

* 在博客目录下执行命令同步到远程的hexo分支

>git add -A
>git commit -m "备份Hexo(提交的描述)"
>git push origin hexo

### 3. 另一台电脑的操作
* git bash将远程仓库克隆到本地

>  git clone 仓库地址

* 然后进入项目目录，**安装依赖**，启动博客服务器，生成静态文件

  > npm install
  > hexo g
  > hexo s
  >
  > 执行以上指令后，便可以在浏览器通过http://localhost:4000/访问博客

* 另一台电脑发布文章

  > hexo clean
  > hexo g 
  >
  > hexo d

### 4. 两台电脑同步写博客
我们的博客仓库有两个分支，master分支和hexo分支

其中，master分支用于存放Hexo生成的静态资源文件，hexo分支用于存放网站的原始文件

所以，我们在一台设备上写好一篇文章或进行了博客的修改后

执行以下命令，将master中的静态资源文件更新

在博客目录下的cmd中

> hexo clean
> hexo d -g


执行以下命令，将hexo中的网站原始文件更新

* 在博客目录下的git bash中

> git pull
> git add -A
> git commit -m "描述"
> git push origin hexo

### 注意事项
**每次有新的操作的时候，别忘了在另一台电脑上更新**

>  git pull hexo

## 手贱将**GitHub Page**仓库转成私有

   将github静态博客网站转成私有后不能访问，再转回公有后还是不能访问。

解决：

1. 随便·改一下gitpage仓库的名字，rename，保存。这样静态博客仓库名字就变了。

   如原本是AAA.github.io改成了BBB.github.io

2. 将仓库名字再改回来，AAA.github.io，并且将仓库改成公有。

3. 改了一些源文件（不知道是否起作用）

4. hexo g （不知道是否起作用）

5. hexo d（不知道是否起作用）

