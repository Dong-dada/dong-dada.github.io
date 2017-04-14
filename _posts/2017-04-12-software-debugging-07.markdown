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

在介绍 DbgkSendApiMessage 函数时，该函数代表调试子系统把调试事件发送给调试器之前会调用 DbgkpSuspendProcess 挂起被调试进程。以防止被调试进程继续运行发生状态变化，给分析和观察带来困难。从被调试进程的角度看，一旦它被调试子系统挂起，那么它便 “戛然而止” 了，代码 (用户态的应用程序代码) 停止执行，一切状态都被冻结器来，在调试领域，我们将这种现象称为 “中断到调试器 (break into debugger)”。从调试器的角度看，又叫做将被调试进程 “拉进调试器 (bring debuggee in)”

下面介绍被调试进程中断到调试器的典型办法，然后介绍几个有关的问题。

### 初始断点

正如之前介绍的，新创建进程的初试线程在初始化时会检查当前进程是否在调试，如果是，那么便调用 NTDLL 中的 BbgBreakPoint 函数，触发一个断点异常，使新进程中断到调试器中。

### 编程时加入断点

Windows 提供了一个用于产生断点异常的 API，其原型非常简单： `void DebugPoint(void);`。当编写程序时，如果希望在某种情况下中断到调试器中，可以加入如下代码：

```cpp
if (IsDebuggerPresent() && <希望中断的附加条件>)
{
    DebugBreak();
}
```

这种方法对于调试某些复杂的多线程问题或随机发生的问题是很有用的，因为可以在应用程序中检测到希望中断到调试器的条件(包括条件断点难以实现的判断条件)，然后中断到调试器中。

事实上，在 x86 平台上，`BreakPoint()` 等价于一条 INT 3 指令，所以直接使用嵌入式汇编也可以达到效果：

```c
__asm{int 3};
```

### 通过调试器设置断点

也就是通过调试器的断点功能向被调试进程动态地插入断点指令，当被调试进程遇到这些断点指令的时候，触发断点异常而中断到调试器。

### 通过远程线程触发断点异常

前面的三种方法都是在程序的固定位置植入断点指令，只有当被调试进程执行到那里时，才会中断到调试器。如果希望被调试进程立刻中断到调试器，比如按下一个热键就会中断下来，那么前面的方法就不适合了。这种根据用户的即时需要而将被调试进程中断到调试器的功能称为异步阻停(Asynchronous Stop)。

实现异步阻停的方法之一是利用 Windows 提供的 CreateRemoteThread API，在被调试进程中创建一个远程线程，让这个线程一运行便执行断点指令，把被调试进程中断到调试器中。

要做到这一点，我们需要被调试进程中有一个包含断点指令的函数，它的原型应当符合 SDK 所定义的线程启动函数原型： `DWORD WINAPI ThreadProc(LPVOID lpParam);`。

事实上，NTDLL.DLL 中已经设计好了这样一个函数，即 DbgUiRemoteBreakin, 它内部会调用 DbgBreakPoint() 执行断点指令，而且是在一个结构化异常保护块(SEH)中做的调用，其伪代码如下：

```cpp
DWORD WINAPI DbgUiRemoteBreakin(LPVOID lpParam)
{
    __try
    {
        if (NtCurrentPeb()->BeingDebugged)
        {
            DbgBreakPoint();
        }
    }
    __except(EXCEPTION_EXECUTE_HANDLER)
    {
        return 1;
    }
    RtlExitUserThread(0);
}
```

使用异常保护块的目的是捕捉断点异常，万一调试器没有处理，那么这个异常处理器会处理它，以防止无人处理而导致整个程序被终止。这样的话，向一个不在被调试的进程中创建远程线程并执行这个函数，并不会触发断点异常。

为了简化上述任务，Windows XP 引入了一个新的 API，叫做 `BOOL DebugBreakProcess(HANDLE Process);`，只需调用这个 API 即可将一个进程中断到调试器中。

包括 WinDBG 在内的很多调试器所提供的 break 功能都使用了远程中断线程。例如，在 WINDBG 中，选择 Debug 菜单的 Break 项，或者按 Ctrl+Break 热键便会发出 break 指令。WinDBG 收到此指令后会通过远程中断线程在被调试进程产生一个端点异常，使其中断到调试器。明白了这个原理后，我们就能理解为什么在被调试器中断后，WinDBG 总是显示如下的内容：

![]( {{ site.url }}/asset/software-debugging-windbg-break.png )

此时的调用栈是在远程线程上的，可以使用线程切换命令来切换到其他线程，比如 `~0 s` 切换到 0 号线程 (初始线程) 中。

### 在线程当前执行位置设置断点

利用远程线程实现异步阻停的一个明显不足，就是要在被调试进程中启动一个新线程，这样会对被调试进程的执行环境有较大的影响，而且可能会干扰被调试进程自身的逻辑。实现这一功能的另一种方式是在被调试进程的现有线程中触发断点。其步骤主要如下：

首先，将被调试进程中的所有线程挂起，这可以使用 SuspendThread 来完成；

然后，取得每个线程的执行上下文(可以调用 GetThreadContext API)，得到线程的程序指针寄存器(PC)的值，并在这个值对应的代码处设置一个断点，也就是把本来的一个字节保存起来，并写入断点指令。这样 CPU 下次执行这个线程时就会立刻出发断点异常而中断到调试器。对进程的每个线程都重复此步骤；

最后，恢复所有线程，这可以使用 ResumeThread 来完成。

VC6 调试器和重构之前的 WinDbg 调试器都是使用以上方法来实现异步阻停的。

因为当用户发出中断指令时，被调试进程通常是在执行某种等待函数(除非它在用户态有特别多的运行任务)，而且大多数等待函数(GetMessage, Sleep, SleepEx)都是调用内核服务而进入内核态执行的，所以调试器取得的 PC 值通常指向的是系统服务返回后将执行的 ret 指令。这个指令地址在 Windows XP 中对应的调试符号是 `ntdll!KiFastSystemCallRet`。

### 动态调用远程函数

与刚才介绍的在程序指针的位置动态增加断点指令类似的一种方法是动态地调用一个函数，让该函数来执行断点指令。Windows 2000 中的 NTDLL.DLL 和 Kernel32.dll 中已经包含了这种方法的实现。简单来说，就是利用 NTDLL 中的 RtlRemoteCall 函数远程调用被调试进程内位于 Kernel32.dll 中的 BaseAttachComplete 函数。

具体来说，应当先将目标线程挂起，或使其锁定在一个稳定的内核状态，然后调用 RtlRemoteCall 函数，并将 Kernel32.dll 中的 BaseAttachCompleteTrunk 的地址作为调用点传给它。

BaseAttachCompleteTrunk 被远程调用后，会直接调用 BaseAttachComplete 函数，而 BaseAttachComplete 中会执行断点指令。

RtlRemoteCall 内部会通过 NtGetContextThread 内核服务取得目标线程的上下文，得到目标线程的栈地址，然后调整栈指针将取得的上下文结构和参数用 NtWriteVirtualMemory 写到目标线程的栈上，而后，将线程上下文结构中的程序指针字段 (EIP) 设置为参数指定的调用点地址 (也就是 BaseAttachCompleteTrunk)。接下来通过 NtSetContextThread 将修改后的 CONTEXT 结构设置回目标线程。最后，调用 NtResumeThread, 恢复目标线程执行，这样它一运行就会执行 BaseAttachCompleteTrunk 的代码。

### 挂起中断

以上三种异步阻停的方法都是希望被调试程序继续执行，然后遇到断点指令就中断到调试器，也就是假定被调试进程依然是可以执行用户态代码的。但是，如果被调试进程因为某种原因不能继续执行用户态代码，那么这三种方式就无法工作了。

针对以上问题，一种替代性的方法是强行将被调试进程的所有线程挂起，然后进入一种准调试状态。之所以叫做准调试，是因为通过这种方式中断到调试器后，不可以使用单步执行等跟踪命令。我们称这种方法为挂起中断 (Breakin by Suspend)。

WinDBG 调试器在使用远程线程中断功能超时后会使用挂起中断方式。如下图所示，WinDBG 在创建远程中断线程后，会等待 30 秒，没有中断的话就使用挂起中断方式。在将所有线程都挂起后，调试器的底层函数会模拟一个唤醒调试器的异常事件，调试器的事件处理函数收到这个事件后就会中断给用户：

![]( {{ site.url }}/asset/software-debugging-breakin-by-suspend.png )

中断后，可以通过切换线程的命令来查看此时线程的调用栈；

### 调试热键 (F12)

当我们使用 WinDBG 调试计算器程序时，除了可以在 WinDBG 中按下 Ctrl+Break 将计算器中断到调试器以外，还可以向计算器程序按下 F12(就是在计算器程序处于前台的时候按下 F12)。

该功能的原理是，Windows 子系统的内核部分受到此热键之后，会通过 LPC 请求 CSRSS 中的 SvrActivateDebugger 服务。

SvrActivateDebugger 会检查要调试的进程是否是自己，并且自己是否处于调试状态，如果是，就调用 DbgBreakPoint 中断到调试器。

可以使用如下注册表键下的 UserDebuggerHotKey 选项，来指定其他按键作为调试热键：

```
HKEY_LOCAL_MACHINE\Software\Microsoft\Windows NT\CurrentVersion\AeDebug
```

### 窗口更新

在被调试进程中断到调试器之后，进程是处于 “停顿” 状态的，直到调试器恢复其运行。在这一阶段，如果被调试程序是窗口程序，那么它的窗口是僵死的，不可移动，不可改变大小，会遮挡住它所对应的桌面区域。为了行文方便，我们把中断到调试器中的进程的窗口称为中断窗口 (Broke Window)。

不同 Windows 版本中中断窗口的表现并不一样。
- 在 Windows 2000 中，系统只会更新中断窗口的非用户区，如果有其他窗口经过了用户区，那么用户区的内容将被擦成背景色；
- 在 Windows XP 中，整个窗口都处于被任意涂抹的状态。如果主窗口在几秒钟之内都没有响应，那么 Windows 会将其视为无响应窗口 (no responding window)，并为其创建一个精灵窗口 (Ghost Window), 用户可以移动这个窗口，也可以通过关闭按钮触发系统终止这个应用程序。


## 输出调试字符串

Windows 提供了 OutputDebugString API 来输出调试字符串。如果程序不在被调试，那么系统调试器会显示该字符串，如果系统也没有在调试状态，那么它什么都不做。

本节介绍 OutputDebugString 的工作原理。

### 发送调试信息

也就是 OutputDebugString 将调试信息发送给调试器的过程。

简单来说，OutputDebugString 是利用 RaiseException() 产生一个特殊的异常，该异常的代码为 `DBG_PRINTEXCEPTION`。我们把它称为调试打印异常 (Debug Print)。

RaiseExcepiton() 中会将调试字符串的地址作为参数。它被调用后，将产生一个标准的异常结构 `EXCEPTION_RECORD`，然后调用内核服务将这个模拟的异常发送到内核中进行分发。

### 使用调试器接收调试信息

内核中的异常分发函数 KiDispatchException 会按照统一的流程来分发调试打印异常。正如之前所说，它会调用 DbgkForwardException 向调试子系统通报异常。如果 DbgkForwardException 检查到当前进程正在被调试，它会将这个异常通过调试子系统发给调试器。

调试器的 WaitForDebugEvent 会收到一个调试事件，在收到这个事件之前，内核会将事件代码设置为 `OUTPUT_DEBUG_STRING_EVENT`, 并将调试字符串的地址也保存到 `DEBUG_EVENT` 结构体中。

调试器会通过参数中的地址，从被调试进程地址空间中读取调试信息字符串，然后显示出来。

### 使用工具接收调试信息

如果应用程序没有被调试，也可以通过工具来接收调试信息。

在 RaiseException 中，如果检查到没有调试器存在，那么它会尝试将调试信息发送到 DBWIN 工具。DBWIN 是旧 Windows SDK 中包含的一个小工具，现在已经没有了，不过其他的调试器比如 Debug View 实现的功能也是类似的。

那么，调试信息是如何发送给 DBWIN 工具的呢？简单来说，是通过几个内核对象使用进程间的通信机制来完成的，其主要步骤如下：
- 检查是否存在名为 "DBWinMutex" 的 Mutex 内核对象，如果没有则创建。它的作用是保证同一时间内只能有一个线程与 DBWIN 通信；
- OutputDebugString 无限期等待 DBWinMutex 的使用权；
- 等待成功后，OutputDebugString 会通过 OpenFileMapping 打开名为 `"DBWIN_BUFFER"` 的内存映射文件对象。这个对象应当是 DBWIN 程序创建的，用户存储调试信息；
- 如果打开成功，OutputDebugString 会调用 MapViewOfFile 将映射文件映射到本进程空间中；
- 接下来， OutputDebugString 会通过 OpenEvent 打开两个事件内核对象 `DBWIN_BUFFER_READY` 和 `DBWIN_DATA_READY`。其中 `DBWIN_BUFFER_READY` 是由 DBWIN 触发的，表示那边的缓冲区准备好了，可以往缓冲区写数据了，而 `DBWIN_DATA_READY` 是由 OutputDebugString 自己触发的，表示已经向缓冲区里填充好了数据，告诉 DBWIN 可以接收并处理了。
- OutputDebugString 等待到 `DBWIN_BUFFER_READY` 事件之后，就向 `DBWIN_BUFFER` 中写入数据，写入完毕后，就触发 `DBWIN_DATA_READY` 事件，告诉 DBWIN 接受并处理数据；
- DBWIN 收到 `DBWIN_DATA_READY` 事件之后，就打开缓冲区并处理数据，如果数据量比较大，这一过程会反复数次；
- 数据传递结束后 OutputDebugString 会释放 `DBWinMutex` 互斥量，以便让其他线程可以与 DBWIN 通信；

`DBWIN_BUFFER_READY` 事件内核对象还有另一个作用，它可以用来保证只有一个 DBWIN 程序能够运行。以上几种对象中，除了 `DBWinMutex`，其他对象都是由 DBWIN 来创建的。

正因为上述原因，如果程序正处于调试状态，那么 DBWIN 这样的程序是接收不到调试信息的。

最后说明一下，从性能角度来看，OutputDebugString 是一种执行效率比较低的方法，对于效率要求比较高的程序，过多使用 OutputDebugString 会影响程序执行效率。导致 OutputDebugString 执行效率低的原因主要有以下几点：
- 首先，OutputDebugString 是通过 RaiseException 抛出异常来触发与调试器或 DBWIN 程序的通信的，这会导致当前线程从用户态切换到内核态，然后经由内核态的异常处理函数进行分发，最后才将信息送给用户态的调试器或 DBWIN 程序；
- 其次，当非调试状态执行时，如果系统中有 DBWIN 执行，那么与 DBWIN 的通信需要一定的开销(打开、等待多个内核对象)。如果系统有大量的 OutputDebugString 调用，那么 DBWIN 程序可能会很忙碌，调用 OutputDebugString 的线程需要排队等待。

使用 OutputDebugString 的另一个缺点是安全性和可控性差，因为任何一个 DBWIN 程序都能够接收到日志。OutputDebugString 并没有提供动态开启或关闭信息输出的机制，需要自己编码对其进行封装。


## 终止调试会话

本节我们介绍终止调试会话的几种典型情况，探索每种情况的内部过程，并比较他们的异同。

### 被调试进程退出

不同类型的程序有不同的退出方式，无论哪一种，通常都会执行内核中的 PspExitThread 函数来退出线程。

PspExitThread 函数内部会通过 EPROCESS 结构的 DebugPort 字段检查当前进程是否正在调试，如果是，它会根据当前是否是最有一个线程而调用 DbgkExitProcess 或 DbgkExitThread 来通知调试子系统。

如果程序被调试，那么 DbgkExitProcess 会通过调试子系统向调试器发送进程退出事件。调试器收到此事件之后便知道被调试进程正在退出。接下来的处理与调试器的实现有关，对于 MSDEV 等调试器，收到进程退出事件后，它便清理内部状态，结束本次调试了。在 WinDBG 中，默认情况下，它会向用户报告此事件，显示如下内容：

![]( {{ site.url }}/asset/software-debugging-windbg-exit-process.png )

因为 PspExitThread 是在释放线程的用户态栈之前通知调试子系统的，所以当收到进程退出事件时，调试器仍然可以观察被调试进程的信息，包括观察栈、内存和 PEB、TEB 等结构，这对于调试应用程序退出的原因是非常有用的。

想要让被调试进程继续完成其退出工作，需要触发 WinDBG 继续调试事件，可以通过 WinDBG 的停止调试功能(Debug -> Stop Debugging)或分离调试对话功能 (Debug->Detach Debuggee)来完成。

进程最后清理和删除工作是由进程管理器的工作线程执行 PspProcessDelete 函数来完成的。PspProcessDelete 函数内部会检查进程 EPROCESS 结构的 DebugPort, 如果不为空，会调用 ObDereferenceObject 取消引用。而后 PspProcessDelete 调用内存管理器的 MmDeleteProcessAddressSpace 删除进程的地址空间，该进程彻底在系统中消失。

### 调试器进程退出

退出调试器是结束调试会话的另一种简单方式。

当调试器的工作线程退出时，PspExitThread 会检查 TEB 结构的 DbgSsReserved[1] 字段，如果该字段不为空，会调用 ObCloseHandle 关闭调试对象 (DebugObject 内核对象) 的句柄。

### 分离被调试进程

Windows XP 允许被调试进程脱离 (detach) 调试会话并继续保持运行。

调试器可以使用 Windows XP 新引入的调试 API DebugActiveProcessStop 来分离调试对话，其原型为：

```cpp
BOOL DebugActiveProcessStop(DWORD dwProcessId);
```

利用这一特征，调试器可以附加到一个正在运行的进程，通过生成 DUMP 文件采集内部的信息，然后再与其安全分离，也就是在基本上不影响目标进程的条件下采集目标进程的信息。这对于调试需要持续运行的进程 (如数据库系统或其他系统服务) 来说非常重要。

### 退出时分离

Windows XP 引入的与分离调试会话有关的另一个功能就是可以设置当调试器退出时不强制退出被调试进程，也就是当调试器退出时分离被调试进程，而不是将其一起退出，这可以预防因为调试器意外终止导致被调试进程也被杀掉。

在退出调试进程前，DbgkbCloseObject 会检查 DebugObject 的 Flags 是否包含 KillOnExit 标志，如果清除了这个标志，那么便不会退出被调试进程，`DebugSetProcessKillOnExit()` 就是用来设置这个标志的。