---
layout: post
title:  "利用 WinDBG 调试死锁"
date:   2017-07-27 14:47:30 +0800
categories: windows
---

 
 


## 什么是死锁

简单来说，死锁就是 A 等待 B 持有的锁，B 等待 A 持有的锁，这样形成循环，导致两个线程都被卡住的现象；

这是一种循环，也可能是 A 等 B, B 等 C, C 等 A 这种，不一定是两个线程或两把锁的事儿；

下面是我写的一个简单示例代码：

```c
#include "stdafx.h"
#include <process.h>

CRITICAL_SECTION g_mutex1;
CRITICAL_SECTION g_mutex2;
CRITICAL_SECTION g_mutex3;

void __cdecl ThreadFunc(void* param)
{
	::EnterCriticalSection(&g_mutex2);
	printf("thread enter mutex2\n");
	::Sleep(500);       // 让主线程有机会获取到 g_mutex1

	::EnterCriticalSection(&g_mutex1);
	printf("thread enter mutex1\n");

	::LeaveCriticalSection(&g_mutex1);
	::LeaveCriticalSection(&g_mutex2);
}

void __cdecl OtherThreadFunc(void* param)
{
	::EnterCriticalSection(&g_mutex3);
	printf("thread enter mutex3\n");
}

int main()
{
	::InitializeCriticalSection(&g_mutex1);
	::InitializeCriticalSection(&g_mutex2);
	::InitializeCriticalSection(&g_mutex3);

	::EnterCriticalSection(&g_mutex3);
	printf("main enter mutex3\n");

	_beginthread(ThreadFunc, 0, nullptr);
	_beginthread(OtherThreadFunc, 0, nullptr);

	::EnterCriticalSection(&g_mutex1);
	printf("main enter mutex1\n");
	::Sleep(1000);      // 让子线程有机会获取到 g_mutex2

	::EnterCriticalSection(&g_mutex2);
	printf("main enter mutex2\n");

	::LeaveCriticalSection(&g_mutex3);
	::LeaveCriticalSection(&g_mutex2);
	::LeaveCriticalSection(&g_mutex1);

	::DeleteCriticalSection(&g_mutex1);
	::DeleteCriticalSection(&g_mutex2);

	return 0;
}
```

上述代码中共有三个线程，其中 `main()` 所在线程先持有 `g_mutex1` 锁，然后 `ThreadFunc()` 所在线程持有 `g_mutex2` 锁。接着 `main` 线程等待 `ThreadFunc` 持有的 `g_mutex2` 锁，`ThreadFunc` 线程等待 `main` 线程持有的 `g_mutex1` 锁，两者发生了死锁。

上述代码中的 `OtherThreadFunc()` 及 `g_mutex3` 是干扰项，实际的项目中不可能只有两个线程在等待锁，因此额外加一个 `g_mutex3` 来模拟这种情况。`OtherThreadFunc()` 等待的锁是由 `main()` 持有的，`main()` 因为死锁而卡住，因此 `OtherThreadFunc()` 也会被卡住。

它们的关系如下图：

![]( {{site.url}}/asset/debugging-dead-lock-example.png )


## 用 WinDBG 分析死锁

运行上述代码，然后用 WinDBG Attach 上去。先通过 `~*` 命令查看一下当前的线程：

![]( {{site.url}}/asset/debugging-dead-lock-threads.png )

可以看到总共有 4 个线程，最后的那个 3 号线程是 WinDBG 附加到进程时产生的远程线程，不用理它，也就是说现在进程里运行的线程有 3 个，分别是 `main`, `ThreadFunc`, `OtherThreadFunc`。其中 0 号线程是 `main()` 所在的线程。

实际情况中不可能只有这 3 个线程，往往会有数十个线程，所以这里直接看线程是起不到作用的。

接着运行 `!locks` 命令查看一下当前进程中的所有 `CRITICAL_SECTION` ：

![]( {{site.url}}/asset/debugging-dead-lock-lock-command.png )

可以看到此时进程中有三个 `CRITICAL_SECTION`, 分别是 `g_mutex1`, `g_mutex2`, `g_mutex3`。重点关注下 `LockCount` 这个字段，它表示当前正在请求这个关键段的线程的数量，它的初始值是 -1, 如果 `LockCount` 为 5, 那么表示已经有 1 个线程获取到了这个关键段的锁，另外还有 5 个线程正在试图获取这个关键段。上图中 `LockCount` 的值都为 1, 表示它们都已经被某个线程持有，同时还另有一个线程正在等待它。

另外还需要关注 `OwningThread` 这个字段，它表示该关键段当前被哪个线程所持有，`g_mutex1` 和 `g_mutex3` 都被 398c 这个线程所持有，`g_mutex2` 则由 1d58 持有。这个 398c 和 1d58 就是线程 ID, 如果想要知道他们的线程号，可以通过菜单栏中的 View --> Processes and Threads 来打开窗口，查看线程号和线程 ID 的对应关系：

![]( {{site.url}}/asset/debugging-dead-lock-processes-and-threads-window.png )

从上图可以看出，398c 对应 0 号线程，1d58 对应 1 号线程。接着可以通过 `~0kb` 和 `~1kb` 来看下这两个线程的调用堆栈：

![]( {{site.url}}/asset/debugging-dead-lock-thread-callstack.png )

观察堆栈中 RtlEnterCriticalSection 函数的第一个参数，分别是 01019150 和 01019138, 这跟我们之前用 `!locks` 命令打印出的 `CRITICAL_SECTION` 信息是对应的：

![]( {{site.url}}/asset/debugging-dead-lock-mutex-address.png )

01019150 和 01019138 实际上就是 `g_mutex2` 和 `g_mutex1` 变量的地址。

到现在其实已经可以确定了，`main()` 与 `ThreadFunc()` 发生了死锁，它们的调用堆栈中已经有了源代码信息及行数，可以很快定位到死锁代码的具体位置。


## 解决死锁

对于上述问题，解决的一个思路是让这两个线程按照同样的顺序申请和释放锁，也就是说，让 `main()` 和 `ThreadFunc()` 都按照先 `g_mutex1` 后 `g_mutex2` 的顺序来申请锁，都按照先 `g_mutex2` 后 `g_mutex1` 的顺序来释放锁。这样的话 `mian()` 在获得 `g_mutex1` 的情况下，`ThreadFunc()` 绝无机会再获取到 `g_mutex2`，也就可以避免两者互相等待了。

在这个例子中，只需要修改 `ThreadFunc()` 中 `EnterCriticalSection()` 的顺序就行了，实际中的情况会复杂一些，很有可能这这两处代码不是在一个函数中，不过知道原因就好办了，总有调整的办法。


## 一些想法

实际工作中几乎不可能遇到上述例子中这么简单的情况，一般至少都会有几十个线程，许多个 `CRITICAL_SECTION` 变量。而且程序卡住不一定是死锁造成的，比如主线程在等待一个由其他线程持有的锁，这时候主线程会被卡住，但并没有死锁，只是别的线程没有释放锁而已。

不过 `!locks` 命令还是很有用的，可以立刻排除一些情况，注意使用这个命令的时候要检查下 `LockCount` 字段，排除掉那些无人等待的 `CRITICAL_SECTION`，从而缩小范围。

总的来说上述问题的定位要从锁而不是线程入手，当前被占用的 `CRITICAL_SECTION` 可能没有几个，但线程一般是有很多的，直接查看线程堆栈可能一时找不到头绪。

另外，怎么发现程序卡住？这就是前期的问题了，比如主界面卡住了，显然就是主线程卡住了，这个时候直接看主线程堆栈就可以知道主线程卡在哪里。如果是其他线程卡住了，有可能界面上不会有明显变化，这种情况我能想到的也就只有通过日志来查看了。


## C++11 的 std::mutex

可以进行互斥操作的方法不只有 `CRITICAL_SECTION`, 先来看看 C++ 11 的 `std::mutex` 能不能用之前的方法来分析。

示例代码如下，逻辑跟之前一样，只是换成了 `std::thread` 和 `std::mutex` ：

```cpp
#include <mutex>
#include <iostream>
#include <thread>

std::mutex g_mutex1;
std::mutex g_mutex2;

void ThreadFunc()
{
	g_mutex2.lock();
	std::cout << "ThreadFunc get g_mutex2\n";
	std::this_thread::sleep_for(std::chrono::milliseconds(500));

	g_mutex1.lock();
	std::cout << "ThreadFunc get g_mutex1" << std::endl;

	g_mutex1.unlock();
	g_mutex2.unlock();
}

int main()
{
	std::thread work_thread(ThreadFunc);

	g_mutex1.lock();
	std::cout << "main get g_mutex1\n";
	std::this_thread::sleep_for(std::chrono::milliseconds(1000));

	g_mutex2.lock();
	std::cout << "main get g_mutex2" << std::endl;

	g_mutex1.unlock();
	g_mutex2.unlock();

	work_thread.join();
	return 0;
}
```

用 `!locks` 查看一下现在占有的锁，可以看到当前并没有锁被占用。这是因为 `!locks` 命令只是用来检查 `CRITICAL_SECTION` 的，不能检查其他锁。再用 `~0kb` 命令看一下 `main()` 中的堆栈：

![]( {{site.url}}/asset/debugging-dead-lock-std-mutex.png )

可以看到 `std::mutex` 最终是使用了 `SRWLock` 而不是 `CRITICAL_SECTION`, 两者区别在于一个是读写锁，一个是互斥锁。不过它使用 `SRWLock` 的时候是通过 `AcquireSRWLockExclusive` 而不是 `AcquireSRWLockShared` 来获取的，也就是说它是以写者的方式，独占地获取锁的。这不就跟 `CRITICAL_SECTION` 一样了嘛！不知道为啥要这么实现，不过总之我们之前的 `!locks` 命令用不了了。

那这种问题要怎么分析呢？

未完待续 ...

## 参考资料

[Who Is Blocking That Mutex? - Fun With WinDbg, CDB And KD](http://weblogs.thinktecture.com/ingo/2006/08/who-is-blocking-that-mutex---fun-with-windbg-cdb-and-kd.html)