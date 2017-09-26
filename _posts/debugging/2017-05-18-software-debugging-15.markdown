---
layout: post
title:  "《软件调试》 学习 15 Windows 的验证机制"
date:   2017-05-18 10:40:30 +0800
categories: debugging
---

 
 

有的时候，即使我们使用了各种测试手段来对程序进行测试，也不能保证能发现所有问题。原因之一是测试时的运行环境和条件不足以将错误触发并暴露出来。举例来说，如果某个程序有轻微的内存泄漏，通常是比较难发现的。再比如，如果一个程序调用系统的 API 时参数使用不当，而且它没有检查系统返回的错误值，于是这个错误就被掩盖了。对于这样的问题，当测试时，我们通常希望系统做严格的检查，发现问题就立刻报告，并且最好能模拟极端和苛刻的运行环境，以便让错误更容易暴露出来。Windows 操作系统的验证机制 (Verifier) 就是为了满足这个需求而设计的。


## 简介

验证机制的主要目标是为了检查被测试软件，或者说是为被测试软件提供一个验证器 (Verifier)。

### 驱动程序验证器

Windows 2000 最先引入了驱动程序验证器 (Driver Verifier)，用于验证各种设备驱动程序和内核模块。

驱动验证器的主体是实现在内核文件 NTOSKRNL.EXE 中的一系列内核函数和全局变量，其名字大多包含 Verifier 字样，或者是以 Vi 和 Vf 开头的。例如用于验证内存池的 `nt!VerifierFreePool`, 用于验证降低 IRQL 操作的 `nt!VerifierKeLowerIrql`。

为了配合驱动验证器工作，Windows 2000 还自带了一个名为 Driver Verifier Manager 的管理程序，名为 Verifier.exe。这个程序在 XP 及以上系统里也有。

### 应用程序验证器

与驱动验证器类似，为了支持应用程序验证，Windows XP 引入了验证应用程序的机制。其实现也分为两部分，一部分是是现在 NTDLL.DLL 中的一系列函数，这些函数都以 AVrf 开头；另一部分是一个名为应用程序验证器 (Microsoft Application Verifier) 的工具包，可以从微软网站免费下载。

应用程序验证器最初是为了测试应用程序与 Windows XP 操作系统的兼容性而设计的，其设计思想是通过监视应用程序与操作系统之间的交互来发现应用程序中隐藏的设计问题，比如内存分配、内核对象使用和 API 调用等。

### WHQL 测试

通过 WHQL (Windows Hardware Quality Labs) 测试是驱动程序取得 Windows 徽标和得到数字签名的必要条件，而通过驱动验证器的各种验证是驱动程序 WHQL 测试的必不可少的内容。


## 驱动验证器的工作原理

驱动验证器的基本设计思想是在驱动程序调用设备驱动接口 (DDI) 函数时，对驱动程序执行各种检查，看其是否符合系统定义的设计要求，特别是 DDK 文档所定义的调用条件和规范。

那么，如何监视驱动程序对 DDI 的调用呢？在每个内核函数处都加入监视代码显然不大现实，工作量太大并且会影响性能，而且不能选择要监视的程序。所以 Windows 实际采用的是通过修改被验证驱动程序的输入地址表 (Import Address Table, 简称 IAT) 来 Hook 驱动程序的 DDI 调用，即通常所说的 IAT Hook 方法。

在验证函数得到调用后，它执行的典型操作如下：
- 更新计数器，或者全局变量；
- 检测调用参数，或者做其他检查，如果检测到异常情况，那么调用 `KeBugCheckEx(DRIVER_VERIFIER_DETECTED_VIOLATION, ...)` 函数，通过蓝屏机制来报告验证失败；
- 如果没有发现问题，那么验证函数会调用原来的函数，并返回原函数的返回值；

下面详细介绍驱动验证器的工作过程。

### 初始化

在 Windows 系统启动的早期，准确说是在执行体进行阶段 0 初始化期间，内存管理器的初始化函数 (MmInitSystem) 会调用驱动验证器的初始化函数，来初始化驱动验证器。

驱动验证器初始化的一个重要工作是创建并初始化一个用来存放被验证驱动程序信息的链表 MiSuspectDriverList, 这个链表也叫作可疑驱动链表。

这个链表的每个节点是一个 `MI_VERIFIER_DRIVER_ENTRY` 结构，用来记录一个被验证的驱动程序。你可以使用系统自带的驱动验证管理器 verifier.exe 来向这个链表中添加或删除节点。

### 挂接验证函数

当系统加载一个内核模块时，它会调用 MiApplyDriverVerifier 函数，该函数的任务是查询将要加载的模块是否在可疑驱动链表中，如果在，则调用 MiEnableVerifier 函数，对它的 IAT 表进行修改。

MiEnableVerifier 函数会通过之前说的 IAT Hook 技术来替换该可疑模块中的待验证函数。系统定义了几个包含被验证函数和验证函数的数组，记录在 MixxxThunks 这样的全局变量中。

### 验证函数的执行过程

成功挂接验证函数后，当被验证驱动程序调用挂接的内核函数时，对应的验证函数就会被调用。例如，当驱动程序调用 ExAllocatePoolWithTag 时，VerifierAllocatePoolWithTag 便会被调用。

VerifierAllocatePoolWithTag 被调用后，首先会从自己的栈上读取本函数的返回地址，这个地址也就是父函数 ExAllocatePoolWithTag(Hook 后) 的地址。接下来它会把这个地址作为参数传给 ViLocateVerifierEnter 函数，后者会在可疑驱动链表中查找到对应的 `MI_VERIFIER_DRIVER_ENTRY` 结构。

如果没有找到该结构，则 VefifierAllocatePoolWithTag 会终止验证，调用原来的被验证函数 ExAllocatePoolWithTag(Hook 前) 函数。

如果找到了该结构，那么它会执行验证动作，包括：
- 对调用 ExAllocatePoolSanityChecks 进行检查；
- 调用 ViInjectResourceFailure 函数判断是否需要模拟分配失败，如果需要，则返回 NULL 来表示分配失败；
- 调用 ExAllocatePoolWithTagPriority 实际分配内存；
- 调用 ViPostPoolAllocation 对分配情况进行记录；

### 报告验证失败

在驱动验证器的验证函数检测到错误情况后，它会通过蓝屏机制进行报告，通常是调用 KeBugCheckEx 函数，并使用以下停止码作为标识：
- `DRIVER_VERIFIER_IOMANAGER_VIOLATION`, 关于 IO 的验证错误；
- `DRIVER_VERIFIER_DMA_VIOLATION`, 关于 DMA 的验证错误；
- `SCSI_VERIFIER_DETECTED_VIOLATION`, 关于 SCSI 的验证错误；
- `DRIVER_VERIFIER_DETECTED_VIOLATION`, 其他验证错误；


## 使用驱动验证器

驱动验证器将要验证的功能分为若干个项目，包括：
- **自动检查**：唯一不可以单独取消的项目，包括是否在合适的 IRQL 级别使用内存、是否有不恰当的切换栈、是否释放包含活动计时器 (Timer) 的内存池、是否在合适的 IRQL 级别获取和释放自旋锁 (spin lock)、驱动程序卸载时是否已经释放了所有资源；
- **特殊内存池**：从特殊内存池为驱动分配内存，系统会监视该内存池的异常状况；
- **强制的 IRQL 检查**：强制让驱动程序使用的分页内存失效，并监视驱动程序是否在错误的 IRQL 级别访问分页内存或持有自旋锁；
- **低资源模拟**：对驱动程序的内存分配请求或其他资源请求，随机地返回失败；
- **内存池追踪 (Pool Tracking)**：记录驱动程序分配的内存，释放时看其是否全部释放；
- **I/O 验证**：从特殊内存池分配 IRP (I/O Request Packet)，并监视驱动程序 I/O 的处理；
- **死锁探测**：监视驱动程序对各种同步资源（自旋锁、互斥量和快速互斥量等）的使用情况；
- **增强的 I/O 验证**：监视驱动程序对 I/O 例程的调用，并对 PnP, 电源 和 WMI 有关的 IRP 进行压力测试；
- **SCSI 验证**：监视 SCSI 小端口驱动程序 (SCSI miniport driver), 如果发现它不恰当地使用 SCSI 端口例程，则报告验证失败；
- **IRP 记录**：监视并记录驱动程序使用 IRP 的情况；
- **驱动滞留探测**：监视驱动程序的 I/O 完成例程和取消例程的执行时间，如果超出限制，则报告验证失败；
- **安全检查**：寻找可能威胁安全的一般错误，比如内核态的函数引用用户态地址等；
- **强制 I/O 请求等待解决**：对驱动程序的 IoCallDriver 调用随机返回 `STATUS_PENDING`，看其是否能正确处理这种需要等待的情况；
- **零散检查**：检查可能导致驱动程序崩溃的典型诱因，比如错误处理已经释放的内存等；

### 启用驱动验证

驱动验证器默认是关闭状态的，使用前首先要用驱动管理程序来启用它。

在运行中输入 verifier.exe 就可以打开驱动验证管理程序：

![]( {{site.url}}/asset/software-debugging-driver-verifier-manager.png )

在管理程序中可以添加或删除要验证的驱动程序，添加后，下次启动操作系统，该驱动程序就会出现在 可疑驱动链表 中。

### 开始验证

启用了驱动验证后，需要重启操作系统，设置才会生效，因为系统只有在启动期间才会初始化可疑驱动链表。

驱动的验证过程是被动的，你必须运行驱动，让驱动去调用系统的函数，验证逻辑才会执行。所以在验证一个驱动程序时应该让它执行起来，并且尽量让更多的执行路径都得到执行。

### 观察验证情况

驱动验证器在验证过程中，会把每个驱动程序的信息记录到可疑驱动链表中对应的 `MI_VERIFIER_DRIVER_ENTRY` 结构中，此外还会把全局信息记录到全局变量 MmVerifierData 中。

你可以通过命令行 `verifier /query` 来查询验证情况：

![]( {{site.url}}/asset/software-debugging-driver-verifier-query.png )


## 应用程序验证器的工作原理

与驱动验证器类似，应用验证器的设计原理也是通过 Hook 应用程序模块的 IAT 表来截取应用程序对 API 的调用，然后验证它是否符合 Windows SDK 所定义的设计规范。

概括说来，应用程序验证器由以下 3 个部分构成：
- **位于 NTDLL 中的支持例程**：这些函数大多以 AVrf 开头，位于系统的 NTDLL.DLL 模块中。NTDLL.DLL 是 Windows 系统中具有特殊意义的一个模块，所有 Windows 程序执行时都依赖该模块，它会被映射到所有 Windows 进程中。位于 NTDLL 中的进程初始化函数和模块加载函数在进程初始化期间会调用 AVrf 系统函数，让应用程序验证器得到执行机会。
- **验证提供器模块**：是安装应用程序验证工具时复制到系统 System32 目录下的多个 DLL 文件，每个 DLL 文件都用于完成某方面的验证任务。每个验证提供器都包含了若干个验证函数和 Trunk 表。目前版本的应用验证工具包含了如下几个提供器模块：verifier.dll(基础模块), vrfcore.dll(基础模块), vfbasics.dll(基本验证项目), vfcompat.dll(兼容性验证项目), vfLuaPriv.dll(用户账号), vfprint.dll(打印 API), vfprintpthelper.dll(打印驱动), vfsvc.dll(系统服务)；
- **应用验证管理器**：用于管理(添加、删除)被验证的应用程序和选择验证项目。名为 appverif.exe ；

### 初始化

在执行一个 Windows 时，Windows 会先在内核中完成进程创建工作，然后在新进程环境下开始加载程序所依赖的动态链接库。负责加载工作的部分通常称为加载器，实际上就是位于 NTDLL 中的一系列以 Ldr 开头的函数。

加载器初始化时，会从注册表中读取当前程序的执行选项：

```
HKEY_LOCAL_MACHINE\SOFTWARE\Microsoft\Windows NT\CurrentVersion\Image File Execution Options\程序名称
```

![]( {{site.url}}/asset/software-debugging-app-verifier-reg-entry.png )

在应用验证管理器中添加了对某个程序的验证之后，该程序就会出现在上述注册表路径中。

关于程序的应用验证选项保存在 VerifierDlls 和 VerifierFlags 这两个键值中，其中 VerifierDlls 记录了需要加载的验证提供器 dll，VerifierFlags 是验证时的选项。

注册表下还有一个 GlobalFlags 标记，这个标记的第 8 位如果为 1，那么就表示这个程序启用了应用验证。

对于启用了引用验证的程序，加载器会调用 LdrpInitializeApplicationVerifierPackage 来初始化验证器。

接着加载器会做许多初始化工作，包括建立当前进程的加载器数据表，加载当前 EXE 静态链接的 DLL，创建堆等。

在加载静态链接 DLL 之前，加载器会先调用 AVrfInitializeVerifier 函数。该函数会加载 verifier.dll, 然后根据注册表中 VerifierDlls 中记录的内容加载验证提供器 dll。

验证提供器 DLL 加载完成后，加载器会执行每个 DLL 中的 DllMain 函数，完成各 DLL 的初始化工作。

AVrf 会把初始化好的验证提供器信息保存到一个链表中，并使用全局变量 AVrfVerifierProvidersList 指向这个链表。

### 挂接 API

加载了所有静态链接的 DLL 之后，加载器会遍历进程中每个模块的 IAT，将其中的输入函数地址字段修改为目标函数的实际地址，完成所谓的动态链接(绑定)。

晚上上述步骤后，加载器会调用 AVrfDllLoadNotification 给应用验证器处理机会。此时，应用验证器会修改 IAT 来实现 API Hook。

### 验证函数的执行过程

应用验证器成功初始化和 Hook API 之后，当被验证的程序再次调用这些 API 时，便会执行验证函数。例如 VirtualFree API 被调用时，会进入到 AVrfVirtualFree 函数中，它会执行以下验证动作：
- 调用 vfbasics!AVrfVirtualFreeSanityChecks 执行健全性检查；
- 调用 vfbasics!VerifierGetAppCallerAddress 函数得到父函数调用地址；
- 调用 vfbasics!AvrfGetTrunkDescriptor 函数得到本函数的原始函数，即 VirtualFree，调用它；

### 报告验证失败

我们故意在程序中两次调用 VirtualFree, 第二次调用时，验证函数会检测到这个错误情况，打印如下信息：

![]( {{site.url}}/asset/software-debugging-app-verifier-failed-info.png )

观察栈回溯信息，可以看到它的执行情况：

![]( {{site.url}}/asset/software-debugging-app-verifier-failed-callstack.png )

### 验证停顿

应用验证器检测到验证失败之后，会中断到调试器，这种情况称为验证停顿(Verifier Stop)。


## 使用应用程序验证器

### 应用验证管理器

应用验证管理器包含在应用验证工具包中，目前可以通过 Windows SDK 来安装：

![]( {{site.url}}/asset/software-debugging-app-verifier-install.png )

appverif.exe 将被安装到 system32 目录中，可以在运行中直接打开：

![]( {{site.url}}/asset/software-debugging-app-verifier-view.png )

### 验证项目

目前版本的验证器提供了 19 个验证项目，分为 6 个类型，下表列出了这些项目：

![]( {{site.url}}/asset/software-debugging-app-verifier-project.png )

### 配置验证停顿

对于可能产生验证停顿的项目，可以配置每个验证停顿的属性。其操作方法是选中一个项目，然后右键，在菜单中选择 Verifier Stop Option，接下来可以在下图所示的对话框中选择验证停顿选项：

![]( {{site.url}}/asset/software-debugging-app-verifier-config-verifier-stop-option.png )


## 本章总结

本章介绍了 Windows 操作系统所提供的验证机制。包括驱动程序验证器和应用程序验证器。

本章介绍的验证机制有时候也被称为运行期验证。它是运行期检查的一种方法。相比于编译期检查，二者最大的区别在于，编译期是静态的检查，通常只能发现静态问题，而运行期检查可以发现运行期间才会体现出来的动态问题。

