---
layout: post
title:  "Python 协程"
date:   2017-05-11 10:19:30 +0800
categories: python
---

* TOC
{:toc}

最近在尝试使用 python 开发 web app, 接触到了 asyncio 模块的代码，其中有一部分涉及协程 coroutine 知识。之前学 python 的时候这部分没有好好看，因为之前在 lua 里就学过协程，结果实际工作中很少用到。没想到协程这个东西在 python 里其实有许多用处，所以收集点资料总结一下好了。


## 什么是协程

首先协程的英文 coroutine, 应该分解为 co-routine, 前面的 co 表示合作、协作，后面的 routine 表示 例程。

例程这个概念在 Windows 编程里接触过，比如 ReadFileEx 这个 API 的最后一个参数名叫 pfnCompletionRoutine, 在文件读取完毕后，会执行这个例程，这里的例程指的是一个函数。

例程这个词儿听起来怪怪的，我觉得翻译为 “子任务” 更好一些。在不同的环境下，例程、子任务、函数，这三个名词之间似乎有微妙的差别，不过在本文的语境里面，我认为三者所指的都是同一个东西。那么 coroutine 就可以翻译为 **协作的子任务**。

这里的 “协作” 体现在，它会执行一段时间就交出执行权，让其他的任务有机会执行，而不是一直独占着执行权直到任务执行完毕。这个跟操作系统的发展史有点儿类似，上古时期的操作系统只是对硬件做了封装，运行于其中的任务是独占式的，必须在这个任务执行结束之后，其他任务才有机会执行；后来又出现了协作式的操作系统，任务可以在运行期间主动交出执行权，让其他任务有机会能够执行；再后来就是抢占式多任务了，由操作系统为任务分配时间片，跟这儿说的协程就没啥关系了。

注意，协程 (coroutine) 和 线程 (Thread) 没有直接的关系。不过有人将协程类比为 “用户态的线程”， 这是在说它们的调度过程有些相似 —— 就像操作系统通过分配时间片，让一个核的 CPU 可以跑许多个线程一样，程序对协程进行调度 (其实是协程自己主动让出执行权)，使得在一个线程中可以跑多个任务，所以会把协程类比为 “用户态的轻量级线程”。我觉得这种说法不太好，容易让人产生 “协程是线程的一种特殊情况” 这种误解，线程就是线程，不应该发散它的含义 (Windows 平台中的线程对应于 CPU 所支持的 “任务”，是个确实存在的物理概念)。

上面是我对协程的一些初步理解，我在网上搜到一篇 [文章](http://www.dabeaz.com/coroutines/)，写得挺好，下面的内容大体上是这篇文章的翻译。


## python 中的 generator 和 coroutine

python 中的 generator 很容易和 coroutine 相混淆，下面先介绍 generator 然后再介绍 coroutine. 

### generator

generator (生成器)是一种特殊的函数，它能够生成一系列的值 (而不像普通函数那样只能返回一次)。

通常生成器都被用在 for 循环中。

```py
def CountDown(n):
    print("Count down from %d" % n)
    while n > 0:
        yield n
        n -= 1

for i in CountDown(5):
    print(i)

>>> 5 4 3 2 1
```

调用 generator function 时并不会执行这个函数，而是会返回一个 generator object, 你必须调用 `next()` 才能执行它：

```
>>> x = CountDown(10)
>>> x
<generator object at 0x58490>
>>> x.next()
Count down from 10
10
>>>
```

`yield` 操作符将返回一个值，同时 “阻塞” 这个函数，直到下次调用 `next()`。

当 generator return 的时候，生成就结束了，此时调用 `next()` 将触发 StopIteration 异常，for 循环也会终止：

```
>>> x.next()
1
>>> x.next()
Traceback (most recent call last):
    File "<stdin>", line 1, in ?
StopIteration
>>>
```

下面的例子展示了生成器的一个有趣的用法，它会每隔一段时间检查一次日志文件中是否有新行，如果有，就返回新日志：

```py
def follow(theFile):
    theFile.seek(0, 2)
    while True:
        line = theFile.readline()
        if not line:
            time.sleep(0.1)
            continue
        yield line

logfile = open("access-log")
for line in follow(logfile):
    print(line)
```


### coroutine

python 中 coroutine 的语法与 generator 一样，但从本质上讲它们是两个不同的概念。

generator 的作用在于 “产生值”, 而 coroutine 的作用在于 “消费值“：

```py
def grep(pattern):
    print("Looking for %s" % pattern)
    while True:
        line = (yield)
        if pattern in line:
            print(line)

g = grep("python")
g.next()

g.send("Yeah, but no, but yeah, but no")
g.send("A series of tubes")
g.send("python generators rock!")
```

coroutine 通过类似于 `line = (yield)` 这样的表达式来接收 `send()` 给它的值。

跟 generator 一样，coroutine function 不会开始执行，而是返回一个 coroutine object. 

你必须先调用 `next()` 或 `send(None)` 来激活 coroutine 对象，让 coroutine 处于准备接受值的状态 (代码执行到 `line = (yield)`) 那里，这样才能继续调用 `send()` 方法来向 coroutine 发送要消费的值。因为调用 `.next()` 很容易被忘掉，可以用以下的装饰器来装饰 coroutine function:

```py
def coroutine(func):
    def start(*args, **kwargs):
        cr = func(*args, **kwargs)
        cr.next()
        return cr
    return start

@coroutine
def grep(pattern):
    ...
```

coroutine 可能会无限运行，你可以调用 `.close()` 来关闭它。调用 `.close()` 将导致 coroutine 内的 `line = (yield)` 抛出 GeneratorExit 异常：

```py
@coroutine
def grep(pattern):
    print("Look for %s" % pattern)
    try:
        while True:
            line = (yield)
            if pattern in line:
                print(line)
    except GeneratorExit:
        print("Going away. good bye")
```

正如之前所介绍的，尽管 coroutine 和 generator 有类似的语法，但你应该把它们理解成是两个不同的概念。generator 只用来生成循环所需要的数据；coroutine 则是用来消费数据，虽然有时候也会在 coroutine 里通过 yield 返回数据，但它跟迭代或循环没有任何关系。


## coroutines, pipelines, and Dataflow

### processing pipelines (执行管线)

coroutines 可以用来设置执行管线，只需要通过 `.send()` 将数据 push 给下一个 coroutine 就可以将多个 coroutine 连成序列：

![]( {{site.url}}/asset/python-coroutine-processing-pipelines.png )

```py

from coroutine import coroutine
import time

# source
def follow(thefile, target):
    thefile.seek(0,2)      # Go to the end of the file
    while True:
         line = thefile.readline()
         if not line:
             time.sleep(0.1)    # Sleep briefly
             continue
         target.send(line)

# A filter.
@coroutine
def grep(pattern,target):
    while True:
        line = (yield)           # Receive a line
        if pattern in line:
            target.send(line)    # Send to next stage

# A sink.  A coroutine that receives data
@coroutine
def printer():
    while True:
         line = (yield)
         print line,

# Example use
if __name__ == '__main__':
    f = open("access-log")
    follow(f,
           grep('python',
           printer()))
```

入上述代码所示，`follow()` 是这个链条的源头，它将读取到的日志 `send()` 给 `grep()` 进行进一步的处理，`grep()` 对日志进行过滤后，发送给了 `printer()` 来完成日志打印。

### Being Branchy (变为分支)

通过 coroutine, 你可以把数据发送给多个目标：

![]( {{site.url}}/asset/python-coroutine-being-branchy.png )

例如下面的代码将数据广播给多个 coroutine 对象：

```py
@coroutine
def broadcast(targets):
    while true:
        item = (yield)
        for target in targets:
            target.send(item)

f = open("access-log")
follow(f, broadcast([grep("python", printer()),
                     grep("ply", printer()),
                     grep("swig", printer())]))
```

![]( {{site.url}}/asset/python-coroutine-being-branchy-example.png )

从上面的例子可以看出，coroutine 可以非常方便地实现数据处理组建的连接，假如你有许多个简单的数据处理组件，你可以通过 `.send()` 将它们连接成 pipes, branches, merging 等等形式。

### Coroutines vs. Objects

coroutine 有点儿像面向对象中的 handler 对象：

```py
## OO 版本
class GrepHandler(object):
    def __init__(self, pattern, target):
        self.pattern = pattern
        self.target = target
    def send(self, line):
        if self.pattern in line:
            self.target.send(line)

## coroutine 版本
@coroutine
def grep(pattern, target):
    while True:
        line = (yield)
        if pattern in line:
            target.send(line)
```

但 coroutine 更加简单，毕竟它只需要一个函数定义就可以完成。


## Coroutines and Event Dispatching

coroutine 可用来编写事件处理器，例如以一个 xml parser 为例：

```py
import xml.sax

class MyHandler(xml.sax.ContentHandler):
    def startElement(self, name, attrs):
        print("startElement " + name)
    def endElement(self, name):
        print("endElement " + name)
    def characters(self, text):
        print("characters " + repr(text)[:40])

xml.sax.parse("somefile.xml", MyHandler())
```

上面使用了 SAX 这种解析 xml 的方式，它使用了事件驱动的方式来暴露对外接口，这种方式可以用较小内存解析尺寸比较大的 xml 文件，缺点是代码编写起来有点麻烦。

上面的例子只是把事件及其参数打印出来，你可以把这些数据发送给 coroutine 来处理：

```py
import xml.sax

class MyHandler(xml.sax.ContentHandler):
    def __init__(self, target):
        self.target = target
    def startElement(self, name, attrs):
        self.target.send( ('start', (name, attrs._attrs)) )
    def endElement(self, name):
        self.target.send( ('end', name) )
    def characters(self, text):
        self.target.send( ('text', text) )
```

![]( {{site.url}}/asset/python-coroutine-event-dispatch.png )

接着看看怎么编写 coroutine, 假设现在的需求是把 xml 中的 bus 标签转换成字典形式：

![]( {{site.url}}/asset/python-coroutine-bus-to-dict.png )

那么可以编写下述 coroutine 来处理传入的事件：

```py
@coroutine
def buses_to_dicts(target):
    while True:
        event, value = (yield)

        if event == 'start' and value[0] == 'bus':
            busdict = {}
            fragments = []

            while True:
                event, value = (yield)
                if event == 'start' : 
                    fragments = []
                elif event == 'text':
                    fragments.append(value)
                elif event == 'end':
                    if value != 'bus':
                        busdict[value] = "".join(fragments)
                    else:
                        target.send(busdict)
                        break
```

可以看到上面的代码实现了一个简单的状态机：

![]( {{site.url}}/asset/python-coroutine-state-machines.png )

- **State A** : 寻找 bus 标签；
- **State B** : 记录 bus 标签的属性；

可以看到由于 coroutine 会记录代码运行的状态，因此它非常适合用来实现状态机。


## From Data Processing to Concurrent Programming

根据之前的介绍，我们有如下了解：
- coroutine 跟 generator 类似，但他们是两个不同的概念；
- 你可以将一系列小的处理组件连接起来, (通过 `.send()`)；
- 你可以通过设置 pipelines, dataflow graphs 等来处理数据；

### 基本的并发 Concurrency

如下图所示，你可以引入额外的层，来把 coroutine 打包到线程或子进程中：

![]( {{site.url}}/asset/python-coroutine-package-coroutine-inside-thread.png )

如上图所示，用 `queue` 来代替 `.send()` 可以将数据发送给其他线程中的 coroutine, 用 `pipe` 来代替 `.send()` 可以将数据发送给其它子进程中的 coroutine.

下面的例子展示了将数据发送给 Thread 的用法：

```py
# 新开一个线程，在该线程中执行 target coroutine
@coroutine
def threaded(target):
    # 创建消息队列
    messages = Queue()

    # 创建并运行一个线程，该线程不断从消息队列中取出消息，然后交给 target coroutine 去执行
    def run_target():
        while True:
            item = message.get()
            if item is GeneratorExit:
                target.close()
                return
            else:
                target.send(item)
    Thread(target=run_target).start()
    
    # 向消息队列中 put 消息，线程随后会从消息队列中取出消息并交给 target coroutine 去执行
    try:
        while True:
            item = (yield)
            message.put(item)
    except GeneratorExit:
        message.put(GeneratorExit)

xml.sax.parse("allroutes.xml", EventHandler(
    buses_to_dicts(
        threaded(
            filter_on_field("route", "22", 
                filter_on_field("direction", "North Bound", 
                    bus_locations()
                )
            )
        )
    )
))
```

![]( {{site.url}}/asset/python-coroutine-thread-target.png )

下面的例子展示了将数据发送给 sub-process 的用法：

```py
@coroutine
def sendto(f):
    try:
        while True:
            item = (yield)
            pickle.dump(item, f)
            f.flush()
    except StopIteration:
        f.close()

def recvfrom(f):
    try:
        while True:
            item = pickle.load(f)
            target.send(item)
    except EOFError:
        target.close()
```

![]( {{site.url}}/asset/python-coroutine-sub-process-target.png )

通过上述例子可以看到，coroutine 可以把一个任务的具体实现与其执行环境隔离开来。例如其中的 `filter_on_field()` 任务，它本身作为一个 coroutine 可以在另一个线程或子进程中执行；

但要注意的是，上述做法很可能会让你的程序变慢！据原作者说可能会慢 50% !