---
layout: post
title:  "堆"
date:   2017-07-10 10:36:30 +0800
categories: debugging
---

 
 


前段时间看了 《软件调试》 中有关堆的章节，现在把 [之前的笔记](https://dong-dada.github.io/debugging/2017/06/05/software-debugging-19.html) 整理消化一下。


## 堆管理器

Windows 中负责堆的创建、分配、释放、销毁操作的组件被称为堆管理器，它对外暴露了一系列 API, 供程序使用堆管理器的功能。这些 API 包括：

- HeapCreate;
- HeapDestroy;
- HeapAlloc;
- HeapReAlloc;
- HeapFree;

程序中所有的堆都由上述 API 进行管理。首先通过 HeapCreate 创建一个堆，接着使用 HeapAlloc 在这个堆上分配内存，内存使用完毕后通过 HeapFree 来释放，堆没用了之后使用 HeapDestroy 来销毁。


## 进程默认堆 和 CRT 堆

Windows 系统创建一个新的进程时，会通过堆管理器创建一个默认堆。可以通过 GetProcessHeap() 获取到这个堆的句柄，得到该句柄之后，就可以使用 HeapAlloc 在这个堆上分配内存。

进程的运行时库在初始化时会创建一个 CRT 堆，用于支持 malloc, free, new, delete 等 C/C++ 内存分配操作。这个 CRT 堆的创建最终也是调用了 HeapCreate 函数。CRT 堆的 malloc 实现有三种模式，不过大多数情况下使用的都是系统模式，这意味着 malloc 会直接调用 HeapAlloc, free 会直接调用 HeapFree, 相当于把管理堆的任务直接交给了 Windows 堆管理器。

默认堆 和 CRT 堆在创建时，给 HeapCreate 的最后一个参数 dwMaximumSize 传入了 0, 这意味着它们是可增长的，没有大小限制。

### CRT 堆和运行时库的关系

CRT 堆的创建是通过 `_heap_init()` 函数来实现的， CRT 初始化时会调用这个函数来创建 CRT 堆。换句话说，CRT 被初始化几次，就会有几个 CRT 堆被创建。

CRT 初始化的次数是由运行时库决定的，程序里有几个运行时库的实例，对应的初始化就会调用几次。

运行时库的实例有几个，跟程序所使用的链接选项有关。假设现在有一个 CTest.exe 它运行后会加载 DllTest.dll：
- CTest 和 DllTest 都使用动态链接同一个运行时库 msvcr90.dll, 只会有一个运行时库实例；
- CTest 和 DllTest 分别动态链接不同的运行时库，比如 msvcr90.dll 和 msvcr140.dll, 那么会有两个运行时库实例；
- CTest 静态链接了 msvcr90.dll, DllTest 动态加载 msvcr90.dll, 会有两个运行时库实例；
- CTest 和 DllTest 都静态链接了 msvcr90.dll, 会有两个运行库实例；

两个运行时库所创建的堆是不同的，用 A 堆上分配的内存地址去释放 B 堆中的内容，会有问题 (释放不成功)。

### 用 WinDBG 观察默认堆和 CRT 堆

直接运行 `!heap` 命令可以列出进程当前的所有堆：

```
0:000> !heap
Index   Address  Name      Debugging options enabled
  1:   00420000                 tail checking free checking validate parameters
  2:   00640000                 tail checking free checking validate parameters
```

Index 表示这个堆的编号，编号 1 代表的是进程的默认堆。其中的 Address 列表示这个堆的其实地址。

使用 `!heap -h` 命令可以列出进城当前所有堆，并列出这个堆中所有的 Segment:

```
0:000> !heap -h
Index   Address  Name      Debugging options enabled
  1:   00420000 
    Segment at 00420000 to 00520000 (0000f000 bytes committed)
  2:   00640000 
    Segment at 00640000 to 00650000 (00003000 bytes committed)
    Segment at 00520000 to 00620000 (00042000 bytes committed)
```


## 堆的内部结构

一个堆中包含了多个内存段 Segment, 每个 Segment 中又包含了多个堆块 chunk, 每个 chunk 的开头都有一个 `HEAP_ENTRY` 结构体来记录堆块的大小等信息：

![堆的内部结构]( {{site.url}}/asset/debugging-heap-layout.png )

这里的 chunk 就对应于使用 HeapAlloc 分配的内存块。chunk 的大小是不固定的，分配粒度为 8 字节，最大 512KB。如果要分配更大的内存块，则会通过 ZwAllocateVirtualMemory 来直接分配虚拟内存块。

堆创建之后所批发来的第一个内存段被称为 0 号段 Segment 0. 这个段中的第一个 chunk 是一个 `_HEAP` 结构，它记录了关于这个堆的详细信息。你可以使用 `dt ntdll!_HEAP 00420000` 命令来查看该结构的信息，字段很多，这里就不介绍了。

多个 chunk 并不是顺序排列的，中间会有间隔，你可以使用 `!heap -a 00420000` 命令来查看堆中的所有 chunk :

```
0:000> !heap -a 00420000
Index   Address  Name      Debugging options enabled
  1:   00420000 
    Segment at 00420000 to 00520000 (0000f000 bytes committed)
    Flags:                40000062
    ForceFlags:           40000060
    Granularity:          8 bytes
    Segment Reserve:      00100000
    Segment Commit:       00002000
    DeCommit Block Thres: 00000200
    DeCommit Total Thres: 00002000
    Total Free Size:      00000612
    Max. Allocation Size: 7ffdefff
    Lock Variable at:     00420138
    Next TagIndex:        0000
    Maximum TagIndex:     0000
    Tag Entries:          00000000
    PsuedoTag Entries:    00000000
    Virtual Alloc List:   004200a0
    Uncommitted ranges:   00420090
            0042f000: 000f1000  (987136 bytes)
    FreeList[ 00 ] at 004200c4: 0042c008 . 004256b8  
        004256b0: 00080 . 00010 [104] - free
        00425800: 00028 . 00010 [104] - free
        004263f8: 00028 . 00018 [104] - free
        00424dc0: 00028 . 00018 [104] - free
        00425c60: 00028 . 00020 [104] - free
        00425f28: 00028 . 00020 [104] - free
        00425550: 00028 . 00020 [104] - free
        0042c000: 00030 . 02fe0 [104] - free

    Segment00 at 00420000:
        Flags:           00000000
        Base:            00420000
        First Entry:     00420588
        Last Entry:      00520000
        Total Pages:     00000100
        Total UnCommit:  000000f1
        Largest UnCommit:00000000
        UnCommitted Ranges: (1)

    Heap entries for Segment00 in Heap 00420000
         address: psize . size  flags   state (requested size)
        00420000: 00000 . 00588 [101] - busy (587)
        00420588: 00588 . 00250 [107] - busy (24f), tail fill
        ...
        ... 省略
        ...
        0042c000: 00030 . 02fe0 [104] free fill
        0042efe0: 02fe0 . 00020 [111] - busy (1d)
        0042f000:      000f1000      - uncommitted bytes.
```

最后几行中记录了当前已分配的 chunk 信息。busy 表示该堆块正在使用中, free 表示该堆块空闲，可供分配。





