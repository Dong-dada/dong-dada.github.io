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

