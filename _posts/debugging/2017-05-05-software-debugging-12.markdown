---
layout: post
title:  "《软件调试》 学习 12 日志"
date:   2017-05-05 13:20:30 +0800
categories: debugging
---

 
 

概括来说，Windows 操作系统建立了两套日志机制。一套是 Windows Vista 引入的名为 CLFS (Common Log File System) 的机制，另一套是从 NT 3.5 就支持的 Event Logging 机制，因为其内部函数大多数是以 Elf (Event log file) 开头的，所以我们将其简称为 ELF。

简单来说，日志就是软件自己写的日记，每一条日志用于记录一件事。基于这个基本原则，一条日志通常包含如下几个要素：
- 时间：记录事件的发生时间；
- 地点：通常包含机器名、进程 ID、线程 ID 等；
- 来源：即该事件的实施者，根据需要可以是服务名称、模块名称、类名或函数名。
- 事件：对所发生事件的描述；
- 严重程度：可以分为 Information, Warning, Error 三种；

接下来我们先介绍 ELF 日志，然后介绍 Vista  引入的 CLFS 日志。


## ELF 日志

![]( {{site.url}}/asset/software-debugging-elf-architecture.png )

左边的是调用 ELF 的应用程序，中间是承担日志服务的服务进程，右边是用来存储日志记录的日志文件。

### ELF 的日志文件

ELF 的日志文件存储在磁盘上，每一类事件放在一个文件中。以 Windows XP 为例，系统共定义了 3 类日志，分别是应用程序 (Application) 日志、安全 (Security) 日志、系统 (System) 日志，它们的文件分别是 AppEvent.Evt, SecEvent.Evt, SysEvent.Evt。

Windows Vista 增加了 HardWareEvent 和 DFS Replication 等日志类型，并且将后缀名改为了 .EVTX 。

每一类日志的配置都记录在以下注册表键下：

```
HKEY_LOCAL_MACHINE\SYSTEM\CurrentControlSet\Services\EventLog
```

如下图所示：

![]( {{site.url}}/asset/software-debugging-elf-register.png )

注意，Application 键下有许多子键，这些子键就是该日志类型的事件源。事件源用来标识日志记录的来源(报告者)，ELF 在显示日志的时候会根据事件源来格式化日志信息。

### ELF 服务

在 Windows 默认启动的服务中包含了负责事件日志的服务，其名称是 Event Log, 我们将其简称为 ELF 服务。ELF 服务的职责是管理和维护事件日志文件，并通过 RPC 机制向应用程序提供各种日志服务，包括添加删除日志记录，获取日志信息，备份日志文件等。


## ELF 的数据组织

在理解 ELF 组织数据的方法之前，有必要先思考一个问题。作为一种通用的日志记录机制，必须要考虑到不同的事件需要记录的信息量和格式可能是大相径庭的，简单的可能只是一句话，复杂的可能包含多个文本，多个数值和数据附件。

那么如何通过统一的方法来组织和存储不同信息量和不同结构的事件记录呢？有以下两种方法：
- 方法 A ： 将所有不同结构的事件格式化为字符串后统一存储；
- 方法 B ： 将每个事件的格式信息抽象出来单独存储，在日志文件中只记录每个事件实例的具体数据。也就是将事件的格式和实际数据分别存储，两者通过一个 ID 联系起来。

方法 A 比较简单，但会产生许多冗余数据，ELF 采用的是方法 B，事件的格式信息存储在消息文件中 (每个事件源都对应有消息文件)，日志文件只存储每个事件描述中的变化部分。

### 日志记录

ELF 以表格的形式来存储日志记录，表格的每一行对应一条日志记录，存储着这条记录的基本信息和附属信息的偏移地址。可以使用以下结构来描述 ELF 文件中的数据表结构：

![]( {{site.url}}/asset/software-debugging-elflogrecord.png )

### 添加日志记录

1. 应该在注册表中注册事件源，方法是调用注册表 API 并按照指定格式来添加注册表键值；
2. 调用 ELF 的 RegisterEventSource API 来取得事件源句柄；
3. 调用 ELF 的 ReportEvent API 来添加日志记录；

ReportEvent API 会通过 RPC 机制将调用转发给 ELF 服务。

### API 一览

除了增加日志，ELF 还提供了以下几个 API 用来读取、查询、清除日志记录：

![]( {{site.url}}/asset/software-debugging-elf-api.png )


## 查看和使用 ELF 日志

Windows 自带了事件查看器 (Event Viewer) 来查看和管理事件日志记录。你可在运行对话框中输入 eventvwr.exe 来打开它：

![]( {{site.url}}/asset/software-debugging-elf-viewer.png )


## CLFS 的组成和原理

CLFS 是 Common Log File System 的简称，它是 Windows Vista 新引入的一种日志机制。与 ELF 相比，CLFS 更快，可靠性更高。

### 组成

CLFS 主要实现在两个模块中，一个是位于内核态的 CLFS.SYS，另一个是位于用户态的 CLFSW32.DLL。下图画出了这两个模块的位置和相互关系：

![]( {{site.url}}/asset/software-debugging-clfs.png )

CLFS.SYS 中实现了 CLFS 的核心功能。它与系统内核文件 NTOSKRNL.EXE 有着直接的相互依赖关系，因此是随着内核文件一起由系统的加载程序在系统启动的早期加载到内存中的。 CLFS.SYS 以 DLL 方式输出了一系列函数，都是以 Clfs 开头，例如 ClfsCreateLogFile, ClfsAddLogContainer 等，内核模块可以通过 DLL 方式直接调用 CLFS 的这些函数。

CLFSW32.DLL 是一个简单的用户态 DLL，它的主要作用是向用户态程序输出 CLFS 的应用程序编程接口 (CLFS API)。它输出两类 API，一种是供管理工具来定义 CLFS 管理策略和注册回调函数参与 CLFS 事务；另一种是供各种应用程序使用 CLFS 的日志服务。

大多数 API 都是通过 DeviceIoControl 来调用内核态的 CLFS.SYS 公开函数，只有少量函数实现在 CLFSW32.DLL 中。

### 存储结构

为了具有高可靠性和好的性能，CLFS 的日志记录与它在物理介质 (磁盘) 中的存储结构有着直接的映像关系。

首先，一个 CLFS 日志 (Log) 是有一个 BLF 文件和多个容器文件 (Container) 组成的：

![]( {{site.url}}/asset/software-debugging-clfs-storage-architecture.png )

BLF 文件用来存储日志的全局信息，其大小通常为 64KB；容器文件是真正用来存储日志数据的，整个文件对应于物理介质上的一段连续存储空间，称为 Extent，其大小一定是存储介质的分配单位的整数倍。以硬盘为例，一个容器文件对应于磁盘上一系列连续的磁盘扇区，其大小是扇区大小 (512 字节) 的整数倍。这样做的好处是使用原始的磁盘读写方法就可以将日志数据写到磁盘上，即使系统崩溃或文件系统错误也能将日志保存下来。

BLF 文件是通过 ClfsCreateLogFile 函数或 CreateLogFile API 来创建的；容器文件是通过 ClfsAddLogContainer 函数或 AddLogContainer API 来创建的。

### LSN

CLFS 使用一个 64 位的整数(ULONGLONG) 来唯一标示一条日志记录，称为日志序列号 (Log Sequence No)，简称 LSN，一个 LSN 由以下三部分组成：
- 容器标识， LSN 的高 32 位；
- IO 块偏移，LSN 的 9 到 32 位；
- 日志记录在 IO 块内的属性号，位于 LSN 的低 9 位；

CLFS.SYS 中提供了一系列函数来操纵 LSN，比如可以调用 ClfsLsnCreate 将以上三部分合成一个 LSN，使用 ClfsLsnContainer, ClfsLsnBlockOffset, ClsfLsnRecordSequence 来获取各个部分。


## CLFS 的使用方法

### 创建日志文件

使用 CreateLogFile API 来创建或打开一个 CLFS 日志。例如 ：

```cpp
hClfsLog = CreateLogFile(szLogFile, GENERIC_WRITE | GENERIC_READ, 0, NULL, OPEN_ALWAYS, 0);
```

其中 szLogFile 用来指定要打开的 CLFS 日志名称，其格式为 `LOG:<log name>[::<log stream name>]` 其中 log stream name 是用在创建复合日志的情况下的。

### 添加 CLFS 容器

添加容器的方法有两种，一种是使用 CLFS 的 API AddLogContainer 或 AddLogContainerSet, 另一种是使用 CLFS 的管理 API, SetLogFileSizeWithPolicy，例如：

```cpp
lpszContainerPath = _T("%BLF%\\CLFSCON01");
if (!AddLogContainer(hClfsLog, &cbContainer, lpszContainerPath, NULL))
{
    // ...
}
```

```cpp
ULONGLONG ullContaners = 2;
ULONGLONG ullResultingSize = 0;
if (!SetLogFileSizeWithPolicy(hClfsLog, %ullContainers, &ullResultingSize))
{
    // ...
}
```

### 创建编组区

在向 CLFS 日志写入记录前，还需要创建一个特殊的内存缓冲区，称为编组区 (Marshalling Area)。它的主要作用是缓存多条日志记录，以减少访问外部存储器（磁盘）的次数：

```cpp
if (!CreateLogMarshallingArea(hClfsLog, NULL, NULL, NULL, 512, 2, 2, &pMarshalContext))
{
    // ...
}
```

以上调用成功后，系统会将一个上下文结构的地址存储到 pMarshalContext 参数中，有了这个结构之后，就可以写入日志数据了。

### 添加日志记录

添加日志需要调用 ReserveAndAppendLog API, 调用前应当先把要写入的数据登记到 `CLFS_WRITE_ENTRY` 结构中，这个 API 允许一次性写入多条记录：

```cpp
_sntprintf(szLogBuffer, MAX_PATH, _T("A testing log record at tick 0x%x", GetTickCount()));

ClfsEntry.Buffer = szLogBuffer;
ClfsLSN = CLFS_LSN_INVALID;
ClfsEntry.ByteLength = (_tcslen(szLogBuffer) + 1) * sizeof(TCHAR);

if (!ReserveAndAppendLog(
    pMarshalContext,
    &ClfsEntry,
    1, 
    &ClfsLSN,
    &ClfsLSN,
    0,
    0,
    CLFS_FLAG_NO_FLAGS,
    &ClfsLSN,
    NULL))
{
    // ...
}
```

如果写入成功，将会把这条日志的 LSN 返回到 ClfsLSN 中。

结束所有日志操作之后，应当调用 CloseHandle API 来关闭日志。

### 读日志记录

读 CLFS 日志前也应当先打开日志文件和创建编组区，然后调用 ReadLogRecord 来开始读取日志记录：

```cpp
if (!ReadLogRecord(
    pMarshalContext, 
    &(li.BaseLsn), 
    ClfsContextForward, 
    &pReadBuffer, 
    &ulSize, 
    &ulRecordType, 
    &lsnUndoNext, 
    &lsnPrevious, 
    &ReadContext, 
    NULL))
{
    // ...
}
```

接下来可以循环调用 ReadNextLogRecord API 来遍历当前日志：

```cpp
while(ReadNextLogRecord(
    pReadContext, 
    &pReadBuffer, 
    &ulSize, 
    &ulRecordType, 
    NULL, 
    &lsnUndoNext, 
    &lsnPrevious, 
    &lsnRecord, 
    NULL));
{
    // ...
}
```

### 查询信息

可以调用 GetLogFileInformation API 来查询日志的详细信息：

```cpp
CLFS_INFORMATION li;
ULONG ulSize = sizeof(li);

if(!GetLogFileInformation(hClfsLog, &li, &ulSize))
{
    // ...
}
```