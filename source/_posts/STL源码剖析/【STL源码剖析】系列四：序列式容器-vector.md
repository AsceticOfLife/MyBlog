---
title: 【STL源码剖析】系列四：序列式容器--vector
date: 2023-10-19 09:42:42
categories:
- STL源码剖析
tags:
- STL
- 读书笔记
typora-root-url: 【STL源码剖析】系列四：序列式容器-vector
---

## 前言

vector是STL中最常用的一种关联式容器，其对应的数据结构是采用顺序存储结构的线性表。

本篇主要内容：
1.vector迭代器介绍；
2.vector的底层存储结构和内存管理方式；
3.vector的一些常用操作实现解析，比如insert：
![](4-2-20.png)



<!-- more -->

## vector

### 概述

vector的数据安排和操作方式与array非常类似，均是采用一块连续的存储空间排列数据元素。

二者的差别在于空间的灵活性：
array是静态空间，一旦申请了就不能改变。如果想要更换空间，就需要用户申请一块新的空间，然后将旧空间中的元素全部移动到新申请的空间，最后把原来的空间释放。
vector是动态空间，随着元素加入，内部机制会自行扩充空间以容纳新元素。

vector实现的关键技术就是对于空间大小的控制以及重新申请空间时的雄数据移动效率。

### vector源代码

书上列举了vector的实现源代码摘要，详情参看书中内容。

### vector的迭代器

vector是一块连续线性空间，因此支持随机访问，又因为普通指针能够满足vector的迭代器要求，因此vector内部将原生指针声明为迭代器类型：

<img src="4-2-1.png" alt="image-20231018090403169" style="zoom:67%;" />

```c++
// 定义vector的迭代器
vector<int>::iterator ivite;
vector<Shape>::iterator svite;

// 实际上ivite的类型就是 int *，svite的类型就是 Shape *
```

### vector的数据结构

vector的数据结构就是一段线性连续空间。以两个迭代器start和finish指向已经被使用的范围，以迭代器end_of_storage指向整块空间的尾端。

<img src="4-2-2.png" alt="image-20231018090744171" style="zoom:67%;" />

这也表明vector的实际大小要比已经使用的大小要大，这是为了以后添加元素时尽量不用重新申请空间。因此实际大小称为capacity，已经使用的大小称为size。

<img src="4-2-4.png" alt="image-20231018091657884" style="zoom:50%;" />

利用上述三个迭代器，很容易提供首尾标志、size、capacity、是否为空等接口：

<img src="4-2-3.png" alt="image-20231018091313714" style="zoom:67%;" />

### vector的构造与内存管理

vector默认情况下以alloc作为空间配置器，基于此定义了一个data_allocator，以元素大小为申请空间单位。

<img src="4-2-5.png" alt="image-20231018092309423" style="zoom:67%;" />

对于默认配置器alloc的allocate函数来说，是以字节为单位申请n个字节的内存；而外层的simple_alloc提供了一层包装，以元素类型为单位申请 sizeof(value_type) * n个字节的内存。

以其中一个构造函数为例：指定空间大小（以元素为单位）和初始值

<img src="4-2-6.png" alt="image-20231018093533086" style="zoom:67%;" />

<img src="4-2-7.png" alt="image-20231018093619878" style="zoom:67%;" />

经过重重传递调用，最终：1.调用分配器的allocate(n)申请n个元素空间；2.调用uninitialized_fill_n(first, n, x)函数，即使用x对象为[first, first + n)迭代器范围内的空间进行初始化。这里会根据迭代器指向的元素类型是否为POD来决定使用高阶算法fill_n还是不断为每一个位置调用construc来完成构造任务。



<img src="4-2-8.png" alt="image-20231018094707759" style="zoom:67%;" />

<img src="4-2-9.png" alt="image-20231018094933439" style="zoom:67%;" />

<img src="4-2-10.png" alt="image-20231018095119458" style="zoom:67%;" />

这里需要注意的是，任何对于vector的操作，一旦引发了空间重新配置，那么指向原vector的所有迭代器均失效。（原空间已经被释放）

### vector的一些元素操作

#### pop_back

<img src="4-2-11.png" alt="image-20231018095716460" style="zoom:67%;" />

#### erase和clear

<img src="4-2-12.png" alt="image-20231018095940185" style="zoom:67%;" />

<img src="4-2-13.png" alt="image-20231018100216774" style="zoom:67%;" />

#### insert

<img src="4-2-14.png" alt="image-20231018100533612" style="zoom:67%;" />

<img src="4-2-15.png" alt="image-20231018104139435" style="zoom:67%;" />

<img src="4-2-16.png" alt="image-20231018154258933" style="zoom:67%;" />

<img src="4-2-17.png" alt="image-20231018154343842" style="zoom:67%;" />

插入完成之后，新的结点将位于哨兵迭代器（也就是指向position位置的迭代器）的前面。

下面对于插入操作进行演示，主要分成剩余空间足够和不够两种情况。

**1.当剩余空间足够时：**
剩余空间足够又分为两种情况，插入结点后面的元素个数（m）大于插入元素个数（n）和反之。
当m大于n时：
<img src="4-2-18.png" alt="image-20231018155731188" style="zoom:67%;" />

当m小于等于n时：
<img src="4-2-19.png" alt="image-20231018155807898" style="zoom:67%;" />
这样做的好处是少做了 n - m 次移动。

**2.当剩余空间不够时：**
<img src="4-2-20.png" alt="image-20231018155903107" style="zoom: 67%;" />
