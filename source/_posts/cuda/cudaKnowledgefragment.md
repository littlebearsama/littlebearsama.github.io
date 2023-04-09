---
title: CUDA知识片段
subtitle: cuda Knoeledge Fragment
author: 小熊
date: 2021-01-17 22:38:14
tags: CUDA
categories: [编程,cuda]
---

<!--more-->

1. 线程模型适合用于OpenMP。进程模型适用于MPI

   在GPU环境下：CUDA使用一个线程块（block）构成网格（grid）。这可以看成是一个进程（即线程块）组成的队列（即网格），而进程之间没有通信。每一个线程块内部有很多线程以批处理的方式运行，称为线程束（warp）