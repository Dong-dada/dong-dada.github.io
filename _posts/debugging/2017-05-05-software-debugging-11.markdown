---
layout: post
title:  "《软件调试》 学习 11 错误报告"
date:   2017-05-05 13:20:30 +0800
categories: debugging
---

 
 


从 Windows 3.0 开始，Windows 中就包含了 Dr.Watson 程序，用来收集错误信息并生成错误报告。Windowx XP 引入了自动发送错误报告的能力，可以根据系统设置将错误报告发送到指定的服务器；Windows Vista 进一步加强和完善了错误报告机制，加入了更多的 API (以 Wer 开头)，并正式将错误报告机制定义为 Windows Error Reporting, 简称 WER。因为 XP 和 Vista 的错误报告机制有所不同，所以在 SDK 文档中，将 XP 的错误报告机制称为 WER 1.0, 而 Vista 的则称为 WER 2.0。

## WER 1.0

从宏观来看，WER 采用的是比较典型的 客户端/服务端 (CS) 结构。客户端负责采集、生成和发送错误报告；服务端负责接收、存储、分类和自动寻找解决方案等任务。

### 客户端

WER 1.0 的客户端主要包含有以下几个部分：
- **FAULTREP.DLL** 导出了若干 API, 用于采集、生成错误报告；
- **DWWIN.EXE** WER 客户端主程序，负责显示 WER 风格的 “应用程序错误对话框” 和发送错误报告；
- **DUMPREP.EXE** 用于检查是否有等待发送的错误报告。如果有，则会通过动态加载 FAULTREP.DLL 中的 ReportFault 函数来启动 DWWIN.EXE 程序发送错误报告；
- 修改了的 **KERNEL32.DLL**, WER 修改了 Kernel32.dll 中的 UnhandledExceptionFilter 函数。当应用程序出现未处理异常时，调用 FAULTREP.DLL 中的 ReportFault API。这是 WER 与系统的一个重要接入点；
- 配置界面，用来启用或禁用 WER；

### 报告模式

WER 定义了如下几种报告模式：
- **共享内存模式(Shared memory mode)**：DWWIN 程序直接从发生错误的应用程序 (内存) 中抓取信息并产生故障转储文件。它只适用于应用程序由于未处理异常而崩溃的情况，所以该模式又称为异常模式；
- **清单模式(Manifest mode)**: DWWIN 程序根据清单文件发送错误报告。既适用于应用程序错误 (异常、僵死)，也适用于内核错误。登录用户必须有管理员权限；
- **排队模式(Queued mode)**: 默认情况下，Windows .NET Server 采用该模式。当有错误发生时，仍然会调用 ReportFault API，但不会显示 UI，而是把要处理的任务放入到一个队列中。当管理员登录时，这些 UI 会弹出，并询问管理员是否发送错误报告。

### 传输方式

WER 1.0 支持两种传输方式来发送报告文件：
- **互联网方式(Internet mode)**: 通过互联网直接发送到微软的错误报告服务器；
- **企业方式(Corporate mode)**: 先发送到企业的一个共享文件夹，然后经过企业 IT 部门审查后发送到微软的服务器。


## WER 2.0

WER 2.0 引入了一些新的模块，它们都位于 system32 目录下：
- **Wer.dll**: 新引入的 WRE API 位于这个 dll 中；
- **Wercon.exe**: WER 控制台界面，用来观察错误报告、寻找解决方案及配置 WER 的选项，取代了 WER 1.0 的简单界面；
- **Wercplsupport.dll**: 控制面板模块，允许用户通过控制面板中的 “问题报告和解决方案” 图标启动 WER 控制台；
- **Werdiagcontroller.dll**: WER 诊断控制器；
- **WerFault.exe** 和 **WerFaultSecure.exe**: WER 2.0 错误报告程序；
- **Wermgr.exe**: 负责管理和调度错误报告发送任务，当应用程序使用进程外方式提交报告时，Wermgr.exe 会负责安排错误报告发送工作；
- **Wersvr.dll**: WER 的系统服务，以 SVCHOST 作为宿主进程运行；

### 创建报告

使用 `WerReportCreate` API 来创建一个 WER 报告，其原型如下：

```cpp
HRESULT WINAPI WerReportCreate(
    PCWSTR pwzEventType, WER_REPORT_TYPE repType,
    PWER_REPORT_INFORMATION pReportInformation, HREPORT* phReportHandle);
```

pwzEventType 用于指定事件名称；repType 用于指定报告的类型，pReportInformation 用于指定报告的应用程序名称等信息；如果调用成功，那么将返回一个 HREPORT 句柄，供其他 API 使用。

接下来，可以通过以下 API 来丰富报告的信息：
- WerReportAddDump 产生并加入 DUMP 信息；
- WerReportAddFile 向报告中加入文件；
- WerReportSerParameter 设置参数；
- WerReportSerUIOption 定制 UI 选项；

### 提交报告

准备好错误报告的内容后，可以使用 WerReportSubmit API 来提交报告：

```cpp
HRESULT WINAPI WerReportSubmit(
    HREPORT hReportHandle, WER_CONSENT consent,
    DWORD dwFlags, PWER_SUBMIT_REUSLT pSubmitResult);
```

### 典型应用

使用 WER 的一种情况是应用程序自己调用上面介绍的 API 来提交报告。

使用 WER 的另一种典型情况是当有未处理异常发生时，系统的 WER 服务 (WERSVC) 收到请求之后会启动 WerFault.exe, 然后在这个进程中产生并发送错误报告。