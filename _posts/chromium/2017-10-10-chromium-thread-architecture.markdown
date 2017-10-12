---
layout: post
title:  "[总结] Chromium 线程架构"
date:   2017-10-10 11:58:30 +0800
categories: chromium
---

* TOC
{:toc}


总结一下目前了解到的 chromium 线程架构，不一定准确。

## 概览

### chrome 中的线程

在 chrome 里，需要有以下几种线程来完成工作：
- 一个主线程：
    + 在 browser process 中负责： 更新 UI;
    + 在 renderer process 中负责： 运行 Blink;
- 一个 IO 线程：
    + 在 browser process 中负责： 处理 IPC 和网络请求；
    + 在 renderer process 中负责： 处理 IPC 请求；
- 一些处理特殊操作的线程（命名线程，例如 `BrowserThread::IO`, `BrowserThread::DB`）；
- 一个处理通用操作的线程池(thread pool)；

这涉及到了两种情况：一个是执行特定操作的特殊线程；另一个是执行通用操作的线程池。

### Event Loop 机制

开发多线程程序总是需要解决线程间通信的问题，比如 A 线程需要在 B 线程里执行一些操作，或者某个资源可能被 A 或 B 同时访问之类的。 chromium 中使用 Event Loop 机制来避免不安全的线程操作，大部分线程都包含有一个 loop, 用来从 queue 里取出 task 并执行。

task 表示你想执行的某个操作，大部分情况下它由一个函数，以及绑定到该函数的一系列参数构成。

把 task 投递到 queue 里面，线程的 loop 从 queue 里取出这个 task 并执行，这是 chromium 中大部分线程的运行方式。

Event Loop 机制可以有效避免线程不安全问题，对于那些通用的 task, 就把它投入到线程池中运行；如果需要某个 task 在特殊的线程上运行，就把它投入到目标线程的 queue 里面。你只需要保证不同线程中的 task 不会同时访问同一个资源即可，并不需要考虑加锁的问题。

### Task 的执行顺序

正如之前所说，根据多个 task 之间的相关性，一组 tasks 可以按照以下几种方式来执行：
- Parallel(并行): 没有执行顺序，有可能同时在多个线程中执行；
- Sequenced(序列): 按照 task 投递的顺序来执行，同一时刻只有一个 task 被执行，但不同的 task 可能在不同的线程上执行；
- Single Threaded(单线程): 按照 task 投递的顺序来执行，同一时刻只有一个 task 被执行，且这组 tasks 都在同一个线程上执行；

可以看到这三个执行方式的限制依次加强：
- `Parallel` 方式适合于那些完全独立的 task, 比如读取一个文件中的内容，不需要考虑它在哪个线程里执行，执行的顺序。这种 task 就适合放到线程池里，让他以 `Parallel` 的方式去执行；
- `Sequenced` 方式增加了一个限制，那就是 task 必须按照投递的顺序去执行，但它对 task 执行的线程没有限制(之所以会这样往往是因为 task 之间具有关联性)。比如写入同一个文件的两个 task, 必须等前一个 task 写入完毕了才能开始执行第二个 task, 这就比较适合以 `Sequenced` 的方式去执行；
- `Single Threaded` 方式进一步增加了限制，不但要求 tasks 按顺序执行，而且必须在同一个线程里执行这些 tasks(之所以会这样往往是因为 task 具有特殊性). 比如用于更新 UI 的 task, 必须在主线程中按顺序执行；

显然，`Parallel` 和 `Sequenced` 方式适合于线程池的情况，`Single Thread` 方式适合于特殊线程的情况。这里需要注意的一点是，对于线程池的情况而言，可以通过调度器来对要执行的 task 进行调度，伸缩性上会比在特殊线程里执行任务更好，因此如果一个 task 没有必要以 `Single Threaded` 的方式执行，那么应当尽量让他们以 `Sequenced` 或 `Parallel` 的方式来执行。

### 小结

在这一节中，我们首先列举了 chrome 中需要的一些线程，它们分为特殊线程和线程池两种情况；

接着，我们了解到 chrome 使用 Event Loop 机制来解决线程间通信的问题，它并不能完全保证线程安全，仍旧需要使用者注意 task 的投递方式和资源访问方式；

接着，我们列举了一组 tasks 的执行顺序，根据 task 之间是否有关联性， task 是否必须在特定线程中执行，将 tasks 的执行顺序分为了三类。我们使用时也应该根据 tasks 的这两个特点来决定 task 运行的方式；


## 使用

这一节介绍一下如何在 chromium 中完成 task 的投递。主要从使用的角度来介绍，先不考虑具体的实现原理。

### 构造 Task

chrome 中的 task 是一个继承自 `base::OneClosure` 的对象，它会被添加到线程的 queue 里异步执行；

一个 `base::OneClosure` 会存储一个函数指针及其参数。它包含有一个 `Run()` 方法，会通过函数指针调用函数，并传入绑定的参数。`base::OneClosure` 对象可以通过 `base::BindOnce()` 函数来创建：

```cpp
void TaskFuncA() {
    // ...
}

void TaskFuncB(int num, double radio) {
    // ...
}

// 绑定普通函数
auto task_a = base::BindOnce(&TaskFuncA);
auto task_b = base::BindOnce(&TaskFuncB, 42, 0.1);

class TestClass {
public:
    TaskFuncC(std::string str);
}

// 绑定类成员函数
TestClass obj;
auto task_c = base::BindOnce(&TestClass::TaskFuncC, &obj, "hahaha");
```

可以看到 `base::OneClosure`, `base::BindOnce()` 类似于 C++ 中的 `std::function` 和 `std::bind()`. 将参数与函数绑定在一起的这一功能也被称为 [柯里化](https://en.wikipedia.org/wiki/Partial_application), 柯里化常被用来进行 [惰性求值](https://zh.wikipedia.org/wiki/%E6%83%B0%E6%80%A7%E6%B1%82%E5%80%BC) 中，我们将 Task 投递到某个线程中执行，也是一种惰性求值的情况。

### 投递 Parallel Task

正如上一节所说，如果一个 task 可以在任意线程上运行，且与其他 task 没有执行顺序等依赖关系，那么它应该以 Parallel 方式运行，你可以使用 `base::PostTask*()` 系列函数来投递它. task 会被投递到调度器 `TaskSchedule` 中(进程唯一)，由调度器将其投递到线程池中合适线程的 message queue 里。

```cpp
// 投递 Task
base::PostTask(FROM_HERE, base::BindOnce(&TaskFuncB, 42, 0.1));

// 投递延时 Task
base::PostDelayedTask(FROM_HERE,
                      base::BindOnce(&TaskFuncC, "hahaha"),
                      base::TimeDelta::FromMinutes(1));

// 投递 Task 并等待回复
base::PostTaskAndReply(FROM_HERE,
                       base::BindOnce(&TaskFuncA, 42),
                       base::BindOnce(&TaskFuncB, 42));

// 投递 Task 并等待回复，TaskFuncA 的返回值将作为参数传给 TaskFuncB
base::PostTaskAndReplyWithResult(FROM_HERE,
                                 base::BindOnce(&TaskFuncA, 42),
                                 base::BindOnce(&TaskFuncB));
```

上述 `base::PostTask*()` 系列函数使用了默认的 traits. `TaskTraits` 封装了 task 的一些信息，调度器会通过 `TaskTraits` 来决定如何调度 task. 上述所有 `base::PostTask*()` 方法都对应有一个复写的 `base::PostTask*WithTraits()`方法，它能够接收 `TaskTraits` 作为参数来对要投递的 task 进行定制：

```cpp
// 第二个参数就是 TaskTraits, USER_BLOCKING 表示该 task 优先级较高，
// 调度器会优先执行该 task
base::PostTaskWithTraits(
    FROM_HERE, {base::TaskPriority::USER_BLOCKING},
    base::BindOnce(...));
```

以下种类的 tasks 不需要通过 `TaskTraits` 来投递，使用默认 traits 即可：
- 不会阻塞;
- 优先级不需要调整；
- 能接受在程序退出时跳过执行该 task. (task scheduler 可以自行选择是否执行完该任务再退出).

其他种类的 tasks 应该根据情况设置合适的 `TaskTraits` 来调用，你可以在 [base/task_scheduler/task_traits.h](https://cs.chromium.org/chromium/src/base/task_scheduler/task_traits.h?type=cs) 中看到详细描述，以下是一些例子：

```cpp
// This task has no explicit TaskTraits. It cannot block. Its priority
// is inherited from the calling context (e.g. if it is posted from
// a BACKGROUND task, it will have a BACKGROUND priority). It will either
// block shutdown or be skipped on shutdown.
base::PostTask(FROM_HERE, base::BindOnce(...));

// This task has the highest priority. The task scheduler will try to
// run it before USER_VISIBLE and BACKGROUND tasks.
base::PostTaskWithTraits(
    FROM_HERE, {base::TaskPriority::USER_BLOCKING},
    base::BindOnce(...));

// This task has the lowest priority and is allowed to block (e.g. it
// can read a file from disk).
base::PostTaskWithTraits(
    FROM_HERE, {base::TaskPriority::BACKGROUND, base::MayBlock()},
    base::BindOnce(...));

// This task blocks shutdown. The process won't exit before its
// execution is complete.
base::PostTaskWithTraits(
    FROM_HERE, {base::TaskShutdownBehavior::BLOCK_SHUTDOWN},
    base::BindOnce(...));
```

### 投递 Sequenced Task

如果你的 tasks 需要按照顺序来执行，但对于运行在哪个线程里没有要求，那么它应该使用 `Sequenced` 的方式来运行，这种情况下你应该使用 `SequencedTaskRunner` 接口进行投递：

```cpp
// 创建一个新的 SequencedTaskRunner
scoped_refptr<SequencedTaskRunner> sequenced_task_runner =
    base::CreateSequencedTaskRunnerWithTraits(...);

// 获取当前设置的 SequencedTaskRunner
scoped_refptr<SequencedTaskRunner> runner =
    base::SequencedTaskRunnerHandle::Get();

// 将两个 task 投递到同一个 SequencedTaskRunner 里，TaskB 将等待 TaskA 执行完毕后
// 再执行，二者可能运行在不同线程里。
runner->PostTask(FROM_HERE, base::BindOnce(&TaskFuncA, 42));
runner->PostDelayedTask(FROM_HERE, base::BindOnce(&TaskFuncB, 42));
```

**注意**： 在一个 parallel task 里调用 SequencedTaskRunnerHandle::Get() 是不合法的，但你可以在一个 single-threaded task 里调用它。因为 `Single Threaded` 是 `Sequenced` 的一种特例。

### 投递 Single Threaded Task

如果你的 tasks 需要在某个特定的线程里按顺序执行，那么它应该使用 `Single Threaded` 方式来运行。

要把一个 task 按照 `Single Threaded` 的方式投递到特定线程，首先要找到这个线程。更具体的，是需要找到这个线程上的 `SingleThreadTaskRunner`。

chrome 的大部分线程中都包含有 **一个** `SingleThreadTaskRunner`, 可以通过 `base::ThreadTaskRunnerHandle::Get()` 获取当前线程的 `SingleThreadTaskRunner` 对象：

```cpp
// 获取当前线程的 SingleThreadTaskRunner 对象
scoped_refptr<SingleThreadTaskRunner> runner =
    base::ThreadTaskRunnerHandle::Get();

// 向当前线程投递 Single Threaded task
runner->PostTask(FROM_HERE, base::BindOnce(&TaskFuncA, 42));
```

chrome 中有一些线程需要执行特殊操作，具有特别的名字，我们把它们叫做命名线程。比如 `FILE` 线程用于执行低优先级的文件操作，`FILE_USER_BLOCKING` 用于执行高优先级的文件操作，`CACHE` 用于处理 cache 操作等。

这些命名线程都由 `content::BrowserThread` 类代表，你可以通过 `content::BrowserThread::GetTaskRunnerForThread()` 函数来获取这些命名线程中的 `SingleThreadTaskRunner` 对象：

```cpp
// 获取命名线程的 SingleThreadTaskRunner, IDENTIFIER 取值可以是 UI, IO, FILE 等
scoped_refptr<SingleThreadTaskRunner> runner =
    content::BrowserThread::GetTaskRunnerForThread(
        content::BrowserThread::[IDENTIFIER]);

// 向命名线程投递 Single Threaded Task
runner->PostTask(FROM_HERE, base::BindOnce(&Task));
```

上述代码中 IDENTIFIER 的取值可参考 [src/content/public/browser/browser_thread.h](https://cs.chromium.org/chromium/src/content/public/browser/browser_thread.h?type=cs)

向自定义线程里投递 Single Thread Task, 需要先自己创建一个 SingleThreadTaskRunner 对象出来，这里暂不介绍。

### 小结

这一节我们首先讨论了 Task 对象的构造方式，它跟 C++ 的 `std::function` 和 `std::bind（）` 很像，通过柯里化来完成了任务的惰性计算。

接着我们讨论了如何按照执行方式来投递 task 到不同的目标：
- 如果是 `Parallel` 方式，则通过 `base::PostTask*()` 来投递；
- 如果是 `Sequenced` 方式，则通过 `SequencedTaskRunner` 来投递；
- 如果是 `Single Threaded` 方式，则通过 `SingleThreadTaskRunner` 来投递；

接下来我们分析下这个线程架构的具体实现。


## 单线程中  Event Loop 的实现

先来看下单个线程中 Event Loop 机制是如何建立的，这部分内容可以参考 [这篇文章](https://docs.google.com/document/d/1_pJUHO3f3VyRSQjEhKVvUU7NzCyuTCQshZvbWeQiCXU/view#heading=h.okllz7fdmm0) 以及 [这篇文章](http://blog.gclxry.com/chromium-multi-thread/)。

总体来说，实现上述机制涉及到了如下几个类：
- [base::Thread](https://cs.chromium.org/chromium/src/base/threading/thread.h?type=cs) ： 表示一个包含了 MessageLoop 的线程，一个 `base::Thread` 对像对应一个线程；
- [base::MessageLoop](https://cs.chromium.org/chromium/src/base/message_loop/message_loop.h?type=cs) ： 表示一个 MessageLoop, 包含了一个 MessagePump 用于驱动 loop 运行，一个 IncomingTaskQueue 存储传入的 tasks, 一个 TaskQueue 存储正在执行的 tasks. 这两个 task queue 构成了类似于双缓冲的技术，后面会提到。另外还有一个 MessageLoopTaskRunner 用于向 InComingTaskQueue 中插入 task;
- [base::MessagePump](https://cs.chromium.org/chromium/src/base/message_loop/message_pump.h?type=cs) ： 表示消息泵，顾名思义用来驱动 MessageLoop 运行；
- [internel::IncomingTaskQueue](https://cs.chromium.org/chromium/src/base/message_loop/incoming_task_queue.h?type=cs) ： 存储传入的 tasks, 等 MessageLoop 有空闲的时候，将它与 `base::TaskQueue` 交换；
- [TaskQueue](https://cs.chromium.org/chromium/src/base/pending_task.h?type=cs&l=56) : 实际上是 `base::queue<PendingTask>`, 一个待执行任务的队列；
- [internal::MessageLoopTaskRunner](https://cs.chromium.org/chromium/src/base/message_loop/message_loop_task_runner.h?type=cs) : 实现了 `SingleThreadTaskRunner` 接口，用于向 InComingTaskQueue 插入 task;

下图是上述几个类协同工作的过程：

![]( {{site.url}}/asset/chrome-thread-architecture-event-loop.png )

上图中的流程分为两个部分：
- 线程外或线程内，通过 `MessageLoopTaskRunner` 将 `Task` 投递到 InComingTaskQueue 里面，完成 PostTask 这部分的工作；
- MessagePump 触发 `MessageLoop::Dowork()`， 执行完 TaskQueue 里的 Tasks 后，将 IncomingTaskQueue 与 TaskQueue 交换，TaskQueue 里又得到新的 Tasks, 继续执行这些 Tasks.

有一些细节需要注意：
- `internal::InComingTaskQueue` 是一个阻塞队列，因此可以在多个线程中向其 post task;
- `TaskQueue` 实际上是 `base::queue<PendingTask>`, 因为它始终只能由当前线程所访问，因此不需要考虑线程安全问题;

### TaskRunner

上述机制中，似乎 `MessageLoopTaskRunner` 的作用不那么明显，它完成的任务仅仅是将 task 投递到 IncomingTaskQueue 里面，直接由 MessageLoop 去暴露一个接口来 PostTask 应该也行，为啥要单独搞出来一个 TaskRunner 呢？

它的目的是为了将 task 如何被执行进行解耦。正如我们在一开始说的, task 有三种不同的执行方式，如果想要修改 task 的执行方式，只需要替换一个新的 TaskRunner 即可。比如一开始有个专门处理文件操作的线程叫 FileThread, 后来我们希望能够按照优先级来调度投递过来的 Task, 这时候只需要换一个新的 TaskRunner 就可以了，不需要更改原来的。

之前说过的 `MessageLoopTaskRunner` 继承自 `SingleThreadTaskRunner` 接口，我们看看另外几个 TaskRunner 接口是干啥用的：

- `TaskRunner` 接口 ： 规定了 `Parallel` 执行方式，不保证 task 执行顺序，不保证 task 在哪个线程执行；
- `SequencedTaskRunner` 接口 ： 规定了 `Sequenced` 执行方式，保证 task 执行顺序，不保证 task 在哪个线程中执行；
- `SingleThreadTaskRunner` 接口 ： 规定了 `Single Threaded` 执行方式，保证 task 在同一线程中按顺序执行；

可以看到 TaskRunner 的几个接口与我们之前说的 tasks 执行方式是一一对应的，要让 task 以哪种方式执行，就需要获取到对应的 TaskRunner 接口。

我们之前提到过，如果需要获取当前线程的 SingleThreadTaskRunner, 可以通过 `base::ThreadTaskRunnerHandle::Get()` 函数来获取，这个函数是怎么获取的呢？

首先，在 MessageLoop 绑定到 Thread 的时候，它会以自己的 `MessageLoopTaskRunner` 对象为参数构造一个 `ThreadTaskRunnerHandle` 对象；

接着，`ThreadTaskRunnerHandle` 的构造函数里，会把自己设置到线程局部存储区里；

最后，调用 `ThreadTaskRunnerHandle::Get()` 静态方法，是从线程局部存储里取出这个 `ThreadTaskRunnerHandle` 对象，然后再取出其中保存的 TaskRunner;

因此，你就可以在当前线程中直接调用静态方法来获取到当前 MessageLoop 中的 TaskRunner 了。

### 小结

这一节介绍了单线程中实现 Event Loop 的方法。它支持了我们之前提到的 Single Threaded 的 task 执行方式。

Event Loop 机制的实现并不复杂，核心就是一个 task 的队列，一个消息泵不断从队列中取出 task 并执行。

chrome 在实现时加入了双缓冲机制，避免 task 的投递与获取发生线程之间的竞争。

`SingleThreadTaskRunner` 接口被保存在线程局部存储区里，因此可以在线程中调用 `ThreadTaskRunnerHandle::Get()` 来获取。

接下来我们看看 Parallel 和 Sequence 执行方式是如何实现的。


## TaskScheduler 任务调度的实现

正如之前所说，对于 `Parallel` 和 `Sequenced` 类型的 task, 应该被投递到线程池里运行。chrome 的 `base::TaskScheduler` 就是干这个事儿的。

`base::TaskScheduler` 是一个单例，负责将 task 调度到合适的线程中执行。

查看 [TaskSchedulerImpl](https://cs.chromium.org/chromium/src/base/task_scheduler/task_scheduler_impl.h?type=cs&l=111) 的代码，可以看到 TaskScheduler 里有 4 个 `SchedulerWorkerPoolImpl`, 这 4 个线程池分别负责 `background`, `background blocking`, `foreground`, `foreground blocking` 这 4 种不同的 task 类型。

每次调用 `TaskScheduler::PostDelayedTaskWithTraits()`, 都会根据 traits 决定将 task 分发 4 个线程池之一。

我现在还不懂为啥要搞 4 个线程池出来，应该是某种策略吧。这里先不追究了。

### SchedulerWorkerPool 线程池

接着 `SchedulerWorkerPool::PostTaskWithSequence()` 会被调用，它的第一个参数是要投递的 task, 第二个参数是一个 `Sequence` 类型的值。

第二个参数看起来有点儿奇怪，其实根据它的名字就可以知道，它就是为了完成我们之前说的 Sequence 这种 task 运行方式而搞出来的一个数据结构。为了让 task 按照顺序执行，需要用一个类似于列表的数据结构来存储 task, 如果我们两次调用 `PostTaskWithSequence()` 都传入了同一个 sequence, 那么这两个 task 就会被放到同一个 sequence 里面，线程池在运行完第一个 task 之后，才会开始运行第二个 task.

`SchedulerWorkerPool` 提供了一个 `CreateSequencedTaskRunnerWithTraits()` 函数，该函数会返回一个 `SchedulerSequencedTaskRunner` 对象，这个对象内部又持有了一个 `Sequence`, 如果我们用这个对象来 PostTask, task 就会被投递到同一个 sequence 里，由此来完成 Sequence 执行方式。

因此，在任何地方，如果我们需要用 Sequence 方式执行一组 task, 那么首先应该创建一个新的 `SchedulerSequencedTaskRunner` 对象，然后把所有 task 都投递到这个 TaskRunner 对象里：

```cpp
scoped_refptr<SequencedTaskRunner> sequenced_task_runner =
    base::CreateSequencedTaskRunnerWithTraits(...);

// TaskB runs after TaskA completes.
sequenced_task_runner->PostTask(FROM_HERE, base::BindOnce(&TaskA));
sequenced_task_runner->PostTask(FROM_HERE, base::BindOnce(&TaskB));
```

### PriorityQueue 优先级队列

`SchedulerWorkerPoolImpl` 里有一个 [shared_priority_queue](https://cs.chromium.org/chromium/src/base/task_scheduler/scheduler_worker_pool_impl.h?type=cs&l=198) 成员变量，它是一个 PriorityQueue 类型的优先级队列，用来存储待执行的 `Sequence`。

查看 [PriorityQueue 的实现](https://cs.chromium.org/chromium/src/base/task_scheduler/priority_queue.h?type=cs&l=43) 可以看到，每次向队列中 Push Sequence, 都需要传入一个 `SequenceSortKey` 参数，这个参数有两个字段：
- TaskPriority : Sequence 的所有任务中，最高的优先级
- sequence_time : Sequence 中，任务的最早投递时间

PriorityQueue 内部使用 `std::priority_queue` 来实现，它的排序规则是：
- 如果 TaskPriority 不同，那么 TaskPriority 高的那个排前面；
- 否则把 `sequence_time` 比较早的那个排前面；

Task 被转添加到 Sequence, 然后添加到 PriorityQueue 里面，始终保持着优先级最高的 Sequence 在前面。等待着 SchedulerWorker 的处理。

### SchedulerWorker 工作线程

`SchedulerWorkerPoolImpl` 里有一个 [SchedulerWorker 数组](https://cs.chromium.org/chromium/src/base/task_scheduler/scheduler_worker_pool_impl.h?type=cs&l=219), 顾名思义它就是用来执行 task 的线程。

查看 [SchedulerWorker 的实现](https://cs.chromium.org/chromium/src/base/task_scheduler/scheduler_worker.cc?type=cs&l=38) 可以发现，它用底层的线程自己封装了一个执行 task 的循环，而没有使用之前说过的 MessageLoop 机制。其中的关键代码如下：

```cpp
  void ThreadMain() override {
    outer_->delegate_->OnMainEntry(outer_.get());

    // A SchedulerWorker starts out waiting for work.
    outer_->delegate_->WaitForWork(&wake_up_event_);

    while (!outer_->ShouldExit()) {
      DCHECK(outer_);

      // Get the sequence containing the next task to execute.
      scoped_refptr<Sequence> sequence =
          outer_->delegate_->GetWork(outer_.get());
      if (!sequence) {
        outer_->delegate_->WaitForWork(&wake_up_event_);
        continue;
      }

      const bool sequence_became_empty =
          outer_->task_tracker_->RunNextTask(sequence.get());

      outer_->delegate_->DidRunTask();

      // If |sequence| isn't empty immediately after the pop, re-enqueue it to
      // maintain the invariant that a non-empty Sequence is always referenced
      // by either a PriorityQueue or a SchedulerWorker. If it is empty
      // and there are live references to it, it will be enqueued when a Task is
      // added to it. Otherwise, it will be destroyed at the end of this scope.
      if (!sequence_became_empty)
        outer_->delegate_->ReEnqueueSequence(std::move(sequence));

      // Calling WakeUp() guarantees that this SchedulerWorker will run
      // Tasks from Sequences returned by the GetWork() method of |delegate_|
      // until it returns nullptr. Resetting |wake_up_event_| here doesn't break
      // this invariant and avoids a useless loop iteration before going to
      // sleep if WakeUp() is called while this SchedulerWorker is awake.
      wake_up_event_.Reset();
    }

    outer_->delegate_->OnMainExit(outer_.get());

    // Break the ownership circle between SchedulerWorker and Thread.
    // This can result in deleting |this| and as such no more member accesses
    // should be made after this point.
    outer_ = nullptr;
  }
```

可见是一个 while 循环，不断通过 delegate (实际上是 `SchedulerWorkerPoolImpl`) 获取要执行的 Sequence, 执行完了之后就进入等待，直到被唤醒。

上述代码中有一个判断，如果任务执行完之后 Sequence 不为空，那么就将它重新入队，这时候 sequence 会重新加入 PriorityQueue 并重新排序，从而保证 PriorityQueue 的队首始终是优先级最高的 sequence。

### SchedulerWorkerPool 对任务的调度

有了 Priority 和 SchedulerWorker, 基本的功能已经可以实现了，`SchedulerWorkerPool` 需要对投递过来的 task 进行一些调度，优先唤醒无事可做的线程，实在没有办法才创建新的线程。

这部分我没有梳理清楚，以后再详细看看吧。。。

### 小结

这一节简略介绍了 chrome 中 TaskScheduler 的实现。关键着眼于 PriorityQueue 和 SchedulerWorker 这两个类。

然而还有许多问题没有讨论清楚：
- 线程池如何对投递过来的 task 进行调度？
- 如何动态增减线程的数量？
- 如何确保 Sequence 中的 tasks 是在上一个执行完毕之后才开始执行下一个的？
