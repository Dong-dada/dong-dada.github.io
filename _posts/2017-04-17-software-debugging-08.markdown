---
layout: post
title:  "《软件调试》 学习 08 中断和异常管理"
date:   2017-04-17 10:06:30 +0800
categories: debugging
---

* TOC
{:toc}

之前我们从硬件 (CPU) 角度介绍了中断和异常机制，本章将从操作系统 (Windows) 的角度进一步讨论中断和异常机制。包括管理中断和异常的核心数据结构，Windows 分发异常的基本过程，以及 Windows 系统的结构化异常处理 (SEH) 和向量化异常处理 (VEH) 机制。

## 中断描述符表

在保护模式下，当有中断或异常发生时，CPU 是通过中断描述符表 (Interrupt Descriptor Table, 简称 IDT) 来寻找处理函数的。因此，可以说 IDT 表是 CPU (硬件) 与操作系统 (软件) 交接中断和异常的关口 (Gate)。操作系统在启动早期的一个重要任务就是设置 IDT 表，准备好处理异常和中断的各个函数。

简单来说，IDT 表是一张位于物理内存中的线性表，共有 256 个表项。32 位下每个表项的长度是 8 个字节。IDT 表的位置和长度是由 CPU 的 IDTR 寄存器来描述的。IDTR 寄存器共有 48 位，高 32 位是 IDT 表的基地址，低 16 位是 IDT 表的长度。LIDT(Load IDT) 指令用于向 IDTR 寄存器写入基地址和长度信息，SIDT (Save IDT) 指令用于从 IDTR 寄存器中读取基地址和长度信息。

IDT 表的每个表项是一个所谓的门描述符 (Gate Descriptor) 结构。之所以这样称呼，是因为 IDT 表项的基本用途就是引领 CPU 从一个空间到另一个空间去执行，每个表项好像是从一个空间进入到另一个空间的大门 (Gate)。在穿越这扇门时 CPU 会做必要的安全检查和准备工作。

IDT 表中包含以下三种门描述符：
- 任务门(task-gate)描述符：用于任务切换，里面包含用于选择任务状态段(TSS)的段选择子。可以使用 JMP 或 CALL 指令通过任务门切换到指定的任务，当 CPU 因为中断或异常转移到任务门时，也会切换到指定的任务；
- 中断门(interrupt-gate)描述符：用于描述中断处理例程的入口；
- 陷阱门(trap-gate)描述符：用于描述异常处理例程的入口；

下图描述了以上 3 中门描述的内存布局，每个门描述符的长度都是 8 个字节：

![]({{ site.url }}/asset/software-debugging-idt-gate.png)

### 执行中断和异常处理函数

接下来我们看看当有异常和中断发生时，CPU 是如何通过 IDT 表来寻找和执行处理函数的。首先，CPU 会根据其向量号和 IDTR 寄存器中的 IDT 表基地址信息找到对应的门描述符。然后判断门描述符的类型，如果是任务描述符，那么 CPU 会执行硬件方式的任务切换，切换到这个描述符所定义的线程，如果是陷阱描述符或中断描述符，那么 CPU 会在当前任务上下文中调用描述符所描述的处理例程。下面分别加以讨论。

我们先来看看任务门的情况。简单来说，任务门描述的是一个 TSS 段，CPU 要做的是切换到这个 TSS 段所代表的线程，然后开始执行这个线程。TSS 段是用来保存任务信息的一段内存区，其格式是 CPU 定义的，它包含了一个任务的关键上下文信息，如段寄存器、通用寄存器和控制寄存器。

CPU 在通过任务门的段选择子找到 TSS 段描述符后，会执行一系列的检查动作。所有检查都通过后，CPU 会将当前任务的状态保存到当前任务的 TSS 段中。然后把 TSS 段描述符中的 Busy 标志设为 1。接下来，CPU 把新任务的段选择子加载到 TR 寄存器，然后把新任务的寄存器信息加载到物理寄存器中。最后，CPU 开始执行新的任务。

大多数中断和异常都是利用中断门或陷阱门来处理的，下面我们看看这两种情况：

首先，CPU 会根据门描述符中的段选择子定位到段描述符，然后再进行一系列检查，检查通过后，CPU 会判断是否需要切换栈。如果目标的代码段的特权级别比当前特权级别高，那么 CPU 需要切换栈，其方法是从当前任务的任务状态段 TSS 中读取新堆栈的段选择子 SS 和 堆栈指针 ESP，并将其加载到 SS 和 ESP 寄存器。然后，CPU 会把中断过程(旧的)的堆栈段选择子 SS 和 堆栈指针 ESP 压入新的堆栈。接下来，CPU 会执行如下操作：
- 把 EFLAGS，CS 和 EIP 的指针压入堆栈。CS 和 EIP 指针代表了转到处理例程前 CPU 正在执行代码的位置；
- 如果发生的是异常，而且该异常具有错误代码，那么把该错误代码也压入堆栈；

如果处理例程所在代码段的特权级别与当前特权级别相同，那么 CPU 便不需要进行堆栈切换，但仍然要执行上述两步操作。

经常做内核调试的读者可能会发现，TR 寄存器的值大多数时候都是固定的。也就是说，并不会随着应用程序的线程切换而变化。事实上，Windows 系统的 TSS 个数并不是与系统中的线程个数相关的，而是与 CPU 个数相关的。在启动期间，Windows 会为每个 CPU 创建 3~4 个 TSS，一个用于处理 NMI，一个用于处理 #DF 异常，一个处理机器检查异常，另一个供所有 Windows 线程共享。当 Windows 切换下线程时，它把当前线程的状态复制到共享 TSS 中。也就是说，普通的线程切换并不会切换 TSS，只有当 NMI 或 #DF 异常发生时，才会切换 TSS，这就是所谓的以软件方式切换线程(任务)。


## 异常的描述和登记

为了更好地管理异常，Windows 系统定义了专门的数据结构来描述异常，并定义了一系列代码来标识典型的异常。

在操作系统层次，除了 CPU 产生的异常，还有通过软件方式模拟出的异常，比如通过 RaiseException API 而产生的异常和使用编程语言 throw 关键字抛出的异常。为了行文方便，我们把前一类异常称为 CPU 异常(或硬件异常)，后一类称为软件异常。Windows 是使用统一的方式来描述和分发这两类异常的。本节介绍异常的描述方式。下一节将介绍异常的分发过程。

### EXCEPTION_RECORD 结构

Windows 使用 `EXCEPTION_RECORD` 结构来描述异常，下图给出了这个结构的定义：

![]( {{ site.url }}/asset/software-debugging-exception-record.png )

其中 ExceptionCode 为异常代码，是一个 32 位的整数；ExcepitonFlags 用来记录异常标志；ExceptionRecord 指针指向与该异常有关的另一个异常记录，如果没有相关的异常，那么该指针为空；ExceptionAddress 字段用来记录异常地址；NumberParameters 是参数个数，即 ExceptionInformation 数组中包含的有效参数个数。

下表列出了常见的用于异常代码的状态代码：

![]( {{site.url}}/asset/software-debugging-exception-code.png)

ExceptionFlags 已定义的标志位有：
- `EH_NONCONTINUABLE`, 该异常不可以恢复执行；
- `EH_UNWINDING`, 当因为执行栈展开而调用异常处理函数时，会设置此标志；
- `EH_EXIT_UNWIND`, 也是用于栈展开，较少使用；
- `EH_STACK_INVALID`, 当检测到栈错误时，设置此标志；
- `EH_NESTED_CALL`, 用于标识内嵌的异常；

### 登记 CPU 异常

对于 CPU 异常，KiTrapXX 例程在完成针对本异常的特别动作之后，通常会调用 CommonDispatchException 函数，并通过寄存器将如下信息传递给这个函数
- 将唯一标识该异常的一个异常代码(ExceptionCode)放入 EAX 寄存器；
- 将导致该异常的指令地址放入 EBX 寄存器；
- 将其他信息作为附带参数分别放入 EDX, ESI, EDI 寄存器，并将参数个数放入 ECX 寄存器。

CommonDispatchException 被调用后，它会在栈中分配一个 `EXCEPTION_RECORD` 结构，并把以上信息存储到该结构中，在准备好这个结构之后，它会调用内核中的 KiDispatchException 函数来分发异常。

### 登记软件异常

简单来说，软件异常是通过直接或简介调用内核服务 NtRaiseException 而产生的。

用户模式的程序可以通过 RaiseException() API 来调用这个内核服务：

```cpp
void RaiseException(DWORD dwExceptionCode, DWORD dwExceptionFlags, DWORD nNumberOfArguments, const DWORD* lgArguments);
```

RaiseException 的实现很简单，它只是将参数放入 `EXCEPTION_RECORD` 结构，然后就去调用 NTDLL 中的 RtlRaiseException(), 后者会将当前的执行上下文(通用寄存器等)放入 CONTEXT 结构，然后通过 NTDLL 中的系统服务机制调用内核中的 NtRaiseException。NtRaiseExcepiton 会调用另一个内核函数 KiRaiseException:

```cpp
NTSTATUS KiRaiseException(
    PEXCEPTION_RECORD ExceptionRecord, 
    PCONTEXT ContextRecord, 
    PEXCEPTION_FRAME ExceptionFrame, 
    PKTRAP_FRAME TrapFrame, 
    BOOLEAN FirstChance);
```

ContextRecord 是指向线程上下文结构的指针，ExceptionFrame 对于 x86 平台总是为空， TrapFrame 就是栈帧的基地址，FirstChance 表示这是该异常的第一轮还是第二轮处理机会。

KiRaiseException 内部会通过 KeContextToKFrames 例程把 ContextRecord 结构中的信息复制到当前线程的内核栈，然后把 ExceptionRecord 中的异常代码的最高位清 0，以便把软件异常与 CPU 异常区分开来，接下来 KiRaiseException 会调动 KiDispatchException 开始分发该异常。

综上所述，不论是 CPU 异常还是软件异常，最终都会调用 KiDispatchException 来分发异常，也就是说，Windows 是通过统一的方式来分发 CPU 异常和软件异常的。


## 异常分发过程

根据前面两节的介绍，当有异常发生时，CPU 会通过 IDT 表找到异常处理函数，即内核中的 KiTrapXX 系列函数，然后转去执行。但是 KiTrapXX 函数通常只是对异常做简单的表征和描述，为了支持调试和软件自己定义的异常处理函数，系统需要将异常分发给调试器或应用程序的处理函数。本节我们将介绍分发异常的核心函数 KiDispatchException 和它的工作过程。

### KiDispatchException 函数

Windows 内核中的 KiDispatchException 函数是分发各种 Windows 异常的枢纽，其函数原型如下：

```cpp
VOID KiDispatchException(
    PEXCEPTION_RECORD ExceptionRecord,
    PKEXCEPTION_FRAME ExceptionFrame,
    PKTRAP_FRAME TrapFrame,
    KPROCESSOR_MODE PreviousMode,
    BOOLEAM FristChance);
```

其中，参数 ExceptionRecord 指向的是我们上一节介绍的 `EXCEPTION_RECORD` 结构，用来描述要分发的异常。参数 ExceptionFrame 对于 x86 系统总是为 NULL；参数 TrapFrame 指向的是 `KTRAP_FRAME` 结构，用于描述异常发生时的处理器状态，包括各种通用寄存器、调试寄存器、段寄存器等；参数 PreviousMode 是一个枚举类型的常量，DDK 的头文件中有这个枚举类型的定义：

```cpp
typedef enum _MODE {
    KernelMode,
    UserMode,
    MaximumMode
} MODE;
```

也就是说, PreviousMode 等于 0 表示前一个模式(通常是触发异常代码的执行模式)是内核模式， 1 表示用户模式。

下图是 KiDispatchException 分发异常的基本过程：

![]( {{ site.url }}/asset/software-debugging-kidispatchexception.png )

从上图可以看出，KiDispatchException 会先调用 KeContextFromKFrames 函数，目的是根据 TrapFrame 参数指向的 `KTRAP_FRAME` 结构产生一个 CONTEXT 结构，以供向调试器和异常处理器函数报告异常时使用。

接下来，会根据 PreviousMode 是内核模式还是用户模式，选取左右两个流程之一来分发异常，下面我们分别做进一步说明。

### 内核态异常的分发过程

如果 PreviousMode 是 KernelMode, 那么 KiDispatchException 会执行右侧的路线。

对于第一轮处理机会，KiDispatchException 会试图先通知内核调试器来处理该异常。内核变量 KiDebugRoutine 用来标识内核调试引擎交互的接口函数。当内核调试引擎被启动时，KiDebugRoutine 指向的是内核调试引擎的 KdpTrap, 这个函数会进一步把异常信息封装为数据包发送给内核调试器。当内核调试引擎没有启动时，KiDebugRoutine 指向的是 KdpStub 函数，它的实现非常简单，做一些简单的处理之后便返回 FALSE。

如果 KiDebugRoutine 返回为 TRUE，也就是内核调试器处理了该异常，那么 KiDispatchException 便停止分发，准备返回。如果 KiDebugRoutine 返回 FALSE，那么 KiDispatchException 会调用 RtlDispatchException, 试图寻找已经注册的结构化异常处理器 (SEH)，下图给出了 RtlDispatchException 函数的伪代码：

![]( {{site.url}}/asset/software-debugging-rtldispatchexception.png )

RtlDispatchException 会调用 RtlpGetRegistrationHead 取得异常注册链表 (Exception Registration List) 的首节点地址，接下来，它会遍历异常登记链表，依次执行每个异常处理器，如果某一个处理器返回了 ExceptionContinueExecution, 那么 RtlDispatchException 便会返回 TRUE，表示已经处理了该异常，我们将在下一节详细讨论 SEH 和异常处理器。

如果 RtlDispatchException 返回 FALSE，也就是没有找到该异常的异常处理器，那么 KiDispatchException 会试图给内核调试器第二次处理机会。如果这次 KiDebugRoutine 仍然返回 FALSE，那么 KiDispatchException 会认为这是个无人处理的异常，简称为 “未处理异常(unhandled exception)”。对于发生在内核中的未处理异常，Windows 会认为这是一个严重的错误，会调用 KeBugCheckEx 引发蓝屏，报告错误并终止系统运行。

### 用户态异常的分发过程

如果 PreviousMode 是 UserMode, 那么 KiDispatchException 会执行左侧的路线。

首先，KiDispatchException 会判断是否需要发送给内核调试器，判断的条件包括，这个异常是否是内核调试器触发的，以及内核调试的设置选项中是否接受用户态异常。如果判断条件是需要发送，那么就通过内核调试会话发送给主机上的内核调试器。但内核调试器通常不处理用户态的异常，它会直接返回不处理，因此，大多数时候 KiDispatchException 会继续执行一下的分发过程。

对于第一轮调试机会，KiDispatchException 会试图将该异常分发给用户态的调试器，方法是调用用户态调试子系统的内核例程 DbgkForwardException。

DbgkForwardException 会检查当前进程的 DebugPort 是否为空，如果不为空，则调用 DbgkSendApiMessage 将异常发给调试子系统，后者将异常发给调试器。如果 DbgkSendApiMessage 返回成功，而且调试器处理了该异常，那么 DbgkForwardException 会返回 TRUE，该异常的分发过程也就结束了。

如果 DbgkSendApiMessage 返回结果为不成功，或者调试器没有处理该异常，那么 DbgkSendApiMessage 会返回 FALSE。这种情况下 KiDispatchException 会试图寻找异常处理块来处理该异常，因为异常发生在用户态代码中，异常处理块也应该在用户态函数中，KiDispatchException 会准备转回到用户态中执行。内核变量 KeUserExceptionDispatcher 记录了用户态中的异常分发函数。在目前的 Windows 中，它指向的是 NTDLL 中的 KiUserExceptionDispatcher 函数。

如何从内核态的 KiDispatchException 转回到用户态的 KiUserExceptionDispatcher 函数呢？其过程是这样的，KiDispatchException 会将 CONTEXT 和 `EXCEPTION_RECORD` 结构复制到用户态栈中，然后将 TrapFrame 所指向的 `KTRAP_FRAME` 结构中的状态信息调整为在用户态执行所需的合适值，包括段寄存器和栈指针。最后 KiDispatchException 将 KiUserExceptionDispatcher 的地址赋值给 `KTRAP_FRAME` 结构中的程序指针 (EIP) 字段，目的是让这个线程返回用户态后从 KiUserExceptionDispatcher 开始执行。以上工作做好之后，KiDispatchException 就会返回，当前线程会转到用户态的 KiUserExceptionDispatcher 开始执行。

回到用户态之后，KiUserExceptionDispatcher 会通过调用 RtlDispatchException 来寻找异常处理器，具体细节我们将在下一节介绍，如果 RtlDispatchException 返回 TRUE，表示已经有异常处理器处理了该异常，KiUserExceptionDispatcher 会调用 ZwContinue 系统服务继续执行原来发生异常的代码。如果 ZwContinue 调用成功，便不会再返回到 KiUserExceptionDispatcher 函数中，如果调用失败，KiUserExceptionDispatcher 会通过调用 RtlRaiseException 来抛出异常。

多个异常处理器是以链表的形式连在一起的，在这个链表的尾部总是保存着系统注册的一个默认的异常处理器， RrlDispatchException 会从表头开始遍历这个链表，如果前面的异常处理器都没有处理这个异常，那么最后便会找到默认的异常处理器并执行 UnhandledExceptionFilter 这个函数。

UnhandledExceptionFilter 函数的细节我们将在下一章讨论，目前我们可以简单地认为，如果当前程序没有被调试，那么该函数会将该异常当做未处理异常，系统会对该异常做一些处置措施，包括弹出 “应用程序错误” 对话框后终止这个进程。也就是说，在没有调试的情况下，用户态异常不会经历第二轮分发过程。如果当前进程正在被调试，那么 UnhandledExceptionFilter 会返回 `EXCEPTION_CONTINUE_SEARCH`, 这会导致 RtlDispatchException 返回 FALSE。

如果 RtlDispatchException 返回 FALSE, 也就是没有找到异常处理块愿意处理该异常，而且当前进程正在被调试，那么 KiUserExceptionDispatcher 会调用 ZwRaiseException 并将 FirstChance 设为 FALSE，发起对这个异常的第二次分发。ZwRaiseException 会通过内核服务 NtRaiseException 把该异常传递给 KiDispatchException 来进行分发，这就进入了异常的第二轮分发流程。

KiDispatchException 会先将第二轮处理机会送给调试子系统的 DbgkForwardException 函数。如果该函数返回 TRUE，那么分发结束；如果返回 FALSE，也就是该进程没有在调试，或者调试器没有处理该异常，那么 KiDispatchException 会尝试将该异常分发给该进程的 ExceptionPort 字段指定的端口，通常环境子系统会在创建进程时将该字段设置为子系统监听的一个 LPC 端口。也就是说，对于第二轮分发机会，KiDispatchException 会把该异常分发到该进程的异常端口，给那里的监听者一次处理异常的机会。

如果向 ExceptionPort 发送异常，DbgkForwardException 再次返回 FALSE，那么 KiDispatchException 会终止当前进程。

### 归纳

至此，我们介绍了 Windows 操作系统的异常分发机制和有关的内核函数与 API，总结如下：

内核服务 NtRaiseException 是产生软件异常的主要方法，用户态代码可以通过 RaiseException API 来调用此内核服务。NtRaiseException 内部会调用 KiRaiseException, 后者再调用 KiDispatchException 进行异常分发。

对于 CPU 异常，CPU 会通过 IDT 表来寻找异常的处理函数入口，也就是 KiTrapXX 例程，对于需要按照异常流程分发的异常，KiTrapXX 例程会调用 CommonDispatchException 准备必要的参数，然后调用 KiDispatchException 进行异常分发。

无论是来自用户态的异常，还是内核态的异常，（如果需要分发）系统都会使用 KiDispatchException 函数来分发异常。对于每个异常系统最多会给它两轮被处理的机会，对于每轮机会，KiDispatchException 又都是先尝试让调试器来处理，如果调试器没有处理，那么 KiDispatchException 会寻找代码中的异常处理块来处理该异常。

对于内核态的异常，KiDispatchException 会直接调用 RtlDispatchException 来寻找异常处理块；对于用户态异常，KiDispatchException 会通过设置 TrapFrame 让 KiUserExceptionDispatcher 来寻找用户空间中的异常处理块。

尽管 KiDispatchException 分发内核态异常和用户态异常的流程有所不同，但也有类似之处，比如都会试图先交给调试器来处理，而且最多会给每个异常两轮处理机会。
