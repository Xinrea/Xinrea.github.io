---
layout: post
title: "🥓Java模块解耦"
date: 2018-08-14
tags: Java
---
## 步骤一 解决简单耦合

### 类型一

对于与对象状态无关的方法，可以直接将该方法本地化，解除耦合，例如：

```clike
class A {
  B b;
...
  aMethod() {
    return b.add(1,2);  
  }
}

class B {
...
  add(a,b) {
    return a+b; 
  }
}
```

可见`B.add()`方法只是完成了简单单一的功能，与类B的对象状态无关，因此可以将其本地化，直接代替原来的调用，以达到解耦的目的。解耦后如下所示:

```clike
class A {
...
  aMethod() {
    return 1+2;  
  }
}
```

### 类型二

调用位置在代码段末尾，而且改调用与自身状态无关，例如：

```clike
class A {
  B b;
  aMethod(){
    dosomething();
    b.inc();
  }
}

class B {
  static int m;
  inc(){
    m++;
  }
}

class C{
  A a;
  cMethod(){
    a.Method();
  }
}
```

由于`B.inc()`涉及到B的状态`int m`，无法直接本地化到A中，但是该方法与A的状态无关，因此可以查找使用了`aMethod`的地方，例如类C中的`cMethod()`方法中调用了`aMethod()`，我们可以将`b.inc()`提到`aMethod()`方法外，例如：

```clike
class A {
  aMethod(){
    dosomething();
  }
}

class C{
  A a;
  B b;
  cMethod(){
    a.Method();
    b.inc();
  }
}
```

可见修改后，类A已经不依赖与类B。

## 步骤二 设计接口和工厂类

对于简单解耦无法完成的工作，就需要设计接口和工厂类来进行解耦了，这种方法简单粗暴，使用Java的反射机制来达到解耦的效果。
例如，有许多模块对模块A有引用，为了解决这种直接的耦合关系，需要为类A设计接口和工厂类。

查找类A中哪些方法被外部所使用，将其提取到IA接口类中，类A设置为implement IA；
实现一个类AFake，同样implement IA，这是当类A未定义时，默认使用的类。

接下来便是实现工厂类AFactory，结合Java的反射机制实现隔离，简单样例如下所示：

```java
//IA.java
package com.test.decouple;
public interface IA {
  boolean method();
}

//A.java
package com.test.decouple;
public class A implements IA {
  public boolean method() {
    return true;
  }
}

//AFake.java
package com.test.decouple;
public class AFake implements IA {
  public boolean method() {
    return false;
  }
}

//AFactory.java
package com.test.decouple;
public class AFactory {
  private static final A = "com.test.decouple.A";
  private static final AFake = "com.test.decouple.AFake";
  public static IA getInstance() {
    Class<IA> mClass = null;
    IA mInst = null;
    try {
      mClass = (Class<IA>)Class.forName(A);
    } catch (ClassNotFoundException e) {
      try {
        mClass = (Class<IA>)Class.forName(AFake);
      } catch (ClassNotFoundException e2) {
        e2.printStackTrace(); 
      }
    }
    mInst = mClass.newInstance();
    return mInst;
  }
}
```

工厂类AFactory中，使用反射来使用类A，当查找不到类A时，转而使用类AFake，确保不会出错。这样进行处理后，工厂类AFactory会根据类A存在与否来选择实例化的方式。替换掉所有使用了`method()`方法的地方后，外部就不再依赖于类A了，这样就达到了单方向完全解耦的效果。
