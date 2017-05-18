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


## 系统转储文件

之前我们介绍过了用户态转储文件，系统转储文件描述的目标是整个系统，包括操作系统内核，内核态的驱动程序和各个用户进程。

### 分类

系统转储文件有三种类型：
- 完整转储 (Complete memory dump), 包含物理内存中的所有数据；
- 内核转储 (Kernel memory dump), 去除了用户进程所使用的内存页，因此大小要比完整转储小得多，对于 Windows XP 系统，其大小为 200MB 左右；
- 小型内存转储 (Small memory dump), 文件大小默认为 64KB, 如果包含用户数据 (通过 BugCheckSecondaryDumpDataCallback 回调函数写入)，那么可能略大；

### 文件格式

对于所有的转储类型，转储文件的第一页内容的格式是一样的，其开始处是一个名为 `DUMP_HEADER` 的结构，其布局如下表：

![]( {{site.url}}/asset/software-debugging-system-dump-header.png )

从上表可以看出，其中包含了很多关键信息，包括页目录基地址、模块列表地址、进程列表地址、异常结构、上下文结构等。特别是 KdDebuggerDataBlock 的地址，这个结构用来支持内核调试，是内核调试器与内核调试引擎之间的重要数据接口，有了这个地址，调试器就可以通过它来与转储文件建立调试会话，使用类似活动内核调试的方式来分析转储文件。

头页之后的内容便是各个物理内存页的数据，包括实现内存管理的特殊内存页，如保存页目录和页表的内存页等。转储文件读取工具会利用其中的页目录和页表结构模拟地址过程来读取内存页的数据。举例来说，如果某个转储文件的 KdDebuggerDataBlock 字段的地址是 0x818f3c40, 那么我们可以在 WinDBG 中使用如下命令来观察这个值：

![]( {{site.url}}/asset/software-debugging-windbg-db-command.png )

### 产生方法

产生系统转储文件有以下两种方法：

一是使用 WinDBG 调试器的 .dump 命令，这需要在内核调试会话中执行。

二是让系统自己来产生，系统发生蓝屏时，默认会生成系统转储文件。


## 分析系统转储文件

### 初步分析

使用 WinDBG 打开书中所提供的示例 dump 文件。一开始可以看到如下信息：

![]( {{site.url}}/asset/software-debugging-system-dump-windbg.png )

根据 WinDBg 的初步分析，崩溃可能是由 RealBug.SYS 模块导致的，并且导致崩溃的代码距离该模块的起始地址 0x4e1 个字节。我们可以用 `lm m real*` 命令来查看模块的起始地址：

```
kd> lm m real*
start    end        module name
fa18b000 fa18bd00   RealBug  T (no symbols)  
```

其中的 T 表示时间戳，因为 WinDBG 没有加载 RealBug.SYS 映像文件，所以得不到时间戳，只能调试 no symbols。

### 线程和栈回溯

可以通过栈回溯信息来进一步了解转储发生时的线程状态和崩溃原因。

上一节我们介绍过，一旦发起蓝屏后，系统就不会再做线程切换或者执行其他任务，因此进行蓝屏绘制和转储的就是崩溃时的线程。我们可以通过 `.thread` 和 `.process` 命令来获取当前线程结构 (KTHREAD) 和进程结构 (EPROCESS) 的地址：

```
kd> .thread
Implicit thread is now 815ef3f0
kd> .process
Implicit process is now 8160eb98
```

可以看到崩溃线程 KTHREAD 结构的地址是 815ef3f0, 崩溃进程 EPROCESS 结构的地址是 8160eb98.

接着，可以通过 `.thread 815ef3f0` 来查看线程的更多信息：

```
kd> !thread 815ef3f0
....
Owning Process            0       Image:         <Unknown>
Attached Process          8160eb98       Image:         ImBuggy.exe
....
```

通过 Attached Process 这行信息可以了解到，崩溃发生于 ImBuggy.exe 这个进程中。

接着使用 `k` 命令可以查看当前线程的栈回溯信息：

```
kd> k
  *** Stack trace for last set context - .thread/.cxr resets it
ChildEBP RetAddr  
f6869ac0 80607339 nt!KeBugCheck+0x10
f6869b20 804dac8f nt!Ki386CheckDivideByZeroTrap+0x23
f6869b20 fa18b4e1 nt!KiTrap00+0x6d
WARNING: Stack unwind information not available. Following frames may be wrong.
f6869b9c 815ef600 RealBug+0x4e1
f6869bcc fa18b54b 0x815ef600
f6869bdc fa18b62e RealBug+0x54b
f6869be8 fa18b73e RealBug+0x62e
f6869c34 804eca36 RealBug+0x73e
f6869c44 8058b076 nt!IopfCallDriver+0x31
f6869c58 8058bc62 nt!IopSynchronousServiceTail+0x5e
f6869d00 805987ec nt!IopXxxControlFile+0x5ec
f6869d34 804da140 nt!NtDeviceIoControlFile+0x28
f6869d34 7ffe0304 nt!KiSystemService+0xc4
0012f8a8 00000000 SharedUserData!SystemCallStub+0x4
```

注意上述代码中有一个警告 `WARNING: Stack unwind information not available. Following frames may be wrong.` 这是在告诉我们 WinDBG 没有找到 RealBug 模块的符号文件，通过 (File --> Symbol File Path) 对话框将符号目录设置进来之后，再次使用 `k` 命令，会得到如下信息：

```
kd> k
  *** Stack trace for last set context - .thread/.cxr resets it
ChildEBP RetAddr  
f6869ac0 80607339 nt!KeBugCheck+0x10
f6869b20 804dac8f nt!Ki386CheckDivideByZeroTrap+0x23
f6869b20 fa18b4e1 nt!KiTrap00+0x6d
f6869bcc fa18b54b RealBug!PropDivideZero+0x3f [c:\dig\training\advdbg\dbglabs\realbug\realbug.c @ 63]
f6869bdc fa18b62e RealBug!DivideZero+0xb [c:\dig\training\advdbg\dbglabs\realbug\realbug.c @ 77]
f6869be8 fa18b73e RealBug!RealBugDeviceControl+0x44 [c:\dig\training\advdbg\dbglabs\realbug\realbug.c @ 114]
f6869c34 804eca36 RealBug!RealBugDispatch+0x8e [c:\dig\training\advdbg\dbglabs\realbug\realbug.c @ 175]
f6869c44 8058b076 nt!IopfCallDriver+0x31
f6869c58 8058bc62 nt!IopSynchronousServiceTail+0x5e
f6869d00 805987ec nt!IopXxxControlFile+0x5ec
f6869d34 804da140 nt!NtDeviceIoControlFile+0x28
f6869d34 7ffe0304 nt!KiSystemService+0xc4
0012f8a8 00000000 SharedUserData!SystemCallStub+0x4
```

可以看到，这里已经显示了相关的源文件路径，以及对应的代码行数，通过这些信息我们可以更容易地定位到错误所在。

有时候使用 `k` 命令时会显示 `Unable to load image RealBug.SYS, Win32 error 2` 错误，这是因为 WinDBG 没有找到 RealBug.sys 模块文件，把模块的路径通过 (File --> Image File Path) 设置进来之后就可以解决。

### 陷阱帧

当有异常发生时，系统会将当时的状态保存到一个 `KTRAP_FRAME` 结构中，称为陷阱帧。因为很多崩溃与异常有关，所以转储文件中经常包含着陷阱帧数据。可以通过 `kv` 命令来搜索当前栈回溯序列中是否有陷阱帧：

```kv
kd> kv
  *** Stack trace for last set context - .thread/.cxr resets it
ChildEBP RetAddr  Args to Child              
f6869ac0 80607339 0000007f 815ef600 8156aae8 nt!KeBugCheck+0x10 (FPO: [1,0,0])
f6869b20 804dac8f f6869b2c 00000046 00000046 nt!Ki386CheckDivideByZeroTrap+0x23 (FPO: [Non-Fpo])
f6869b20 fa18b4e1 f6869b2c 00000046 00000046 nt!KiTrap00+0x6d (FPO: [0,0] TrapFrame @ f6869b2c)
f6869bcc fa18b54b 00000008 00000387 f6869be8 RealBug!PropDivideZero+0x3f (FPO: [Non-Fpo]) (CONV: stdcall) [c:\dig\training\advdbg\dbglabs\realbug\realbug.c @ 63]
f6869bdc fa18b62e 00000004 f6869c34 fa18b73e RealBug!DivideZero+0xb (FPO: [Non-Fpo]) (CONV: stdcall) [c:\dig\training\advdbg\dbglabs\realbug\realbug.c @ 77]
f6869be8 fa18b73e 8158bb38 00000001 00000000 RealBug!RealBugDeviceControl+0x44 (FPO: [Non-Fpo]) (CONV: stdcall) [c:\dig\training\advdbg\dbglabs\realbug\realbug.c @ 114]
f6869c34 804eca36 8166dd30 814c5368 806c7fe0 RealBug!RealBugDispatch+0x8e (FPO: [Non-Fpo]) (CONV: stdcall) [c:\dig\training\advdbg\dbglabs\realbug\realbug.c @ 175]
f6869c44 8058b076 814c53d8 8158bb38 814c5368 nt!IopfCallDriver+0x31 (FPO: [0,0,1])
f6869c58 8058bc62 8166dd30 814c5368 8158bb38 nt!IopSynchronousServiceTail+0x5e (FPO: [Non-Fpo])
f6869d00 805987ec 00000068 00000000 00000000 nt!IopXxxControlFile+0x5ec (FPO: [Non-Fpo])
f6869d34 804da140 00000068 00000000 00000000 nt!NtDeviceIoControlFile+0x28 (FPO: [Non-Fpo])
f6869d34 7ffe0304 00000068 00000000 00000000 nt!KiSystemService+0xc4 (FPO: [0,0] TrapFrame @ f6869d64)
0012f8a8 00000000 00000000 00000000 00000000 SharedUserData!SystemCallStub+0x4 (FPO: [0,0,0])
```

注意上述代码中的 TrapFrame 字段， @ 符号之后就是陷阱帧 `KTRAP_FRAME` 的地址。使用 `.trap` 命令就可以切换到某个异常发生时的状态：

```
kd> .trap f6869b2c
ErrCode = 00000000
eax=00000001 ebx=814c5368 ecx=00000004 edx=00000000 esi=8156aae8 edi=815ef600
eip=fa18b4e1 esp=f6869ba0 ebp=f6869bcc iopl=0         nv up ei pl zr na pe nc
cs=0008  ss=0010  ds=0023  es=0023  fs=0030  gs=0000             efl=00000346
RealBug!PropDivideZero+0x3f:
0008:fa18b4e1 f77de0          idiv    eax,dword ptr [ebp-20h] ss:0010:f6869bac=00000000
```

以上寄存器便是异常发生时的状态，第二行的 ErrorCode 是异常的错误码，最后一行的汇编指令就是导致这个异常的指令，可见它是一条除法指令。`[ebp-20h] ss:0010:f6869bac=00000000` 表示除法指令的参数，可以看到这个参数为 0, 也就是发生了除零异常。

### 自动分析

为了简化蓝屏分析，WinDBG 把很多可以自动完成的工作实现在 `!analyze` 命令中了。因此，分析蓝屏问题的第一步通常是先执行这个命令，让 WinDBG 作自动分析，然后再做手动分析。

下图是 `!analyze -v` 命令显示结果的说明，其中 `-v` 开关表示启动最详细的分析方式。

![]( {{site.url}}/asset/software-debugging-dump-analyze.png )