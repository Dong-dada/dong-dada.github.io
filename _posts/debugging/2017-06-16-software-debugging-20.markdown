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

### 执行异常处理函数

以下是异常发生后，系统调用和执行 `_raw_seh_handler` 函数的过程：

![]( {{site.url}}/asset/software-debugging-exception-raw-seh-handler.png )



