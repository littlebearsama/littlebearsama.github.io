---
title: SLAM_tutorial
subtitle: SLAM学习指南
date: 2019-07-1 15:17:57
tags: [CV-Source]
categories: [CV-Source,2.SLAM]
---

SLAM技术分类

SLAM资源

<!--more-->

## 技术分类

[slamCN](http://www.slamcn.org/index.php/%E9%A6%96%E9%A1%B5#.E4.B8.BB.E6.B5.81.E5.BC.80.E6.BA.90SLAM.E6.96.B9.E6.A1.88)

基于这两种传感器有激光SLAM和视觉SLAM。

* ### 视觉传感器

  - 稀疏法(特征点):
    - [ORB-SLAM](http://www.slamcn.org/index.php/ORB-SLAM)(单目，双目，RGBD)[[1\]](http://webdiis.unizar.es/~raulmur/orbslam/)(ORB-SLAM: a Versatile and Accurate Monocular SLAM System中文翻译)[[2\]](http://qiqitek.com/blog/?p=13)
    - [PTAM](http://www.slamcn.org/index.php/PTAM)(单目)[[3\]](http://www.robots.ox.ac.uk/~gk/PTAM/)
    - MonoSLAM(单目)[[4\]](https://github.com/hanmekim/SceneLib2)

  - 半稠密法:
    - [LSD-SLAM](http://www.slamcn.org/index.php/LSD-SLAM)(单目，双目，RGBD)[[5\]](http://vision.in.tum.de/research/vslam/lsdslam)
    - [DSO](http://www.slamcn.org/index.php?title=DSO&action=edit&redlink=1)(单目)[[6\]](https://vision.in.tum.de/research/vslam/dso)
    - [SVO](http://www.slamcn.org/index.php/SVO)(单目, 仅VO)[[7\]](https://github.com/uzh-rpg/rpg_svo)[详细安装使用说明](https://github.com/uzh-rpg/rpg_svo/wiki)

  - 稠密法:
    - [DTAM](http://www.slamcn.org/index.php?title=DTAM&action=edit&redlink=1)(RGBD): Paper: [[8\]](http://homes.cs.washington.edu/~newcombe/papers/newcombe_etal_iccv2011.pdf) Open source code:[[9\]](https://github.com/anuranbaka/OpenDTAM)
    - [Elastic Fusion](http://www.slamcn.org/index.php?title=Elastic_Fusion&action=edit&redlink=1)(RGBD): Open source code:[[10\]](https://github.com/mp3guy/ElasticFusion)
    - [Kintinous](http://www.slamcn.org/index.php?title=Kintinous&action=edit&redlink=1)(RGBD):Open source code: [[11\]](https://github.com/mp3guy/Kintinuous)
    - [DVO](http://www.slamcn.org/index.php?title=DVO&action=edit&redlink=1): Open source code: [[12\]](https://github.com/tum-vision/dvo_slam)
    - [RGBD-SLAM-V2](http://www.slamcn.org/index.php?title=RGBD-SLAM-V2&action=edit&redlink=1): Open source code: [[13\]](https://github.com/felixendres/rgbdslam_v2)
    - [RTAB-MAP](http://www.slamcn.org/index.php?title=RTAB-MAP&action=edit&redlink=1): Code: [[14\]](https://github.com/introlab/rtabmap)
    - [MLM](http://www.slamcn.org/index.php?title=MLM&action=edit&redlink=1):(单目) paper:[[15\]](https://groups.csail.mit.edu/rrg/papers/greene_icra16.pdf) Demo Video:[[16\]](https://www.youtube.com/watch?v=qk2ViPVxmq0)

  - 其他
    - [ScaViSLAM](http://www.slamcn.org/index.php?title=ScaViSLAM&action=edit&redlink=1): Open source code [[17\]](https://github.com/strasdat/ScaViSLAM)

  ### 激光传感器

  - Hector SLAM[[18\]](http://wiki.ros.org/hector_slam)
  - Gmapping [[19\]](http://wiki.ros.org/gmapping)
  - tinySLAM（基于蒙特卡洛定位算法的简单SLAM实现） [svn co https://svn.openslam.org/data/svn/tinyslam]

## SLAM资源

[**ORB-SLAM-bydianyunpcl**](https://github.com/dianyunPCL/ORB-SLAM-bydianyunpcl)

