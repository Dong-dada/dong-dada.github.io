---
layout: post
title:  "《软件调试》 学习 17 运行库和运行期检查"
date:   2017-05-22 16:37:30 +0800
categories: debugging
---

* TOC
{:toc}

上一章介绍的编译期检查，分析的是程序的静态特征。为了发现只有在运行期才会暴露出的问题，编译器通常还设计了运行期检查功能，编译器的运行库(Run-Time Library)是支持运行期检查的载体。

## C/C++ 运行库

当编译器在将高级语言编译到低级语言的过程中时，因为高级语言中的某些比较复杂的运算符要对应比较多的低级语言指令，为了防止这样的指令段多次重复出现在目标代码中，编译器通常会将这些指令段封装为函数，然后将高级语言的某些操作翻译为函数调用。比如 VC 编译器通常把 n = f (n 为整形，f 为浮点型)这样的赋值编译为调用 __ftol 函数，把 new 和 delete 翻译为调用 malloc 和 free 函数。

同时，为了增强编程语言的能力，几乎所有编程语言都定义了相配套的函数库或类库，比如 C 标准定义的标准 C 函数、C++ 标准定义的模板库，这些库通常称为支持库(support library)。

对于使用了某一支持库编译的程序来说，支持库使它们运行时的必要条件，这些程序在运行时必须可以以某种方式找到支持库。出于这个原因，支持库有时候也被称为运行库 (Run-Time Library)。以 VC 编译器为例，它同时提供了支持 C 语言的 C 运行时库和支持 C++ 语言的 C++ 标准库。

### C 运行库

下表列出了 VC 所实现的 C 运行库所包含的主要函数：

![]( {{site.url}}/asset/software-debugging-c-runtime.png )

从 VC 编译器的安装目录可以找到 CRT 函数的源程序文件，包括头文件和 .c 文件。

### C++ 标准库

C++ 标准库由三大部分组成，第一部分是 C 标准库，基本上是上表所列出的各个函数；第二部分是 IO 流 (iostream)；第三部分是标准模板库 (Standard Template Library)；

VC 编译器通常把 C++ 编译器所使用的 C 标准库与 C 编译器所使用的 C 运行库一起实现，即 MSVCRT 或 MSVCR80；而把 IO 流和标准模板库单独来实现，即 MSVCP80。


## 链接运行库

为了满足不同情况的需要，编译器的运行库通常有多个版本。编译器通常会将大多数调试支持只放在调试版本的支持库中。例如 VC8 编译器的 C 运行时库发布版为 MSVCR80.DLL, 而调试版为 MSVCR80D.DLL。

### 静态链接和动态链接

为了使编译好的程序可以顺利运行，使用运行库的程序在运行时必须找到库中的函数。实现这一目的有两种方法，一种是静态链接，另一种是动态链接。

简单来说，静态链接就是将程序中所使用的支持库中的函数复制到程序文件中。这样一来，这些支持库的函数的实现就位于程序模块中，成为本模块中的代码。

对于由多个模块组成的大型软件来说，如果属于同一个进程的几个模块都选择了静态链接 C 运行时库，那么在这个进程内，某些 C 运行时库的函数会重复存在于多个模块中。

动态链接是利用动态链接库技术，在程序运行时再动态地加载包含支持函数的动态链接库 (DLL), 并更新程序的 IAT (Import Address Table) 表，使程序可以顺利地调用 DLL 中的支持函数。


## 运行库的初始化和清理

本节讨论运行库如何工作，包括介入到应用程序中的方法，以及如何进行初始化和清理。

### 介入方法

因为很多运行库函数和设施是运行其它代码的基础，比如 CRT 堆必须在其它代码调用 CRT 的内存分配函数之前被初始化好。这意味着，必须在执行用户代码之前初始化运行时库。

编译器会为每个模块自动插入一个 “编译器编写” 的入口函数，在这个入口函数中进行好各种初始化工作后再调用用户的入口函数，在用户的入口函数返回后再运行自己的清理函数。我们把编译器插入的这个入口函数称为 CRT 入口函数。

![]( {{site.url}}/asset/software-debugging-enter-function.png )

DLL 模块的入口函数是 DllMain。

在链接阶段，模块的入口函数是被注册到目标程序文件 (PE) 文件中的，默认情况下，链接器会将 CRT 入口函数注册为模块的入口，这样模块执行时，首先执行的是 CRT 的入口函数，它可以完成运行库的初始化工作后再调用用户的入口函数。等用户的入口函数返回后，CRT 的入口函数可以执行清理工作。

VC 的链接器选项中可以自行指定入口函数名，如果进行了这项设置，就需要自己来考虑如何初始化和清理运行库了。

### 初始化

在 CRT 源代码的 crt0.c 文件中包含了 CRT 入口函数的源代码，概括说来，CRT 入口函数执行的初始化工作主要有如下几项：
- 调用 `__security_init_cookie()` 初始化安全 Cookie；
- 初始化全局变量，包括环境变量和表示操作系统版本号的全局变量等；
- 调用 `_heap_init()` 函数创建 C 运行库所使用的堆，简称 CRT 堆，程序中调用 malloc 等函数所获得的内存便是从这个堆中分配的；
- 调用 `_mtinit()` 函数初始化多线程支持；
- 如果使用 VC7 引入的运行期检查功能，那么需要调用 `_RTC_Initialize()`；
- 调用 `_ioinit()` 函数初始化低级 IO；
- 调用 `_cinit()` 函数初始化 C 和 C++ 数据，包括初始化浮点计算包、除 0 向量，以及调用 `_initterm(__xi_a, __xi_z)` 执行注册在 PE 文件数据区的初始化函数，调用 `_initterm(__xc_a, __xc_z)` 执行 C++ 初始化函数，例如全局 C++ 对象的构造函数；

### 多个运行库实例

对于静态链接运行库的程序模块，不论是 EXE 还是 DLL，每个模块内部都复制了一份运行库的变量和代码。也就是说，每个模块中都有一个运行库的实例。

当进程中有多个运行库实例时，每个运行库实例会各自使用自己的数据和资源 (堆)，理论上是可以正常工作的。但有时也可能会引发问题，其中之一就是从一个 CRT 堆分配的内存被送给另一个 CRT 实例来释放，在调试版本中，CRT 的内存检查功能会发现这一问题并报告错误，但在发布版本中，这有可能造成严重的问题。


## 运行期检查

运行期检查的目的是发现程序运行期间所暴露出的各种错误，即运行期错误。因为程序运行是编译过程结束后才发生的事情，所以要实现运行期检查，编译器通常采取如下几种措施：
- 使用调试版本的支持库和库函数，调试版本的库函数中包含了更多的调试支持和检查功能。比如调试版本的内存分配和释放函数会插入额外的信息来支持各种内存检查功能；
- 在编译时插入额外代码对栈指针、局部变量等进行检查；
- 提供断言 (ASSERT)、报告 (_RPT) 等机制让程序员在编写程序时加入检查代码和报告运行期错误；

### 自动的运行期检查

VC8 编译器可以自动检查如下运行期错误；
- 栈指针被破坏 (Stack pointer corruption)，负责栈指针检查的函数是 `_RTC_CheckEsp`；
- 分配在栈上的局部变量越界 (Overruns)，以及因此导致的栈被破坏 (Stack corruption)；
- 依赖未初始化过的局部变量，负责该功能的 RTC 函数是 `_RTC_UninitUse`；
- 因为赋值给较短的变量导致数据丢失，负责该项检查的函数式 `_RTC_Check_x_to_y`，其中 x 和 y 可以是 8,4,2,1 这几个数字的组合，例如 `_RTC_Check_8_to_4(int64)`；
- 使用堆有关的错误；

### 断言 (ASSERT)

断言是程序员手工插入运行期检查的一种常用方法。通常用来检查某一条件是否成立。如果指定的条件没有成立，那么便会弹出如下的断言错误框：

![]( {{site.url}}/asset/software-debugging-assert-dialog.png )

VC 运行库提供了两个宏来提供断言功能，分别是 `_ASSERT` 和 `_ASSERTE`, 后者会将断言表达式也输出到窗口中。在头文件 crtdbg.h 中可以找到它们的定义：

```cpp
#define _ASSERT_EXPR(expr, msg) \
        (void) ((!!(expr)) || \
                (1 != _CrtDbgReportW(_CRT_ASSERT, _CRT_WIDE(__FILE__), __LINE__, NULL, msg)) || \
                (_CrtDbgBreak(), 0))

#ifndef _ASSERT
#define _ASSERT(expr)   _ASSERT_EXPR((expr), NULL)
#endif

#ifndef _ASSERTE
#define _ASSERTE(expr)  _ASSERT_EXPR((expr), _CRT_WIDE(#expr))
#endif
```

可以看到它们实际上都调用了 `_CrtDbgReportW` 这个函数，它会把事件信息输出到调试文件、调试器的监视窗口、或者弹出一个失败窗口。

标准 C 语言也提供了一个名为 assert 的宏来实现断言，使用这个宏的好处是代码具有更好的可移植性。

ASSERT 宏只在调试版本中存在，发布版本中是不存在的，因此不能用它来代替常规错误检查。

### _RPT 宏

在 VC8 宏中，可以通过 `_RPT` 系列宏来调用 `_CrtDbgReportW` 函数报告调试信息，它包含以下两个系列：

一个系列是 `_RPT0`, `_RPT1`, `_RPT2`, `_RPT3`, `_RPT4` 简称 `_RPTn`，它们都具有如下原型：

```
_RPTn( reportType, format, ...[args]);
```

其中，reportType 可以是 `_CRT_WARN`, `_CRT_ERROR`, `_CRT_ASSERT` 三个常量之一。

另一个系列是 `_RPTF0`, `_RPTF1`, `_RPTF2`, `_RPTF3`, `_RPTF4`, 简称 `_RPTFn`，它跟 `_RPTn` 系列宏的唯一差别是 `_RPTFn` 中包含了源文件名和行号。


## 报告运行期检查错误

本节将介绍 CRT 报告运行期检查错误的方法，包括 `_CrtDbgReport` 函数、如何配置报告方式、以及如何编写回调函数参与报告过程。

## _CrtDbgReport

`_CrtDbgReport` 是报告运行期检查的一个主要函数，断言失败和 `_RPT` 宏都是通过 `_CrtDbgReport` 来报告调试信息的。它的函数原型为：

```cpp
_CRTIMP int __cdecl _CrtDbgReportW(
        _In_ int _ReportType,
        _In_opt_z_ const wchar_t * _Filename,
        _In_ int _LineNumber,
        _In_opt_z_ const wchar_t * _ModuleName,
        _In_opt_z_ const wchar_t * _Format,
        ...);
```

第一个参数是报告类型，可以是 `_CRT_WARN`, `_CRT_ERROR`, `_CRT_ASSERT` 三个常量之一。`_Filename` 是要报告的文件名，`_LineNumber` 是行号，`_ModuleName` 是模块名，`_Format` 是格式化字符串，与 printf 的 format 参数一样。

`_CrtDbgReport` 函数的返回值逻辑是：

![]( {{site.url}}/asset/software-debugging-crtdbgreport.png )

CRT 的源文件目录中包含了 `_CrtDbgReport` 函数的实现。下述栈回溯显示了 CRT 的变量检查函数发现错误并调用错误报告函数的执行过程：

![]( {{site.url}}/asset/software-debugging-crtdbgreport-stack.png )

其中 `failwithmessage` 函数是 VC8 的 RTC 系列函数用来报告错误的枢纽。

`failwithmessage` 会首先检查当前程序是否是在 IDE 的集成调试器中执行的，其判断法方法是使用 RaiseException API 抛出一条代码为 406D1388h 的异常，如果 IDE 存在，它会处理这个异常，使 `failwithmessage` 函数收到 1，否则收到 0。

接着 `failwithmessage` 会调用 `DebuggerRuntime` 函数，如果当前是在 IDE 环境中，它会再次抛出 406D1388h 异常，这时候调试器会显示对应的 RTC 错误对话框；

如果不是在 IDE 环境中，`failwithmessage` 会调用 `IsDebuggerPresent` API 来判断是否在其他调试器中运行，如果是，那么它调用 `DbgBreakPoint` 触发断点将当前程序中断到调试器中；

否则便向上述清单显示的那样弹出错误对话框。

### _CrtSetReportMode

`_CrtDbgReport` 可以把调试信息送往三个目的地：调试文件，调试器的监视窗口和调试消息对话框。

可以调用 `_CrtSetReportMode` 来配置 CRT 的报告方式，默认是调试消息对话框。

### _CrtSetReportHook

可以通过 `_CrtSetReportHook` 函数来设置一个回调函数，每当 `_CrtDbgReport` 被调用时，都会先回调这个函数。