---
title: cudanote3
subtitle: 常量内存与事件
date: 2019-05-24 10:36:20
tags: CUDA
categories: [编程,CUDA]
---

**第六章 常量内存与事件**

光线追踪、常量内存与事件、常量内存带来的性能提升、线程束

[GitHub](<https://github.com/littlebearsama/CUDA-notes>)

建议下载下来用Typora软件阅读markdown文件

<!--more-->

作者github:littlebearsama [原文链接](https://github.com/littlebearsama/CUDA-notes/tree/master/1.CUDA_by-example)

**(建议下载Typora来浏览markdown文件)**

# 第六章 常量内存与事件

## 0.光线追踪

- 常量内存用于保存在核函数**执行期间不会发生变化的数据**。Nvidia硬件提供了64KB的常量内存，并且对常量内存采取了不同于标准全局内存的处理方式。在某些情况中，用常量内存来替换全局内存能有效地减少内存带宽。
- 在光线跟踪的例子中，没有利用常量内存的代码运行时间为1.8ms，利用了常量内存的代码运行时间为0.8ms
- 将球面数组存入常量内存中。

```C++
#include "cuda.h"
#include "../common/book.h"
#include "../common/image.h"

#define DIM 1024

#define rnd( x ) (x * rand() / RAND_MAX)
#define INF     2e10f

struct Sphere {
    float   r,b,g;
    float   radius;
    float   x,y,z;//球心相对于图像中心的坐标
    __device__ float hit( float ox, float oy, float *n ) {
        float dx = ox - x;
        float dy = oy - y;
        //计算来自于(ox,oy)处像素的光线（垂直于图像平面），计算光线是否与球面相交
        //然后计算相机到光线命中球面出的距离
        if (dx*dx + dy*dy < radius*radius) {
            float dz = sqrtf( radius*radius - dx*dx - dy*dy );
            *n = dz / sqrtf( radius * radius );
            return dz + z;
        }
        return -INF;
    }
};
#define SPHERES 20

__constant__ Sphere s[SPHERES];

__global__ void kernel( unsigned char *ptr ) {
    // map from threadIdx/BlockIdx to pixel position
    int x = threadIdx.x + blockIdx.x * blockDim.x;
    int y = threadIdx.y + blockIdx.y * blockDim.y;
    int offset = x + y * blockDim.x * gridDim.x;
    float   ox = (x - DIM/2);//(ox, oy)=(x, y)相对于图像中心点（DIM / 2，DIM / 2）位置或者说将图像移到中心
    float   oy = (y - DIM/2);

    float   r=0, g=0, b=0;
    float   maxz = -INF;
    for(int i=0; i<SPHERES; i++) {
        float   n;
        float   t = s[i].hit( ox, oy, &n );
        if (t > maxz) {
            float fscale = n;
            r = s[i].r * fscale;
            g = s[i].g * fscale;
            b = s[i].b * fscale;
            maxz = t;
        }
    } 

    ptr[offset*4 + 0] = (int)(r * 255);
    ptr[offset*4 + 1] = (int)(g * 255);
    ptr[offset*4 + 2] = (int)(b * 255);
    ptr[offset*4 + 3] = 255;
}

// globals needed by the update routine
struct DataBlock {
    unsigned char   *dev_bitmap;
};

int main( void ) {
    DataBlock   data;
    // capture the start time
    cudaEvent_t     start, stop;
    HANDLE_ERROR( cudaEventCreate( &start ) );
    HANDLE_ERROR( cudaEventCreate( &stop ) );
    HANDLE_ERROR( cudaEventRecord( start, 0 ) );

    IMAGE bitmap( DIM, DIM );
    unsigned char   *dev_bitmap;

    // 在GPU上分配内存以计算输出位图
    HANDLE_ERROR( cudaMalloc( (void**)&dev_bitmap,
                              bitmap.image_size() ) );

    // 分配临时内存，对其初始化，并复制到GPU上的常量内存，然后释放临时内存
    Sphere *temp_s = (Sphere*)malloc( sizeof(Sphere) * SPHERES );
    for (int i=0; i<SPHERES; i++) {
        temp_s[i].r = rnd( 1.0f );
        temp_s[i].g = rnd( 1.0f );
        temp_s[i].b = rnd( 1.0f );
        temp_s[i].x = rnd( 1000.0f ) - 500;
        temp_s[i].y = rnd( 1000.0f ) - 500;
        temp_s[i].z = rnd( 1000.0f ) - 500;
        temp_s[i].radius = rnd( 100.0f ) + 20;
    }
    HANDLE_ERROR( cudaMemcpyToSymbol( s, temp_s, 
                                sizeof(Sphere) * SPHERES) );
    free( temp_s );

    // generate a bitmap from our sphere data
    dim3    grids(DIM/16,DIM/16);
    dim3    threads(16,16);
    kernel<<<grids,threads>>>( dev_bitmap );

    // copy our bitmap back from the GPU for display
    HANDLE_ERROR( cudaMemcpy( bitmap.get_ptr(), dev_bitmap,
                              bitmap.image_size(),
                              cudaMemcpyDeviceToHost ) );

    // get stop time, and display the timing results
    HANDLE_ERROR( cudaEventRecord( stop, 0 ) );
    HANDLE_ERROR( cudaEventSynchronize( stop ) );
    float   elapsedTime;
    HANDLE_ERROR( cudaEventElapsedTime( &elapsedTime,
                                        start, stop ) );
    printf( "Time to generate:  %3.1f ms\n", elapsedTime );

    HANDLE_ERROR( cudaEventDestroy( start ) );
    HANDLE_ERROR( cudaEventDestroy( stop ) );

    HANDLE_ERROR( cudaFree( dev_bitmap ) );

    // display
    bitmap.show_image();
}
```

- 变量前面加上**`__constant__`**修饰符：`__constant__ Sphere s[SPHERES];`常量内存为静态分配空间，所以不需要调用 cudaMalloc(), cudaFree()；
- 在主机端分配临时内存，对其初始化`Sphere *temp_s = (Sphere*)malloc( sizeof(Sphere) * SPHERES );`在把变量复制到常量内存后释放内存`free( temp_s );`
- 使用函数**`cudaMemcpyToSymbol()`**将变量从主机内存复制到GPU上的常量内存。（cudaMencpyHostToDevice()的cudaMemcpy()之间的唯一差异在于，**`cudaMemcpyToSymbol()`**会复制到常量内存，而cudaMemcpy()会复制到**全局内存**）

## 1.常量内存带来的性能提升

与从全局内存中读取数据相比，从常量内存中读取相同的数据可以节约带宽，原因有二：

1. 对常量内存的单次读操作可以广播到其他的“邻近”线程，这将节约15次读取操作。
2. 常量内存的数据将缓存(cache)起来，因此对相同地址的连续读操作将不会产生额外的内存通信量。

## 2.线程束Warp

在CUDA架构中，线程束是指一个包含32个线程的集合，这个线程集合被“编织在一起”并且以“步调一致（Lockstep）”的形式执行。在程序中的每一行，线程束中的每个线程都将在不同数据上执行相同的指令。

### 线程束

当处理常量内存是，NVIDIA硬件将把**单次内存读取操作**广播到**每半个线程束（Half-Warp）**。在半线程束中包含了16和线程。如果在半线程束中的每个线程都**从常量内存的相同地址上读取数据**，那么GPU只会产生**一次读取请求**并在随后**将数据广播到每个线程**。如果从常量内存中读取大量的数据，那么这种方式生产的内存流量只是全局内存的1/16（大约6%）。

### 常量内存与缓存

但在读取常量内存是，所节约的并不只限于减少94%的带宽。由于这块内存的内容是不会发生变化的，**因此硬件将主动把这个常量数据缓存在GPU上。**在第一次从常量内存的某个地址上读取后，当其他半线程束请求同一地址是，那么将**命中缓存(cahce)**，这同样减少了额外的内存流量。在光线追踪程序中，将球面数据保存在常量内存后，硬件只需要请求这个数据一次。**在缓存数据后，其他每个线程将不会产生内存流量，原因有两个：**1. 线程将在半线程结束的广播中收到这个数据。 2. 从常量内存缓存中收到数据。

### 负面影响

当使用常量内存是，也可能对性能产生负面影响。**半线程束广播功能实际是把双刃剑**。虽然当所有16个线程地址都读取相同地址是，这个功能可以极大地提高性能，但当所有16个线程分别读取不同地址时，它实际上会降低性能。

## 3.使用事件来测试性能

代码：

```C++
    cudaEvent_t     start, stop;
    cudaEventCreate( &start );
    cudaEventCreate( &stop );
    cudaEventRecord( start, 0 );

    // 在GPU上执行一些工作

    cudaEventRecord( stop, 0 );
    cudaEventSynchronize( stop );
    float   elapsedTime;
    cudaEventElapsedTime( &elapsedTime,start, stop );
    printf( "Time to generate:  %3.1f ms\n", elapsedTime );
    cudaEventDestroy( start );
    cudaEventDestroy( stop );

```

- 运行记录事件start时，还指定了第二个参数。` cudaEventRecord( start, 0 );`在上面代码中为0，流(Stream)的编号。
- 当且仅当GPU完成了之间的工作并且记录了stop事件后，才能安全地读取stop时间值。幸运的是，**还有一种方式告诉CPU在某个事件上同步，这个时间API函数就是`cudaEventSynchronize();`,**当`cudaEventSynchronize`返回时，我们知道stop事件之前的所有GPU工作已经完成了，因此可以安全地读取在stop保存的时间戳。
- 由于CUDA事件是直接在GPU上实现的，因此它们不适用于对同时包含设备代码和主机代码的混合代码计时。也就是说，**你通过CUDA事件对核函数和设备内存复制之外的代码进行计时，将得到不可靠的结果**。

