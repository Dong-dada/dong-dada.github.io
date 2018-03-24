---
layout: post
title:  "《软件调试》 学习 06 用户态调试模型"
date:   2017-04-10 10:23:30 +0800
categories: debugging
---

* TOC
{:toc}

本章将介绍 Windows 操作系统中用于支持用户态调试的模型和各种基础设置，包括内核中的调试支持例程、调试子系统和调试 API 等。

## 概览

回想我们在调试器中(比如 VC6)中调试程序(比如一个 HelloWorld 程序)的情景，可以随时将被调试程序中断到调试器，然后观察变量信息或跟踪执行，仿佛一切都在调试器中进行。但事实上，从进程的角度看，VC6 和 HelloWorld 是两个分别运行的进程，分别有自己的进程空间。而且根据我们在上一章对进程和进程空间的介绍，每个进程的空间都是收到严密的系统机制所保护的。那么调试器进程是如何观察和控制被调试进程的呢？简单的回答是使用调试 API。但要深入理解调试 API 是如何工作的，就必须挖掘调试子系统在调试中所起的作用。

下图画出了在 Windows 下进行用户态调试时参与调试过程中的各个角色，包括调试器进程 (Debugger Process), 被调试进程 (Debuggee Process), 调试子系统，调试 API，以及位于 NTDLL.DLL 中和内核中的支持函数。

![]( {{ site.url }}/asset/software-debugging-user-mode-debugging.png )

- 调试器进程是调试过程的主导者，它负责发起调试会话，读取和处理调试事件，并通过用户界面接受调试人员下达的调试指令，然后执行。调试器进程通过调试 API 与系统的调试支持函数和调试子系统进行交互；
- 被调试进程是调试目标。为减少海森伯效应，应尽可能减少向被调试进程中加入支持调试的设施，以免影响问题的重现和分析；
- 调试子系统是沟通被调试进程和调试器进程的桥梁，它的职责是帮助调试器完成各种调试功能，比如控制和访问被调试进程，管理和分发调试事件，接收和处理调试器的服务请求。从内部来看，调试器大多时候是在和调试子系统对话；

### 调试子系统

调试子系统分为三个部分：
- 位于 NTDLL.DLL 中的支持函数，包括供调试器使用的 DbgUi 开头的 API；以 Dbg 开头的用于实现调试的 API，如 DbgBreakPoint 是 DbgBreak API 的实现；
- 位于内核文件中的支持函数，负责采集和传递调试事件，以及控制被调试进程。这些函数都是以 Dbgk 开头的；
- 调试子系统服务器，负责管理调试会话和调试事件，是调试消息(事件)的集散地，也是所有调试设施的核心；

### 调试事件驱动

与 Windows 程序的消息驱动机制类似，Windows 用户态调试时通过调试事件来驱动的。调试器程序在与被调试进程建立调试会话后，调试器进程便进入所谓的调试事件循环 (Debug Event Loop)，等待调试事件发生，然后处理，再等待，直到调试会话中止，其核心代码如下：

```cpp
while ( WaitForDebugEvent(&DbgEvt, INFINITE) )
{
    // 处理得到的调试事件

    // 处理完毕后，恢复调试目标继续执行
    ContinueDebugEvent(DbgEvt.dwProcessId, DbgEvt.dwThreadId, dwContinueStatus);
}
```

在处理调试事件的过程中，被调试进程处于挂起状态。处理调试事件之后，调试器调用 ContinueDebugEvent 将处理的结果回复给调试子系统，让被调试程序继续执行，调试器则调用 WaitForDebugEvent 继续等待。


## 采集调试信息

为了能了解到调试有关的系统动作，调试子系统的内核部分对外公开了一系列函数，供内核的其它部分调用，这些函数都是以 Dbgk 开头的，是调试子系统公开给内核其它部件的接口函数，我们将它们简称为 Dbgk 采集例程。

### 消息常量

Dbgk 采集例程将所有调试事件 (消息) 分为 8 种类型，并使用以下常量来代表不同类型的调试消息：

```cpp
typedef enum _DBGKM_APINUMBER
{
    DbgKmExecptionApi = 0,          // 异常
    DbgkmCreateThreadApi = 1,       // 创建线程
    DbgkmCreateProcessApi = 2,      // 创建进程
    DbgkmExitThreadApi = 3,         // 退出线程
    DbgkmExitProcessApi = 4,        // 进程退出
    DbgkmLoadDllApi = 5,            // 映射 DLL
    DbgkmUnloadDllApi = 6,          // 卸载 DLL
    DbgkmErrorReportApi = 7,        // 内部错误
    DbgkmMaxApiNumber = 8,          // 这组常量的最大值
} DBGKM_APINUMBER;
```

下面逐个介绍每种调试事件的采集过程。

### 进程和线程创建消息

操作系统的一大核心任务就是管理系统中运行的各个进程和线程，包括创建新的进程和线程、调度等待运行的线程、进程间通信、终止进程和线程、以及资源分配回收等，这些任务通常被称为进程管理。

当进程管理器创建新的用户态 Windows 线程时，它首先要为该线程建立必要的内核对象和数据结构，并分配栈空间，这些工作完成后，该线程就会处于挂起状态 (CREATE_SUSPEND)，而后进程管理器会通知环境子系统，子系统做必要的设置和登记，最后进程管理器会调用 PspUserThreadStartup 例程，准备启动该线程。为了支持调试，PspUserThreadStartup 总是会调用调试子系统的内核函数 `DbgkCreateThread`，以便让调试子系统得到处理机会。

DbgkCreateThread 会检查新创建线程所在的进程是否正在被调试，如果是，则会继续检查该进程的用户态运行时间是否是 0, 如果是 0 说明该线程是第一个线程。这时候会通过 DbgkSendApiMessage() 向 DebugPort 发送 DbgkmCreateProcessApi 消息如果不是，则发送 DbgkmCreateThreadApi 消息。调试器收到的进程创建消息(`CREATE_PROCESS_DEBUG_EVENT`) 和线程创建消息 (`CREATE_THREAD_DEBUG_EVENT`) 就是源于这两个消息的。

### 进程和线程退出消息

进程管理器的 PspExitThread 函数负责线程的退出和清除。为了支持调试，在销毁该线程的结构和资源之前，PspExitThread 会调用调试子系统的函数以便让调试器得到处理机会。如果正在退出的线程不是进程中的最后一个线程，那么 PspExitThread 会调用 DbgkExitThread 函数通知线程退出，如果是最后一个线程，那么 PspExitThread 会调用 DbgkExitProcess 函数通知进程退出。

DbgkExitThread 会检查是否处于调试状态，如果是，则通过 DbgkpSendApiMessage() 向 DebugPort 发送 DebkmExitThread 消息；

调试器收到的线程退出消息(`EXIT_THREAD_DEBUG_EVENT`) 和 进程退出消息 (`EXIT_PROCESS_DEBUG_EVENT`) 就是源于这两个消息。

### 模块加载和卸载消息

DLL 的加载和卸载最终都会调用 DbgkMapViewOfSection 和 DbgkmUnMapViewOfSection 这两个函数。它们检测到 DebugPort 不为空时，会通过 DbgkpSendApiMessage() 向 DebugPort 发送 DbgkmLoadDllApi, DbgkmUnloadDllApi 消息。

调试器收到的模块加载消息 (`LOAD_DLL_DEBUG_EVENT`) 和 模块卸载消息 (`UNLOAD_DLL_DEBUG_EVENT`) 就是源于这两个消息。

### 异常消息

为了支持调试，系统会把被调试程序中发生的所有异常发给调试器。内核中的 KiDispatchException 是分发异常的枢纽，它会给每个异常安排最多两轮被处理的机会，对于每一轮处理机会，它都会调用调试子系统的 DbgkForwardException 函数来通知调试子系统。

DbgkForwardException 函数既可以向进程的异常端口发送消息，也可以向调试端口发送消息，KiDispatchException 函数会在调用它时通过一个布尔变量来指定。如果向调试端口发消息，那么 DbkgForwardException 会通过 DbgkpSendApiMessage 函数发送 DbgkmExceptionApi 消息。

调试器收到的异常事件 (`EXCEPTION_DEBUG_EVENT`) 和输出调试字符串事件 (`OUTPUT_DEBUG_STRING_EVENT`)，都是源于 DbgkmExceptionApi 消息。

简单来说，系统的进程管理器、内存管理器和异常分发函数会调用调试子系统的 Dbgk 采集例程，来向调试子系统通报调试消息，这些例程被调用后会产生一个 `DBGKM_APIMSG` 结构，然后通过 DbgkpSendApiMessage 函数来发送调试消息；


## 发送调试消息

上一节介绍了调试信息的采集过程，这一节介绍信息采集后如何发送给调试子系统服务器。

### DbgkpSendApiMessage 函数

消息发送给调试子系统服务器最终是调用了 DbgkpSendApiMessage 函数。

因为 WindowsNT 和 Windows2000 的调试子系统服务器位于用户态。所以 DbgkpSendApiMessage 会通过 LPC(本地过程调用) 机制来发送调试消息。它会先把消息发送给 环境子系统的服务器进程，即 CSRSS.EXE，然后再转发给位于会话管理器进程中的调试子系统服务器，由它去通知等候调试事件的调试器。

在 WindowsXP 及后续的 Windows 中，调试子系统服务器被移到内核空间中，因此这些版本中的 DbgkpSendApiMessage 改为通过 DbgkpQueueMessage 来发送消息。

### 控制被调试进程

调试子系统设计了两个内核函数来控制被调试进程，他们是 DbgkpSuspendProcess 和 DbgkpResumeProcess。

在调试子系统给调试器发送调试事件之前，通常会先调用 DbgkpSuspendProcess 函数。**这个函数内部会调用 KeFreezeAllThreads() 函数冻结 (Freeze) 被调试进程中除了调用线程以外的所有线程**。这时，被调试进程中就只有当前线程 (发送调试消息的这个线程) 还活动着。

接着，调试子系统会执行实际的消息发送函数，在 Windows XP 前是使用 LPC 机制，从 XP 之后开始使用 DbgkpQueeuMessage 函数。这两种方法都会阻塞当前线程。换句话说，所有线程都停住了。

接着，调试子系统会通知调试器来读取调试信息，作处理后回复给调试子系统，后者会唤醒当前线程，再通过 DbkgpResuleProcess 来唤醒被冻结的其他线程。



## 调试子系统服务器 （XP 之后）

与 Windows 2000 和 NT 相比，Windows XP 对用户态调试子系统做了重大改进，将调试子系统服务器由用户态移入内核态。新的子系统是以新引入的内核对象 DebugObject 为核心的。

### DebugObject

DebugObject 内核对象是专门用于内核态调试的，它不仅承担了同步调试器和调试子系统的功能，而且也是调试器和调试子系统之间传递数据的重要纽带，取代了调试子系统各部件之间本来使用的 LPC 通信方式。以下是 DebugObject 的内部数据结构：

```cpp
typedef struct _DEBUG_OBJECT
{
    KEVENT EventsPresent;           // 用于指示有调试器事件发生的内核对象
    FAST_MUTEX Mutex;               // 用于同步
    LIST_ENTRY StateEventListEntry; // 保存调试事件的链表
    ULONG Flags;
} DEBUG_OBJECT;
```

结构中最重要的就是 StateEventListEntry 这个字段了，它是一个链表，用来存储调试事件，被称为调试消息队列。

DbgkpQueueMessage 函数会将调试信息插入到这个队列中，调试信息结构的定义如下：

```cpp
typedef struct _DBGKM_DEBUG_EVENT
{
    LIST_ENTRY EventList;       // 与兄弟节点相互连接的节点结构
    KEVENT ContinueEvent;       // 用于等待调试器回复的事件对象
    CLIENT_ID ClientId;         // 调试事件所在线程的线程 ID 和进程 ID
    PEPROCESS Process;          // 被调试进程的 EPROCESS 结构地址
    PETHREAD Thread;            // 被调试进程中触发调试事件的线程的 ETHREAD 结构地址
    NTSTATUS Status;            // 对调试事件的处理结果
    ULONG Flags;
    PETHREAD BackoutThread;     // 产生杜撰消息的线程
    DBGKM_MSG ApiMsg;           // 调试事件的详细信息
}
```

建立调试会话后，调试器工作线程便进入调试事件循环，等待调试事件，这实际上就是通过调动 NtWaitForDebugEvent 内核服务等待调试对象中的 EventsPresent 对象。得到调试对象后，调试器会读取 StateEventListEntry 调试消息队列，读取到调试消息并进行处理。

### 全景

下图画出了 WindowsXP 及 Windows Vista 用户态调试子系统的完整模型。图中虚线矩形框代表调试对象，其中双向链表是用来临时存储调试事件的消息队列，即 DebugObject 的 StateEventListEntry 字段，小旗标志代表调试对象中的 EventsPresent 事件对象。

![]( {{ site.url }}/asset/software-debugging-user-mode-debugging-xp.png )



