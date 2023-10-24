---
title: 《Effective C++》第四章笔记
description: >-
  《Effective
  C++》第四章读书笔记，软件设计就是希望软件做出期望的步骤和做法，通常从一般性的构想开始，逐渐实现细节，以允许特殊接口的开发。这些接口最终实现为C++的声明。本章主要是对良好的接口设计与声明进行规范。
categories:
  - 《Effective C++》
tags:
  - 读书笔记
abbrlink: 3988871433
date: 2023-10-10 10:48:25
---

# 第四章 设计与声明

软件设计就是希望软件做出期望的步骤和做法，通常从一般性的构想开始，逐渐实现细节，以允许特殊接口的开发。这些接口最终实现为C++的声明。

本章主要是对良好的接口设计与声明进行规范。

## 条款18：让接口容易被正确使用，不易被误用

下面以一个例子说明接口的设计：

```c++
// 定义一个表示日期的类
class Date {
public:
    // 构造函数需要月、日、年
    Date(int month, int day, int year);
    ...
};
```

上面的构造函数需要传入月份、天、年，但是实际上用户可能会按照任意顺序或者传入非理想值（比如月份传入13）。

解决顺序问题：也就是月、天、年必须是不同的类型，因此可以采用类型系统的方式，即设计三种类型表示（下面为了演示使用struct实现，其实应该仔细设计自定义类型为class）：

```c++
struct Day {
	explicit Day(int d) : val(d) {}
	int val;
};
// 剩下两种也类似

// 重写Date类的接口
class Date {
public:
    // 构造函数需要月、日、年
    Date(const Month &m, const Day &day, const Year &year); // 注意这里用const，因为可以将右值（临时对象）作为参数
    ...
};

// 使用例子
Date date(Month(1), Day(12), Year(1995));
```

解决限制值的问题：也就防止用户输入意料之外的值，比如month只能是1~12。注意这里如果使用enum表示月份的话不具有类型安全性，因为enums可以作为一个ints使用。因此理想的做法是预先定义所有有效的month：

```c++
class Month {
public:
    // 因为需要预定义对象，所以这个对象需要是non-local static
    // 但是由于不同编译单元中的non-local static对象会出现初始化次序问题
    // 所以这里均采用静态成员函数的方式实现
    static Month Jan() { return Month(1); } 
    ...
    static Month Dec() { return Month(12); }
private:
    explicit Month(int m);
}

// 使用方式
Date d(Month::Mar(), Day(30), Year(1995));
```

以上的例子说明接口的正确使用和不被误用是需要精心设计的。

- **促进正确使用：**尽量保持用户自定义类型class与内置类型(比如int、short等)的一致性（比如在设计operator=运算符时要返回对象的引用而不是对象，因为像内置类型int是不允许a * b = c的，如果返回值是一个对象，那么就允许上述赋值），以及尽量与内置类型的行为兼容（即内置类型有哪些操作，尽量让自定义类型也有）。
- **阻止误用：**包括设置新类型（**构建符合要求的参数类型**，让用户难以传入错误类型，比如**自定义类型**）、限制类型上的操作（比如operator=运算符返回值是引用等）、束缚对象值（使用const等）、消除用户的资源管理责任（不让用户进行资源释放操作等）。

PS：本节重点介绍了shared_ptr指针（为了说明不要将资源管理的任务交给用户），包括将资源放进指针内，定义完成删除器之后再交给用户使用指针管理的资源。

**shared_ptr一个优秀的性质是：会为对象自动使用专属的删除器。**这样就消除了cross-DLL problem，指的是对象在一个动态链接库（DLL）中被new创建，却在另一个DLL内被delete销毁。之所以shared_ptr没有这个问题，是因为它的删除器是来自对象诞生所在的那个DLL的delete。定义删除器的形式：

```c++
// 创建一个Investment对象并返回这个对象
std::shared_ptr<Investment> CreateInvestment() {
    // 强制将0指针转换为一个空指针
    // 以函数GetRidOfInvestment作为删除器
    std::shared_ptr<Investment> ret_val(static_cast<Investment*>(0), GetRidOfInvestment);
    ret_val = ...; // 指向正确的对象
    return ret_val;
}
```

总之，shared_ptr指针大且慢，换来的是减低客户错误。（条款55）

## 条款19：设计class如同设计内置类型

如何设计高效的class呢？在设计一个class时应该考虑这些问题：·

- **class的对象应该如何被创建和销毁？**这会影响到构造函数、析构函数以及内存分配和释放函数（operator new、operator new []、operator delete、operator delete []）。
- **对象的初始化和赋值有什么差别？**这意味着copy 构造函数和copy assignment运算符有什么不同。
- **对象如果被按值传递，意味着什么？**这会决定copy 构造函数的设计与实现。
- **对象的合法值？**对于class的成员变量什么是合法范围，这会影响构造函数、赋值运算符的实现，需要对错误进行检查。
- **是否继承或者被继承？**如果继承了某个类，就会受到那个类的约束，比如virtual函数等；如果准备被继承，需要考虑析构函数是否为virtual等问题。
- **class需要什么样的类型转换？**如果需要其它类型转换为该类型，则需要考虑单实参构造函数（non-explicit-one-argument）以及是否为explicit等；如果需要将类转换为其他类型，则需要考虑设计类型转换运算符以及是否为explicit等。
- **什么样的操作符和函数对于class是合理的？**影响设计成员函数、运算符（成员还是友元）。
- **什么样的函数应该禁止？**可以使用private或者delete实现。
- **谁需要class的成员？**帮助决定成员为public或者private，以及友元函数友元类等。
- **什么是class的“未声明接口”？**
- **class的泛化能力？**是否需要设计为模板类。
- **是否真的需要一个class？**

## 条款20：尽量以pass-by-reference-to-const替换pass-by-value

- **对于用户自定义类型：**尽量使用按引用传递，为防止函数内部修改加上const。原因：1.比较高效。如果按值传递对象，那么至少调用一次copy 构造函数和析构函数，如果该对象成员中还存在其它类对象，那么就会调用更多copy 构造函数和析构函数。（reference往往以指针实现，因此按引用传递通常意味着传递的是指针）；2.能够避免切割问题，比如在一个继承体系中如果函数参数声明的是基类按值传递，如果传递一个派生类对象，那么派生类对象独有的部分就被切割，只有基类部分被传递（因为调用的是基类对象的复制构造函数）。
- **对于内置类型和STL的迭代器和函数对象：**按值传递往往比较妥当。

## 条款21：必须返回对象时，别妄想返回引用

1.不要返回指针或者引用指向一个local stack对象（自动变量）；

2.不要返回指针或引用指向一个heap-allocated对象（动态内存）；（因为极有可能会忘记释放内存）

3.不要返回指针或引用指向一个local static对象（无链接性的静态变量）而有可能同时需要多个这样的对象时。（单线程合理返回指向一个local static对象的方式是在一个函数内初始化之后返回，并且只有一个引用指向这个对象）

## 条款22：将成员变量声明为private

好处：

- **保持一致性：**将所有成员设置为private之后，用户使用类时对所有成员的调用只能是方法，而不需要考虑哪个是成员数据（不需要圆括号），哪个是成员方法（需要圆括号）。
- **精确控制成员的访问权限：**1.不可访问：不提供get函数和set函数；2.只读访问：提供get函数；3.只写访问：提供set函数；4.读写访问：提供get和set函数。
- **封装：**封装性与当前内容改变时可能造成的代码破坏量成反比，也就是说，如果一个成员是public的，外部可以随便使用，那么当这个成员被删除时，几乎所有直接使用这个成员的代码都被破坏。
- **为“可能的实现”提供弹性：**可以选择是否实现成员变量被读或者被写时通知其他对象、验证class的约束条件等。

protected成员并不比public成员更具封装性，因为protected成员更改时破坏的是派生类，public破坏的是自身类。

## 条款23：宁以non-member、non-friend函数替换member函数

- **封装性：**封装性越高，越少人去看到它，也就具有越大的弹性去改变它，因为改变仅仅直接影响看到它的人或事物。因此，越多东西被封装，我们改变这些东西的能力也就越大，即封装使得改变事物只影响有限客户。member函数向外提供接口，因此是降低了封装性，而non-member和non-friend函数对内部并没有访问权限，因此加强了封装性。

- **使用non-member non-friend函数不意味着这个函数不能是另一个类的成员函数：**比如可以将这个函数定义为工具类的成员函数。
- **机能扩充性（C++组织标准库的形式）：**比如一个完整的类定义中可能有多种功能，但是当只需要其中的一部分时，可以声明一个命名空间，在命名空间中添加non-member non-friend函数使用类的其中一部分功能（也就是将多种不同对于类操作的函数分在不同的命名空间中，需要哪一部分功能就包含哪一部分命名空间。）
  例如C++标准库并不是在一个标准头文件（比如\<C++StandardLibrary>）中包含了std命名空间中的每一样东西，而是数十个文件比如<iostreeam\>、<vector\>等分别声明std的某部分功能。当需要哪一部分机能时就与哪个文件建立编译关系。这种性质也是一个类成员函数无法提供的。

## 条款24：如果所有参数都需要类型转换，请为此采用non-member函数

如果需要为某个函数的所有参数（包含this指针所指向的参数）**进行类型转换**，那么这个函数必须是个non-member函数。

例如为一个类obj定义了同类加法，如果此时定义了单参数构造函数（即允许其他类型向该类进行类型转换），那么 obj + int能够完成（obj.operator+(int)，int类型可以隐式转换为obj类型）；但是int + obj却是错误的，这是因为int类型无法与obj进行运算（int.operator+(obj)，并不存在将obj转换为int类型的转换函数）。

因此这个同类加法不应该是成员函数。如果定义为非成员函数，就能够满足交换律：operator+(obj, int)和operator(int, obj)

**需要注意的是：成员函数的对立面是非成员函数，并不是friend函数。**也就是说，如果一个函数不是成员函数，不一定要定义为友元函数。

## 条款25：尝试写出一个不抛出异常的swap函数

- **当标准库中的std::swap函数对于自定义的class或class template提供可接受的效率（类中需要提供copying操作）：**直接使用这个版本就行。

  ```c++
  namespace std {
      template <typename T>
      void swap(T &a, T &b) {
          T temp(a);
          a = b;
          b = temp;
      }
  }
  ```

  

- **如果std::swap默认版本效率不足（即class或者class template运用了pimpl（pointer to implementation，以指针指向一个对象，内含真正数据）的手法），如果进行交换，其实只需要交换两个指针即可，但是如果调用copying操作，一般需要将所有的内容都进行交换。**因此尝试这么做：

1.提供一个public swap成员函数，让它高效地置换两个对象值，注意这个函数不要抛出异常；

2.在class或者class template所在命名空间（std命名空间不允许添加定义和重载）提供一个**non-member swap函数**或者**部分具体化的swap函数**，让它调用1中的成员swap函数；

3.如果class不是class template，那么可以在std命名空间中**具体化swap函数**，让它调用成员swap函数。



3存在的意义是：即使2已经定义了合适的swap函数，但是有可能会被**std::swap(obj1, obj2)**的形式误用，所以多具体化定义了一个std::swap版本。

同上面原因，**实际调用swap函数前**，应该使用**using std::swap**声明使得标准的swap可见，这样函数查找时会首先查找当前命名空间定义的swap，即2定义的版本；如果没有，就使用std中具体化版本的swap，就是3；如果还没有，最后才会调用一般默认版本。

关于为什么成员版本的swap不要抛出异常（详见条款29）。

下面是非模板类class的两种方式：

```c++
class WidgetImpl {
public:
    ...
private:
    int a, b, c;
    std::vector<double>v;
};

// 1.提供一个public swap成员函数
class Widget {
public:
    ...
    Widget& operator=(const Widget &rhs) {
        ...
       *PImpl = *(rhs.pImpl); // 复制Widget对象时的一般实现
        ...
    }
    
    // 提供一个高效的swap成员函数
    void swap(Widget &other) {
        using std::swap;
        swap(pImpl, other.pImpl); // 仅仅只是置换指针
    }
    
private:
    WidgetImpl *pImpl; // 指向真诚的Widget数据
};

// 针对class的第一种方案
// 直接在std命名空间中提供swap的具体化版本
namespace std {
    template <>
    void swap<Widget>(Widget &a, Widget &b) {
        a.swap(b);
    }
}

// 针对class的第二种方案
// 先对于class提供一个非成员swap函数
void swap(Widget &a, Widget &b) {
    a.swap(b); // 调用成员函数swap
}
// 然后为了防止误用再提供一个std的版本
namespace std {
    template <>
    void swap<Widget>(Widget &a, Widget &b) {
        a.swap(b);
    }
}

```

下面是非模板类class和模板类template class的错误示范，以及模板类的一种正确方式：

<u>注意：</u>**模板函数是不允许部分具体化的！**模板类允许部分具体化。所以不能够对于swap默认版本进行部分具体化（使用另一个模板作为模板函数的参数类型也是一种部分具体化）：

```c++
template <typename T>	// std原始版本
void swap(T& a, T& b) {
	T temp(a);
	a = b;
	b = temp;
}


// 非模板类
class Widget {
private:
    ptr;
public:
    void swap(Widget& b);
};

// 针对非模板类而言
template <>	// 具体化版本1，错误
void swap<Widget>(Widget& a, Widget& b) {
    swap(a.ptr, b.ptr);	// 无法访问私有成员
}

// 针对非模板类而言
template <>	// 具体化版本2，正确
void swap<Widget>(Widget& a, Widget& b) {
    a.swap(b);	// 使用类内定义的方法
}

// 假如有个模板类
template <typename T>
class Widget {
private:
    ptr;
public:
    void swap(Widget& b);
};

template <typename T>	// 部分具体化模板函数，不合法！
void swap<Widget<T>>(Widget<T>& a, Widget<T>& b) {
    a.swap(b);	// 使用类内定义的方法
}

template <typename T>	// 尝试重载std::swap函数，不是部分具体化，不合法！因为std不允许增加内容
void swap(Widget<T>& a, Widget<T>& b) {
    a.swap(b);	// 使用类内定义的方法
}

// 因此只能在自己的命名空间中为class template类模板添加一个部分具体化的swap函数。
template <typename T>
void swap(Widget<T> &a, Widget<T> &b) {
    a.swap(b); // 调用成员函数
}

```



总结就是：

**为class声明一个非默认的swap函数**，可以选择具体化std::swap函数；也可以在自己的命名空间中定义一个swap，后者情况下为了以防有人使用std::swap的方式调用，所以也在std中具体化一个swap。

**为class template声明一个非默认的swap函数**，只能在自己命名空间中定义一个swap函数。因为std不允许重载以及模板函数不允许部分具体化。
