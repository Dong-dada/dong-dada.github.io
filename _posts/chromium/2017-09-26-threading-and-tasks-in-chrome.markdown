---
layout: post
title:  "Chrome 中的 Threading 和 Tasks"
date:   2017-09-26 12:33:30 +0800
categories: chromium
---

* TOC
{:toc}


## 概览

### 线程

每个 chrome 进程都包含有：
- 一个主线程：
    - 在 browser 进程中负责：更新 UI；
    - 在 renderer 进程中负责：运行 Blink；
- 一个 IO 线程：
    - 在 browser 进程中负责：处理 IPC 和网络请求；
    - 在 renderer 进程中负责：处理 IPC 请求；
- 一些处理特殊操作的线程；
- 一个处理通用操作的线程池；

大部分的线程都包含有一个 loop, 用来从 queue 中获取 tasks 并运行(这个 queue 可能被多个线程共享)；

### Tasks

一个 task 是一个继承自 `base::OneClosure` 的对象，它会被添加到线程的 queue 里异步执行；

一个 `base::OneClosure` 会存储一个函数指针及其参数。它包含有一个 `Run()` 方法，该方法执行时，会通过函数指针调用函数，并传入绑定的参数。`base::OneClosure` 对象可以通过 `base::BindOnce()` 来创建，具体可参考 [Callback<> and Bind()](https://chromium.googlesource.com/chromium/src/+/master/docs/callback.md)：

```cpp
void TaskA() {}
void TaskB(int v) {}

auto task_a = base::BindOnce(&TaskA);
auto task_b = base::BindOnce(&TaskB, 42);
```

一组 tasks 可以通过以下几种方式来执行：
- [Parallel](https://chromium.googlesource.com/chromium/src/+/master/docs/threading_and_tasks.md#Posting-a-Parallel-Task)(并行): 没有执行顺序，有可能同时在多个线程中执行；
- [Sequenced](https://chromium.googlesource.com/chromium/src/+/master/docs/threading_and_tasks.md#Posting-a-Sequenced-Task)(序列): 按照 task 投递的顺序来执行，同一时刻只有一个 task 被执行，但不同的 task 可能在不同的线程上执行；
- [Single Threaded](https://chromium.googlesource.com/chromium/src/+/master/docs/threading_and_tasks.md#Posting-Multiple-Tasks-to-the-Same-Thread)(单线程): 按照 task 投递的顺序来执行，同一时刻只有一个 task 被执行，且这组 tasks 都在同一个线程上执行；
    - [COM Single Threaded](https://chromium.googlesource.com/chromium/src/+/master/docs/threading_and_tasks.md#Posting-Tasks-to-a-COM-Single-Thread-Apartment-STA_Thread-Windows_)(COM 单线程): 在初始化了 COM 环境的单线程上执行这组 tasks.

### Sequenced 模式比 Single Threaded 模式更好

在只需要保证线程安全的场景下，Sequenced 这种执行模式比 Single Threaded 模式更好。因为它可以对要执行的任务进行调度，比如在跳过那些正在忙碌的线程，这也意味着进程中的线程数量可以根据机器环境动态调整。


## 投递一个 Parallel Task

### 直接投递到 Task Scheduler

如果一个 task 可以在任意线程上运行，并且与其他 task 没有执行顺序等依赖关系，那么它应该使用 `base::PostTask*()` 系列函数来投递：

```cpp
base::PostTask(FROM_HERE, base::BindOnce(&Task));
```

上述 `PostTask` 函数使用了默认的 traits.

使用 `base::PostTask*WithTraits()` 系列函数可以对投递过程进行一些特殊设置，可参考 [Annotating Tasks with TaskTraits](https://chromium.googlesource.com/chromium/src/+/master/docs/threading_and_tasks.md#Annotating-Tasks-with-TaskTraits)

```cpp
base::PostTaskWithTraits(
    FROM_HERE, {base::TaskPriority::BACKGROUND, MayBlock()},
    base::BindOnce(&Task));
```

### 通过 TaskRunner 来投递

`TaskRunner` 是直接调用 `base::PostTask*()` 的一种替代品。它主要用在当你不知道应该用何种方式投递 task 的时候(in parallel, in sequence, or to single-thread?)。

`TaskRunner` 是 `SequencedTaskRunner`, `SingleThreadTaskRunner` 的基类，所以你可以用 `scoped_ptr<TaskRunner>` 指针来指向它们之中某一个的实例，下述代码展示了在不知道该选择何种投递方式的时候，通过 `scoped_ptr<TaskRunner>` 成员来进行测试的例子：

```cpp
class A {
 public:
  A() = default;

  void set_task_runner_for_testing(
      scoped_refptr<base::TaskRunner> task_runner) {
    task_runner_ = std::move(task_runner);
  }

  void DoSomething() {
    // In production, A is always posted in parallel. In test, it is posted to
    // the TaskRunner provided via set_task_runner_for_testing().
    task_runner_->PostTask(FROM_HERE, base::BindOnce(&A));
  }

 private:
  scoped_refptr<base::TaskRunner> task_runner_ =
      base::CreateTaskRunnerWithTraits({base::TaskPriority::USER_VISIBLE});
};
```

除非是上述代码中需要显式控制 task 如何执行的情况，你都应该直接调用 `base::PostTask*()` 系列函数来投递 parallel task；


## 投递一个  Sequenced Task

正如之前所说，一组 sequenced tasks 中，每个任务都按照投递的顺序依次执行(但不一定要在同一个线程中)。要投递一个 sequenced task, 应该使用 `SequencedTaskRunner`.

### 投递到一个新的  sequence

可以通过 `base::CreateSequencedTaskRunnerWithTraits()` 来创建一个新的 `SequencedTaskRunner`:

```cpp
scoped_refptr<SequencedTaskRunner> sequenced_task_runner =
    base::CreateSequencedTaskRunnerWithTraits(...);

// TaskB runs after TaskA completes.
sequenced_task_runner->PostTask(FROM_HERE, base::BindOnce(&TaskA));
sequenced_task_runner->PostTask(FROM_HERE, base::BindOnce(&TaskB));
```

### 投递到一个已有的  sequence

你可以通过 `SequencedTaskRunnerHandle::Get()` 来获取当前 task 被投递到的 `SequencedTaskRunner` 对象：

```cpp
// The task will run after any task that has already been posted
// to the SequencedTaskRunner to which the current task was posted
// (in particular, it will run after the current task completes).
// It is also guaranteed that it won’t run concurrently with any
// task posted to that SequencedTaskRunner.
base::SequencedTaskRunnerHandle::Get()->
    PostTask(FROM_HERE, base::BindOnce(&Task));
```

注意：在一个 parallel task 里调用 `SequencedTaskRunnerHandle::Get()` 是不合法的，但你可以在一个 single-threaded task 里调用它。因为 `SingleThreadTaskRunner` 是一种特殊的 `SequencedTaskRunner`.


## 使用 Sequences 而不是 Locks

chromium 项目不鼓励使用锁，因为 Sequences 内置了线程安全支持。相对于自己通过锁来实现某个类的线程安全性，更好的解决方法是在相同的 sequence 里访问类成员：

```cpp
class A {
 public:
  A() {
    // Do not require accesses to be on the creation sequence.
    DETACH_FROM_SEQUENCE(sequence_checker_);
  }

  void AddValue(int v) {
    // Check that all accesses are on the same sequence.
    DCHECK_CALLED_ON_VALID_SEQUENCE(sequence_checker_);
    values_.push_back(v);
}

 private:
  SEQUENCE_CHECKER(sequence_checker_);

  // No lock required, because all accesses are on the
  // same sequence.
  std::vector<int> values_;
};

A a;
scoped_refptr<SequencedTaskRunner> task_runner_for_a = ...;
task_runner->PostTask(FROM_HERE,
                      base::BindOnce(&A::AddValue, base::Unretained(&a)));
task_runner->PostTask(FROM_HERE,
                      base::BindOnce(&A::AddValue, base::Unretained(&a)));

// Access from a different sequence causes a DCHECK failure.
scoped_refptr<SequencedTaskRunner> other_task_runner = ...;
other_task_runner->PostTask(FROM_HERE,
                            base::BindOnce(&A::AddValue, base::Unretained(&a)));
```


## 投递多个 task 到同一个线程

如果有多个 task 需要运行在同一个线程，你可以把他们投递到同一个 `SingleThreadTaskRunner` 里面。这些任务将按照投递顺序在同一个线程中依次执行。

### 向 browser 进程的主线程或  IO 线程投递 task

要向主线程或 IO 线程投递 task, 需要首先通过 `content::BrowserThread::GetTaskRunnerForThread()` 获取到正确的 `SingleThreadTaskRunner`：

```cpp
content::BrowserThread::GetTaskRunnerForThread(content::BrowserThread::UI)
    ->PostTask(FROM_HERE, ...);

content::BrowserThread::GetTaskRunnerForThread(content::BrowserThread::IO)
    ->PostTask(FROM_HERE, ...);
```

主线程或 IO 线程通常都很忙，所以应该尽可能把 task 投递到 通用线程里。只有在需要更新 UI 或者访问那些与 UI 有关的对象的时候才适合投递到主线程；只有在需要访问那些与 IO 线程相绑定的组件的时候才适合投递到 IO 线程。

### 向 renderer 进程的主线程投递  task

TODO

### 向自定义的 SingleThreadTaskRunner 投递 task

如果一些任务需要运行在相同线程，并且这些线程不一定得是主线程或 IO 线程，那你可以把它们投递到一个由 `base::CreateSingleThreadTaskRunnerWithTraits` 创建的 `SingleThreadTaskRunner` 里面：

```cpp
scoped_refptr<SequencedTaskRunner> single_thread_task_runner =
    base::CreateSingleThreadTaskRunnerWithTraits(...);

// TaskB runs after TaskA completes. Both tasks run on the same thread.
single_thread_task_runner->PostTask(FROM_HERE, base::BindOnce(&TaskA));
single_thread_task_runner->PostTask(FROM_HERE, base::BindOnce(&TaskB));
```

### 向当前线程投递  task

要向当前线程投递 task, 可以使用 `ThreadTaskRunnerHandle`:

```cpp
// The task will run on the current thread in the future.
base::ThreadTaskRunnerHandle::Get()->PostTask(
    FROM_HERE, base::BindOnce(&Task));
```

注意：如果你投递的 task 只是与 sequence 中的其他 task 有依赖，但不一定需要在当前线程中运行，那么你应该使用 `SequencedTaskRunnerHandle::Get()`, 这表示这个任务并不要求一定要在当前线程运行。

### 向 COM Single-Thread Apartment (STA) 线程投递 task

如果 task 需要在 COM Single-Thread Apartment (STA) 线程中执行，那么必须通过 `CreateCOMSTATaskRunnerWithTraits()` 来创建一个 `SingleThreadTaskRunner`, 然后再把这个 task 投递到 TaskRunner 上。

```cpp
// Task(A|B|C)UsingCOMSTA will run on the same COM STA thread.

void TaskAUsingCOMSTA() {
  // [ This runs on a COM STA thread. ]

  // Make COM STA calls.
  // ...

  // Post another task to the current COM STA thread.
  base::ThreadTaskRunnerHandle::Get()->PostTask(
      FROM_HERE, base::BindOnce(&TaskCUsingCOMSTA));
}
void TaskBUsingCOMSTA() { }
void TaskCUsingCOMSTA() { }

auto com_sta_task_runner = base::CreateCOMSTATaskRunnerWithTraits(...);
com_sta_task_runner->PostTask(FROM_HERE, base::BindOnce(&TaskAUsingCOMSTA));
com_sta_task_runner->PostTask(FROM_HERE, base::BindOnce(&TaskBUsingCOMSTA));
```


## 通过 TaskTraits 来定制 Tasks

`TaskTraits` 封装了 tasks 的一些信息，task scheduler 通过 `TaskTraits` 来决定如何调度 task.

`base/task_scheduler/post_task.h` 中的所有 `PostTask*()` 方法都对应有一个复写的方法，它能够接收 `TaskTraits` 作为参数来对要投递的 task 进行定制。

那些不接收 `TaskTraits` 参数的方法适用于投递以下种类的 tasks:
- 不会阻塞;
- 优先级不需要调整；
- 能接受在程序退出时跳过执行该 task. (task scheduler 可以自行选择是否执行完该任务再退出).

不满足上述条件的 tasks 都应该使用 TaskTraits 来进行投递。你可以在 `base/task_scheduler/task_traits.h` 中找到 TaskTraits 的详细文档，以下是一些设置 TaskTraits 的例子：

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


## 保持浏览器的响应性

应该尽量避免在主线程上执行耗时操作，最好把这些操作放到 IO 线程或者其它低优先级的 sequence 上。你可以通过 `base::PostTaskAndReply*()` 或者 `SequencedTaskRunner::PostTaskAndReply()` 系列函数来异步执行耗时操作。

例如，以下代码有可能会导致主线程长时间无响应：

```cpp
// GetHistoryItemsFromDisk() may block for a long time.
// AddHistoryItemsToOmniboxDropDown() updates the UI and therefore must
// be called on the main thread.
AddHistoryItemsToOmniboxDropdown(GetHistoryItemsFromDisk("keyword"));
```

以下代码解决了上述代码中的问题，它把耗时的 `GetHistoryItemsFromDisk()` 函数放到了线程池中执行，接着在原始 sequence 里调用 `AddHistoryItemsToOmniboxDropdown()` (这里也就是主线程). 第一次调用的返回值将自动作为参数传递给第二个函数：

```cpp
base::PostTaskWithTraitsAndReplyWithResult(
    FROM_HERE, {base::MayBlock()},
    base::BindOnce(&GetHistoryItemsFromDisk, "keyword"),
    base::BindOnce(&AddHistoryItemsToOmniboxDropdown));
```


## 投递带有延时的  task

### 投递只执行一次的 延时 task

如果希望投递的 task 在延时一段时间后再执行，可以使用 `base::PostDelayedTask*()` 或 `TaskRunner::PostDelayedTask()`：

```cpp
base::PostDelayedTaskWithTraits(
  FROM_HERE, {base::TaskPriority::BACKGROUND}, base::BindOnce(&Task),
  base::TimeDelta::FromHours(1));

scoped_refptr<base::SequencedTaskRunner> task_runner =
    base::CreateSequencedTaskRunnerWithTraits({base::TaskPriority::BACKGROUND});
task_runner->PostDelayedTask(
    FROM_HERE, base::BindOnce(&Task), base::TimeDelta::FromHours(1));
```

注意：当你要去设置一个一小时延时的 task 时，考虑一下这个 task 是不是不一定要求必须在一个小时后执行，你可以为它设置 `base::TaskPriority::BACKGROUND`, 降低它运行的优先级，从而避免延时到达的时候影响浏览器执行其他 task.

### 投递重复执行的延时  task

要让 task 按照一定间隔时间重复执行，可以使用 `base::RepeatingTimer`:

```cpp
class A {
 public:
  ~A() {
    // The timer is stopped automatically when it is deleted.
  }
  void StartDoingStuff() {
    timer_.Start(FROM_HERE, TimeDelta::FromSeconds(1),
                 this, &MyClass::DoStuff);
  }
  void StopDoingStuff() {
    timer_.Stop();
  }
 private:
  void DoStuff() {
    // This method is called every second on the sequence that invoked
    // StartDoingStuff().
  }
  base::RepeatingTimer timer_;
};
```


## 取消 task

### 通过 base::WeakPtr

`base::WeakPtr` 可以保证，当某个对象被销毁的时候，任何绑定到它的回调也被取消：

```cpp
int Compute() { … }

class A {
 public:
  A() : weak_ptr_factory_(this) {}

  void ComputeAndStore() {
    // Schedule a call to Compute() in a thread pool followed by
    // a call to A::Store() on the current sequence. The call to
    // A::Store() is canceled when |weak_ptr_factory_| is destroyed.
    // (guarantees that |this| will not be used-after-free).
    base::PostTaskAndReplyWithResult(
        FROM_HERE, base::BindOnce(&Compute),
        base::BindOnce(&A::Store, weak_ptr_factory_.GetWeakPtr()));
  }

 private:
  void Store(int value) { value_ = value; }

  int value_;
  base::WeakPtrFactory<A> weak_ptr_factory_;
};
```

注意上述例子中的 `weak_ptr_factory_` 成员变量，它可以产生一个 A 对象的 `base::WeakPtr` 指针，我们把这个指针传给 `base::PostTaskAndReplyWithResult()`, 如果 `weak_ptr_factory_` 被销毁了，那么当回调即将执行的时候，它将无法通过 weakptr 获取到 `this` 指针，从而取消这次 task.

### 通过 base::CancelableTaskTracker

`base::CancelableTaskTracker` 可以取消其他 sequence 上的 task:

```cpp
auto task_runner = base::CreateTaskRunnerWithTraits(base::TaskTraits());
base::CancelableTaskTracker cancelable_task_tracker;
cancelable_task_tracker.PostTask(task_runner.get(), FROM_HERE,
                                 base::Bind(&base::DoNothing));
// Cancels Task(), only if it hasn't already started running.
cancelable_task_tracker.TryCancelAll();
```


## 遗留的 PostTask APIs

chrome browser process 有一些遗留的命名线程 (也叫做 "BrowserThreads"). 这些线程用来执行某种特定任务，比如 `FILE` 线程用于执行低优先级的文件操作，`FILE_USER_BLOCKING` 线程用于执行高优先级的文件操作，`CACHE` 线程用于处理 cache 操作等等。。现在不推荐使用这些命名线程了，新代码应该通过 post task 到 task scheduler 来完成类似操作。

如果因为某些原因你必须得把 task 投递到某个命名线程(比如要投递的 task 与命名线程中的某些 task 有依赖关系)，你可以使用如下方法：

```cpp
content::BrowserThread::GetTaskRunnerForThread(content::BrowserThread::[IDENTIFIER])
    ->PostTask(FROM_HERE, base::BindOnce(&Task));
```

其中的 `IDENTIFIER` 可以取值为：`DB`, `FILE`, `FILE_USER_BLOCKING`, `PROCESS_LAUNCHER`, `CACHE`.


## 在新进程中使用  TaskScheduler

在之前那些代码跑起来之前，你得先在进程中初始化 TaskScheduler. 这些初始化步骤在 chrome browser process 及其子进程中已经做好了，如果需要在其它进程中使用 TaskScheduler, 那么你需要在主函数中对它进行初始化：

```cpp
// This initializes and starts TaskScheduler with default params.
base::TaskScheduler::CreateAndStartWithDefaultParams(“process_name”);
// The base/task_scheduler/post_task.h API can now be used. Tasks will be
// scheduled as they are posted.

// This initializes TaskScheduler.
base::TaskScheduler::Create(“process_name”);
// The base/task_scheduler/post_task.h API can now be used. No threads
// will be created and no tasks will be scheduled until after Start() is called.
base::TaskScheduler::GetInstance()->Start(params);
// TaskScheduler can now create threads and schedule tasks.
```

进程退出的时候也要对其进行反初始化：

```cpp
base::TaskScheduler::GetInstance()->Shutdown();
// Tasks posted with TaskShutdownBehavior::BLOCK_SHUTDOWN and
// tasks posted with TaskShutdownBehavior::SKIP_ON_SHUTDOWN that
// have started to run before the Shutdown() call have now completed their
// execution. Tasks posted with
// TaskShutdownBehavior::CONTINUE_ON_SHUTDOWN may still be
// running.
```
