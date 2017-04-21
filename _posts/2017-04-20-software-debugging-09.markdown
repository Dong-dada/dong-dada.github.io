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


## 未处理异常过滤函数

系统默认提供的异常处理器中，都使用了 UnhandledExceptionFilter 作为过滤表达式。如果说默认的异常处理器是系统处理 “未处理异常” 的地方，那么 UnhandledExceptionFilter 就是选择处理办法的决策者。

### Windows XP 之前

以下是 Windows2000 和 NT 的 UnhandledExceptionFilter 函数执行流程：
- 首先 UnhandledExceptionFilter 的参数是 `EXCEPTION_POINTERS` 结构，其返回值为 `EXCEPTION_CONTINUE_SEARCH`, `EXCEPTION_CONTINUE_EXECUTION`, `EXCEPTION_EXECUTE_HANDLER` 这三个常量之一，它们分别表示 继续搜索异常处理器、异常已经处理，继续执行、执行异常处理器；
- 第一步，如果参数中的异常代码为 `EXCEPTION_ACCESS_VIOLATION`(非法访问)，并且导致异常的原因是写操作，那么调用 BasepCheckForReadOnlyResource 函数进行处理，该函数会根据地址找到非法访问的内存块，然后判断内存块的类型，如果是执行映像(`MEM_IMAGE`), 那么它会返回 `EXCEPTION_CONTINUE_SEARCH`；如果是资源块，那么该函数会尝试将该内存块的属性修改为可写，如果修改成功，那么返回 `EXCEPTION_CONTINUE_EXECUTION` 让程序继续执行；
- 第二步，如果当前进程正在被调试，那么返回 `EXCEPTION_CONTINUE_SEARCH`，当 KiUserExceptionDispatcher 收到此返回值后，会通过 ZwRaiseException 来调用 KiDispatchException 进行第二轮异常分发。因为进程正在被调试，第二轮会先分发给调试器。对第二轮处理机会，调试器的典型处理是返回 `DBG_CONTINUE`，也就是恢复执行，但因为错误情况没有消除，所以异常通常会再次发生，我们看到的现象就是有问题的代码反复执行；
- 第三步，如果函数指针 BasepCurrentTopLevelFilter 不为空，便调用它。这个函数指针是定义在 kernel32.dll 中的一个全局变量(当前进程范围内)，用户可以通过 SetUnhandledExceptionFilter API 来设置它。通过这一机制，程序可以设置自己的一个用于处理未处理异常的调用函数；
- 第四步，如果当前进程的 ErrorMode 中包含 `SEM_NOGPFAULTERRORBOX` 标志，就返回 `EXCEPTION_EXECUTE_HANDLER`，也就是执行异常处理块。之前已经介绍过，异常处理块的操作一般是退出进程，因此这里会导致进程退出；
- 第五步，整理异常信息和读取注册表中的 AeDebug 设置，这个我们会在下一节详细介绍；
- 第六步，如果 JIT 调试器设置了自动启动，那么就直接启动 JIT 调试器。否则通过系统的 HardError 机制弹出一个对话框来询问用户是否启动 JIT 调试(或者结束进程)；

### Windows XP

Windows XP 对未处理异常的处理办法做了改进，引入了新的报告应用程序错误的界面和方法，支持通过网络发送报告，并加入了对应用程序验证机制的支持。

与之前相比，主要的不同之处有以下几点：
- 第一，新增了对应用程序验证机制的支持，对 `STATUS_ACCESS_VIOLATION` 和 `STATUS_INVALID_HANDLE` 异常，如果当前进程处于调试状态，则调用 RtlApplicationVerifierStop 函数，对异常信息进行打印以供调试使用。简单来说，Windows XP 的 UnhandledExceptionFilter 函数对调试上述两个异常做了特别的支持；
- 第二，增加了新的方法来提示应用程序错误，XP 会首先尝试通过 faultrep.dll 中的 ReportFault 函数启动 DWWIN.EXE 来提示 “应用程序错误”。如果失败，再使用以前的 HardError 机制来提示错误；


## 应用程序错误对话框

Windows 检测到程序内发生了未处理异常或者其他严重错误时，系统的策略是将其终止，在终止前，系统通常会弹出一个对话框来提示用户，这个对话框被称为应用程序错误对话框(Application Fault Dialog)或者叫 GPF 错误框，GPF 是 General Protection Fault (通用保护错误) 的缩写。

错误对话框上有几个按钮，用来征求用户的处理意见，是立刻终止、启动 JIT 调试器调试程序，还是先发送错误报告然后再终止(Windows XP 引入)。

根据上一节的介绍，GPF 对话框是由过滤器函数 UnhandledExceptionFilter 来触发的。考虑到显示对话框时，程序本身已经发生了严重错误，所以系统是使用其他进程来弹出这个对话框的。具体是哪个进程根据版本而有所不同，在 XP 以前是通过 HardError 机制，在 XP 中是使用 ReportFault API，在 Vista 中是 WER 2.0 API。

### 用 HardError 机制提示应用程序错误

简单地说，就是调用 NtRaiseHardError 内核服务。

经过多次负责 HardError 处理和分发的内核函数的依次处理，系统把 HardError 的提示请求通过 LPC 端口发给 Windows 子系统进程 (CSRSS.EXE)。CSRSS 会根据 NtRaiseHardError 的参数，弹出如下对话框：

![]( {{ site.url }}/asset/software-debugging-hardware-dialog.png )

如果点击确定按钮，那么 UnhandledExceptionFilter 会返回 `EXCEPTION_EXECUTE_HANDLER` 给默认的异常处理器，执行异常处理流程，结束进程。

如果点击取消按钮，那么 UnhandledExceptionFilter 会启动注册在系统中的 JIT 调试器，如果系统中没有任何 JIT 调试器，那么不会有取消按钮。

### 用 ReportFault API 提示应用程序错误

Windows XP 会优先考虑使用 ReportFalut API 启动一个新的程序来提示应用程序错误，如果该机制失败，则仍使用 HardError 机制。

UnhandledExceptionFilter 需要弹出错误对话框后，它首先拼出 FALUTREP.DLL 的全路径。FAULTREP.DLL 是 Windows XP 引入的专门用于错误提示的系统模块。它受到系统文件保护机制(System File Protection, 简称 SFP)的保护，因此，假如你用一个同名文件替换了它，系统会很快把它恢复回来。

UnhandledExceptionFilter 首先会加载 FAULTREP.DLL 然后通过 GetProcAddress 获取 ReportFault 函数，其原型如下：

```cpp
EFaultRepRetVal APIENTRY ReportFault(LPEXCEPTION_POINTERS pep, DWORD dwOpt);
```

接着，UnhandledExceptionFilter 函数通过动态取得的函数指针调用 ReportFault 函数。ReportFault 会读取注册表项，以获取当前的错误提示设置，生成记录文件。

如果一切顺利，ReportFault 会调用另一个位于 FAULTREP.DLL 中的函数 StartDWException。它会在临时目录里创建一个 appcompat.txt 文件，然后启动 DWWIN.EXE 程序，从而创建出 Windows XP 风格的应用程序错误对话框：

![]( {{ site.url }}/asset/software-debugging-reportfault-dialog.png )

与之前介绍的 HardError 方式弹出的对话框相比，这个对话框具有如下不同：

新的对话框增加了 Send Error Report 按钮，如果用户选择该按钮，那么程序将发送信息给微软公司或者本企业的用来收集错误信息的专用服务器。不论错误报告是否发送成功，StartDwException 都会返回 ffrvOk 给 ReportFault 函数，ReportFault 再将这个返回值传递给 UnhandledExceptionHandler，收到之后会返回 `EXCEPTION_EXECUTE_HANDLER`，告知异常处理器执行异常处理块。如果用户选择的是 Don't Sent 按钮，那么 StartDWException 会直接返回 ffrvOk。也就是说，不论用户是否发送错误报告，异常处理块都会执行。

Windows Vista 改为通过新引入的 WER 系统服务来提示应用程序错误。WER 系统服务收到请求后会启动一个名为 WerFault.exe 的程序来显示错误对话框。
