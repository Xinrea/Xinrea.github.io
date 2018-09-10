---
layout: post
title: "设计模式 - 单例模式"
date: 2018-08-01
tags: c++ singleton design-pattern
---
## 静态成员方式

类中所有成员均设为静态（static），例如：

```cpp
class Singleton {
 private:
  static int data1; // 类内声明
  static int data2;
 public:
  static int getData1(){
    return data1;
  }
  static int getData2(){
    return data2;
  }
};
int Singleton::data1; // 类外定义
int Singleton::data2;
```

当然，这样声明的话，所有的成员均只会存在一份，实现了单例模式的`基本要求`，这种方法只是给全局变量及方法加上了一个类域而已。

使用静态成员函数的缺点如下所示：

* 静态成员函数不能为`const`。静态成员函数只能够访问其参数、类的静态成员和全局变量，参数中并没有隐藏的`this`指针，而C++中`const`是用于修饰`this`指针的，因此静态成员函数不可能为`const`；同理，`volatile`也是如此，被`volatile`修饰的成员函数可以被`volatile`对象访问，`volatile`对象无法访问非`volatile`成员函数。
* 静态成员函数不能为`virtual`。虚函数机制由虚函数指针和虚函数表实现，其中虚函数指针为类的普通成员变量，这意味着一个虚函数必定与一个实例对象紧密相关，没有对象的话，虚函数指针和虚函数表都不存在，就无从谈起“虚静态成员函数”了，因此`static`和`virtual`相冲突，成员函数可不能可能既是静态的又是虚的。
* 静态成员变量的初始化顺序难以控制。变量的初始化过程：`基类的静态变量/全局变量->子类的静态变量/全局变量->基类的成员变量->子类的成员变量`，其中静态成员变量的定义在类外，当静态成员变量间有初始化依赖顺序时，在类外的定义顺序就变得额外重要，如果顺序出错就会导致初始化出错，难以控制和修改。**注意：`const`成员变量在进行列表初始化时，初始化顺序与列表中的顺序无关，而只与在内存中的位置-即在类内定义的顺序有关**

## 饿汉模式

单例实例作为静态成员变量，类内声明类外定义，在静态变量初始化过程中生成单例，例如：

```cpp
class Singleton {
 private:
  static Singleton instance;
  Singleton(){
  }
 public:
  static Singleton& getInstance(){
    return instance;
  }
};
Singleton Singleton::instance;
```

这种方式仍然存在初始化顺序的问题，由于单例在静态变量初始化过程中完成创建，多个单例类在此期间的初始化顺序时难以控制的，若它们之间有着依赖关系，在构造函数中用到了其他单例对象，则可能导致错误。

在使用时，应注意用引用来获得实例：

```cpp
Singleton& ins = Singleton::getInstance();
```

否则会导致一次构造，多次解析。

## 懒汉模式

单例实例只在第一次使用时进行初始化。例如：

```cpp
class Singleton {
 private:
  static Singleton* instance;
  Singleton(){
  }
  ~Singleton(){
  }
 public:
  static Singleton* getInstance(){
    if (instance == NULL) {
      instance = new Singleton();
    }
    return instance;
  }
};
Singleton* Singleton::instance = NULL;
```

缺点如下：

* `getInstance()`中使用的`if`语句，在多线程情况下并不安全。
* `getInstance()`获得的是单例的指针，无法自动析构。

可以添加一个Destructor()方法，来手动析构。

## 懒汉模式改进

使用局部静态变量，当进入`getInstance()`方法中时，局部变量初始化。例如：

```cpp
class Singleton {
 private:
  Singleton(){
  }
 public:
  static Singleton& getInstance(){
    static Singleton instance;
    return instance;
  }
};
```

缺点：

* 任意两个单例类的构造函数（`getInstance()`）不能够互相引用对方的实例。
* 单例的实例互相引用时，注意析构带来的问题。
* 多线程下，编译器仍然是相当于使用if语句判断`instance`是否初始化，因此也是多线程不安全的。为了解决多线程的问题，可以向`instance`加锁，但是初始化后，每次`getInstance()`仍会经历锁的过程，导致效率下降。

## 最终实现

boost的实现方式：综合局部静态变量和类静态全局变量，使用代理类的静态全局变量的构造函数，来调用`getInstance()`生成局部静态变量，即在静态变量/全局变量初始化的过程中，完成了单例类的实例化。例如：

```cpp
class Singleton {
 private:
  Singleton(){
  }
 protected:
  class Proxy {
    Proxy(){
      Singleton::getInstance();
    }
  };
  static Proxy proxy;
 public:
  static Singleton& getInstance(){
    static Singleton instance;
    return instance;
  }
};
Singleton::Proxy Singleton::proxy;
```