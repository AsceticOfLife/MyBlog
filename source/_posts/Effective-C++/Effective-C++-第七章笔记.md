---
title: 《Effective C++》第七章笔记
date: 2023-10-15 09:52:14
description: 《Effective C++》第七章读书笔记，本章主要是介绍在使用模板编程时的一些注意事项。
categories:
- 《Effective C++》
tags:
- 读书笔记
---

# 模板与泛型编程

## 条款41：了解隐式接口和编译期多态

class和template都支持接口和多态：

- **显式接口和运行期多态：**<u>显式接口</u>指的是当使用一个对象时，这个对象可以调用所有所属类型的接口（类中定义的public方法），可以直接找到源代码。显式接口通常由函数的声明确定，即返回值类型、函数名称、参数类型、常量性。<u>运行期多态</u>指的是当一个类中有virtual函数，对象在调用这些函数时会在运行期间确定调用哪个函数。
- **隐式接口和编译期多态：**在函数模板中，对于类型为T的对象来说，需要根据施加于对象的操作来猜测T类型具有哪些接口（详见下面的例子）。一个T类型的对象调用任何函数都用可能造成template实例化，这些实例化发生在编译期，编译器根据类型生成函数供调用。运行期多态和编译期多态就像“哪一个重载函数被调用”（发生在编译期）和“哪一个virtual函数应该被绑定”（发生在运行期）的关系。

```c++
// 隐式接口不依赖于函数声明
template <typename T>
void doSomething(T& w) {
	if (w.size() > 10 && w != someNastyWidget)
	...
}
```

看似w属于的T类型应该提供一个size函数且size函数返回值为int类型（10的类型）、支持operator!=运算符并比两个类型。

但是实际上并不一定。比如w.size()确实需要一个成员函数size，但是这个可以通过继承获得；且返回值不一定非得是int，只要是能够和10进行比较的类型对象都可以，或者返回值类型可以隐式转换为一个能够和10进行比较的类型也行。

类型T也不一定非要支持operator!=运算符，只要T能够转换为运算符需要的类型Y也行。

总结：隐式接口基于有效表达式，编译期多态基于模板实例化和函数重载解析。

## 条款42：typename双重含义

- **在template中声明参数类型时**，前缀关键字class与typename完全相同，但是还是推荐使用typename，因为typename能够表示类型并非是一个class。
- **表示嵌套从属类型名称时前面需要加上typename**，另外不能在基类列表中（即定义一个派生类时冒号:后面的基类列表）和成员初始化列表中以typename作为基类的修饰符。<u>嵌套从属类型名称：</u>从属名称指的是在一个模板中，依赖于template模板参数T的名称；嵌套从属名称指的是从属名称在class类内呈现嵌套状；嵌套从属类型名称指的是涉及到类型。说白了就是对于“T::类型”编译器不知道是T类中的成员还是T类中的类型，因此需要typename告诉编译器这是一个类型。

## 条款43：处理模板化基类中的名称

在模板化基类的派生类中，不能直接使用模板基类的成员函数。
因为编译器假设模板基类有可能被具体化（特例化），具体化的基类可能与基类的行为不完全一致。
也就是C++编译器在派生类定义时不去观察模板基类的定义，从而导致找不到模板基类的成员函数。

有三种方式阻止C++不去检查模板基类的定义（意思就是让C++编译器去看模板基类到底有没有成员函数定义，当然如果没有肯定会报错）

- 1.在派生类调用基类方法时使用this指针；
- 2.在派生类中使用using声明式，即告诉编译器去基类中查找；
- 3.在派生类中使用“基类::函数”的方式调用基类方法，但是这种行为相当于关闭了virtual函数行为。

## 条款44：与参数无关的代码抽离template

template会由编译器生成具体的代码，即根据参数类型产生多个class和多个函数。

代码膨胀：指的是二进制代码中含有重复的代码、数据。

- 为了避免由于**非类型参数**造成代码膨胀，需要将template代码与非类型模板参数分离：以函数参数传递非类型参数或者以class成员变量替换template参数。
- 对于因模板**类型参数**造成的代码膨胀，比如如果int和long具有相同的底层大小，那么vector\<int>和vector\<long>的代码是重复的。做法是让让完全相同的二进制表述的类型共享实现码。比如指针类型都是一样大小的，不管是int*还是char\*，所以如果一个成员函数操作某个强型指针（具有类型的指针），应该令它们调用另一个操作无类型指针(void\*)的函数。

注意：抽离非类型参数需要考虑实际效率：

1.未抽离非类型参数的模板会根据类型参数的不同而生成不同的二进制代码，这些代码由于将非类型参数作为编译期常量，因此有可能被生成指令中，这也算一种优化；

抽离非类型参数的模板会缩小二进制代码的大小，因此工作集（代码运行时的页数）会小一些，这样会使得高速缓冲区中的指令比较集中，因此程序执行更快速。

2.在更改了非类型参数之后，需要考虑使用非类型参数的部分与模板之间的关系。比如如果抽离类模板中的非类型参数，需要在派生模板类中调用基类模板（非类型参数也在基类模板中声明），那么就要考虑两个类之间如何共享数据，以及由于共享数据带来的内存消耗。

## 条款45：使用成员函数模板接受所有兼容类型

（同一个模板实例化的两个函数或者类之间没有什么关系，比如根据一个继承体系中的基类和派生类实例化的模板之间没有继承关系，因此也就无法执行派生类向基类隐式转换的功能，只能进行显式转换。）

在类class之间（没有继承关系的两个类，也就是上面的类模板实例化后的关系）进行类型转换时，即将其他类转换为该类，是通过接受一个参数的构造函数进行的（可以隐式也可以explicit显式）。如果想要一个类A接受另一个类B转换，就必须声明一个参数类型为类B的构造函数。如果又有一个类C，那么就需要更改类。

因此有了成员函数模板：即**使用构造函数模板来为不同的其它类提供转换为类A的方法**。

```c++
template <typename T>
class shared_ptr {
public:
    template <typename Y>
    explicit shared_ptr(Y *p); // 兼容类型的内置指针
    
    template <typename Y>
    shared_ptr(shared_ptr<Y> const &r); // 兼容任何其它类型的智能指针。这里没有声明为explicit，表示可以进行隐式转换
    
};
```

<u>需要注意的是：</u>由于存在不能进行类型转换的类，因此需要对成员函数模板支持的类型转换进行筛选，筛选的方式就是通过在类型转换函数（包括构造函数、copy构造函数、copy assignment运算符）中调用内置类型初始化或者赋值（因为类class就是由内置类型组成的），如果不允许转换就无法通过编译。比如上面的“泛化的复制构造函数”，允许基类向派生类进行转换，所以需要进行限制：

```c++
template <typename T>
class SmartPtr {
public:
    // 以另一个类型持有的指针初始化自身指针
    // 这样如果内置类型指针不允许转换（比如将基类指针赋给派生类指针）
    // 那么模板类也不允许转换
    template <typename U>
    SmartPtr(const SmartPtr<U> &other) : heldPtr(other.get()) {} 
    T * get() const { return hedlPtr; } // get方法返回持有的内置指针
private:
    T &heldPtr; // 持有的内置指针
};
```

<u>另外：</u>成员模板函数如果用于泛化copy构造和泛化assignment操作，那么仍需要提供正常的copy构造函数和copy assignment运算符。

## 条款46：需要类型转换时请为模板定义非成员函数

用处：

考虑需要在一个模板类中执行该类与其它类型的运算（即其它类型需要向模板类进行类型转换）

```c++
template <typename T>
class Rational {
	...
};

// 定义一个非成员函数是为了满足交换律
// 参考条款24
template <typename T>
const Rational<T> operator*(const Rational<T> &lhs, const Rational<T> &rhs) {
	...
}

// 假如发生了如下调用
Rational<int> a;
int b;
Rational c = a + b; // 编译失败
```

原因分析：编译器对于表达式“a + b"需要调用函数模板（function template），**调用函数模板需要对参数类型进行推导（推导出参数类型才能生成函数定义）：**根据a确定第一参数类型为Rational<int\>，但是对于int类型b来说，不是一个Rational\<T>类，因此无法推断出T的类型。<u>这是因为：</u>template**参数推导过程**不会涉及隐式类型转换！这样的转换在普通函数调用中确实存在，但是在调用函数前，必须先知道函数的存在。而为了生成这样的函数，函数模板必须推导出参数类型才能做到。

解决方案：将这个函数声明为类模板的友元。因为类模板并不依赖于模板的参数推导（因为实例化一个类对象时，必须为其显式传递参数T，比如上面代码中显式声明参数类型T为int），所以编译器能够在为类模板实例化时得到T。

```c++
template <typename T>
class Rational {
	...
	friend const Rational<T> operator*(const Rational<T> &lhs, const Rational<T> &rhs); // 声明为类模板的友元
    // 另一种等价式声明
    // friend const Rational operator*(const Rational &lhs, const Rational &rhs);
    
};

// 在类外定义
template <typename T>
const Rational<T> operator*(const Rational<T> &lhs, const Rational<T> &rhs) {
	...
}

// 发生如下调用
Rational<int> a;
int b;
Rational c = a + b; // 编译成功，链接失败

```

如果非成员函数operator*被声明为类模板的友元，当a被实例化时，类模板生成Ration\<int>的定义，同时friend函数也被声明出来，因而存在了一个被声明的函数，对于这个函数编译器会使用隐式类型转换，把int类型的b转换为Ration\<int>对象。

PS：在一个类模板内，template名称（类名）可以用来表示”template及其参数“的简略表达方式，即Rational等价于Rational\<T>

但是为什么不能链接？因为这个友元只是有了一个声明，并没有被定义出来。没有定义式，自然无法链接。
**解决方案：**直接把定义式写在类模板定义里

```c++
template <typename T>
class Rational {
	...
	// 定义为类模板的友元
	friend const Rational<T> operator*(const Rational<t> &lhs, const Rational<T> &rhs) {
		...
	}
};

Rational<int> a;
int b;
Rational c = a + b; // 编译成功，链接成功
```

更进一步，由于定义在类中的函数会被inline，这里的operator*函数比较简单inline问题不大，但是如果有一个非常复杂的函数，再inline就不合理。因此往往需要这个inline函数调用一个辅助函数。（当然一个模板函数调用的函数一般也是模板函数比较方便，如果是特定参数类型的函数，就需要声明很多比较麻烦）

总结：当编写一个类模板时，如果一个函数中所有参数都有可能进行隐式转换，那么将这个函数定义为类模板内部的friend函数。

## 条款47：使用traits class表现类型信息

（主要是使用traits class表现一个类在编译期的类型信息，RTTI技术用于表示运行时类型信息）

**想要表现一个类的类型信息，为什么不能在类中嵌套信息用于向外表示类型信息呢？**

因为traits这项技术不仅要求对于用户自定义类型可用，对于内置类型如指针等也要适用，而指针等内置类型是没办法嵌套信息表示类型信息的。

因为标准方法是将这些信息放入一个模板以及一个或多个特化版本中。

**如何设计并实现一个traits class：**

- 确认若干希望将来能够获取的类型信息，比如对于迭代器来说，希望获取其分类（category）；
- 为该信息选择一个名称（例如iterator_category）；
- 提供一个template和一组特化版本，内含希望的类型信息。

（下面以迭代器为例子，说明如何在**编译期间获取迭代器的类型**。

C++ STL中的迭代器一共有5种基础类型，分别是输入、输出、正向、双向、随机存取迭代器，其实现为：

```c++
// tag struct
struct input_iterator_tag {};
struct output_iterator_tag {};
struct forward_iterator_tag : public input_iterator_tag {};
struct bidirectional_iterator_tag : public forward_iterator_tag {};
struct random_access_iterator_tag : public bidirectional_iterator_tag {};
```

)

希望实现一个加法操作advance，对于不同的迭代器能够进行不同的操作，比如对于随机存取迭代器执行+=操作，对于其它迭代器++n次

```c++
template <typename IterT, typename DistT>
void advance(IterT &iter, DistT d) {
    // 先判断是否为一个随机存取迭代器，如果是随机存取迭代器就可以直接执行+=操作
    // 如果不是，那只能通过不断自增达到+d操作
	if (iter is a random access iterator ) iter += d;
	else {
		if (d >= 0) {
			while (d--) iter++;
		} else {
			while (d++) iter--;
		}
	}
}

// 1.在每一个用户自定义的类型种嵌套定义一个迭代器类型信息
template <...>
class dequeue {
public:
    // 嵌套迭代器类型信息
    class iterator {
    public:
        typedef random_access_iterator_tag iterator_category;
        ...
    }
};

// 2.提供一个template用于表现迭代器的traits信息
template <typename IterT>
struct iterator_traits {
    typedef typename IterT::iterator_category iterator_category;
    ...
};

// 3.提供一个具体化版本用于指针
template <typename IterT>
struct iterator_traits<IterT *> {
    // 在内部声明某个typedef为iterator_category, 即迭代器类型
    // 这个迭代器类型用于确认迭代器分类
    typedef random_access_iterator_tag iterator_category;
    ...
};

// 4.劳工函数模板-针对随机存取迭代器
template <typename IterT, typename DistT>
void doAdvance(IterT &iter, DistT d, std::random_access_iterator_tag) { // 注意这里的第三个参数，如果有多种参数类型，C++函数解析会根据参数类型选择函数
    iter += d;
}

// 5.工头函数模板-调用劳工函数
template <typename IterT, typename DistT>
void advance(IterT &iter, DistT d) {
    doAdvance(iter, d, typename std::iterator_traits<IterT>::iterator_category()); // 注意这里第三个参数是一个对象，它具有适当的迭代器分类
}
```

**使用traits class类：**

- 建立一组重载函数（身份像劳工）或者函数模板，彼此之间的差异只在于各自的traits参数；
- 建立一个控制函数（身份像工头）或者函数模板，调用上述的劳工函数并传递traits_class所包含的类型信息。

这样C++编译器会根据传入的参数类型判断使用哪一个劳工函数。

## 条款48：认识template元编程

Template metaprograming（TMP， 模板元编程）是编写template-based C++程序并执行与编译期的过程。以C++写成、执行于C++编译器内的程序，一旦TMP程序执行，也就是从template具现出来若干C++源码，便会一如既往的编译。

作用1：让某些事情变得容易，有些事没有它是困难甚至不可能的

作用2：将一些工作从运行期转移到编译期。这导致的结果一是让某些运行期才能发现的问题在编译期就能发现，二是可能会在一下方面高效：较小的可执行文件、较短的运行期、较少的内存需要。但是编译时间会变长。

不使用TMP编程产生的错误例子：

以上条款47为例子，假如使用traits class时不采用函数重载的方法，而是使用typeid的运行期类型识别（RTTI）方法：

```c++
template <typename IterT, typename DistT>
void advance(IterT &iter, DistT d) {
	if (typeid(typename std::iterator_traits<IterT>::iterator_category) == 
		typeid(std::random_access_iterator_tag)) {
		iter += d;	
	else {
		if (d >= 0) {
			while (d--) iter++;
		} else {
			while (d++) iter--;
		}
	}
}
// 声明一个双向迭代器而非随机存取迭代器
std::list<int>::iterator iter;
// 调用会报错
advance(iter, 10);
```

**为什么编译会出错？**

首先编译器会根据参数类型生成相应的函数代码，在生成“iter += d”的代码时，由于iter的类型是双向迭代器而非随机存取迭代器，所以这行代码是错误的；

虽然在运行期间通过typeid可以知道这行代码是永远不会执行的，但是在编译期间编译器会为“永远不会执行的代码”也生成目标码，所以这行代码无法通过编译。

因此在上一个条款中使用的逻辑是：if...else逻辑（即判断是哪个类型在使用哪个操作）被template和它的具体化版本体现出来（函数重载解析判断参数类型选择函数也就是操作）。

注意，TMP元编程被证明是“图灵完全机”，这意味着它可以计算任何事物，声明变量、执行循环、编写以及调用函数。

**下面以TMP通过递归模板具现化实现循环：(以及如何在TMP中创建和使用变量）**

计算阶乘：

```c++
// 类模板
template <usigned n>
struct Factorial {
	enum { value = n * Factorial<n-1>::value };
};

// 类模板特化（具体化）
template <>
struct Factorial<0> {
	enum { value = 1 };
}；
```

TMP能够做到的目标：

1.确保度量单位正确。确保度量单位的正确组合是根关键的，比如允许把“距离”类型的变量与一个“时间”类型的变量的商赋给一个“速度”类型的变量，再比如不允许把一个质量类型的变量赋给一个速度类型的变量。使用TMP可以确保在编译期中程序中所有的度量单位组合正确；

2.优化矩阵运算。比如实现了一个矩阵运算，那么Matrix res = a * b * c *d这个表达式会生成4个临时矩阵变量，如果使用高级的TMP技术，有可能消除临时对象；

3.用来生成基于政策选择组合的客户定制代码，即用户可以选择以什么样的设计模式（设计模式如strategy、observer等可以有很多种方式实现，把这些实现作为选项提供给用户，用户选择组合选项实现行为）组合实现行为。
