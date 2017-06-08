---
layout: post
title:  "《软件调试》 学习 19 堆和堆检查"
date:   2017-06-05 10:02:30 +0800
categories: debugging
---

* TOC
{:toc}

内存是软件工作的舞台，当用户启动一个程序时，系统会将程序文件从外部存储器 (硬盘等) 加载到内存中。当程序工作时，需要使用内存空间来放置代码和数据。

在使用一段内存之前，程序需要以某种方式 (库函数或 API 等) 发出申请，接收到申请的一方 (C 运行库或各种内存管理器) 根据申请者的要求从可用 (空闲) 空间中寻找满足要求的内存区域分配给申请者。当程序不再需要该空间时应该通过与申请方式相对应的方式来归还空间，即释放。

堆 (Heap) 是组织内存的一种重要方法，是程序在运行期间动态申请内存空间的主要途径。与栈空间是由编译器产生的代码自动分配和释放不同，堆上的空间需要程序员自己编写代码来申请和释放。而且分配和释放操作应该严格匹配，忘记释放或者多次释放都是不正确的。

与栈上的缓冲区溢出类似，如果向堆上的缓冲区写入超过其大小的内容，也会因为溢出而破坏堆上的其他内容，可能导致严重的问题，包括程序崩溃。

为了帮助发现堆方面的使用问题，堆管理器、编译器和软件类库 (如 MFC, .NET FrameWork 等) 提供了检查和辅助调试机制。比如 Win32 堆支持参数检查、溢出检查及释放检查等功能。VC 编译器设计了专门的调试堆并提供了一系列用来追踪和检查堆使用情况的函数，在编译 Debug 版本的可执行文件时，可以使用这些调试支持来解决内存泄漏等问题。

本章将先介绍堆的基本概念，然后分两大部分分别介绍 Win32 堆和 CRT 堆。


## 理解堆

从操作系统的角度看，堆是系统的内存管理功能向应用软件提供服务的一种方式。通过堆，内存管理器将一块较大的内存空间委托给堆管理器 (Heap Manager) 来管理。堆管理器将大块内存分割成不同大小的很多个小块来满足应用程序的需要。应用程序的内存需求通常是频繁而零碎的，如果把这些请求都直接传递给位于内核中的内存管理器，会影响系统的性能。有了堆管理器，内存管理器就只需要处理大规模的分配请求。这样做不仅可以减轻内存管理器的负担，也可以大大缩短应用程序申请内存分配的时间，提供程序的运行速度。

下图画出了 Windows 系统中实现的这种多级内存分配体系：

![]( {{site.url}}/asset/software-debugging-heap-alloc-architecture.png )

我们从下往上来看这个图。

首先，内存管理器暴露了一系列的 虚拟内存 API 来管理内存，包括 VirtualAlloc, VirtualAllocEx, VirtualFree, VirtualFreeEx, VirtualLock, VirtualUnlock, VirtualProtect, VirtualQuery 等。它们的内核版本是 NtAllocateVirtualMemory, NtProtectVirtualMemory 等。这些虚拟内存 API 可以被上层的每一层调用；

接着是内核池，这个内核池是专门给驱动程序使用的，它也被称为 池管理器 (Pool Manager), 它公开了一组驱动程序接口 (DDI) 以向外提供服务，包括 ExAllocatePool, ExAllocatePoolWithTag, ExAllocatePoolWithTagPriority, ExAllocatePoolWithQuota, ExFreePool, ExFreePoolWithTag 等。池管理器通过虚拟内存 API 来向内存管理器请求内存空间；

接着是 Win32 堆，它的作用与池管理器一样，只不过它是为用户态的应用程序服务的。Win32 堆实现在 NTDLL.DLL 中实现，SDK 中公开了一组 API 来访问 Win32 堆管理器的功能，比如 HeapAlloc, HeapFree 等。Win 32 堆管理器通过虚拟内存 API 来向内存管理器请求内存空间；

接着是 CRT 堆。为了支持 C 的内存分配函数和 C++ 的内存分配运算符 (new 和 delete)，编译器的 C 运行时库会创建一个专门的堆供这些函数使用，通常称为 CRT 堆。根据分配堆块的方式不同，CRT 堆有三种工作模式：SBH (Small Block Heap) 模式、旧 SBH(Old SBH) 模式、系统模式 (System Heap)，当创建 CRT 堆时，会选择其中一种。对于前面两种模式，CRT 堆会使用虚拟内存 API 从内存管理器中批发大的内存块过来，然后分割成小的堆块满足应用程序的需要。对于系统模式，CRT 堆只是把堆块分配请求转发给它所基于的 Win32 堆，因此处于系统模式的 CRT 堆只是对 Win32 堆的一种简单封装，在原来的基础上附加了一些功能；

在往上，应用程序自己可以实现堆管理器，其方式是通过虚拟内存 API 从内存管理器 “批发” 内存块过来然后提供给自己的客户代码使用。此外应用程序可以通过 C/C++ 语法来向 CRT 堆申请内存，这也是应用程序使用堆的一般方法。


## 堆的创建和销毁

### 进程的默认堆

Windows 系统在创建一个新的进程时，在加载器函数执行进程的用户态初始化阶段，会调用 RtlCreateHeap 函数为新的进程创建第一个堆，称为进程的默认堆，有时简称进程堆 (process heap)。

创建好的堆句柄会被保存到进程环境块 (PEB) 的 ProcessHeap 字段中，可以使用 WinDBG 的 dt 命令来观察 PEB 结构，以下是与进程堆有关的几个字段：

![]( {{site.url}}/asset/software-debugging-windbg-process-heap.png )

可以看到进程堆的保留大小是 1MB (默认值)，初始提交大小是两个内存页 8KB。可以通过链接选项 /HEAP 来改变进程堆的保留大小和初始提交大小；

使用 `GetProcessHeap()` API 可以取得当前进程的进程堆句柄。得到堆句柄之后，就可以使用 HeapAlloc API 从这个堆上申请空间：

```c
CStructXXX* pStruct = HeapAlloc(GetProcessHeap(), 0, sizeof(CStructXXX));
```

### 创建私有堆

除了系统为每个进程创建的默认堆，应用程序也可以通过调用 `HeapCreate()` API 来创建其他堆，这样的堆只能被发起调用的进程自己访问，通常被称为私有堆 (Private Heap)。

HeapCreate 内部主要是调用 RtlCreateHeap 函数，因此私有堆与默认堆并没有本质差异，只是创建的用途不同。RtlCreateHeap 内部会调用 ZwAllocateVirtualMemory 系统服务从内存管理器中申请内存空间，初始化用于维护堆的数据结构，最后将堆句柄记录到进程的 PEB 结构中，确切说是我们将要介绍的 PEB 结构的堆列表中。

### 堆列表

每个进程的 PEB 结构以列表的形式记录了当前进程的所有堆句柄，包括进程的默认堆。

具体来说，PEB 中有三个字段用于记录每个堆的句柄。NumberOfHeaps 字段用来记录堆的总数，ProcessHeaps 字段用于记录每个堆的句柄，它是一个数组，它可以容纳的句柄数记录在 MaximumNumberOfHeaps 字段中。如果 NumberOfHeaps 达到 MaximumNumberOfHeaps, 那么堆管理器会增大 MaximumNumberOfHeaps 值，并重新分配 ProcessHeaps 数组。

使用 dd 命令观察 ProcessHeaps 字段的值便可以看到当前进程所有堆指针：

![]( {{site.url}}/asset/software-debugging-processheaps.png )

使用 !heap 命令可以列出当前进程的所有堆：

![]( {{site.url}}/asset/software-debugging-windbg-process-heaps.png )

可以看到堆句柄的值实际上就是这个堆的起始地址。

### 销毁堆

应用程序可以调用 HeapDestroy API 来销毁进程的私有堆。它会调用 NTDLL 中的 RtlDestoryHeap 函数，该函数会从 PEB 的堆列表中将要销毁的堆句柄移除，然后调用 NtFreeVirtualMemory 向内存管理器归还内存。

程序不应该销毁进程的默认堆，因为进程内的很多系统函数会使用这个堆。

当进程退出和销毁进程对象时，系统会两次调用内存管理器的 MmCleanProcessAddressSpace 函数来释放进程的内存空间。
- 第一次是在要退出的进程中执行的，如果退出的是最后一个线程，那么 PspExitThread 会调用 MmCleanProcessAddressSpace, 后者会先删除进程用户空间中的文件映射和虚拟地址，释放虚拟地址描述符，然后删除空间的系统部分，最后删除进程的页表和页目录设施。这是在退出进程上下文时执行的最后几项任务之一；
- 当系统的工作线程删除进程对象时会再次调用 MmCleanProcessAddressSpace 函数，再一次清理进程的内存空间；

因此程序没必要在退出时一一销毁每个私有堆，系统在清理进程空间时会自动释放。


## 分配和释放堆块

之前我们已经介绍过，Win32 堆管理器提供了 HeapCreate, HeapDestroy 接口来从内核的内存管理器那里申请和释放大块内存。这里的 Win32 堆管理器相当于是个大批发商，上层的 CRT 堆在申请到大块内存空间后，还需要继续调用 HeapAlloc 等方法，从堆中申请小尺寸的内存。

当应用程序调用堆管理器的分配函数向堆管理器申请内存时，堆管理器会从自己维护的内存区中分割出一个满足用户指定大小的内存块，然后把这个内存块中允许用户访问部分的起始地址返回给应用程序，堆管理器把这样的块叫做一个 chunk, 本文译为堆块。应用程序用完一个堆块后，应该调用堆管理器的释放函数归还堆块。本节将介绍分配和释放堆块的典型方法，并讨论这些方法的区别和联系。

### HeapAlloc

在 Windows 系统中，从堆中分配空间的最直接方法就是调用 HeapAlloc API, 其原型如下：

```c
LPVOID HeapAlloc(HANDLE hHeap, DWORD dwFlags, SIZE_T dwBytes);
```

其中 hHeap 是堆句柄，dwBytes 是要分配的字节数，dwFlags 可以为如下标志位的组合：
- `HEAP_GENERATE_EXCEPTIONS` ：使用异常来报告失败情况，如果不设置这个标记，则用返回 NULL 的方式来报告错误；
- `HEAP_NO_SERIALIZE` ：不需要对该次分配实施串行化控制 (加锁)。如果希望对该堆的所有分配调用都不需要串行化控制，那么可以在创建堆时指定这个选项。对于进程默认堆，调用 HeapAlloc 时不应该指定该标志，因为系统代码可能随时会调用堆函数访问默认堆；
- `HEAP_ZERO_MEMORY` ：将所分配的内存初始化为 0；

如果 HeapAlloc 成功，它会返回指向所分配内存块的指针。

HeapAlloc 实际上是调用了 RtlAllocateHeap 来进行内存分配的。

### CRT 分配函数

也可以使用标准 C 所定义的内存分配函数来从堆中分配内存，例如 malloc 和 calloc。这两个函数会从我们之前提到过的 CRT 堆中分配内存。

编译器的运行时库会在初始化阶段创建 CRT 堆，创建之前会选择三种模式之一，大多数时候选择的都是系统分配模式，在这种模式下，CRT 堆是建立在 Win32 堆之上的。这里的意思是说，CRT 堆会使用 Win32 堆管理器的 HeapCreate 接口来创建出一个堆，实际上不存在 Win32 堆这个东西，只有一个 Win32 堆管理器。

在系统模式下，向 CRT 堆申请内存，最后还是会调用到 HeapAlloc 接口来分配内存块，这时候向 HeapAlloc 传入的堆句柄，指向的就是运行时库初始化阶段创建的那个堆。

既然 HeapAlloc 已经可以从堆上分配内存了，为啥还需要再封装一个 malloc 呢？首先的原因是为了降低编译器与操作系统之间的耦合，另一个好处是可以借助这些中间函数加入内存检查功能来辅助调试。

因此，一般情况下，进程运行起来之后会有两个堆，即进程默认堆(可通过 GetProcessHeap() 获取) 和 CRT 堆。

### 释放从堆中分配的内存

应该调用 HeapFree API 来释放使用 HeapAlloc API 分配的内存。其原型为：

```c
BOOL HeapFree(HANDLE hHeap, DWORD dwFlags, LPVOID lpMem);
```

其中 dwFlags 可以指定 `HEAP_NO_SERIALIZE` 标记，用来告诉 Win32 堆管理器不需要对该次调用进行防止并发的串行化操作以提高速度。

HeapFree 被链接到 NTDLL.DLL 中的 RtlFreeHeap 函数。所以他们两个是等价的。

通过 malloc 分配的内存应当使用 free 来释放，通过 new 分配的内存应当使用 delete 来释放，但二者最终都是调用 HeapFree 来释放内存块的。

### GlobalAlloc 和 LocalAlloc

16 位的 Windows (Windows 3.x) 支持所谓的全局堆和局部堆，简单来说，局部堆是进程内的，全局堆是系统提供给所有进程来共享使用的，从全局堆和局部堆分配内存的 API 是不同的，全局堆应该使用 GlobalAlloc 和 GlobalFree, 局部堆应该使用 LocalAlloc 和 LocalFree。

NT 系列的 Windows 不再支持全局堆，但为了保持与 16 位 Windows 程序兼容，以上 API 仍然保留着，不过无论是 GlobalAlloc 还是 LocalAlloc 实际上都是从进程的默认堆上来分配内存的。也就是说，今天这两个函数已经没有大的差异，新的程序也不应该使用它们。

### 解除提交

释放从堆上分配的内存并不意味着堆管理器会立刻把这个内存块所对应的空间立刻归还给系统的内存管理器 (调用 HeapFree 并不意味着一定会调用 VirtualFree) 。为了减少与内存管理器的交互次数，堆管理器只有在一下两个条件 **都满足** 时才会立即调用 ZwFreeVirtualMemory 来向内存管理器释放内存，通常称为解除提交 (Decommit)。

- 本次释放的堆块大小超过了堆参数中 DeCommitFreeBlockThreshold 所代表的阈值，默认取值为 4KB；
- 累积起来的总空闲空间 (包括本次) 超过了堆参数中的 DeCommitTotalFreeThreshold 所代表的阈值，默认取值为 64 KB；

也就是说，当要释放的堆块超过 4KB, 并且堆上的总空闲空间达到 64KB 时，堆管理器才会立即向内存管理器执行解除提交操作，真正释放内存；否则，堆管理器会把这个块加入到空闲列表中，并更新堆管理区的总空闲值。


## 堆的内部结构

### 结构和布局

可以把堆管理器想象成经营内存业务的零售商，它从 Windows 内核的内存管理器那里批发内存过来，然后零售给应用程序。

下图画出了一个 Win32 堆所拥有的内存块：

![]( {{site.url}}/asset/software-debugging-heap-memory-layout.png )

左边的大矩形是这个堆创建之初所批发过来的第一个内存段 (Segment), 我们将其简称为 0 号段。

每个堆中至少会有一个 0 号段，最多可以有 64 个段，在一个段用完之后，如果这个堆是可增长的, (堆标志中包含 `HEAP_GROWABLE` 标记)，那么会创建一个新的段。上图右上角的那个就是新创建的内存段。

注意上图，内存段中的内存被分割成一系列 **不同大小** 的堆块 (chunk)，每个堆块的起始处都有一个 8 字节的 `HEAP_ENTRY` 结构。这个 `HEAP_ENTRY` 结构的前 2 个字节记录了这个 chunk 的大小，分配粒度通常为 8 字节, 也就是说每个 chunk 的最大值是 2^16 * 8 = 512KB, 换句话说，即使内存段中的空闲空间远大于 512KB, 按照这种方法分配的 chunk 最大也只能有 512KB。

不过这并不意味着不可以从 Win32 堆管理器上分配到更大的内存块。当一个应用程序要分配大于 512KB 的堆块时，如果堆标志中包含 `HEAP_GROWABLE`, 那么堆管理器会直接调用 ZwAllocateVirtualMemory 来满足这次分配，并把分配得到的地址记录在 HEAP 结构的 VirtualAllocdBlocks 所指向的链表中。可以参考上图中从 VirtualAllocdBlocks 引出的虚线，它所指向的 `HEAP_VIRTUAL_ALLOC_ENTRY` 就是由 ZwAllocateVirtualMemory 所分配的内存。

这意味着，堆管理器批发过来的大内存块有两种形式，一种形式是形如 0 号段那样的段，另一种形式就是直接的虚拟内存分配，我们将后者称为大虚拟内存块。因为大虚拟内存块是用链表的形式来管理的，所以没有数量的限制。

接着看 0 号段中的内容。在 0 号段开始处有一个 HEAP 结构，这个结构中定义了许多字段用来记录这个堆的属性和 “资产” 状况。例如 `Segments[64]` 这个数组记录了这个堆拥有的所有段，VirtualAllocdBlocks 记录了这个堆所拥有的所有大虚拟内存块。

在 0 号段 HEAP 结构后面，紧跟着一个 `HEAP_SEGMENT` 结构，这个结构用来描述段本身。`HEAP` 结构是 0 号段独有的，`HEAP_SEGMENT` 则是每个段都有的结构，对于其他段来说，`HEAP_SEGMENT` 就在段的起始处。

`HEAP_SEGMENT` 结构后面是一个特殊的堆块，用来存放已经释放的堆块的信息。当程序释放一个小型堆块时，堆管理器可能把这个堆块的信息记录到一个列表中，然后就返回。在新分配堆块时，堆管理器会优先查看这个列表中是否有合适的堆块。因为这个从这个列表中分配堆块是优先于其它分配逻辑的，所以它又叫做前端堆 (Front End Heap)。前端堆主要用来提高释放和分配堆块的速度。

接着就是分配给用户的用户堆块了。注意每个堆块的前 8 个字节都是 `HEAP_ENTRY` 结构，而该结构的前 2 个字节记录了该堆块的大小。所以你可以在代码中通过指针前移的方式来获取到这个大小。

### HEAP 结构

堆管理器使用 HEAP 结构来记录和维护堆的管理信息，因此我们把这个结构称为堆的管理结构，有时也称为堆的头结构。

事实上, HeapCreate 返回的句柄便是指向 HEAP 结构的指针。

下图显示了 HEAP 结构各个字段及在进程堆中的取值：

![]( {{site.url}}/asset/software-debugging-heap-struct.png )

FreeLists 是一个包含 128 个元素的数组，用来记录堆中空闲堆块链表的表头。当有新的分配请求时，对管理器会遍历这个链表寻找可以满足请求大小的最接近堆块，如果找到了，就把这个块分配出去，否则，便考虑为这次请求提交新的内存页和建立新的堆块。当释放一个堆块时，除非是这个堆块满足解除提交的条件要直接释放给内存管理器，大多数情况下也都是将其修改属性后加入到空闲链表中。

### HEAP_SEGMENT 结构

下图给出了用来描述堆中内存段 `HEAP_SEGMENT` 结构的各个字段，其中字段的取值是针对进程堆的第一个段的：

![]( {{site.url}}/asset/software-debugging-heap-segment-struct.png )

### HEAP_ENTRY 结构

描述堆块的最重要数据结构就是 `HEAP_ENTRY`。下表列出了该结构的字段：

![]( {{site.url}}/asset/software-debugging-heap-entry-struct.png )

其中，Flags 字段代表堆块的状态，其值是下面几个标志位的组合：

![]( {{site.url}}/asset/software-debugging-heap-entry-flags.png )

`HEAP_ENTRY` 结构的长度固定位 8 个字节，位于堆块的起始处，其后便是堆块的用户数据。也就是说，将 HeapAlloc 函数返回的地址减去 8 个字节便是这个堆块的 `HEAP_ENTRY` 结构的地址。

### 分析堆块的分配和释放过程

下面举例来说明从堆上分配和释放内存块的过程。

编写如下代码：

```c
#include <windows.h>

int main(int argc, char* argv[])
{
	void* pStruct = HeapAlloc(GetProcessHeap(), 0, MAX_PATH);

	HeapFree(GetProcessHeap(), 0, pStruct);

    return 0;
}
```

在函数开头加上断点，执行 HeapAlloc 之后查看 pStruct 的地址为 0x00142ab0, 看一下这里的内存：

![]( {{site.url}}/asset/software-debugging-heap-debug-sample.png )

可以看到整个内存区都被填充为 0xBAADF00D (BAD FOOD)。这是因为我们在调试器中运行程序，系统自动启动了堆的调试支持。

该地址之前 8 个字节就是 `HEAP_ENTRY` 结构体：

![]( {{site.url}}/asset/software-debugging-heap-debug-sample2.png )

其中 0x24 是以粒度为单位的块大小，0x24*8 = 288 == 用户请求大小 256 + UnusedBytes 28。

Flags 为 7, 表明该块处于占用 (Busy) 状态, SegmentIndex 为 0，表明这个块位于这个堆的 0 号段。

继续执行 HeapFree, 再次观察刚才的那两个地址：

![]( {{site.url}}/asset/software-debugging-heap-debug-sample3.png )

可以看到内存区被填充为 feee (FREE), 表示此内存已经成为空闲状态，此时的 `HEAP_ENTRY` 结构变为：

![]( {{site.url}}/asset/software-debugging-heap-debug-sample4.png )

大小从刚才的 0x24 变成了 0xb1, 即 0xb1*8=1416 字节，这是堆管理器将刚才释放的块与其后面的空闲区域合并成了一个大的空闲块，留作以后使用。合并的过程叫做拼接 (Coalesce)。

另外 Flags 变为了 0x14, 表示这个块变成了空闲状态。

### 使用 !heap 命令观察堆块信息

使用 WinDBG 的 !heap 命令加上 -h 开关和堆句柄可以显示出堆的堆块信息：

![]( {{site.url}}/asset/software-debugging-heap-command.png )

上述第 7 行显示的是空闲链表中堆块的总大小 (0x9ae)，它是以粒度为单位的，字节数为 0x9ae*8 = 0x4d70。第 13, 14 行显示出了两个空闲堆块的信息，第一个的大小是 0x1ea8, 第二个的大小是 0x2ec8, 加起来刚好是 0x4d70.

上述列表中对堆块的显示格式是：

```
堆块的起始地址: 前一个堆块的字节数 . 本堆块的字节数 [堆块标志] - 堆块标志的文字表示 (堆块的用户数据区字节数) (堆块的标记序号)

例如：
00140680: 00040 . 01808 [01] - busy (1800) (Tag d0)
```

其中堆块的起始地址就是 `HEAP_ENTRY` 结构的地址。
