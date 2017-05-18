---
layout: post
title:  "软件调试小知识点"
date:   2017-03-20 17:58:30 +0800
categories: debugging
---

* TOC
{:toc}

## WinDBG 中查看调用堆栈

在 WinDBG 中调试正在运行的程序时，有时候会设置断点，或者发生异常时，WinDBG 会自动中断到代码的位置，呈现如下界面：

![自动中断到代码位置]( {{site.url}}/asset/debugging-tips-interrupt-to-code.png )

这个时候怎样查看函数的调用堆栈呢？可以通过 菜单 --> view --> call stack, 打开 call stack 窗口来查看调用堆栈，就像在 Visual Studio 里调试时查看堆栈那样：

![Call Stack 窗口查看调用堆栈]( {{site.url}}/asset/debugging-tips-callstack-viewer.png )

在 call stack 窗口里，可以点击不同的函数，查看函数调用时的参数等信息。


## 在 release 模式下禁止某个文件被优化的方法

这实际上是 Visual Studio IDE 的一个小技巧。

有时候项目只有 release 版本，没有 debug 版本，或者 debug 版本编译不过，这样本机调试的时候只能用 release 版本，很多变量和过程看不清晰，这个时候可以将要调试的文件设为禁止优化。这样一来就可以像 debug 版本那样观察运行时的详细信息了：

在目标 cpp 文件上右键 --> 选择 Properties --> C/C++ --> Optimization --> Optimization 选项选择 Disabled (/Od)。

![]( {{ site.url }}/asset/cpp-tips-release-disable-optimization.png )

记得在调试完成后，将优化选项重设为 Maximize Speed (/O2), 以免影响产品发布后的性能。


## 只中断到指定线程

有的函数会被多个线程调用，如果在这种函数中设置断点的话，每个线程进来都会命中到断点。断点指令在不同的线程间跳来跳去，调试器来很不方便。比如下面的例子代码：

```cpp
#include <iostream>

#include <process.h>
#include <Windows.h>

unsigned __stdcall ThreadFunc(void* params)
{
    ::Sleep(1); // 在这里设置一个断点，每个线程都可能被中断
    DWORD thread_id = GetCurrentThreadId();
    std::cout << "ThreadFunc Enter" << std::endl;

    return 0;
}

int main()  
{
    for (int i = 0; i < 10; ++i)
    {
        _beginthreadex(NULL, 0, ThreadFunc, NULL, 0, 0);
    }

    ::Sleep(10);

    return 0;
} 
```

ThreadFunc 这个函数会在多个线程中被调用，如果你在这个函数中设置了断点，那么每个线程调用 ThreadFunc 的时候都会命中这个断点，并且断点命中之后，再按下 F10 单步调试，不会继续在当前线程执行，而是调到另一个线程的断点上面去了。调试起来很不方便，因为我的目标可能只是一开始跟进来的那个线程。

在 VS2008 里面可以先通过 Thread 工具栏或者通过 Thread 视图来记录下当前的线程 id, 然后在对这个断点设置条件：

上方工具栏空白处右键，选择 Debug Location 菜单项，即可打开 Thread 工具栏：

![Thread 工具栏]( {{site.url}}/asset/debugging-tips-vs2008-thread-toolbar.png )

菜单 Debug --> Windows --> Threads 即可打开 Thread 视图：

![Thread 视图]( {{site.url}}/asset/debugging-tips-vs2008-threads-view.png )

从 Thread 工具栏 或 Thread 视图中可以看到当前断点命中的线程 id 是 9766, 把这个线程 id 记下来。

在原来断点的下方多加一个断点，从 Breakpoints 视图中找到这个断点，然后右键，选择 Filter :

![Breakpoints 视图的 Filter]( {{site.url}}/asset/debugging-tips-breakpoints-filter.png )

在过滤器中添加一个线程 id 的过滤条件：

![Breakpoints 设置 ThreadID Filter]( {{site.url}}/asset/debugging-tips-breakpoints-filter-threadid.png )

新增加的这个断点只会命中到 9766 这个线程中，这就可以方便下一步调试了。