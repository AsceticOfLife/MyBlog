---
title: >-
  【STL源码剖析】系列十三：关联式容器-unordered-set、unordered-map、unordered-multiset和unordered-multimap
keywords: 'STL源码剖析, unordered_set, unordered_map, unordered_multiset, unordered_multimap'
categories:
  - STL源码剖析
tags:
  - STL
  - 读书笔记

  【STL源码剖析】系列十三：关联式容器-unordered-set、unordered-map、unordered-multiset和unordered-multimap
abbrlink: 4268388846
date: 2023-11-13 17:51:35
typora-root-url: 【STL源码剖析】系列十三：关联式容器-unordered-set、unordered-map、unordered-multiset和unordered-multimap
---

## 前言

在了解hashtable之后，下面就可学习基于hashtable的set和map。在之前的介绍中，我们直到set和map主要是为了提升查找效率，所以采用红黑树的底层机制，查找效率可以达到O(logn)。但是这是基于元素的随机性。

而对于hashtable而言，不需要基于元素的随机性假设就可以使得元素的存取和删除达到常数级。

本篇主要是介绍基于hashtable实现的set、map、multiset和multimap。虽然这里介绍时采用的是C++11标准之前的hash_set、hash_map、hash_multiset和hash_multimap，但是设计思想是一致的，只是在一些接口上有些许差别。

本篇主要内容：

1.基于hashtable实现的set和map；

2.基于hashtable实现的multiset和multimap。

注：基于hashtable实现的set和map对外接口在hashtable几乎以及实现，因此只需要调用即可，实现细节参考hashtable。本篇基本只是代码注释和简介，会比较枯燥。

<!-- more -->

## hash_set(C++11中的unordered_set)

之前介绍的set底层实现是RB-Tree（红黑树），但是SGI提供了另一种以hashtable作为底层机制的set（C++11标准中是unordered_set）

使用set的目的是快速查找元素，这一点不管是红黑树还是哈希表都能完成。但是底层实现是红黑树的set实际上是有序的，而底层实现是哈希表的set是无序的(注意，如果插入元素小于哈希表大小，那么可能会造成有序的假象)。

hash_set所有对外接口都由hashtable提供，所以hash_set的所有行为都是调用哈希表的行为。

注意set的元素键值就是实值，hash_set与set的使用方式完全相同。

下面是hasn_set的源代码摘录：

<img src="5-8-1.png" alt="image-20231111082647551" style="zoom:67%;" />

<img src="5-8-2.png" alt="image-20231111082953021" style="zoom:67%;" />

<img src="5-8-3.png" alt="image-20231111083217683" style="zoom:67%;" />

<img src="5-7-32.png" alt="image-20231111083358643" style="zoom:67%;" />

<img src="5-8-4.png" alt="image-20231111083518185" style="zoom:67%;" />

<img src="5-8-5.png" alt="image-20231111084141988" style="zoom:67%;" />

<img src="5-8-6.png" alt="image-20231111084251327" style="zoom:67%;" />

## hash_map(C++11中的unorder_map)

与set一样，map也可以使用hashtable作为底层实现机制，表示为hash_map，使用与map完全一样。

hash_map内部元素同样是无序的。

下面是源代码摘录：

<img src="5-8-7.png" alt="image-20231111085126085" style="zoom:67%;" />

<img src="5-8-8.png" alt="image-20231111085326469" style="zoom:67%;" />

<img src="5-8-9.png" alt="image-20231111085816823" style="zoom:67%;" />

<img src="5-8-10.png" alt="image-20231111091116075" style="zoom:67%;" />

<img src="5-8-11.png" alt="image-20231111091304595" style="zoom:67%;" />

<img src="5-8-12.png" alt="image-20231111091714339" style="zoom:67%;" />

<img src="5-8-13.png" alt="image-20231111092032739" style="zoom:67%;" />

## hash_multiset(C++11中的unordered_multiset)

hash_multiset特性与multiset完全相同，唯一的区别就是底层实现是hashtable。

hash_multiset实现上与hash_set的唯一区别就是：前者的插入操作使用底层机制hashtable的insert_equal，而后者采用insert_unique。

因此这里不再列举源代码。



## hash_multimap(C++11中的unordered_map)

hash_multimap特性与multimap完全相同，唯一的区别就是底层实现是hashtable。

hash_multimap实现上与hash_map的唯一区别就是：前者的插入操作使用底层机制hashtable的insert_equal，而后者采用insert_unique。

因此这里不再列举源代码。
