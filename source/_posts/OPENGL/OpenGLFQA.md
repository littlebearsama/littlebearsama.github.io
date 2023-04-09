---
title: OpenGL遇到的问题
subtitle: Learn OpenGL
author: 小熊
date: 2021-04-03 0:20:00
tags: OpenGL
categories: [可视化,learn OpenGL]
---

显示问题

1. OpenGL画点，为什么点的坐标值超出[-1,1]范围就显示不了？？？

2. 如何使用GLSL
3. 其他显示问题

<!--more-->

# 一、显示问题

## 1. OpenGL画点，为什么点的坐标值超出[-1,1]范围就显示不了？？？

https://ask.csdn.net/questions/259655

>1.OpenGL工作在一个叫做NDC（Normalized Device Coordinates）的坐标系统下，在这个坐标系统中，x、y和z的值全部都坐落在[-1, +1]范围内，超出这个范围的点会被OpenGL忽略。因为我们直接使用OpenGL的NDC坐标系统，所以顶点坐标的值在-1到+1之间。

> 2.设置投影模式，这样坐标就可以是超过-1到1了。Opengl的坐标是可以无限大的，通过设定视场大小来确定显示的范围，只有在视场范围内的坐标才能被看见，在视场之外的看不到。视场是可以用函数设定的。具体你要看看投影模式怎么回事。我很久没弄opengl了，具体也不太了解。



## 2. 如何使用GLSL

三大着色器：顶点、几何、片段

[OpenGL顶点着色器、编译着色器、片段着色器](https://blog.csdn.net/baidu_28949227/article/details/91990567)

boid.vert.glsl

```C++
#version 330

in vec4 Position;
in vec4 Velocity;
out vec4 vFragColorVs;

void main() {
    vFragColorVs = Velocity;
    gl_Position = Position;
}
```

使用in 关键字，在顶点着色器中声明所有的输入顶点属性。GLSL有一个向量数据类型，包含1到4个float分量。由于每个顶点都有一个3D坐标，我们就创建一个vec3输入变量aPos。
为了设置顶点着色器的输出，我们必须把位置数据赋值给**预定义的gl_Position**，它在幕后的vec4.
在main函数最后，我们将gl_Position设置的值会成为该顶点着色器的输出



boid.geom.glsl

```c++
#version 330

in vec4 vFragColor;
out vec4 fragColor;

void main() {
    fragColor.r = abs(vFragColor.r);
    fragColor.g = abs(vFragColor.g);
    fragColor.b = abs(vFragColor.b);
}
```





boid.frag.glsl

```C++
#version 330

uniform mat4 u_projMatrix;

layout(points) in;
layout(points) out;
layout(max_vertices = 1) out;

in vec4 vFragColorVs[];
out vec4 vFragColor;

void main() {
    vec3 Position = gl_in[0].gl_Position.xyz;
    vFragColor = vFragColorVs[0];
    gl_Position = u_projMatrix * vec4(Position, 1.0);
    EmitVertex();
    EndPrimitive();
}

```



## 3.其他显示问题

glDrawElements

原型：

```C++
void glDrawElements(GLenum mode,GLsizei count,GLenum type,const GLvoid *indices);
```

mode:接受的值和在glBegin()中接受的值一样，可以是GL_POLYGON、GL_TRIANGLES、GL_TRIANGLE_STRIP、GL_LINE_STRIP等。

count：组合几何图形的元素的个数，一般是点的个数。

type:indeices数组的数据类型，既然是索引，一般是整型的。

indices:索引数组
