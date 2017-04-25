---
layout: post
title:  "《软件调试》 学习 10 硬错误和蓝屏"
date:   2017-04-25 12:10:30 +0800
categories: debugging
---

* TOC
{:toc}

一套好的错误处理方案通常应该考虑以下三个方面：
- 即时提示(instant nofitication) : 当错误发生时立刻提示给用户；
- 永久记录(persitent recording) : 将错误永久记录在文件或数据库中供事后分析；
- 自动报告(automatic reporting) : 自动收集错误现场的详细情况并生成错误报告，用户可以通过简单的方式发送到专门用来收集错误报告的服务器。从 Windows XP 开始，Windows 引入了一整套设施来实现这一目标，称为 Windows Error Reporting, 简称为 WER；

本章先介绍用于报告严重错误的硬错误机制 (HardError) 和蓝屏机制 (BSOD)。然后介绍系统转储文件的产生方法和分析方法。介绍声音和闪动窗口等辅助的错误提示机制，介绍配置错误提示机制。介绍使用错误提示机制时应该注意的问题。


## 硬错误提示

硬错误 (HardError) 的本意是指与硬件有关的严重错误，是与重新启动便可以恢复的软件错误 (SoftError) 相对而言的。逐渐地，这个词被用来泛指比较严重的错误情况。

例如下面的缺盘错误对话框，它是在光盘上的程序正在运行时把光盘取出时发生的，该对话框就是使用 HardError 提示机制弹出的：

![]( {{ site.url }}/asset/software-debugging-harderror-example.png )

尽管这个对话框是使用 MessageBox API 弹出的，但是调用 MessageBox API 只是硬错误提示机制的一部分，此前还经历了复杂的发起和分发过程。

硬错误提示既可以在用户态，也可以在内核态使用。在用户态使用的方法是调用 NtRaiseHardError 内核服务。

```cpp
NTSYSAPI NTSTATUS NTAPI NtRaiseHardError(
    NTSTATUS ErrorStatus,
    ULONG NumberOfParameters,
    ULONG UnicodeStringParametersMask,
    PVOID* Parameters,
    HARDERROR_RESPONSE_OPTION ResponseOption,
    PHARDERROR_RESPONSE Response);
```

ErrorStatus 用来传递错误代码，通常是定义在 NTSTATUS.H 中的常量；NumberOfParameters 指定 Parameters 数组中的元素个数；UnicodeStringParametersMask 参数的各个二进制位与 Parameters 数组一一对应，如果某一位为 1, 则说明对应的参数指针指向的是一个 `UNICODE_STRING`, 否则是个整数；ResponseOption 用来定义错误消息的按钮个数、响应方式等选项；Response 参数用来返回错误提示的响应结果，这个结果可以是用户选择的，也可以是因为超时而导致的默认值，或者是处理失败，它的值为如下常量之一：

```cpp
typedef enum _HARDERROR_RESPONSE {
    ResponseReturnToCaller, 
    ResponseNotHandled,
    ResponseAbort,
    ResponseCancel,
    ResponseIgnore,
    ResponseNo,
    ResponseOk,
    ResponseRetry,
    ResponseYes} HARDERROR_RESPONSE, *PHARDERROR_RESPONSE;
```

NtRaiseHardError 的工作主要是参数检查和预处理，在它们把所有错误信息都复制到一个用户态可访问的内存结构中后便调用 ExpRaiseHardError.

### ExpRaiseHardError

