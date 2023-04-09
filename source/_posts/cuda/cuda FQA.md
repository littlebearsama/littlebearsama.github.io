---
title: CUDA笔记补充
subtitle: cuda FQA
author: 小熊
date: 2019-06-14 11:03:14
tags: CUDA
categories: [编程,cuda FQA]
---

* 几种同步方式

<!--more-->

### 几种同步方式

[参考](<https://blog.csdn.net/zhouliyang1990/article/details/38094709>)

1.`cudaDeviceSynchronize();`

2.`cudaThreadSynchronize();`

3.`cudaStreamSynchronize();`

1. `cudaDeviceSynchronize()`会阻塞当前程序的执行，直到所有任务都处理完毕（这里的任务其实就是指的是**所有的线程都已经执行完了kernel function**）
2. ~~`udaThreadSynchronize()`的功能和`cudaDeviceSynchronize()`基本上一样，这个函数在新版本的cuda中已经被“废弃”了，~~不推荐使用，如果程序中真的需要做同步操作，推荐使用`cudaDeviceSynchronize()`。
3. `cudaStreamSynchronize()`和上面的两个函数类似，这个函数带有一个参数，cuda流ID，它只阻塞那些cuda流ID等于参数中指定ID的那些cuda例程，对于那些流ID不等的例程，还是异步执行的。

