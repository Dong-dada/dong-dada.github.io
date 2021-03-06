---
layout: post
title:  "《软件调试》 学习 14 内核调试引擎"
date:   2017-05-11 10:29:30 +0800
categories: debugging
---

* TOC
{:toc}

简单来说，内核调试就是分析和调试位于内核空间中的代码和数据。运行在内核空间的模块主要有操作系统的内核、执行体和各种驱动程序。从操作系统的角度看，可以把驱动程序看做是对操作系统内核的扩展和补充。因此可以把内核调试简单地理解为操作系统的广义内核。

使用调试器调试的一个重要特征就是可以把调试目标中断到调试器中，换言之，当我们在调试器中分析调试目标时，调试目标是处于冻结状态的。

进行用户态调试的时候，被调试进程将被挂起。同样的，内核调试的时候，被调试的操作系统内核也将停止运行，接受调试器的分析和检查。这时候怎么对已经冻结的操作系统进行调试呢？

目前主要有三种方案来解决上述问题：
1. 使用硬件调试器，它可以通过特定的接口(如 JTAG)与 CPU 建立连接并读取它的状态；
2. 在内核中插入专门用于调试的中断处理函数和驱动程序，当系统内核被中断时，由这些中断处理函数和驱动程序来接管系统硬件，营造出一个可供调试器运行的简单环境；
3. 在内核中加入调试支持，当内核需要中断到调试器时，只保留这些代码还在运行。由另外一个操作系统与这些调试支持程序通信，完成调试功能。这两个操作系统间通过电缆连接；

Windows 操作系统推荐的内核调试方法是第三种。操作系统中内建的部分被称为内核调试引擎 (Kernel Debug Engine)；


## 概览

### KD

Windows 操作系统的每个系统部件中都有一个简短的名字，通常为两个字符，比如 MM 代表内存管理器，OB 代表对象管理器，PS 代表进程和线程管理，等等。同样的，内核调试引擎也有一个这样的名字，叫做 KD (Kernel Debug)。内核模块中用来支持内核调试的函数和变量大多都是以这两个字母开头的。

### 角色

从下图左侧可以看出，内核调试引擎是 内核调试器 与 被调试内核之间的桥梁。

![]( {{site.url}}/asset/software-debugging-kernel-debug-engine.png )

内核调试引擎 和 内核调试器 之间通过 内核调试协议 进行通信。通过这个协议，调试器可以请求调试引擎帮助它访问和控制目标系统，调试引擎也会主动将目标系统的状态报告给调试器。

从访问内核的角度看，内核调试引擎为内核调试器提供了一套特殊的 API，我们将其称为内核调试 API，简称 KdAPI。使用 KdAPI, 调试器可以以一种类似远程调用的方式访问到内核，这与应用程序通过 Win32 API 访问内核 (服务) 很类似。上图右侧更好地显示了 内核调试器、调试引擎、内核其他部分 这三者之间的关系。

### 组成

可以把内核调试引擎分为如下几个部分：

- **与系统内核的接口函数** 这是调试引擎向内核暴露的一些接口，内核会调用这些接口，来初始化引擎、让引擎处理异常、检查中断命令等。比如内核启动过程中会调用 KdInitSystem 来初始化内核调试引擎；内核分发异常时会调用 KdpTrap 或 KdpStub 等；
- **与调试器的通信函数** 负责与另一个系统中的调试器进行通信，包括建立和维护通信端口，收发数据包等；
- **断点管理** 负责记录所有断点，调试引擎使用一个数组来记录断点，其名称为 KdpBreakpointTable；
- **内核调试 API** 这是内核调试引擎与调试器之间的逻辑接口。之所以说是逻辑接口，是因为内核调试引擎和调试器不在同一个机器上，所以不可以直接调用。实际的做法是调试器通过数据包将要调用的 API 号码和参数传递给调试引擎，调试引擎收到后调用对应的函数，然后再把函数执行结果以数据包的形式返回给调试器。这些接口向调试器提供了许多支持，包括读写内存、读写 IO 空间、读取和设置上下文、设置和恢复断点等；
- **系统内核控制函数** 包括负责将系统内核中断到调试器的 KdEnterDebugger 函数，恢复系统运行的 KdExitDebugger 函数；
- **管理函数** 包括启用和禁止内核调试引擎的 KdEnableDebugger 和 KdDisableDebugger, 以及修改选项的 KdChangeOption. WinDBG 工具包中的 kdbgctrl 工具就是通过这些函数来工作的；
- **ETW 支持函数** 与 ETW 机制配合将追踪数据输出到调试器所在的主机上。负责这一功能的主要函数是 KdReportTraceData；
- **驱动程序更新服务** 从主机上读取驱动程序文件来更新被调试系统中的驱动程序；
- **本地内核调试支持** 包括 NtSystemDebugControl 和 KdSystemDebugControl ;

下图展示了这些部分的作用：

![]( {{site.url}}/asset/software-debugging-kernel-debug-engine-modules.png )

### 模块文件

这一小节介绍内核调试引擎所在的模块文件 (DLL) 文件。

在 Windows XP 之前，内核调试引擎的所有函数都位于 NT 内核文件中，即 NTOSKRNL.EXE, 从 Windows XP 开始，内核调试引擎中的通信部分被拆分到一个单独的 DLL 中，名为 KDCOM.DLL。


## 连接

这一节讨论调试器所在的系统如何与被调试系统建立连接。它们都是通过电缆连接在一起的，但是有 3 中不同的连接方式： 串行口、1394、USB 2.0，下面分别介绍它们：

### 串行口

串行通信 (Serial Communication) 是 Windows 内核调试的本位 (native) 通行方式。因为内核调试一开始就是针对串行通信设计的，所以直至今天，串行通信仍然是进行 Windows 内核调试的最稳定方式。这其中一个主要原因就是内核调试通信协议是面向字节而不是数据包定义的，而串行通信是最适合按字节来读写数据和同步的，其他的通信方式都是以数据包的形式来组织数据的。

### 1394

1394 又称火线，是一种高性能的串行总线通信标准。Windows XP 引入了使用这种接口来进行内核调试的支持。

### USB 2.0

USB 是 Universal Serial Bus 的缩写，是一种低成本高性能的串行总线标准。

USB 总线的节点连接具有方向性，USB 端口分为用于连接设备的上游 (upstream) 和用来连接主机的下游 (downstream) 端口。而一般个人电脑系统上的 USB 端口都是所谓的上游 (upstream) 端口。因为两个上游端口是无法简单连接而进行通信的，所以使用 USB 2.0 方式进行内核调试时，首先需要有一根特殊的 USB 2.0 主机到主机的电缆。

### 管道

虚拟机 (Virtual Machine) 技术可以在一台物理系统中构建出多个虚拟机，每个虚拟机可以安装和运行一个操作系统。

虚拟机技术的流行使人们很自然地想到可以利用它来进行内核调试。

接下来要考虑的问题是如何建立两个系统之间的通信连接。这需要使用软件的通信方式来模拟硬件通信端口，比如使用命名管道模拟串行端口。其做法是在虚拟机管理软件中使用命名管道虚拟出一个 COM 口，下图展示了这一设置的方法：

![]( {{site.url}}/asset/software-debugging-kernel-debug-virtual-machine-setting1.png )

![]( {{site.url}}/asset/software-debugging-kernel-debug-virtual-machine-setting2.png )

![]( {{site.url}}/asset/software-debugging-kernel-debug-virtual-machine-setting3.png )

经过以上设置之后，虚拟机中所有对串行端口 1 的读写操作都会被虚拟机管理软件转换为对宿主系统中的命名管道的读写。因此，运行在宿主系统中的调试器便可以通过这个命名管道来与虚拟机中的内核调试引擎痛心了。可以通过以下两种方式来设置调试器，一种是命令行参数：

```
windbg [-y SymPath] -k com:pipe, port=\\.\pipe\PipeName[,resets=0][,reconnect]
```

其中 `PipeName` 应该替换为虚拟机管理软件中设置的名称，即 com_1。

另一种方式是通过图形界面，也就是使用下图中的内核调试属性对话框，选中 Pipe 复选框，然后在 Port 中指定命名管道的全路径，即 `\\.\pipe\com_1`，再选中 Reconnect 复选框。

![]( {{site.url}}/asset/software-debugging-kernel-debug-windbg-setting.png )

使用虚拟机进行内核调试的优点是简单方便，但也有如下缺点：一是难以调试硬件相关的驱动程序；二是涉及到某些底层操作的函数或指令设置断点时，可能导致虚拟机意外重新启动；三是当目标系统中断到调试器时，虚拟机管理软件可能会占用很高的 CPU。


## 启用

尽管内核调试引擎已经包含在每个 Windows 系统中，但出于安全和性能考虑，它默认处于禁止状态。因此在进行内核调试前要先启用它。对于 Vista 之前的 Windows，需要修改 BOOT.INI 文件，对于 Vista，需要修改启动配置数据 (Boot Configuration Data)。

这部分就不介绍了，实际需要时再查找就可以了。


## 初始化

本节介绍内核调试引擎初始化的过程。因为这一过程是穿插在 Windows 系统的启动过程中的，所以我们先介绍 Windows 的启动过程。

### Windows 的启动过程概述

计算机开机后，先执行的是系统的固件 (Firmware), 即 BIOS (Basic Input/Output System, 基本输入输出系统) 或 EIF (Extexed Firmware Interface)。BIOS 或 EIF 完成基本的硬件检测和平台初始化工作后，将控制权交给磁盘上的引导程序。

引导程序执行操作系统的加载程序 (OS Loader), 即 NDLDR(Vista 以前) 或 WinLoader.exe (Vista)。

OS Loader 首先对 CPU 做必要的初始化工作，包括从 16 位实模式切换到 32 位保护模式，启用分页机制等，然后通过启动配置信息 (Boot.INI 或 BCD) 得到 Windows 系统的系统目录并加载内核文件 NTOSKRNL.EXE 及其依赖文件。其中包含了用于内核调试通信的硬件扩展 DLL (KDCOM.DLL, KD1394.DLL, KDUSB.DLL) 加载程序会选择加载其中的一个。

OS Loader 接着读取注册表的 System Hive, 加载其中定义的启动类型的驱动程序，包括磁盘驱动程序。

完成上述工作后，OS Loader 会从内核文件 NTOSKRNL.DLL 的 PE 文件头中找到入口函数，即 KiSystemStartup，然后调用它。

接下来的启动过程分为下图所示的三个部分。左侧是发生在初始启动进程中的过程，这个初始的进程就是启动后的 Idle 进程。中间是发生在系统进程 (System) 中的所谓执行体阶段 1 初始化过程。右侧是发生在会话管理器进程 (SMSS) 的过程。

![]( {{site.url}}/asset/software-debugging-windows-startup.png )

首先详细看看左侧 KiSystemStartup 函数的执行过程：
- 调用 HalInitializeProcessor() 初始化 CPU；
- 调用 KdInitSystem 初始化内核调试引擎；
- 调用 KiInitializeKernel 开始内核初始化，这个函数会调用 KiInitSystem 来初始化系统的全局数据结构，调用 KeInitializeProcess 来创建并初始化 Idle 进程，调用 KeInitizlizeThread 来初始化 Idle 线程，调用 ExpInitializeExecutive() 进行所谓的执行体阶段 0 初始化，包括调用 MmInitSystem 构建页表和内存管理器的基本数据结构，调用 ObInitSystem 建立名称空间，调用 SeInitSystem 初始化 Token 对象，调用 PsInitSystem 对进程管理器进行阶段 0 初始化，调用 PpInitSystem 让即插即用管理器初始化设备链表；

在 KiInitializeKernel 返回后，KiSystemStartup 函数将当前 CPU 的中断请求级别 (IRQL) 降低到 `DISPATCH_LEVEL`, 然后跳转到 `KiIdleLoop()`, 退化为 Idle 进程中的第一个 Idle 线程。

接下来仔细看一下进程管理器阶段 0 初始化过程中发生了什么：
- 定义进程和线程对象类型；
- 建立记录系统中所有进程的链表结构，并使用 PsActiveProcessHead 全局变量指向这个链表，此后 WinDBG 的 !process 命令才能工作；
- 为初始的进程创建一个进程对象 (PsIdleProcess), 并命名为 Idle；
- 创建系统进程和线程，并将 Parse1Initialization 函数作为线程的起始地址；

注意最后一步，它衔接着系统启动的下一个阶段，即执行体阶段 1 初始化。阶段 1 初始化占据了系统启动的大多数时间，其主要任务就是调用执行体各机构的阶段 1 初始化函数。其中重要的几个有：
- 调用 KeStartAllProcessors() 初始化所有 CPU。这个函数会先构建并初始化好一个处理器结构，然后调用硬件抽象层的 HalStartNextProcessor 函数将这个结构赋给一个新的 CPU。新的 CPU 仍然从 KiSystemStartup 开始执行；
- 再次调用 KdInitSystem 函数，并且调用 KdDebuggerInitialize1 来初始化内核调试通信扩展 DLL (KDCOM.DLL 等)；
- 在这一阶段结束前，它会创建第一个使用映像文件创建的进程，即会话管理器进程 (SMSS.EXE)

会话管理器进程会初始化 Windows 子系统，创建 Windows 子系统进程和登录进程 (WinLogin.EXE), 后者会创建 LSASS (Local Security Authority Subsystem Service) 进程和系统服务进程 (Services.EXE) 并显示登录界面，至此启动过程基本完成。

### 第一次调用 KdInitSystem

从之前的初始化流程图中可以看出，系统在启动过程中会两次调用内核调试引擎的初始化函数 KdInitSystem. 第一次是在系统内核开始执行后由入口函数 KiSystemStartup 调用。这次调用时，KdInitSystem 会执行以下动作：
- 初始化调试器数据链表，使用变量 KdpDebuggerDataListHead 指向这个链表；
- 初始化 KdDebuggerDataBlock 数据结构，该结构包含了内核基地址、模块链表指针、调试器数据链表指针等重要数据，调试器需要读取这些信息以了解目标系统；
- 根据参数指针指向的 `LOADER_PARAMETER_BLOCK` 结构寻找调试有关的选项，然后保存到变量中；
- 对于 XP 之后的版本，调用 `KdDebuggerInitialize()` 来对通信扩展模块进行阶段 0 初始化；

此外，KdInitSystem 第一次调用过程中会初始化以下全局变量：
- KdPitchDebugger: 标识是否显式抑制内核调试；
- KdDebuggerEnabled: 标识内核调试是否被启用；
- KdDebugRoutine: 函数指针，用来记录内核调试引擎的异常处理回调函数，当内核调试引擎处于活动状态时，它指向 KdTrap 函数，否则指向 KdpStub 函数；
- KdpBreakpointTable: 结构数组，用来记录断点，每个元素都是一个 `BREAKPOINT_ENTRY` 结构，用来描述一个断点；

### 第二次调用 KdInitSystem

在目前的实现中，KdInitSystem 的阶段 1 初始化只是简单地调用 KeQueryPerformanceCounter 来对变量 KdPerformanceCounterRate(性能计数器频率) 初始化，然后返回；

### 通信扩展模块的阶段 1 初始化

在阶段 1 初始化中，系统会调用通信扩展模块的 KdDebuggerInitialize1 函数来让通信扩展模块得到阶段 1 初始化的机会。


## 内核调试协议

尽管没有详细文档，但是从 NT3.51 到 Windows 2000 的 DDK 中都包含了一个名为 Windbgkd.h 的头文件，其中包含了内核调试协议所使用的所有数据结构、常量和简单说明。

### 数据包

内核调试引擎和调试器是以数据包的形式来通信的，根据包中的内容，可以把数据包分成如下三类：
- 中断包 (Breakin Packet) : 供调试器通知内核调试引擎中断到调试器；
- 信息包 (Information Packet) : 用来传递调试信息或调试命令；
- 控制包 (Control Packet) : 用来建立通信连接或控制通信流程，例如确认收到数据，要求重新发送数据，或者请求建立连接。

### 典型对话过程

深入理解内核调试协议的一种有效办法，就是使用串行口通信监视工具记录下调试器与内核调试引擎之间的所有通信记录，然后分析它们的对话过程。下表列出了使用 WinDBG 通过串行口调试一个 Windows XP SP2 系统时二者之间的对话过程：

![]( {{site.url}}/asset/software-debugging-kernel-debug-talk.png )

上表中包含了如下几个重要步骤：
- 建立连接，即 4~9 行；
- 调试器读取目标系统信息，初始化调试引擎的过程，第 10 ~ 18 行，其中第 23 ~ 24 行包含了一次调试引擎要求重新发送的动作；
- 内核调试引擎通过状态变化信息包通知调试器加载初始模块的调试符号，第 29 ~ 37 行；
- 调试器发送中断包，将目标系统中断到调试器，交互调试后又恢复执行的过程，第 38 ~ 46 行；
- 因为断点命中，目标系统中断到调试器的过程，第 47 ~ 49 行；
- 内核中的模块输出调试字符串 (DbgPrint) 到调试器，第 51 ~ 52 行；


## 本地内核调试

因为使用内核调试引擎进行内核调试需要两个系统，设置和使用需要较多的步骤，所以，如果只希望执行一些观察变量或检查符号之类的简单任务，那么可以使用本节介绍的本地内核调试方法。

### LiveKD

LiveKD 是 Mark Russinovich 编写的一个小工具，可以在保持系统工作的情况下产生一个故障转储文件，名为 LiveKD.DMP.

你可以用 WinDBG 来调试这个转储文件。

### Windows 自己的本地内核调试支持

使用 WinDBG 就可以进行本地内核调试 (File -> Kernel Debug -> Local)

本地内核调试尽管使用比较方便，但是它只支持有限的调试命令，不支持设置断点、单步执行和将系统中断到调试器这样的高级功能。因此，通常只使用这种方法来观察系统的模块、函数、内存、进程和内核对象等。

