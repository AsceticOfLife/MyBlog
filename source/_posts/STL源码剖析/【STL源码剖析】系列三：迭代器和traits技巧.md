---
title: 【STL源码剖析】系列三：迭代器和traits技巧
categories:
  - STL源码剖析
tags:
  - STL
  - 读书笔记
typora-root-url: 【STL源码剖析】系列三：迭代器和traits技巧
abbrlink: 1569660478
date: 2023-10-19 09:31:20
---

# 前言

迭代器是一种抽象概念，在设计模式中，迭代器对外提供了一种遍历一个聚合对象（可以简单理解为一堆相同类型的对象的集合体）各个元素的方式，但是又避免了暴露内部结构。

本篇主要内容是：
1.以一个容器的迭代器例子说明迭代器模式的典型实现；
2.介绍Traits编程技巧的实现，说明如何提取出与迭代器相关的五种类型；
![](3-4-7.png)

3.SGI在iterator_traits的基础上额外提供了对于每种类型的__type_traits，以表示每种类型的特性。

<!-- more -->

# 迭代器与tiaits技巧

设计模式中对于迭代器模式的描述：提供一种方法能够顺序访问一个聚合对象中的各个元素，而又不暴露内部表示。

## 迭代器设计思维

STL的中心思想是将算法与容器分开，使其泛型化。可以分别使用函数模板和类模板实现。

迭代器充当二者之间的胶着剂，算法通过迭代器访问容器中的所有元素，这样就不需要了解容器内部表示。

以find算法为例，如果想要操作不同的容器，比如list和queue，在不使用迭代器的情况下，必须知道如何访问list和queue的所有元素的方法，但是list和queue由于结构不同，其遍历所有元素的方法有可能不同，比如list可以采用链式存储结构实现，也可以采用顺序存储结构实现，queue同样也有自己的结构实现，那么遍历方法肯定是不相同的。于是对于两种不同的容器就存在两种find算法。

如果使用迭代器，那么迭代器向提供统一的一种遍历容器的所有元素的方法，例如将接口定义为itrator++的这种表示，虽然不同的容器迭代器的接口实现不同（比如对于链式存储结构iterator++实现为next方法，对于顺序存储结构实现为指针加法），但是对于高层的算法find来说，只要不断使用iterator++就可以实现所有元素的遍历。

## 迭代器

迭代器是一种抽象概念，可以理解为一系列标准的集合（比如说苹果是一个概念，苹果的简单劣质标准就是：红的、圆的、水果，那么主要符合这三个标准就认为是苹果）。

最重要的标准就是通过迭代器能够访问指向的元素，也就是解引用（operator*）和成员访问（operator->）。

迭代器类似于智能指针的概念。因为智能指针能够满足迭代器的标准要求。

下面以list容器为例设计一个迭代器：

首先是list和结点结构的定义：

<img src="3-2-1.png" alt="image-20231016095219326" style="zoom:67%;" />

list容器的迭代器定义为：

<img src="3-2-2.png" alt="image-20231016095447808" style="zoom: 80%;" />

这里使用的例子如下：

<img src="3-2-3.png" alt="image-20231016095604833" style="zoom: 67%;" />

<img src="3-2-4.png" alt="image-20231016095634437" style="zoom:67%;" />

<img src="3-2-5.png" alt="image-20231016095722792" style="zoom:67%;" />

上面就可以看出迭代器向上层隐藏了不同容器的实现细节，但是迭代器本身一定要非常清楚容器的实现细节，所以一般每一种STL容器都有专属的迭代器。

## 迭代器相关的类型

算法中有可能需要用到**迭代器相关的类型**（包括所指向对象的类型、所指向对象的指针和引用类型以及const形式的指针和引用类型、迭代器类型）。例如需要在算法中声明一个临时变量，这个变量的类型与迭代器指向的对象类型一致。

解决方法是利用**编译器对于模板函数的参数类型推导机制**。

<img src="3-3-1.png" alt="image-20231016151209397" style="zoom:67%;" />

比如这里想要使用迭代器所指向对象的类型，但是如果直接将迭代器对象作为参数传递给算法，那么算法只能推导出迭代器的类型。

这里将接口和实现分开，在接口中，接受一个迭代器对象，编译器推导出迭代器类型；在实现中，接受一个迭代器所指向对象，根据传入的所指对象（即迭代器解引用的对象）编译器推导出迭代器所指之物的类型。

## Traits编程技巧

利用编译器推导函数模板的参数类型并非是全能的。因为本质上是**根据传入的实参**来推断形参类型，从而能够使用形参的类型。

但是如果算法的返回值需要迭代器相关的类型，那么就无法推断。

解决方法是在迭代器类中声明一个**嵌套类型声明**：

<img src="3-4-1.png" alt="3-4-1" style="zoom:67%;" />

算法根据传入的迭代器类型推断出迭代器的class type，根据class中的嵌套声明就可以确定返回值类型。

但是这种方法存在限制，即要求迭代器必须是class type，但是并不是所有的迭代器都是class，比如原生指针就是一种迭代器，但是没有办法在原生指针中添加声明。

解决方法是利用类模板的偏特化——针对任何模板参数更进一步的条件限制所设计出来的一个特化版本。例如：

<img src="3-4-2.png" alt="3-4-2" style="zoom:67%;" />

所以，即使原生指针没有办法声明嵌套类型，也可以通过设计特化版本的迭代器，来处理算法的模板参数（迭代器参数）为指针的情况。

结合上面两种方式，设计一个类模板专门用于萃取迭代器的特性：

<img src="3-4-3.png" alt="image-20231016153210490" style="zoom:67%;" />

这个类模板接受一个迭代器类型，必须是class type，于是就可以访问迭代器类中声明的嵌套类型value_type。

如果是原生指针的化，就需要一个偏特化版本：

<img src="3-4-4.png" alt="image-20231016153427788" style="zoom:67%;" />

这个特化版本接接受一个原生指针类型，并将原生指针所指类型typedef为value_type。

提供以上两个萃取类模板，于是算法就可以根据传入的迭代器对象传给这个萃取模板，萃取模板根据迭代器是class还是原生指针来选择以不同的方式返回所指对象的类型。

<img src="3-4-5.png" alt="image-20231016153723392" style="zoom:67%;" />

最后需要补充的是，针对“指向常熟对象的指针”，iterator_traits<const int *>::value_type得到的类型是一个const int，但是这并不是我们所期望的，我们希望利用这种机制声明一个临时变量，这个临时变量的类型与迭代器的value_type相同。但是如果这个临时变量不可赋值，那么没有什么意义。所以**当迭代器是一个pointer-to-const类型，应该令其value_type为一个non-const类型。**这就需要另一个特化版本：

<img src="3-4-6.png" alt="image-20231016154115640" style="zoom:67%;" />

### Traits技巧总结

**iterator_traits模板类作用：**

<img src="3-4-7.png" alt="image-20231016154251504" style="zoom:67%;" />



**与迭代器相关的类型有五种：**
如果想要自定义的容器与STL的算法兼容，那么自定义容器的迭代器一定要定义五种相应的类型。

<img src="3-4-8.png" alt="image-20231016154427219" style="zoom:67%;" />

### 迭代器相关类型：value type

指的是迭代器所指对象的类型。

实现方法为：

```c++
// 1.在迭代器类中声明value_type类型
template <typename T>
class iterator {
public:
    typedef T value_type; // 声明指向的对象的类型是value_type
};

// 2.在iterator_traits 类模板中嵌套声明value_type类型
template <typename I>
struct iterator_traits {
    typedef typename I::value_type value_type;
};

// 偏特化原生指针类型
template <typename T>
struct iterator_traits<T *> {
	typedef T value_type;  
};

// 偏特化指向常熟对象的指针（pointer-to-const）
template <typename T>
struct iterator_traits<const T *> {
	typedef T value_type;  
};

// 3.使用时根据迭代器类型得到value_type
// 以count为例
// 根据传入的迭代器对象获取迭代器类型I
// 将迭代器类型I传入iterator_traits模板，得到value_type
template <typename I, typename T>
typename iterator_traits<T>::difference_type
count(I first, I last, const T &value) {
    typename iterator::traits<I>::difference__type n = 0;
    for ( ; first != last; ++first) ++n;
    
    return n;
}
```



### 迭代器相关类型：difference type

表示两个迭代器之间的距离。也可以用来表示一个容器的最大容量。

实现方法：

```c++
// 1.在迭代器class type中声明difference type
template <typename T>
class iterator_type {
public:
    typedef ptrdiff_t difference_type; // 假设迭代器的difference_type底层类型就是ptr
};

// 2.在iterator_traits类模板中声明difference_type类型
template <typename I>
struct iterator_traits {
	typedef I::difference_type difference_type;
};

// 原生指针特化版本
template <typename T>
struct iterator_traits<T *> {
	typedef ptrdiff_t difference_type;  
};

// 指向常量对象的指针特化版本
template <typename T>
struct iterator_traits<const T *> {
	typedef ptrdiff_t difference_type; 
};

// 3.使用时先对推断出迭代器类型I，再通过将I传递给iterator_traits类模板提取类型
typename iterator_traits<I>::difference_type
```

### 迭代器相关类型：reference type

迭代器可以分为两种：不允许改变所指对象内容，称为constant iterator，比如const int *pic；允许改变所指对象内容，称为mutable iterator，比如int *pi。

当对于一个mutable iterator进行解引用操作(operator*)时，获得的应该是是一个左值而不是一个右值，因为左值允许赋值操作，右值不允许赋值操作。

<img src="3-4-9.png" alt="image-20231016213859704" style="zoom:67%;" />

在C++中，函数如果想要传回左值，都是以引用的方式返回类型。所以当p是一个mutable iterator时，如果其value_type是T，那么*p的类型不应该是T，应该是T &；如果p是一个constant iterator，其value_type是T，那么\*p的类型不应该是const T，应该是const T &。

*p的类型就是reference type。

### 迭代器相关类型：pointer type

reference type类型指的是传回一个左值类型，令它代表p所指之物；pointer type类型指的是传回一个左值类型，令它代表p所指之物的地址。

实现方法：

```c++
// 1.将reference type和pointer type加入迭代器中
template <typename T>
class iterator {
public:
    typedef T & reference;
    typedef T * pointer;
};

// 2.在iterator traits类模板中添加声明
template <typename I>
struct iterator_traits {
	typedef typename I::reference reference;
    typedef typename I::pointer pointer;
};

// 针对原生指针的偏特化版本
template <typename T>
struct iterator_traits<T *> {
	typedef T & reference;
    typedef T * pointer;
};

// 针对指向常量的指针的偏特化版本
template <typename T>
struct iterator_traits<const T *> {
	typedef T & reference;
    typedef T * pointer;
};

// 3.在算法中使用时根据传入的迭代器对象确定迭代器类型I
// 在将I传给iterator_traits类模板生成合适的类定义并访问其中声明的reference和pointer
template <typename I>
void func(I iterator) {
    iterator_traits<I>::pointer 
    iterator_traits<I>::reference 
}
```

### 迭代器相关类型：iterator_type

迭代器一共有五种类型：

- 输入迭代器：针对于程序来说从容器中输入，只读，只能递增，单通行（不保证第一次和第二次遍历顺序相同，递增之后不保证前一个迭代器的值还可解引用）
- 输出迭代器：针对于程序来说向容器中输出，只写，只能递增，单通行。
- 正向迭代器：可读可写，只能递增，多通行（总是按照相同的顺序遍历，并且递增之后前一个迭代器仍然可以解引用）
- 双向迭代器：可以双向移动，支持正向迭代器的所有特性。
- 随机访问迭代器：前四种只提供部分算术能力（前三种只支持operator++，第四种支持operator++和operator--），第五种支持p + n、p - n、p1 - p2，p1 < p2等。

<img src="3-4-10.png" alt="image-20231017084549299" style="zoom:67%;" />

箭头表示概念与强化的关系。

为什么需要迭代器类型的概念：因为针对不同类型的迭代器，算法可能会有不同的效率。比如advance算法，接受一个迭代器参数和一个整数参数，功能是将迭代器p + n。如果是输入迭代器，只能递增n次，算法复杂度为O(n)；如果是双向迭代器，当n大于等于0时，递增n次，当n小于0时，递减n次，时间复杂度时O(n)；如果是随机访问迭代器，直接p + n，时间复杂度O(1)。
因此**算法中有时需要判断迭代器的类型**，对于不同的迭代器采取不同的处理方式。

想要使用迭代器类型，首先需要定义迭代器类型：

```c++
// 1.定义迭代器类型
struct input_iterator_tag {};
struct output_iterator_tag {};
struct forward_iterator_tag : public input_iterator_tag {};
struct bidirectional_iterator_tag : public forward_iterator_tag {};
struct random_access_iterator_tag : public bidirectional_iterator_tag {};
```

然后在迭代器类中声明迭代器类型，比如在一个迭代器中声明其类型为上述其中一种：

```c++
template <typename T>
class iteartor {
public:
    typedef input_iterator_tag iterator_category; // 这个迭代器类型就是输入迭代器
};
```

接着在萃取类iterator traits模板中根据传入的迭代器class type提取迭代器类型：

```c++
template <typename I>
struct iterator_traits {
    typedef typename I::iterator_category iterator_category;
}

// 针对原生指针偏特化的版本
template <class T>
struct iterator_traits<T *> {
    typedef random_access_iterator_tag iterator_category; // 原生指针是随机访问迭代器类型
}

// 针对指向常量的指针的特化版本
template <class T>
struct iterator_traits<const T *> {
    typedef random_access_iterator_tag iterator_category; // 原生指针是随机访问迭代器类型
}
```

最后如果需要获取迭代器类型，需要将算法的接口和实现分开，在接口中，接受迭代器对象并推断出迭代器的class type，根据这个参数实例化萃取类型模板并得到iterator_category；在实现中的参数中多接受一个迭代器类型参数，根据传入的实参推断类型：

```c++
// 算法对外接口
template <typename InputIterator, typename Distance>
inline void advance(InputIterator &i, Distance n) {
    // 第三个实参是获取迭代器内部声明的迭代器类型而创建的临时对象
    // 根据这个实参对象的类型选择调用哪个具体实现
    __advance(i, n, itrator_traits<InputIterator>::iterator_category());
}

// 算法真正实现
// 对于输入迭代器类型的实现
template <typename InputIterator, typename Distance>
inline void __advance(InputIterator &i, Distance n, input_iterator_tag) {
    // 单层递增
    while (n--) ++i;
}

// 对于正向迭代器类型的实现
template <typename InputIterator, typename Distance>
inline void __advance(InputIterator &i, Distance n, forward_iterator_tag) {
    // 单纯传递调用
    advance(i, n, input_iterator_tag());
}

// 对于双向迭代器类型的实现
template <typename InputIterator, typename Distance>
inline void __advance(InputIterator &i, Distance n, bidirectional_iterator_tag) {
    if (n >= 0) while (n--) ++i;
    else while (n++) --i;
}

// 对于随机访问迭代器类型的实现
template <typename InputIterator, typename Distance>
inline void __advance(InputIterator &i, Distance n, random_access_iterator_tag) {
    i += n;
}
```

上面的每一个__advance作为算法实际实现，其第三个参数只是为了激活编译器的函数重载机制，在算法中并不会使用，所有不需要形参名。

另外，STL标准中，**要以算法能够接受的最低阶的迭代器类型来为迭代器类型参数命名**。

为什么以在迭代器类型存在public继承：好处是可以消除传递调用版本的实现。例如上面例子中对于输入迭代器和正向迭代器的实现是相同的，那么可以只定义输入迭代器的实现，当参数为输入迭代器类型和正向迭代器类型时，会将正向迭代器类型隐式转换为其基类——输入迭代器类型。

## STL提供迭代器基类

为了符合规范，任何迭代器都应该提供五个嵌套相关类型声明，以方便traits进行萃取。

如果不想自己定义，STL定义了一个迭代器的基类iterator:

```c++
template <typename Category, typename T, typename Distance = ptrdiff_t, typename Pointer = T *, typename Reference = T &>
struct iterator {
	typedef Category	iterator_category;
    typedef T			value_type;
    typedef Distance	difference_type;
    typedef Pointer		pointer;
    typedef Reference	reference;
};
```

在这个标准基类中不存在任何成员，只是对于相关类型进行定义。后面三个类型都有默认参数，前两参数需要在定义自己的迭代器时声明，比如声明在3.2节自定义的ListIter：

```c++
template <typename Item>
stuct ListIter : public std::iterator<std::forward_iterator_tag, Item> {
    ...
};
```



**总结：**

<img src="3-5-1.png" alt="image-20231017092921462" style="zoom:67%;" />

## 对于源码的汇总

前面描述迭代器类型时分别描述如何实现，这里将所有类型汇总，并提供一些类型同名函数来返回类型。

详细看书。

## SGI STL的另一个技巧：__type_traits

iterator_traits主要是萃取迭代器的特性，而__type_traits主要是为了萃取类型的特性。

根据iterator_traits的特性，希望采取如下形式使用类型T的特性：

<img src="3-7-1.png" alt="image-20231017101142590" style="zoom:67%;" />

希望得到的类型应该是真或者假。但是如果希望利用编译器进行参数类型的推导，那么就必须是一个对象。所以需要定义真类型和假类型：

```c++
struct __true_type {};
struct __false_type {};
```

为了定义上面五个特性，__type_traits类内必须定义一些typedef，SGI 在默认情况下定义为假类型：

<img src="3-7-2.png" alt="image-20231017101517167" style="zoom:67%;" />

如果想要对于某种类型T定义某些特性为真类型，就必须提供上面模板的特化版本，比如对于内置类型chary提供特化（显式实例化）版本：

```c++
template <> struct __type_traits<char> {
	typedef __true_type		has_trivial_default_constructor;
    typedef __true_type 	has_trivial_copy_constructor;
    typedef __true_type		has_trivial_assimnment_operator;
    typedef __true_type 	has_trivial_destructor;
    typedef __true_type		is_POD_type;
};
```

类似的，如果想要针对自定义的类型，比如Test，萃取特性，需要提供一个Test特化版本的__type_traits：

```c++
template <> struct __type_traits<Test> {
    typedef __true_type		has_trivial_default_constructor;
    typedef __true_type 	has_trivial_copy_constructor;
    typedef __true_type		has_trivial_assimnment_operator;
    typedef __true_type 	has_trivial_destructor;
    typedef __true_type		is_POD_type;
}
```

使用时就可以根据类型获取特性类型，根据特性类型选择不同的处理方式。

参考2.3.2节的uninitialized_fill_n中，先使用value_type萃取出迭代器的value_type，也就是所指对象的类型；然后将value_type传递给__type_traits模板，得到实例化版本的类型声明：

<img src="3-7-3.png" alt="image-20231017102941786" style="zoom:67%;" />

<img src="3-7-4.png" alt="image-20231017103049909" style="zoom:67%;" />

之后就可以根据type_traits的特性选择不同的处理方式：

<img src="3-7-5.png" alt="image-20231017103316355" style="zoom:67%;" />
