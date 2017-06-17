---
layout: post
title:  "《软件调试》 学习 20 异常处理代码的编译"
date:   2017-06-16 09:52:30 +0800
categories: debugging
---

* TOC
{:toc}

根据运行模式和编程语言的不同，Windows 系统中的程序可以选择使用不同的异常处理机制，比如驱动程序可以使用结构化异常处理 (SEH) 机制，C++ 语言编写的应用程序可以使用向量化异常处理和 C++ 标准定义的异常处理机制。其中 SEH 是其他异常处理机制的基础，因此，我们将围绕结构化异常处理代码的编译为中心展开讨论。

## 概览

我们之前介绍过 SEH 的用法，概括来说，一段使用 SEH 的异常代码由被保护体、过滤表达式、异常处理块三部分构成：

```c
__try
{
    // 被保护体 (guarded body), 也就是要保护的代码块
}
__except(过滤表达式)
{
    // 异常处理块 (exception-handling block)
}
```

如果被保护体的代码发生了异常，不论是 CPU 级的硬件异常，还是软件发起的软件异常，系统都应该评估过滤表达式的内容，也就是执行过滤表达式中的代码。这意味着，程序的执行路线是从被保护体飞跃到过滤表达式中的。要正确地飞跃到表达式中，就需要准确知道过滤表达式的位置，又要保持栈的平衡。

为了实现这样的飞跃，编译器在编译期间必须产生必要的代码和数据结构，与系统的异常分发机制相配合。

概括说来，首先要分析出异常处理代码的结构，并对每个部分进行必要的封装和标记。然后注册异常处理器函数，以便有异常发生时被调用。记录和注册异常处理器的方式主要有两种：
- 栈帧 (Stack Frame) : 将异常处理器注册到所在函数的栈帧中。通常被称为基于帧的异常处理(Frame Based Exception Handling), 32 位 Windows 使用的是此种方式；
- 表格 (Table) : 将异常处理器的基本信息以表格的形式存储在可执行文件 (PE) 的数据段中。通常被称为基于表的异常处理 (Table Based Exception Handling), 64 位 Windows 使用的是此种方式；

异常处理的另一个特征就是线程相关性。也就是说，异常的分发和处理是在线程范围内进行的，异常处理器的注册也是相对线程而言的。理解这一点对于理解系统寻找和调用异常处理器的方法很重要。


## FS:[0] 链条

Windows 系统中的每一个线程都包含了一个线程环境块 (Thread Environment Block), 简称 TEB。在 TEB 结构的起始处有一个被称为线程信息块 (Thread Information Block) 的结构，简称 TIB。TIB 的第一个字段 ExceptionList 记录了一个表头地址，这个表就是结构化异常处理链表。

在 x86 系统中，段寄存器 FS 总指向线程的 TEB/TIB 结构，也就是说 `FS:[0]` 总指向结构化异常处理链表的表头，所以我们也把这个链表称为 `FS:[0]` 链条。

### TEB 和 TIB 结构

![]( {{site.url}}/asset/software-debugging-exception-tib-struct.png )

可以看到第一个字段就是 ExceptionList. NTDLL 中有一个未公开的函数 NtCurrentTeb, 可以获取到当前线程的 TEB 结构地址。

我们观察下 ExceptionList 结构：

![]( {{site.url}}/asset/software-debugging-exception-registration-record-struct.png )

其中 Next 字段用来指向下一个 `_EXCEPTION_REGISTRATION_RECORD` 结构，Handler 字段用来指向这个异常处理器的处理函数，可以看到这里是 NTDLL 中的 `_except_handler3` 函数，其符合 SEH 处理函数的标准函数原型：

```c
EXCEPTION_DISPOSITION SehHandler(
    _EXCEPTION_RECORD* ExceptionRecord,     // 用来描述要处理的异常
    void* EstablisherFrame,                 // 指向存放在栈帧中的异常登记结构
    _CONTEXT* ContextRecord,                // 用来传递异常发生时的上下文结构
    void* DispatcherContext);               // 供异常分发函数来传递额外信息
```

对 ExceptionList 的值和 StackBase 及 StackLimit 的值进行比较，可以看出 ExceptionList 的值是介于这两个值之间的，也就是说 `FS:[0]` 链条的各个节点是存储在栈上的。

介绍到这里，我们知道系统可以通过 `FS:[0]` 这种便捷方式引用到 ExceptionList 字段，获得一个链表的头指针，这个链表的每个节点是一个 `_EXCEPTION_REGISTRATION_RECORD` 结构，描述了一个结构化异常处理的处理函数 (Handler), 当有异常发生时，系统会依次调用这个链条上的处理函数来分发异常；

概括说来，可以把结构化异常处理看做是操作系统和用户代码协同处理软硬件异常的一种模型。而 `FS:[0]` 链条就是这二者协作的接口。当有异常发生时，操作系统通过 `FS:[0]` 链条寻找异常处理器，给用户代码处理异常的机会。

接下来我们介绍用户代码是如何登记异常处理器的。

### 登记异常处理器

这里先介绍手工插入代码来登记和注销异常处理器的方法，稍后会介绍编译器自动插入这些代码的过程。

要手工登记异常处理器，只需编写一个符合标准 SehHandler 函数原型的函数，然后将其地址记录到 `FS:[0]` 链条中即可：

```asm
push seh_handler    // 先把处理器函数地址 push 到栈中
push FS:[0]         // 把前一个 SEH 处理器函数的地址 push 到栈中
mov FS:[0], ESP     // 登记新结构到链条中，此时 ESP 指向刚刚 push 进去的链条节点地址。
```

从上述代码可以看到，你编写的异常处理器和前一个 SEH 处理器函数的地址都保存在栈中 (也就是一个 `_EXCEPTION_REGISTRATION_RECORD` 结构)，`FS:[0]` 链条中保存的地址是它们在栈中的地址。最后一行代码的作用就是将这个地址更新到 `FS:[0]` 中。

上述代码会将你编写的异常处理器登记到 `FS:[0]` 链条的表头。当 CPU 继续执行这段代码下面的代码时，一旦有异常发生，就会在遍历 `FS:[0]` 链条时找到你编写的异常处理器，给其处理机会。因此，一旦插入以上代码，那么它之后的代码就进入了它所安装的异常处理器的 “保护” 范围，直到这个处理器被注销为止。

可以使用下列代码来注销前面登记的异常处理器：

```asm
mov eax, [ESP]      // 取得前一个异常登记结构的地址
mov FS:[0], eax     // 将前一个异常登记结构的地址更新到 FS:[0] 中
add esp, 8          // 清理栈上的异常登记结构
```

值得注意的是，在注销异常登记结构时，需要保证这中间所执行的代码是栈平衡的。这样上述代码才能正常运行。


## 遍历 FS:[0] 链条

这一节讨论当异常发生时，系统是如何遍历这个链条来寻找和执行其中的异常处理器的。

### RtlDispatchException

简单来说, RtlDispatchException 函数的工作过程就是找到注册在线程信息块中异常处理器链表的头结点，然后依次访问每个节点，调用它的处理器函数，直到有人处理了异常，或者到达链条末尾。

![]( {{site.url}}/asset/software-debugging-exception-rtldispatchexception.png )

注意上述流程图中的 RtlpExecuteHandlerForException 函数，它会执行我们之前登记的异常处理器。它有如下几种返回值：

![]( {{site.url}}/asset/software-debugging-exception-disposition.png )

对于第一种返回值 ExceptionContinueExecution, 如果异常标志中包含了 `EXCEPTION_NONCONTINUABLE`(不可继续)，说明在试图恢复一个不可继续的异常，这时候会调用 RtlRaiseException 再次抛出异常；如果没有包含该标志，那么会继续调用 ZwContinue 服务，让 CPU 返回到异常发生处继续执行。

另外两种返回值很好理解，就是字面上的意思。

如果 RtlDispatchException 遍历到了链表的最后一个节点，那么它会返回 FALSE，我们之前介绍过，每个线程启动的时候都有一个默认的异常处理器，这个异常处理器总是会处理异常，因此一般情况下不会出现 RtlDispatchException 返回 FALSE 的情况。


## 执行异常处理函数

前面两节分别介绍了如何注册 SEH 处理函数，以及当有异常发生时系统如何遍历 `FS:[0]` 链条寻找 SEH 处理函数。本节介绍系统执行 SEH 处理函数的细节。

### SehRaw 实例

为了方便理解，我们编写了如下函数，它用之前介绍过的手工方法来注册 SEH 处理器：

```c
#include "stdafx.h"

// 自己定义的异常处理器
EXCEPTION_DISPOSITION __cdecl _raw_seh_handler(
    struct _EXCEPTION_RECORD* ExceptionRecord,
    void* EstablishFrame,
    struct _CONTEXT* ContextRecord,
    void* DispatcherContext)
{
    printf("_raw_seh_handler: code-0x%x, flags-0x%x\n", ExceptionRecord->ExceptionCode, ExceptionRecord->ExceptionFlags);

    // 如果是除 0 异常，那么将被除数 ECX 设为 10，并继续执行
    if (ExceptionRecord->ExceptionCode == STATUS_INTEGER_DIVIDE_BY_ZERO)
    {
        ContextRecord->Ecx = 10;
        return ExceptionContinueExcution;
    }

    // 否则继续执行其他异常处理器你
    return ExceptionContinueSearch;
}

int main()
{
    __asm
    {
        // 登记异常处理器到 FS:[0] 链表
        push    OFFSET _raw_seh_handler
        push    FS:[0]
        mov     FS:[0], ESP

        xor     edx, edx    // 将 EDX 寄存器清 0
        mov     eax, 100    // 设置 EAX 寄存器为 100
        xor     ecx, ecx    // 设置 ECX 寄存器为 0
        idiv    ecx         // EAX 除以 ECX, 结果保存到 EDX 中

        // 从 FS:[0] 链表中注销异常处理结构
        mov     eax,[ESP]
        mov     FS:[0], EAX
        add     esp, 8
    }

    printf("SehRaw exits\n");
    return 0;
}
```

## __try{}__except() 结构

之前我们介绍了用手工方法来登记异常处理器，现在看看编译器怎样将 `__try{}__except()` 编译为对应的异常处理代码。

概括说来，`__try{}__except()` 结构就是将手动方法中的函数和嵌入式汇编代码简化成高级语言中的标记符和表达式。具体说来, `__try{}` 中的被保护块对应于我们手工添加的登记和注销代码；`__except()` 中的过滤表达式对应于我们在 `_raw_seh_handler` 中编写的 `if` 判断语句：

![]( {{site.url}}/asset/software-debugging-exception-try-except.png ) 

可以看到使用 `__try{}__except()` 大大简化了异常处理代码的编写，但其在汇编级别跟我们的代码没啥差别。

### __try{}__except() 结构的编译

先看一个用 C/C++ 编写的 TrySeh 函数：

```c
int TrySeh(int n)
{
    __try
    {
        n = 1/n;
    }
    __except(EXCEPTION_EXECUTE_HANDLER)
    {
        n = 0x122;
    }
    return n;
}
```

以下是该函数编译后的汇编代码：

![]( {{site.url}}/asset/software-debugging-exception-try-seh-sample.png )

可以看到第 3~8 行是在登记 异常处理器。

有一点值得注意的是这里自动安装了一个异常处理器 `_except_handler3`, 它是编译器统一使用的异常处理函数，VC6 使用的是 `_except_handler3`, VS2005 使用的是 `_except_handler4`。这就有个问题，统一的异常处理器是如何满足不同 SEH 异常代码块的需要的呢？答案是通过传递给 `_except_handler3` 的参数来区分。

但异常处理函数在分发过程中时没有带上我们需要的参数的： RtlDispatchException --> ExecuteHandler --> ExecuteHandler2 --> `_except_handler3` 那么这些参数是怎么传递过去的呢？

答案在汇编代码的第 5~7 行，可以看到编译器往栈上压入了一个 trylevel 整数和一个 `scopetable_entry` 指针。这样在栈上实际上就形成了一个如下图所示的 `_EXCEPTION_REGISTRATION` 结构：

```c
struct _EXCEPTION_REGISTRATION
{
    // 指向前一个 _EXCEPTION_REGISTRATION 结构
    struct _EXCEPTION_REGISTRATION* prev;

    // 处理函数，也就是 _except_handler3
    void (*handler)(PEXCEPTION_RECORD, PEXCEPTION_REGISTRATION, PCONTEXT, PEXCEPTION_RECORD);

    // 以下是增加的字段
    struct scopetable_entry* scopetable;    // 范围表的起始地址
    int trylevel;                           // 行数当前 __try 块的编号
    int _ebp;                               // 栈帧的基地址
}
```

接下来我们看看传给 `_except_handler3` 的新增字段起什么作用。

### 范围表

为了描述程序代码中的 `__try{}__except()` 结构，编译器会在编译每个使用此结构的函数时为其建立一个数组，并存储在模块文件的数据区中，通常被称为异常处理范围表。数组的每个元素都是一个 `scopetable_entry` 结构，用来描述一个 `__try{}__except()` 结构：

```c
struct scopetable_entry
{
    DWORD   previousTryLevel;   // 上一个 __try{} 结构的编号
    FARPROC lpfnFilter;         // 过滤表达式的起始地址
    FARPROC lpfnHandler;        // 异常处理块的起始地址
}
```

观察上一小节中汇编代码的第 4 行，可以看到这次压入的 `scopetable_entry` 结构的地址是 `0x004070e0`, 再来看看它里面存储的内容：

![]( {{site.url}}/asset/software-debugging-exception-try-seh-sample2.png )

可以看到过滤表达式 lpfnFilter 的地址是 `0x0040103b`, 也就是代码中第 20,21 行的内容，其对应的指令是 `mov eax,1`, 这与我们用 C 语言编写的过滤表达式 `EXCEPTION_EXECUTE_HADNLER`(1) 正好对应。

而异常处理块 lpfnHandler 的地址是 `0x00401041`, 也就是代码中第 22,23 行的内容。其指令与我们用 C 语言编写的异常处理代码也是对应的。

### TryLevel 

编译器是以函数为单位来登记异常处理器的，在函数的入口处进行登记，在出口处注销。如果有多个 `__try` 块，当异常发生的时候，如何判断发生异常的代码属于哪一个 `__try` 块呢？

这就需要对每个 `__try` 块进行编号，然后用一个全局变量来记录当前处于哪个 `__try` 块中，这个局部变量被称为 trylevel, 也就是栈上 `_EXCEPTION_REGISTRATION` 结构的 trylevel 字段。

这个编号是从 0 开始的，常量 `TRYLEVEL_NONE(-1)` 代表当前不在任何 `__try` 结构中。也就是说 trylevel 变量被初始化为 -1, 执行到 `__try` 结构中时便将当前 `__try` 块的编号赋值给它，离开 `__try` 块后，trylevel 会被重置为 -1.

查看汇编代码中的第 3 行，可以看到 trylevel 被初始化为 -1, 当执行到 `__try` 块中时，会将 trylevel 设置为 0 (第 14 行)，再函数出口处(第 24 行)，又将 trylevel 设置为 -1.

我们来看看下面更复杂的情况：

```c
int TryLevel(int n)
{
    __try
    {
        n = n - 1;
        __try
        {
            n = 1/n;
        }
        __except(EXCEPTION_CONTINUE_SEARCH)
        {
            n = 0x221;
        }

        n++;
        __try
        {
            n = 1/n;
        }
        __except(EXCEPTION_CONTINUE_SEARCH)
        {
            n = 0x222;
        }
    }
    __except(EXCEPTION_EXECUTE_HANDLER)
    {
        n = 0x223;
    }

    return n;
}
```

上述代码中包含了 4 个 `__try` 块，对应有 4 个 `scopetable_entry` 结构：

![]( {{site.url}}/asset/software-debugging-exception-try-seh-sample3.png )

上图是异常处理表中每个元素的值，其中第 2 列就是 `scopetable_entry` 的 previousTryLevel 字段，可以看到第 1, 4 行的第 2 列都是 -1, 表明第 1,4 个 `__try` 块是顶层 try 块, 而第 2,3 行的第 2 列是 0, 表明他们是第一个 try 块的子块。

### __try{}__except() 结构的执行

我们来看看异常发生后 `_except_handler3` 中会执行那些操作：

第一，将第二个参数 pRegistrationRecord 从系统默认的 `EXCEPTION_REGISTRATION_REOCRD` 强制扩展为包含扩展字段的 `_EXCEPTION_REGISTRATION` 结构。

第二，从 pRegistrationRecord 中取出 trylevel 的值并且赋值给一个局部变量 nTryLevel, 然后根据 nTryLevel 找到对应的 `scopetable_entry` 结构；

第三，从 `scopetable_entry` 结构中取出 lpfnFilter 字段，如果不为空，则调用该函数，如果为空，则调到第 5 步；

第四，如果 lpfnFilter 的返回值不是 `EXCEPTION_CONTINUE_SEARCH`, 则准备执行 lpfnHandler 字段所指定的函数，并且不再返回。否则直接调到第 5 步：

第五，判断 `scopetable_entry` 结构的 previousTryLevel 字段，如果不等于 -1, 那么将 previousTryLevel 赋值给 nTryLevel 并返回到第 2 步继续循环；如果 previousTryLevel 等于 -1，那么执行第 6 步；

第六，返回 `DISPOSITION_CONTINUE_SEARCH`，让系统 (RtlDispatchException) 继续寻找其它异常处理器。
