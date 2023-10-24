---
title: 【STL源码剖析】系列一：STL概述与版本介绍
categories:
  - STL源码剖析
keywords: STL源码剖析, STL概述
tags:
  - STL
  - 读书笔记
typora-root-url: 【STL源码剖析】系列一：STL概述与版本介绍
abbrlink: 2713497384
date: 2023-10-19 08:25:57
---

## 前言

本章是《STL源码剖析》的第一章内容摘要和读书笔记，主要是介绍STL的发展历史和几个不同的版本。

读完本章需要了解的是STL六大组件之间的关系。

![](1-2-4.png)

<!-- more -->

## STL概论

STL低层次来说是一个组件库，方便使用；高层次来说是一个抽象概念库，定义了一些概念应该具有的标准。

<img src="1-1-1.png" alt="image-20231007085138361" style="zoom:67%;" />

## STL六大组件 功能和运用

<img src="1-2-1.png" alt="image-20231007085543857" style="zoom:67%;" />

<img src="1-2-2.png" alt="image-20231007085616544" style="zoom:67%;" />

<img src="1-2-3.png" alt="image-20231007090023570" style="zoom:67%;" />

<img src="1-2-4.png" alt="image-20231007090139020" style="zoom:67%;" />



STL源码就在C++的头文件中，一般存在两个版本，具有扩展名（为了向下兼容）和不具有扩展名（C++建议的命名方式）。GNU C++的SGI版本不但有一线的<vector.h>和<vector\>，还有二线的<stl_vector.h>

## GNU源码开放精神

<img src="1-3-1.png" alt="image-20231007090636314" style="zoom:67%;" />

实际上是一种代码前的声明，即允许使用，但是条件是加上这个声明。

由于出现了很多不同的开放源码分支，为了整合各方，后面又出现了open source名字，只要符合9条定义就是open source软件。

GCC全称是GNU Compiler Collection，是由GNU开发的编程语言编译器。

## HP实现版本

STL最早的版本是HP实现版本，需要遵守的是在所有文文件加上HP的版本声明，这种授权不属于GNU GPL，属于open source。

## P.J.Plauger实现版本

PJ版本继承HP版本，但是还加上了自己的声明，因此既不属于open source，也不属于GNU GPL。

被Visual C++采用。由于Visual C++对于C++的语言特性支持并不理想，所以PJ版本变现受到影响。

## Rouge Wave实现版本

由Rouge Wave公司开发，继承HP版本，加上自己的版权声明。因此既不属于open source，也不属于GNU GPL。

由C++ Bulider采用。

## STLport实现版本

<img src="1-7-1.png" alt="image-20231007093229552" style="zoom:67%;" />

## SGI STL实现版本

SGI版本由Silicon Graphics Computer System，Inc.公司发展，继承HP版本，属于open source的一员，但是不属于GNU GPL。

SGI版本被GCC采用，可以在GCC的include字幕下找到所有的STL文件。

### GNU C++头文件分布

众多头文件大致可以分为：

<img src="1-8-1.png" alt="image-20231007093907304" style="zoom:67%;" />

<img src="1-8-2.png" alt="image-20231007093927544" style="zoom:67%;" />

### SGI STL文件分布与简介

1.STL标准头文件（无扩展名）：

<img src="1-8-3.png" alt="image-20231007094131488" style="zoom:67%;" />

2.C++ Standard定案前，HP规范的STL头文件（扩展名.h）：

<img src="1-8-4.png" alt="image-20231007094300435" style="zoom:67%;" />

<img src="1-8-5.png" alt="image-20231007094325649" style="zoom:67%;" />

3.SGI STL内部私用文件（SGI STL真正实现于此）：

<img src="1-8-6.png" alt="image-20231007094425418" style="zoom:67%;" />

<img src="1-8-7.png" alt="image-20231007094448737" style="zoom:67%;" />

<img src="1-8-8.png" alt="image-20231007094518419" style="zoom:67%;" />

### SGI STL的编译器组态设置

（“组态(Configure)”的含义是“配置”、“设定”、“设置”等意思，是指用户通过类似“搭积木”的简单方式来完成自己所需要的软件功能，而不需要编写计算机程序，也就是所谓的“组态”。）

SGI STL准备了一个环境组态文件<stl_config.h>定义了许多常量，标志着某些组态成立与否。所有STL文件都会直接或者间接包含这个文件，以条件式写法，让预处理器根据常量取舍一段程序代码。

通过这些组态能够判断对于C++特性的支持情况。

## 可能会令你困惑的语法

指的是一些C++中的语法层次的特性。

### stl.config中出现的组态

#### 1.**模板类**中的**静态数据成员**初始化

常量__STL_STATIC_TEMPLATE_MEMBER_BUG

（1）首先搞清楚隐式实例化、显式实例化、显式具体化的意思，这里以函数模板来说明含义：
隐式实例化：根据传递的参数类型由编译器根据模板定义生成函数定义；
显式实例化：主动声明对哪种参数类型生成函数定义，使用的是通用模板定义；
显式具体化：对于特定的类型生成与通用模板不同的函数定义。

```c++
#include <iostream>

// 通用模板
template <typename T>
void swap(T &a, T &b);

// 显式实例化
// 不用提供定义，使用通用模板的定义
template void swap<int>(int &a, int &b);

// 显式具体化
// 需要提供定义，使用不同于通用模板的定义
template <> void swap<double>(double &a, double &b);

int main(void) {
    char a = 'a', b = 'b';
    swap(a, b); // 隐式实例化

    int a1 = 1, b1 = 2;
    swap(a1, b1); // 显式实例化

    double a2 = 12.8, b2 = 13.1;
    swap(a2, b2); // 显式具体化

    return 0;
}

// 通用模板定义
template <typename T>
void swap(T &a, T &b) {
    using std::cout;

    T temp = a;
    a = b;
    b = temp;

    cout << "a = " << a << " b = " << b << '\n';
}

// 显式具体化模板定义
template <> void swap<double>(double &a, double &b) {
    using std::cout;

    double temp = a;
    a = b;
    b = temp;

    cout << "b = " << b << " a = " << a << '\n';
}

```

（2）其次搞清楚类模板的具体化：

```c++
// 1.隐式实例化：根据需要声明对象时传递类型参数来实例化一个类模板

// 2.显式实例化：未进行声明对象时传递类型参数来实例化一个类模板
template class ClassType<AnyType>;

// 3.显式具体化：对于特定类型参数，类的定义需要不同时，需要定义一个不同的类模板
template <> class ClassType<Antype> {
    ...
};

// 4.部分具体化：如果一个类模板有多个类型参数，可以使得某些类型具体化
// 通用模板
template <typename T1, typename T2>
class Pair {...};
// 对于typename T2进行具体化
// 注意具体化是需要提供定义的
// 关键字template后面的<>声明的是没有被具体化的类型参数
// 如果所有类型都被指定，那么<>里面是空的，就成了显式具体化
template <Typename T1>
class Pair<T1, int> {...};
```

（3）普通类的中的静态成员初始化：在类中声明，在类外初始化。一般初始化放在方法文件中，因为头文件可能被包含很多次，导致多个初始化语句副本。

```c++
class StringBad {
public:
    static int nums;
};

// 静态成员初始化
// 不需要关键字
int StringBad::nums = 0; 
```

（4）模板类中的静态成员数据初始化：

```c++
// 两种数据成员
// 第一种不依赖于模板的类型参数
// 第二种依赖于模板的类型参数

template <typename T> 
class TestTemStatic
{
    public:
    static int knownTypeVar;
    static T unKnownTypeVar;
};

// 第一种类型的初始化方式
// 1.范化定义,定义num时不需要知道T的类型
template <typename T> int TestTemStatic<T>::knownTypeVar=50;
// 2.具化定义，给出T类型,同时定义num，T可以是其他任意特定类型。
template <> int TestTemStatic<int/* any other type */>::knownTypeVar=2;

// 第二种类型的初始化方式
// 具化定义
template <> float TestTemStatic<float>::unKnownTypeVar=4.0f;
```

MinGW支持这个特性。

#### 2.模板类的部分具体化

常量：__STL_CLASS_PARITIAL_SPECIALIZATION

```c++
#include <iostream>
using namespace std;

// 一般化模板
template <typename I, typename O>
struct Test {
    Test() {
        cout << "I, 0" << endl;
    }
};

// 部分特殊化设计
// 这里的部分具体化需要这样理解：
// 关键字template后面的<>里面是没有被具体化的类型
// 后面的<>里面中第二个类型参数使用的是第一个没有被具体化的类型
//（按照过去的认知具体化应该是某些已知类型，比如int、char等，但是实际上是可以使用未被具体化的类型的）
template <typename T>
struct Test<T*, T*> {
    Test() {
        cout << "T*, T*" << endl;
    }
};

// 部分特殊化设计
// 这里的部分具体化理解与上面一个例子类似
// 注意const与否也表示一种重载
template <typename T>
struct Test<const T*, T*> {
    Test() {
        cout << "const T*, T*" << endl;
    }
};

int main(void) {
    // 初始化三种模板并且调用默认构造函数
    Test<int, char> obj1;
    Test<int *, int *>obj2;
    Test<const int *, int *>obj3;

    return 0;
}
```

MinGW支持这个特性。

#### 函数模板的重载

常量：__STL_FUNCTION_TMPL_PARTIAL_ORDER

虽然未文件中声明这个常量的意义与partial specialization of function templates相同，但是实际上并不相同。前者的意义如下表示，后者参考C++语法书籍。

```c++
// 节选自stl_vector.h内容
class alloc {};

template <class T, class Alloc = alloc>
class vector {
public:
    void swap(vector<T, Alloc> &) {
        cout << "swap()" << endl;
    }
};

#ifdef __STL_FUNCTION_TMPL_PARTIAL_ORDER
template <class T, class Alloc>
inline void swap(vector<T, Alloc> &x, vector<T, Alloc> &y) {
    x.swap(y);
}
#endif
// 上面这一段是节选自stl_vecor.h，表示根据常量进行条件编译
// 如果定义了这个常量，说明支持这个语法
// 所以会调用这个模板函数，模板函数就会调用类方法。从而打印出swap
// 但是如果没有定义这个常量，就不会生成这个模板

int main(void) {
    vector<int> x, y;
    swap(x, y); // 没有输出，说明调用的不是重载的模板函数

    return 0;
}

```

（我使用的是MinGW8.1.0的64位编译器，这里没有定义这个常量。因此实际上调用的是另一个模板函数。）

#### 显式函数模板参数

常量：__STL_EXPLICIT_FUNCTION_TMPL_ARGS

整个SGI STL没有用到这一个定义

#### 模板成员

常量：__STLMEMBER_TEMPLATES

```c++
#include <iostream>
using namespace std;

// 声明一个空类
class alloc {};

// 声明一个模板类
template <typename T, typename Alloc = alloc>
class vector {
public:
    typedef T value_type;
    typedef value_type* iterator;

    // 类模板中的模板成员函数
    template <typename I>
    void insert(iterator position, I first, I last) {
        cout << "insert()" << endl;
    }
};

int main(void) {
    int ia[5] = {1, 2, 3, 4, 5};

    vector<int> x;
    vector<int>::iterator it;
    x.insert(it, ia, ia + 5); // insert()

    return 0;
}
```

MinGW支持这个特性。

#### 模板参数可否根据前一个模板参数设定默认值

常量：__STL_LIMITED_DEFAULT_TEMPLATES

```c++
#include <iostream>
using namespace std;

// 声明一个空类
class alloc {};

// 声明一个模板类
template <typename T, typename Alloc = alloc, size_t BufSiz = 0>
class deque {
public:
    deque() {
        cout << "deque" << endl;
    }
};

// 根据前一个参数值T，设定下一个参数的默认值
template <typename T, typename Sequence = deque<T>>
class stack {
public:
    stack() {
        cout << "stack" << endl;
    }
private:
    Sequence c;
};


int main(void) {
    stack<int> x; // 输出 deque 和 stack

    return 0;
}
```

MinGW支持这个特性

#### 类模板是否可以拥有non-type参数

常量：__STL_NON_TYPE_TMPL_PARAM_BUG

MinGW支持类模板的非类型参数。





后面还介绍了一些常量，均表示是否支持的一些C++语法功能，但是我采用的版本并不存在这些常量（应该大部分都支持），所以暂时先不看了。

### 临时对象的产生与运用

临时对象又称为匿名对象（比如按照传递时就会产生一个临时对象）。

刻意创造一个临时对象的方法是：在类型后加一对小括号，可以指定初始值。例如 Shape(3, 5)。相当于调用类型的构造函数到那时不指定名称。

STL将这个技巧应用于仿函数和算法的搭配上。

```c++
#include <iostream>
#include <algorithm>
#include <vector>
using namespace std;

// 定义一个仿函数
template <typename T>
class Print {
public:
    // 重载()运算符
    void operator()(const T &elem) {
        cout << elem << ' ';
    }
};


int main(void) {
    using std::for_each; // 算法
    using std::vector;
    int arr[5] = {1, 2, 3, 4, 5};
    vector<int> iv(arr, arr + 5);

    // print<int>()是一个临时对象
    // 调用重载运算符()的方法
    for_each(iv.begin(), iv.end(), Print<int>());

    return 0;
}
```

### 静态常量整数成员在类内直接初始化

如果类中包含const static integtal 的成员数据，那么可以直接给与初始值。注意这里必须是const；以及integral指的是所有的整数类别，包括int、short、char等。

### 自增、自减、解引用操作符

```c++
class Test {
public:
    // 前置自增或者自减的重载方式
    Test& operator++() {
        ++(this->i_);
        return *this;
    }
    
    // 后置自增或者自减的重载方式
    Test operator++(int) {
        Test temp = *this;
        ++(*this); // 调用前者自增运算符
        return temp;
    }
    
    // 解引用运算符
    int& operator*() const {
        return (int&)i_; // 上述转换操作明确告诉编译器需要将const int转为non-const lvalue
    }
private:
    int i_;
};
```

### 前闭后开区间表示法

任何STL算法都需要获得一对迭代器所标示的区间，用以表示操作范围。这区间是前闭后开的，即后面的迭代器指向的是最后一个元素的下一个位置。

<img src="1-9-1.png" alt="image-20231008090304336" style="zoom:67%;" />

### 函数调用操作符operator()

<img src="1-9-2.png" alt="image-20231008090510347" style="zoom:67%;" />

这一整组操作可以是以函数的形式实现，通过**函数指针**将函数作为参数传递。

<img src="1-9-3.png" alt="image-20231008090707579" style="zoom:67%;" />

<img src="1-9-4.png" alt="image-20231008090733191" style="zoom:67%;" />





