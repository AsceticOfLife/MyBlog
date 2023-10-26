---
title: 【STL源码剖析】系列七：序列式容器--stack和queue
keywords: 'STL源码剖析, stack, queue'
categories:
  - STL源码剖析
tags:
  - STL
  - 读书笔记
typora-root-url: 【STL源码剖析】系列七：序列式容器-stack和queue
abbrlink: 2172343303
date: 2023-10-26 21:02:02
---

## 前言

本篇主要是介绍STL中stack和queue，主要内容有：

1.stack和queue分别是先进后出和先进先出的数据结构；

![](4-5-1.png)

![](4-5-7.png)

2.stack和queue均以deque作为底层容器，对外通过限制操作提供接口（设计模式中的适配器模式）；

3.可以将list作为二者的底层容器。

<!-- more -->

## stack

### stack概述

stack（栈）是一种先进后出的数据结构。栈只有一个出口，只能在这个出口进行添加元素（push）、删除元素（pop）、获取最顶端元素（top）。

除了上述三种操作之外，不允许有其它操作存取栈的其它元素，即不允许遍历行为。

<img src="4-5-1.png" alt="image-20231026203258256" style="zoom:67%;" />

### stack完整实现

从上述结构图可以发现，如果**以deque作为底层数据结构**，然后限制其**只能在一个端口进行操作**，就可以轻易完成stack的实现。

因此，SGI STL以deque作为默认情况下的stack的底层结构。

（实际上，stack是在deque的基础上修改其对外接口，这种设计模式称为适配器（配接器）模式。即在内部以组合方式包含deque，但是在stack中对外提供不同的接口）

<img src="4-5-2.png" alt="image-20231026203947177" style="zoom:67%;" />

<img src="4-5-3.png" alt="image-20231026204421812" style="zoom:67%;" />

<img src="4-5-4.png" alt="image-20231026204547507" style="zoom:67%;" />

### stack不提供迭代器

stack的所有元素进出都必须符合“先进后出”，只有顶端元素可以被外接访问。所以不提供遍历功能。

### 以list作为stack的底层容器

由于list也是双向开口的数据结构。上面调用的底层容器deque的函数有empty、size、back、push_back、pop_back，这些list也都具备，所以如果以list作为底层结构一样可以形成一个stack。

<img src="4-5-5.png" alt="image-20231026204933615" style="zoom:67%;" />

<img src="4-5-6.png" alt="image-20231026204954843" style="zoom:67%;" />

## queue

### queue概述

queue（队列）是一种先进先出的数据结构。

队列有两个端口，其中一个端口（尾端）只能用于新增元素（push）和查看尾端（back），另一个端口（首端）只能用于删除元素（pop）和查看首端（front）。

队列中的元素除了上面的操作之外不允许进行存取，即queue不允许遍历。

<img src="4-5-7.png" alt="image-20231026205406254" style="zoom:67%;" />

### queue完整实现

同stack一样，queue也可以将deque作为底层容器，在其基础上对外限制两端操作即可实现queue。

因此SGI STL以默认情况下以deque作为queue的底层结构。

<img src="4-5-8.png" alt="image-20231026205706299" style="zoom:67%;" />

<img src="4-5-9.png" alt="image-20231026205726593" style="zoom:67%;" />

<img src="4-5-10.png" alt="image-20231026205746334" style="zoom:67%;" />

### queue没有迭代器

与stack类似，queue所有元素进出都必须符合“先进先出”，所以必须限制对于内部元素的访问，因此不提供迭代器。

### 以list作为queue的底层容器

上述实现中可以看出queue需要底层容器提供的接口有empty、size、front、back、push、pop，这些list也能提供，所以可以把list作为底层容器：

<img src="4-5-11.png" alt="image-20231026210020843" style="zoom:67%;" />









