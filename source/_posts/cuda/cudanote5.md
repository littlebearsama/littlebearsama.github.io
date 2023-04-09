---
title: cudanote5
subtitle: 互操作性
date: 2019-05-24 10:36:56
tags: CUDA
categories: [编程,CUDA]
---

**第八章 互操作性**

OpenGL与互操作性

[GitHub](<https://github.com/littlebearsama/CUDA-notes>)

建议下载下来用Typora软件阅读markdown文件

<!--more-->

作者github:littlebearsama [原文链接](https://github.com/littlebearsama/CUDA-notes/tree/master/1.CUDA_by-example)

**(建议下载Typora来浏览markdown文件)**

# 第八章 互操作性

GPU的成功要归功于它能实时计算复杂的渲染任务，同时系统的其他部分还可以执行其他工作。

## 互操作性

### 概念：

- 通用计算：譬如前面的计算，在GPU上面进行的计算
- 渲染任务
- 互操作是指在通用计算与渲染模式之间互操作

### 提出问题：

问1：那么能否在**同一个应用程序中GPU既执行渲染计算，又执行通用计算？**

问2：如果要**渲染的图像**依赖于**通用计算**的结果，那么该如何处理？

问3：如果想要在**已经渲染的帧上**执行某种图像处理或者统计，又该如何实现？

## 与OpenGL的互操作性

CUDA程序生成图像数据传递给OpenGL驱动程序并进行渲染

### 1.代码：

```C++
/********************************************************************
*  SharedBuffer.cu
*  interact between CUDA and OpenGL
*********************************************************************/

#include <stdio.h>
#include <stdlib.h>
//下面两个头文件如果放反了会出错
#include "GL\glut.h"
#include "GL\glext.h"

#include <cuda_runtime.h>
//#include <cutil_inline.h>
#include <pcl\cuda\cutil_inline.h>
#include <cuda.h>
#include <cuda_gl_interop.h>

#define GET_PROC_ADDRESS(str) wglGetProcAddress(str)
#define DIM 512

PFNGLBINDBUFFERARBPROC    glBindBuffer = NULL;
PFNGLDELETEBUFFERSARBPROC glDeleteBuffers = NULL;
PFNGLGENBUFFERSARBPROC    glGenBuffers = NULL;
PFNGLBUFFERDATAARBPROC    glBufferData = NULL;

// step one:
// Step1: 申明两个全局变量，保存指向同一个缓冲区的不同句柄，指向要在OpenGL和CUDA之间共享的数据；
GLuint bufferObj;
cudaGraphicsResource *resource;


__global__ void cudaGLKernel(uchar4 *ptr)
{
	int x = threadIdx.x + blockIdx.x * blockDim.x;
	int y = threadIdx.y + blockIdx.y * blockDim.y;
	int offset = x + y * blockDim.x * gridDim.x;

	//将图像中心设为原点后的像素索引
	float fx = x / (float)DIM - 0.5f;
	float fy = y / (float)DIM - 0.5f;

	unsigned char green = 128 + 127 * sin(abs(fx * 100) - abs(fy * 100));

	ptr[offset].x = 0;
	ptr[offset].y = green;
	ptr[offset].z = 0;
	ptr[offset].w = 255;

}

void drawFunc(void)
{
	glDrawPixels(DIM, DIM, GL_RGBA, GL_UNSIGNED_BYTE, 0);
	glutSwapBuffers();
}

static void keyFunc(unsigned char key, int x, int y)
{
	switch (key){
	case 27:
		cutilSafeCall(cudaGraphicsUnregisterResource(resource));
		glBindBuffer(GL_PIXEL_UNPACK_BUFFER_ARB, 0);
		glDeleteBuffers(1, &bufferObj);
		exit(0);
	}
}

int main(int argc, char* argv[])
{
	// step 2:
	// 初始化CUDA
	// Step2: 选择运行应用程序的CUDA设备(cudaChooseDevice),告诉cuda运行时使用哪个设备来执行CUDA和OpenGL (cudaGLSetGLDevice）；
	cudaDeviceProp prop;
	int dev;

	memset(&prop, 0, sizeof(cudaDeviceProp));
	prop.major = 1;
	prop.minor = 0;
	cutilSafeCall(cudaChooseDevice(&dev, &prop));
	//为CUDA运行时使用openGL驱动做准备
	cutilSafeCall(cudaGLSetGLDevice(dev));

	//初始化OpenGL
	//在执行其他的GL调用之前，需要首先执行这些GLUT调用。
	glutInit(&argc, argv);
	glutInitDisplayMode(GLUT_DOUBLE | GLUT_RGBA);
	glutInitWindowSize(DIM, DIM);
	glutCreateWindow("CUDA interact with OpenGL");


	glBindBuffer = (PFNGLBINDBUFFERARBPROC)GET_PROC_ADDRESS("glBindBuffer");
	glDeleteBuffers = (PFNGLDELETEBUFFERSARBPROC)GET_PROC_ADDRESS("glDeleteBuffers");
	glGenBuffers = (PFNGLGENBUFFERSARBPROC)GET_PROC_ADDRESS("glGenBuffers");
	glBufferData = (PFNGLBUFFERDATAARBPROC)GET_PROC_ADDRESS("glBufferData");

	// Step3：在OpenGL中创建像素缓冲区对象；
	glGenBuffers(1, &bufferObj);
	glBindBuffer(GL_PIXEL_UNPACK_BUFFER_ARB, bufferObj);
	glBufferData(GL_PIXEL_UNPACK_BUFFER_ARB, DIM*DIM * 4, NULL, GL_DYNAMIC_DRAW_ARB);//glBufferData()的调用需要OpenGL驱动程序分配一个足够大的缓冲区来保存DIM*DIM 个32位的值

	// step 4:
	// Step4: 通知CUDA运行时将像素缓冲区对象bufferObj注册为图形资源，实现缓冲区共享。
	cutilSafeCall(cudaGraphicsGLRegisterBuffer(&resource, bufferObj, cudaGraphicsMapFlagsNone));//cudaGraphicsMapFlagsNone表示不需要为缓冲区指定特定的行为
	                                                                                            //cudaGraphicsMapFlagsReadOnly将缓冲区指定为只读的
	                                                                                            //通过标志cudaGraphicsMapFlagsWriteDiscard来制定缓冲区之前的内容应该抛弃，从而使缓冲区变成只写的

	uchar4* devPtr;
	size_t size;
	cutilSafeCall(cudaGraphicsMapResources(1, &resource, NULL));
	cutilSafeCall(cudaGraphicsResourceGetMappedPointer((void**)&devPtr, &size, resource));

	dim3 grids(DIM / 16, DIM / 16);
	dim3 threads(16, 16);
	cudaGLKernel << <grids, threads >> >(devPtr);

	cutilSafeCall(cudaGraphicsUnmapResources(1, &resource, NULL));
	glutKeyboardFunc(keyFunc);
	glutDisplayFunc(drawFunc);
	glutMainLoop();
	return 0;
}



```

### 2.代码解析：

- Step1: 申明两个全局变量，保存指向同一个缓冲区的不同句柄，指向要在OpenGL和CUDA之间共享的数据；

```C++
	GLuint bufferObj;
	cudaGraphicsResource *resource;

```

- Step2: 选择运行应用程序的CUDA设备(cudaChooseDevice),告诉cuda运行时使用哪个设备来执行CUDA和OpenGL (cudaGLSetGLDevice）cutilSafeCall(cudaChooseDevice(&dev, &prop));
- Step3：**共享数据缓冲区**是在CUDA C核函数和OpenG渲染操作之间实现互操作的关键部分。要在OpenGL和CUDA之间传递数据，我们首先要创建一个缓冲区在这两组API之间使用，在OpenGL中创建像素缓冲区对象；，并将句柄保存在全局变量`GLuint bufferObj`中：

```C++
	glGenBuffers(1, &bufferObj);
	glBindBuffer(GL_PIXEL_UNPACK_BUFFER_ARB, bufferObj);
	glBufferData(GL_PIXEL_UNPACK_BUFFER_ARB, DIM*DIM * 4, NULL, GL_DYNAMIC_DRAW_ARB);

```

- Step4: 通知CUDA运行时将像素缓冲区对象bufferObj注册为图形资源，实现缓冲区共享。

```C++
  cutilSafeCall(cudaGraphicsGLRegisterBuffer(&resource, 
                                             bufferObj,
                                             cudaGraphicsMapFlagsNone));

```

- 互操作性基本上就是调用接口，可以通过GPU Computing SDK的代码示例来学习

## 与DirectX的互操作性（略）