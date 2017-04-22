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


## JIT 调试和 Dr.Watson

所谓 JIT 调试(Just-In-Time Debugging)，就是指在应用程序出现严重错误后而启动的紧急调试。因为 JIT 调试建立时，被调试的应用程序内已经发生了严重错误，通常都无法恢复执行，所以 JIT 调试又被称为事后调试(Postmortem Debugging)。

JIT 调试的主要目的是分析和定位错误原因，或者收集和记录错误发生时的现场数据供事后分析。很多调试器都可以作为 JIT 调试器来使用，比如 WinDBG, CDB, NTSD, Visual C++, Dr.Watson 等。其中 Dr.Watson 是 Windows 系统中默认的 JIT 调试器。

### 配置 JIT 调试器

关于 JIT 调试器的配置信息被保存在注册表的如下表键中：

```
HKEY_LOCAL_MACHINE\SOFTWARE\Microsoft\Windows NT\CurrentVersion\AeDebug
```

![]( {{ site.url }}/asset/software-debugging-jit-aedebug.png)

它包含了如下几个键值：
- Debugger : 定义启动调试器的命令行；
- Auto : 是否自动启动调试器；
- UserDebuggerHotKey : 用来定义中止到调试器的热键

### 启动 JIT 调试器

在之前的章节中，我们介绍了建立调试会话的两种情况，分别是在调试器中启动被调试进程、将调试器附加在被调试进程中， JIT 调试显然属于后者。这意味着 UnhandledExceptionFilter 必须将被调试进程的进程 ID 告诉 JIT 调试器。因此 UnhandledExceptionFilter 函数要求 Debugger 选项的值应该为如下格式：

```
jitdebugger.exe -p %ld -e %ld [调试器的其他参数]
```

UnhandledExceptionFilter 会将进程 ID(第一个 %ld) 和一个事件量句柄(第二个 %ld)作为命令行参数传给 JIT 调试器。

接下来 UnhandledExceptionFilter 会通过 CreateProcess 来创建调试器进程，接着无限期等待事件量。以便调试器能够完成初始化等准备工作。

当 JIT 调试器准备完毕并设置事件量后，UnhandledExceptionFilter 便会返回 `EXECPTION_CONTINUE_SEARCH` 让系统继续搜索其他异常处理器，如果当前异常处理器不是最后一个 SEH，那么系统会评估另外一个 SEH 过滤函数，通常也是 UnhandledExceptionFilter, 而且也会返回 `EXCEPTION_CONTINUE_SEARCH`，这样第一轮分发就结束了，然后发起第二轮分发。此时因为 JIT 调试器已经准备好，所以系统会发异常分发给 JIT 调试器。

Debugger 选项的默认值为 `drwtsn32 -p %ld -e %ld -g` 也就是使用 Dr.Watson 作为 JIT 调试器。你也可以指定其他调试器作为 JIT 调试器，例如 `c:\windbg\windbg.exe -p %ld -e %ld -g` 将 WinDBG 指定为 JIT 调试器。


## 顶层异常过滤函数

前面三节我们介绍了 Windows 系统对未处理异常的处理方法。如果应用程序不希望使用这些默认逻辑来处理异常，那么可以注册一个自己的未处理异常过滤函数，并通过 SetUnhandledExceptionFilter API 进行注册。因为这个过滤函数只有未处理异常发生时才会被调用，所以通常被称为 顶层异常过滤函数 (Top Level Exception Filter)。

### 注册

在 Windows XP 以前，SetUnhandledExceptionFilter API 的实现非常简单，它先把 BasepCurrentTopLevelFilter 的值保存起来作为返回值，然后再把新的值赋值给它。BasepCurrentTopLevelFilter 是 KERNEL32.DLL 中定义的一个全局变量，默认值为空。

为了防止 BasepCurrentTopLevelFilter 的值被篡改，从 XP 开始，这个值经过了编码，并且编解码方式不公开。

### C 运行时库的顶层过滤函数

如果当前进程直接或间接调用了微软的 C 运行时库 (MSVCRT.DLL)，那么 C 运行时库在初始化期间会调用 SetUnhanldedExceptionFilter 将当前进程的顶层过滤函数设置为 `msvcrt!__CxxUnhandledExceptionFilter` 函数：

```cpp
// 伪代码

#define CXX_FRAME_MAGIC 0x19930520
#define CXX_EXCEPTION 0xe06d7363

LONG WINAPI CxxUnhandledExceptionFilter(struct _EXCEPTION_POINTERS* ExceptionInfo)
{
    PEXCEPTION_RECORD pER;
    pER = ExceptionInfo->ExceptionRecord;

    if (pER->ExceptionCode == CXX_EXCEPTION && pER->NumberParameters == 3 && pER->ExceptionInformation[0] == CXX_FRAME_MAGIC)
    {
        terminate();
    }

    if (UnDecorator::fGetTempltateArgumentList)
    {
        if (_ValidateExecute(UnDecorator::fGetTemplateArgumentList) != 0)
        {
            return FUNC_UNK(ExceptionInfo);
        }
    }
    return EXCEPTION_CONTINUE_SEARCH;
}
```

因为 VC 编译器实现的 C++ 异常都有统一的异常代码 0xe06d7363 且参数为 0x19930520, 所以 CxxUnhandledExceptionFilter 可以判断出未处理异常是否是 C++ 异常。如果是，那么它会调用 VC++ 的 terminate() 来进行必要的清理工作。

对于其他异常，CxxUnhandledExceptionFilter 会返回 `EXCEPTION_CONTINUE_SEARCH` 使 UnhandledExceptinFilter 继续向下执行，如果 CxxUnhandledExceptionFilter 返回了 `EXCEPTION_EXECUTE_HANDLER` 或 `EXCEPTION_CONTINUE_EXECUTION` 那么 UnhandledExceptionFilter 会立刻返回。

### 执行

根据我们之前的介绍，可以了解顶层过滤函数被调用有两个前提：有未处理异常发生，而且所在程序未处于调试状态。

在以上两个条件都满足后，是不是我们注册的顶层过滤函数就一定会被调用呢？答案是否定的。因为系统是用一个全局变量而不是一个链表来存储顶层过滤函数的，这意味着可能有其他程序在我们之后调用了 SetUnhandledExceptionFilter，将我们设置的过滤函数覆盖掉。

理论上来说，如果每个模块都能在调用 SetUnhandledExceptinFilter 成功后，把上一个过滤函数记录下来，并在自己的过滤函数处理完之后调用前一个过滤函数，那么每个过滤函数就都能被调用了。但实际上很多顶层过滤函数都没有这么做，为了确保自己的过滤函数能够被调用，某些软件使用了非常不好的做法，比如反复调用 SetUnhandledExceptionFilter，甚至有的软件在自己注册成功后会修改 SetUnhandledExceptionFilter 的 API，使其他模块再也无法成功调用这个 API。


## Dr.Watson

Dr.Watson(华生医生) 本来是福尔摩斯探案集中的人物。在 Windows 中， Dr.Watson 是以下几个程序的别称：

- DRWATSON.EXE : 16 位版本的 Windows 中收集和记录错误的小工具；
- DRWTSN32.EXE : 32 位版本的 Dr.Watson 程序。是系统中默认的 JIT 调试器，具有生成错误报告、产生内存转储文件等功能；
- DWWIN.EXE : Windows XP 引入的提示应用程序错误和发送错误报告的工具。Windows XP 中的 “应用程序错误” 对话框便是由该程序弹出的，该程序还负责通过网络将错误报告发送到服务器；

DRWATSON.EXE 已经过时，DWWIN.EXE 的主要目的是提示和报告错误，本书的 Dr.Watson 是指与调试关系更为密切的 DRWTSN32.EXE 程序。

DRWTSN32.EXE 的核心功能是当应用程序出现错误时，以 JIT 调试器的身份收集错误信息产生记录文件和错误报告。

### 配置和查看模式

当不带任何参数直接运行 DRWTSN32.EXE 时，它会显示如下的界面：

![]( {{ site.url }}/asset/software-debugging-drwtsn32.png )

该界面可以对 DRWTSN32.EXE 进行配置，其配置信息存储在注册表的如下键中：

```
HKEY_LOCAL_MACHINE\SOFTWARE\Microsoft\DrWatson
```

具体的配置就不介绍了，主要是对日志文件、故障转储文件的配置。

界面中还有一个编辑框区域，这里会展示上次出现的错误记录。

值得一提的是我的 Windows7 下没有这个程序，可能是被移除了。

### 安装为 JIT 调试器模式

通过修改注册表键值，或者调用 `drwtsn32.exe -i` 命令，可以将 DRWTSN32.EXE 安装为 JIT 调试器。

在 JIT 调试器模式下，DRWTSN32.EXE 会执行以下动作：

- 第一，如果 VisualNotification 配置为 1，则弹出对话框，提示某程序发生了错误，DRWTSN32.EXE 正在产生错误日志；
- 第二，如果 SoundNotification 配置为 1，则播放 WaveFile 选项指定的波形文件；
- 第三，向系统日志写入错误记录。使用系统日志观察工具可以观察这些记录(我的电脑-->管理-->系统工具-->事件查看器-->应用程序)；
- 第四，如果 CreateCrashDump 为 1，则生成故障转储文件；
- 第五，生成日志文件，文件名称为 drwtsn32.log, 默认位置为 `Documents and Settings\AllUsers\Application Data\Microsoft\Dr Watson`


## DRWTSN32.EXE 的日志文件

drwtsn32.log 中记录了许多错误信息，包括异常信息、系统信息、模块列表、线程状态、函数调用序列、原始栈数据 等信息。这里不做介绍，用到的时候再看吧。


## 用户态转储文件

简单地说，用户态转储文件 (User Mode Dump) 就是用于保存应用程序在某一时刻运行状态的二进制文件。与描述整个系统状态的系统转储文件相比，用户态转储文件的描述范围仅限于用户进程，二者的格式也是不同的。

一些文档中用 minidump 来指代用户态转储文件，其实也可以生成很大的用户态转储文件。

### 文件格式概览

为了方便生成和读取，转储文件的格式非常简单，主要由四个部分组成。第一部分是文件头，他是一个固定格式的数据结构 (`MINIDUMP_HEADER`)。第二部分是目录表，目录表中的每个目录项是一个固定长度的名为 `MINIDUMP_DIRECTORY` 的结构，用来描述一个数据流。第三部分便是一个个的数据流。第四部分是不定数量的内存块。下图画出了以上三种数据的布局示意图：

![]( {{ site.url }}/asset/software-debugging-minidump-format.png)

### 数据流

文件头和目录表的意思很明确，这里就不介绍了。下面着重介绍一下数据流的格式。

数据流的格式和长度是与类型有关的，这些信息会记录在目录表中。下图给出了目前已经定义的数据流类型和每种数据流的格式：

![]( {{ site.url }}/asset/software-debugging-minidump-datastream.png )

可以看到，其中包含了线程列表、模块列表、内存列表、异常信息、系统信息、增强线程列表、注释、句柄数据、函数表、卸载模块表、线程信息、内存块信息、句柄操作记录、用户数据等多种信息。

### 产生转储文件

系统提供了 MiniDumpWriteDump() API 来生成转储文件：

```cpp
BOOL WINAPI MiniDumpWriteDump(
  __in          HANDLE hProcess,
  __in          DWORD ProcessId,
  __in          HANDLE hFile,
  __in          MINIDUMP_TYPE DumpType,
  __in          PMINIDUMP_EXCEPTION_INFORMATION ExceptionParam,
  __in          PMINIDUMP_USER_STREAM_INFORMATION UserStreamParam,
  __in          PMINIDUMP_CALLBACK_INFORMATION CallbackParam
);
```

DumpType 参数可以指定写入信息的类型和选项：

![]( {{ site.url }}/asset/software-debugging-minidump-dumptype.png )

### 读取转储文件

MiniDumpReadDumpStream() API  用来读取转储文件：

```cpp
BOOL WINAPI MiniDumpReadDumpStream(
  __in          PVOID BaseOfDump,
  __in          ULONG StreamNumber,
  __out         PMINIDUMP_DIRECTORY* Dir,
  __out         PVOID* StreamPointer,
  __out         ULONG* StreamSize
);
```

### 利用转储文件分析问题

使用 WinDbg 打开一个 dump 文件。打开后，WinDBG 便会显示转储文件的概要信息、注释信息和异常信息：

![]( {{ site.url }}/asset/software-debugging-windbg-open-dump.png )

如果需要进一步定位异常的发生位置，需要导入符号文件，我们可以将 UEF 示例程序的符号文件所在目录设置到 WinDBG 的符号路径中 (File-->Symbol File Path)，然后执行 .reload 指令。

接下来通过 .excr 指令让 WinDBG 显示异常信息：

![]( {{site.url}}/asset/software-debugging-windbg-add-symbol-path.png)

这次可以看到异常发生在 main 函数入口附近。

使用 `~` 指令可以显示线程信息，通过 `~<线程号> s` 指令可以切换当前线程：

![]( {{ site.url }}/asset/software-debugging-windbg-thread-command.png )

使用 `lm` 指令可以显示模块信息，使用 `kv` 指令可以观察栈回溯信息，使用 `!handle` 指令可以观察句柄信息：

![]( {{ site.url }}/asset/software-debugging-windbg-lm-kv.png )

