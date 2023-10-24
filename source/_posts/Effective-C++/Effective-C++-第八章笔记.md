---
title: 《Effective C++》第八章笔记
categories:
  - 《Effective C++》
tags:
  - 读书笔记
abbrlink: 2041722108
date: 2023-10-15 09:54:19
---

《Effective C++》第七章读书笔记，主要是关于C++内存管理的行为。主角是operator new和operator delete，配角是new-handler，当operator new无法满足内存需求时调用的函数。

多线程下的内存管理更加复杂。由于heap是全局性资源，多线程会出现对于这一资源的竞争访问。所以在这种情况下内存管理更加复杂（本章并没有提到应该如何处理，只是提醒需要采用同步的方法进行控制）。

另外还需要注意的是分配单一对象和多个对象的区别。即operator new和operator new[]、operator delete和operator delete[]。

最后STL容器使用的heap内存是由分配器管理，而不是new和delete直接管理（本章不讨论分配器）。

<!-- more -->

# 定制new和delete

## 条款49：了解new_handler的行为

当时operator new申请分配内存失败时，会首先尝试调用一个错误处理函数（new handler），如果这个函数不存在，就会抛出异常。

如果用户想要指定这个new handler函数，可以使用set_new_handler函数指定：

```c++
#include <new>
namespace std {
	typedef void (*new_handler)();
	new_handler set_new_handler(new_handler p) throw();
};
```

在std命名空间中，new_handler类型其实是一个函数指针；set_new_handler函数接受一个处理函数指针，返回的函数指针是之前的处理函数。

最简单的new_handler函数：

```c++
void outOfMem() {
    std::cerr << "Out of Memory";
    std::abort(); // 终止程序
}

int main(void) {
  set_new_handler(outOfMem);
  int *p = new int[1000000000000L]；
  ...
};
```

（当op new申请内存失败时，会不断调用new_handler函数，直到找到足够内存。条款51）

**一个设计良好的new_handler函数应该做的事：**

- 让更多内存被使用：以便让new的下一次分配内存成功，一个策略是程序一开始就分配一大块内存，当首次分配失败时，将这块内存归还。
- 设置另一个new_handler函数：如果当前的new_handler函数无法做到申请更多内存，可以设置新的new_handler函数，即调用set_new_handler函数修改处理函数替换自己。还可以修改自己的行为，以便下次被调用的时候做一些不同的事。
- 卸载new_handler函数：即**设置处理函数为空指针**，这样下一次分配失败时就会直接抛出异常。
- 抛出bad_alloc异常：这样的异常不会被new捕捉，而是传播到内存申请的地方。
- 不返回：直接调用abort或者exit函数终止程序。

**对于不同的class分配失败时采用不同的处理方式：**

C++不支持每个class专属的new_handler函数，但是可以通过以下方式实现这种行为：

1.每一个class类定义一个set_new_handler函数：用于指定想要调用的handler处理函数；

2.每一个class类定义一个operator new函数：在分配过程中以class专属的handler函数代替全局的handler函数。

```c++
class Widget {
private:
	static std::new_handler current_handler;
public:
	static std::new_handler set_new_handler(std::new_handler p) noexcept;
	static void * operator new(std::size_t size);
};

// 静态成员数据初始化
// 需要在class实现文件初始化指向new_handler函数的指针
std::new_handler Widget::current_handler = nullptr;

// set_new_handler函数需要做的工作是保存获得的指针p，并将之前的指针作为参数返回
// 标准版的也是这么做的
std::new_handler Widget::set_new_handler(std::new_handler p) noexcept {
    std::new_handler old_handler = current_handler;
    current_handler = p;
    return old_handler;
}
```

**注意:**这里只是把某个new_handler函数与class绑定，但是如果想要在内存分配不足时调用这个new_handler函数需要使用std的set_new_handler函数安装这个处理函数才行。

class的operator new函数应该做的事情：

- 调用std::set_new_handler函数，将class专属的new_handler函数传递给它，以完成安装动作（即使用类专属的new_handler替换global new_handler）。
- 调用std的全局global operator new函数，执行内存分配。**这里需要注意的是：**当内存申请失败时，如果最终抛出异常，那么class的operator new必须保证全局的new_handler函数恢复为原来的默认处理方式。
- 如果std的全局global operator new函数分配内存成功，class的operator new函数返回一个指向内存的指针，但是同时，class的析构函数也需要管理global new_handler函数，也就是需要卸载当前的new_handle函数，将class进行operator new之前的handler恢复。

由于上述原因，需要创建一个资源管理类。这个类保存过去的new_handler，并且当这个对象过期时（其实就是在进入class 的 operator new函数时，创建这个对象保存过去的handler，当new函数结束时，就可以重新安装它了，也就是class专属的new_handler已经没用了）重新安装它保存的new_handler

```c++
class NewHandlerHolder {
private:	
	std::new_handler handler;
public:
	explicit NewHandlerHolder(std::new_handler nh) : handler(nh) {}
	~NewHandlerHolder() { std::set_new_handler(handler); } // 重新安装new_handler
    
    // 禁止copying
    NewHandlerHolder(const NewHandlerHolder &other) = delete;
    NewHandlerHolder & operator=(const NewHandlerHolder &other) = delete;
};
```

有了资源管理类，就可以定义class的operator new函数：

```c++
void * Widget::operator new (std::size_t size) {
	// h对象保留过去的handler，当该类的new动作完成之后，该函数结束时即可恢复原来的handler
	NewHandlerHolder h(std::set_new_handler(current_handler)); 
	return ::operator new(size);
}
```

如何使用上面定义的Widget类专属的new_handler函数：

```c++
// 定义一个Widget类的new_handler函数
void outOfMem() {}

// 设定outOfMem为Widget的new_handler函数
Widget::set_new_handler(outOfMem); 

// 尝试为一个对象申请内存
// 如果内存分配失败就调用outOfMem函数
// 并且处理之后会重置为原来的global new_handler函数
Widget *pwl = new Widget; 

// 如果内存分配失败的话就会调用全局的new_handler函数
string *ps = new string;

```



**复用上述方案：**

即把这个资源管理类当作一个基类，这样所有派生类都可以继承它的特性——独特的new错误处理方式。

同时为了使得每个派生类能够获得实体互异的基类成员数据复件，需要把这个类改成模板。（其实就是不同的派生类有不同的基类，这样就会保存不同的静态成员变量current_handler）

```c++
class NewHandlerHolder;
template <typename T>
class NewHandlerSupport {
private:
	static std::new_handler current_handler;
public:
	static std::new_handler set_new_handler(std::new_handler p) noexcept;
	static void * operator new(std::size_t size);
};

// 需要在class实现文件初始化指向new_handler函数的指针
std::new_handler Widget::current_handler = nullptr;

// set_new_handler函数需要做的工作是保存获得的指针p，并将之前的指针作为参数返回
template <typename T>
std::new_handler NewHandlerSupport<T>::set_new_handler(std::new_handler p) noexcept {
    std::new_handler old_handler = current_handler;
    current_handler = p;
    return old_handler;
}

template <typename T>
void * NewHandlerSupport<T>::operator new (std::size_t size) {
	// h对象保留过去的handler，当该类的new动作完成之后，即可恢复原来的handler
	NewHandlerHolder h(std::set_new_handler(current_handler)); 
	return ::operator new(size);
}

// 资源管理类
class NewHandlerHolder {
private:	
	std::new_handler handler;
public:
	explicit NewHandlerHolder(std::new_handler nh) : handler(nh) {}
	~NewHandlerHolder() { std::set_new_handler(handler); } // 重新安装new_handler
    
    // 禁止copying
    NewHandlerHolder(const NewHandlerHolder &other) = delete;
    NewHandlerHolder & operator=(const NewHandlerHolder &other) = delete;
};

// 使用方式
class Widget : public NewHandlerSupport {
    ...
    // 和之前一样，不必再定义new和set_handler_new函数
}
```

**目前的new：**

之前的new如果申请失败返回空指针，目前的C++如果申请失败会抛出异常。

取消抛出异常：C++11支持采用nothrow的方式阻止抛出异常并返回空指针：

```c++
Widget *p = new (std::nothrow) Widget;
```

上面的表达式做两件事：1.nothrow版本的new被调用，如果分配失败返回空指针；2.如果分配成功调用Widget类的构造函数，而因为这个构造函数可以做任何事情，比如申请一块空间。

**因此虽然new (std::nothrow)不抛出异常，但是上述表达式中的构造函数可能会。综上没有使用它的必要。**

## 条款50：了解new和delete的合理替换时机

有时可能默认的new和delete不能满足要求，因此需要自定义new和delete，理由有如下这些：

- **检测运用上的错误：**比如将”new所得的内存delete失败“、”在new得到的内存上多次使用delete“等等，可以在自定义的new和delete上做一些额外操作检测错误。
- **收集使用上的数据：**比如分配区块的大小分布如何、倾向于先入先出还是后进先出来分配和归还、最大动态分配量是多少。
- **为了增加分配和归还的速度：**由于默认的new和delete往往是泛用的，需要考虑的事情比较多，所以并不针对某一特定类型具有优秀性能，因此为了提高某种类型的分配和规划的速度，可以采用定制化new和delete，但是要首先确认程序的效率瓶颈是默认的new和delete带来的。
- **为了降低默认内存管理器带来的空间额外开销：**泛用型分配器往往不仅比定制型慢，而且还会占用更多内存，因为常常在每一个分配区块上使用更多额外开销。
- **为了弥补默认分配器中的非最佳齐位：**齐位（alignment）是许多计算机体系结构要求特定的类型必须放在特定的内存地址上，比如要求指针的地址必须是4的倍数或者double的地址必须是8的倍数，如果没有齐位，有可能会导致硬件异常或者运行效率不高。因此为了效率更高，需要自行进行齐位操作。
- **为了将相关对象成簇集中：**如果某些对象往往一起使用，并且希望在处理这些对象的时候”内存页错误“的频率最低（尽量将它们放在一起会降低页错误率，操作系统内存管理页结构），那么可以定制化这种行为。
- **为了获得非传统行为：**希望做一些默认版本分配器没有做的行为，比如归还的时候希望把所有的内存都覆盖为0。

ps：并不一定需要自己定制化new和delete，原因1.某些编译器已经在它们的内存管理函数中切换为调式和志记状态，可以查看相关文档从而不必自己编写；2.有很多商业产品可以替代编译期自带的内存管理器；3.开放源码中也会有内存管理器，比如Boost库中的Pool（条款55）就是一款内存管理器。

## 条款51：编写new和delete时需要遵守一些规则

- 自定义的new函数应该：1.返回正确的值；2.申请失败时调用new_handler函数；3.应对0内存需求；4.注意派生类会继承基类的new

```c++
void * operator new(std::size_t size) {
	using namespace std;
	if (size == 0) size = 1;
	
	while (true) {
		尝试分配size bytes;
		if (分配成功) return 指针(指向分配得来的内存)；
		
		// 分配失败，找出目前的new_handler函数
		new_handler global_handler = set_new_handler(0);
		set_new_handler(global_handler);
		
		if (global_handler) global_handler();
		else throw std::bad_alloc();
	}
}
```

**需要注意的是：**上述的new函数在一个循环中，如果申请失败不断调用new_handler函数，退出循环的唯一办法是内存被分配成功或者new_handler函数做出如下行为（条款49）：让更多内存可用、安装另一个new_handler函数、卸载当前new_handler函数、抛出异常、承认失败直接return。

如果operator new作为一个类的成员函数，派生类会继承基类的new运算符，所以当使用new申请一个派生类对象时，调用的是基类的new。如果基类的new并非想处理派生类的内存申请，可以先判断是否申请的是基类的空间，如果不是则转交给std的new函数进行处理：

```c++
void * Base::operator new(std::size_t size) {
	if (size != sizeof(Base)) return ::operator new(size);
	
	// 剩余部分在这里处理
}
```

如果需要实现operator new []，唯一需要做的事情就是分配一块内存，无法计算array中包含多少个对象，因为首先对象大小不确定（因为基类的operator new[]有可能被继承调用，将内存分配给派生类对象使用）

- 自定义的delete函数应该：1.保证删除null指针是合法的；2.如果成员delete内存大小有误，交给std的delete函数处理

```c++
void Base::operator delete(void *rawMemory, std::size_t size) {
	if (rawMemory == 0) return;
	if (size != sizeof(Base)) {
		::operator delete(rawMemory);
		return;
	}
	
	// 归还内存
	return;
}
```

## 条款52：写了placement new也要写palcement delete

**正常new和delete如何处理内存泄漏:**

```c++
Widget *p = new Widget;
```

上述语句先调用new分配内存，再调用Widget的构造函数，如果在调用构造函数时出现异常，需要释放new的内存，由于用户并未获得内存指针，所以无法操作这块内存，**C++会自动调用与之匹配的delete函数**。

```c++
void * operator new(std::size_t); // 正常的new

void operator delete(void *rawMemory); // global作用域中的delete
void operator delete(void *rawMemory, std::size_t); // class作用域中的delete
```

**placement new和delete的含义：**

即带有正常形式参数之外的参数的new和delete，比如处理size之外还接受一个指针指向被构造之处(在用户指定的内存位置上)：

```c++
void *operator new(std::size_t, void *pMemory);
```

（这个new的用途之一就是在vector未使用的空间上创建对象）

语法格式：

```c++
// address：placement new所指定的内存地址
// ClassConstruct：对象的构造函数
Object * p = new (address) ClassConstruct(...);
```

**因此为了防止内存泄漏，就需要有一个placement delete与placement new参数个数个类型完全一致，再加上一个自定义的delete，一共需要一个new和两个delete。（placement new在指定位置申请内存，正常的operator delete在构造期间无任何异常抛出，后续需要归还分配空间时使用，placement delete在构造时如果出现异常由C++运行期系统调用）**

另外，为了防止类中的new和delete掩盖正常的new和delete（如果类中定义了一个placement new但是没有定义普通的operator new，那么无法使用new创建对象，只能使用placement new创建对象，因为成员函数的名称掩盖了其外围作用域中的相同名称），可以采用的**方案是建立一个base class，把所有的正常形式的new和delete包含，然后让class继承这个base class：**

```c++
// 默认的new
void * operator new(std::size_t);
void * operator new(std::size_t, void *);
void * operator new(std::size_t, const std::nothrow_t &);

class StandardNewDeleteForm {
public:
	// 正常形式new和delete
	static void * operator new(std::size_t size) { return ::perator new(size); }
	static void * operator delete(void *pMemory) { return ::delete(pMemory); }
	// placement new和delete
	static void * operator new(std::size_t size, void *ptr) { return ::new(size, ptr); }
	static void * operator delete(void *pMemory, void *ptr) { return ::delete(pMemory, ptr); }
	// nothrow的new和delete
	static void * operator new(std::size_t size, const std::nothrow_t &nh) { return ::new(size, nh); }
	static void * operator delete(void *ptr, const std::nothrow_t &nh) { return ::delete(ptr, nh); }
};

// 正常类继承并声明其中的new和delete函数
class Widget : public StandardNewDeleteForm {
public:
    // using 声明式
    // 令这些形式可见
    using StandardNewDeleteForm::new;
    using StandardNewDeleteForm::delete;
    
    
   // 声明class自己的new和delete
    ...
}
```



