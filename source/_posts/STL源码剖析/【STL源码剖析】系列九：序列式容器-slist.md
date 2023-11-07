---
title: 【STL源码剖析】系列九：序列式容器--slist
keywords: 'STL源码剖析, slist'
categories:
  - STL源码剖析
tags:
  - STL
  - 读书笔记
typora-root-url: 【STL源码剖析】系列九：序列式容器-slist
abbrlink: 1058202749
date: 2023-11-07 16:33:36
---

## 前言

本篇主要是介绍SGI STL中提供的单链表容器——slist，主要内容如下：

1.分析slist（单链表）与list（双向循环链表）的不同，引入slist的主要原因是占用空间更小，虽然操作上受限；

2.slist的迭代器和节点设计架构采用了继承的方式，这种设计与后面关联式容器中的RB-Tree（红黑树）类似；

![](4-8-14.png)

3.slist常见的对外接口。

<!-- more -->

## slist

### 概述

slist是SGI STL提供的一种单向链表（single linked list）。

slist与list的主要差别是，前者的迭代器是单向迭代器，后者的迭代器属于双向迭代器。

slist与list共同特点是，由于每个元素都被装在一个节点中，因此插入（insert）、移除（erase）、接合（splice）等操作不会使得原有的迭代器失效。

STL的习惯是插入操作将新元素插入指定位置之前，但是作为一个单向链表，slist没有方便的方法可以向前空出一个位置，因此它必须从头开始查找。所以对于slist而言，除了起点附近的区域，在其它位置采用插入或者删除操作都是不合适的。（对于list而言在任何位置插入都是可以的）基于同样的效率考虑，slist不提供push_back，只提供push_front。

### slist的节点

slist的节点与迭代器的设计比list复杂很多，使用了继承关系，所以在类型转换上有复杂的表现。下图是slist节点和迭代器的设计架构：

<img src="4-8-1.png" alt="image-20231107154029724" style="zoom:67%;" />



下面是slist的节点结构：

<img src="4-8-2.png" alt="image-20231107154313214" style="zoom:67%;" />

<img src="4-8-3.png" alt="image-20231107154336804" style="zoom:67%;" />

### slist的迭代器

slist的迭代器为单向迭代器，因此只能递增：

<img src="4-8-4.png" alt="image-20231107154724220" style="zoom:67%;" />

**base迭代器结构：**

<img src="4-8-5.png" alt="image-20231107154953870" style="zoom:67%;" />

<img src="4-8-6.png" alt="image-20231107155014812" style="zoom:67%;" />



**派生的迭代器结构：**

<img src="4-8-7.png" alt="image-20231107155446072" style="zoom:67%;" />

<img src="4-8-8.png" alt="image-20231107155515618" style="zoom:67%;" />

判断两个slist的迭代器是否相等，最终会调用base迭代器的operator==，即最终是判断两个迭代器拥有的成员指针是否指向同一个元素。

### slist的数据结构

<img src="4-8-9.png" alt="image-20231107161940839" style="zoom:67%;" />

<img src="4-8-10.png" alt="image-20231107162154849" style="zoom:67%;" />

<img src="4-8-11.png" alt="image-20231107162338716" style="zoom:67%;" />

<img src="4-8-12.png" alt="image-20231107162528808" style="zoom:67%;" />

### slist的元素操作

实际上slist也提供了insert等操作，上面源代码介绍中并没有包含，这是因为其它操作效率比较低，不是slist常用的操作。

这里另外需要注意的是，slist的end()函数是构造了一个临时迭代器对象，这个临时迭代器对象成员指针是一个空指针，与正常的迭代器不同：

<img src="4-8-13.png" alt="image-20231107163231968" style="zoom:67%;" />



