---
layout: post
title:  "右值引用"
date:   2017-10-17 19:12:30 +0800
categories: cpp
---

* TOC
{:toc}

本文参考自 [这篇文章](https://www.ibm.com/developerworks/cn/aix/library/1307_lisl_c11/index.html) 以及 [这篇文章](http://shaoyuan1943.github.io/2016/03/26/explain-move-forward/)


## 什么是右值

C++ 里所有的表达式和变量，要么是左值，要么是右值。通俗的说左值就是非临时变量，可以在多条语句中使用；右值则是指临时的对象，它们只在当前的语句中有效。

比如： `int i = 0;` 这条语句中， i 是左值， 0 是右值；

再比如如下的代码：

```cpp
class A
{};
A a; // a为左值，因为其有明确名字，且对a进行 &a 是合法的。

void Test(A a)
{
  __Test(a);
}
Test(A()); // A() 为右值，因为A()产生一个临时对象，临时对象没有名字且无法进行 &取址操作。
```

从上述例子可以看出，右值要么是常量，要么是临时的变量，总之你不能对他进行取址操作，换句话说，在右值引用的特性被引入以前，你没法引用右值。

```cpp
// 编译能够通过，但没意义，因为 A() 产生的右值很快就会被销毁，你无法接着使用 ptr
A* ptr = &A();
```


## 什么是右值引用

利用右值引用的特性，你可以用 `&&` 符号来获取右值的引用。获取到右值的引用之后，右值的生命周期将得到延长：

```cpp
#include <cstdio>

int g_obj_index = 0;

class Foo
{
public:
    Foo()
    {
        index_ = g_obj_index++;
        printf("construct \t%d\n", index_);
    }
    Foo(const Foo& foo)
    {
        index_ = g_obj_index++;
        printf("copy construct \t%d\n", index_);
    }
    ~Foo()
    {
        printf("distruct \t%d\n", index_);
    }
private:
    int index_;
};

Foo GetFoo()
{
    Foo tmp;
    return tmp;
}

int main()
{
    Foo&& obj = GetFoo();
    printf("use obj\n");
    return 0;
}
```

上述代码的输出为：

```
construct       0
copy construct  1
distruct        0
use obj
distruct        1
```

第一个构造函数构造了 `GetFoo()` 内的局部变量 `tmp`, 第二个拷贝构造函数是用 `tmp` 构造了一个临时变量，也就是 `GetFoo()` 的返回值(右值)。

本来这个返回值在返回之后就应该立刻销毁了，但因为我们使用 `obj` 引用了它，因此这个右值的生命周期被延长了，它得以在执行 `use obj` 之后才被销毁。

通过右值引用后，原来右值的生命周期变得跟变量 `obj` 的生命周期一样长了。


## 右值引用的语法是怎样的？

右值引用的 `&&` 操作符，只能获取右值的引用，不能获取左值的引用：

```cpp
int&& const_num = 0;    // ok
int&& num = GetNum();   // ok

int num = 0;
int&& ref = num;        // compile error: cannot bind 'int' lvalue to 'int&&'
```

虽然不能直接获取左值的引用，但可以通过 `std::move()` 来将左值强制转换为右值。

**注意：** 它内部不会做移动的操作，只是做了强制转换，使得左值也可以被 `&&` 引用。

```cpp
Foo obj;
Foo&& ref = std::move(obj);

// 这个时候 obj 和 ref 指的是同一个东西
```

你可以将右值引用作为函数参数类型：

```cpp
void Func(Foo&& obj)
{
    // 调用右值的方法
    obj.DoSomething();
}

// 只会调用一次 Foo 的构造函数
Func(std::move(Foo()))
```

你可以将右值引用作为函数返回值类型：

```cpp
Foo&& GetFoo()
{
    Foo tmp;
    return std::move(tmp);
}

Foo&& ref = GetFoo();   // ok
Foo obj = GetFoo();     // 如果有移动构造函数，那么调用它，否则调用拷贝构造函数
```

上述代码中提到了移动构造函数，这需要你自己定义一个：

```cpp
class Foo
{
public:
    // 以下是移动构造函数的定义方式
    Foo(const Foo&& other)
    {
        data_ = other.data_;
        other.data_ = nullptr;  // 因为 other 是右值，意味着它是个临时值，给它设置一个空值也没关系
    }

private:
    int* data_;
    std::string str_;
}

Foo&& GetFoo()
{
    Foo tmp;
    return std::move(tmp);
}

// 这里会调用移动构造函数来构造 obj
Foo obj = GetFoo();
```

在上述代码中，因为声明了移动构造函数，所以 `Foo obj = GetFoo();` 在执行的时候，会通过 `GetFoo()` 返回的右值来对 obj 进行移动构造。

在实现移动构造函数的时候，因为我们清楚的知道传进来的参数是个临时值，因此只需要把它的数据拿过来用就好了，不需要进行深拷贝。


## 右值引用有什么用

### 节省拷贝成本

因为右值引用可以延长临时值的生命周期，因此你可以把右值引用当作左值来使用，从而节省拷贝开销：

```cpp
Foo&& GetFoo()
{
    Foo tmp;
    return std::move(tmp);
}

// 这里的 obj 实际上是 GetFoo() 内部的 tmp
Foo&& obj = GetFoo();
```

在上述例子中，只是构造了一个 tmp 变量，然后把这个变量通过 `std::move()` 和右值引用移动到了 `GetFoo()` 外部的 obj 上面。不必像下面的写法那样创建返回值对象和目标对象，节省了两次拷贝的开销：

```cpp
Foo GetFoo()
{
    Foo tmp;
    return tmp;
}

Foo obj = GetFoo();
```

### 移动构造函数节省深拷贝开销

在过去的 stl 容器中，每次加入一个新元素，都需要进行拷贝操作：

```cpp
std::vector<Foo> vec;

// 构造一次
Foo tmp;
// 拷贝构造一次
vec.push_back(tmp);
```

现在，如果你为 Foo 定义了移动构造函数，那么它可以减少深拷贝的开销：

```cpp
std::vector<Foo> vec;

Foo tmp;
vec.push_back(std::move(tmp));  // 将调用移动构造函数，不需要深拷贝

// 按照我们之前定义的移动构造函数， tmp 将失效，它的 data_ 字段被设为了 null
```


## 总结

因为还没怎么用过，所以这篇文章讲的并不具体，希望以后能通过实际的例子来加深理解。
