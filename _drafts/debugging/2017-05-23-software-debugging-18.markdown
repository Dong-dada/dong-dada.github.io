---
layout: post
title:  "《软件调试》 学习 18 栈和函数调用"
date:   2017-05-23 09:53:30 +0800
categories: debugging
---

* TOC
{:toc}


栈 (stack) 和 函数调用 (function call) 这两项技术是支持软件大厦的基石，很难找到那个软件没有使用 栈 和 函数调用的。

本章先介绍栈的基本概念和栈的创建过程，然后分别介绍栈在函数调用和局部变量分配方面所起的作用，以及栈帧的概念、帧指针省略和栈指针检查。之后将详细介绍调用协定。栈空间的分配和自动增长机制。栈有关的安全问题，以及编译器所提供的检查和保护机制。


## 简介

从数据结构角度看，栈是一种用来存储数据的容器 (container)。放入数据的操作被称为压入，从栈中取出数据的操作被称为弹出。存取数据的一条基本规则是后进先出 (last-in-first-out, LIFO)。

对基于栈的计算机系统而言，栈是存储局部变量和进行函数调用所必不可少的连续内存区域。编译器、操作系统和 CPU 按照规范各尽其责，保证正确合理地使用栈。编译器在编译时会将函数调用和局部变量存取编译为合适的栈操作；操作系统在创建线程时，会为每个线程创建栈，包括分配栈所需的内存空间和初始化有关的数据结构及寄存器；CPU 在执行程序时，会假定 SS 寄存器(Stack Segment 用来描述栈所在的内存段) 和 ESP 寄存器(Extended Stack Pointer 用来记录栈的栈顶地址) 已经指向一个设置好的栈，执行 PUSH 指令时便向 ESP 所指向的内存地址写入数据，然后调整 ESP 的值，使其指向新的栈顶，执行 POP 指令时便从栈顶弹出数据，并会向栈底方向调整 ESP 寄存器的值，保证它们始终指向栈的顶部。当 CPU 使用 CALL 和 RET 这样的函数调用指令时，它也会使用栈。

从线程角度看，栈是每个 Windows 线程的必备设施。在 Windows 系统中，每个线程至少有一个栈，系统线程之外的每个线程有两个栈，一个供该线程在用户态下执行时使用，称为用户态栈；另一个供该线程在内核态下执行时使用，称为内核态栈。在一个运行着多任务的系统中，因为有很多个线程，所以会有多个栈存在。尽管有如此多的栈，但对于 CPU 而言，它只是用当前栈，即 SS 和 ESP 寄存器所指向的栈。当进行不同任务间的切换，以及同一任务内的内核态与用户态之间的切换时，系统会保证 SS 和 ESP 寄存器始终指向合适的栈。

另外值得说明的是，在 x86 系统中，**栈是朝低地址方向生长的**。也就是说，压栈操作会导致 ESP 寄存器(栈指针)的值减小，弹出操作会导致 ESP 值变大。这也意味着，后压入数据的地址总是比先压入数据的地址小。

### 用户态栈和内核态栈

一个线程可能在不同的特权级 (privilege) 下执行，比如用户态的代码调用系统服务时，该线程便会切换到系统模式下执行，所调用的系统服务执行完毕后再切换回用户态执行 (这个过程通常被称为 Context Switch)。如果让用户态的代码和系统服务共用同一个栈，那么必然会存在安全问题。为了保证不同优先级的代码和数据的安全，线程在不同优先级下运行时会使用不同的栈。

x86 系统中有四个特权级，Windows 只使用了其中的两种，所以每个普通的 Win32 线程都有两个栈，一个供线程在内核态下使用，称为内核态栈 (kernel-mode stack)，另一个供线程在用户态下执行时使用，称为用户态栈 (user-mode stack)。对于那些只运行在内核模式下的线程，没有用户态栈，因为它们不需要再用户模式下运行。

CPU 在执行跨越特权级的代码转移时，会自动切换到不同的栈，那么 CPU 是如何找到每个特权级应该使用的栈的呢？答案是每个任务的任务状态段 (TSS) 记录了不同优先级所使用的栈的基本信息。在 TSS 中，偏移 4 到 28 的 24 个字节是用来记录栈的段信息 (SS) 和 栈指针 (ESP) 值的。

在 Windows 为每个线程所维护的基本数据结构中，记录了内核态栈和用户态栈的基本信息。内核态栈记录在 `_KTHREAD` 结构中，用户态栈记录在 `_TEB` 结构中。

每个线程都有一个名为 `_KTHREAD` 的数据结构，该结构位于内核空间中，是 Windows 系统管理、记录线程信息、进行线程调度的重要依据。在这个结构中，有几个字段专门用来记录栈信息：
- StackBase : 内核态栈的基地址；
- StackLimit : 内核态栈的边界；
- LargeStack ：是否已经切换为大内核态栈；
- KernelStack ：内核态栈的栈顶地址，用于保存栈顶 ESP；
- KernelStackResident ：内核态栈是否位于物理内存中；
- InitialStack ：供内核态代码逆向调用用户态代码时记录本来的栈顶位置；
- CallbackStack ：也是在逆向调用时使用；

用户态栈的基本信息记录在线程信息块 `_NT_TIB` 结构中，`NT_TIB` 是线程环境块结构 (TEB) 的一部分，因此可以根据 TEB 的地址来显示 `_NT_TIB` 结构。这里就不单独介绍它了。

内核态栈和用户态栈有一个很明显的区别，用户态栈是可以指定大小的，默认大小为 1MB；而内核态栈完全由系统来控制其大小，其大小因处理器结构不同而不同，但通常在十几 KB 到 几十 KB 之间。考虑到 GUI 线程在调用 GDI 等内核服务时通常需要更大的内核态栈，所以在一个线程被转变为 GUI 线程后，Windows 会为其创建一个较大的可增长的内核态栈来替换掉原来的栈，称为大内核态栈。

### 函数、过程和方法

在软件工程中，函数 (function), 过程 (procedure 或 subroutine) 和 方法 (method) 这三个概念经常被替换使用，因为它们都可以用来指代一段可以被调用的程序代码。

尽管以上三个术语有细微的差异，但本章统一使用函数这一术语来泛指以上三个概念。


## 栈的创建过程

### 内核态栈的创建

PspCreateThread 是 Windows 内核中用于创建线程的一个重要内部函数。无论是创建系统线程 (PsCreateSystemThread) 还是用户线程 (NtCreateThread 服务)，都离不开这个函数。除了创建重要的 ETHREAD 结构， PspCreateThread 的另一个重要任务就是创建内核态栈。

之前我们说过，对于 GUI 线程，Windows 会为其创建的大内核栈。但线程刚刚被创建时，并不是 GUI 线程。Windows 会在创建线程时先统一调用 MmCreateKernelStack 创建一个默认大小的内核态栈。当一个线程别转化为 GUI 线程时，系统的 PsConvertToGuiThread 会给该线程重新创建一个大内核栈，然后使用 KeSwitchKernelStack 切换到新的栈。这个栈是可以改变大小的，当需要时，调用 MmGrowKernelStack 来增长栈，每次增长的幅度至少为一个页面 (4KB)。

内核态的代码可以调用 IoGetStackLimits 函数和 IoGetRemainingStackSize 函数分别得到当前栈的边界和剩余大小。

### 用户态栈的创建

Windows 线程的用户态栈是由 KERNEL32.DLL 中的 BaseCreateStack 函数创建的。

对于进程的初始线程来说，NtCreateProcess 会在调用 NtCreateThread 之前先调用 BaseCreateStack 函数创建栈。

对于初始线程之外的其他线程来说，CreateThread 会调用 CreateRemoteThread 函数，然后 CreateRemoteThread 会在调用内核服务 NtCreateThread 前，先调用 BaseCreateStack 来创建栈。

BaseCreateStack 是一个未公开的函数，其原型大致如下：

```cpp
NTSTATUS BaseCreateStack(
    IN HANDLE hProcess,
    IN DWORD dwCommitStackSize,
    IN DWORD dwReservedStackSize,
    OUT PINITIAL_TEB pInitialTeb)
```

BaseCreateStack 将在 hProcess 参数所指定的进程地址空间中根据 dwCommitStackSize 参数所指定的大小提交一部分作为栈的初始空间。它会把所保留和提交内存区域的参数保存在 pInitialTeb 结构中。这个结构会被传给 NtCreateThread 内核服务，最终被保存到线程的环境块(TEB)中。

dwReservedStackSize 用来指定要创建的栈的保留内存区大小，dwCommitStackSize 用来指定要创建的栈的已经提交的内存区大小。前者指明了为栈保留的最大内存地址空间，后者指明了初始提交的内存空间大小，后者属于前者的一部分。保留空间只是一个地址范围，在使用前还是得进行提交，提交时系统才真的进行内存分配。

进一步来说，保留和提交内存都是通过系统的虚拟内存分配函数来完成的，SDK 中公开了 VirtualAlloc 和 VirtualAllocEx API，事实上它们都是调用内核服务 NtAllocateVirtualMemory:

```cpp
NTSTATUS NtAllocateVirtualMemory(
    IN HANDLE hProcessHandle,
    IN OUT PVOID lpBaseAddress, // 地址指针
    IN ULONG ZeroBits,
    IN OUT PULONG plRegionSize, // 区域大小
    IN ULONG flAllocationTypes, // 要分配的内存类型, MEM_RESERVE 是保留内存，MEM_COMMIT 是提交内存
    IN ULONG flProtect);        // 保护属性，如 PAGE_READONLY, PAGE_READWRITE
```

可以把 BaseCreateStack 创建用户态栈的过程归纳为如下几个重要步骤：
1. 提交空间大小取为内存页大小的倍数，将总保留大小取整为内存分配的最小粒度 (4 字节)；
2. 调用内存分配函数 (NtAllocateVirtualMemory) 保留内存地址空间，内存分配类型为 `MEM_RESERVE`，分配大小为栈空间保留大小；
3. 调用 NtAllocateVirtualMemory 为保留空间的高地址端提交初始栈空间，内存分配类型为 `MEM_COMMIT`，分配的大小为初始提交大小；
4. 如果保留空间大于初始提交空间，则第 3 步会多提交一个页面用作栈保护页面。保护的方法是调用虚拟内存保护函数 `VirtualProtect` 对这个页面设置 `PAGE_GUARD` 属性；

其中第 4 步创建的栈保护页面是实现栈自动增长功能所必须的，我们后续会介绍栈增长机制。

下图展示了刚创建好的栈的示意图，此时栈的基地址为 0x130000, 栈的边界为 0x12f000, 栈中可使用空间为 4KB, 栈的总大小为 1MB。

![]( {{site.url}}/asset/software-debugging-create-stack.png )

地址 0x12f000 和 0x12e000 之间的一个页是保护页，当已经提交的栈空间用完触及到保护页时，系统的栈增长机制会提交更多空间并移动保护页，右侧的图表示栈自动增长一个内存页之后的情形。


## CALL 和 RET 指令

在基于 x86 处理器的系统中，CALL 和 RET 指令是专门用来进行函数调用和返回的。理解这两条指令有助于我们深刻理解函数调用的内部过程和栈的使用方法。

### CALL 指令

CALL 指令是 x86 CPU 中专门用来用作函数调用的命令，简单来说，它的作用就是将当前的函数指针 (EIP 寄存器的值) 保存到栈中 (称为 linking information)，然后转移到 (branch to) 目标操作数所指定的函数 (被调用过程) 继续执行。

根据被调用过程是否位于同一个代码段，CALL 调用被分为进调用 (Near Call) 和远调用 (Far Call) 两种，对于 近调用，CPU 所执行的操作如下：
1. 将 EIP 寄存器的当前值压入栈中供返回时使用；
2. 将被调用过程的偏移 (相当于当前段) 加载到 EIP 寄存器中；
3. 开始执行被调用过程；

对于远调用，CPU 所执行的操作如下：
1. 将 CS 寄存器的当前值压入到栈中供返回时使用；
2. 将 EIP 寄存器的当前值压入到栈中供返回时使用；
3. 将包含被调用过程的代码段的段选择子加载到 CS 寄存器中；
4. 将被调用过程的偏移加载到 EIP 寄存器中；
5. 开始执行被调用过程；

可以看到远调用相对于近调用来说多了一个 CS 寄存器的处理，因为近调用是一个代码段内的调用，因此不需要向栈中压入和切换到代码段，而远调用因为发生在不同代码段间，因此需要保存和切换代码段。

对于 NT 系列的 Windows, 因为使用了平坦内存模型，同一进程内的代码都在一个大的 4GB 段中，因此不需要考虑段的差异，几乎所有时候使用的都是近调用。

### RET 指令

RET 指令用于从被调用过程返回到发起调用的过程。RET 指令可以有一个可选的参数 n, 用于指定 ESP 寄存器要递增的字节数，ESP 递增 n 个字节相当于从栈中弹出 n 个字节，经常用来释放压在栈上的参数。相对于近调用的返回被称为近返回，相对于远调用的返回被称为远返回。

对于近返回，CPU 所执行的操作如下：
1. 将位于栈顶的数据弹出到 EIP 寄存器，这个值应该是发起近调用时 CALL 指令压入的返回地址；
2. 如果 RET 指令中包含参数 n, 那么便将 ESP 寄存器的字节数增加 n ；
3. 继续执行程序指针所指向的指令，通常就是父函数中调用指令的下一条指令；

从以上过程我们可以看到，RET 指令只是单纯地返回到执行这条指令时栈顶所保存的地址，如果栈寄存器 (ESP) 没有指向合适的位置或者栈上的地址被破坏了，那么 RET 指令就会返回到其他地方，这也正是缓冲区溢出攻击的基本原理，我们稍后会详细讨论它。


## 局部变量和栈帧

编译器在为局部变量分配空间时通常有两种做法：使用寄存器和使用栈。我们把分配在寄存器中的局部变量称为寄存器变量，把分配在栈上的局部变量称为栈变量。

从性能上来看使用寄存器来分配局部变量是最好的。但寄存器的空间和数量都是有限的，所以编译器在优化的时候会尽量把频繁使用的变量放到寄存器中。

对于调试版本来说，编译器会在栈上分配所有局部变量。本节讨论的重点也是栈变量。

### 局部变量的分配和释放

简单来说，局部变量的分配和释放是由编译器插入的代码通过调整栈指针 (Stack Pointer) 的位置来完成的。

编译器在编译的时候，会计算当前代码块 (如函数中) 所声明的所有局部变量所需要的空间，并将其按照内存对齐规则取值为满足对其要求的最接近整数值。例如在 32 位系统中，内存分配是按照 4 字节对其的，这意味着不满 4 字节的空间分配会按照 4 字节来分配。距离来说，如果某个 ANSI 类型的字符数组长度是 13, 那么实际上会分配 16 个字节的栈空间给它。

计算好空间之后，编译器会插入适当的指令来调整栈指针，为局部变量分配出空间来。对于 x86 系统而言，栈指针保存在 ESP 寄存器中，所以调整栈指针实际上也是调整 ESP 寄存器的值。

编译器有几种方法来调整 ESP 寄存器的值，一种方法是对其进行加减运算：

```asm
sub esp, 10h        ; 将 ESP 寄存器的值减去 16, 也就是增加 16 字节的栈空间；
add esp, 10h        ; 将 ESP 寄存器的值加上 16, 也就是减小 16 字节的栈空间；
add esp, 0FFFFFCC   ; 加上一个负数 (-34)，相当于减去 34；
```

另一种方式是使用 PUSH 和 POP 指令，这两个指令的执行速度比 sub 和 add 要快，如果只需要一两个 PUSH 和 POP 就能干的事儿，编译器会尽量使用它，而不是使用 add 和 sub. 

另外注意到栈是向低地址方向生长的，所以分配空间的时候，是使 ESP 寄存器递减，释放空间是使 ESP 寄存器递增。

下面看个例子：

```c
int FuncA()
{
    int l, m, n;
    char sz[] = "Advanced SW Debugging";
    l = sz[0];
    m = sz[4];
    n = sz[8];
    return l*m*n;
}
```

函数 FuncA 中定义了 4 个局部变量，其中 3 个整形变量占 12 个字节，一个字符串变量占 22 个字节。我们来看看它的反汇编代码 (release 版本)：

![]( {{site.url}}/asset/software-debugging-local-variable-stack.png )

可以看到上述代码的第二行，`sub esp 0x18` 这条指令，分配了 24 个字节的栈空间。这个空间是给字符串变量分配的，它本来是 22 字节，因为字节对齐而被扩展为 24 个字节。这里没有为 3 个整型变量分配栈空间，因为是 release 版本，编译器会尽可能将局部变量存储到寄存器上。

接下来 4 到 5 行把 esi 和 edi 寄存器中的内容保存到了栈上，因为接下来将要用这两个寄存器来初始化字符串变量。

第 6 行把字符串常量 `"Advanced SW Debugging"` 的地址赋值给了 esi 寄存器。

第 7 行把栈上 sz 变量的地址赋值给了 edi 寄存器，因为第 4 行和第 5 行压入了两个 4 字节的值到栈上，所以现在 sz 的地址是栈顶地址加 8, 也就是 `esp + 0x8`；

第 8 行是个循环指令，循环的次数记录在 ecx 寄存器中，之前第 3 行已经把 ecx 设定为 5, 所以会循环 5 次，每次循环都会把 4 个字节的内容拷贝到 sz 变量中，也就是会拷贝 20 个字节；

第 9 行拷贝了 2 个字节的内容到 sz 变量中，补足了 sz 变量的长度 (22)；

第 10 ~ 12 行，是为 l, m, n 这三个局部变量赋值的过程，将 sz 变量的第 0, 4, 8 位值赋值给这三个变量，这三个变量依次存储在 EDX, ECX, EAX 这三个寄存器中；

第 13 行和 第 15 行执行了乘法运算，运算结果最终保存在了 EAX 寄存器中；

第 14 行和 第 16 行将栈中的 edi 和 esi 寄存器信息恢复了回去；

第 17 行的 `add esp 0x18` 用来释放栈空间，对应于一开始的 `sub esp 0x18` 指令，可以看到栈是平衡的；

第 18 行调用 ret 指令返回到调用方，这时候栈顶存放的是调用 FuncA 时的 EIP 的信息，ret 会根据这个地址来返回到原来的调用处，然后弹出这个地址信息，将栈顶恢复成原调用方的情况；

### EBP 寄存器和栈帧

对于分配在栈上的局部变量，编译器是如何来引用它们的呢？对于前面 FuncA 中的 sz 变量，第 7 行是通过 `esp + 0x8` 来引用它的，也就是以 ESP 寄存器作为参照物来引用局部变量 sz. 这种做法的缺点是不稳定，ESP 的值可能会发生变化，这时候对 sz 的引用也会发生变化。例如下面的 FuncB 函数：

```c
void FuncB(char* szPara)
{
    char szTemp[5];
    strncpy(szTemp, szPara, sizeof(szTemp) - 1);
    printf("%s; Len = %d.\n", szTemp, strlen(szTemp));
}
```

我们来看看它的反汇编代码 (release) 版本：

![]( {{site.url}}/asset/software-debugging-local-variable-stack-example2.png )

可以看到上述代码中，有三个地方都引用了 szTemp, 这三次引用分别使用的是 esp, esp+0x10, esp+0x4, 之所以会这样是因为在这三次引用中间存在多条 push, pop 指令，导致 esp 的值一直在变化。

如果想通过反汇编来调试上述代码，我们不得不时时注意 esp 的变化，才能知道当前表示 sz 的值是多少，这太麻烦了。

为了解决上述问题，x86 CPU 设计了 EBP 寄存器，它的全称是 Extended Base Pointer, 即扩展的基址指针。它记录了函数自己栈空间的基准地址，换句话说，就是刚刚进入函数时的栈顶地址。可以用它来引用局部变量和参数，在同一函数内，EBP 寄存器的值是保持不变的，这样函数内的局部变量便有了一个固定的参照物。

通常，一个函数在入口处将当时的 EBP 值压入堆栈，然后把 ESP 值(栈顶)赋给 EBP，这样 EBP 中的地址就是进入本函数时的栈顶地址，这一地址上面(递减方向)接下来的空间就是该函数将要使用的栈空间，它的下面(递增方向)就是父函数的栈空间。这样设置了 EBP 之后，就可以通过 EBP + n 的方式来引用父函数栈空间中的内容，使用 EBP - n 的方式来引用本函数栈空间中的内容。

因为在将栈顶地址 ESP 的值赋值给 EBP 寄存器之前，会先把旧的 EBP 值保存在栈中，所以 EBP 所指向的栈单元中保存的是前一个 EBP 寄存器的值，通常也就是父函数的 EBP 的值。类似的，复函数中 EBP 的值指向的是更上一层函数的 EBP 值，依次类推，直到当前线程的最顶层函数。这也正是栈回溯的基本原理。

现在看个例子，把之前的 FuncB 改名为 FuncC, 并且在函数前面加上 `#pragma optimize("", off)`, 在后面加上 `#pragma optimize("", on)`, 从而屏蔽优化。以下是它的反汇编代码：

![]( {{site.url}}/asset/software-debugging-local-variable-stack-ebp.png )

可以看到上述例子中，每次对 szTemp 变量的引用，其地址都是 `[ebp-0x8]`, 而不是像之前 FuncB 那样每次是不同的值。

下图画出了当 CPU 执行 FuncC 函数时栈的状态：

![]( {{site.url}}/asset/software-debugging-local-variable-stack-funcc.png )

从上面的分析我们可以看到，尽管栈中的数据是连续存储的，好像所有数据都混作一团，但事实上，它们是按照函数调用关系依次存放的，而且这种关系非常严格。为了更好地描述和指代栈中的数据，我们把每个函数在栈中所使用的区域成为一个栈帧(Stack Frame)。在上图中，共有 4 个栈帧，分别属于 Func, main, mainCRTStartup, BaseProcessStart 函数。

关于栈帧还有以下几点值得说明：
1. 在一个栈中，依据函数调用关系，发起调用的函数 (Caller) 的栈帧在下面(高地址方向), 被调用的栈帧在上面；
2. 每发生一次函数调用，便产生一个新的栈帧，当一个函数返回时，这个函数所对应的栈帧被消除 (eliminated)；
3. 线程正在执行的那个函数所对应的栈帧位于栈的最顶部，它也是栈内仍然有效的最年轻(建立时间最晚)栈帧；

我们可以归纳出建立栈帧的典型指令序列：

![]( {{site.url}}/asset/software-debugging-stack-frame-begin.png )

在函数的出口，通常有对应的消除栈帧的指令序列：

![]( {{site.url}}/asset/software-debugging-stack-frame-end.png )

### 帧指针和栈帧的遍历

指向每个栈帧的指针被称为帧指针 (Frame Pointer), 因为在 x86 系统中通常使用 EBP 寄存器作为帧指针使用，所以经常会使用 ChildEBP 或 EBP 来指代帧指针。

当一个函数建立一个新的栈帧时，它会将当时的 EPB 值压入栈保存起来，然后立刻把当时的栈指针赋值给 EBP 寄存器。这样 EBP 的值便是新栈帧的基础地址，而这个地址在栈中的值便是 EBP 的以前值，也就是前一个栈帧的基础地址。以此类推，我们可以遍历整个栈中的所有栈帧，这也正是调试器的 Calling Stack 功能 (显示函数调用序列) 的基本原理。

这样从当前栈帧层层追溯而得到函数调用记录的过程被称为栈回溯 (Stack Backtrace)。栈回溯信息对软件调试有着非常重要的意义，比如可以通过参数值核对参数的正确性，根据帧指针观察局部变量，根据程序指针信息了解函数调用关系。因此，大多数调试器都提供了显示栈回溯信息的功能。比如 WinDBG 提供了一系列以 k 开头的命令 (k, kb, kd, kp, kP, kv) 来显示各种格式的栈回溯信息。


## 帧指针省略(FPO)

上一节我们介绍了栈帧的概念和用于标志栈帧位置的帧指针。需要注意的是，并不是所有函数都会建立帧指针，某些优化过的函数省去了建立和维护帧指针所需的命令，所以这些函数所对应的栈帧不再含有帧指针，这种情况被称为帧指针省略 (Frame Pointer Omission), 简称 FPO；

之前 FuncB 的反汇编代码就使用了 FPO, 从其反汇编代码中我们找不到建立和恢复栈帧的指令。

从理论上讲，使用 FPO 可以省略一些指令，减小目标文件，提高运行速度，因此 FPO 称为速度优化和空间优化的一种方法。VC6 编译器在编译 release 版本时默认会开启 FPO 选项；

在使用 FPO 的函数中，因为没有固定位置的帧指针可以参考，所以必须使用其他参照物来引用局部变量，在 x86 系统中，通常会使用 ESP 寄存器，正如我们之前所说，ESP 可能会经常变化，这给调试增加了难度。

使用 FPO 的另一个副作用就是对于采用 FPO 优化的函数，因为没有了帧指针，所以给生成栈回溯信息带来了不便。

当执行使用 FPO 优化过的函数时，EBP 指针指向的仍然是前一个栈帧，而不是当前栈帧。这使得我们很难为这样的函数产生帧信息。为了解决这一问题，编译器在生成调试符号的时候，会为使用 FPO 的函数产生 FPO 信息。利用调试符号中的 FPO 信息，调试器可以为省略帧指针的函数生成回溯信息。

利用 WinDBG 加载之前 FuncB 的 release 版本，用 kv 命令看看 WinDBG 产生的栈回溯信息：

![]( {{site.url}}/asset/software-debugging-fpo.png )

可以看到 EPB 寄存器指向的仍然是 main 函数的栈帧，而非 FuncB 的栈帧。那么 WinDBG 是如何生成 FuncB 函数的栈信息的呢？答案是依靠调试符号中的附加信息。

我们先来看一下 `FPO:[1,2,1]` 是什么含义。FPO 代表了 FuncB 函数采用了 FPO 优化，方括号中的三个数字的含义分别是：
1. 第一个 1 表示 FuncB 有一个参数；
2. 第二个 2 表示 为局部变量分配的栈空间是 2 个 DWORD 长，即 8 个字节；
3. 第三个 1 表示 使用栈保存的寄存器个数为 1，即有一个寄存器被压入栈中保存 (EDI)；

我们来看一下 WinDBG 是如何利用符号文件来产生 FuncB 的栈帧记录的。
- 根据 EIP 寄存器的值，可以找到最靠近的函数符号，这样就找到了当前执行的指令存在于 FuncB 函数中；
- FuncB 的符号信息中有一个字段是这个函数的 RVA, 也就是这个函数的入口相对于模块起始地址的偏移，假设这里是 0x1040；
- 符号文件中的 FPO 信息是按照 RVA 来组织的，所以可以通过 FuncB 的 RVA 来找到它的 FPO 信息；
- WinDBG 从 FPO 信息中了解到，函数的参数长度是 0x4, 局部变量的长度是 0x8, 代码的长度是 0x62；
- 根据上述信息，结合当前 EIP 和 ESP 的值，以及对应的反汇编代码，调试器可以推算出当前函数的帧指针值；比如 CPU 执行到了 FuncB 的偏移 7 字节处，根据反汇编分析 WinDBG 可以知道已经执行了局部变量分配操作，因此 ESP 的值加上局部变量的长度便是当前栈帧的边界，即 ESP + 8 = 0012ff78；根据惯例帧指针指向的是当前栈帧的第一个 DWORD，因此应该把这个值减去 4，即 0x0012ff74 是当前函数的帧指针 (ChildEBP)。得到帧指针后，便可以像普通函数那样显示参数和返回值了；


## 栈指针检查

栈的有序性是依靠每个函数都遵守栈的使用规则来完成的，每个函数在返回时，其栈指针 ESP 都应当与进入函数时一致，即保持栈平衡。否则的话，栈数据可能会错位甚至面目全非。

下面通过一个例子来说明，以下代码通过汇编指令向栈中压入了 4 个字节的内容，但是却没有对应的弹出操作：

```c
void BadEsp()
{
    _asm push eax;
}
```

当执行到这个函数时，栈顶存放的是返回地址，执行 PUSH EAX 指令后，栈顶存放的内容变为 EAX 寄存器的值。当执行 RET 指令时，因为 RET 指令总是返回到栈顶所存放的地址处，所以执行 RET 后程序会跳转到 EAX 寄存器所包含的地址。以下是笔者跟踪执行 RET 指令后的结果：

![]( {{site.url}}/asset/software-debugging-check-esp-error.png )

可见程序指针指向了 00000000d 的位置，这里根本不是有效的代码区， `??` 代表 WinDBG 无法显示这个地址的内容。

为了及时发现以上问题，编译器设计了专门的栈指针检查函数来帮助发现问题。在 VC 6 中编译 Debug 版本时，VC6 会自动在每个函数的末尾插入指令来调用一个名为 `_chkesp` 的函数：

![]( {{site.url}}/asset/software-debugging-check-esp-vc6-chkesp.png )

可以看到，编译器为只包含一条汇编指令的 BadEsp 函数生成了 20 多条指令。这是因为在调试版本中，编译器插入了很多指令来帮助检查错误和支持调试。比如，尽管我们没有定义局部变量，但编译器仍然分配了一段空间，并将其填充为 0xCC (第 5 行，第 9 ~ 12 行)，万一 CPU 意外执行到了该区域，可以中断到调试器。

因为我们使用 PUSH EAX 破坏了栈平衡，第 16 ~ 18 行的弹出操作都错位了，EDI 会被恢复为 EAX 的值，ESI 会被恢复为 EDI 的值。。。而且第 19 行执行后的 ESP 的值也会比预期的小 4, 这会导致 `_chkesp` 函数弹出如下图所示的错误对话框 (VS2008)：

![]( {{site.url}}/asset/software-debugging-check-esp-error-dlg.png )

`_chkesp` 是 C 运行时库 (CRT) 中的一个函数，用来检查栈指针 esp 的完好性。检查方法是比较 EBP 和 ESP 的值，如果相等则通过，否则就调用 `_CrtDbgReport()` 函数报告错误；

很明显，`_chkesp` 不适合经过 FPO 优化的函数。编译器的 /GZ 用来控制是否插入栈指针检查函数。

VC8 的 `_RTC_CheckEsp` 函数与 `_chkesp` 的原理和工作方法是一样的，只是改变了函数名称和报告错误的方式。


## 调用协定

要复用现有函数的一个基本问题就是如何调用这些函数，包括如何传递参数，如何接收计算结果，如何维持程序的上下文状态和清理栈，等等。考虑到要调用的函数可能是不同人使用不同语言和编译器开发的，所以这个看似简单的问题并不简单。通过上一节对栈的介绍，我们知道参数传递或者栈操作中的任何计算误差都有可能导致严重的错误。

要解决以上问题，在发起调用的一方和被调用函数之间必须建立一种契约，这便是函数调用协定 (Calling Convention)。通常在设计一个函数时，将调用协定写在函数声明中：

```c
return-type [调用协定] function-name([argument-list]);

// 例如
BOOL WINAPI IsDebuggerPresent(VOID);
```

其中的 WINAPI 就是调用协定，如果没有指定的话，那么编译器会根据语言环境使用默认的调用协定。

在 x86 系统中应用比较广泛的调用协定有以下几种：
- C 调用协定；
- 标准调用协定；
- 快速调用协定；
- C++ 成员函数所使用的 this 调用协定；

### C 调用协定

C 调用协定是 C/C++ 的普通函数所使用的默认调用协定，完全使用栈来传递参数，函数调用者 (caller) 在发起调用前将参数按照从右到左的顺序依次压入栈中，并且负责函数返回后调整栈指针清除压入栈的参数。

从右到左压入参数的好处是对于被调用函数来说第一个参数相对于栈顶的偏移总是固定的 (栈顶的第一个就是)，而且是调用者负责清理参数，所以 C 调用协定支持可变数目的参数 (调用者负责清理参数，它知道传入参数的数量，所以这个数量是可变的)。

C 调用协定所使用的关键字是 `__cdecl`。

### 标准调用协定

标准调用协定和 C 调用协定很类似，也是用栈来传递参数，传递顺序也是从右到左。不同之处在于标准调用协定规定由被调用函数清理栈中的参数。

对于被频繁调用的函数而言，标准调用协定的好处在于可以减小目标程序的大小，因为清理栈的指令不必反复出现在调用函数中 (只存在于被调用函数中)。

标准调用协定所使用的关键字是 `__stdcall`。

大多数 Windows API 使用的都是标准调用协定。

### 快速调用协定

因为 CPU 访问寄存器的速度要比访问内存快很多，使用寄存器传递参数比使用栈速度更快，因此很多编译器都设计了使用寄存器传递参数的调用协定。在 x86 平台上，比较典型的便是所谓的快速调用协定 `__fastcall`。

快速调用协定使用 ECX 和 EDX 来传递前两个长度不超过 32 位的参数，其他参数使用栈来传递 (从右到左)。

在调试时， kv 这样的栈回溯命令只显示放在栈上的参数，不会显示使用寄存器传递的参数，因此使用快速调用协定是不利于调试的。

### This 调用协定

C++ 中的成员函数默认使用的调用协定是 this 调用协定。这种调用协定的最重要特征是 this 指针会被放入 ECX 寄存器 (Borland 编译器使用 EAX) 传递给被调用的方法。

This 调用协定也是要求被调用函数自己来清理栈，因此不支持可变数量的参数。当我们在 C++ 中定义了可变数量参数的成员函数时，编译器会自动改为使用 C 调用协定。当调用这样的方法时，编译器会将所有参数压入栈之后，再将 this 指针压入栈。例如：

```cpp
enum MEAL
{
    BREAKFAST,
    LUNCH,
    SUPPER
};

class Cat
{
public:
    char* ChooseFood(MEAL i, ...);
};

// 调用 Cat 的可变数量参数函数
Cat cat;
printf("Cat choose food %s.", cat.ChooseFood(BREAKFAST, "meat", "beaf", "rice"));
```

以下是对应的汇编指令：

![]( {{site.url}}/asset/software-debugging-this-call-example.png )

当执行到 ChooseFood 方法时，使用 kv 命令显示栈回溯信息，其结果如下：

![]( {{site.url}}/asset/software-debugging-this-call-example-call-stack.png )

### 通过编译器开关改变默认调用协定

对于函数声明中没有指定调用协定的函数，编译器会根据情况使用合适的默认调用协定。可以通过编译开关来定义默认的调用协定。以 VC 编译器为例：
- /Gd, 默认设置，使用 C 调用协定；
- /Gr, 使用快速调用协定；
- /Gz, 使用标准调用协定

### 函数返回值

传递返回值的方法不像传递参数那样有很多种。通常编译器会根据以下原则来选择传递返回值的方法：
- 如果返回值是 EAX 能够容纳的整数、字符或指针，那么使用 EAX 寄存器；
- 如果返回值超过 4 字节但小于 8 字节，那么使用 EDX 来存放 4 字节以上的值；
- 如果返回一个结构或类，那么分配一个临时变量作为隐含的参数传递给被调用函数，被调用函数将返回值复制到这个隐含参数中，并且将其地址赋值为 EAX 寄存器；
- 浮点类型的返回值和参数通过专门的浮点指令使用栈来传递；

下面通过例子来看一下返回结构或类的情况：

```cpp
#define NAME_LENGTH 20
class Cat
{
    char m_szName[NAME_LENGTH];
public:
    Cat() {
        m_szName[0] = 0;
    }
    
    Cat(char* sz);
    
    char* ChooseFood(MEAL i, ...);
    
    Cat GetChild(int n)
    {
        Cat c("Data");
        return c;
    }
    
    char* Name() 
    {
        return m_szName;
    }
}

// 调用 GetChild 方法：
Cat cat("Looi"), cat2("");
cat2 = cat.GetChild(2);
```

上面的第二条语句对应的汇编指令如下：

![]( {{site.url}}/asset/software-debugging-ret-stack-example.png )

也就是说，编译器会定义一个额外的临时对象，并将它传递给函数，当函数返回的时候，EAX 寄存器指向的是这个临时对象。

从 WinDBG 显示的栈回溯信息中我们可以看到隐含参数：

![]( {{site.url}}/asset/software-debugging-ret-implied-param.png )

再看看 return 对应的汇编指令：

![]( {{site.url}}/asset/software-debugging-ret-implied-param-return.png )

在上述调用过程中，共发生了两次对象复制操作，一次是 GetChild 函数中将局部变量 c 赋值给隐含参数，另一次是将隐含参数赋值给调用方的局部变量 cat2.

这种返回方式需要定义额外的局部变量和承担对象复制所需的开销，所以其效率是比较低的，一种改进的方式是将 cat2 对象以指针或引用的形式直接传递给被调用函数。

### 归纳和补充

- 对于 32 位程序，所有长度小于 32 位的参数会被自动加宽到 32 位，换句话说，每个参数所占的空间至少是 32 位，不论在栈中还是寄存器中；
- 在函数调用过程中，被调用函数必须保证某些寄存器的内容在函数返回时和进入函数时是一样的，这些寄存器被称为 nonvolatile 寄存器，如果一个函数要使用 nonvolatile 寄存器，那么它应该先把寄存器的本来内容保存起来，待函数返回时，再将这些寄存器的值恢复成原来的值。
- 在 kv, kb 等栈回溯命令的结果中，返回地址右侧的三列被称为 Args to child, 通常解释为函数的前三个参数。这种解释其实是不准确的：

![]( {{site.url}}/asset/software-debugging-args-to-child-example.png )

3fc00000 确实是第一个参数，但是 696f6f4c 和 00000000 却不是第二个和第三个参数，它们是第三个参数 cat 对象的前 8 个字节。而真正的第二个和第四个参数分别是在 ECX 和 EDX 寄存器中的。使用 r 命令可以看到它们：

![]( {{site.url}}/asset/software-debugging-args-to-child-example-register.png )

因此准确的说法是，kv, kb 命令显示的是栈帧中参数区域的前三个 DWORD，这三个 DWORD 的确切含义要视函数所采用的调用协定而定。因为大多数 Windows 系统函数使用的都是标准调用，所以很多时候这三个 DWORD 确实对应于前三个参数。


## 栈空间的增长和溢出

栈为函数调用和定义局部变量提供了一块简单易用的内存空间，定义在栈上的变量不需要考虑内存申请和释放这样的麻烦事，因为编译器会生成代码来完成这些任务。

从栈上分配空间意味着栈指针下移 (向更低地址移动)，腾出更多空间；释放空间意味着栈指针移回到原来的位置，也就是说，只要调整栈指针就可以分配和释放栈中的空间，这比从堆上分配空间要高效得多。

![]( {{site.url}}/asset/software-debugging-stack-allocation-release.png )

### 栈空间的自动增长

VC 编译器在构建一个程序时，为栈设置的默认参数是保留 1MB, 初始提交 4KB，这意味着系统会为这个进程的初始线程创建一个 1MB 大小的栈，并先提交其中的一小部分 (8KB, 其中 4KB 为保护页) 供程序使用。

只有已提交的内存才是可以访问的，访问保留的内存会导致访问违规，这时候会触发栈增长机制来扩大提交区域。

之前介绍栈创建过程时，我们提到过系统在提交栈空间时会故意多提交一个页面，这个页面称为栈保护页面 (Stack Guard Page)。栈保护页面具有特殊的 `PAGE_GUARD` 属性，当具有如此属性的内存页被访问时，CPU 会产生页错误异常并开始执行系统的内存管理函数。

在内存管理函数检测到 `PAGE_GUARD` 属性后，会先清除对应页面的 `PAGE_GUARD` 属性，然后调用一个名为 MiCheckForUserStackOverflow 的系统函数函数，这个函数会从当前线程的 TEB 中读取用户态栈的基本信息并检查导致异常的地址，如果导致异常的被访问地址不属于栈空间范围，则返回 `STATUS_GUARD_PAGE_VIOLATION`；否则，该函数会计算栈中是否还有足够的剩余保留空间可以创建一个新的栈保护页面。如果有，则调用 ZwAllocateVitualMemory 从保留空间再提交一个具有 `PAGE_GUARD` 属性的内存页，新的栈保护页与原来的紧邻。

经过这样的操作后，栈的保护页向低地址方向平移了一位，栈的可用大小增加了一个页面的大小，这便是所谓的栈空间自动增长。

### 栈溢出

经过如上的自动增长过程，当栈保留空间只剩两个页面空间的时候，`MiCheckForUserStackOverflow` 会提交倒数第二个页面，但不再设置它的 `PAGE_GUARD` 属性，同时最后一个页面不会被提交，因为最后一个页面永远保留不可访问，所以这时候栈增长到达了极限。为了让应用程序知道栈即将用完，`MiCheckForUserStackOverflow` 函数会返回 `STATUS_STACK_OVERFLOW`，触发栈溢出异常：

![]( {{site.url}}/asset/software-debugging-stack-overflow.png )

栈溢出异常是一个警告性的异常，在其发生后，栈的倒数第二个页面通常还没有真正被使用，所以应用程序还可以继续运行，但当这个页面用完，访问到最后一个页面时，边会引发访问违规异常。

### 分配检查

之前介绍的栈空间自动增长机制中，有一种情况没有考虑到，如果某一次栈分配需要的空间特别大，超过了一个页面，这时这个栈帧的某些部分就有可能一下子被分配到了保护页之外的未提交空间，这就会导致访问违规。

为了防止这种情况发生，对于要分配较大（超过一个页面）栈空间的情况，编译器会自动调用一个分配检查函数来把超过一个页面的分配分割成很多次来逐个提交。

具体说来，对于超过一个页面的栈分配，分配检查函数会按照每次不超过一个页面的大小逐渐调整栈指针，而且每调整一次，它会访问一次新分配空间中的内容，称为 Probe, 目的是触发栈保护页平移，让栈增长。

VC 编译器使用的分配检查函数名叫 `_chkstk`。


## 栈下溢

通常把访问栈边界以上的空间称为 栈溢出(Stack overflow)。类似的，把访问栈底以下的空间称为栈下溢 (Stack Underflow)。因为栈是向低地址空间生长的，这里的栈底以下是指比栈底的地址还大的内存区域。

例如以下代码：

```c
#pragma optimize("", off)
void main()
{
    char sz[20];
    sz[2000] = 0;
}
```

执行以上代码会导致访问违规 (Access Violation) 异常，因为 `sz[2000]` 所引用的地址是比栈底地址还大的内存区域。


## 缓冲区溢出

缓冲区是程序用来存储数据的连续内存区域，一旦分配完成，其起止地址（边界）和大小便固定下来。当使用缓冲区时，如果使用了超出缓冲区边界的区域，那么便成为缓冲区溢出 (Buffer Overflow)，或缓冲区越界 (Buffer Overrun)。

如果缓冲区是分配在栈上的，那么又称为栈缓冲区溢出，如果是分配在堆上的，又称为堆缓冲区溢出，本节我们将讨论发生在栈上的缓冲区溢出。

### 感受缓冲区溢出

下面的代码需要用 VC6 编译，因为 VC8 加入了缓冲区检查功能，无法观察到该现象。

```cpp
int main(int argc, char* argv[])
{
    char szInput[5];
    printf("Enter string:\n");

    gets(szInput);
    printf("You Entered:\n%s\n", szInput);

    return 0;
}
```

上述代码声明了一个 szInput 局部变量用来存储用户输入的内容，我们用 VC6 调试这个程序，然后输入 `987654321012345`，程序会打印出我们输入的字符串，但紧接着会出现应用程序错误对话框：

![]( {{site.url}}/asset/software-debugging-buffer-overflow-example.png )

下图展示了这一过程中函数栈的变化，可以在 VS 中通过 ESP 寄存器的值找到栈顶，然后用 Memory 窗口观察此时栈中的数据：

![]( {{site.url}}/asset/software-debugging-buffer-overflow-memory.png )

可以看到，局部变量 szInput 的大小为 5，但编译器分配了 8 个字节来存储它，因为 32 位系统下栈分配的最小单位是 4 字节。

我们输入的 `987654321012345` 需要占用 16 个字节，因此有一部分数据超出了 szInput 的栈空间。可以看到它覆盖了存储在栈中的 main 函数的栈帧指针 EBP，以及 main 函数的返回地址。这就是缓冲区溢出。

因为它覆盖的是 main 函数的返回地址，所以 gets 函数不受影响，还可以顺利执行并返回到 main 函数中，接下来 main 函数还可以继续执行一段时间，所以我们可以看到 printf 打印的字符串。**这是缓冲区溢出的一个特征，即错误不会立即体现出来，程序可以继续执行一段时间才暴露出异常**。

当执行到 ret 指令时，栈顶的值为 0x00353433, 这个值将被设定到 EIP 寄存器中，由 CPU 执行到该地址指向的指令。该地址不属于任何模块，对应的内存区不具有合适的类型和保护属性，所以当 CPU 读取并执行这里的指令时发生了访问违规异常。

如果我们把输入替换为 `987654321012i4@`, 那么 ret 时 栈顶的值就是 0x00403469, 这个地址处有编译器补位用的断点指令。此时程序会因为断点指令而中断下来。

推而广之，只要我们巧妙地设计要输入给缓冲区的数据，就可以控制该缓冲区所在的函数的返回位置。如果我们加长输入的内容，使其包含一段代码，然后再将返回地址指向这段代码，那么就可以让这个程序执行我们动态注入的代码了。这就是利用缓冲区溢出进行恶意攻击的基本原理。

### 缓冲区溢出攻击

```c
void HandleInput(char* input)
{
    char buffer[31];
    strcpy(buffer, input);
    OutputDebugStringA(buffer);
}

int main(int argc, char* argv[])  
{
    char* input = "\x6a\x01\x33\xc0"
        "\x50\x50\x50\xba\x0b\x05\xd8\x77"
        "\xff\xd2\x48\x85\xc0\x74\xed"
        "\x32\x32\x32\x32\x32\x31\x31\x31\x31"
        "\x31\x31\x31\x31\x31\x31\x31\x31"
        "\x30\x4a\x42";

    HandleInput(input);

    return 0;
}
```

上述代码中 HandleInput 函数的实现非常常见，它定义了一个局部变量 buffer 来存储外部传入的数据。

值得注意的是我们输入的内容，即 input 变量，前三行是 “恶意” 代码的机器码，最后一行是用来覆盖函数返回地址的新地址，即恶意代码的内存地址。中间部分是用来调整位置的。前三行对应的机器码如下：

![]( {{site.url}}/asset/software-debugging-buffer-overflow-attack.png )

可以看到，上述恶意代码调用 MessageBox API 弹出了一个对话框。


## 变量检查

我们之前介绍的两个缓冲区溢出的例子，都是在 VC6 下实现的。随后的 VC 编译器逐步引入了一系列功能来帮助应用程序检查缓冲区溢出，包括用于精确检查每个局部变量是否溢出的变量检查功能，以及用于保护函数返回地址的安全 Cookie 功能。

把之前的例子放到 VC8 里面运行，结果会出现如下错误框：

![]( {{site.url}}/asset/software-debugging-variable-check-error-dlg.png )

可以看到编译器插入的代码检测到了缓冲区溢出，而且还指出了发生溢出的变量名是 buffer，那么编译器插入了什么样的代码，是如何检查的呢？以下是 VC8 编译出的 HandleInput 函数的反汇编代码：

![]( {{site.url}}/asset/software-debugging-variable-check-handle-input-disasm.png )

观察上述反汇编代码，可以看出 VC8 与 VC6 编译产生的代码有以下几点明显不同：
- VC8 为每个函数的 Debug 版本所分配的默认局部变量比 VC6 更长，VC6 通常是 64 字节，而 VC8 分配的是 192 字节。即使一个函数内部没有任何语句，编译器也会在栈帧中为这个默认局部变量分配空间；
- VC8 会在紧邻栈帧指针的位置分配 4 个字节用于存放安全 Cookie (13 ~15 行)。安全 Cookie 就像一个护栏，用于保护紧邻的栈帧指针和函数返回地址。安全 Cookie 一旦损坏，就说明函数内已经发生了缓冲区溢出，栈帧数据很可能已经受损。在函数的出口 VC8 会插入代码 (43 ~ 45 行) 检查 Cookie 的完好性；
- 在给每个局部变量分配空间时，VC8 会给每个变量多分配 8 个字节，前面放 4 个，后面放 4 个，这 8 个字节总被初始化为 0xCC，这样每个变量都有了前后两个屏障，一旦屏障受损，说明该变量附近发生了溢出；
- VC8 默认使用 `_RTC_CheckEsp` 函数来检查栈指针的平衡性，其原理与 VC6 使用的 `_chkesp` 完全相同。只是对于被调用函数清理栈的调用协定 (`__stdcall`)，VC8 会在调用这个函数前先记录下栈指针的值，调用函数之后进行检查(25, 29, 30 行)；
- 对于有可能发生缓冲区溢出的函数，在函数返回前，VC8 会调用 `_RTC_CheckStackVars` 做变量检查，也就是检查每个变量前后的屏障字段是否完好。如果发现屏障字段不再是 0xCCCCCCCC，则调用错误报告函数报告错误；

在以上措施中，我们把插入和检查安全 Cookie 称为安全检查 (Security Check)。检查栈指针和针对每个变量的检查是运行期检查 (runtime check) 的一部分，之前我们已经介绍过运行期检查。

下面介绍变量检查的工作原理，下一节介绍 Cookie 检查。

变量检查过程分为三个步骤：
- 在分配局部变量时，为每个局部变量多分配 8 个字节的额外空间，并用 0xCC 填充，这些 0xCC 字节称为栅栏字节；
- 为了在运行期间能够知道每个变量的长度、位置和名称，编译器会产生一个变量描述表，用来记录局部变量的详细信息；
- 在函数返回前，调用 `_RTC_CheckStackVars` 函数，根据变量描述表逐一检查其中的每个变量，如果发现变量前后的栅栏字节发生变化则报告检查失败；

以下是 `_RTC_CheckStackVars` 的伪代码：

![]( {{site.url}}/asset/software-debugging-variable-check-rtc-check-stack-vars.png )

其中的 `_RTC_framedesc` 就是变量描述表。

如果 `_RTC_CheckStackVars` 函数发现变量前后的栅栏被破坏，那么便会调用 `_RTC_StackFailure` 报告错误。

VC8 编译器引入了 `/RTCs` (s 代表 stack) 选项来启用包括变量检查功能在内的栈检查机制。不过在 VC8 生成的项目文件中，为调试版本设置的默认选项使用的通常是 `/RTC1`，它是 `RTCsu` 的等价形式，即同时启用 RTCs 和 RTCu。

### RTCu 检查变量是否初始化

例如下面的代码：

```c
int n, j;
j = n;
```

如果启用了 RTCu 选项，编译器会为变量 n 定义一个标志变量，位于栈帧中 Debug 版默认局部变量上方。标志变量为 bool 类型，但其前后也会加上变量屏障，因此它会占用 12 字节的栈空间。标志变量用来记录变量是否被初始化，在函数入口处，这个值会被设为 0；在对该变量赋值的地方，编译器会加入代码将该标记设为 1；在访问该变量的地方会加入如下代码来发出警告：

![]( {{site.url}}/asset/software-debugging-variable-check-rtcu.png )

其中第 1 行判断初始化标志是否为 0，避免重复发出警告。第 4 行调用 `__RTC_UnInitUse` 来发出警告。


## 基于 Cookie 的安全警告

VC8 在编译可能发生缓冲区溢出的函数时，会定义一个特别的局部变量，该局部变量会被分配在栈帧中所有其他局部变量和站帧指针与函数返回地址之间。这个变量专门用来保存 Cookie, 因此我们将它称为 Cookie 变量。

Cookie 变量是一个 32 位整数，它的值是从全局变量 `__security_cookie` 得到的。该全局变量定义在 C 运行时库中，在 CRT 源代码目录的 gs_cookie.c 文件中可以看到其定义：

```c
#ifdef _WIN64
#define DEFAULT_SECURITY_COOKIE 0x00002B992DDFA232
#else  /* _WIN64 */
#define DEFAULT_SECURITY_COOKIE 0xBB40E64E
#endif  /* _WIN64 */

DECLSPEC_SELECTANY UINT_PTR __security_cookie = DEFAULT_SECURITY_COOKIE;

DECLSPEC_SELECTANY UINT_PTR __security_cookie_complement = ~(DEFAULT_SECURITY_COOKIE);
```

可以看到 `__security_cookie` 在 32 位下的默认值是 0xBB40E64E。在程序的启动函数中还会调用 `__security_init_cookie` 来对该变量进行初始化：

```c
int _tmainCRTStartup(void)
{
    __security_init_cookie();

    return __tmainCRTStartup();
}
```

`__security_init_cookie` 函数定义在 `gs_support.c` 文件中，其中 Cookie 初始化的代码如下所示：

```c
GetSystemTimeAsFileTime(&systime.ft_struct);
cookie = systime.ft_struct.dwLowDateTime;
cookie ^= systime.ft_struct.dwHighDateTime;
cookie ^= GetCurrentProcessId();
cookie ^= GetCurrentThreadId();
cookie ^= GetTickCount();

QueryPerformanceCounter(&perfctr);
cookie ^= perfctr.LowPart;
cookie ^= perfctr.HighPart;

__security_cookie = cookie;
__security_cookie_complement = ~cookie;
```

可以看到 Cookie 是通过当前系统时间，再与进程 ID, 线程 ID, 系统 TickCount 和性能计数器进行逐位异或 计算的来的。这是为了让 Cookie 有较好的随机性。

编译器在为一个函数插入 Cookie 时，它还会与当时的 EBP 寄存器的值做一次异或操作，然后再保存到栈上的 Cookie 变量中。

与 EBP 异或有两个好处：
- 进一步提高 Cookie 的随机性，因为 `__security_cookie` 是程序启动时初始化的，进程生命周期内只会初始化一次。通过异或操作可以让每次进入函数的 Cookie 都不相同；
- 可以起到检查 EBP 不被破坏的作用，在检查 Cookie 变量时，先将 `__security_cookie` 再与 EBP 异或一次，然后比较两次的 Cookie 值是否相同就可以判断 EBP 是否被破坏；

为了降低对可执行文件大小和运行性能的影响，`__security_check_cookie` 函数是直接用汇编写的，整个函数只有 4 条汇编指令，没有使用任何变量和改变寄存器。

在 VC8 中，缓冲区安全检查是默认启用的，但也可以在编译选项中加入 /GS 来显式启用安全检查，或者加入 /GS- 来禁用安全检查。


## 本章总结

下表归纳了编译器在产生栈和函数调用有关的代码时所加入的支持软件调试的机制，以及这些机制在调试版本和发布版本中的可用性：

![]( {{site.url}}/asset/software-debugging-compiler-stack-support.png )