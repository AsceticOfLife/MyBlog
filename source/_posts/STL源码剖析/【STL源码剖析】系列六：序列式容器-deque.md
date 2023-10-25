---
title: 【STL源码剖析】系列六：序列式容器--deque
keywords: 'STL源码剖析, STL deque'
categories:
  - STL源码剖析
tags:
  - STL
  - 读书笔记
typora-root-url: 【STL源码剖析】系列六：序列式容器-deque
abbrlink: 3681141565
date: 2023-10-25 17:49:51
---

## 前言

这篇博客是对STL中的序列式容器deque的实现讲解。主要包括：

1.deque对比vector来说是双端可以进行操作的线性表，采用多段定长连续空间组成；

2.deque的关键任务：借助map中控器完成多段连续空间的管理、借助专属的迭代器对外提供随机访问接口；
![](4-4-7.png)

3.deque的构造与内存管理，主要涉及到map的调整和新的连续存储空间的分配会回收；

4.常见的一些deque操作实现原理，比如insert：

![](4-4-41.png)
![](4-4-42.png)

<!-- more -->

## deque

### 概述

vector是单向开口的连续线性空间，deque是双向开口的连续线性空间。

即deque可以在头尾两端做插入和删除操作。

deque与vector的差异在于：1.deque可以在常数时间内对头端进行插入和移除；2.deque没有容量的概念，它的底层存储空间是由分段的连续存储空间组合而成，也就是随时可以动态申请一段新的空间连接在头部或者尾部。

deque的迭代器也是随机存取迭代器，但是由于其分段的连续存储空间结构，它的迭代器不能是普通指针，具有复杂的处理结构。因此，尽可能使用vector而不是deque。对于deque的一些操作比如排序，为了效率更高可以将deque中的元素复制到vector中，在vector排序之后再复制回deque。

### deque的中控器

连续线性存储空间一般都不能动态增长，比如array。vector之所以可以实现动态增长，而且只能尾部增长，其实也是一种假象，实际上是：1.重新分配存储空间；2.复制原数据；3.释放原空间 三个步骤完成，只是对外隐藏细节。vector在每次动态增长空间之后都会留下一些富裕空间，否则动态增长带来的代价非常昂贵。

deque实际上是**由一段一段的定量连续空间**组成。如果需要在deque的前端或者后端增加新的空间，就需要申请一段**定量连续空间**，连接在deque的头部或者尾部。deque的关键任务就是**维护这些多段定量连续空间**，并对外**提供随机存取的接口**。

deque采用map作为中控器来维护这些多段定量连续空间。map本身是一段连续存储空间，其中每一个元素都是一个指针，指向一段连续存储空间，称为缓冲区。缓冲区默认大小为512bytes。

<img src="4-4-2.png" alt="image-20231025090521784" style="zoom:67%;" />

<img src="4-4-3.png" alt="image-20231025090617283" style="zoom:67%;" />

实际上map就是一个T**，即map是一个指针，指向一块连续空间，连续空间中存放着多个指针，每一个指针指向一个缓冲区。

<img src="4-4-4.png" alt="image-20231025090911757" style="zoom: 67%;" />

### deque的迭代器

deque的两个关键任务，一是维护多段定长连续存储空间（缓冲区），由map中控器完成；二是对外提供随机存取接口，由迭代器完成。

为了维护deque整体是一段连续空间的假象，迭代器的operator++和operator--必须对外表现出连续的假象。因此，对于一个迭代器来说，它必须知道：1.当前缓冲区（连续空间）的位置；2.处在当前缓冲区的哪个位置，方便判断是否处在边缘，如果是变换，那么operator++和operator--就必须跳跃到下一个缓冲区或上一个缓冲区；3.为了实现在缓冲区之间的跳跃，迭代器还必须借助map

<img src="4-4-5.png" alt="image-20231025091945232" style="zoom:67%;" />

这里的用于获取缓冲区大小的函数**buffer_size**调用的式全局函数：

<img src="4-4-6.png" alt="image-20231025092106298" style="zoom:67%;" />

如果n不为默认值0，说明缓冲区大小由用户定义，返回n；
如果n为默认值0，说明缓冲区大小使用默认值512字节，如果元素类型大小sz小于512，则返回512/sz(即一个缓冲区中有多少个元素)，否则传回1(说明缓冲区大小装不下一个元素)。

下图式deque的中控器、缓冲区和迭代器的关系：

<img src="4-4-7.png" alt="image-20231025092405827" style="zoom:67%;" />

下面是deque迭代器的几个关键行为。

**跳跃缓冲区：**

<img src="4-4-8.png" alt="image-20231025093744548" style="zoom:67%;" />



**重载运算符：**

<img src="4-4-9.png" alt="image-20231025093851188" style="zoom:67%;" />

<img src="4-4-10.png" alt="image-20231025094147462" style="zoom:67%;" />

<img src="4-4-11.png" alt="image-20231025094239647" style="zoom:67%;" />



**实现随机存取：**

<img src="4-4-12.png" alt="image-20231025095933261" style="zoom:67%;" />

<img src="4-4-13.png" alt="image-20231025100259278" style="zoom:67%;" />

如果offset是正数很好理解，如果是负数，需要理解以上红线标注的部分：

![image-20231025111548447](4-4-15.png)

需要计算中间红色部分占据多少个缓冲区，其中的节点个数为 -offset - 1，减去的是offset这个位置。
接着整除buffer_size，得到中间间隔的缓冲区个数。
得到缓冲区个数取负再减去1到达offset所在的缓冲区。



**其它一些运算：**基本上只要基础上面的operator+=运算即可。

<img src="4-4-14.png" alt="image-20231025100555023" style="zoom:67%;" />

### deque的数据结构

deque处理维护一个指向map中控器的指针外，还需要维护start、finish两个迭代器，分别指向第一个缓冲区的第一个元素和最后一个缓冲区的超尾元素。
同时还需要一个记录当前map大小的量，这样才能在map节点不够时重新配置更大的一块map。

<img src="4-4-16.png" alt="image-20231025113254946" style="zoom:67%;" />

<img src="4-4-17.png" alt="image-20231025113349636" style="zoom:67%;" />



有了这几个量，下面函数便可以实现：

<img src="4-4-18.png" alt="image-20231025114035896" style="zoom:67%;" />



### deque的构造与内存管理

deque内部定义了两个专属的空间配置器，一个用于每次配置一个元素大小的空间，一个用于每次配置一个指针大小的空间。

<img src="4-4-19.png" alt="image-20231025114724915" style="zoom:67%;" />

以一个构造函数为例说明如何配置空间：构造函数创建n个value元素。

<img src="4-4-20.png" alt="image-20231025114906751" style="zoom:67%;" />

内部调用**fill_initialize**来生成一个deque。

<img src="4-4-21.png" alt="image-20231025115244348" style="zoom:67%;" />

其中**create_map_and_nodes**函数创建map以及缓冲区：

<img src="4-4-22.png" alt="image-20231025115823337" style="zoom: 80%;" />

<img src="4-4-23.png" alt="image-20231025115935051" style="zoom:80%;" />

**push_back：**

在向尾端插入元素时，如果最后一个缓冲区还有备用空间，即除了一个超尾元素之外还有剩余空间，那么就直接在备用空间上构造元素：

<img src="4-4-24.png" alt="image-20231025151540937" style="zoom:67%;" />

如果最后一个缓冲区无备用空间或者只剩一个空间，就需要调用**push_back_aux**函数：

<img src="4-4-25.png" alt="image-20231025152217534" style="zoom:67%;" />

**push_front：**

在deque的前端插入元素，如果前端有备用空间就直接进行构造；如果前端没有备用空间就需要重新申请一块缓冲区：

<img src="4-4-26.png" alt="image-20231025152449082" style="zoom:67%;" />

即调用**push_front_aux**函数完成重新申请缓冲区的操作：

<img src="4-4-27.png" alt="image-20231025152612008" style="zoom:67%;" />



**map的重新整治：**

在后端插入时，如果只剩一个备用空间时就会调用push_back_aux，在重新申请一个缓冲区之前会调用**reserve_map_at_back()**调整map；

在前端插入时，如果前面没有备用空间就会调用push_front_aux，在重新申请一个缓冲区之前会调用**reserve_map_at_front()**调整map。

<img src="4-4-28.png" alt="image-20231025153235067" style="zoom:67%;" />

<img src="4-4-29.png" alt="image-20231025153256329" style="zoom:67%;" />

实际上就是当map指针数组中的指针都使用完时，就必须重新申请一个更大的map指针数组。这个工作由**reallocate_map**完成：

<img src="4-4-30.png" alt="image-20231025154123738" style="zoom:67%;" />

<img src="4-4-31.png" alt="image-20231025154358390" style="zoom:67%;" />

### deque的一些常见操作

#### pop_back和pop_front

**pop_back:**
当最后一个缓冲区有一个或者更多元素时（每个迭代器的cur指向的是当前缓冲区最后一个元素的后一个元素），摧毁最后一个元素即可；否则就需要先删除最后一个缓冲区，再执行销毁最后一个元素操作。

<img src="4-4-32.png" alt="image-20231025155113511" style="zoom:67%;" />

**pop_front：**

与上面的思路类似：

<img src="4-4-33.png" alt="image-20231025155216747" style="zoom:67%;" />

#### clear和erase

**clear：**

用于清除整个deque。deque最初状态下保留一个缓冲区，所以清除之后也要保留一个缓冲区：

<img src="4-4-34.png" alt="image-20231025155539293" style="zoom:67%;" />



**erase：清除一个元素的版本**

<img src="4-4-35.png" alt="image-20231025160232419" style="zoom: 67%;" />

**erase：清除[first, last)范围内的元素**

<img src="4-4-36.png" alt="image-20231025170525563" style="zoom:67%;" />

<img src="4-4-37.png" alt="image-20231025170609651" style="zoom:67%;" />



#### insert

insert再deque中存在许多版本，下面主要介绍这个版本：再某个点之前插入一个元素：

<img src="4-4-38.png" alt="image-20231025170719998" style="zoom:67%;" />

<img src="4-4-39.png" alt="image-20231025170738897" style="zoom:67%;" />

如果是头部和尾部就交给push_front和push_back去做，否则由**inset_aux**函数完成：

<img src="4-4-40.png" alt="image-20231025171114665" style="zoom: 80%;" />

如果前面元素比较少，插入过程如图：

<img src="4-4-41.png" alt="image-20231025173624704" style="zoom:80%;" />

如果后面元素比较少，插入过程如图：

<img src="4-4-42.png" alt="image-20231025174730115" style="zoom:80%;" />

