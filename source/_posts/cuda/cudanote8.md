---
title: cudanote8
subtitle: 多GPU
date: 2019-05-24 10:37:12
tags: CUDA
categories: [编程,CUDA]
---

**第十一章 多GPU**

零拷贝主机内存，可移动固定的内存

[GitHub](<https://github.com/littlebearsama/CUDA-notes>)

建议下载下来用Typora软件阅读markdown文件

<!--more-->

作者github:littlebearsama [原文链接](https://github.com/littlebearsama/CUDA-notes/tree/master/1.CUDA_by-example)

**(建议下载Typora来浏览markdown文件)**

# 第十一章 多GPU

## 零拷贝主机内存

- 前面使用函数`cudaHostAlloc()`申请固定内存，并且设定参数`cudaHostAllocDefault`来获得默认的固定内存。
- 在函数`cudaHostAlloc()`使用其他参数值：**`cudaHostAllocMapped`**分配的主机内存也是固定的，它与通过`cudaHostAllocDefault`分配的固定内存有着相同的属性，特别当它不能从物理内存中交换出去或者重新定位时。
- 这种内存除了可以用于主机和GPU之间的内存复制外，还可以打破第三章主机内存规则之一：**可以在CUDA C核函数中直接访问这种类型的主机内存。由于这种内存不需要复制到GPU，因此也称为零拷贝内存**

```c++
float cuda_pinned_alloc_test(int size) {
	cudaEvent_t start, stop;
	float *a, *b, c, *partial_c;
	float *dev_a, *dev_b, *dev_partial_c;
	float elapsedTime;

	cudaEventCreate(&start);
	cudaEventCreate(&stop);

	// allocate the memory on the CPU
	cudaHostAlloc((void**)&a, size * sizeof(float),
		cudaHostAllocWriteCombined | cudaHostAllocMapped);
	cudaHostAlloc((void**)&b, size * sizeof(float),
		cudaHostAllocWriteCombined | cudaHostAllocMapped);
	cudaHostAlloc((void**)&partial_c, blocksPerGrid * sizeof(float),
		cudaHostAllocMapped);

	// find out the GPU pointers
	cudaHostGetDevicePointer(&dev_a, a, 0);
	cudaHostGetDevicePointer(&dev_b, b, 0);
	cudaHostGetDevicePointer(&dev_partial_c, partial_c, 0);

	// fill in the host memory with data
	for (int i = 0; i < size; i++) {
		a[i] = i;
		b[i] = i * 2;
	}

	cudaEventRecord(start, 0);

	dot << <blocksPerGrid, threadsPerBlock >> >(size, dev_a, dev_b, dev_partial_c);

	cudaThreadSynchronize();
	cudaEventRecord(stop, 0);
	cudaEventSynchronize(stop);
	cudaEventElapsedTime(&elapsedTime, start, stop);

	// finish up on the CPU side
	c = 0;
	for (int i = 0; i < blocksPerGrid; i++) {
		c += partial_c[i];
	}
	//无论使用什么标志都使用这个函数来释放
	cudaFreeHost(a);
	cudaFreeHost(b);
	cudaFreeHost(partial_c);

	// free events
	cudaEventDestroy(start);
	cudaEventDestroy(stop);

	printf("计算结果:  %f\n", c);

	return elapsedTime;
}

```

- **`cudaHostAllocMapped`**：这个标志告诉运行时将从GPU中访问这块内存（分配零拷贝内存）
- **`cudaHostAllocWriteCombined`**：这个表示，运行时将内存分配为“合并式写入（Write-Combined）”内存。这个标志并不会改变应用程序的功能，但却可以显著地提升GPU读取内存时的性能。然而CPU也要读取这块内存时，“合并式写入”会显得很低效。
- **`cudaHostGetDevicePointer`**：获取这块内存在GPU上的有效指针。这些指针将被传递给核函数。
- **`cudaThreadSynchronize()`**：将CPU与GPU同步，在同步完成后面就可以确信核函数已经完成，并且在零拷贝内存中包含了计算好的结果。

### 零拷贝内存的性能

- 所有固定内存都存在一定的局限性，零拷贝内存同样不例外：**每个固定内存都会占用系统的可用物理内存，这终将降低系统的性能。**(只用在使用一次的情况的原因)
- 使用零拷贝内存通常会带来性能的提升，因为内存在物理上与主机是共存的。将缓冲区声明为零拷贝内存的唯一作用就是避免不必要的数据复制。
- 当输入内存和输出内存都只是用一次时，那么独立GPU上使用零拷贝内存将带来性能提升。如果多次读取内存，那么最终将得不偿失，还不如一开始将数据复制到GPU。

## 使用多个GPU

NVIDIA一个显卡可能包含多个GPU。例如GeForce GTX 295、Tesla K10。虽然GeForce GTX 295物理上占用一个扩展槽，但在CUDA应用程序看来则是两个独立的GPU。

将多个GPU添加到独立的PCIE槽上，通过NVIDIA的SLI技术将他们桥接。

略。。。

## 可移动的固定内存---使得多个GPU共享固定内存

问题：

固定内存实际上是主机内存，只是该内存页锁定在物理内存中，以防止被换出或重定位。然而这些内存页**仅对于单个GPU线程（书上写的是单个CPU线程）来说是“固定的”**，如果某个线程分配了固定内存，那么这些内存页只是对于分配它们的线程来说是页锁定的。**如果在线程之间共享指向这块内存的指针，那么其他的线程把这块内存视为标准的、可分页的内存**。

副作用：当其他线程（不是分配固定内存的线程）试图在这块内存上执行`cudaMemcpy()`时，将按照标准的可分页内存速率来执行复制操作。这种速率大约为最高传输速度的50%。更糟糕的时，如果线程视图将一个`cudaMemcpyAsync()`调用放入CUDA流的队列中，那么将失败，因为`cudaMemcpyAsync()`需要使用固定内存。由于这块内存对于除了分配它线程外的其他线程来说视乎是可分页的，因为这个调用会失败，甚至导致任何后续操作都无法进行。

解决：

可以将固定内存分配为可移动，这意味着可以在主机线程之间移动这块内存，并且每个线程都将其视为固定内存。

- 使用`cudaHostAlloc()`来分配内存，并在调用时使用一个新标志：**`cudaHostAllocPortable`**这个标志可以与其他标志一起使用，例如`cudaHostAllocWriteCombined`和`cudaHostAllocMapped`这意味着分配主机内存时，可以将其作为可移动，零拷贝以及合并写入等的任意组合。

