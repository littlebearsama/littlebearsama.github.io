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

## 1. EVO
evo是一款用于视觉里程计和slam问题的轨迹评估工具。核心功能是能够绘制相机的轨迹，或评估估计轨迹与真值的误差。支持多种数据集的轨迹格式（TUM、KITTI、EuRoC MAV、ROS的bag），同时支持这些数据格式之间进行相互转换。在此仅对其基本功能做简要介绍。

### 一、安装

安装方式极其简单，采用pip安装：

> pip install evo --upgrade --no-binary evo

或者通过github下载源码后(https://github.com/MichaelGrupp/evo)，使用源码安装：

> pip install --editable . --upgrade --no-binary evo

安装时会自动安装相关依赖项。

安装完毕后，在命令行输入evo，若显示了相关信息，则表明安装成功。若提示"command not found"也不用惊慌，很多人遇到这种问题，重启电脑即可找到evo相应指令。

### 二、绘制轨迹

1. 基础指令

evo绘制轨迹的指令为：evo_traj，后跟必要参数有：数据的格式（tum/kitti/bag/euroc等），轨迹文件。轨迹文件可以有多个，例如：

> evo_traj tum traj1.txt traj2.txt

这个指令只是显示轨迹的基本信息，若要绘制轨迹，则增加可选参数 -p 或 --plot

> evo_traj tum traj1.txt –p

2. 轨迹对齐

我们时常需要将估计轨迹与真实轨迹同时绘制，可采用指令：

> evo_traj tum realTraj.txt estTraj.txt -p

然而存储时轨迹多为相对位置变化，所以绘制出的轨迹在初始位置上存在一定的位置和角度偏移，出现以下情况。
（如图所示，左图为绘制的两条曲线，通过调整可以发现两个曲线形状大体相同，但没有对齐，从而具有较大的误差）

这时我们采用对齐指令将两条轨迹进行对齐。为此我们需要通--ref参数指定参考轨迹，并增加参数-a（或--align）进行对齐（旋转与平移）

> evo_traj tum estTraj.txt --ref realTraj.txt -p -a

### 三、轨迹评估

evo可以评估两条轨迹的误差，主要有两个命令：

evo_ape：计算绝对位姿误差(absolute pose error)，用于整体评估整条轨迹的全局一致性；

evo_rpe：计算相对位姿误差(relative pose error)，用于评价轨迹局部的准确性。

这两个指令也支持evo_traj的可选参数，轨迹对齐-a与尺度缩放-s。完整指令如下：

evo_ape tum realTraj.txt estTraj.txt -a –s

此时将显示轨迹误差相关结果
若增加可选参数-p，可以绘制误差相关曲线

### 四、进阶学习

这里只介绍了evo的基本操作，除此之外evo可以进行数据格式转化、曲线颜色配置、轨迹导出等多种功能，详细请参考evo在github上的wiki：

​ ​https://github.com/MichaelGrupp/evo/wiki​​

同时，可以在命令行通过-h参数查看当前evo指令的参数及相关说明。例如：

evo_traj tum –h

注意一定要输入完整的evo指令（evo_traj, evo_ape等），与必选参数，即数据格式(tum/kitti/euroc/bag)

