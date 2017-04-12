---
layout: post
title:  "《软件调试》 学习 07 用户态调试过程"
date:   2017-04-12 10:00:30 +0800
categories: debugging
---

* TOC
{:toc}

上一章我们介绍了 Windows 操作系统中用于支持用户态调试的各种基础设施，描述了这些设施的静态特征和功能。本章将讨论这些设施的动态特征，解析 Windows 系统中用户态调试的关键过程，特别是 调试器、被调试程序、及调试子系统这三者是如何相互配合完成各种调试功能的。

## 调试器进程

调试器进程和被调试进程是调试过程中的两个主角。从用户的角度看，调试过程就是使用调试器进程来控制和观察被调试进程的过程。

调试器的主要功能分成如下两个方面：
- 人机接口，以某种界面形式将调试功能展现给用户，并监听和接受用户的输入，收到用户输入后，进行解析和执行，然后把执行结果返回给用户；
- 与被调试进程交互，包括与被调试进程建立调试关系，然后监听和处理调试事件，根据需要将被调试进程中断到调试器，读取和修改被调试进程的数据，或者操纵它的行为。根据上一节的介绍，调试器主要是通过调试子系统与被调试进程交互的，但从用户的角度看，可以认为调试器是直接与被调试程序交互的。

调试器通常会有两个线程，每个线程负责一个方面的对话。负责人机交互的称为 UI 线程，负责与被调试进程交互的称为调试器工作线程(Debugger's Worker Thread)，简称 DWT。

调试器工作线程的核心代码如下：

```cpp
if (bNewProcess)
{
    // 如果是以调试模式创建新进程，则调用 CreateProcess 并传入 DEBUG_PROCESS 标记
    CreateProcess(..., DEBUG_PROCESS, ...);
}
else
{
    // 如果是附加到已经存在的进程，则调用 DebugActiveProcess 并传入进程 id
    DebugActiveProcess(dwPID);
}

while ( 1 == WaitForDebugEvent(&DbgEvt, INFINITE) )
{
    switch(DbgEvt.dwDebugEventCode)
    {
        case EXIT_PROCESS_DEBUG_EVENT:
            break;
        // 其他情况
    }

    ContinueDebugEvent(...);
}
```


## 被调试进程

为了不影响问题的重现和分析结果，调试过程本身应该尽可能少地改变被调试进程的属性。但为了实现某些调试功能，系统不得不修改被调试进程的某些属性或者在其中执行一些用于调试的代码。概括地说，一个处于被调试状态的 Windows 进程与普通进程相比，会有如下差异：
- 进程执行块(Executive Process Block, 即 EPROCESS 结构)的 DebugPort 字段不为空。这是在内核空间中判断一个进程是否正在被调试的主要特征。
- 进程环境块(Process Environment Block, 简称 PEB) 的 BeingDebugged 字段不等于 0。这是用户态判断一个进程是否正在被调试的主要方法。
- 可能会存在一个由调试器远程启动的线程，这个线程的作用是将被调试进程中断到调试器，我们称之为 RemoteBreakin 线程；

### DebugPort 字段

我们之前简要介绍过进程执行块 EPROCESS，它是所有 Windows 进程都拥有的一个数据结构，EPROCESS 位于内核空间中，是系统用来标识和管理每个 Windows 进程的基本数据结构。EPROCESS 结构的具体定义因 Windows 的版本不同而不同，但是都包含一个有关调试的重要字段，即 DebugPort。如果一个进程没有被调试，则这个字段为 NULL，否则 DebugPort 字段是一个指针，在 XP 及以后的操作系统中，它指向一个 DebugObject 内核对象。

### BeingDebugged 字段

进程环境块是每个 Windows 进程的另一个重要数据结构，与 EPROCESS 不同，PEB 结构是位于用户空间中的，而其地址通常是位于用户空间的较高地址区域，例如 0x7FFDF000：

```cpp
typedef struct _PEB
{
    BOOLEAN InheritedAddressSpace;
    BOOLEAN ReadImageFileExecOptions;
    BOOLEAN BeingDebugged;
} PEB, *PPEB, **PPPEB;
```

如果一个进程不在被调试状态，那么 BeingDebugged 字段为 0，否则为 1，IsDebuggerPresent() API 就是通过判断 BeingDebugged 字段实现的。

### 调试会话

我们把调试器进程与被调试进程之间的交互称为调试会话(debugging session)。一次调试会话是从建立调试关系开始，直到这种关系解除为止。更准确地说，建立调试关系的标准是被调试进程与调试器进程之间通过调试端口建立的通信连接。调试关系解除的标准是被调试进程的调试端口被清除。

建立调试会话有两种方式，一种是在调试器中启动被调试程序。另一种是当开始调试时，被调试程序已经在运行，这时需要把调试器附加 (attach) 到被调试进程上。


## 从调试器中启动被调试程序

这种方式下，只需要在创建进程时指定 `DEBUG_PROCESS` 或 `DEBUG_ONLY_THIS_PROCESS` 标志就可以了。

### CreateProcess API

如果在 `CreateProcess` 的 `dwCreationFlags` 参数中指定了下述两个标记，那么系统就会把调用进程当做调试器进程，把新创建的进程当做被调试进程，为二者建立起调试关系：

```cpp
#define DEBUG_PROCESS               0x00000001      // 调试正在创建的进程和它的子进程
#define DEBUG_ONLY_THIS_PROCESS     0x00000002      // 声明不要调试调试目标的子进程
```

如果创建进程时指定了上述标记，那么系统将执行下述三个动作：

第一：在进程创建的早期调用 DbgUiConnectToDbg() 是调用线程和调用子系统建立连接。在 Windows XP 中，DgbUiConnectToDbg() 内部会调用 ZwCreateDebugObject 创建 `DEBUG_OBJECT` 内核对象，将其保存在当前线程环境块的 DbgSsReserved[1] 字段中。完成这一操作之后，调用线程便由普通线程晋升为调用子系统“眼”里的调试器工作线程了。

第二：当调用进程创建内核服务 NtCreateProcess 或 NtCreateProcessEx 时，将 DbgSsReserved[1] 字段中记录的对象句柄以参数的形式传给内核中的进程管理器。接下来，内核中的进程创建函数 (PspCreateProcess) 会检查这个句柄是否为空，如果不为空，会取得它的对象指针，然后设置到进程执行块 EPROCESS 的 DebugPort 字段中。完成这一步之后，新创建的进程便由普通的进程晋升为调试子系统 “眼” 里的被调试进程了。

第三：当 PspCreateProcess 调用 MmCreatePeb 函数创建新进程的进程环境块时，MmCreatePeb 函数内部会根据 EPROCESS 结构的 DebugPort 字段设置 BeingDebugged 字段，如果 DebugPort 不为空，BeingDebugged 字段会被设为 1；

### 第一批调试事件

以上三个动作执行成功后，调试会话便建立起来了，调试器线程接下来应该进入调试事件循环来接受调试事件。

一个新创建进程的初试线程是从内核中的 KiThreadStartup 开始执行的，KiThreadStartup 将线程的 IRQL 降低到 APC 级别之后，便将执行权交给了 PspUserThreadStartup 函数。

PspUserThreadStartup 函数内部会调用 DbgkCreateThread 向调试子系统通知新线程创建事件。调试子系统会向调试事件队列里放入一个进程创建事件，等待调试器的处理。

接着，新进程会在自己的上下文中执行一系列初始化工作，包括映射和加载映像文件。因此，调试器接下来会收到一系列加载 DLL (`LOAD_DLL_DEBUG_EVENT`) 事件。

### 初始断点

当新进程的初试线程在自己的上下文中初始化时，作为进程初始化的一个步骤，NTDLL.DLL 中的 LpdrInitializeProcess 函数会检查正在初始化的进程是否处于被调试状态，如果是，它会调用 DbgBreakPoint() 触发一个断点异常，目的是中断到调试器。这实际上相当于系统在新进程中为我们设置了一个断点，这个断点通常被称为初始断点。

当初始断点发生时，被调试程序自己的主函数还没有开始执行，因此这个调试时间是很早的，对于调试在程序初始化阶段发生的问题是非常有意义的。

### 自动启动调试器

有时，要调试的进程可能是被系统或者其他进程动态启动的。在调试器中执行它可能无法提供合适的参数和运行条件。而且，当我们发现它启动并且将调试器附加到该进程时，需要调试的代码可能已经运行结束了。对于这种情况，可以通过在注册表中设置 “映像文件执行选项 (Image File Execution Options)” 来让操作系统先启动调试器，然后再从调试器中启动目标进程。

```
HKEY_LOCAL_MACHINE\SOFTWARE\Microsoft\Windows NT\CurrentVersion\Image File Execution Options
```

如果上述注册表中包含要执行的映像文件名（不包含路径）。如果有这样的子键，那么系统会继续检查该子键是否存在命名为 “Debugger” 的键值。如果该键值存在，那么系统就会将当前的命令行附加到 Debugger 键值所定义的内容后，再将此作为新的命令行来启动。


## 附加到已经启动的进程

将调试器附加到已经启动的进程，是通过 DebugActiveProcess() API 来完成的。

DebugActiveProcess API 的原型非常简单，即 `DebugActiveProcess(DWORD dwProcessId)`，只需传入对方的进程 id 即可。下面介绍它的内部工作过程：

第一，通过 DbgUiConnectToDbg() 是调用进程与调试子系统建立连接，实质上就是获得一个调试通信对象并存放在当前线程 TEB 结构的 DbgSsReserved 数组中。这与调试一个新进程的第一步相同；

第二，调用 ProcessIdToHandle() 函数，获取指定进程 ID 的进程句柄；

第三，调用 NTDLL 中的 DbgUiDebugActiveProcess 函数。这个函数内部会调用 NtDebugActiveProcess 内核服务，并将要调试进程的句柄和调试对象句柄作为参数传给这个内核服务。它内部主要做三个动作：(1) 根据参数中指定的句柄取得被调试进程的 EPROCESS 结构和调试对象的对象指针；(2) 想调试对象发送杜撰的调试事件；(3) 调用 DbgkpSetProcessDebugObject 函数，这个函数内部会将调试对象设置到调试进程的调试端口，并调用 DbgkpMarkProcessPeb 来设置 BeingDebugged 字段。

以上操作都成功后，DebugActiveProcess() 会返回真，表示已经建立了调试会话。


## 处理调试事件

上一章我们已经介绍过，Windows 的用户态调试是通过调试事件来驱动的。我们之前已经介绍过调试事件的采集和传递过程，本节我们将介绍调试器是如何读取和处理调试事件的。

### DEBUG_EVENT 结构

在调试 API 一层，Windows 使用名为 DEBUG_EVENT 的结构体来表示调试事件，该结构的定义如下：

![]( {{ site.url }}/asset/software-debugging-debug-event.png )

其中 dwDebugEventCode 用来标识调试事件的类型，其值如下表：

![]( {{ site.url }}/asset/software-debugging-debug-event-code.png )

### WaitForDebugEvent API

Windows 设计了 WaitForDebugEvent API 来供调试器等待和接受调试事件。这个 API 是实现在 Kernel32.dll 中的：

```cpp
BOOL WaitForDebugEvent(LPDEBUG_EVENT lpDebugEvent, DWORD dwMilliSeconds);
```

调用 WaitForDebugEvent 会导致所在线程阻塞，直到有调试事件发生，或等待时间已到或发生错误才返回。

WaitForDebugEvent 内部主要完成两项任务，一是调用 NTDLL.DLL 中的 DbgUiWaitStateChange 函数，二是将这个函数返回的 `DBGUI_WAIT_STATE_CHANGE` 结构体转换为 `DEBUG_EVENT` 结构。

上一章我们介绍过，内核中是使用 `DBGKM_APIMSG` 来描述调试事件的，调试 API 一层是使用 `DEBUG_EVENT` 来描述的，而 DbgUi 函数是使用 `DBGUI_WAIT_STATE_CHANGE` 来描述的。下图显示了这些结构的场合和转换函数：

![]( {{ site.url }}/asset/software-debugging-debug-event-struct.png )

### 调试事件循环

在调试事件循环中，对于等待得到的一个调试事件，调试器通常有以下几种处理方式：

- 什么也不做或者更新内部状态，用户察觉不到有调试事件发生。比如 C++ 异常的第一轮处理机会，调试器的默认行为是什么都不做，让系统继续分发异常。
- 中断给用户，开始交互调试，直到用户发出继续运行命令。例如，当接收到断点异常时，调试器总是会中断给用户；
- 输出调试信息；

### 回复调试事件

调试器处理好调试事件之后，应该调用 ContinueDebugEvent API 来向调试子系统回复处理结果：

```cpp
BOOL ContinueDebugEvent( DWORD dwProcessId, DWORD dwThreadId, DWORD dwContinueStatus);
```

dwProcessId, dwThreadId 也就是 `DEBUG_EVENT` 中包含的进程和线程 ID。dwContinueStatus 可以为 `DBG_CONTINUE` 和 `DBG_EXCEPTION_NOT_HANDLED` 两个常量之一。对于异常事件(`EXCEPTION_DEBUG_EVENT`)之外的所有其他事件，这两个常量没有区别，调试子系统收到之后，都会回复被调试进程继续运行。而对于异常事件来说，这两个值存在差异：

`DBG_CONTINUE` 表示调试器处理了该异常，`DbgkForwardException` 函数（异常事件就是通过它接收并传给调试子系统的）收到这个返回值之后会向它的调用者(KiDispatchException) 返回真，这种情况下 KiDispatchException 会认为调试器已经处理了该异常，于是结束该异常的分发过程；

`DBG_EXCEPTION_NOT_HANDLED` 表示调试器不处理该异常。这会导致 `DbgkForwardException` 函数返回假给 KiDispatchException 函数。KiDispatchException 不会结束异常分发，而是寻找异常处理块。如果第二轮的处理机会，`DbgkForwardException` 又返回了假，那么它会再次调用发给异常端口。如果这次调用也返回假，则终止该进程。


## 中断到调试器


## 输出调试字符串


## 终止调试会话