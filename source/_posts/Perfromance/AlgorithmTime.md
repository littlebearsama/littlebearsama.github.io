---
title: 算法性能时间记录
subtitle: 算法性能时间记录
date: 2021-04-02 20:00:00
tags: [Performance]
categories: [Performance]
---



本机计算机硬件参数

处理器：Intel(R) Core(TM) i7-9750H CPU @ 2.60GHz   2.59 GHz

显卡：NVIDIA GeForce RTX 2060

机带RAM：16.0 GB

操作系统：window 64 位操作系统, 基于 x64 的处理器



测试内容：

一、从硬盘加载点云数据

二、CUDA-ICP算法表现

<!--more-->

# 一、加载点云数据

| 方法                                                         | 数据量                  | 时间  |
| ------------------------------------------------------------ | ----------------------- | ----- |
| LoadData（自己写的加载txt点云时间）                          | 32469（ASCII码txt文件） | 463ms |
| pcl::io::loadPCDFile()                                       | 32469（二进制pcd文件）  | 2ms   |
| PCL加载ascii码                                               | 32469（ASCII码pcd文件） | 452ms |
| utilityCore::safeGetline(this->fp_in, line);（自己写的加载txt点云时间） | 32469（ASCII码txt文件） | 304ms |



# 二、迭代最近点ICP

| 方法                                                         | 数据量                       | kdtree构建时间为              | 迭代次数 | 平均迭代时间（不算kdtree构建） | 时间      |
| ------------------------------------------------------------ | ---------------------------- | ----------------------------- | -------- | ------------------------------ | --------- |
| 自己写的ICP（由于里面多了一些多余的复制操作导致比pcl的慢？） | target:32469/source:24183    | 4ms                           | 38       | 118.8ms                        | 4516ms    |
| pcl::IterativeClosestPoint<pcl::PointXYZ, pcl::PointXYZ>     | target:32469/source:24183    | ？                            | 38       | 82.4ms                         | 3134ms    |
| 自己写的cudaicp                                              | target:32469/source:24183    | 61ms（与CPU版本的kdtree不同） | 43       | 1.186ms                        | 112.112ms |
| **自己写的cudaicp**                                          | target:600343/source:5992673 | 1207ms                        | 59       | 4.755ms                        | 1487.55ms |

