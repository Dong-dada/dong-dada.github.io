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