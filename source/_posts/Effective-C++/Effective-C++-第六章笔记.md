---
title: 《Effective C++》第六章笔记
date: 2023-10-13 10:30:58
description: 《Effective C++》第六章读书笔记，本章主要介绍C++在面向对象编程上的一些细节，比如继承可以是单一继承或者多重继承；每一个继承连接可以是public、protected、private，也可以是virtual(虚基类)或者non-virtual；成员函数是否为virtual或者non-vritual等等。
categories:
- 《Effective C++》
tags:
- 读书笔记
---

# 第六章 继承与面向对象设计

## 条款32：public继承

public继承意味着“is-a”关系，即适用于基类的行为也一定适用于派生类，因为每一个派生类对象都是一个基类对象。

但是，这会与真实世界相悖，比如正方形是一种矩形，但是施加于矩阵的动作比如只改变长度不改变宽度却不适用于矩形，因为正方形的长宽一起变化。

再比如鸟会飞，企鹅是一种鸟，但是企鹅类不会飞，不应该有fly函数。这里其实是语言的不严谨导致的，即鸟会飞指的是大部分鸟会飞。

## 条款33：避免遮掩继承而来的名称

名称遮掩不会在意类型，比如在全局名称空间和函数局部空间内如果分别由double类型和int类型的同名变量，那么函数内部的变量名会遮掩函数外的全局变量，**无关类型。**

同样的，如果派生类中有与基类中**同名**的函数或者变量（**无关函数特征标和类型，不论函数是否为virtual和non-virtual**），那么就会遮掩基类的函数或者变量，**就如同派生类作用域嵌套在基类中一样**。
解决方法之一就是在派生类中使用using声明式声明基类中的同名函数，并且**重新定义或者覆写**基类中的同名函数。。

**在public继承中，不应该出现遮掩**，因为遮掩本身就表示派生类不能完成基类的行为，这违反了is-a关系。

如果是private继承，可以使用一个转交函数，即调用基类的同名方法。

## 条款34：区分接口继承和实现继承

接口继承和实现继承不同。**在public继承下，派生类总是继承基类的接口**。（接口在这里可以理解为类向外提供的方法）

- **pure virtual 函数：只继承接口。**声明一个纯虚函数的目的目的是只让派生类继承接口，派生类需要根据自己的需求进行具体的实现。（C++中也可以为纯虚函数定义，但是调用方式必须指出类作用域。）

- **impure virtual 函数：继承接口和一份缺省实现。**如果派生类自己不想实现这个接口，那么就会使用缺省的实现（即基类的实现）。
  默认情况下基类的接口和实现是同一个函数。改变这种方式使得派生类必须了解基类的缺省行为才能使用它：1.在基类中将接口和缺省实现分开，接口使用纯虚函数实现，缺省实现使用protected方法实现，然后在派生类对于纯虚函数的实现中调用基类的protected实现；2.在基类中使用纯虚函数，并为纯虚函数提供一份定义，然后在派生类中调用基类的纯虚函数的实现。

  ```c++
  // 基类
  class Airplane {
  public:
      virtual void fly() = 0; // 将接口声明为纯虚函数
  protected:
      void defaultFly(); // 缺省实现为protected，只在继承体系中可见
  };
  
  void defaultFly() {
      // 缺省（默认）行为
  }
  
  
  // 模型A使用缺省实现
  class ModelA : public Airplane {
  public:
      virtual void fly() {
          defaultFly(); //调用缺省实现
      }
  };
  
  // 模型B不想使用缺省实现
  class ModelB : public Airplane {
  public:
      virtual void fly() {
          // 定义不同的行为
      }
  }
  
  ```

  （上述的目的主要在于防止在继承时如果有自己独立的行为忘记实现特殊性，导致调用了默认实现）

  其实可以直接在基类中定义纯虚函数，并提供一份实现。如果派生类不希望定义特殊行为，就直接调用这份实现；如果希望定义特殊行为，就提供不同的定义。

  ```c++
  // 基类
  class Airplane {
  public:
      virtual void fly() = 0;
  }
  // 对于纯虚函数提供一份默认实现
  void Airplane::fly() {
      // 默认实现
  }
  
  // 模型A愿意使用默认实现
  class ModelA : Airplane {
  public:
      virtual void fly() {
          // 调用基类的默认实现
          Airplane::fly(); // 注意调用时需要加上类作用域
      }
  }
  
  // 模型B不想使用默认实现
  class ModelB : Airplane {
  public:
  	virtual void fly() {
          // 提供一份自己的实现
      }
  }
  ```

- **non-virtual 函数：继承接口和一份强制实现。**这意味着派生类必须拥有一份基类的方法实现，代表了不变性凌驾于特异性。

注意：

- 不要把所有的方法都声明为non-virtual 函数，除非这个类永远不用作基类。
- 不要把所有的方法都声明为virtual 函数，除非派生类真的需要重新定义行为。

## 条款35：考虑virtual函数之外的其他选择

virtual函数目的：派生类与基类的同名方法有不同的行为（多态），即在运行期间选择调用的函数。下面是一些可以考虑的替代方案：

- **由non-virtual interface 手法实现Template Method模式：**将原virtual函数设计为public的non-virtual函数，然后把真正的行为实现在一个private的virtual函数中，这样派生类会继承non-virtual函数，并在该函数内部调用private 的virtual函数，派生类可以通过重写这个私有方法来实现特异性。这个public的non-virtual称为virtual函数的外附器（wrapper），包装起来之后就可以在调用真正的virtual函数之前和之后做一些其它动作（比如上锁、日志、验证约束条件等等）。

  ```c++
  // 基类中定义了一个缺省实现
  class GameCharacter {
  public:
      int healthValue() {
          // 在调用真正的行为函数之前做一些处理
          ...
          doHealthValue(); // 调用虚函数
          
          // 在调用之后做一些处理
          ...
      }
  private:
      // 真正实现行为
      // 派生类可重新定义这个行为
      // 也不一定非要是private
      virtual void doHealthValue() {
          ...
      }
  }
  ```

  

- **借由Function Pointers实现Strategy模式：**类构造函数接受一个函数指针，并通过指针调用该函数。1.同一个类可以根据传入的函数指针（相当于函数）不同而有不同的函数；2.类的成员函数指针可以在运行期间发生变化。
  缺点：由于这个真正的实现函数不是成员函数或者友元函数，因此不能使用类的私有成员信息。如果想要使用类内部信息，就需要声明为友元，这样就会降低封装性。（使用函数指针在运行时指向不同的函数来实现虚函数）

  ```c++
  class GameCharacter; // 前置声明
  // 这是一个计算健康指数的默认实现
  void DefaultHealth();
  
  // 
  class GameCharacter {
  public:
      typedef void (*HealthFunc)(); // 重命名函数指针类型
      explicit GameCharacter(HealthFunc f = DefaultHealth) : h(f) {}
      
  private:
      HealthFunc h; // 函数指针，需要指向真正的实现函数
  }
  ```

  

- **借由std::function 完成Strategy模式：**对于特征标（返回类型和参数类型）相同的函数、函数指针、函数对象以及有名称的匿名对象等（这些都是可调用类型，callable type），可以用一个function对象实例化，然后调用这个function就相当于调用这些函数指针、函数对象。使用这个function的方式与使用函数指针、函数对象相同，都是名称(参数)。（使用函数对象在运行期间有不同的实例化来完成虚函数）
  std::function在头文件functional中声明，使得能够以统一的方式处理多种类似于函数的形式。从调用特征标(有返回类型以及用括号括起来并用头号分隔的参数类型列表定义)的角度定义一个对象，用于包装调用特征标相同的函数指针、函数对象或lambda表达式。比如，下面的声明创建了一个名为fdci的function对象，它接受一个char参数和一个int参数，返回一个double值：
  std::function<double(char, int)> fdci;
  然后可以将接受一个char参数和一个int参数，返回一个double值的任何函数指针、函数对象或lambda表达式赋给它；
  最后就可以把function类型的对象fdci当作可调用物使用，即fdci(char, int)的形式。
  下面的例子将上面一个例子中的函数指针替换为std::function对象实现策略模式：

  ```c++
  class GameCharacter; // 前置声明
  // 这是一个计算健康指数的默认实现
  void DefaultHealth();
  
  // 
  class GameCharacter {
  public:
      // 任何可调用物（只要接收void参数，返回void参数）
      // 可以是函数指针、函数对象（重载了operator()运算符的类、lambda表达式
      typedef std::function<void (void)> HealthFunc;
      explicit GameCharacter(HealthFunc f = DefaultHealth) : h(f) {}
      
  private:
      HealthFunc h; 
  }
  ```

  

- **传统的Strategy模式：**将一个继承体系的virtual函数替换为另一个继承体系的virtual函数。（在前者继承体系里面有一个成员是指向后者继承体系对象的指针，通过这个指针来调用后者体系中的虚函数）

  ```c++
  class GameCharacter; // 前置声明
  
  // 算法族（也就是上面所说的另一个继承体系）
  class HealthFunc {
  public:
      // 这个方法接受一个GameCharacter类型的对象
      // 后面会解释为什么
      virtual int calc(const GameCharacter &gc) const {
          // 这是一份默认实现，算法族继承体系中的子类可以改变实现
      }
  };
  
  HealthFunc defaultHealthCalc; // 创建一个算法对象
  
  class GameCharacter {
  public:
      explicit GameCharacter(HealthFunc *p = &defaultHealthCalc) : pHealthCalc(p) {}
      
      // 真正调用算法族
      int healthValue() const {
          return pHealthCalt->calc(*this);
      }
  private:
      // 这里一定要用指针而不能是对象
      // 观察两个类的关系：
      // GameCharacter调用HealthFunc对象的方法
     	// HealthFunc方法会接受一个GameCharacter类对象
      // 也就是两个类之间其实是相互包含关系
      // 如果下面使用的是类对象形式而不是指针
      // 那么由于相互包含的编译关系，无法通过编译
      // 在条款31中有提到，这是因为C++看到自定义类型之后，需要去寻找定义，这样才能判断需要多少空间。但是在算法族的定义中又需要包含GameCharacter的定义，这就会产生循环依赖
      // 但是如果是一个指针类型，那么其大小是固定的，不需要查看定义。只有在解引用时才会用到定义
      HealthFunc *pHealthCalc;
  }
  ```

  

## 条款36：绝不重新定义继承而来的non-virtual函数

对于public继承体系来说，非虚成员函数是静态绑定的（不论指针指向继承体系中哪个类的对象，调用的方式都是指针类型的方法）；虚成员函数是动态绑定的（根据指针指向的对象类型来选择调用方法）。引用同理。

理论上解释：

- public继承是一种is-a关系，这意味着适用于基类的每一件事都应该适用于派生类，因此两个方法不一样是不合理的。
- 派生类一定会继承基类的non-virtual方法接口和实现。

所以如果派生类真的需要一个和基类同名但是实现不同的函数，那就应该声明为virtual。

## 条款37：不要重定义继承而来的默认参数

（只能继承两种函数：虚函数和非虚函数，上一个条款已经说明不能重定义一个非虚函数，所以接下来的讨论只针对虚函数是否带有默认参数）

默认参数值是静态绑定的（因为实现在运行期间确定virtual函数的默认参数的机制更为复杂），而virtual函数是动态绑定的。

因此当对于**派生类对象调用virtual函数**时使用的默认参数是**基类中声明的默认参数**。

如果真的想要重新定义默认参数，可以采用条款35提到的virtua函数之外的实现方法，比如non-virtual函数加上private的virtual函数实现，在基类的non-virtual函数中指定默认参数类型。**即可以覆写的只能是virtual函数。**

## 条款38：通过复合塑模出has-a关系和“根据某物实现出”

复合（composition）是类型之间的一种关系，可以表示has-a关系和“根据某类型实现出”的关系。

- has-a关系：某些类型是另一种类型的组成部分，比如Person类具有string类作为名字等；
- is-implemented-in-terms-of关系：一种类是根据另一种类实现出来的类型。比如set容器可以借助list容器实现，但是不应该是继承list，因为继承表示所有适用于list的操作也适用于set，比如重复元素就不合适set，因此set应该是根据list实现出来的类型。

## 条款39：明智而审慎地使用private继承

（public继承与private继承之前的区别：在public继承体系中，编译器会在必要时刻（为了让函数调用成功）会将派生类转换为基类，而private不会自动将一个派生类对象转换为基类对象。）

private继承意味着只有实现部分被继承，接口部分被略去。这是因为基类中的public部分（接口）和protected都被变成private形式。

- **private继承意味着“根据某物实现出”的关系：**通常使用复合而不使用继承。只有在派生类需要访问基类的protected成员和需要重新定义继承而来的virtual函数时才会使用private。（这种方式也可以使用在一个类中定义它所需要的那个类的派生类（定制化版本）来完成，即复合+public继承）
- **private继承可以造成empty base最优化：**当一个类是空时，即没有成员数据时，它的大小一般被设置为1（不同的编译器可能为了对齐位而设置int大小）。也就当被用作另一个类的成员时，空类会占用内存；但是用作继承的基类时，空类不占用内存。

## 条款40：明智而审慎地使用多重继承

一、多重继承意味着在一个继承体系中某个派生类与某个基类之间有两个以上的相通路线。这就意味着基类成员可能经由多个路线被复制，出现多个副本。

C++并没有限制是否应该只有一份这样的基类副本，但是可以使用虚基类的方式使得派生类只有一个基类副本，就是在该基类的直接派生类使用virtual继承。

二、virtual继承需要付出空间和时间代价。由编译器提供若干trick，因此会造成空间和访问时间上的消耗。

并且派生类的初始化需要**直接**调用虚基类的构造函数，这意味着不管这个派生类距离基类有多远，都需要明确认知基类。。

1.非必要不适用虚基类；2.如果需要使用虚基类，那么尽量不要在虚基类中声明变量，以减少初始化的麻烦。

三、当涉及到“public继承某个接口类（抽象基类，带有纯虚函数的类）”和“private继承某个协助实现的类”时，需要多重继承这两个类。
