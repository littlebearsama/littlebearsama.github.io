---
title: SSE coding
subtitle: SSE指令集入门教程
author: 小熊
date: 2021-05-17 20:00:00
tags: SSE
categories: [编程,SSE,SIMD]
---



SSE指令集入门教程

<!--more-->

# 简介



官方文档：[intel intrinsics guide](https://software.intel.com/sites/landingpage/IntrinsicsGuide/#techs=SSE,SSE2,SSE3,SSSE3,SSE4_1,SSE4_2&expand=3413,6053,6044,6071,1443,5671,5668,4933,2942,4076,4085,3416,5185,5152,6077,6020,5152,5196,5193,5671,5595,5671,4928,4928,627,4141,3543,678,2815,2823,5025,3834,391,2435,3148,4905,5021,5592,5587,5595,5652,1978,5152,4414,2184,5726,3928,3649,3656,3674,2184,6098,158,2184,2946,291&cats=Logical)

SSE指令集使用方法和CUDA类似

```C++
void sse_cal(float *a,float*b)
{
    __m128 m1, m2, m3;            //声明变量
    __m128 SSEA = _mm_load_ss(a); //将a地址指向的值复制给SSEA
    __m128 SSEB = _mm_load_ss(b); //将a地址指向的值复制给SSEA
    __m128 h = _mm_set_ss(1.0f);  //声明变量并赋值


    for(int i=0;i<LOOP;i++)
    {
        //类似于cuda里面的thrust函数进行一些常规操作
        m1 = _mm_mul_ss(SSEA, SSEB);//相乘
        m2 = _mm_sqrt_ss(SSEB);     //平方和
        m3 = _mm_add_ss(m1,m2);     //相加
        SSEA = _mm_add_ss(SSEA, h); 
        SSEB = _mm_add_ss(SSEB, h); 
    }
}
```

参考：

[入门教程](https://blog.csdn.net/weixin_33910434/article/details/86392722?utm_medium=distribute.pc_relevant.none-task-blog-BlogCommendFromMachineLearnPai2-2.control&depth_1-utm_source=distribute.pc_relevant.none-task-blog-BlogCommendFromMachineLearnPai2-2.control)

[SSE指令的使用学习](https://blog.csdn.net/a200800170331/article/details/48706695?utm_medium=distribute.pc_relevant.none-task-blog-searchFromBaidu-2.control&depth_1-utm_source=distribute.pc_relevant.none-task-blog-searchFromBaidu-2.control)

[SSE双线性插值](https://www.cnblogs.com/wangguchangqing/p/5445652.html)

# OpenMP+SSE



```C++
void TransformPointCloud(std::vector<Vector3>& pointcloud, const Eigen::Matrix4& transformation_matrix)
{
	if (transformation_matrix == Eigen::Matrix4::Identity())
	{
		return;
	}
#pragma parallel for
	for (int i = 0; i < pointcloud.size(); i++)
	{
		pointcloud[i] = transformation_matrix.topLeftCorner<3, 3>()*pointcloud[i] + transformation_matrix.topRightCorner<3, 1>();
	}
}


void TransformPointCloud_SSE(std::vector<Vector3>& pointsIn, std::vector<Vector3>& pointsOut, const Eigen::Matrix4& transformation_matrix)
{
	int num = pointsIn.size();
	if (num != pointsOut.size())
		pointsOut.resize(num);
	__m128 c[4];
	for (int i = 0; i < 4; i++)
	{
		c[i] = _mm_load_ps(transformation_matrix.col(i).data());
	}
	//T11*X+T21*Y+T31*Z+T41
	//T12*X+T22*Y+T32*Z+T42
	//T13*X+T23*Y+T33*Z+T43
	//T14*X+T24*Y+T34*Z+T44
#pragma omp parallel for
	for (int i = 0; i < num; i++)
	{
		Vector4 temp;
		__m128 p0 = _mm_mul_ps(_mm_load_ps1(&(pointsIn[i].x())), c[0]);
		__m128 p1 = _mm_mul_ps(_mm_load_ps1(&(pointsIn[i].y())), c[1]);
		__m128 p2 = _mm_mul_ps(_mm_load_ps1(&(pointsIn[i].z())), c[2]);
		_mm_store_ps(temp.data(), _mm_add_ps(p0, _mm_add_ps(p1, _mm_add_ps(p2, c[3]))));
		pointsOut[i] = temp.head<3>();
	}
}
```

780000个点运行100次

普通点云变换时间为：582ms
SSE加速变换时间为：173ms