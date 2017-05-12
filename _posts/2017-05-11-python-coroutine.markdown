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

“协作” 的另外一个含义在于，在协程让出自己的执行权之后，有朝一日它还会再次获取到执行权，继续执行接下来的代码。在这时候，协程必须还保留有原来的上下文 (之前声明的局部变量之类的)。这就要求在切换协程的时候，要对协程的上下文进行暂存和恢复。

更进一步，“协作” 还有另外一层含义，两个协程之间要进行协作，只进行执行权的切换是没啥意思的 (各干各的不叫协作)，它们可以在切换时传递数据。在把控制权交给另一个协程的同时，也把自己的一些数据发过去，让另一个协程帮自己算一算；在收到控制权的时候，同时也收到了处理完的数据。

从上述讨论可以知道：
- 协程可以交出自己的执行权；
- 协程可以重新获取执行权；
- 协程间切换的时候可以进行数据传递；

具体到 python 里面，我们看看 python 协程的语法是怎样的：

```py
def grep(pattern):
    print("Looking for %s" % pattern)

    while True:
        line = (yield)  # yield 表达式，用来交出执行权, yield 返回即重新获取执行权
        if pattern in line:
            print line

g = grep('python')
g.next()                # next() 执行协程
g.send("Yeah, but no, but yeah, but no")    ## send() 执行协程，同时发送数据
g.send("A series of tubes")
g.send("hello python")
```

在 python 中，如果一个函数的定义里面包含了 `yield` 表达式，那么这个函数就会被认为是一个协程 (暂不考虑 generator, 它跟 coroutine 不一样，以后会介绍)。

需要注意的是 `g = grep('python')` 这行代码并不会让协程开始执行，而是返回了一个协程实例，你需要调用 `next()` 才能切换到协程去执行。

- 协程可以通过 `yield` 将自己的执行权交出去，交付的目标是之前调用 `next()` 或 `send()` 方法的那个子任务；
- 你可以调用 `next()` 或 `send()` 方法将执行权交给另一个协程；
- `send()` 方法中的参数将传递给 `yield`；`yield` 后面也可以加上数据，这个数据将作为 `next()` 或 `send()` 的返回值传递出去；

`yield` 是交出，`send()` 是交给，它们虽然都能用于执行权的切换，但切换的方向上有所差别。

## 协程有什么用

跟之前在 lue 里学习协程一样，光看这些定义，真想不出来协程能有什么特别的用处。

搜了一圈别人的说法，其中有种说法 “协程是一种用户态的轻量级线程”，看起来似乎协程跟线程有啥关系，其实不能这么理解。之所以会有这句话，是因为程序对协程的调度有一点像操作系统对线程的调度 —— 就像操作系统通过分配时间片，让一个核的 CPU 可以跑许多个线程一样，程序对协程进行调度 (其实是协程自己主动让出执行权)，使得在一个线程中可以跑多个任务，所以会把协程类比为 “用户态的轻量级线程”。我觉得这种说法不太好，容易让人产生 “协程是线程的一种特殊情况” 这种误解，线程就是线程，不应该发散它的含义 (Windows 平台中的线程对应于 CPU 所支持的 “任务”，是个确实存在的物理概念)。

看了几篇文章都觉得比较含糊，找到一篇 [文章](http://www.dabeaz.com/coroutines/) 写得不错，下面介绍下它的推导过程：

首先，协程可以 消费/处理 一些数据，这跟普通的函数没两样：

```py
# 用协程来处理数据
g = grep('python')
g.next()
g.send("hello python")

# 用函数来处理数据
grep_func("hello python")
```

接着，几个协程可以链式调用，形成一个处理管线 (processing pipelines)：

![]( {{site.url}}/asset/python-coroutine-processing-pipelines.png )

举个代码上的例子：

```py
# 把几个协程通过 send 连起来，形成处理管线

from coroutine import coroutine
import time

# 源头，该函数通过 target 输出 thefile 中包含的日志信息
def follow(thefile, target):
    thefile.seek(0,2)      # Go to the end of the file
    while True:
         line = thefile.readline()
         if not line:
             time.sleep(0.1)    # Sleep briefly
             continue
         target.send(line)

# filter 协程，根据 pattern 过滤发来的数据，如果匹配，则交给 target 继续处理
@coroutine
def grep(pattern,target):
    while True:
        line = (yield)           # Receive a line
        if pattern in line:
            target.send(line)    # Send to next stage

# printer 协程，把数据打印出来
@coroutine
def printer():
    while True:
         line = (yield)
         print(line)

if __name__ == '__main__':
    f = open("access-log")

    # 把 follow, grep, printer 这三者连起来，形成一个处理管线
    follow(f, grep('python', printer()) )
```

上面的 `@coroutine` 装饰器的作用是简化对 `.next()` 的调用，它的实现大概是：

```py
def coroutine(func):
    def start(*args, **kwargs):
        cr = func(*args, **kwargs)
        cr.next()
    return start

# 经过 @coroutine 装饰后，就无需调用 next 了：
@coroutine
def grep(pattern):
    #...

g = grep('python')
g.send('hello python')
```

再看看普通函数怎么才能连起来形成处理管线：

```py
def follow(thefile, callback):
    thefile.seek(0, 2)
    while True:
        line = thefile.readline()
        if not line:
            time.sleep(0.1)
            continue
        callback(line)

def grep(pattern, line, callback):
    if pattern in line:
        callback(line)

def printer(line):
    print(line)

if __name__ == '__main__':
    f = open("access-log")

    def grep_callback(line):
        printer(line)

    def follow_callback(line):
        grep('python', line, grep_callback())

    follow(f, follow_callback)
```

可以看到，把普通函数连起来需要使用 callback, 会麻烦一些。python 对匿名函数的支持不太好，在 lua 里写起来会简单一些：

```
follow(f, function(line)
    grep('python', line, function(line)
        printer(line)
    end)
end)
```