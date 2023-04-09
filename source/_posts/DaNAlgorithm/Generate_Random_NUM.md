---
title: 生成随机数
subtitle: STL 生成随机数
author: 小熊
date: 2019-12-16 22:05:33
tags: [C++,STL,std]
categories: [编程,算法与数据结构]
---

**打乱数组中的内容**

<!--more-->

[参考1](http://www.cplusplus.com/reference/random/)

[参考2](https://blog.csdn.net/cywosp/article/details/17958895)

目前我用到的是std::shuffle_order_engine.功能是打乱数组中的内容

-------

random功能主要由两个部分组成：generators和distribution(不一定用分布)

1. generators

   >a.Pseudo-random（伪随机数） number engines (templates)
   >
   >- [**linear_congruential_engine**](http://www.cplusplus.com/reference/random/linear_congruential_engine/)
   >
   >- [**mersenne_twister_engine**](http://www.cplusplus.com/reference/random/mersenne_twister_engine/)
   >
   >- [**subtract_with_carry_engine**](http://www.cplusplus.com/reference/random/subtract_with_carry_engine/)
   >
   >b. Engine adaptors
   >
   >- [**independent_bits_engine**](http://www.cplusplus.com/reference/random/independent_bits_engine/)
   >
   >- [**shuffle_order_engine**](http://www.cplusplus.com/reference/random/shuffle_order_engine/)
   >
   >c. 伪随机数引擎(实例化)
   >
   >- [**default_random_engine**](http://www.cplusplus.com/reference/random/default_random_engine/)
   >
   >- [**minstd_rand**](http://www.cplusplus.com/reference/random/minstd_rand/)
   >
   >- [**minstd_rand0**](http://www.cplusplus.com/reference/random/minstd_rand0/)
   >
   >- [**mt19937**](http://www.cplusplus.com/reference/random/mt19937/)
   >
   >- [**mt19937_64**](http://www.cplusplus.com/reference/random/mt19937_64/)
   >
   >- [**ranlux24_base**](http://www.cplusplus.com/reference/random/ranlux24_base/)
   >
   >- [**ranlux48_base**](http://www.cplusplus.com/reference/random/ranlux48_base/)
   >
   >- [**ranlux24**](http://www.cplusplus.com/reference/random/ranlux24/)
   >
   >- [**ranlux48**](http://www.cplusplus.com/reference/random/ranlux48/)
   >
   >- [**knuth_b**](http://www.cplusplus.com/reference/random/knuth_b/)
   >
   >d.随机数生成器
   >
   >[**random_device**](http://www.cplusplus.com/reference/random/random_device/)

2. distributions

   >Uniform:
   >- [**uniform_int_distribution**](http://www.cplusplus.com/reference/random/uniform_int_distribution/)
   >- [**uniform_real_distribution**](http://www.cplusplus.com/reference/random/uniform_real_distribution/)
   >Related to Bernoulli (yes/no) trials:
   >- [**bernoulli_distribution**](http://www.cplusplus.com/reference/random/bernoulli_distribution/)
   >- [**binomial_distribution**](http://www.cplusplus.com/reference/random/binomial_distribution/)
   >- [**geometric_distribution**](http://www.cplusplus.com/reference/random/geometric_distribution/)
   >- [**negative_binomial_distribution**](http://www.cplusplus.com/reference/random/negative_binomial_distribution/)
   >Rate-based distributions:
   >- [**poisson_distribution**](http://www.cplusplus.com/reference/random/poisson_distribution/)
   >- [**exponential_distribution**](http://www.cplusplus.com/reference/random/exponential_distribution/)
   >- [**gamma_distribution**](http://www.cplusplus.com/reference/random/gamma_distribution/)
   >- [**weibull_distribution**](http://www.cplusplus.com/reference/random/weibull_distribution/)
   >- [**extreme_value_distribution**](http://www.cplusplus.com/reference/random/extreme_value_distribution/)
   >Related to Normal distribution:
   >- [**normal_distribution**](http://www.cplusplus.com/reference/random/normal_distribution/)
   >- [**lognormal_distribution**](http://www.cplusplus.com/reference/random/lognormal_distribution/)
   >- [**chi_squared_distribution**](http://www.cplusplus.com/reference/random/chi_squared_distribution/)
   >- [**cauchy_distribution**](http://www.cplusplus.com/reference/random/cauchy_distribution/)
   >- [**fisher_f_distribution**](http://www.cplusplus.com/reference/random/fisher_f_distribution/)
   >- [**student_t_distribution**](http://www.cplusplus.com/reference/random/student_t_distribution/)
   >Piecewise distributions:
   >- [**discrete_distribution**](http://www.cplusplus.com/reference/random/discrete_distribution/)
   >- [**piecewise_constant_distribution**](http://www.cplusplus.com/reference/random/piecewise_constant_distribution/)
   >- [**piecewise_linear_distribution**](http://www.cplusplus.com/reference/random/piecewise_linear_distribution/)
   >

实例1：

```C++
#include <iostream>
#include <algorithm>
#include <random>
#include <functional>
#include <chrono> // std::chrono::system_clock
//refernce:http://www.cplusplus.com/reference/random/

int main(_In_ int _Argc, _In_reads_(_Argc) _Pre_z_ char ** _Argv, _In_z_ char ** _Env)
{
	//该seed使得每次运行的结果都不同
    //用时间作为seed或者随机数生成器作为seed
	unsigned seed = std::chrono::system_clock::now().time_since_epoch().count();
    std::default_random_engine generator(seed);
	std::uniform_int_distribution<int> distribution(1, 6);
	// generates number in the range 1..6 
	for (size_t i = 0; i < 10; i++)
	{
		int dice_roll = distribution(generator);
		std::cout << dice_roll << " ";
	}
	std::cout << std::endl;
	//为了重复使用将两着绑定在一起
	auto dice = std::bind(distribution, generator);

	for (size_t i = 0; i < 10; i++)
	{
		int wisdom = dice() + dice() + dice();
		std::cout << wisdom << " ";
	}
	getchar();
	return 0;
}
```

输出：

>3 1 3 6 5 2 6 6 1 2
>9 8 11 13 9 12 12 9 5 9

实例2：（使用Engine adaptors：shuffle打乱vector顺序）

```C++
#include <iostream>
#include <vector>
#include <algorithm> // std::move_backward
#include <random> // std::default_random_engine
#include <chrono> // std::chrono::system_clock

int main(int argc, char* argv[])
{
	std::vector<int> v;

	for (int i = 0; i < 10; ++i) {
		v.push_back(i);
	}

	// obtain a time-based seed:
	//unsigned seed = std::chrono::system_clock::now().time_since_epoch().count();
	//std::shuffle(v.begin(), v.end(), std::default_random_engine(seed));
    //或者用随机数生成器
    std::random_device rand_dev;
	std::shuffle(v.begin(), v.end(), std::default_random_engine(rand_dev()));

	for (auto& it : v) {
		std::cout << it << " ";
	}

	std::cout << "\n";
	getchar();
	return 0;
}
```

输出：

>9 1 4 6 2 7 0 5 3 8

