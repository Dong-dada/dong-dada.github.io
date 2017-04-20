---
layout: post
title:  "《软件调试》 学习 09 未处理异常和 JIT 调试"
date:   2017-04-20 10:40:30 +0800
categories: debugging
---

* TOC
{:toc}

本章的前半部分将详细介绍 Windows 系统对 “未处理异常” 的处置方法和过程，包括默认的异常处理函数、UnhandledExcepitonFilter 函数、应用程序错误对话框。后半部分将介绍与未处理异常密切相关的 JTI 调试、顶层过滤函数、系统自带的 JIT 调试器—— Dr.Watson 程序、 Dr.Watson 产生的日志文件、用户态转储文件。

## 简介

如果异常发生后，系统问遍所有异常处理器，都说“处理不了”，那么这个异常就被称为 “未处理异常”。根据程序的运行模式，发生在驱动程序等内核态模块中的未处理异常被称为内核态的未处理异常，发生在应用程序中的未处理异常被称为用户态的未处理异常。

回忆上一章介绍的异常分发流程，系统只会在第一轮分发的时候，把异常分发给用户代码注册的异常处理器，第二轮分发时并不会这么做。因此对于 SEH 或 VEH 异常处理器来说，只有一轮处理异常的机会。如果在这一轮异常分发的过程中没有处理，那么它便会称为未处理异常。进入到第二轮分发的都是未处理异常。

对于用户态的未处理异常，Windows 会使用系统登记的默认异常处理函数来处理。事实上，Windows 为应用程序的每个线程都设置了默认的 SEH 异常处理器。

对于内核态的未处理异常，如果有内核调试器存在，则系统 (KiDispatchException) 会给调试器第二次处理机会，如果调试器没有处理该异常，或者根本没有调试器，那么系统就会调用 KeBugCheckEx 发起蓝屏机制，报告错误并停止整个系统。

## 默认的异常处理器

对于典型的 Windows 应用程序，系统会为它的每个线程登记一个默认的异常处理器 (SEH)。此外，编译器在编译时插入的启动函数通常也会注册一个 SEH 处理器。对于使用 C 运行库的程序，C 运行库包含了基于信号的异常处理机制。下面分别加以介绍。

### BaseProcessStart 函数中的 SEH 处理器

我们先来看一下创建进程的几个阶段：
1. 打开要执行的程序映像文件，创建 section 对象用于将文件映射到内存中；
2. 建立进程运行所需的各种数据结构 (EPROCESS, KPROCESS 及 PEB) 和地址空间；
3. 调用 NtCreateThread, 创建处于挂起状态的初始线程，将用户态的起始地址存储在 ETHREAD 结构中；
4. 通知 Windows 子系统注册新的进程；
5. 开始执行初始线程；
6. 在新进程的上下文 (context) 中对进程做最后的初始化工作；

其中第三点，决定了初始线程的起始地址。Windows 程序的 PE 文件中登记了程序的入口地址，即 `IMAGE_OPTIONAL_HEADER` 结构的 AddressOfEntryPoint 字段。但是在创建进程时，系统通常不会把这个地址用作新线程的起始地址，而是把起始地址指向 kernel32.dll 中的进程启动函数 BaseProcessStart, 这样做的原因之一，是要注册一个默认的 SEH 处理器：

```cpp
VOID BaseProcessStart(PPROCESS_START_ROUTINE lpfnEntryPoint)
{
    __try
    {
        NtSetInformationThread(
            GetCurrentThread(), 
            ThreadQuerySetWin32StartAddress,
            &lpfnEntryPoint, 
            sizeof(lpfnEntryPoint));
        
        // 调用登记的入口函数
        int ret = lpfnEntryPoint();

        // 函数执行结束，退出线程
        ExitThread(ret);
    }
    __except(UnhandledExceptionFilter(GetExceptionInformation()))
    {
        ExitThread(GetExceptionCode());
    }
}
```

从上述代码可以看出，BaseProcessStart 会先设置一个 SEH 处理器，然后再进入真正的入口函数。这个 SEH 是程序最早注册的异常处理器，因为 RtlDispatchException 是从最晚注册的异常处理器来查找的，所以这个 SEH 会最晚得到处理机会。

### 编译器插入的 SEH 处理器

BaseProcessStart 中插入的 lpfnEntryPoint 是否是应用程序的 main 函数地址呢？答案是否定的。

因为在执行 main 函数之前，还有许多准备工作要做，比如初始化 C 运行库、初始化全局变量、初始化 C 运行库所使用的堆、准备命令行参数等。为了做这些准备工作，操作系统通常会把自身提供的一个启动函数登记为程序的入口，让系统先执行这个函数，在这个启动函数内部再去调用 main 函数。

VC 编译器总是将自己提供的以下函数之一登记为程序的入口：
- mainCRTStartup : 非 Unicode 的控制台程序；
- wmainCRTStartup : Unicode 的控制台程序；
- WinMainCRTStartup : 非 Unicode 的 Win32 程序；
- wWinMainCRTStartup : Unicode 的 Win32 程序；

我们随便挑一个，看一下它的伪代码：

```cpp
void mainCRTStartup()
{
    int mainret;

    __try
    {
        _cinit();

        mainret = main(__argc, __argv, __environ)

        // 正常退出
        exit(mainret)
    }
    __except(_XcptFilter(GetExceptionCode(), GetExceptionInformation()))
    {
        // 异常退出
        _exit(GetExceptionCode());
    }
}
```

从上述伪代码可以看出，C 运行库的入口函数中也包含了一个 SEH 处理器，它的保护范围中包含了对 main 或 WinMain 函数的调用。这个 SEH 是应用程序初始线程的第二个异常处理器(倒数第二个得到处理机会)。

### 基于信号的异常处理

上一小节的代码中有一个 `_XcptFilter` 函数作为过滤表达式。这个函数的作用是什么呢？

在 UNIX 系统中，通常使用 C 风格的信号处理机制来处理异常，当有异常发生时，系统通过检查一张专门的异常应对表来寻找异常处理函数，为了便于把这种机制的软件移植到 Windows 系统中，Windows 的 C 运行库包含了对这种异常机制的支持。

`_XcptFilter` 会首先从异常应对表中查找是否存在对应的异常处理函数，如果没有，那么就会调用 UnhandledExceptionFilter。

这种兼容措施对于大多数 Win32 程序来说已经不起什么作用了。从上面的介绍中可以看到，两个 SEH 处理器：一个是 BaseProcessStart 函数设置的，一个是 C 运行库的启动函数设置的，它们的过滤表达式都直接或间接地调用了 UnhandledExceptionFilter 函数。

### BaseThreadStart 函数中的 SEH 处理器

对于初始线程之外的其他线程，其用户态的起始地址是系统提供的另一个位于 kernel32.dll 中的函数 BaseThreadStart，下面给出了该函数的伪代码：

```cpp
VOID BaseThreadStart(
    LPTHREAD_START_ROUTINE lpStartAddress,
    LPVOID lpParameter)
{
    __try
    {
        int ret = lpStartAddress(lpParameter);
        ExitThread(ret);
    }
    __except(UnhandledExceptionFilter(GetExceptionInformation()))
    {
        if (!BaseRunningInServerProcess)
        {
            ExitProcess(GetExceptionCode());
        }
        else
        {
            ExitThread(GetExceptionCode());
        }
    }
}
```

显而易见，系统为 BaseThreadStart 也提供了一个默认的 SEH 处理器。