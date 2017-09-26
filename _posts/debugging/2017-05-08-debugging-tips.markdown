---
layout: post
title:  "软件调试小知识点"
date:   2017-03-20 17:58:30 +0800
categories: debugging
---

 
 

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


## DebugView 的过滤规则

之前没注意到过，以为 DebugView 过滤的时候只会过滤 “以 xxx 开头的日志输出”，今天试了一下，发现规则应该是 “过滤日志里包含 xxx 的输出”，也就是说过滤条件可以是输出字符串的任意一部分。

另外 DebugView 的日志开头是有一个进程 id 的，你可以把它加入到过滤条件里面，从而达到只过滤指定进程日志的效果。

另外 DebugView 是可以添加命令行参数的，你可以在命令行中输入 `dbgview.exe /?` 来查看它支持的命令行参数有哪些，例如下面的命令行将 dbgview 最小化到托盘区，并指定将日志输出到文件中。

```
dbgview.exe /t /l C:\dbgview.log
```

也许可以根据 DebugView 输出的日志进行一些简单的定制，不过 DebugView 输出的日志太简单了，想想也没啥可定制的。。。


## 在 VS 中查看寄存器的值

有时候需要在 VS 中查看寄存器的值，这时可以通过菜单 Debug --> Windows --> Registers 来打开寄存器窗口：

![]( {{site.url}}/asset/software-debugging-tips-show-registers-window.png )

这样就可以看到寄存器的值了。


## VS 退出调试模式时崩溃

有时候用 VC 进行调试，对调试的各种窗口 (比如 Output, Local 等窗口) 进行拖动后，再点击停止调试按钮时，VS 会崩溃。重新启动 VS 并进入调试模式之后，各种调试窗口会恢复会默认设置的样子，很难看，再次尝试拖动调试窗口 (比如 Output) 到 VS 底部，点击停止调试按钮，又会出现崩溃。

通过 WinDBG 跟踪，发现是在调用一个 SaveUI 的函数时崩溃的。怀疑是保存界面设置的时候出了问题。

网上搜了一圈没有答案，我试了一下，只有在把一些调试窗口拖动到 VC 底部的时候会触发这个崩溃。

要避免崩溃，只能在进入调试模式后，不拖动任何调试窗口，可这样就太难看了。

最后手动试出了一个方法：

进入调试模式，把所有调试相关的窗口都关掉 (Output, BreakPoint, Local 等)，然后从菜单里打开这些窗口，把他们拖动到合适的位置。再点击停止调试按钮，这样就不会崩溃了，并且界面设置也能够保存。


## WinDBG 的 x 命令，列出符合条件的符号

有的时候需要在 WinDBG 里打断点，但是记不清楚函数的名字，这时候可以用 x 命令来列举出符合条件的符号：

```
x [选项] 模块名字!符号匹配表达式
```

这里的符号匹配表达式类似于 dos 里的文件名匹配表达式，可以使用 * 和 ? 作为通配符，比如：

```
x user32!GetWindowT*
```

可以列出 user32.dll 中所有以 GetWindowT 开头的符号。

不过要注意使用这个命令时，符号必须已经加载到 WinDBG 中，否则会出现如下错误：

```
0:000> x CTest!*
              ^ Couldn't resolve 'x CTest'
```

这表示 x 命令找不到该模块。


## VisualAssistX 导致 VS2008 显示有残影的解决办法

有时候用 VS2008 开发，代码编辑区会莫名其妙变得有残影：

![]( {{site.url}}/asset/debugging-tips-vax-bug.png )

尝试了一下发现是 VisualAssistX 的问题，把 VisualAssistX 插件禁用后再启用一下就可以了：

![]( {{site.url}}/asset/software-debugging-tips-vax-disable.png )


## 64 位系统下抓取 32 位程序 dump 的问题

一般让用户或技术支持抓取 dump 的时候，都是通过任务管理器上右键选择 “创建转储文件” 来完成的：

![]( {{site.url}}/asset/debugging-tips-get-dump.png )

在 64 系统下抓取 32 位程序的时候，这种方法会有问题：

![]( {{site.url}}/asset/debugging-tips-incorrect-dump.png )

可以看到堆栈上是 wow64cpu 这个模块的内容，并不是我们需要的内容。因为我们得到的是一个 64 位地址空间的 dump 文件，可以看到上述堆栈中的地址都是用 64 位来表示的。

如果需要在 64 位系统上正确抓取 32 位程序的 dump, 应该使用 `C:\Windows\SysWOW64\taskmgr.exe` 这个任务管理器来进行，它是一个 32 位程序。