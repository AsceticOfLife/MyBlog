---
title: 【STL源码剖析】系列二：空间配置器allocator
categories:
  - STL源码剖析
tags:
  - STL
  - 读书笔记
typora-root-url: 【STL源码剖析】系列二：空间配置器allocator
abbrlink: 3076589165
date: 2023-10-19 09:13:27
---

# 前言

STL的空间配置器，负责的主要工作是容器的空间管理，一般情况下空间都是指内存（存在将空间置于外存的配置器），所以也可称为内存管理器。

读完《STL源码剖析》第二章内容，需要知道的是：
1.SGI标准下将空间配置和构造析构分为两个阶段。空间配置是申请和释放内存；构造析构是对象的创建和销毁。
2.SGI标准下的空间配置器alloc具有两级，第一级负责128字节以上的空间申请，第二级以内存池的形式管理。二者均以字节为单位管理内存，对外提供标准接口以元素为单位管理内存。

![](2-2-10.png)

3.对于未初始化的空间除了构造和析构的支持外，还提供复制、填充、以对象x填充的管理操作。

<!-- more -->

# 空间配置器allocator

<img src="2-1-1.png" alt="image-20231008091511056" style="zoom:67%;" />

<img src="2-1-2.png" alt="image-20231008091534119" style="zoom:67%;" />

## 空间配置器的标准接口

<img src="2-1-3.png" alt="image-20231008105406145" style="zoom:67%;" />

<img src="2-1-4.png" alt="image-20231008105431614" style="zoom:67%;" />

<img src="2-1-5.png" alt="image-20231008105452081" style="zoom:67%;" />

### 设计一个简单的空间配置器：JJ::allocator

知识点1：ptrdiff_t是表示指针之间的差距的数据类型，可以是负数；size_t是表示数组长度的类型，必须是正数。

知识点2：**set_new_handler()**函数定义在头文件new中，当operator new函数申请分配内存失败时，会尝试调用错误处理函数，这个函数如果不存在就会抛出异常。使用set_new_handler函数可以指定这个handler函数。

知识点3：placement new指的是在用户指定的内存位置上申请空间。语法格式为：

```c++
// address：placement new所指定的内存地址
// ClassConstruct：对象的构造函数
Object * p = new (address) ClassConstruct(...);
```

知识点4：作用域解析运算符::(1)作用1是表示指定类中的成员或者命名空间中的成员，比如使用A::member表示类A中的成员；(2)作用2是当前面没有类或者命名空间时，表示全局作用域符号，可以用于区分同名的全局和局部变量。

```c++
int jmz = 2; //全局变量
int main() {
    int jmz = 3; //局部变量
    jmz = jmz* jmz;//局部=局部*局部
    ::jmz = ::jmz* jmz;//全局=全局*局部
    cout << jmz << endl;
    cout << ::jmz << endl;
}
```

自定义一个allocator类：

```c++
#ifndef _JJALLOC_
#define _JJALLOC_

#include <new> // for placement new
#include <cstddef> // for ptrdiff_t, size_t
#include <cstdlib> // for exit()
#include <climits> // for UINT_MAX
#include <iostream> // for cerr

namespace JJ {

// allocate的实际实现，申请size个T类型的空间
template <typename T>
inline T* _allocate(ptrdiff_t size, T*) {
    set_new_handler(0); // 设置当new失败时的处理函数为空，所以会直接退出
    T* tmp = (T*)(::operator new((size_t)(size * sizeof(T)))); // 申请size个T类型的空间
    // 如果得到空指针
    if (tmp == 0) {
        cerr << "out of memory" << endl;
        exit(1); // 退出
    }

    return tmp;
}

// deallocate的实际实现，释放指定位置内存
template <typename T>
inline void _deallocate(T *buffer) {
    ::operator delete(buffer); // 调用global operator delete
}

// construct的实际实现，在指定位置创建一个对象
template <typename T1, typename T2>
inline void _construct(T1 *p, const T2 &value) {
    new(p) T1(value);
}

// destory的实际实现，调用指定对象的析构函数
template <typename T>
inline void _destory(T *ptr) {
    ptr->~T();
}

// 自定义的空间配置器类
template <typename T>
class allocator {
public:
    // 定义类型
    typedef T           value_type;
    typedef T*          pointer;
    typedef const T*    const_pointer;
    typedef T&          reference;
    typedef const T&    const_reference;
    typedef size_t      size_type;
    typedef ptrdiff_t   difference_type;

    // 一个嵌套的类模板
    // rebind allacator of type U
    template <typename U>
    struct rebind {
        typedef allocator<U> other;
    }

    // 配置空间，用于存储n个T对象
    // hint used for locality
    pointer allocate(size_type n, const void *hint = 0) {
        return _allocate((difference_type)n, (pointer)0);
    }

    // 归还配置的空间
    void deallocate(pointer p, size_type n) {
        _deallocate(p);
    }

    // 使用T对象value在指定位置p创建一个新对象T
    void construct(pointer p, const T &value) {
        _construct(p, value);
    }

    // 析构指定位置p的对象T
    void destory(pointer p) {
        _destory(p);
    }

    // 返回某个对象的地址
    pointer address(reference x) {
        return (pointer)&x;
    }

    // 返回某个const对象的const地址
    const_pointer const_address(const_reference x) {
        return (const_pointer)&x;
    }

    // 返回可成功配置的最大值
    size_type max_size() const {
        return size_type(UINT_MAX/sizeof(T));
    }

};


} // end of namespace JJ

#endif
```

使用MinGW无法通过编译，**因为SGI STL的allocator并不完全符合STL规范，我们编写的符合规范的自然不能搭配使用。**

## 具备层次配置能力的SGI空间配置器

<img src="2-2-1.png" alt="image-20231009093741874" style="zoom: 67%;" />

### SGI标准的空间配置器std::allocator

这个allocator部分符合STL标准，它在文件 defalloc.h 中实现。但是SGI STL的容器并不使用它，也不建议我们使用，它存在的意义仅在于为用户提供一个兼容老代码的折衷方法，其实现仅仅是对new和delete的简单包装因此效率不高。

### SGI特殊的空间配置器std::alloc

C++内存配置操作和释放操作：

```c++
class Foo {...};
Foo *pf = new Foo; // 配置内存，然后构造对象
delete pf;	// 析构对象，然后释放内存
```

new运算符包括两给阶段的操作：（1）调用::operatro new配置内存；（2）调用Foo::Foo()构造对象内容；

delete运算符号包括两阶段的操作：（1）调用Foo::~Foo()将对象析构；（2）调用::operator delete释放内存。

<img src="2-2-2.png" alt="image-20231009193732479" style="zoom:67%;" />

<img src="2-2-3.png" alt="image-20231009194118094" style="zoom:67%;" />

<img src="2-2-4.png" alt="image-20231009194159187" style="zoom:67%;" />

### 构造和析构的基本工具：construct()和destory()

construct()和destory()示意图：
<img src="2-2-5.png" alt="image-20231009200509941" style="zoom:67%;" />

下面是作者注释的<stl_construct.h>的部分内容，主要是为了说明construct和destroy的功能。（不同版本的SGI STL代码可能不同，但是设计思路是一致的。）

<img src="2-2-6.png" alt="image-20231009201210619" style="zoom:67%;" />

<img src="2-2-7.png" alt="image-20231009201251845" style="zoom:67%;" />

这两个构造和析构的函数被设计为全局函数，STL规定配置器应该具有construct和destroy两个成员函数，但是实际上SGI STL中的std::alloc配置器并没有遵守这一规则。

关于上面的construct函数：接收一个指针p和一个初始值value，改函数的作用就是将初始值value设置到指针所指的空间上。（使用C++ placement new完成）

关于上面的destroy函数：第一个版本接受一个指针，将指针所指的对象析构。第二个版本接收first和last两个迭代器，准备将[first, last)内的对象析构。考虑到这个范围可能很大，并且如果每个对象的析构函数都是trivial destructor(即系统默认的析构函数)，那么会降低效率。因此这里的方法是先利用**vaule_type()**获得迭代器所指对象的类型，再利用**__type_traits<T\>**判断这种类型的析构函数是否无关痛痒。如果是(\_\_true_type)，那么什么都不做；如果否(\__false_type)，以循环方式遍历每一个对象，并对于每一个对象调用第一个版本的destroy()。

### 空间配置和释放，std::alloc

<img src="2-2-8.png" alt="image-20231010114141367" style="zoom:67%;" />

为了解决小型区块可能造成的内存破碎的问题，SGI设计采用了双层配置器，第一级配置器直接使用malloc和free，第二级配置器当配置区块超过128bytes时，认为内存够大调用第一级配置器，否则认为过小，采用内存池（memory pool）的方式。第一级和第二级配置器的关系：

<img src="2-2-9.png" alt="image-20231010114558211" style="zoom:67%;" />

无论alloc被定为第一级还是第二级配置器，SGI都为其增加一层包装为simple_alloc。第一级配置器和第二级配置器的包装以及使用方式：

<img src="2-2-10.png" alt="image-20231010114745673" style="zoom: 80%;" />

### 第一级配置器解析malloc_alloc_template

第一级配置器以malloc()、free()、realloc()等C函数实现实际的内存配置、释放、重配置操作。

```c++
void * malloc(size_t size); // 申请大小为size个字节的内存，如果分配成功返回一个void *指针，否则返回空指针
void * realloc(void *ptr, size_t size); // 更改已经分配的内存空间的大小，返回新的空间地址
void * calloc(size_t num, size_t size); // 申请num个大小为size个字节的内存空间
void  free(void *ptr); // 释放指针指向内存
```

（第一级配置器定义为template <int inst\> class __malloc_alloc_template，在使用时会指定inst为0，

```c++
typedef __malloc_alloc_template<0> malloc_alloc; // 之后就可以将malloc_alloc作为第一级配置器的名称
```

这样用户在不知道这个类是什么模板的情况下无法生成其它模板。下面均是类中的成员定义）

<img src="2-2-11.png" alt="image-20231010161734519" style="zoom:67%;" />

<img src="2-2-12.png" alt="image-20231010161806524" style="zoom:67%;" />

由于作者采用的版本仍然使用malloc等C函数，所以并没有用到C++中的new-handler机制（即当new申请空间失败时，在抛出std::bad_alloc异常之前，可以选择调用一个指定的处理函数，参考《Effective C++》第三版条款49）。所以这个版本的SGI第一级配置器实现了一个类似的set_malloc_handler：

（下面仍然是第一级配置器类内的成员函数）

<img src="2-2-13.png" alt="image-20231010162652236" style="zoom:67%;" />

（下面是在类外进行malloc-handler函数的初始化）

<img src="2-2-14.png" alt="image-20231010162717093" style="zoom:67%;" />

<img src="2-2-15.png" alt="image-20231010162823575" style="zoom:67%;" />

如果第一级配置器的allocate和realloca在调用malloc和realloc失败之后，改为调用oom_malloc和oom_realloc，后二者都有内循环，不断调用”内存不足处理例程“（即new-handler），希望能够处理好内存分配工作。但是如果未设置new-handler（即为空指针时）则直接抛出异常。

<img src="2-2-16.png" alt="image-20231010163535768" style="zoom:67%;" />

<img src="2-2-17.png" alt="image-20231010163624088" style="zoom:80%;" />

### 第二级配置器解析default_alloc_template

为什么需要第二级配置器？避免太多小额区块造成内存碎片，以及配置时的额外负担（overhead）。如果区块越小，那么额外负担所占比例就越大。
<img src="2-2-18.png" alt="image-20231010184810091" style="zoom: 67%;" />

第二级配置器的做法是：如果申请区块超过128byte，交给第一级配置器处理；否则以内存池管理，此方法又称为次层配置。

所谓内存池，就是先配置一大块内存，使用自由链表（free-list）管理，当有需要相同大小的内存需求时，从自由链表中取出一块内存使用；如果归还空间，就添加到链表上。

为了方便管理，第二级配置器会把任何区块需求量上调至8字节的倍数，一共维护16个free-list，大小分别为8，16，...，128。free-list的结点结构为：

```c++
union obj {
	union obj *free_list_link;
    char client_data[1];
};
```

使用union节省空间的巧妙之处：当没有作为用户使用的空间时，在链表上使用第一个字段指向下一个obj；当作为用户使用的空间时，使用第二个字段指向实际区块。

<img src="2-2-19.png" alt="image-20231010185528301" style="zoom:67%;" />

下面是第二级配置器的实现内容：

<img src="2-2-20.png" alt="image-20231010185623532" style="zoom:67%;" />

<img src="2-2-21.png" alt="image-20231011153809092" style="zoom:67%;" />

<img src="2-2-22.png" alt="image-20231010185751889" style="zoom:67%;" />

<img src="2-2-23.png" alt="image-20231010185811896" style="zoom:67%;" />

### 空间配置函数allocate

接口函数**allocate()**的主要流程是：先判断区块大小，如果超过128byte，就交给第一级配置器；小于128就检查对应的free list，如果链表上有可用的区块，就直接取第一块来用，如果没有，就将区块大小上调至8的倍数，然后调用**refill()**函数为这一个free list填充空间。

```c++
static void * allocate(size_t n) {
    obj * volatile * my_free_list;
    obj *result;
    // 大于128就调用第一级配置器
    if (n > (size_t)__MAX_BYTES) {
        return malloc_alloc::allocate(n);
    }
    
    // 循环16个free list中适当的一个
    my_free_list = free_list + FREELIST_INDEX(n);
    result = *my_free_list;
    if (result == 0) {
        // 如果free list上没有空闲的区块就重新填充区块
        void *r = refill(ROUND_UP(n));
        return r;
    }
    
    // 取出当前free list的第一个区块
    *my_free_list =- result->free_list_link;
    return result;
}
```

<img src="2-2-24.png" alt="image-20231011154406479" style="zoom:67%;" />

### 空间释放函数deallocate

接口函数**deallocate()**主要流程是：如果内存块大于128byte，调用第一级配置器释放内存；否则找到区块大小对应的free list，将这个区块插在链表第一个位置。

```c++
static void deallocate(void *p, size_t n) {
    obj *q = (obj *)p;
    obj * volatile *my_free_list;
    
    // 大于128就调用第一级配置器
    if (n > (size_t)__MAX_BYTES) {
        malloc_alloc::deallocate(p, n);
        return;
    }
    
    // 找到对应大小的free list
    my_free_list = free_list + FREELIST_INDEX(n);
    // 调整free list，将区块插入为第一个
    q->free_list_link = *my_free_list;
    *my_free_list = q;
}
```

<img src="2-2-25.png" alt="image-20231011161922484" style="zoom:67%;" />

### 重新填充链表函数refill

当allocate发现适当位置的链表为空时，就调用**refill()**为这一个链表重新填充空间。新的空间取自内存池（由chunk_alloc完成）。默认情况下获取20个新区块，如果内存池空间不足，获取区块数会小于20.

<img src="2-2-26.png" alt="image-20231011163212262" style="zoom:67%;" />

<img src="2-2-27.png" alt="image-20231011163410061" style="zoom:67%;" />

### 内存池(memory pool)管理

**chunk_allco**函数以end_free - start_free（单位是字节数）来判断内存池中的空间。
如果空间足够满足总需求量（默认是20个需求大小区块），那么直接将空间分配；
如果空间不够总需求量，但是可以满足至少一个（含）以上需求区块大小，那么修改默认返回的区块大小（也就是将20转换为实际上能够满足的区块数量），然后将这些空间返回；
如果空间连一个需求的区块大小都不能满足，那么就需要向堆区申请内存，申请内存大小为2倍的总需求量加上一个随着分配次数越来越大的附加量。不过在正式向堆区申请内存之前，先将内存池中的零头分配给适当的free list。之后正式调用**malloc**函数向堆区申请内存。如果申请成功，那么就调用自身将需求满足；如果申请失败，那么先尝试将free list中未使用并且足够大的（也就是空间至少大于1个需求区块大小）区块拿出来，补充内存池空间，然后调用自己分配内存。如果free list后面的均为空，那么最后尝试将任务交给第一级配置器尝试进行分配。

<img src="2-2-28.png" alt="image-20231015172629792" style="zoom:67%;" />

<img src="2-2-29.png" alt="image-20231015172815949" style="zoom:67%;" />

<img src="2-2-30.png" alt="image-20231015173027951" style="zoom:67%;" />

<img src="2-2-31.png" alt="image-20231015173123208" style="zoom:67%;" />

<img src="2-2-32.png" alt="image-20231015173205110" style="zoom:67%;" />



### 总结

无论是第一级还是第二级配置器，最终都被**simple_alloc**包装：

<img src="2-2-33.png" alt="image-20231015173352882" style="zoom:67%;" />

使用配置器的方法：

<img src="2-2-34.png" alt="image-20231015173532284" style="zoom:67%;" />

<img src="2-2-35.png" alt="image-20231015173546855" style="zoom:67%;" />

缺省参数alloc已经被typename为第一级配置器或者第二级配置器，SGI STL将其设置为第二级配置器。

## 内存处理基本工具

STL定义五个全局函数，用于处理未初始化空间。前两个是用于构造的**construct**和用于析构的**destroy**，另外三个是uninitialized_copy()、uninitialized_fill()、uninitialized_fill_n()，对应着高层次函数copy、fill、fill_n。如果想要使用这三个此层次函数，应该包含头文件memory，但是实际定义于stl_uninitialized。

### 概述

**uninitialized_copy：**

```c++
template <typename InputIterator, typename ForwardIterator>
ForwardIterator
uninitialized_copy(InputIterator first, InputIterator last, ForwardIterator result);
```

将内存的申请与对象的初始化行为分开，主要负责完成对象的初始化功能。

对于目的地result，调用复制构造函数，利用[first, last)的每一个迭代器指向的对象复制一个新的对象。相当于调用了construct(&*(result + (i - first)), *i)，在相应的位置产生相应的复制品。

如果需要实现一个容器，容器的全区间构造函数（range constructor）通常以两个步骤完成：
1.申请内存空间，足以包含范围内所有元素；
2.使用uninitialized_copy在内存空间上构造元素。

C++标准要求具有commit or rollback特性，也就是要么构造出所有元素，要么一个都不构造。

**uninitialized_fill：**

```c++
template <typename ForwardIterator, typename T>
void uninitialized_fill(ForwardIterator first, ForwardIterator last, const T &x);
```

将内存的申请与对象的初始化行为分开，主要负责完成对象的初始化功能。

对于[first, last)迭代器指向的内存，均使用x进行初始化。也就是对于范围内的每个迭代器 i 指向的内存地址调用construct(&*i, x)。

同样需要具有commit or rollback特性。

**uninitialized_fill_n：**

```c++
template <typename ForwardIterator, typename Size, typename T>
ForwardIterator
uninitialized_fill_n(ForwardIterator first, Size n, const T &x);
```

将内存的申请与对象的初始化行为分开，为指定范围内的所有元素设定相同的初始值。

对于[first, first + n)范围内的每一个迭代器指向的内存调用复制构造函数，产生x的复制品。

同样需要具有commit or rollback特性。



### 源代码

**uninitialized_fill_n：**

<img src="2-3-1.png" alt="image-20231016091351869" style="zoom:67%;" />

<img src="2-3-2.png" alt="image-20231016091507205" style="zoom:67%;" />

<img src="2-3-3.png" alt="image-20231016091617090" style="zoom:67%;" />



**uninitialized_copy：**

<img src="2-3-4.png" alt="image-20231016091714181" style="zoom:67%;" />

<img src="2-3-5.png" alt="image-20231016091755619" style="zoom:67%;" />

<img src="2-3-6.png" alt="image-20231016091905466" style="zoom:67%;" />

<img src="2-3-7.png" alt="image-20231016091935680" style="zoom:67%;" />



**uninitialized_fill：**

<img src="2-3-8.png" alt="image-20231016092012397" style="zoom:67%;" />

<img src="2-3-9.png" alt="image-20231016092106840" style="zoom:67%;" />

### 源代码总结

<img src="2-3-10.png" alt="image-20231016092217446"  />
