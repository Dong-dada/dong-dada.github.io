---
layout: post
title:  "std::function"
date:   2017-09-27 21:55:30 +0800
categories: cpp
---

* TOC
{:toc}


`std::function` 是标准库提供的一个类模板，它可以存储任何可以调用的 (Callable) 东西，包括普通函数、lambda 表达式、仿函数等。

没了，这就是它的功能，包装各种 Callable.

看个例子：

```cpp
#include <functional>
#include <iostream>

int NormalFunc(double x, int y)
{
    return x * y;
}

class Functor
{
public:
    Functor(int num) :
        num_(num)
    {
    }

    bool operator() (int x, int y) const
    {
        return x * num_ > y;
    }

private:
    int num_;
};

class TestClass
{
public:
    int Foo(int a)
    {
        return a * 10;
    }

    static int Bar(int a)
    {
        return a * 100;
    }
};

int main()
{
    using std::cout;
    using std::endl;

    // 包装普通函数
    std::function<int(double, int)> normal_func = NormalFunc;
    cout << normal_func(1.3, 3) << endl;

    // 包装仿函数
    std::function<bool(int, int)> functor_func = Functor(10);
    cout << functor_func(1, 3) << endl;

    // 包装 lambda 表达式
    std::function<int(bool)> lambda_func = [](bool is_odd) -> int {
        return is_odd ? 13 : 42;
    };
    cout << lambda_func(false) << endl;

    // 类成员函数，需要使用 std::bind 把 this 指针保存下来
    TestClass obj;
    std::function<int(int)> obj_member_func = std::bind(&TestClass::Foo, obj, std::placeholders::_1);
    cout << obj_member_func(3) << endl;

    // 类静态函数(其实和普通函数一样)
    std::function<int(int)> class_member_func = TestClass::Bar;
    cout << class_member_func(13) << endl;

    return 0;
}
```

`std::function` 相对于函数指针来说，具有类型安全的特点，`std::function<int(double)>` 与 `std::function<doubel(bool)>` 是两种不同的类型，而函数指针则没有这种限制，可以随便转换，这样容易出问题。

另外，对于参数和返回值相同的 Callable, 他们可以转换为相同类型的 `std::function`, 一个稍微实际点的例子：

```cpp
#include <functional>
#include <iostream>
#include <vector>

class Downloader
{
public:
    // 监听下载进度改变事件
    // callback(size_t progress), progress 表示下载进度，100 表示下载完毕
    void AttachProgressChangeEvent(std::function<void(size_t)> callback)
    {
        callback_list_.push_back(callback);
    }

    // 没法儿编写 Detach 函数，因为 std::function 不能进行比较
    // void DetachProgressChangeEvent(std::function<void(size_t)> callback);

private:
    std::vector<std::function<void(size_t)>> callback_list_;
};

void OnProgressChange(size_t progress)
{
    std::cout << progress << std::endl;
}

int main()
{
    Downloader downloader;
    // 因为 callback 用 std::function 存储，所以既可以传入普通函数，也可以传入 lambda 表达式
    downloader.AttachProgressChangeEvent(OnProgressChange);
    downloader.AttachProgressChangeEvent([](size_t progress) {
        std::cout << progress << std::endl;
    });

    // ...

    return 0;
}
```

上面代码中使用 `std::function` 存储 callback，这样外界可以传入普通函数、lambda 表达式、成员函数等内容，方便了很多。当然上述代码在实际中应该是不好用的，我还没想到编写 Detach 函数的方法。