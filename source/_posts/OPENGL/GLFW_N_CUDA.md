---
title: glfw相关知识以及与cuda之间的互操作
subtitle: glfwhecuda
author: 小熊
date: 2021-03-30 0:20:00
tags: glfw
categories: [可视化,glfw和cuda]
---

OpenGL与CUDA互操作

<!--more-->

# OpenGL与CUDA互操作

## 0. [基本概念：](https://blog.csdn.net/dcrmg/article/details/53556664)

### VBO：顶点缓冲对象（Vertex Buffer Objects，VBO）


顶点缓冲对象VBO是在显卡存储空间中开辟出的一块内存缓存区，用于存储顶点的各类属性信息，如顶点坐标，顶点法向量，顶点颜色数据等。在渲染时，可以直接从VBO中取出顶点的各类属性数据，由于VBO在显存而不是在内存中，不需要从CPU传输数据，处理效率更高。

所以可以理解为VBO就是显存中的一个存储区域，可以保持大量的顶点属性信息。并且可以开辟很多个VBO，每个VBO在OpenGL中有它的唯一标识ID，这个ID对应着具体的VBO的显存地址，通过这个ID可以对特定的VBO内的数据进行存取操作。

**VBO的创建以及配置**

创建VBO的第一步需要开辟（声明/获得）显存空间并分配VBO的ID：

```C++
//创建vertex buffer object对象
GLuint vboId;//vertex buffer object句柄
glGenBuffers(1, &vboId);
```
##  1.openGL的基本使用

```C++
//被init()调用
void errorCallback(int error, const char *description);
void keyCallback(GLFWwindow* window, int key, int scancode, int action, int mods);//键盘响应
void mouseButtonCallback(GLFWwindow* window, int button, int action, int mods);//鼠标按键响应
void mousePositionCallback(GLFWwindow* window, double xpos, double ypos);//鼠标位置响应
void updateCamera();
void initVAO();
void initShaders(GLuint *program);
```


```C++
bool init()
{
	//初始化前设置错误回调函数
	glfwSetErrorCallback(errorCallback);
	//调用glfwInit函数，初始化glfw，在程序终止之前，必须终止glfw，这个函数只能在主线程上被调用
	if (!glfwInit()) {
		std::cout
			<< "Error: Could not initialize GLFW!"
			<< " Perhaps OpenGL 3.3 isn't available?"
			<< std::endl;
		return false;
	}
	//在创建窗口之前调用glfwWindowHint，设置一些窗口的信息
	//这些hints，设置以后将会保持不变，只能由glfwWindowHint、glfwDefaultWindowHints或者glfwTerminate修改。
	glfwWindowHint(GLFW_CONTEXT_VERSION_MAJOR, 3);
	glfwWindowHint(GLFW_CONTEXT_VERSION_MINOR, 3);
	glfwWindowHint(GLFW_OPENGL_FORWARD_COMPAT, GL_TRUE);
	glfwWindowHint(GLFW_OPENGL_PROFILE, GLFW_OPENGL_CORE_PROFILE);
	//如果创建window失败则终结glfw，指定尺寸
	//(int width, int height, const char* title, GLFWmonitor* monitor, GLFWwidnow* share);
	window = glfwCreateWindow(width, height, windowName.c_str(), NULL, NULL);
	if (!window) {
		glfwTerminate();//glfwTerminate会销毁窗口释放资源，因此在调用该函数后，如果想使用glfw库函数，就必须重新初始化
		return false;
	}
	//
	glfwMakeContextCurrent(window);
	glfwSetKeyCallback(window, keyCallback);
	glfwSetCursorPosCallback(window, mousePositionCallback);
	glfwSetMouseButtonCallback(window, mouseButtonCallback);
	glewExperimental = GL_TRUE;
	if (glewInit() != GLEW_OK) {
		return false;
	}
	// 初始化绘图状态
	initVAO();
	//互操作
	cudaGLSetGLDevice(0);//设置CUDA环境
	//用cuda登记缓冲区
	cudaGLRegisterBufferObject(pointVBO_positions);//pointVBO_positions=0 
	cudaGLRegisterBufferObject(pointVBO_velocities);//pointVBO_velocities=0

	updateCamera();
	initShaders(program);
	glEnable(GL_DEPTH_TEST);
	return true;
}
```


```C++
void loop()
{
	double fps = 0;
	double timebase = 0;
	int frame = 0;
	//返回指定窗口是否关闭的flag变量，可以在任何线程中被调用。
	while (!glfwWindowShouldClose(window) && !BreakLoop)
	{
		//这个函数主要用来处理已经在事件队列中的事件，通常处理窗口的回调事件，包括输入，窗口的移动，窗口大小的改变等，
		//回调函数可以自己手动设置，比如之前所写的设置窗口大小的回调函数；如果没有该函数，则不会调用回调函数，同时也不会接收用户输入，例如接下来介绍的按键交互就不会被响应；
		glfwPollEvents();
		frame++;
		double time = glfwGetTime();//当前时间
		if (time - timebase > 1.0) {
			fps = frame / (time - timebase);
			timebase = time;
			frame = 0;
		}
		runcuda();//更新点云
		endrun();
	}
}
```


```C++
void runcuda()
{
	// Map OpenGL buffer object for writing from CUDA on a single GPU
	// No data is moved (Win & Linux). When mapped to CUDA, OpenGL should not
	// use this buffer
	float4 *dptr = NULL;
	float *dptrVertPositions = NULL;
	float *dptrVertVelocities = NULL;
	cudaGLMapBufferObject((void**)&dptrVertPositions, pointVBO_positions);
	cudaGLMapBufferObject((void**)&dptrVertVelocities, pointVBO_velocities);
	//
	float c_scale = 1;
	//自己定义的复制函数
	copyPointsToVBO(dptrVertPositions, dptrVertVelocities, numObjects_fixed, numObjects_rotated, blockSize, c_scale);//显存里面的数据相互赋值
	// 解除yi
	cudaGLUnmapBufferObject(pointVBO_positions);
	cudaGLUnmapBufferObject(pointVBO_velocities);
}
```


```C++
void endrun()
{
	glClear(GL_COLOR_BUFFER_BIT | GL_DEPTH_BUFFER_BIT);

	glUseProgram(program[PROG_POINT]);
	glBindVertexArray(pointVAO);
	glPointSize((GLfloat)pointSize);
	glDrawElements(GL_POINTS, N + N2 + 1, GL_UNSIGNED_INT, 0);
	glPointSize(1.0f);
	glUseProgram(0);
	glBindVertexArray(0);
	glfwSwapBuffers(window);//opengl采用双缓冲机制，该函数用于交换前后颜色缓冲区的内容
	glfwDestroyWindow(window);
	glfwTerminate();
}
```
## 2. opengl和cuda互操作基本原理

###  [**CUDA和OpenGL互操作的过程**](http://blog.chinaunix.net/uid-20620288-id-4755572.html)

#### CUDA和OpenGL互操作具体步骤如下：

>  1) 创建窗口及OpenGL运行环境。
>
> 2) 设置OpenGL视口和坐标系。要根据绘制的图形是2D还是3D等具体情况设置。(1)和(2)是所有OpenGL程序必需的，这里也没什么特殊之处，需要注意的是，后面的一些功能需要OpenGL 2.0及以上版本支持，所以在这里需要进行版本检查。
>
> 3)创建CUDA环境。可以使用cuGLCtxCreate或**cudaGLSetGLDevice**来设置CUDA环境。该设置一定要放在其他CUDA的API调用之前。
>
> 4)产生一个或多个**OpenGL缓冲区**用以和CUDA共享。使用PBO和使用VBO差不多，只是有些函数调用参数不同。以下是具体过程。
>
> (5)用CUDA登记缓冲区。登记可以使用cuGLRegisterBufferObject或
> **cudaGLRegisterBufferObject**，该命令告诉OpenGL和CUDA 驱动程序该缓冲区为二者共同使用。
>
> (6)将OpenGL缓冲区映射到CUDA内存。可以使用cuGLMapBufferObject或**cudaGLMapBufferObject**，它实际是将CUDA内存的指针指向OpenGL的缓冲区，这样如果只有一个GPU，就不需要数据传递。**当映射完成后，OpenGL不能再使用该缓冲区**。
>
> (7)使用CUDA往该映射的内存写图像数据。前面的准备工作在这里真正发挥作用了，此时可以调用CUDA的kernel，像使用全局内存一样使用映射了的缓冲区，向其中写数据。
>
> (8)取消OpenGL缓冲区映射。要等前面CUDA的活动完成以后，使用cuGLUnmapBufferObject或**cudaGLUnmapBufferObject**函数取消映射。
>
> (9)前面的步骤完成以后就可以真正开始绘图了， OpenGL的PBO和VBO的绘图方式不同，分别为以下两个过程。
>
> ①如果只是绘制平面图形，需要使用OpenGL的PBO及纹理。
>
> glEnable(GL_TEXTURE_2D)； //使纹理可用
> glGenTextures(1，&textureID)； //生成一个textureID
> glBindTexture(GL_TEXTURE_2D，textureID)；
> //使该纹理成为当前可用纹理
> glTexImage2D(GL_TEXTURE_2D，0，GL_RGBA8，Width， Height，0，GL_BGRA，GL_UNSIGNED_BYTE，NULL)；
> //分配纹理内存。最后的参数设置数据来源，这里设置为NULL，表示数据来自PBO，不是来自主机内存
> glTexParameteri(GL_TEXTURE_2D，GL_TEXTURE_MIN _FILTER，GL_LINEAR)；
> glTexParameteri(GL_TEXTURE_2D，GL_TEXTURE_MAG_ FILTER，GL_LINEAR)；//必须设置滤波模式，GL_LINEAR允许图形伸缩时线性差值。如果不需要线性差值，可以用GL_TEXTURE_RECTANGLE_ARB代替GL_TEXTURE_2D以提高性能，同时在glTexParameteri()调用里使用GL_NEAREST替换GL_LINEAR
> 然后就可以指定4个角的纹理坐标，绘制长方形了。
>
> ②绘制3D场景，需要使用VBO。
> glEnableClientState(GL_VERTEX_ARRAY)；
> //使顶点和颜色数组可用
> glEnableClientState(GL_COLOR_ARRAY)；
> glVertexPointer(3，GL_FLOAT，16，0)；
> //设置顶点和颜色指针
> glColorPointer(4，GL_UNSIGNED_BYTE，16，12)；
> glDrawArrays(GL_POINTS，0，numVerticies)；
> //根据顶点数据绘图，参数可以使用GL_LINES， GL_LINE_STRIP， GL_LINE_LOOP， GL_TRIANGLES，GL_TRIANGLE_STRIP， GL_TRIANGLE_FAN， GL_QUADS，GL_QUAD_STRIP，GL_POLYGON
> (10)前后缓存区来回切换，实现动画显示效果。调用SwapBuffers()，缓冲区切换通常会在垂直刷新间隙来处理，因此，可以在控制面板上关掉垂直同步，使得缓冲区切换立刻进行。

## 3. 详细文档

### 一、 缓冲区相关

#### [1. glGenVertexArrays：](https://blog.csdn.net/MSK1111/article/details/103114272)

名称: glGenVertexArrays —生成顶点数组对象名称

```C++
void glGenVertexArrays（GLsizei n， GLuint *arrays）;
```

n
指定要生成的顶点数组对象名称的数量。

arrays
指定一个数组，在其中存储生成的顶点数组对象名称。

#### 2. glGenBuffers：

**generate buffer object names** 该函数用来生成缓冲区对象的**名称**，第一个参数是要生成的缓冲区对象的数量，第二个是要用来存储缓冲对象名称的数组

```C++
GLuint vbo;
glGenBuffers(1,&vbo);
GLuint vbo[3];
glGenBuffers(3,vbo);
```



#### 3. glBindBuffer：

**bind a named buffer object**

第一个就是缓冲对象的类型，第二个参数就是要绑定的缓冲对象的名称，也就是我们在上一个函数里生成的名称，使用该函数将缓冲对象绑定到OpenGL上下文环境中以便使用。如果把target绑定到一个已经创建好的缓冲对象，那么这个缓冲对象将为当前target的激活对象；但是如果绑定的buffer值为0，那么OpenGL将不再对当前target使用任何缓存对象。

```C++
void glBindBuffer(GLenum target,GLuint buffer);//函数原型
```



#### 4. [glBindVertexArray：](https://blog.csdn.net/MSK1111/article/details/103072687) 

**绑定一个顶点数组对象**

```C++
void glBindVertexArray( GLuint array);//原型,array 指定要绑定的顶点数组的名称
```

> 描述
> glBindVertexArray将顶点数组对象与名称数组绑定。 array是先前从glGenVertexArrays调用返回的顶点数组对象的名称，或者为0以绑定默认的顶点数组对象绑定。
>
> 如果不存在名称为array的顶点数组对象，则在第一次绑定array时创建一个对象。 如果绑定成功，则不会更改顶点数组对象的状态，并且任何先前的顶点数组对象绑定都会中断。
>
> 错误
> 如果array不为零或先前从调用glGenVertexArrays返回的顶点数组对象的名称，则生成GL_INVALID_OPERATION。



#### 5. [glBufferData：](https://blog.csdn.net/qq_24283329/article/details/75453013)

```c++
void glBufferData( GLenum target,
GLsizeiptr size,
const GLvoid * data,
GLenum usage);//原型
//用法
glBufferData(GL_ARRAY_BUFFER, 4 * (2 * N) * sizeof(GLfloat), bodies.get(), GL_DYNAMIC_DRAW); // transfer data，创建和初始化一个buffer object的数据存储。
```

>target 指定target buffer object。必须是GL_ARRAY_BUFFER or GL_ELEMENT_ARRAY_BUFFER常量。

>size 指定buffer object的新data store的大小。

>data 指定即将被拷贝进data store并初始化的数据的指针，或者null，如果没有data要拷贝。

>usage 指定你希望data store使用的模式。必须是GL_STREAM_DRAW, GL_STATIC_DRAW, or GL_DYNAMIC_DRAW.

>1.glBufferData为绑定在target上的buffer object currently创建了一个新的data store。任何之前存在的data store被删除。新创建的data store将被指定size和usage.如果data参数为NULL,data store将被这个pointer指向的data初始化。
>
>2.usage参数提示了在GL实现中一个a buffer object’s data store将如何被访问。这使GL做出更明智的可能显著提升性能的选择。然而，如果没有，这将限制data store的准确的usage。usage可以被拆成两部分：一，被访问的频率（修改或使用），二，访问的性质。访问的频率是如下几个之一:
>STREAM
>该data store内容将被修改一次，且被使用的次数也很少；
>
>STATIC
>该data store内容将被修改一次，但会被多次使用；
>
>DYNAMIC
>该data store内容将被不断修改，且被多次使用；
>
>访问的性质如下：
>DRAW
>该data store内容将被应用程序修改，且被当做GL渲染和图像设置命令的源数据。
>
>



#### 6. [glEnableVertexAttribArray：](https://blog.csdn.net/qq_24283329/article/details/76120787) 

```C++
void glEnableVertexAttribArray(GLuint index);
void glDisableVertexAttribArray(GLuint index);
//index
//指定一个将被使能或禁止的已生成的顶点属性数组的索引。
```

> glEnableVertexAttribArray enables the generic vertex attribute array specified by index. glDisableVertexAttribArray disables the generic vertex attribute array specified by index. 默认情况下，对所有客户端都是禁止状态，包括所有已生成的顶点属性数组。当使能时，如当调用 glDrawArrays or glDrawElements这些顶点数组命令时，这些顶点属性数组将被访问和用来渲染。



#### 7. [glVertexAttribPointer：](https://blog.csdn.net/flycatdeng/article/details/82667374)

```c++
void glVertexAttribPointer
（GLuint index,
GLint size,
GLenum type,
GLboolean normalized,
GLsize stride,
const GLvoid * pointer）;//原型
```

>*index*
>
>指定要**修改**的通用顶点属性的索引。
>
>*size*
>
>指定每个通用顶点属性的组件数。 必须为1,2,3或4.初始值为4。
>
>*type*
>
>指定数组中每个组件的数据类型。 接受符号常量**GL_BYTE**，**GL_UNSIGNED_BYTE**，**GL_SHORT**，**GL_UNSIGNED_SHORT**，**GL_FIXED**或**GL_FLOAT**。 初始值为**GL_FLOAT**。
>
>*normalized*
>
>指定在访问定点数据值时是应将其标准化（**GL_TRUE**）还是直接转换为定点值（**GL_FALSE**）。
>
>*stride*
>
>指定连续通用顶点属性之间的字节偏移量。 如果*stride*为0，则通用顶点属性被理解为紧密打包在数组中的。 初始值为0。
>
>*pointer*
>
>指定指向数组中第一个通用顶点属性的第一个组件的指针。 初始值为0。



----------------

缓冲区对象只是OpenGL众多对象中的一种，其实当我们使用其它对象时，都是类似的思路

```C++
GLuint vbo;
glGenObject(1,&vbo);
GLuint vbo[3];
glGenObject(3,vbo);
glBindObject(GL_WINDOW_TARGET,vbo[1]);
```


创建对象，绑定类型，设置数据。

### 二、

#### [1. glutMainLoop]()

glutLeaveMainLoop()

#### [2. glUseProgram](https://blog.csdn.net/flycatdeng/article/details/82667360)

glUseProgram- 使用程序对象作为当前渲染状态的一部分

void **glUseProgram**（GLuint *program*）;

**参数**

*program* 指定程序对象的句柄，该程序对象的可执行文件将用作当前渲染状态的一部分。

### 三、线程相关

GLFWthread glfwCreateThread( GLFWthreadfun fun, void *arg )

A thread can wait for another thread to die with the command glfwWaitThread:

int glfwWaitThread( GLFWthread ID, int waitmode )

终止一个线程

void glfwDestroyThread( GLFWthread ID )

产生互斥锁

GLFWmutex glfwCreateMutex( void )

终止

void glfwDestroyMutex( GLFWmutex mutex )

上锁

void glfwLockMutex( GLFWmutex mutex )

解锁

void glfwUnlockMutex( GLFWmutex mutex )

 这个是等待状态变量

void glfwWaitCond( GLFWcond cond, GLFWmutex mutex, double timeout )

void glfwSignalCond( GLFWcond cond )

void glfwBroadcastCond( GLFWcond cond )