---
title: 【STL源码剖析】系列五：序列式容器--list
date: 2023-10-23 21:32:31
categories:
- STL源码剖析
tags:
- STL
- 读书笔记
typora-root-url: 【STL源码剖析】系列五：序列式容器-list
---

## 前言

本篇内容是《STL源码剖析》中对于list容器的读书笔记和个人理解。主要内容是：

1.list容器在概念上对应于线性表的链式存储结构，实现上是双向循环链表；
![](4-3-6.png)

2.list的构造与内存管理，重点是如何插入一个元素；
![](4-3-22.png)

3.list中的一些关键操作实现，包括splice、merge、reverse、sort等以及底层实现transfer。

<!-- more -->

## list

### 概述

list与vector相比，主要区别是底层存储空间不同。vector使用连续存储结构表示线性表，list使用链式存储结构表示线性表。
使用链式存储结构的好处就是每次插入和删除一个元素，就配置或者释放一个元素空间。另外就是对于任何位置的元素插入和删除是常数时间。

### list的节点

<img src="4-3-1.png" alt="image-20231022092900641" style="zoom:67%;" />

### list的迭代器

因为list的存储结构并不是连续的，所以不能像vector一样使用普通指针作为迭代器完成随机存取。

list的迭代器需要具备的功能有：1.递增，指向下一个节点；2.递减，指向前一个结点；3.取值，解引用时取的是节点的数据；4.成员访问，成员访问时是节点的成员。

<img src="4-3-2.png" alt="image-20231022093546612" style="zoom:67%;" />

由于STL中的list是双向链表。所以迭代器的类型是双向迭代器。

list迭代器与vector迭代器的不同：插入、删除和连接操作不会使原有的list迭代器失效（删除只会让被删除的迭代器失效），这是因为对于list的操作不会影响现有的空间，对于vector的任何操作有可能会导致空间重新分配。

list迭代器源代码：

<img src="4-3-3.png" alt="image-20231022094054153" style="zoom:67%;" />

<img src="4-3-4.png" alt="image-20231022094217730" style="zoom:67%;" />

### list的数据结构

SGI的list是一个双向循环链表，因此只需要一个指针就可以表示整个链表。

<img src="4-3-5.png" alt="image-20231022094420366" style="zoom:67%;" />

为了满足前闭后开区间的要求，这里需要让这个指针指向最后一个元素后面的一个空结点：

<img src="4-3-6.png" alt="image-20231022094621428" style="zoom:67%;" />

由此便可以实现下面几个函数：

<img src="4-3-7.png" alt="image-20231022094711960" style="zoom:67%;" />

<img src="4-3-8.png" alt="image-20231022094826598" style="zoom:67%;" />

### list的构造与内存管理

list默认使用alloc（默认是第二级配置器）作为空间配置器，并据此借助接口simple_alloc定义一个专属配置器list_node_allocator，方便以元素为单位申请空间。

<img src="4-3-9.png" alt="image-20231022095322494" style="zoom:67%;" />

<img src="4-3-10.png" alt="image-20231022095342246" style="zoom:67%;" />

所以就可以使用list_node_allocator(n)表示申请n个节点的空间。下面四个函数用于配置、释放、构造、销毁一个节点：

<img src="4-3-11.png" alt="image-20231022095744952" style="zoom:67%;" />



**以默认构造函数为例说明list的创建过程：**

<img src="4-3-12.png" alt="image-20231022100017648" style="zoom:67%;" />

<img src="4-3-13.png" alt="image-20231022100039139" style="zoom:67%;" />



当调用push_back()插入新元素时，该函数调用其中一个重载版本的insert函数：

<img src="4-3-14.png" alt="image-20231022100216879" style="zoom:67%;" />

<img src="4-3-15.png" alt="image-20231022100253014" style="zoom:67%;" />

注意插入节点之后返回的是哨兵迭代器（插入点）的前方。

关于插入节点的操作顺序：

插入节点主要是对指针进行操作，这里的关键在于**修改指针之后要保证能够找到需要被操作的节点**，不要出现修改指针之后丢失节点问题。所以插入时进行每一步指针操作时都要考虑清楚这步操作完成之后能否找到需要被操作的节点。

以单链表为例，假如需要在节点P后面（单链表只有next指针所以只能在后面插入）插入一个节点S：

<img src="4-3-21.png" alt="image-20231022155406548" style="zoom:67%;" />

这里一共有两个指针操作，即P的next指针和S的next指针。这两个操作的顺序是要有先后顺序的，一定是先让S的next指针指向P->next，然后再让P的next指针指向S。这里的原因是，我们目前可以明确知道P和S节点，获取未知节点（P->next）的唯一方法就是通过P的next指针。如果我们先修改P的next指针使其指向S，那么就丢失了获取P节点后面节点的方法。所以这里一定是先让S的next指针指向后面的节点，此时有两种方式（S->next和P->next）可以获取到后面的节点，所以接下来就可以修改其中一种方式（P->next）。

下面以双链表为例，假如需要在节点P之前插入一个新节点S：

<img src="4-3-22.png" alt="image-20231022180633816" style="zoom:67%;" />

首先这里一共需要四个指针操作，分别是P->prev、T(P->prev)->next、S->next、S->prev。
这里需要注意的是，T是未知节点，目前只能通过P->prev来访问。所以如果想要更新这个指针，必须先增加一个指针指向T节点，也就是必须先令S->prev指向T，所以必须先更新S->prev才能更新P->prev。其它的操作顺序没有要求。
这里采用的顺序先确定从S节点出去的指针，即S->next和S->prev；再确定指向S节点的指针，即T->next和P->prev。






### list的一些元素操作

#### 插入和删除操作

<img src="4-3-16.png" alt="image-20231022104056753" style="zoom:67%;" />

<img src="4-3-17.png" alt="image-20231022104231361" style="zoom:67%;" />

<img src="4-3-18.png" alt="image-20231022104506672" style="zoom:67%;" />

<img src="4-3-19.png" alt="image-20231022104528183" style="zoom:67%;" />

#### 迁移操作

list内容提供一个迁移操作：将某段连续范围的元素迁移到某个指定位置之前。这个操作为其它复杂操作如splice、sort、merge提供了基础。

<img src="4-3-20.png" alt="image-20231022111956967" style="zoom:67%;" />

对于迁移操作，一共需要操作6个指针。需要将范围[first, last - 1]插入到P节点之前，这是4个指针的操作；还需要将first - 1的节点与last节点相连，这是2个指针的操作。

<img src="4-3-24.png" alt="image-20231023210851380" style="zoom:67%;" />

这里需要明确的是，涉及的节点一共有5个，已知的是P、first、last，不能直接取得的是P - 1(P->prev)、firs - 1(first->prev)、last - 1(last->prev)，再修改这些指针之前一定要保证存在另一个指向这个节点的指针。但是这里注意，由于firs - 1(first->prev)需要指向P - 1(P->prev)，所以这里一定是需要一个临时指针取保存其中一个的。因此6次指针再加一次保存临时指针的操作，一共是7个操作。
下面对于这7次操作的顺序进行演示，这里的逻辑是：先连通一个方向（保证first-1 ->last和P-1->first->...->last - 1->P），再连通另一个方向（last->first-1和P-1->last-1 -> ...->first->P-1）。

<img src="4-3-23.png" alt="image-20231023210828563" style="zoom:67%;" />

上面的transfer并不是对外接口，对外的接口是接合操作splice：将某连续范围的元素从一个list移动到另一个（或同一个list）的某个定点。

<img src="4-3-25.png" alt="image-20231023211156558" style="zoom:67%;" />

为了提供各种接口的弹性，splice有许多版本：

<img src="4-3-26.png" alt="image-20231023212140513" style="zoom:67%;" />



merge函数：将另一个链表合并到当前链表上。这里要求两个链表已经经过递增排序。

<img src="4-3-27.png" alt="image-20231023212545025" style="zoom:67%;" />



reverse函数：将list逆向

<img src="4-3-28.png" alt="image-20231023212921145" style="zoom:67%;" />



sort函数：采用快速排序算法

<img src="4-3-29.png" alt="image-20231023213059269" style="zoom: 80%;" />
