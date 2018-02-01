---
layout: post
title:  "Java 集合"
date:   2018-02-01 18:40:30 +0800
categories: java
---

* TOC
{:toc}

## 线程操作

### 启动线程

构造 Thread 对象时，需要传入一个 `Runnable` 类型的 target 参数作为线程的入口点。

这个 Runnable 可以是 lambda 表达式、或者是一个实现了 `Runnable` 接口的类对象。因此可以有如下几种写法：

```java
class SomeService implements Runnable {
    public void init() {
        System.out.println(Thread.currentThread().getName());

        // 自身实现 Runnable 接口
        Thread thread1 = new Thread(this, "thread1");
        thread1.start();

        // 通过 lambda 来实现 Runnable 接口
        Runnable runnable = () -> {
            System.out.println(Thread.currentThread().getName());
        };
        Thread thread2 = new Thread(runnable, "thread2");
        thread2.start();

        // 通过内部类来实现 Runnable 接口
        Thread thread3 = new Thread(new Worker(), "thread3");
        thread3.start();
    }

    @Override
    public void run() {
        System.out.println(Thread.currentThread().getName());
    }

    private class Worker implements Runnable {

        @Override
        public void run() {
            System.out.println(Thread.currentThread().getName());
        }
    }
}
```

注意：
- Java 中主线程结束后，子线程并不会结束，这一点跟 windows 的线程不太一样。具体说来，Java 中有守护线程的概念，只有当所有非守护线程都结束的时候，进程才会结束。
- Java 中某个线程里发生了非受查异常，那么该线程会结束，但这不会导致其它线程也结束。

### 中断线程

正如常识，线程结束的唯一正确方法是自然退出，这一点对 Java 而言也是一样的。

Java 提供了一个 `Thread.interrupt()` 方法，该方法可以为线程对象设置一个 boolean 标记，你可以在线程中通过 `Thread.currentThread.isInterrupted()` 来判断该标记是否为 true, 从而决定是否结束线程：

```java
public class ThreadTest {

    public static void main(String[] args) {
        Runnable runnable = () -> {
            // 判断中断标记，从而决定是否退出线程
            while (!Thread.currentThread().isInterrupted()) {
                // do more work
            }
        };

        Thread thread = new Thread(runnable, "worker");
        thread.start();

        // 设置中断标记
        thread.interrupt();
    }
}
```

注意，当线程此时被阻塞的时候，调用 `interrupt()` 方法无法起到预期的效果，因为线程没机会去检查中断标记。这时候阻塞的方法会被中断，然后抛出一个 `InterruptedException` 异常：

```java
public class ThreadTest {

    public static void main(String[] args) {
        Runnable runnable = () -> {
            try {
                while (!Thread.currentThread().isInterrupted()) {
                    // ...

                    // sleep 阻塞了当前线程，此时调用 interrupt 方法，将引起 InterruptedException
                    Thread.sleep(1000);
                }
            } catch (InterruptedException e) {
                e.printStackTrace();
            }
        };

        Thread thread = new Thread(runnable, "worker");
        thread.start();
        thread.interrupt();
    }
}
```

并不是所有的阻塞操作都可以被中断，有些阻塞操作是无法被 `interrupt()` 方法中断的，这些以后再讨论。

### 线程状态

线程有如下 6 个状态 (定义于 `Thread.State` 枚举中，可以通过 `Thread.getState()` 来获取)：
- NEW ： 线程还未 start；
- RUNNABLE : 线程已经 start, 正在运行中；
- BLOCKED ： 当线程试图获取一个内部的对象锁(而不是 java.util.concurrent 中的锁)，而该锁被其它线程持有，则该线程进入 BLOCKED 状态；
- WAITING ： 当线程等待另一个线程通知调度器一个条件 (condition) 的时候，该线程进入 WAITING 状态。
- TIMED_WAITING ： 某些方法具有超时参数，调用这些方法将导致线程进入 TIMED_WAITING 状态；
- TERMINATED ： 线程已经终止。

`BLOCKED` 和 `WAITING` 这两个状态看起来有点难以区分。我理解 BLOCKED 是因为 *互斥* 而导致的阻塞； 而 `WAITING` 是因为 *条件* 而导致的阻塞。有点儿类似于 Windows 的关键段和条件变量，Java 认为这两种情况导致的阻塞是不一样的。这点稍后再议。

### 守护线程

调用 `Thread.setDaemon(true)` 可以把一个线程转换成 “守护线程”。 它跟普通线程没有啥区别，唯一不同的在于，如果进程中只剩 “守护线程”，那么程序就会退出。

从这点来看，“守护线程” 的定位应该是给其它线程服务的，它自己不应该有什么业务逻辑，也不应该去访问什么资源。

比较典型的 “守护线程” 是计时线程，它每隔一定时间给其他线程发个消息，告诉其它线程你该清理内存啦之类的。

### 未捕获异常处理器

`run()` 方法不会抛出任何异常，因此你需要在 `run()` 方法中处理所有受查异常。但非受查异常会导致线程终止，然后线程死亡。

线程因为异常而死亡，并不会导致整个程序退出，其它线程还是可以正常跑的。

如果你想要接受线程中的非受查异常，那么可以给线程设置一个未捕获异常的处理器：

```java
void uncaughtException(Thread t, Throwable e);

thread.setUncaughtExceptionHandler(...)
```

如果没有设置未捕获异常处理器，那么 Java 会使用 ThreadGroup 对象作为异常处理器，它实现了 Thread.UncaughtExceptionHandler 接口。


## 同步基础

java.util.concurrent 里提供了一系列类来帮助完成线程同步。

### 锁

java.util.concurrent 提供了一个可重入的锁 `ReentrantLock`。类似于 C++ 中的 `std::mutex`，这个锁提供了互斥的功能：

```java
Lock lock = new ReentrantLock();
lock.lock();
// 用 try finally 来包裹关键段代码，避免发生异常后无法 unlock
try {
    // critical section
} finally {
    lock.unlock();
}
```

Java 中没有提供不可重入的锁。我觉得最好把锁看成是不可重入的，不要依赖于可重入而编写对应的代码。

### 条件变量

java.util.concurrent 提供了一个条件变量 `Condition`。类似于 C++ 中的 `std::condition_variable`。

```java

class MessageQueue {
    private List<Integer> queue = new LinkedList<>();
    private ReentrantLock queueLock = new ReentrantLock();
    private Condition queueCondition = queueLock.newCondition();

    public void put(int message) {
        queueLock.lock();
        queue.add(message);
        queueLock.unlock();

        // 队列不为空，唤醒一个等待者
        queueCondition.signal();
    }

    public int take() {
        int top = 0;

        queueLock.lock();
        try {
            // 如果队列为空，进入等待
            while (queue.size() == 0) {
                // await 会原子地 unlock, 不会导致此时无法调用 put
                queueCondition.await();
                // await 返回后，会自动 lock, 继续保护 queue
            }
            assert queue.size() > 0;

            top = queue.get(0);
            queue.remove(0);
        } catch (InterruptedException e) {
            e.printStackTrace();
        } finally {
            queueLock.unlock();
        }

        return top;
    }
}
```

注意：
- Condition 需要通过 `Lock.newCondition()` 来创建，而不能独立于 Lock 存在；
- 等待的方法是 `await()` 而不是 `wait()`，估计是因为 `wait()` 被 Object 占用了；

Condition 之所以需要用 Lock 来创建，是因为进入 await 状态之后，必须把保护变量的锁释放掉，否则其它地方 (put) 无法写入变量。

梳理一下上面提到的锁和条件变量：
- 锁用来保护代码片段、任何时候只能有一个线程执行被保护的代码；
- 锁可以管理试图进入被保护代码段的线程；
- 锁可以拥有一个或多个相关的条件对象；
- 每个条件对象管理那些已经进入被保护的代码段但还不能运行的线程；

### synchronized 关键字

除了直接使用 Lock 和 Condition, Java 还给每个对象都添加了一个内部锁以及一个内部条件变量，相当于每个 Object 实例里都有一个 Lock 对象和一个 Condition 对象。

`synchroinzed` 关键字可以用来声明一个方法，如此一来每次进入这个方法，都会通过内部锁来给这个方法加锁，每次退出这个方法，都会通过内部锁来给这个方法解锁：

```java
public synchronized void method() {
    // ...
}

// 相当于
public void method() {
    this.intrinsicLock.lock();
    try {
        // ...
    }
    finally {
        this.intrinsicLock.unlock();
    }
}
```

此外， Object 类还有几个方法 `wait()`, `notify()`, `notifyAll()`。它们通过内部的条件变量来完成工作：

```java
public synchronized int take() {
    while (queue.size() == 0) {
        wait();
    }

    // ...
}

public synchronized void put(int message) {
    queue.add(message);

    notify();
}
```

synchronized 关键字虽然方便，但是不像 Lock 那样明显。如果一个类中有多个 field 被保护，那么 synchronized 就不太合适了。`wait()` 和 `notify()` 也是同样的道理。


### volatile 关键字

Java 中的 volatile 是一个弱化版本的锁。

锁主要提供了两种特性：
- 互斥 (mutual exclusion) : 同一时间只能有一个线程来持有锁，或者说同一时间只能有一个线程访问某个变量；
- 可见性 (visiblity) : 也叫做 happens-before, 就是说，当 A 线程修改了某个变量之后， B 线程获取到的是修改后最新的值；

为啥会出现可见性问题，我没有一个特别清晰的概念，因为对底层不太清楚。

有一种说法是，在多核的机器上，两个线程分别在不同的 CPU 内核上运行，各自有自己的 CPU 缓存，有可能 A 线程写入的是自己 CPU 的缓存，而 B 线程读取的是自己 CPU 的缓存，这就导致 A 线程修改变量后，有可能 B 线程无法读取到正确的值。这种说法不太对，因为 Java 代码是运行在 JVM 上的，而不是直接运行在 CPU 上，把 CPU 缓存替换为 JVM 的主内存区和线程栈可能比较恰当。

volatile 只能保证可见性，但不具备互斥的特性。也因为这一点， volatile 的应用场景非常有限，要使用 volatile, 必须满足以下条件：

- 对变量的写操作不依赖于当前的值 (`count++ 是不对的`)；
- 该变量没有包含在具有其它变量的不变式中；

第二个条件比较难理解，不过你可以认为： 给 volatile 变量赋值的时候，赋给它的值不能依赖于任何程序的状态，包括变量的当前状态。因此 `count++`, `flag = !flag`, `count = size` 都是不对的。

之所以会有上述限制，是因为存在多个线程同时对 volatile 变量进行写操作的情况，例如 `count++` 这一操作，其实分了三个步骤：
1. 获取 count 的当前值
2. 计算 count + 1
3. 把计算结果保存到 count 中

这就可能造成 A 和 B 线程同时执行了第 1 步，当它们执行到第 3 步的时候，保存的结果是一样的。只有在去掉第 1 步的操作的情况下，两个线程同时写的操作才是安全的。

这使得 volatile 的应用场景非常有限，代码也会比较难维护。

以下是一个安全使用 volatile 关键字的例子：

```java
// 把 volatile 变量作为标志来使用
volatile boolean shutdownRequested;

public void shutdown() {
    shutdownRequested = true;
}

public void doWork() {
    while (!shutdownRequest) {
        // ...
    }
}
```

总的说来 volatile 比较危险，最好不要用它。另外 Java 中的 volatile 和 C++ 是不太一样的。 C++ 的 volatile 只是告诉编译器，这个值可能在当前代码以外的地方被修改，不要认为它的值就是固定的就把它优化掉，但它不能保证这个代码在运行的时候不会被 CPU 乱序执行或者做别的优化，因此 C++ 的 volatile 更加不安全，似乎更不应该被使用。

以上内容参考自如下文章：
- [正确使用 volatile 变量](https://www.ibm.com/developerworks/cn/java/j-jtp06197.html)
- [C++ 多线程中什么时候应该使用 volatile 呢？](https://www.zhihu.com/question/31459750)
- [谈谈 C++ 中 volatile 关键字](https://zhuanlan.zhihu.com/p/33074506)

### 原子操作

java.util.concurrent.atomic 包中有很多类提供了原子操作。例如 AtomicInteger :

```java
public static AtomicLong nextNumber = new AtomicLong();

long id = nextNumber.incrementAndGet();
```

### 线程局部存储

线程局部存储的意义在于，它能够确保变量不被其它线程所访问。Java 提供了 ThreadLocal 辅助类来访问线程局部存储区：

```java
// 构造 ThreadLocal 变量
public static final ThreadLocal<SimpleDateFormat> dateFormat =
    ThreadLocal.withInitial(() -> new SimpleDateFormat("yyyy-mm-dd"));

// 访问 ThreadLocal 变量
String dateStamp = dateFormat.get().format(new Date());
```


## 高级同步

上述介绍了一些线程同步的基础设施，实际情况下应该尽量避免使用这些基础设施，多尝试使用现有的模型。

### 阻塞队列

java.util.concurrent 包提供了阻塞队列的几个实现：
- LinkedBlockingQueue : 阻塞队列，默认容量无限；
- LinkedBlockingDeque : 阻塞的双端队列；
- ArrayBlockingQueue : 指定容量的阻塞队列；
- PriorityBlockingQueue : 带优先级的阻塞队列；

阻塞队列可以用于实现生产者消费者模式，这儿就不废话了。

### 线程安全的集合

java.util.concurrent 包提供了一些线程安全的集合：
- ConcurrentHashMap;
- ConcurrentSkipListMap;
- ConcurrentSkipListSet;
- ConcurrentLinkedQueue;

### Callable 和 Future

之前已经介绍过 Runnable 接口，你可以把它看成是一个异步计算的任务，只是它没有返回值。

Callable 与 Runnable 接口类似，只是它会有一个返回值：

```java
public interface Callable<V> {
    V call() throws Exception;
}
```

如此一来，你可以用 `Callable<Integer>` 来表示一个最终会返回 Integer 对象的异步计算。你可以把 Callable 包装成 Runnable, 交给某个线程去执行这个计算，问题在于如何获取计算的结果。

Java 另外提供了一个接口 `Future`, 它可以用来保存计算的结果：

```java
public interface Future<V> {
    V get() throws ...;
    V get(long timeout, TimeUnit unit) throws ...;
    void cancel(boolean mayInterrupt);
    boolean isCancelled();
    boolean isDone();
}
```

当子线程计算完毕后，你可以利用 Future 来获取计算的结果。如果还没有计算结束，`get()` 方法会阻塞。

Java 提供了 FutureTask 包装器，它实现了 Runnable 和 Future 接口，并且通过 Callable 来构造。因此，你可以通过 Callable 构造一个 FutureTask, 然后把这个 task 交给某个线程去执行，接着通过 `FutureTask.get()` 等待计算结果：

```java
Callable<Integer> myComputation = (String dirPath) -> {
    int count = 0;
    // 计算文件数量 ...
    return count;
};

FutureTask<Integer> task = new FutureTask<Integer>(myComputation);
Thread worker = new Thread(task);
worker.start();

// ...

int filesCount = task.get();
```

直接创建子线程来执行异步计算的场景不太常见。下面的小节可以看到，Callable 和 Future 也可以用在线程池里面。

### 执行器

Java 提供了一个执行器 Executor 类来提供构建线程池的接口，它提供了如下几个方法来构建特定的线程池：
- newCachedThreadPool : 必要的时候创建新线程，空闲线程会被保留 60 秒；
- newFixedThreadPool : 包含固定量的线程；
- newSingleThreadExecutor : 只有一个线程的 “池”，顺序执行每一个提交的任务；
- newScheduledThreadPool : 用于预订执行而构建的固定线程池，替代 java.util.Timer；
- newSingleThreadScheduledExecutor : 用于预订执行而构建的单线程 “池”；

前 3 个接口会返回一个实现了 ExecutorService 接口的 ThreadPoolExecutor 对象，接着你可以使用该接口的 submit 方法将 Callable 投递到线程池里运行，submit 会返回给你一个实现了 Future 接口的对象，利用 Future 接口可以获取计算的结果。

下面是一个简单的例子：

```java
public class ThreadPoolTest {
    public static void main(String[] args) {
        ExecutorService pool = Executors.newCachedThreadPool();

        FileParser parser1 = new FileParser("C:/file1.txt");
        Future<Integer> result1 = pool.submit(parser1);

        FileParser parser2 = new FileParser("C:/file2.txt");
        Future<Integer> result2 = pool.submit(parser2);

        try {
            System.out.println(result1.get());
            System.out.println(result2.get());
        } catch (InterruptedException e) {
            e.printStackTrace();
        } catch (ExecutionException e) {
            e.printStackTrace();
        }

        pool.shutdown();
    }

    private static class FileParser implements Callable<Integer> {
        private String filePath;

        public FileParser(String filePath) {
            this.filePath = filePath;
        }

        @Override
        public Integer call() throws Exception {
            System.out.println("Parsing file: " + filePath);
            Thread.sleep(1000);
            return 0;
        }
    }
}
```

线程池的每个线程应该都有一个 task queue 之类的东西，顺序执行每个 submit 进来的 Callable。

之前提到 Executor 还有另外两个接口 `newScheduledThreadPool` 和 `newSingleThreadScheduledExecutor`，这两个方法会返回一个实现了 ScheduledExecutorService 接口的对象。用于预订执行 (Scheduled Execution) 或重复执行任务。

你可以预订 Runnable 或 Callable 在一定延迟之后执行，也可以预订一个 Runnable 对象周期性地执行。这里就不多介绍了。
