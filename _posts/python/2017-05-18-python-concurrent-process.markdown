---
layout: post
title:  "Python 并发编程——进程"
date:   2017-05-18 19:36:30 +0800
categories: python
---

* TOC
{:toc}


本文基于 python3

python 提供了标准模块 `multiprocessing` 来对多进程提供支持。

`multiprocessing` 中提供了类似于 `threading` 模块的接口，可以对子进程进行操作。此外 `multiprocessing` 中还包含了一些 `threading` 中所不具有的功能，例如 `multiprocessing.Pool` 对象可以创建一个进程池，利用它可以对一些数据进行并行计算。

本文参考自 [官方文档](https://docs.python.org/3/library/multiprocessing.html)

## Process 进程对象

在 `multiprocessing` 模块中，创建子进程的方式是创建一个 `Process` 对象，然后调用它的 `start()` 方法：

```py
from multiprocessing import Process

def foo():
    print("foo")

if __name__ == "__main__":
    p = Process(target=foo)
    p.start()
    p.join()
```

我们看看 Process 中提供了那些 API:

```py
# 构造方法
multiprocessing.Process(group=None, target=None, name=None, args=(), kwargs={}, *, daemon=None)

# 启动该进程
start()

# 默认提供的进程入口函数，如果在构造时传入了 target, 那么 run 方法将被赋值为 target 中的函数
# 你可以在子类中重写 run 方法，这是设定 Process 入口函数的另一种方法
run()

# 结束该进程
terminate()

# 等待该进程对象所指的进程执行结束
join(timeout = None)

# 进程名，可以用属性的方式来访问
name

# 进程 ID, 可以用属性的方式来访问
pid

# 进程是否活着
is_alive()

# 进程是否是 daemon 进程，python 进程退出的时候会结束所有的 daemon 子进程
daemon 

# 进程的退出码
exitcode

# 不知道干嘛用的
authkey

# 不知道干嘛用的
sentinel
```

可以看到 `Process` 和 `Thread` 都有 `start()`, `run()`, `join()` 这些方法，此外 `Process` 提供了 `exitcode`, `authkey`, `sentinel` 这些属性。


## 进程间数据交换

`multiprocessing` 支持两种进程间数据交换的方式： Queues 和 Pipes 。

### Queues

`Queue` 类类似于 `queue.Queue`, 之前我们介绍过，`queue.Queue` 是一个线程安全的队列，可以用于线程间的数据交换。而 `multiprocessing.Queue` 则用于进程间的数据交换。

```py
from multiprocessing import Process, Queue

def f(q):
    q.put([42, None, 'hello'])

if __name__ == '__main__':
    q = Queue()
    p = Process(target=f, args=(q,))
    p.start()
    p.join()
    print(q.get())    # prints "[42, None, 'hello']"
```

如同上面的例子，先把 Queue 作为参数传给新进程，在新进程中向这个队列中 push 内容。

### Pipes

`Pipe()` 函数会返回一对由管道连接的对象(支持双工，也就是双方都可以收发)。你可以用这两个对象来进行数据的发送和接受。

```py
from multiprocessing import Process, Pipe

def f(conn):
    conn.send([42, None, 'hello'])
    conn.close()

if __name__ == '__main__':
    parent_conn, child_conn = Pipe()
    p = Process(target=f, args=(child_conn,))
    p.start()
    print(parent_conn.recv())   # prints "[42, None, 'hello']"
    p.join()
```


## 进程间同步

`multiprocessing` 提供了跟 `threading` 模块中所包含的所有同步支持，包括 `Lock`, `RLock`, `Condition`, `Semaphore`, `Event`, `Timer` 等。

例如：

```py
from multiprocessing import Process, Lock

def f(l, i):
    l.acquire()
    try:
        print('hello world', i)
    finally:
        l.release()

if __name__ == '__main__':
    lock = Lock()

    for num in range(10):
        Process(target=f, args=(lock, num)).start()
```


## 进程间共享数据

对并发编程而言，应当尽量避免使用共享数据。如果你确实需要的话，`multiprocessing` 提供了两种机制来完成进程间数据共享：

### 共享内存

`multiprocessing` 提供的 `Array`, `Value` 对象可以用来存储共享数据：

```py
from multiprocessing import Process, Value, Array

def f(n, a):
    n.value = 3.1415927
    for i in range(len(a)):
        a[i] = -a[i]

if __name__ == '__main__':
    num = Value('d', 0.0)
    arr = Array('i', range(10))

    p = Process(target=f, args=(num, arr))
    p.start()
    p.join()

    print(num.value)
    print(arr[:])
```

初始化 `Value` 对象时传入的 `'d'` 参数，表示后续的数值是 double 类型，初始化 `Array` 对象时传入的 `'i'` 参数，表示后续的数值是 int 类型。

这些共享数据的访问是进程和线程安全的。

`multiprocessing` 中还提供了 `multiprocessing.sharedctypes` 模块，它可以支持在共享内存中创建任意对象 (arbitrary ctypes objects);

### 服务器进程

一个 `multiprocessing.Manager` 对象会控制一个服务器进程，其他进程可以通过代理的方式来访问这个服务器进程。

常见的共享方式有以下几种：

- Namespace。创建一个可分享的命名空间。
- Value/Array。和上面共享ctypes对象的方式一样。
- dict/list。创建一个可分享的dict/list，支持对应数据结构的方法。
- Condition/Event/Lock/Queue/Semaphore。创建一个可分享的对应同步原语的对象。

```py
from multiprocessing import Manager, Process
def modify(ns, lproxy, dproxy):
    ns.a **= 2
    lproxy.extend(['b', 'c'])
    dproxy['b'] = 0

if __name__ == '__main__':
    manager = Manager()
    ns = manager.Namespace()
    ns.a = 1
    lproxy = manager.list()
    lproxy.append('a')
    dproxy = manager.dict()
    dproxy['b'] = 2

    p = Process(target=modify, args=(ns, lproxy, dproxy))
    p.start()
    
    print('PID: %d' % p.pid)
    p.join()
    print ns.a
    print lproxy
    print dproxy
```

进程中的 `Manager` 对象可以被其他网络中的计算机访问，所以它比共享内存的方式要慢一些。


## 进程池

```py
from multiprocessing import Pool, TimeoutError
import time
import os

def f(x):
    return x*x

if __name__ == '__main__':
    # 创建一个有 4 个工作进程的进程池
    with Pool(processes=4) as pool:

        # 让进程池中的方法去计算 f(x)
        # print "[0, 1, 4,..., 81]"
        print(pool.map(f, range(10)))

        # print same numbers in arbitrary order
        for i in pool.imap_unordered(f, range(10)):
            print(i)

        # evaluate "f(20)" asynchronously
        res = pool.apply_async(f, (20,))      # runs in *only* one process
        print(res.get(timeout=1))             # prints "400"

        # evaluate "os.getpid()" asynchronously
        res = pool.apply_async(os.getpid, ()) # runs in *only* one process
        print(res.get(timeout=1))             # prints the PID of that process

        # launching multiple evaluations asynchronously *may* use more processes
        multiple_results = [pool.apply_async(os.getpid, ()) for i in range(4)]
        print([res.get(timeout=1) for res in multiple_results])

        # make a single worker sleep for 10 secs
        res = pool.apply_async(time.sleep, (10,))
        try:
            print(res.get(timeout=1))
        except TimeoutError:
            print("We lacked patience and got a multiprocessing.TimeoutError")

        print("For the moment, the pool remains available for more work")

    # exiting the 'with'-block has stopped the pool
    print("Now the pool is closed and no longer available")
```

只是简单地翻译了一下官方文档中有关 `multiprocessing` 的内容，往后遇到多进程的实战，再去做相关总结会比较好一些。