---
title: glfw显示点云
subtitle: Learn OpenGL
author: 小熊
date: 2021-04-03 0:20:00
tags: OpenGL
categories: [可视化,learn OpenGL]1
---

一个简单的glfw显示点云的例程

<!--more-->

# 一个简单的glfw显示点云的例程

```C++
#include <functional>
#include <memory>
#include <iostream>
#include <string>
#include <vector>
#include <GL/glew.h>
#include <GLFW/glfw3.h>
#include <glm/glm.hpp>
#include <glm/gtc/matrix_transform.hpp>
#include <glm/gtx/transform.hpp>
#include "src/glslUtility.hpp"


#ifndef PI
#define PI 3.141592654
#endif

using namespace std;
//要实现设置显示点的数量
static int N=100000;
static int N2;
GLFWwindow *window = nullptr;

// For camera controls
static bool leftMousePressed = false;
static bool rightMousePressed = false;
static bool middleMousePressed = false;
GLuint positionLocation = 0;   // Match results from glslUtility::createProgram.
GLuint velocitiesLocation = 1; // Also see attribtueLocations below.

GLuint pointVAO = 0;//所有需要画的粒子都绑定在上面
GLuint pointVBO_positions = 0;
GLuint pointVBO_velocities = 0;
GLuint pointIBO = 0;//indexs
GLuint displayImage;
GLuint program[2];
const unsigned int PROG_POINT = 0;//

const float fovy = (float)(PI / 4);
const float zNear = 0.10f;
const float zFar = 10.0f;
//窗口相关
std::string windowName = std::string("GLvisualizationWin");
int width = 1280;
int height = 720;
int pointSize = 2;
bool BreakLoop = false;
double lastX;
double lastY;
float theta = 1.22f;
float phi = -0.70f;
float zoom = 4.0f;
glm::vec3 lookAt = glm::vec3(0.0f, 0.0f, 0.0f);
glm::vec3 cameraPosition;
glm::mat4 projection;

//外部输入数据
int numObjects_fixed = 1000;
int numObjects_rotated = 1000;
int blockSize = 128;

//被init()调用
void errorCallback(int error, const char *description);
void keyCallback(GLFWwindow* window, int key, int scancode, int action, int mods);//键盘响应
void mouseButtonCallback(GLFWwindow* window, int button, int action, int mods);//鼠标按键响应
void mousePositionCallback(GLFWwindow* window, double xpos, double ypos);//鼠标位置响应
void updateCamera();
void initShaders(GLuint *program);
void endrun();

void initVAO()
{
	std::unique_ptr<GLfloat[]> bodies{ new GLfloat[4 * N] };//数据
	std::unique_ptr<GLfloat[]> rgbs{ new GLfloat[4 * N] };//数据
	std::unique_ptr<GLuint[]> bindices{ new GLuint[N] };//索引
	glm::vec4 ul(-1.0, -1.0, 1.0, 1.0);
	glm::vec4 lr(1.0, 1.0, 0.0, 0.0);
	for (int i = 0; i < N; i++) {
		float x = float(rand()) / 9.99f;
		x = x - (int)x;
		float y = float(rand()) / 9.99f;
		y = y - (int)y;
		float z = float(rand()) / 9.99f;
		z = z - (int)z;

		bodies[4 * i + 0] = x;
		bodies[4 * i + 1] = y;
		bodies[4 * i + 2] = z;
		bodies[4 * i + 3] = 1.0f;
		rgbs[4 * i + 0] = 1;
		rgbs[4 * i + 1] = 0;
		rgbs[4 * i + 2] = 0;
		rgbs[4 * i + 3] = 1.0f;
		bindices[i] = i;
	}
	//创建VAO 把所有需要画粒子的东西都粘在上面
	glGenVertexArrays(1, &pointVAO); // Attach everything needed to draw a particle to this
	glGenBuffers(1, &pointVBO_positions);//该函数用来生成缓冲区对象的名称，第一个参数是要生成的缓冲区对象的数量，第二个是要用来存储缓冲对象名称的数组
	glGenBuffers(1, &pointVBO_velocities);
	glGenBuffers(1, &pointIBO);//生成索引缓存区的名字
	glBindVertexArray(pointVAO);//绑定一个顶点数组对象

	// Bind the positions array to the pointVAO by way of the pointVBO_positions
	glBindBuffer(GL_ARRAY_BUFFER, pointVBO_positions); // bind the buffer pointVBO_positions变成了一个顶点缓冲类型
	glBufferData(GL_ARRAY_BUFFER, 4 * N * sizeof(GLfloat), bodies.get(), GL_DYNAMIC_DRAW); // transfer data，创建和初始化一个buffer object的数据存储。
	glEnableVertexAttribArray(positionLocation);
	glVertexAttribPointer((GLuint)positionLocation, 4, GL_FLOAT, GL_FALSE, 0, 0);

	// Bind the velocities array to the pointVAO by way of the pointVBO_velocities
	glBindBuffer(GL_ARRAY_BUFFER, pointVBO_velocities);
	glBufferData(GL_ARRAY_BUFFER, 4 * N * sizeof(GLfloat), rgbs.get(), GL_DYNAMIC_DRAW);
	glEnableVertexAttribArray(velocitiesLocation);
	glVertexAttribPointer((GLuint)velocitiesLocation, 4, GL_FLOAT, GL_FALSE, 0, 0);

	//给索引数据也绑定buffer
	glBindBuffer(GL_ELEMENT_ARRAY_BUFFER, pointIBO);
	glBufferData(GL_ELEMENT_ARRAY_BUFFER, N * sizeof(GLuint), bindices.get(), GL_STATIC_DRAW);
	glBindVertexArray(0);
}

void testGLFW()
{
	//初始化前设置错误回调函数
	glfwSetErrorCallback(errorCallback);
	//调用glfwInit函数，初始化glfw，在程序终止之前，必须终止glfw，这个函数只能在主线程上被调用
	if (!glfwInit()) {
		std::cout
			<< "Error: Could not initialize GLFW!"
			<< " Perhaps OpenGL 3.3 isn't available?"
			<< std::endl;
		return ;
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
		std::cout << "Failed to create GLFW window" << std::endl;
		glfwTerminate();//glfwTerminate会销毁窗口释放资源，因此在调用该函数后，如果想使用glfw库函数，就必须重新初始化
		return ;
	}
	//
	glfwMakeContextCurrent(window);
	glfwSetKeyCallback(window, keyCallback);
	glfwSetCursorPosCallback(window, mousePositionCallback);
	glfwSetMouseButtonCallback(window, mouseButtonCallback);
	glewExperimental = GL_TRUE;
	if (glewInit() != GLEW_OK) {
		std::cout << "Failed to initialize GLEW" << std::endl;
		return ;
	}
	// 初始化绘图状态
	initVAO();
	//互操作
	//cudaGLSetGLDevice(0);//设置CUDA环境
	////用cuda登记缓冲区，该命令告诉OpenGL和CUDA 驱动程序该缓冲区为二者共同使用。
	//cudaGLRegisterBufferObject(pointVBO_positions);//pointVBO_positions=0 
	//cudaGLRegisterBufferObject(pointVBO_velocities);//pointVBO_velocities=0

	updateCamera();
	initShaders(program);
	glEnable(GL_DEPTH_TEST);

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
		//runcuda();//更新点云
		glClear(GL_COLOR_BUFFER_BIT | GL_DEPTH_BUFFER_BIT);

		glUseProgram(program[PROG_POINT]);
		glBindVertexArray(pointVAO);
		glPointSize((GLfloat)pointSize);
		glDrawElements(GL_POINTS, N + N2 + 1, GL_UNSIGNED_INT, 0);
		glPointSize(1.0f);
		glUseProgram(0);
		glBindVertexArray(0);
		glfwSwapBuffers(window);//opengl采用双缓冲机制，该函数用于交换前后颜色缓冲区的内容
	}
	endrun();
	return ;

}

void endrun()
{
	glfwDestroyWindow(window);
	glfwTerminate();

}
void initShaders(GLuint *program)
{
	GLint location;
	const char *vertexShaderPath = "shaders/boid.vert.glsl";
	const char *geometryShaderPath = "shaders/boid.geom.glsl";
	const char *fragmentShaderPath = "shaders/boid.frag.glsl";
	const char *attributeLocations[] = { "Position", "Velocity" };
	program[PROG_POINT] = glslUtility::createProgram(vertexShaderPath, geometryShaderPath, fragmentShaderPath, attributeLocations, GLuint(2));

	glUseProgram(program[PROG_POINT]);
	if ((location = glGetUniformLocation(program[PROG_POINT], "u_projMatrix")) != -1) {
		glUniformMatrix4fv(location, 1, GL_FALSE, &projection[0][0]);
	}
	if ((location = glGetUniformLocation(program[PROG_POINT], "u_cameraPos")) != -1) {
		glUniform3fv(location, 1, &cameraPosition[0]);
	}
}
void errorCallback(int error, const char *description)
{
	fprintf(stderr, "error %d: %s\n", error, description);
}
void keyCallback(GLFWwindow* window, int key, int scancode, int action, int mods)
{
	if (key == GLFW_KEY_ESCAPE && action == GLFW_PRESS) {
		glfwSetWindowShouldClose(window, GL_TRUE);
	}
}


void mouseButtonCallback(GLFWwindow* window, int button, int action, int mods)
{
	leftMousePressed = (button == GLFW_MOUSE_BUTTON_LEFT && action == GLFW_PRESS);
	rightMousePressed = (button == GLFW_MOUSE_BUTTON_RIGHT && action == GLFW_PRESS);
	middleMousePressed = (button == GLFW_MOUSE_BUTTON_MIDDLE && action == GLFW_PRESS);
}

void mousePositionCallback(GLFWwindow* window, double xpos, double ypos)
{
	if (leftMousePressed) {
		// compute new camera parameters
		phi += (xpos - lastX) / width;
		theta -= (ypos - lastY) / height;
		theta = std::fmax(0.01f, std::fmin(theta, 3.14f));
		updateCamera();
	}
	else if (rightMousePressed) {
		zoom += (ypos - lastY) / height;
		zoom = std::fmax(0.1f, std::fmin(zoom, 5.0f));
		updateCamera();
	}
	else if (middleMousePressed){
		glm::vec3 forward = -glm::normalize(cameraPosition);
		forward.y = 0.0f;
		forward = glm::normalize(forward);
		glm::vec3 right = glm::cross(forward, glm::vec3(0, 1, 0));
		right.y = 0.0f;
		right = glm::normalize(right);

		lookAt -= (float)(xpos - lastX) * right * 0.01f;
		lookAt += (float)(ypos - lastY) * forward * 0.01f;
		updateCamera();
	}

	lastX = xpos;
	lastY = ypos;
}

void updateCamera()
{
	cameraPosition.x = zoom * sin(phi) * sin(theta);
	cameraPosition.z = zoom * cos(theta);
	cameraPosition.y = zoom * cos(phi) * sin(theta);
	cameraPosition += lookAt;
	cout << lookAt.x << ", " << lookAt.y << ", " << lookAt.z << "," << endl;
	projection = glm::perspective(fovy, float(width) / float(height), zNear, zFar);
	glm::mat4 view = glm::lookAt(cameraPosition, lookAt, glm::vec3(0, 0, 1));
	projection = projection * view;
	GLint location;
	glUseProgram(program[PROG_POINT]);
	if ((location = glGetUniformLocation(program[PROG_POINT], "u_projMatrix")) != -1) {
		glUniformMatrix4fv(location, 1, GL_FALSE, &projection[0][0]);
	}
}
```

