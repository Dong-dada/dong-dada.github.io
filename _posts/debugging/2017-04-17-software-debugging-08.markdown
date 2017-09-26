---
layout: post
title:  "《软件调试》 学习 08 中断和异常管理"
date:   2017-04-17 10:06:30 +0800
categories: debugging
---

 
 

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


## 结构化异常处理 (SEH)

为了让系统和应用程序代码都可以简单方便地支持异常处理， Windows 定义了一套标准的机制来规范异常处理代码的设计(对程序员)和编译(对编译器)，这套机制被称为结构化异常处理 (Structured Exception Handling), 简称为 SEH。

从系统的角度看，SEH 是对 Windows 操作系统中的异常分发和处理机制的总称，其实现遍布在 Windows 系统的很多模块和数据结构中。比如 KiDispatchException 函数和 NtRaiseException 函数时位于内核模块中的，KiUserExceptionDispatcher 是位于 NTDLL.DLL 中的，异常注册链表的表头是登记在每个线程的线程信息块 (TEB) 中的。

从编程的角度看，SEH 是一套规范，利用这套规范，程序员可以编写处理代码来复用系统的异常处理设施。可以将其理解为是操作系统的异常机制的对外接口，也就是如何在 Windows 程序中使用 Windows 的异常处理机制。

从使用角度来看，结构化异常处理为程序员提供了终结处理 (Termination Handling) 和异常处理 (Exception Handling) 两种功能。终结处理用于保证终结处理块始终可以得到执行，无论被保护的代码块如何结束。异常处理用于接收和处理被保护块中的代码所发生的异常。

### SEH 的终结处理

终结处理的语法结构如下(以 VC++ 编译器为例)：

```cpp
__try
{
    // 被保护体 (guarded body), 也就是要保护的代码块
}
__finally
{
    // 终结处理块
}
```

显而易见，终结处理有两部分构成，使用 `__try` 关键字定义的被保护体和使用 `__finally` 关键字定义的终结处理块。终结处理的目标是只要被保护体执行，那么终结处理块也就会被执行，除非被保护体中的代码终结了当前线程。因为终结处理块的这种特征，它非常适合做状态恢复或资源释放等工作。比如释放被保护块中获得的信号量以防止被保护块中发生意外时因为没有释放信号量而导致线程死锁。

根据被保护块的执行路线，SEH 把被保护块的退出(执行完毕)分为正常结束和非正常结束两种。如果被保护块正常执行并进入到终结处理块，那么就是正常结束；如果被保护块是因为 发生异常、进入 return, goto, break 等原因离开被保护块，那么就是非正常结束。无论是否正常结束，最终都会进入终结处理块，你可以在终结处理块中调用 AbnormalTermination() 函数来判断被保护块是否正常退出。

除了上面出现的 `__try` 和 `__finally` 关键字，终结处理还有一个关键字 `__leave`，它的作用是立即离开被保护块，或者可以理解为立刻跳转到被保护块的末尾。

下面示例展示了终结处理的几种情况：

```cpp
#include <stdlib.h>
#include <excpt.h>

int main(int argc, char* argv[])
{
    int nNum = 0;
    int nRet = 0;
    const char* WeekDays[] = {"Sunday", "Monday", "Tuesday", "Wednesday", "Thursday", "Firday", "Saturday"};

    __try
    {
        nNum = atoi(argv[1]);

        if (nNum <= 0)
        {
            nRet = -3;
            __leave;        // 使用 __leave 离开被保护块
        }
        
        if (nNum == 6666)
        {
            goto EXIT_BYE;  // 使用 goto 离开被保护块
        }

        if (nNum == 6)
        {
            return -2;      // 使用 return 离开被保护块
        }

                            // 正常离开被保护块
    }
    __finally
    {
        printf("Termination/CleanUp block is executed with %d. \n", AbnormalTermination());
    }

EXIT_BYE:
    printf("Exit!");

    return ret;
}
```

### SEH 的异常处理

异常处理的语法如下：

```cpp
__try
{
    // 被保护体(guarded body)，也就是要保护的代码块
}
__except (过滤表达式)
{
    // 异常处理块 (exception-handling block)
}
```

除了 `__try` 和 `__except` 关键字，VC 编译器还提供了以下两个宏来辅助编写异常处理代码：
- `DWORD GetExceptionCode()`: 返回异常代码，只能在过滤表达式或异常处理块 `__except` 中使用这个宏；
- `LPEXCEPTION_POINTERS GetExceptionInformation()`: 返回一个指向 `EXCEPTION_POINTERS` 结构的指针，该结构包含了指向 CONTEXT 结构和异常记录 (exception record) 结构的指针。只能在过滤表达式中使用这个宏。

其中，`EXCEPTION_POINTERS` 结构的定义如下：

```cpp
typedef struct _EXCEPTION_POINTERS
{
    PEXCEPTION_RECORD ExceptionRecord;      // 异常记录
    PCONTEXT ContextRecord;                 // 异常发生时的线程上下文
}
```

通过 ExceptionRecord 可以获取异常的详细信息，通过 ContextRecord 可以获取异常发生时的线程上下文，包括寄存器取值等。

### 过滤表达式

在 `__except` 关键字后面可以跟一个过滤表达式，对要处理的异常进行过滤。过滤表达式可以是常量、函数调用，也可以是条件表达式或其他表达式，但表达式的结果应该是 0,1,-1 这三个值之一。它们的含义如下：

- `EXCEPTION_CONTINUE_SEARCH(0)`: 本保护块不处理该异常，让系统继续寻找其他异常保护块；
- `EXCEPTION_CONTINUE_EXECUTION(1)`：已经处理异常，让程序回到异常发生的位置继续执行，如果异常导致的情况没有被消除，那么很可能会再次发生异常；
- `EXCEPTION_EXECUTE_HANDLER(-1)`：本保护块要处理该异常，让系统执行异常处理块中的代码；

下面给出一个通过过滤函数修正错误情况后再恢复执行的例子：

```cpp
#include <excpt.h>
#include <windows.h>
#include <stdlib.h>

char g_szDefPara[] = "0123456789";
int ExceptionFilter(LPEXCEPTION_POINTERS pException, char** ppPara)
{
    PEXCEPTION_RECORD pER = pException->ExceptionRecord;
    PCONTEXT = pException->ContextRecord;

    if (**ppPara == NULL && pER->ExceptionCode == STATUS_ACCESS_VIOLATION)
    {
        // 如果 ppPara 为 NULL，则将其设为默认字符串，并返回 EXECPTION_CONTINUE_EXECUTION，让 CPU 重新执行发生异常的指令
        *ppPara = g_szDefPara;
        pContext->Eip -= 3;     // 由于高级语言和汇编语言的差异，需要重设程序指针，具体请参考原文
        return EXCEPTION_CONTINUE_EXECUTION;
    }

    return EXCEPTION_EXECUTE_HANDLER;
}

void FuncA(char* lpsz)
{
    __try
    {
        *lpsz = '2';    // 当 lpsz 为 NULL 时将导致页错误异常
    }
    __except(ExceptionFilter(GetExceptionInformation(), &lpsz))
    {
        printf("Executing handling block in FuncA.\n");
    }
    printf("Exiting from FuncA with lpsz=%s.\n", lpsz);
}

int FuncB(int nPara)
{
    __try
    {
        nPara = 1/nPara;    // 当 nPara 为 0 时将产生除 0 异常

        *(int*)0 = 1;       // 将产生非法访问异常
    }
    __except(GetExceptionCode() == EXCEPTION_ACCESS_VIOLATION ? EXCEPTION_EXECUTE_HANDLER : EXCEPTION_CONTINUE_SEARCH)
    {
        // 如果是非法访问异常，那么将执行异常处理块，否则抛出异常到上层
        printf("Executing handling block in FuncB [%X].\n", GetExceptionCode());
    }
    printf("Exiting from FuncB with Para=%d.\n", nPara);
    return nPara;
}

int main(int argc, char* argv[])
{
    int nRet = 0;

    FuncA(argv[1]);

    __try
    {
        nRet = FuncB(argc-1);
    }
    __except(EXCEPTION_EXECUTE_HANDLER)
    {
        printf("Executing exception handling block in main [%X].\n", GetExceptionCode());
    }

    printf("Exit from main with nRet = %d.\n", nRet);
    return nRet;
}
```

注意 FuncB 的处理中，一开始会产生一个除 0 异常，这个异常进入过滤表达式后，会返回 `EXCEPTINO_CONTINUE_SEARCH`，意思是“我不处理异常，你继续找其他人吧”，因为 main 函数在调用 FuncB 时也使用了 SEH，所以 main 函数会捕捉到这个异常，并进行处理。从调用关系看，由于发生除零异常，CPU 执行完 FuncB 函数之后便继续执行 main 函数了，仿佛是从 FuncB 中飞了出来。这相当于在 FuncB 中多了一个额外的 “函数出口” 而且该出口的位置是不固定的。这种不可预测的 “出口” 是违背结构化编程理念的，会使软件的执行流程变复杂，也给软件调试带来了困难。为了降低异常的负面影响，应该及早捕获和处理异常。

### 嵌套使用终结处理和异常处理

需要注意的是，一个异常保护块 `__try` 不能同时配有终结处理块 `__finally` 和 异常处理块 `__except`，只能通过嵌套的方式来让一段代码同时得到终结处理和异常处理：

```cpp
#include <excpt.h>

void main(void)
{
    __try
    {
        __try
        {
            int n = 0;
            int i = 1/n;
        }
        __finially
        {
            printf("Executing terminating block.\n");
        }
    }
    __except (printf(Executing ExceptiongFilter.\n), EXCEPTION_EXECUTE_HANDLER)
    {
        printf("Executing exception handling block.\n");
    }
}
```

异常发生后，将首先执行异常处理块 `__except`, 然后执行终结处理块 `__finally`。


## 向量化异常处理 (VEH)

除了结构化异常处理，从 xp 开始，Windows 还支持一种名为向量化异常处理 (Vectored Excepiton Handling) 的异常处理机制，简称为 VEH。与 SEH 既可以用在用户态又可以用在内核态不同，VEH 只能用在用户态程序中。

### 登记和注销

VEH 的基本思想是通过注册以下原型的回调函数来接收和处理异常：

```cpp
LONG CALLBACK VectoredHandler (PEXCEPTION_POINTERS ExceptionInfo);
```

其中 ExceptionInfo 是指向 `EXCEPTION_POINTERS` 结构的指针，与 `GetExceptionInformation()` 函数的返回值是相同类型的。VectoredHandler 的返回值是 `EXCEPTION_CONTINUE_EXECUTION`(-1)恢复执行，或者 `EXCEPTION_CONTINUE_SEARCH`(0)继续搜索。

相应的，Windows 公布了两个 API，AddVectoredExceptionHandler 和 RemoveVectoredExceptionHandler 分别用来登记和注销回调函数：

```cpp
PVOID AddVectoredExceptionHandler(ULONG FirstHandler, PVECTORED_EXCEPTION_HANDLER VectoredHandler);

ULONG RemoveVectoredExceptionHandler(PVOID VectoredHandlerHandle);
```

参数 FirstHandler 用来指定该回调函数的被调用顺序，为 0 表示希望最后被调用，为 1 表示希望最先被调用，如果注册了多个回调函数，而且 FirstHandler 都为非零值，那么最后注册的会最先被调用。如果注册成功，返回的是一个系统为该异常处理器分配的 `VEH_REGISTRATION` 指针，程序应当保存这个指针，以便以后注销时使用。

可以看到 `VEH_REGISTRATION` 结构其实是链表的一个节点：

```cpp
typedef struct _VEH_REGISTRATION
{
    _VEH_REGISTRATION* next;
    _VEH_REGISTRATION* prev;
    PVECTORED_EXCEPTION_HANDLER pfnVeh;
} VEH_REGISTRATION, * PVEH_REGISTRAION;
```

NTDLL 中的全局变量 RtlpCalloutEntryList 指向这个链表的头结点

### 调用 VEH

前面几节我们介绍过，在用户态下发生的异常，KiUserExceptionDispatcher 会调用 RtlDispatchException 来寻找异常处理器，在支持 VEH 的系统中，在寻找结构化异常处理器之前，RtlDispatchException 会先调用 RtlCallVectoredExceptionHandlers 给 VEH 优先处理机会。该函数会从前面介绍的 RtlCalloutEntryList 开始遍历 VEH 记录列表。

- 如果 RtlpCalloutEntryList 中的 next 指针指向自身，说明没有注册的 VEH 需要调用，则 RtlCallVectoredExceptionHandlers 返回 FALSE；
- 如果 RtlpCalloutEntryList 中的 next 指针指向了一个 `VEH_REGISTRATION` 结构，那么 RtlCallVectoredExceptionHandlers 会先调用 RtlEnterCriticalSection 防止其他线程访问链表，然后调用 VEH 回调函数，如果该回调函数返回 `EXCEPTION_CONTINUE_SEARCH`, 那么将会寻找下一个 VEH 回调函数，如果找不到，那么返回 FALSE；
- 如果一个 VEH 处理器返回了 `EXCEPTION_CONTINUE_EXECUTION`, 那么 RtlCallVectoredExceptionHandlers 返回 TRUE；

### 归纳

我们归纳一下 VEH 与 SEH 的区别与联系。

从应用范围来看，SEH 既可以用于用户态(比如用户程序)代码中，又可以用于内核态(比如驱动程序)代码中。但 VEH 只能用在用户态代码中。另外 VEH 只能在 XP 及更高版本中才能使用。

从优先级角度看，同时注册了 VEH 和 SEH 的代码所触发的异常，VEH 具有更高的优先级。

从登记方式看，SEH 的注册信息是以固定的结构存储在线程栈中的，不同层次的各个 SEH 的注册信息依次被压入到栈中，分布在栈的不同位置上，依靠结构体内的指针相联系，因为人们经常将一个函数所对应的栈区域称为栈帧(Stack Frame)，所以 SEH 的异常处理器又经常被称为基于帧的异常处理器 (frame-based exception handler)；VEH 的注册信息是保存在进程的内存堆中的。

从作用域的角度看，VEH 处理器相对于整个进程都有效，具有全局性；SEH 处理器是动态建立在所在函数的栈帧上的，会随着函数的返回而注销，因此 SEH 只对当前函数或这个函数所调用的子函数有效。

从编译的角度看，SEH 的登记和注销都是依赖编译器编译时产生的数据结构和代码的，VEH 的注册和注销都是通过调用系统 API  显式完成的，不需要编译器特殊处理。
