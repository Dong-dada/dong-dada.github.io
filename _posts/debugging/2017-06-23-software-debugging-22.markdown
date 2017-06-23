---
layout: post
title:  "《软件调试》 学习 22 WinDBG 用法详解"
date:   2017-06-23 09:48:30 +0800
categories: debugging
---

* TOC
{:toc}


## 工作空间

WinDBG 使用工作空间 (Workspace) 来描述和存储调试项目的属性、参数以及调试器设置等信息，其功能相当于 IDE 中的项目文件。

WinDBG 定义了两种工作空间，一种称为默认的工作空间 (Default Workspace), 另一种称为命名的工作空间 (Named Workspace). 当没有明确使用某个命名的工作空间时，WinDBG 总是使用默认的工作空间。

所谓的命名工作空间，需要在打开 WinDBG 调试程序时，在菜单项中选择 Save workspace As... 选项，然后自己为当前的工作空间进行命名保存：

![]( {{site.url}}/asset/software-debugging-windbg-save-workspace.png )

下次再调试时，可以通过菜单项中的 Open workspace 来打开之前保存的工作空间：

![]( {{site.url}}/asset/software-debugging-windbg-open-workspace.png )


## 命令概览

WinDBG 有三类命令：标准命令、元命令、扩展命令。

### 标准命令

标准命令用来提供适用于所有调试目标的基本调试功能，执行这些命令时不需要加载任何扩展模块。

WinDBG 共实现了 130 多条标准指令，分为 60 多个系列。为了便于记忆，可以归纳为如下 18 个子类：

- 控制调试目标执行，包括恢复运行的 g 系列命令、跟踪执行的 t 系列命令、单步执行的 p 系列命令、追踪监视的 wt 命令；
- 观察和修改通用寄存器的 r 命令，读写 MSR 寄存器的 rdmsr 和 wrmsr, 设置寄存器显示掩码的 rm 命令；
- 读写 IO 端口的 ib/iw/id 和 ob/ow/od 命令；
- 观察、编辑和搜索内存数据的 d 系列命令、e 系列命令和 s 命令；
- 观察栈的 k 系列命令；
- 设置和维护断点的 bp(软件断点), ba(硬件断点) 和管理断点的 bl(列出所有断点), bc/bd/be(清除、禁止和重启断点)命令；
- 显示和控制线程的 ~ 命令；
- 显示进程的 I 命令；
- 评估表达式的 ? 命令和评估 C++ 表达式的 ?? 命令；
- 用于汇编的 a 命令和用于反汇编的 u 命令；
- 用于显示段选择子的 dg 命令；
- 执行命令文件的 $ 命令；
- 设置调试事件处理方式的 sx 系列命令，启用与禁止静默模式的 sq 命令，设置内核选项的 so 命令，设置符号后缀的 ss 命令；
- 显示调试器和调试目标版本的 version 命令，显示调试目标所在系统信息的 vertarget 命令；
- 检查符号的 x 命令；
- 控制和显示源程序的 ls 系列命令；
- 加载调试符号的 ld 命令，搜索相邻符号的 ln 命令和显示模块列表的 lm 命令；
- 结束调试会话的 q 命令，包括用于远程调试的 qq 命令，结束调试会话并分离调试目标的 qd 命令；

在命令编辑框中输入一个 ? 可以显示出主要的标准命令和每个命令的简单介绍。

### 元命令

元命令 (Meta-Command) 用来提供标准命令没有提供的常用调试功能，与标准命令一样，元命令也是内建在调试器引擎或者 WinDBG 程序文件中的。

所有元命令都是以一个点 (.) 开始，所以元命令页称为点命令 (Dot Command)。

元命令可以分为如下几类：
- 显示和设置调试会话和调试器选项，比如用于符号选项的 .symopt, 用于符号路径的 .sympath 和 .symfix, 用于源程序文件的 .srcpath, .srcnoise 和 .srcfix, 用于扩展命令模块路径的 .extpath, 用于匹配扩展命令的 .extmatch, 用于可执行文件的 .exepath, 设置反汇编选项的 .asm, 控制表达式评估器的 .expr 命令；
- 控制调试会话或调试目标。如重新开始的 .restart 命令。放弃用户态调试目标的 .abandon。创建新进程的 .create 命令和附着在存在进程的 .attach 命令。打开转储文件的 .opendump。分离调试目标的 .detach。用于杀掉进程的 .kill；
- 管理扩展命令模块。如加载模块的 .load 命令。卸载模块的 .unloead 命令和 .unloadall 命令。显示已加载模块的 .chain 命令；
- 管理调试器日志文件。如 .logfile(显示信息), logopen(打开文件), .logappend(追加) 和 .logclose (关闭文件)；
- 远程调试命令；
- 控制调试器。如让调试器睡眠一段时间的 .sleep 命令。唤醒调试器的 .wake 命令。启动另一个调试器来调试当前调试器的 .dbgdgb 命令；
- 编写命令程序。包括一系列类似于C语言关键字的命令。如 .if、.else、.elseif、.foreach、.do、.while、.continue、.catch、.break、.leave等；
- 显示或转储调试目标数据。如产生转储文件的 .dump 命令。将原始数据写到文件的 .writemem 命令。显示调试会话时间的 .time 命令。显示线程时间的 .ttime 命令。显示任务列表的 .tlist 命令。以不同格式显示数字的 .format 命令；

输入 .help 命令可以列出所有元命令和每个元命令的简单说明。

### 扩展命令

扩展命令 (Extension Command) 用于实现针对特定调试目标的调试功能。与标准命令和元命令是内建在 WinDBG 程序文件中不同，扩展命令是实现在动态加载的扩展模块中的。

利用 WinDBG 的 SDK, 用户可以自己编写扩展模块和扩展命令。WinDBG 的程序包中包含了常用的扩展命令模块，下表列出了 WINEXT 和 WINXP 目录中的所有扩展命令模块：

![]( {{site.url}}/asset/software-debugging-windbg-ext-command.png )

执行扩展命令时，应该以叹号 (!) 开始，叹号在英文中被称为 bang, 因此扩展命令也被称为 Bang Command, 执行扩展命令的格式是：

```
![扩展模块名].<扩展命令名>[参数]
```


## 用户界面

以下是 WinDBG 的工作窗口：

![]( {{site.url}}/asset/software-debugging-windbg-work-viewer.png )

可以在菜单中选择显示这些窗口，保存工作空间时，窗口布局也会被保存。

### 命令窗口和命令提示符

命令窗口是用户与 WinDBG 交互的最主要接口，它分为信息显示区和命令横条两部分，命令横条中又包含了命令提示符和命令编辑框：

![]( {{site.url}}/asset/software-debugging-windbg-command-window.png )


## 输入和执行命令

### 要点

- 可以在同一行输入多条命令，用分号 (;) 作为分隔符；
- 直接按回车键可以重复上一条命令；
- 按上下方向键可以浏览和选择以前输入过的命令；
- 输入元命令时应该以点 (.) 开头，输入扩展命令时以叹号 (!) 开头；
- 可以使用 Ctrl+break 来终止一个长时间未完成的命令，如果使用的是 KD 或 CDB, 那么应该使用 Ctrl+C；
- 使用 Ctrl+Alt+V 可以开启 WinDBG 的详细输出模式，观察到更多调试信息，再按一次恢复原来的模式；
- 输入 .hh 加上希望了解的命令，可以随时打开帮助文档进行查看；

### 表达式

这一节的内容我不太懂，基本意思是说 WinDBG 里可以输入一些表达式，这些表达式可以在被调试进程的上下文中执行，比如 `?? const_cast<unsigned int>(0xffffffee)` 表示在当前进程上下文中执行一个 C++ 语句。

这个表达式还可以跟 bu 等命令结合起来用，比如：

```
bp `ctest!main.cpp:100`
```

表示在 CTest 模块的 main.cpp 文件的第 100 行加一个断点。

其他的不太了解了，需要用到的话可以参考相关文章。

### 伪寄存器

为了可以方便地引用被调试程序中的数据和寄存器，WinDBG 定义了一系列伪寄存器：

![]( {{site.url}}/asset/software-debugging-windbg-pseudo-register.png )

### 进程和线程限定符

在很多命令前可以加上进程和线程限定符，用来指定这些命令所适用的进程和线程：

![]( {{site.url}}/asset/software-debugging-windbg-process-thread-identifier.png )

例如，可以使用如下命令来显示 0 号线程的寄存器和栈回溯，即使当前是在 1 号线程：

```
0:001> ~0r; ~0k;
```

### 记录到文件

WinDBG 可以把输入的命令和命令的执行结果输出到文件中，可以使用 Edit 菜单中的 Open/Close Log 来启用和关闭日志文件，也可以使用 .logopen, .logclose, .logfile 这几个命令来打开、关闭、显示日志文件；

