---
title: C++面试我一定会问的一个问题
date: 2016-05-21 20:10:36
categories: 
- C++
tags:
- OOP 
---

C++是一门难学易用的语言，如果没有深刻理解一些基本概念，编写的代码就会经常出现一些奇怪的问题。

下面这个问题就是我当初学习C++ OOP概念时遇到的，也是我在工作中遇到的，这个问题没有复杂的技巧，也不是“茴香豆的茴有几种写法”之类的死板教条，只要你基础扎实，一定能回答出来。

问题很简单，我们都知道，C++中基类指针可以指向派生类对象，那么下面的代码是否正确？

<!-- more -->

```cpp
#include <iostream>
using namespace std;

class Base
{
public:
    int a;

    Base(int a): a(a) {}
};

class Derived: public Base
{
public:
    Derived(int a): Base(a) {}

    void Print() {
        cout << "a: " << a << endl;
    }
};

int main()
{
   Derived d(100);
   Base* pb;
   pb = &d;
   pb->Print();

   return 0;
}
```

有啥想法了吗？

如果没有，编译时会输出下面信息，能看出什么端倪了吗？

```
$ g++ test.cpp -o test
test.cpp: In function ‘int main()’:
test.cpp:27: error: ‘class Base’ has no member named ‘Print’
```

提示错误：基类‘Base’没有成员的名字是‘Print’，我们预想通过基类指针去调用派生类的函数成员。

> “如果调用非虚函数，则无论实际对象是什么类型，都执行基类类型所定义的函数。”
>
>--《C++ Primer》第四版中文版480页

没错，如果要想通过基类指针调用派生类的函数成员，得将函数定义成虚函数，像下面这样：

```c++
#include <iostream>
using namespace std;

class Base
{
public:
    int a;

    Base(int a): a(a) {}
    
    virtual void Print() {}
};

class Derived: public Base
{
public:
    Derived(int a): Base(a) {}

    virtual void Print() {
        cout << "a: " << a << endl;
    }
};

int main()
{
   Derived d(100);
   Base* pb;
   pb = &d;
   pb->Print();

   return 0;
}
```

```
$ g++ test.cpp -o test
$ ./test 
a: 100
```

这里涉及到C++ OOP的多态概念，希望你通过此问题对C++多态有更深入的理解，也说不定这个问题在你眼里根本就不是问题呢，哈哈。
