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


## 低碎片堆

在堆上的内存空间被反复分配和释放一段时间后，堆上的可用空间可能被分割得支离破碎，当再试图从这个堆上分配空间时，即使可用空间加起来的总额大于请求的空间，但是因为没有一块连续的空间可以满足要求，那么分配请求就会失败，这种现象被称为堆碎片 (Heap Fragmentation)。

堆碎片与磁盘碎片的形成机理是一样的，但却比磁盘碎片的影响更大，因为多个磁盘碎片加起来仍可以满足磁盘分配需求，但堆碎片是无法累加起来满足内存分配要求的，因为堆函数返回的必须是地址连续的一块空间。

针对堆碎片问题，Windows XP 引入了低碎片堆 (Log Fragmentation Heap)，简称 LFH。首先，LFH 将堆上的可用空间划分成 128 个桶位 (Bucket)，编号为 1~128, 每个桶位的空间大小依次递增, 1 号桶为 8 个字节, 128 号桶为 16384 字节 (即 16KB)。当需要从 LFH 上分配空间时，堆管理器会根据堆函数参数中所请求的字节将满足要求的最小可用桶分配出去。例如，如果应用程序请求 7 个字节，而且 1 号桶空闲，那么就将 1 号桶分配给它，如果 1 号桶被占用，那么便尝试分配 2 号桶。

另外，LFG 为不同编号区域的桶规定了不同的粒度，桶的容量越大，分配桶时的粒度也越大，比如 1~32 号桶的粒度是 8 字节，这意味着这些桶的最小分配单位是 8 字节，不足 8 字节的分配请求也会分配给 8 字节，下表列出了 LFG 各个桶位的分配粒度和最佳适用范围：

![]( {{site.url}}/asset/software-debugging-lfh-bucket.png )

通过 HeapSetInformation API 可以对一个已经创建好的 Win32 堆启用低碎片堆支持。调用 HeapQueryInformation API 可以查询一个堆是否启用了 LFH。

关于 LFH 堆的一篇文章：
[关于LFH堆](http://blog.csdn.net/hanzz2007/article/details/6805075)


## 堆的调试支持

为了帮助发现内存有关的问题，堆管理器提供了一系列功能来辅助调试；
- 堆尾检查 (Heap Tail Checking), 简称 HTC, 是在堆块的末尾附加额外的标记信息 (通常为 8 个字节)，用于检查堆块是否发生溢出，其原理与我们上一章介绍的防止栈上缓冲区溢出的栅栏字节类似。
- 释放检查 (Heap Free Checking), 简称 HFC, 是在释放堆块时对堆进行各种检查，可以防止多次释放同一个堆块；
- 参数检查，对传递给堆管理器的参数进行更多的检查；
- 调用时验证 (Heap Validation on Call), 简称 HVC, 即每次调用堆函数时都对整个堆进行验证和检查；
- 堆块标记 (Heap Tagging), 即为堆块增加附加标记 (Tag), 以记录堆块的使用情况或其他信息；
- 用户态栈回溯 (User mode Stack Trace), 简称 UST, 即将每次调用的堆函数的函数调用信息 (Calling Stack) 记录到一个内存数据库中；
- 专门用于调试的页堆 (Debug Page Heap), 简称 HPH 堆；

### 全局标志

创建堆时，堆管理器根据当前进程的全局标志来决定是否启用堆的调试功能。操作系统的进程加载器在加载一个进程时会从注册表中读取进程的全局标志，具体来说是在注册表的 `HKEY_LOCAL_MACHINE\SOFTWARE\Microsoft\Windows NT\CurrentVersion\Image File Execution Options` 表键下寻找以改程序名命名的子键，如果存在，那么读取该子键下面的 GlobalFlags 键值。

![]( {{site.url}}/asset/software-debugging-heap-debug-global-flags.png )

如果在调试器中运行程序，并且该程序没有设置全局标志的话，那么会默认将全局标志设为 0x70, 也就是启用 htc, hfc, hpc 这三个堆调试功能。

### 释放检查

很多堆损坏的情况是由于错误的释放动作导致的，比如多次释放同一个堆块，或者从一个堆释放本不属于这个堆的堆块。堆释放检查 (HFC) 功能可以比较有效地发现这类问题。

为了说明 HFC 的作用，我们在 VC6 中编写如下程序：

```c
#include <stdio.h>
#include <windows.h>

int main(int argc, char* argv[])
{
    void* p;
    BOOL bRet;
    HANDLE hHeap;

    hHeap = HeapCreate(0, 4096, 0);
    p = HeapAlloc(hHeap, 0, 20);
    
    bRet = HeapFree(hHeap, 0, p);
    printf("HeapFree return  first, %d\n", bRet);

    bRet = HeapFree(hHeap, 0, p);
    printf("HeapFree return second, %d\n", bRet);

    bRet = HeapValidate(hHeap, 0, NULL);
    printf("HeapValidate return bRet = %d, \n", bRet);

    bRet = HeapDestroy(hHeap);
    printf("HeapDestry return bRet = %d, \n", bRet);

    return getchar();
}
```

直接运行 Debug 或 Release 版本，命令行窗口在输出如下内容后停止了输出，CPU 占用变得很高：

![]( {{site.url}}/asset/software-debugging-heap-hfp-sample.png )

第一行输出表明第一次执行 HeapFree 成功了，第二行输出表明第二次执行 HeapFree 明显发生了错误，但堆管理器并没有返回 False 来报告这个错误。直到执行 HeapValidate 对堆做全面检查的时候，该函数遇到了问题，出现了类似于死循环的症状。

下面启用堆释放检查功能，这可以通过 gflag 工具来实现，它会帮助我们修改注册表，打开 CTest.exe 的 HFC 功能：

```
gflags -i CTest.exe +hfc
```

此时再次执行 CTest.exe, 得到的结果是：

![]( {{site.url}}/asset/software-debugging-heap-hfp-sample2.png )

可以看到第二次 HeapFree 尝试释放一个已经释放的堆时，被及时制止了，因此 HeapValidate 能够正常执行。

堆释放检查可以发现并制止不正常的堆释放操作。但它也会启用带有全面检查功能的分配和释放函数，导致堆的分配和释放速度急速下降。


## 栈回溯数据库

当调试内存问题时，很多时候我们希望知道每个内存块是由哪段代码或哪个函数分配的，而且最好有这个函数被调用的完整过程，这样便可以大大提高定位错误代码的速度。堆管理器所实现的用户态栈回溯 (User-Mode Stack Trace, 简称 UST) 机制就是为了实现这个目的而设计的。

### 工作原理

如果当前进程的全局标志中包含了 UST 标志 (`FLG_USER_STACK_TRACE_DB`, 0x1000), 那么堆管理器会为当前进程分配一块大的内存区，并建立一个 `STACK_TRACE_DATABASE` 结构来管理这块内存区，然后使用全局变量 `ntdll!RtlpStackTraceDataBase` 指向这块内存结构。

这个内存区被称为用户态栈回溯数据库 (User-Mode Stack Trace Database), 简称栈回溯数据库或 UST 数据库。下图显示了 UST 数据库的头结构：

![]( {{site.url}}/asset/software-debugging-heap-ust-struct.png )

上述结构中最重要的是 Buckets 这个指针数组，数组中的每个元素都指向一个桶位。堆管理器在存放栈回溯记录时，先计算这个记录的哈希值，然后对桶位数求余，将得到的值作为这个记录所在的桶位。位于同一个桶位的多个记录是以链表的方式链接在一起的。每个栈回溯记录是一个 `RTL_STACK_TRACE_ENTRY` 结构。下图列出了该结构的各个字段：

![]( {{site.url}}/asset/software-debugging-heap-stack-trace-entry-struct.png )

可以看到 BackTrace 这个数组中记录了发起堆块分配请求的函数调用栈。

建立了 UST 数据库后，当堆块分配函数再被调用的时候，堆管理器便会将当前栈回溯信息记录到 UST 数据库中。

### DH 和 UMDH 工具

可以使用 DH.EXE (Display Heap) 和 UMDH.EXE (User-Mode Dump Heap) 工具来查询包括 UST 数据库在内的堆信息。

尽管这两个工具的用法不尽相同，但它们的基本功能和工作原理是基本一致的。都是利用堆管理器的调试功能将堆信息显示出来或转储到文件中。

例如，通过以下命令可以将进程 5622 的堆信息转储到文件 DH_5622.dmp 中：

```
C:\>set _NT_SYMBOL_PATH=D:\symbols
C:\>dh -p 5622
```

尽管以 .dmp 为后缀，但 DH 生成的文件就是文本文件，内部包含了进程中所有堆的列表和 UST 数据库中的所有栈回溯记录；

可以针对应用程序运行的不同时间点生成多个转储文件，然后使用 dhcmp.exe 工具比较这些文件的差异，利用这种方法可以为定位内存泄漏提供线索。WinDBG 工具包中的 UMDH 将转储和比较的功能都集成在一个工具中，因此更适合使用它来定位内存泄漏。

### 定位内存泄漏

下面介绍使用 UMDH 来定位内存泄漏的基本步骤。

首先，使用 gflags 工具启用 ust 功能，也就是在程序所在目录执行 `gflags /i CTest.exe +ust`.

然后运行 CTest.exe 程序，并使用 UMDH 工具对其进行第一次采样，也就是执行 `c:\windbg\umdh -p:1228 -d -f:u1.log -v`

接着继续运行 CTest.exe, 使其执行内存分配，然后使用 UMDH 工具对其进行第二次采样，也就是执行 `c:\windbg\umdh -p:1228 -d -f:u2.log -v`

最后，使用 UMDH 比较两个采样文件，也就是执行 `c:\windbg\umdh -h u1.log u2.log -v`：

![]( {{site.url}}/asset/software-debugging-heap-memory-leak-umdh.png )

UMDH 会将比较结果用以下格式显示出来：

```
+ 字节差异 (新字节数 - 旧字节数) 新的发生次数 allocs BackTrace UST记录的索引号
+ 发生次数差异 (新次数值 - 旧次数值) BackTrace UST记录的索引号 allocations 栈回溯列表
```

对应到比对结果上，可以看出 UMDH 共发现了一个差异，第 9~18 行报告了这一差异的详情。

第 9 行的含义是，索引号为 BackTraceA2 的 UST 记录在两次采样中新增了 100 字节，新的字节数为 11308, 上次字节数为 11208。这一记录所代表的函数调用发生了 20 次。

第 10 行的含义是，BackTraceA2 所代表的调用过程在两次采样间新增 1 次，新的发生次数是 20 次，上次发生次数为 19 次。

最后一行的含义是，第 2 次采样比第 1 次采样增加了 128 个字节，其中 100 字节属于用户数据区，28 字节属于堆的管理信息。其中 8 字节属于 `HEAD_ENTRY` 结构，另 20 字节为堆末尾的自动填充和 `HEAP_ENTRY_EXTRA` 结构。


## 堆溢出和检测

我们在上一章讨论过，如果堆分配在栈上的缓冲区写入超过其容量的数据，就可能将存放在栈上的栈帧指针、函数返回地址等信息覆盖掉，即所谓的栈缓冲区溢出。

类似的，对于分配在堆上的缓冲区也可能因为访问其分配空间以外的区域而导致溢出，即所谓的堆缓冲区溢出，有时也简称为堆溢出。

### 堆缓冲区溢出

根据我们前面的介绍，堆被划分成很多堆块 (chunk), 每个堆块又可以分为用于管理该堆块的控制数据和该堆块的用户数据区两个部分。对于已经分配的堆块，用户数据前面是 `HEAP_ENTRY` 结构，后面可能会有 `HEAP_ENTRY_EXTRA` 结构。对于空闲的堆块，起始处是一个 `HEAP_FREE_ENTRY` 结构。

因为堆上的用户数据和控制数据是混合存放的，并不是把所有的控制数据放在一个单独的地方特殊保护起来的，因此如果访问用户数据区之前或之后的数据都可能将控制数据覆盖掉。这就是堆缓冲区溢出。

![]( {{site.url}}/asset/software-debugging-heap-overflow.png )

如上图所示, p0 是堆块起始处, p1 到 p2 是用户数据区, HeapAlloc 会将地址 p1 以返回值的形式返回给用户，假设应用程序使用 pMem 来记录这个地址。

p2 到 p3 是可能存在的 `HEAP_ENTRY_EXTRA` 结构, p3 开始就是下一个堆块。

正常情况下，用户使用 pMem 只应该访问 p1 到 p2 的空间。但是因为指针操作不当, pMem 访问了 p1 以前的空间，就有可能破坏当前堆块的控制结构或者上一个堆块的数据，更严重时可能会破坏堆的段结构 (`HEAP_SEGMENT`) 或者整个堆的管理结构 (`HEAP`), 这种情况称为下溢 (underflow). 同理，如果 pMem 访问了 p2 以后的空间，程序便可能破坏存放在堆尾的管理信息和下一个堆块的数据，这种情况称为上溢 (overflow).

下面通过一个小程序来进一步说明：

```c
#include <windows.h>

int main(int argc, char* argv[])
{
    char* p1;
    char* p2;
    HANDLE hHeap;

    // 创建一个大小为 1024 字节的堆
    hHeap = HeapCreate(0, 1024, 0);

    // 分配一个大小为 9 字节的堆块
    p1 = (char*)HeapAlloc(hHeap, 0, 9);

    // 循环访问堆块，存在溢出
    for (int i = 0; i < 50; ++i)
    {
        *p1 = i;
        p1 ++;
    }

    // 再分配一个堆块
    p2 = (char*)HeapAlloc(hHeap, 0, 1);
    printf("Allocation after overflow got 0x%x\n", p2);

    HeapDestroy(hHeap);
    return 0;
}
```

在 HeapCreate 处加入断点，查看 hHeap 的值为 0x00390000, 这个值就是堆内存区的起始地址，即 HEAP 结构的地址，在 WinDBG 中通过 `!heap` 命令可以查看这个堆的信息：

![]( {{site.url}}/asset/software-debugging-heap-overflow-sample.png )

可以看到堆中已经有 4 个堆块，三个是占用堆块，一个是空闲堆块。这三个占用堆块是用于存放管理数据的。空闲堆块的起始地址为 0x00391e98, 我们用 dt 命令来查看该空闲堆块的信息：

![]( {{site.url}}/asset/software-debugging-heap-overflow-sample2.png )

可以看到除了 16 字节的控制数据外，用户数据区都被填充为 0xFEEEFEEE (free)。

单步执行第一个 HeapAlloc 语句，执行完之后观察返回的指针 p1, 其值为 0x00391ea0. 

![]( {{site.url}}/asset/software-debugging-heap-overflow-sample3.png )

此时内存的情况如上图中中间所示的情况：
- 其中前两行是 `HEAP_ENTRY` 结构，
- 第 3,4 行加上第 5 行的第一个字节是堆管理器分配给应用程序的 9 个字节，它们被填充为 baadf00d (bad food); 
- 之后的 5~7 行是堆管理器为了支持溢出检测而多分配的几个字节。
- 第 7,8 行中的 0xfeee 是堆尾补齐用的未使用字节 (unused bytes), 
- 第 9,10 行的 8 个字节是 `HEAP_ENTRY_EXTRA` 结构，因为没有启用 UST 功能，它的值都是 0;
- 第 11 行开始是新的空闲块，也就是说，堆管理器将刚刚的空闲块分配一部分来满足我们的需要，然后把空闲块的位置向后调整了；

继续执行代码到第二次调用 HeapAlloc 处，此时的内存情况变成了上图右侧的部分。

因为我们向申请空间的 9 个字节的缓冲区写入了 50 个字节的数据，因此缓冲区严重溢出，不仅覆盖掉了本堆块堆尾的内容，而且将本堆块后面空闲堆块的 `HEAP_FREE_ENTRY` 结构完全覆盖了。此时再执行 !heap 命令：

![]( {{site.url}}/asset/software-debugging-heap-overflow-sample4.png )

可以看到对于空闲块的描述完全混乱了，使用 dt 命令观察位于 0x00391ec0 处的 `HEAP_ENTRY_EXTRA` 结构：

![]( {{site.url}}/asset/software-debugging-heap-overflow-sample5.png )

可见所有信息都失常了，特别是最后 FreeList 的两个指针也被覆盖为无效的值。

这时候执行 HeapAlloc 语句，试图再次分配内存，会得到访问异常：

![]( {{site.url}}/asset/software-debugging-heap-overflow-sample6.png )

此时寄存器 ebx 的值为 0x2f2e2d2c, 这正是由于缓冲区溢出而覆盖到 `HEAP_FREE_ENTRY` 中的值，访问这个地址就会导致访问异常。

事实上，上面的异常是这样导致的，当第二次 HeapAlloc 被执行时，堆管理器接收到调用后会遍历空闲链表，寻找满足的空闲堆块，因为这个块的 Flags 字段被标识为占用块，所以堆管理器需要通过 Flink 指针访问下一个节点，因此访问到了 0x2f2e2d2c 这个地址。

值得说明的是，如果溢出时覆盖到链表指针处的数据是有效的内存地址，而且堆块的标志仍然是空闲，那么堆管理器下次分配堆块时便会从该块分割出一个堆块，然后调整空闲块的位置，这时需要更新链表指针，但因为此时指针已经指向了被修改过的地址，所以这时就会导致堆管理器向新的地址写入数据。如果精心设计新的地址和写入内容，那么便可能意外修改系统的数据，导致系统执行意外的代码，所谓的堆溢出攻击便是基于类似的原理实施的。

### 调用时验证

为了及时发现堆中的异常情况，可以让堆管理器在堆函数每次被调用时对堆进行检查，这便是堆的调用时验证 (Heap Validataion on Call) 功能，简称 HVC.

HVC 功能默认关闭，可以通过 `gflag /i CTest.exe +hvc` 来启用，或者在 WinDBG 中执行 `!gflag +hvc` 来启用。

启用该功能后，在 WinDBG 中打开并执行之前的 HeapOver 程序，就会得到如下信息：

![]( {{site.url}}/asset/software-debugging-heap-hvc-sample.png )

这是因为在堆发生溢出后又调用 HeapAlloc 函数时触发了堆管理器的验证功能，该功能检测到专用的空闲链表中位于 0x00391EC0 的元素被标志位占用后，触发断点异常并中断到了调试器中。使用 kn 命令可以观察到详细的调用过程：

![]( {{site.url}}/asset/software-debugging-heap-hvc-sample2.png )

其中 RtlValidateHeap 函数便是用来验证堆的函数，它会执行一系列检查，对堆的头信息、段信息和堆块进行全面检查。

### 堆尾检查 (Tail Check)

除了使用 HVC 功能，也可以使用堆尾检查 (Heap Tail Check, HTC) 功能来发现堆溢出。该功能的原理是在每个堆块的用户数据后面多分配 8 个字节的固定内容，通常为连续 8 个字节的 0xAB。如果该内容被破坏，便说明发生了溢出。

要让堆管理器检查这段内容是否被破坏，还需要启用其他两种调试检查：释放检查和参数检查。其目的是让堆管理器使用支持调试检查的 slowly 系列函数，如 RtlFreeHeapSlowly 和 RtlAllocHeapSlowly, 这里简称为慢速堆函数。这些函数在执行时会调用包含检查功能的释放和分配函数，如 RtlDebugFreeHeap 和 RtlDebugAllocHeap. 

注意这种机制只有在堆被释放的时候才会触发检查，这时候去判断内容是否被破坏。