---
layout: post
title:  "《软件调试》 学习 10 硬错误和蓝屏"
date:   2017-04-25 12:10:30 +0800
categories: debugging
---

* TOC
{:toc}

一套好的错误处理方案通常应该考虑以下三个方面：
- 即时提示(instant nofitication) : 当错误发生时立刻提示给用户；
- 永久记录(persitent recording) : 将错误永久记录在文件或数据库中供事后分析；
- 自动报告(automatic reporting) : 自动收集错误现场的详细情况并生成错误报告，用户可以通过简单的方式发送到专门用来收集错误报告的服务器。从 Windows XP 开始，Windows 引入了一整套设施来实现这一目标，称为 Windows Error Reporting, 简称为 WER；

本章先介绍用于报告严重错误的硬错误机制 (HardError) 和蓝屏机制 (BSOD)。然后介绍系统转储文件的产生方法和分析方法。介绍声音和闪动窗口等辅助的错误提示机制，介绍配置错误提示机制。介绍使用错误提示机制时应该注意的问题。


## 硬错误提示

硬错误 (HardError) 的本意是指与硬件有关的严重错误，是与重新启动便可以恢复的软件错误 (SoftError) 相对而言的。逐渐地，这个词被用来泛指比较严重的错误情况。

例如下面的缺盘错误对话框，它是在光盘上的程序正在运行时把光盘取出时发生的，该对话框就是使用 HardError 提示机制弹出的：

![]( {{ site.url }}/asset/software-debugging-harderror-example.png )

尽管这个对话框是使用 MessageBox API 弹出的，但是调用 MessageBox API 只是硬错误提示机制的一部分，此前还经历了复杂的发起和分发过程。

硬错误提示既可以在用户态，也可以在内核态使用。在用户态使用的方法是调用 NtRaiseHardError 内核服务。

```cpp
NTSYSAPI NTSTATUS NTAPI NtRaiseHardError(
    NTSTATUS ErrorStatus,
    ULONG NumberOfParameters,
    ULONG UnicodeStringParametersMask,
    PVOID* Parameters,
    HARDERROR_RESPONSE_OPTION ResponseOption,
    PHARDERROR_RESPONSE Response);
```

ErrorStatus 用来传递错误代码，通常是定义在 NTSTATUS.H 中的常量；NumberOfParameters 指定 Parameters 数组中的元素个数；UnicodeStringParametersMask 参数的各个二进制位与 Parameters 数组一一对应，如果某一位为 1, 则说明对应的参数指针指向的是一个 `UNICODE_STRING`, 否则是个整数；ResponseOption 用来定义错误消息的按钮个数、响应方式等选项；Response 参数用来返回错误提示的响应结果，这个结果可以是用户选择的，也可以是因为超时而导致的默认值，或者是处理失败，它的值为如下常量之一：

```cpp
typedef enum _HARDERROR_RESPONSE {
    ResponseReturnToCaller, 
    ResponseNotHandled,
    ResponseAbort,
    ResponseCancel,
    ResponseIgnore,
    ResponseNo,
    ResponseOk,
    ResponseRetry,
    ResponseYes} HARDERROR_RESPONSE, *PHARDERROR_RESPONSE;
```

NtRaiseHardError 的工作主要是参数检查和预处理，在它们把所有错误信息都复制到一个用户态可访问的内存结构中后便调用 ExpRaiseHardError.

### ExpRaiseHardError

ExpRaiseHardError 与 NtRaiseHardError 有着相同的函数原型，它是分发硬错误的枢纽。

首先，ExpRaiseHardError 会检查 ResponseOption 参数是否等于 OptionShutdownSystem (关机), 如果是，那么它会检查调用者是否具有 SeShutdownPrivilege 权限，如果没有权限，那么返回错误 (0xc0000061)；

接着，ExpRaiseHardError 检查全局变量 nt!ExReadyForErrors, 如果该变量为 FALSE, 说明用户态的 HardError 提示系统还没有准备好；如果为 TRUE, 那么它会进一步检查 ErrorStatus 参数，以及当前线程的 ETHREAD 结构的 HardErrorAreDisabled 标志为 0 (表示没有禁止 HardError), 那么 ExpRaiseHardError 就会调用 ExpSystemErrorHandler 在内核态处理和提示这个 HardError。

ExpSystemErrorHandler 函数是以蓝屏的方式来提示 Hard Error 的。它会调用 KeBugCheckEx 或 PoShutdownBugCheck 函数来启动蓝屏，蓝屏的停止码 (Stop Code) 为 `FATAL_UNHANDLED_HARD_ERROR` (0x0000004C)，停止码的第一个参数就是提示 HardError 的状态码。

接下来，ExpRaiseHardError 会按照一定的规则寻找用来发送 HardError 的错误端口 (ErrorPort)，默认情况下，这个错误端口存在于 CSRSS 进程中，也就是说，CSRSS 是系统默认的硬错误提示进程。

ExpRaiseHardError 会把 HardError 信息发送给 CSRSS 的错误端口(`\Windows\ApiPort`)，CSRSS 中再对它进行分发。

### CSRSS 中的分发过程

下图显示了 CSRSS 进程的工作线程处理硬错误提示的过程：

![]({{ site.url }}/asset/software-debugging-csrss-harderror.png)

可以看到最下面的 (栈帧#05)CSRSRV!CsrApiRequestThread 是 CSRSS 进程中专门监听 `\Windows\ApiPort` 端口的工作线程，它负责接收、分发和回复发送到 `\Windows\ApiPort` 端口的 LPC 消息。

CsrApiRequestThread 会枚举进程内已经注册的所有服务模块，查询它是否设置了 HardErrorRoutine 指针。在典型的 Windows XP 系统中，设置了 HardErrorRoutine 指针的服务是 WINSRV, 并且指针指向 WINSRV 的 UserHardError 函数。该函数会直接调用 UserHardErrorEx 函数。

UserHardErrorEx 会建立一个数据结构来保存 HardError 信息，然后把这个数据结构保存到一个链表中，这个链表被称为 全局硬错误信息链表(gloable hard error information pointer list) 简称为 PHI 链表。随后 UserHardErrorEx 会调用 ProcessHardErrorRequest 来处理链表中的信息。

ProcessHardErrorRequest 调用 HardErrorHandler 来处理链表。它会依次从链表中取出要处理的 HardError 任务，然后执行如下操作：
- 第一，向 NtUserHardErrorControl 发出连接桌面命令；
- 第二，准备窗口风格(TOPMOST)、按钮等信息，然后调用 MessageBoxTimeoutW 函数弹出消息对话框；
- 第三，向 NtUserHardErrorControl 发出分离桌面命令；
- 第四，将响应结果通过 ReplyHardError 回复给等待线程；


## 蓝屏终止 (BSOD)

蓝屏是 Windows 中用于提示严重的系统级错误的一种方式，一旦出现蓝屏，Windows 系统便宣告终止，只有重新启动才能恢复到桌面环境，所以蓝屏才被称为蓝屏终止(Blue Screen Of Death)，简称 BSOD。

因为产生蓝屏会终止整个系统的运行，所以蓝屏是 Windows 中代价最高的错误提示方式，通常只有在发生严重错误，或者其他错误提示方式都不可用的情况下才会使用这种方式。

![]({{ site.url }}/asset/software-debugging-blue-screen.jpg )

上面是手动产生的而一个蓝屏。

通常，蓝屏会包含以下几部分内容：
- 第一，错误信息，用来描述错误情况和错误原因的文字；
- 第二，解决错误的建议，通常没有太大意义；
- 第三，技术信息，其格式为 STOP : 停止码(参数1，参数2，参数3，参数4)。停止码代表了导致蓝屏的根本原因，是诊断蓝屏故障的重要技术资料。停止码的参数用来进一步描述错误原因，代表了更深层次的错误信息；
- 第四，显示转储的过程和结果；

### 蓝屏的发起和产生过程

DDK 中提供了两个 DDI(Device Drive Interface) 用来提示蓝屏错误，KeBugCheck 和 KeBugCheckEx, 其原型如下：

```cpp
VOID KeBugCheck(IN ULONG BugCheckCode);
VOID KeBugCheckEx(IN ULONG BugCheckCode, 
    IN ULONG_PTR BugCheckParameter1, IN ULONG_PTR BugCheckParameter2,
    IN ULONG_PTR BugCheckParameter3, IN ULONG_PTR BugCheckParameter4);
```

KeBugCheck 只是简单地调用 KeBugCheckEx, BugCheckCode 是显示在蓝屏上的停止码, BugCheckParameter1 等参数是停止码的参数。

KeBugCheckEx 的工作过程如下：
1. 初始化和准备，将全局变量 nt!KeBugCheckActive 设置为 TRUE，表示进入到了 Bug Check 状态；产生描述系统状态的上下文结构 CONTEXT；
2. 根据 BugCheckCode 寻找错误提示信息，也即是蓝屏的第一部分。例如，如果内核代码的当前中断级别 (IRQL) 不低于 `DISPATCH_LEVEL` 时访问内存，那么系统检测到之后就会发起蓝屏，然后把停止码设置为 `IRQL_NOT_LESS_OR_EQUAL`, 并且把执行访问操作的指令地址设置到 BugCheckParameter4，这样 KeBugCheckEx 就能够根据 BugCheckParameter4 找到对应的模块名称，最终显示到蓝屏的提示信息里；
3. 如果启用了内核调试，那么调用 KdPrint 打印出停止码和参数，然后尝试中断到调试器；
4. 使系统进入到单纯的错误检查状态，停止其他一切活动；
5. 绘制蓝屏画面；
6. 调用错误检查回调函数，通知驱动程序，系统已经进入错误检查阶段，驱动程序可以执行必要的清理工作，或者通知自己的硬件；
7. 如果内核调试引擎没有启用，那么尝试启用；
8. 准备系统转储 (System Dump) 数据，然后调用 IoWriteCrashDump 函数将转储信息写入到硬盘中；
9. 判断是否需要重新启动系统，这一点可以通过 (My Computer --> Properties --> Advanced --> Startup and Recovery --> Automatically Restart) 来设置；

### 如何诊断蓝屏错误

对于蓝屏错误，可以通过如下步骤逐步分析原因：
1. 根据停止码和它的参数做出初步的判断。在 WinDBG 的帮助文件中 (Debugging Techniques --> Bug Checks(Blue Screen) --> Bug Check Code Reference) 列出了系统定义的所有蓝屏错误，包括错误码和每个错误的含义，以及解决问题的建议；
2. 在微软的知识库 (supports.microsoft.com) 中搜索，或者通过搜索引擎搜索蓝屏的停止码和参数，了解更多信息；
3. 分析转储文件，系统默认会为蓝屏产生小型转储文件，默认位置为 Windows 系统目录的 MiniDump 文件夹；
4. 如果经过以上步骤还是没有找到问题原因，应该考虑通过内核调试做进一步的调试和分析。使用内核调试可以设置断点，跟踪代码的执行过程，更容易准确定位到错误根源；

### 手工触发蓝屏

1. 在内核调试会话中，执行 WinDBG 的 .crash 命令；
2. 在注册表的 `HKEY_LOCAL_MACHINE\SYSTEM\CurrentControlSet\Services\i8042prt\Parameters` 中加入一个 `REG_DWORD` 类型的键值，并取名为 CrashOnCtrlScroll, 值为 1，然后重启系统，再按下 Ctrl + Scroll Lock 键，就可以触发蓝屏；
3. 有硬件经验的话，可以将内存条的两个数据线短路；

