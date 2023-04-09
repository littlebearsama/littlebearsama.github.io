---
title: surface reconstruction
subtitle: 表面重建（点云到网格）
date: 2021-05-11 20:00:00
tags: [Reconstruction,surface Reconstruction]
categories: [Reconstruction,2.surface Reconstruction]
---

点云生成网格

<!--more-->

# 表面重建（点云到网格）

[[线上分享录播]基于点云数据的 Mesh重建与处理](https://mp.weixin.qq.com/s?__biz=MzI0MDYxMDk0Ng==&mid=2247487165&idx=1&sn=9d62faf73d2d5a85777e2e5d14812471&chksm=e9197550de6efc4652a893761a1a4dceb83edb66c7e023cb2cc326d46ca0fcbdb6912ff29e83&scene=21#wechat_redirect)

笔记TODO

## PCL中的表面重建

1. gp3

   greedy projection---贪婪投影算法

   [参考](https://mp.weixin.qq.com/s/FfHkVY-lmlOSf4jKoZqjEA)

   贪心投影三角化的大致流程是这样的：

   （1）先将点云通过法线投影到某一二维坐标平面内

   （2）然后对投影得到的点云做平面内的三角化，从而得到各点的拓扑连接关系。平面三角化的过程中用到了基于Delaunay三角剖分 的空间区域增长算法

   （3）最后根据平面内投影点的拓扑连接关系确定各原始三维点间的拓扑连接，所得三角网格即为重建得到的曲面模型

   

2. grid_projection

   网格投影

3. marching_cubes 移动立方体

   marching_cubes_hoppe

   marching_cubes_rbf

4. mls

   移动最小二乘法

5. organized_fast_mesh

6. poisson

   泊松重建

7. EarClipping

   [耳切法](https://blog.csdn.net/noahzuo/article/details/78478617)

   将一个普通多边形拆解成多个三角形

8. delaunay2.5D（cloudcompare）

   **所有边都是德劳内边的剖分是德劳内三角剖分**。

   1. 扫描线法（Sweepline）
   
   2. 随机增量法（Incremental）
   
      >按照随机的顺序依次插入点集中的点
      >
      >1. 首先确定vi落在哪个三角形中（或边上）
      >2. 然后将vi与三角形三个顶点连接起来构成三个三角形（或与共边的两个三角形的对顶点连接起来构成四个三角形）
      >3. 由于新生成的边以及原来的边可能不是或不再是Delaunay边，故进行边翻转来调整使之都成为Delaunay边，从而得出DT(v1,v2,...,vi)。
      >
      >
   
   3. 分治法（Divide and Conquer）
   
   三角剖分原理：
   
   1. [德劳内三角剖分](https://en.wikipedia.org/wiki/Bowyer%E2%80%93Watson_algorithm)
   2. [三种计算方法](https://www.cnblogs.com/soroman/archive/2007/05/17/750430.html)
   
   
   
   开源：
   
   [ofxDelaunay:](https://github.com/obviousjim/ofxDelaunay)
   
   [delaunay-triangulation:](https://github.com/bl4ckb0ne/delaunay-triangulation)
   
   Delaunay三角剖分算法是一种常用的算法，**它的特点是剖分结果的每个三角形都尽量接近等边三角形。**
   
   1. 平面域三角剖分、平面投影法
   
      将三维点投影到某个平面，如XY平面，然后对投影点集 作平面域的三角剖分, 最终形成的曲面三角剖
      分的点间连接关系与相应的投影点间的连接关系相同,
   
   2. 最佳平面拟合
   
      同理，只是投影到最佳拟合的平面上，再对点进行剖分。
      
      



------

# 直线或线段与mesh网格相交的计算

https://blog.csdn.net/qq_33263124/article/details/88179289



# MLS

[CUDA-MLS](https://github.com/EvangelosKatsavrias/CUDA_MLS)



# VOXELIZER

https://github.com/Forceflow/cuda_voxelizer

