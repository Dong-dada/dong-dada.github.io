---
layout: post
title:  "Python 并发编程——线程"
date:   2017-05-17 15:19:30 +0800
categories: python
---

* TOC
{:toc}


本文基于 python3

python 提供了两个标准模块 `_thread` 和 `threading` 来对多线程操作提供支持，下面分别介绍它们。

## _thread 模块

具体内容可参考 [官方文档](https://docs.python.org/3/library/_thread.html)。

`_thread` 模块提供了低层次的线程封装。包括线程的启动、关闭，以及一个简单的互斥锁。

下面是它提供的提供的 API:

```py
# 启动一个新线程
_thread.start_new_thread(function, args[, kwargs])

# 退出当前线程
_thread.exit()

# 结束主线程
_thread.interrupt_main()

# 获取线程 ID
_thread.get_ident()

# 分配一个互斥锁，返回的互斥锁对象默认是没加锁的
lock = _thread.allocate_lock()

# lock 对象要求获取锁
lock.acquire(waitflag = 1, timeout = -1)

# lock 对象主动释放锁
lock.release()

# 判断 lock 对象此时是否拥有锁
lock.locked()
```

接下来用一个简单的累加代码来示范上述 API 的使用方法：

```py
import _thread
import time

sum = 0

lock = _thread.allocate_lock()

def add():
    print("thread %d enter" % _thread.get_ident())

    # 在函数中使用全局变量，要先用 global 声明一下
    global sum

    for i in range(5):
        lock.acquire()

        # 把 sum += 1 分成两个语句，中间加上 sleep, 以便观察线程切换可能导致的问题
        new = sum + 1
        time.sleep(0.001)
        sum = new

        lock.release()

_thread.start_new_thread(add, ())
_thread.start_new_thread(add, ())

time.sleep(1)

print(sum)

# 输出：
# thread 6280 enter
# thread 7576 enter
# 10
```

`_thread` 模块所提供的功能非常简单，一般用功能更强大的 `threading` 来完成多线程操作。


## threading 模块

具体内容可参考 [官方文档](https://docs.python.org/3/library/threading.html)

`threading` 模块提供了高层次的线程封装，提供了 `Thread` 对象来表示线程，并提供了 互斥锁、可重入的互斥锁、条件变量、信号量、事件量、Timer 对象、Barrier 对象等线程同步支持。

`threading` 模块本身提供了如下 API:

```py
# 获取当前的 Thread 对象
threading.current_thread()

# 获取主线程 Thread 对象
threading.main_thread()

# 获取当前线程的线程 ID
threading.get_ident()

# 返回当前线程数量
threading.active_count()

# 返回当前所有线程的列表
threading.enumerate()

# 返回当前线程的 Thread-Local Data
threading.local()
```

可以看到 `threading` 模块本身暴露的 API 只是做一些管理工作，对于线程的封装主要在 `Thread` 对象上，以下是 `Thread` 对象暴露的 API：

```py
# 构造方法
threading.Thread(group = None, target = None, name = None, args = (), kwargs = {}, *, daemon = None)

# 启动该线程
start()

# 默认提供的线程函数，如果在构造时传入了 target, 那么 run 方法将被赋值为 target 中的函数
# 你可以在子类中重写 run 方法，这是设定 Thread 线程函数的另一种方法
run()

# 等待该线程对象所指的线程执行结束
join(timeout = None)

# 线程名，可以用属性的方式来访问
name

# 线程 ID, 可以用属性的方式来访问
ident

# 线程是否活着
is_alive()
```

对比 `_thread` 模块而言，Thread 对象中提供了线程名这个属性，并且增加了 `join()` 方法可以等待线程结束，下面是使用 Thread 的例子：

```py
import threading

def foo():
    print("foo")

# 通过构造函数传入 target 的方式来指定线程函数
t = threading.Thread(target = foo)
t.start()
t.join()

# 通过复写 Thread.run() 函数的方式来指定线程函数
class BarThread(threading.Thread):
    def run(self):
        print("BarThread run")

t = BarThread()
t.start()
t.join()
```

上面这些功能看起来跟 `_thread` 区别不大，下面介绍一下 `threading` 中提供的同步机制。


## threading 模块中提供的同步机制

`threading` 中提供了如下几种对象来对线程同步进行支持：
- **Lock Objects** : 互斥锁；
- **RLock Objects** : 可重入的互斥锁；
- **Condition Objects** : 条件变量
- **Semaphore Objects** : 信号量；
- **Event Objects** : 事件量；
- **Timer Objects** : 定时器；
- **Barrier Objects** : 

### Lock 互斥锁

`Lock` 对象中只有两个方法：`acquire(blocking = True, timeout = -1)` 和 `release()`。

```py
import threading

lock = threading.Lock()

lock.acquire()
# ...
lock.release()
```

### RLock 可重入的互斥锁

`RLock` 对象中也只有两个方法：`acquire(blocking = True, timeout = -1)` 和 `release()`。

可重入的意思是，如果当前线程已经通过 `acquire()` 获取了锁，那么它再次调用 `acquire()` 的时候不会阻塞，可以继续得到这个锁。这跟 `Lock` 不同，对于 `Lock` 而言，如果当前线程已经获取了锁，再次调用 `Lock` 就会造成死锁：

```py
import threading

lock = threading.Lock()

def foo():
    print("foo enter")
    global lock

    lock.acquire()  # 因为是不可重入的 Lock 对象，执行到这一句时会死锁，如果是 RLock 就不会
    # ...
    lock.release()
    print("foo leave")

def bar():
    print("bar enter")
    global lock

    lock.acquire()
    foo()
    lock.release()
    print("bar leave")

bar()

print("exit")

# 输出
# bar enter
# foo enter
```

过去读过 《linux多线程服务器端编程》 这本书，作者提到过，可重入的互斥锁 RLock 容易造成潜在的问题(锁的不配对释放等)，最好只使用不可重入的 Lock 互斥锁。

### Condition 条件变量

严格说来之前的 `Lock` 和 `RLock` 只能用于互斥，不能用于同步 —— 你无法用互斥量来实现 "A 等待某个条件成立，B 设置这个条件" 这种需求。例如线程 A 等待 `num > 100` 这个条件，线程 B 对 `num` 变量进行累加操作：

```py
import threading

num = 0

def foo_a():
    global num
    if num > 100:
        num = 0

def foo_b():
    global num
    for i in range(200):
        num += 1

t_a = threading.Thread(target = foo_a)
t_a.start()

t_b = threading.Thread(target = foo_b)
t_b.start()
```

上述逻辑不能用互斥量实现，因为 A 和 B 都需要访问 num, 必须用互斥锁来保护 num 这个共享变量。在不修改原有代码的情况下，无论这个锁加在哪里，都无法实现 "A 等待 B 将 num 设为 100 以上后继续执行" 这种功能。

条件变量 Condition 正是为了实现线程同步功能所引入的一个机制。我们看一下用条件变量实现上述功能的代码：

```py
import threading
import time

num = 0
cond = threading.Condition()

def foo_a():
    print("foo_a enter")
    global num

    cond.acquire()
    while num <= 5:
        cond.wait()
    num = 0
    cond.release()

    print("foo_a leave")

def foo_b():
    print("foo_b enter")
    global num

    for i in range(10):
        cond.acquire()
        num += 1
        print("foo_b num = %d" % num)
        if num == 6:
            cond.notify()
            print("foo_b notify")
        cond.release()
        time.sleep(0.001)    # 没有这一句就会有问题，其原因估计跟 GIL 有关
    print("foo_b leave")

t_a = threading.Thread(target = foo_a)
t_a.start()

t_b = threading.Thread(target = foo_b)
t_b.start()

t_a.join()
t_b.join()

# 输出
# foo_a enter
# foo_b enter
# foo_b num = 1
# foo_b num = 2
# foo_b num = 3
# foo_b num = 4
# foo_b num = 5
# foo_b num = 6
# foo_b notify
# foo_a leave
# foo_b num = 1
# foo_b num = 2
# foo_b num = 3
# foo_b num = 4
# foo_b leave
```

注意上述代码中加入了一个 `time.sleep(0.001)` 的代码，这句代码的作用是让线程 B 执行完操作之后给 A 一个执行的机会，之所以需要这一句，估计与 python GIL 有关，这个稍后会介绍。

最后看一下 Condition 类暴露的 API:

```py
# 构造方法，可以传入一个 Lock 或 RLock, 如果不传，那么 Condition 会自己创建一个 RLock
threading.Condition(lock = None)

# 请求获取锁
acquire(*args)

# 主动释放锁
release()

# 等待被触发
wait(timeout = None)

# 等待满足指定条件, predicate 是一个函数
wait_for(predicate, timeout = None)

# 触发 n 个正在等待的线程
notify(n = 1)

# 触发所有正在等待的线程
notify_all()
```

思考如下问题：
- 为什么判断条件时是使用 `while num <= 5:` 而不是使用 `if num <= 5:` ？
- `notify()` 和 `notify_all()` 有什么区别？
- 调用 `wait()` 时，线程所持有的锁会自动释放， `wait()` 被触发后，线程会重新持有锁，这是为什么？

### Semaphore 信号量

信号量用于实现互斥，而不是同步。它也有 `acquire()` 和 `release()` 这两个方法，所不同的是信号量有一个初始的计数值，比如 5, 每次调用 `acquire` 都会让这个计数值减 1, 一旦计数值变为了 0, 再次调用 `acquire()` 就会阻塞当前线程。这一特性使得它可以控制同一时间访问资源的最大数量。

这样看来，互斥量是信号量的一种特例，它的计数值为 1 。

### Event 事件量

事件量是一种很简单的同步机制。它包含了一个 `wait()` 方法，可以在另一个线程中调用 `set()` 方法来激发它：

```py
# 构造方法，刚创建时事件是未触发状态
threading.Event()

# 事件是否被触发
is_set()

# 触发事件
set()

# 重置事件
clear()

# 等待事件被触发
wait(timeout = None)
```

Event 跟 Condition 相比，主要的区别在于 Event 没有锁来对资源进行保护。试想一下之前条件变量的累加例子里，如果只用 Event 变量是无法实现该功能的，因为 `num` 这个值不但决定了触发的条件，同时也是两个线程的共享资源，必须用锁保护起来才行。

### Timer 计时器

计时器的作用很简单，过一段时间之后启动一个子线程，执行指定的方法：

```py
import threading
import time

def foo():
    print("foo %d" % threading.current_thread().ident)

    # 再次启动 Timer 这样就会一直产生 Timer 事件了
    threading.Timer(1, foo).start()

threading.Timer(1, foo).start()

time.sleep(3.5)

print("main %d" % threading.current_thread().ident)

# 输出
# foo 11100
# foo 9008
# foo 14300
# main 13260
```

### Barrier 

不是很懂。。。


## Queue


## GIL


## 线程池


## 参考

[理解Python并发编程一篇就够了 - 线程篇](http://www.dongwm.com/archives/%E4%BD%BF%E7%94%A8Python%E8%BF%9B%E8%A1%8C%E5%B9%B6%E5%8F%91%E7%BC%96%E7%A8%8B-%E7%BA%BF%E7%A8%8B%E7%AF%87/)




