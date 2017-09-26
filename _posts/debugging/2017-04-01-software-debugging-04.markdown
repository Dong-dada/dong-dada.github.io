---
layout: post
title:  "《软件调试》 学习 04 分支记录和性能监视"
date:   2017-04-01 14:08:30 +0800
categories: debugging
---

 
 

沿着正确的轨迹执行是软件正常工作的必要条件。很多软件错误都是因为程序运行到了错误的分之。尽管这通常不是错误的根本原因，但却是可以顺藤摸瓜的直接线索。因此，了解软件的运行轨迹对于寻找软件问题的根源有着重要意义。因为很多性能问题是因为执行了不必要的代码或循环导致的，所以运行轨迹对于分析软件的运行效率和软件调优也有着重要意义。

CPU 的设计者们早就想到了这一点。CPU 提供了监视和记录分支(branch)、中断(interrupt)、异常(exception)事件的功能，简称为分支监视和记录。奔腾 4 处理器对这一功能又做了很大的增强，允许将分支信息记录到内存中一块被称为 BTS(Branch Trace Buffer)的缓存区中，称为调试存储机制(Debug Store)。

## 分支监视概览

分支监视是跟踪和记录 CPU 执行路线 (history) 的基本措施，对软件优化和软件调试都有着至关重要的作用。

由于 CPU 集成了高速缓存，CPU 会成批地将代码读入高速缓存中，然后再从高速缓存中读取指令并解码执行。这样的话你就没办法通过前端总线上的调试工具来观察 CPU 执行的细节，也就无法观察到 CPU 的执行轨迹。

为了解决上述问题，奔腾处理器引入了一种专门的总线事务(bus transaction), 称为 Branch Trace Message(分支踪迹消息)，简称为 BTM。当你启用 BTM 之后，CPU 每次执行分支和改变执行路线时，CPU 都会发起一个 BTM 事务，将分支消息发送到前端总线上。这样，调试工具便可以通过监听和读取 BTM 消息跟踪 CPU 的执行路线了。

收到 BTM 消息之后，CPU 需要一个地方来记录分支信息，P6 处理器中引入了 MSR 寄存器来记录这些信息，奔腾 4 处理器引入了通过特定的内存区域来记录分支信息的功能。

### 使用寄存器的分支记录

P6 处理器中引入了 MSR 寄存器来记录最近一次分支的源和目标地址，称为 Last Branch Recording，奔腾 4 处理器对其做了增强，增加了寄存器的个数，以栈的方式可以保存多个分支记录，称为 LBR 栈(LBR Stack)。

P6 处理器中包含了 5 个 MSR(Machine/Model Specific Registers) 寄存器用来实现 LBR 机制：
- 用来记录分支的 LastBranchToIP 和 LastBranchFromIP 寄存器对；
- 用来记录异常的 LastExceptionToIP 和 LastExceptionFromIP 寄存器对；
- 一个 MSR 寄存器用来控制新加入的调试功能，称为 DebugCtrl；

当发生分支时，LastBranchFromIP 用来记录分支指令的地址，LastBranchToIP 用来记录这个分支要转移到达的目标地址；

当异常(调试异常除外)或中断发生时，CPU 会先把 LastBranchToIP 和 LastBranchFromIP 中的内容分别复制到 LastExceptionToIP 和 LastExceptionFromIP 中，然后再把发生异常或中断时被打断的地址更新到 LastBranchFromIP, 再把异常或中断处理程序的地址更新到 LastBranchToIP 寄存器中。

如此一来，异常发生后，LastExceptionToIP/LastExceptionFromIP 中记录了异常发生前最后一个分支的信息，LastBranchToIP/LastBranchFromIP 中则记录了异常发生时正在执行的指令信息。

上述机制只能记录最近一次的分之和异常，奔腾 4 处理器对其做了增强，引入了 “最近分支记录堆栈”(Last Branch Record Stack)，简称 LBR 栈，可以记录 4 次或更多次的分支和异常。

下图是原书中的一个例子，展示了异常发生后 LBR 栈的信息：

![]( {{ site.url }}/asset/software-debugging-lbr-stack.png )

### 使用内存的分支记录

MSR 寄存器所能记录的分支次数比较少，因此奔腾 4 中引入了一个新的机制，可以讲分支和异常信息记录到内存中。这便是 分支踪迹存储(Branch Trace Store)机制，简称为 BTS。

BTS 允许把分支记录保存在一个特定的被称为 BTS 缓冲区的内存区内。BTS 缓冲区与用于记录性能监视信息的 PEBS 缓冲区是使用类似的机制来管理的，这种机制被称为调试存储区(Debug Store)，简称为 DS 缓冲区。

PEBS 的全称是 Percise Event-Based Sampling, 即精确的基于事件采样，是奔腾 4 处理器引入的一种性能监控机制。你可以将某个性能计数器设置为触发 PEBS 功能，当这个计数器溢出时，CPU 便会把当时的寄存器状态以 PEBS 记录的形式保存到 DS 存储区中的 PEBS 缓冲区内。稍后我们会详细讨论性能监视功能。

DS 存储区中包含了三个部分：
- **管理信息区**：用来定义 BTS 和 PEBS 缓冲区的位置和容量。管理信息区的功能与文件头很类似，CPU 通过查询管理信息区来管理 BTS 和 PEBS 缓冲区；
- **BTS 缓冲区**：用来以线性表的形式存储 BTS 记录。每个 BTS 记录的长度固定位 12 个字节，分成 3 个双字，第一个 DWORD 是分支的源地址，第二个 DWORD 是分支的目标地址，第三个 DWORD 从来表示该记录是否是预测出的；
- **PEBS 缓冲区**：用来以线性表的形式存储 PEBS 记录。每个 PEBS 记录的长度固定为 40 个字节；


## 性能监视

这一章有些东西没看明白，这里先略过吧。。。

