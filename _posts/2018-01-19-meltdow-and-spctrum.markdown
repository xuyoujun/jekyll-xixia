---
layout: post
title:  "Meltdown and Spectre"
categories: security
tags: archtecture
author: xuyoujun
description: 介绍最近CPU的漏洞.
---

前言
=============
&nbsp; &nbsp; 最近Intel 的处理器爆出了两个严重的漏洞，而我最近又从事安全相关的工作，就了解了一下原理，一时兴起决定写下来。看了很多相关的资料，每个资料从不同的角度阐述了此次CPU的漏洞，本文主要结合一些图片描述漏洞。



体系结构
=============
&nbsp; &nbsp; 如下图所示，是一个较简单的存储器层次结构，在很多资料上都有描述，cpu访问memory的数据时，如果这个数据已经在cache中时(hit)，则直接从cache中获取;如果不存在(miss)，则从memory中读取到cache中，再从cache中读取(也可直接从memory中读取，同时缓存到cache中)。然而，cpu访问cache时，hit比miss要花费的时间少很多(大约1：50)，这个时间差可以反映出一些信息，这是这次漏洞利用的条件之一。


![fugure1]({{ site.baseurl }}/picture/20180119_meltdown/figure1.png)

&nbsp; &nbsp; 分枝预测（branch prediction）和推测执行（speculation execution）是CPU动态执行技术中的主要内容，动态执行是目前CPU主要采用的先进技术之一。采用分枝预测和动态执行的主要目的是为了提高CPU的运算速度。推测执行是依托于分枝预测基础上的，在分枝预测程序是否分枝后所进行的处理也就是推测执行。

&nbsp; &nbsp; 总的来说，cpu会提前一些指令，如果顺序执行的指令，则推测执行机制会预先执行一些随后的指令；如果是分支指令，则分支预测作出跳转预测，然后推测执行机制预先执行分支预测出的分支上的指令。预先执行的指令不一定会被commit,即不一定会影响处理器的状态，但是预先执行的指令会把一些数据加载到cache中，即使这些指令最终没有commit。按理说，对于预先执行但是最终没有commit的指令，应该把其引起的被加载到cache中的数据flush出cache(或invalid)，令人遗憾的是，cpu并没有这样做，这是这次漏洞利用的条件之二。

&nbsp; &nbsp; 至于为什么在cpu发现推测执行的指令不对时，没有把这些指令引起的被加载进cache的数据flush掉，应该还是处于性能的考虑，这些数据都已经在cache中了，万一以后需要用到的时候，就可以直接使用，不用再从memeory中取数据。

简单的例子
==============
&nbsp; &nbsp; 结合上面两个条件就可以猜测出一个地址处的数据,如下面的代码所示，我们可以猜测变量a的值。现在假设我们不知道a的值，只知道a的地址放在变量p中，在函数victim_func()中，通过数组A解引用p，这时数组A的某一块数据被加载到了cache中，如果我们知道了数组A的第几块数据被加载进了cache，那么也就知道了p指向的数据为几。例如，我们知道了数组A的第5块数据被加载进了cache中，那么p指向的内容就为5。

&nbsp; &nbsp; 根据上面描述的条件一，我们可以利用cache的命中与否造成的时间差来获取一个合法地址的内容，这里核心问题在于如何将cache的状态与地址内容建立起联系。下面的程序借助于一个数组实现将cache的状态装换成地址的内容，地址p的内容可能性有256种(一个字节)，数组A有256个数据块，用p的内容去索引数组A，被缓存进cache的数据块有256个选择，缓存的数据块和p的一种可能对应，这样就建立起了一一对应的关系，通过考察A的数据块在cache中的缓存情况，就可以确定p的值。

```
char a = 5;
char *p = &a;
int temp = 0;
int A[256 * BLOCK_SIZE]; //BLOCK_SIZE可以取为cache块大小的倍数

viod victim_func(){
	int i = 0;
	temp &= A[(*p) * BLOCK_SIZE];
	
	for (i = 0; i < 255;i++){
		time1 = __rdtscp();
		temp &= A[i * BLOCK_SIZE];
		gap = __rdtscp() - time1;
	}
}
```
&nbsp; &nbsp; 利用条件一可以获取一个合法地址的内容，这是因为可以直接引用这个地址的值去引起cache的变化，从而推测出地址的内容。然后对于一些用户空间无法直接访问的地址，由于推测执行的存在，也可能存在cpu偷偷访问的情况存在，Meltdown和Spectre就是利用了cpu偷偷访问的行为。

Meltdown
==============

Spectre
==============


参考文献
========
* [分枝预测（branch prediction）和推测执行](https://baike.baidu.com/item/%E5%88%86%E6%9E%9D%E9%A2%84%E6%B5%8B%E5%92%8C%E6%8E%A8%E6%B5%8B%E6%89%A7%E8%A1%8C%E6%8A%80%E6%9C%AF/5923791?fr=aladdin)
* [meltdown](https://meltdownattack.com/meltdown.pdf)
* [spectre](https://spectreattack.com/spectre.pdf)




