---
title: 用到的opencv接口记录
subtitle: 用到的opencv接口记录
date: 2020-07-08 15:17:57
tags: [图像]
categories: [图像,用到的opencv接口记录]
---

[TOC]

<!--more-->

## 一、基本操作

## 1. 遍历像素

```C++
	for (int i = 0; i < rows; i++)
	{
		for (int j = 0; j < cols; j++)
		{
			int index = i*cols + j;
			image.ptr<uchar>(i)[j] = 255;
		}
	}

```

## 2. 显示图像并且保持图像比例

```C++
	namedWindow("window", WINDOW_NORMAL | WINDOW_KEEPRATIO);
	imshow("window", image);
	waitKey();
	destroyAllWindows();

```

