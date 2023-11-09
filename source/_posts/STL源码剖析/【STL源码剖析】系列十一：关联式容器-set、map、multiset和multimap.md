---
title: 【STL源码剖析】系列十一：关联式容器--set、map、multiset和multimap
keywords: 'STL源码剖析, set, map, multiset, multimap'
categories:
  - STL源码剖析
tags:
  - STL
  - 读书笔记
typora-root-url: 【STL源码剖析】系列十一：关联式容器-set、map、multiset和multimap
abbrlink: 2860365271
date: 2023-11-09 20:59:58
---

## 前言

在了解STL的RB-tree（红黑树）数据结构之后，实现set和map就非常简单，因为set和map均以红黑树作为底层实现（因此set和map中的元素实际上均是一个有序的序列），其对外接口均是调用底层实现的接口。本篇主要内容：

1.set和map的特性以及如何基于红黑树实现容器；

2.multiset和multimap如何基于红黑树实现。

注：本篇基本上是对set、map、multiset、multimap源代码的注释，实现技巧在红黑树中已经说明。

因此比较枯燥。

<!-- more -->

## set

### set概述

set（集合）中的所有元素都会根据元素的键自动被排序，每个元素的键(key)就是实值(value)。

set中不允许存在两个元素具有相同的键值，也就是不允许存在两个相同的元素。

正是由于set中实值就是键值，因此不能通过迭代器随意更改实值。set容器中底层实现是RB-tree，也就是说所有元素都是按照一定的键值顺序排列的，如果随意更改实值，那么键值也会随之改变，破坏了二叉搜索树的结构。

由于set和list一样均使用离散式存储结构（都以结点表示元素），因此当客户端进行新增操作或者删除操作时，操作之前的迭代器不会失效。

### set源代码摘录

由于RB-tree是一种平衡二叉搜索树，其自动排序以及查找的效率很不错，因此STL set以红黑树作为底层机制。

并且set需要对外开放的接口，均可以调用红黑树的接口来实现。

<img src="5-3-1.png" alt="image-20231109192131007" style="zoom:67%;" />

<img src="5-3-2.png" alt="image-20231109192634762" style="zoom:67%;" />

<img src="5-3-3.png" alt="image-20231109192723713" style="zoom:67%;" />

<img src="5-3-4.png" alt="image-20231109193224489" style="zoom:67%;" />

<img src="5-3-5.png" alt="image-20231109193359150" style="zoom:67%;" />

<img src="5-3-6.png" alt="image-20231109193753483" style="zoom:67%;" />

<img src="5-3-7.png" alt="image-20231109193829693" style="zoom:67%;" />

<img src="5-3-8.png" alt="image-20231109194003348" style="zoom:67%;" />

### set的find操作

对于关联式容器来说，应该使用其内部提供的find函数来进行查找元素，这样比使用STL算法的find更有效率。

这是因为STL的算法只是根据迭代器遍历搜寻，而关联式容器的搜寻会基于其底层实现来进行搜寻，比如set的底层实现是红黑树，那么对于n个元素的搜寻效率是O(logn)，而STL的算法采用遍历搜寻，搜寻效率是O(n)。

## map

### map概述

map（字典）中的每一个元素都是一对键值（value）和实值（value），所有元素根根据**键值**（key）自动排序。

map中不允许存在键值相同的两个元素。

map中的每一个元素是一个pair，pair的第一元素是键值（key），第二元素是实值（value），定义如下：

<img src="5-3-9.png" alt="image-20231109195524098" style="zoom:67%;" />

<img src="5-3-10.png" alt="image-20231109195541100" style="zoom:67%;" />

对于map来说，如果想要通过迭代器修改键值（key），这是不被允许的，因为键值关系到所有元素的排列规则；如果想要通过迭代器改变实值（value）是可以的。

由于map和set、list一样采用离散式存储结构，所以当进行增加、删除等操作之后，原来的迭代器仍然可以正常使用。

### map源代码摘录

map同样以RB-tree作为底层实现机制，并且map需要向外提供的接口，红黑树中已经提供，所以其实现比较简单：

<img src="5-3-11.png" alt="image-20231109200334538" style="zoom:67%;" />

<img src="5-3-12.png" alt="image-20231109200522220" style="zoom:67%;" />

<img src="5-3-13.png" alt="image-20231109200709761" style="zoom:67%;" />

<img src="5-3-14.png" alt="image-20231109201115338" style="zoom:67%;" />

<img src="5-3-15.png" alt="image-20231109201153114" style="zoom:67%;" />

<img src="5-3-16.png" alt="image-20231109201254950" style="zoom:67%;" />

<img src="5-3-17.png" alt="image-20231109201453165" style="zoom:67%;" />

<img src="5-3-18.png" alt="image-20231109201649418" style="zoom:67%;" />

注意，在map的实现中，只要返回值是迭代器的函数，一般都有两个版本，一个返回普通迭代器，一个返回const 迭代器。

### 针对下标访问方式的说明

map中可以使用下标加键值的方式访问元素的实值：

<img src="5-3-19.png" alt="image-20231109202647450" style="zoom:67%;" />

上面A式的说明：

```c++
// 首先，最内层创建一个元素对象：
// 这个元素对象的实值不重要
// 键值必须是通过[]传递的即可
value_type(k, T())；
    
    
// 接着，尝试调用insert函数将该元素根据键值key插入到红黑树中
// 假设键值已经存在，那么一定会找到这样一个键值为k的结点
insert(value_type(k, T()))；

    
// 插入操作返回值一个pair，第一个元素是迭代器，第二个元素表示是否能够插入
// 如果键值存在，那么第一个元素刚好就是要找的结点
(insert(value_type(k, T()))).first
// 解引用这个迭代器可以得到元素
*((insert(value_type(k, T()))).first)
// map的每个元素都是pair，第二个是value
(*((insert(value_type(k, T()))).first)).second

// 如果键值不存在就会报错
    
```



## multiset

multiset与set特性与用法与set基本相同，唯一的区别在于multiset允许键值重复，因此它的插入操作采用的是底层红黑树的insert_equal（set采用的是insert_unique）。

下面是multiset的源代码摘要，只列出与set的不同之处：

<img src="5-3-20.png" alt="image-20231109205555864" style="zoom:80%;" />

## multimap

multimap的特性和用法与map完全相同，唯一的区别就是multimap允许键值重复，因此插入操作采用的是底层红黑树的insert_equal。

下面是源代码摘要，只列出与map的不同之处：

<img src="5-3-21.png" alt="image-20231109205751827" style="zoom: 80%;" />







