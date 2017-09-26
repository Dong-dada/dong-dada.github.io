---
layout: post
title:  "《软件调试》 学习 13 事件追踪"
date:   2017-05-09 09:46:30 +0800
categories: debugging
---

* TOC
{:toc}

简单来说，事件追踪 (Event Tracing) 要解决的问题就是记录软件运行的动态轨迹，包括代码的执行轨迹和变量的变化轨迹。

## 简介

概括来说，事件追踪的目标就是要把软件执行的踪迹以一种可以观察的方式输出出来。与上一章介绍的日志机制相比，事件追踪机制更关心软件的 “变化和运动过程”。如果说日志只要记录下软件的重要事件的结果，那么事件追踪则要记录下导致这一结果的完整过程。因此要求事件追踪机制必须能够适应频繁的数据输出和庞大的数据量，这通常是普通的日志机制难以胜任的。由于这一原因，事件追踪机制通常是以二进制方式而不是文本方式来传输和记录信息的。另外，事件追踪机制的目标 “读者” 主要是开发人员，而事件日志的主要对象还包含系统管理员。这一差异导致了事件追踪信息通常会包含更多的技术细节，比如函数名称、变量取值等。最后一个差异是，日志机制通常是始终开启的，而事件追踪机制在软件正常运行时通常是关闭的，只在观察和分析时才会开启。

事件追踪机制不依赖于软件的调试版本，也不需要调试器。因此是了解软件执行情况的既简单又有效的途径。特别是在以下任务中有重要的作用：
- **性能分析**：分析程序的执行轨迹，寻找热点和瓶颈，确定优化目标；
- **产品期调试**：在用户环境中动态开启事件追踪机制，观察执行过程，寻找故障原因；
- **单元测试或自动测试**：记录下测试时的执行情况，为分析异常或错误提供依据；

为了使事件追踪机制能更好地满足以上应用的需要，好的事件追踪机制应该满足以下要求：
- **高效性**：开销小 (low overhead), 只要占用少量 CPU 时间和内存资源就可以支持频繁地输出信息；
- **动态性**：可以动态开启或关闭，无需重启软件或操作系统；
- **灵活性**：可以方便地定义信息的输出目标，比如重定向到不同位置的文件、调试器或其他观察工具；
- **选择性**：可以选择性开启要追踪的模块，而且最好可以定义过滤条件来控制信息的详细程度；
- **易用性**：可以让程序员非常容易地在代码中嵌入用于追踪的代码；在观察追踪信息时要简单易行，不需要复杂的工具或繁琐的步骤；

为了更好地支持事件追踪，从 Windows 2000 开始，Windows 操作系统提供了一套完整的事件追踪机制(包括内部实现、API、辅助工具)，称为 ETW (Event Tracing for Windows)。
- ETW 将追踪消息先输出到由系统管理和维护的缓冲区中，然后异步写入到追踪文件或送给观察器，如果系统崩溃，崩溃前没有写入到文件中的信息会记录到转储文件中；
- ETW 中的追踪消息是以二进制形式传输和存储的，所有格式信息存放在私有的消息文件中，这样可以防止软件本身的保密技术泄露；
- ETW 机制支持动态开启，没有开启时，开销几乎可以忽略(只需判断一个标志)；


## ETW 的架构

从架构上看，ETW 使用了经典的 Provider/Consumer/Controller 模式：

![]( {{site.url}}/asset/software-debugging-etw-architecture.png )

- Provider 提供器：指使用 ETW 技术来追踪的目标程序；
- Consumer 消耗器：接受和查看追踪消息的工具，如 Trace View 或输出文件；
- Controller 控制器：控制追踪会话 (Session) 的工具软件；

用于支持 ETW 传输追踪信息的通信连接被称为 ETW 会话 (Session)。

系统会为每个 ETW 会话维护一定数量的缓冲区用来缓存 ETW 消息，ETW 控制器可以定制这些缓冲区的大小。系统会定期 (每秒钟一次) 将缓冲区中的信息发送给 ETW 消耗器或者 flush 到追踪文件中。当缓冲区已满或者会话停止时，系统会提前 flush 缓冲区中的内容。ETW 控制器也可以显式要求 flush 缓冲区。当系统崩溃时，系统会将缓冲区中的内容放入到 DUMP 文件中。

ETW 会话可以在没有控制器或消耗器的情况下存在。从 Windows XP 开始，系统最多支持 64 个 ETW 会话。

从实现角度看，ETW 基础设施的核心部分是实现在内核模块中的，它是作为 WMI 的一个部分来实现的。既可以在应用程序中使用 ETW，也可以在驱动程序中使用 ETW，其用法基本一致。


## 提供 ETW 消息

ETW 通过 GUID 来标识系统中的 ETW 提供器 (Provider)。因此，ETW Provider 应该通过 RegisterTraceGuids API 来向系统注册自己的 GUID。这样，ETW controller 才可以通过 GUID 找到 provider：

```cpp
ULONG RegisterTraceGuids(
  WMIDPREQUEST RequestAddress,              // 回调函数
  PVOID RequestContext,                     // 回调函数的参数
  LPCGUID ControlGuid,                      // 标识 provider 的 GUID
  ULONG GuidCount,                          // TraceGuidReg 数组的大小
  PTRACE_GUID_REGISTRATION TraceGuidReg,    // 标识追踪事件 GUID 的数组
  LPCTSTR MofImagePath,                     // NULL
  LPCTSTR MofResourceName,                  // NULL
  PTRACEHANDLE RegistrationHandle           // 返回句柄
);
```

在 provider 中调用 RegisterTraceGuids 成功之后，如果有 controller 启动或停止该 provider, 系统就会调用回调函数 RegisterAddress。在回调函数中，provider 可以通过参数中的请求代码来判断被调用的原因，如果是被启用，那么应该调用 GetTraceLoggerHandle API 来获取 ETW 会话的句柄。下面的清单给出了回调函数 RegisterAddress 的典型写法：

```cpp
ULONG WINAPI MyControlCallback(WMIDPREQUESTCODE RequestCode, PVOID Context, ULONG* Reserved, PVOID Buffer)
{
    if (RequestCode == WMI_ENABLE_EVENTS)
    {
        g_hTrace = GetTraceLoggerHandle(Buffer);    // 取得会话句柄
        g_dwFlags = GetTraceEnableFlags(Buffer);    // 读取控制器设置的启用标志
        g_dwLevel = GetTraceEnableLevel(Buffer);    // 读取控制器设置的启用级别
    }
    else if (RequestCode == WMI_DISABLE_EVENTS)
    {
        g_hTrace = NULL;
    }

    return 0;
}
```

有了 ETW 会话句柄 g_hTrace, 就可以调用 TraceEvent API 来向 ETW 会话输出消息了：

```cpp
ULONG TraceEvent(
  TRACEHANDLE SessionHandle,        // 会话句柄，即上文取得的 g_hTrace
  PEVENT_TRACE_HEADER EventTrace    // 存放追踪信息的内存缓冲区的起始地址
);
```

缓冲区应当以一个 `EVENT_TRACE_HEADER` 结构开始，后面跟追踪事件的具体数据，即有效负载 (payload)，下面的代码显示了如何组织缓冲区、填写头结构并调用 TraceEvent :

![]( {{site.url}}/asset/software-debugging-etw-traceevent.png )

除了 TraceEvent, Windows XP 引入的 TraceMessage 和 TraceMessageVa API 也可以向 ETW 会话发送追踪消息。这三个 API 都是调用了内核的 NtTraceEvent 服务，NtTraceEvent 又调用了 ETW 的内核函数 WmiTraceMessage.


## 控制 ETW 会话

ETW controller 应该调用 StartTrace API 来开始一个 ETW 追踪会话，并取得这个会话的信息和句柄。

```cpp
ULONG StartTrace(
  PTRACEHANDLE SessionHandle,           // 输出会话句柄
  LPCTSTR SessionName,                  // 会话名称
  PEVENT_TRACE_PROPERTIES Properties    // 描述会话选项
);
```

其中 Properties 指向一个 `EVENT_TRACE_PROPERTIES` 结构：

![]( {{site.url}}/asset/software-debugging-event-trace-properties.png )

注意其中的 LogFileMode 用于指定消息的投递模式，分为两种，一种是将 ETW 消息写入到文件中，简称文件模式，ETW 要求文件的后缀名应当是 ETL (Event Tracing Log)。另一种模式是实时地将 ETW 消息投递给 ETW consumer 简称实时模式。

调用 StartTrace 成功后，会话句柄会通过 SessionHandle 返回，接着就可以通过 EnableTrace 来启动 ETW provider, 使其向这个 ETW 会话输出消息:

![]( {{site.url}}/asset/software-debugging-enabletrace.png )

启动会话后，ETW controller 可以通过 ControlTrace API 来查询会话状态或执行其他控制动作：

![]( {{site.url}}/asset/software-debugging-etw-controltrace.png )

下面是 ControlCode 参数的可选项：

![]( {{site.url}}/asset/software-debugging-etw-controlcode.png )

此外，controller 还可以使用 QueryAllTraces API 来查询当前系统内启动的所有 ETW 会话，使用 EnumerateTraceGuids API 枚举系统内注册的所有 ETW 提供器。


## 消耗 ETW 消息

ETW consumer 在接收 ETW 消息前，必须先使用 OpenTrace API 打开一个 ETL 文件(文件模式) 或实时的 ETW 会话(实时模式)，同时注册自己用于接收消息的回调函数：

```cpp
TRACEHANDLE OpenTrace(
  PEVENT_TRACE_LOGFILE Logfile
);
```

参数 LogFile 用来描述要打开的会话和回调函数地址，是一个 `EVENT_TRACE_LOGFILE` 结构，其定义如下：

![]( {{site.url}}/asset/software-debugging-etw-event-trace-logfile.png )

ETW consumer 做好准备后，应当调用 ProcessTrace API 通知系统启动消息递送：

```cpp
ULONG ProcessTrace(
  PTRACEHANDLE HandleArray,     // 会话句柄数组，consumer 可以同时处理多个 ETW 会话
  ULONG HandleCount,
  LPFILETIME StartTime,
  LPFILETIME EndTime
);
```

在调用 ProcessTrace 成功之后，如果指定的 ETW 会话中有输出事件，那么系统便会调用 consumer 注册的回调函数。ETW consumer 最多可以指定三种回调函数：BufferCallback, EventCallback, EventClassCallback. 第一个用于接受关于会话缓冲区的统计信息，后两个用来接收追踪事件：

```cpp
VOID WINAPI EventCallback(PEVENT_TRACE pEvent);
```

默认情况下，系统会把 ProcessTrace 时指定的所有 ETW 会话的所有事件都发给 EventCallback 函数，但是 ETW 允许通过 SetTraceCallback API 来指定按照事件所属的分类来分发给不同的回调函数：

```cpp
ULONG SetTraceCallback(
  LPCGUID pGuid,
  PEVENT_CALLBACK EventCallback
);
```
