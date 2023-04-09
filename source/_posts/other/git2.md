---
title: GIT教程2
subtitle: GIT教程2
author: 小熊
date: 2021-01-16 19:45:21
tags: git
categories: [git]
---



<!--more-->

# GIT2---笔记补充                                                                                                                                                                                                                                                                                                                          

```
快速创建分支：
git branch test
切换分支：
git checkout test
删除分支：
git branch -d test

修改了很多文件时候，需要commit，通过添加-a来避免每个都commit
git commit -a -m "Changed some files"
查看远端：
git remote -v
移除远端关联：
1 git remote rm origin // 移除本地关联
2 git remote add origin git@github.com/example.git // 添加线上仓库
3 git push -u origin master // 注意：更改后，第一次上传需要指定 origin
```

