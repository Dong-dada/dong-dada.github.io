---
layout: post
title:  "Python 并发编程——asyncio"
date:   2017-05-20 18:05:05 +0800
categories: python
---

* TOC
{:toc}

本文基于 python 3

本文参考自：
- [A Web Crawler With asyncio Coroutines](http://aosabook.org/en/500L/a-web-crawler-with-asyncio-coroutines.html)
- [A Web Crawler With asyncio Coroutines 译文](http://ls-a.me/translation/crawler/)
- [500lines crawler](https://github.com/aosabook/500lines/tree/master/crawler)
- [asyncio](https://docs.python.org/3/library/asyncio.html)
- [单台服务器并发 TCP 连接数到底可以有多少](http://www.52im.net/thread-561-1-1.html)
- [高性能网络编程(二)：上一个10年，著名的C10K并发连接问题](http://www.52im.net/thread-566-1-1.html)
- [什么是 yield from](http://blog.csdn.net/u010161379/article/details/51645264)

## asyncio 解决了什么问题

python 3 中的 `asyncio` 模块提供了一系列基础设施，用于编写 **单线程**, 支持 **I/O 多路复用** 的代码，你可以用它来完成 socket 访问，客户端或服务端网络编程等工作。

### I/O 多路复用

先解释一下什么是 I/O 多路复用。在网络编程中，有时候需要在一台机器上同时建立多个连接，这方面我还没什么接触，就瞎想一个例子吧，比如游戏服务器里需要支持 1000 个玩家在线，那么就需要建立 1000 条连接。如果是用阻塞的 socket 来建立这种连接，一个线程显然不行，需要建立 1000 个线程才行，这种做法很耗资源，如果玩家数量变成 10000 个，对机器的要求就很高了。而 I/O 多路复用就能解决这个问题，它是操作系统提供的一种支持，你把文件描述符告诉操作系统，操作系统会在 I/O 操作完成后通知你，这样一来只需要一个线程就可以处理多个 I/O 请求。这也就是多路复用的意思。

现在对网络编程还没啥经验，上面只是我现有的理解，可能理解有偏差。

I/O 多路复用技术经常跟 Event Loop 联系在一起。类似于下面的伪代码：

```c
while (!done)
{
    int timeout_ms = max(1000, GetNextTimedCallback());
    int retval = ::poll(fds, ndfs, timeout_ms); 	// poll 将等待多个 文件描述符，直到其中某个可用
    if (retval < 0)
    {
        // 处理错误，回调用户的 error handler
    }
    else
    {
        // 处理到期的 timers, 回调用户的 timer handler
        if (retval > 0)
        {
            // 处理 IO 事件，回调用户的 IO event handler
        }
    }
}
```

可以看到上面的逻辑非常简单，就是用一个 while 循环不断等待 `poll` 返回(代表有连接处于可用状态)，返回后就调用对应的 handler 去处理该连接。

### python 的 selector

上文提到的 `::poll` 是 linux 提供的一个系统函数，你把一系列 文件描述符 作为参数传给他，它会阻塞住，直到某个文件描述符变为可用，或者超时。事实上很久以前 Unix 提供了一个 select 函数，poll 函数是它的改进版，性能更好。

python 中提供了 selector 来实现类似的功能，请看如下代码：

```py
#coding:utf-8

from selectors import DefaultSelector, EVENT_WRITE
import socket

selector = DefaultSelector()
stopped = False

# 创建 socket 对象，并设置为非阻塞模式
sock = socket.socket()
sock.setblocking(False)

# 开启连接
try:
    sock.connect(("www.baidu.com", 80))
except BlockingIOError:
    pass

# 连接后的回调函数
def connected():
    print("connected!")
    selector.unregister(sock.fileno())

    global stopped
    stopped = True

# 把 sock 的文件描述符注册给 selector
selector.register(sock.fileno(), EVENT_WRITE, connected)

# 事件循环
while not stopped:
    events = selector.select()
    for event_key, event_mask in events:
        callback = event_key.data
        callback()
```

相比于原先的 Event Loop 的实例代码，这里增加了一个回调函数的逻辑，便于将 selector 返回的事件分发给对应的 handler 函数。

上述代码中的 stopped 全局变量用于标识是否结束，这里我们的处理比较简单，连接成功后就可以退出事件循环了。如果不设置这个标记的话，`selector.select()` 会在 connected 注销后因为没有注册的文件描述符而抛出异常。

### callback 编程模型的困难

基于 callback 的编程是一种很常见的手法，然而当嵌套层次比较深的时候编写这样的代码会很困难。

例如编写一个爬虫代码，`socket.connect()` 只是第一步。随后我们还得向服务器发送一个 GET 请求，为了避免这一步被阻塞，我们得注册另外一个回调函数。GET 之后，它不能一次读取完所有请求，所以还得再一次注册，如此反复。

下面是基于 callback 方式编写的爬虫代码：

```py
#coding:utf-8

from selectors import DefaultSelector, EVENT_WRITE, EVENT_READ
import socket

selectors = DefaultSelector()
urls_todo = set([])
seen_urls = set([])
stopped = False

class Fetcher:
    def __init__(self, url):
        self.response = b''
        self.url = url
        self.sock = None
        urls_todo.add(url)

    def fetch(self):
        # 先发起连接
        self.sock = socket.socket()
        self.sock.setblocking(False)

        try:
            self.sock.connect(("www.baidu.com", 80))
        except BlockingIOError:
            pass
        
        selectors.register(self.sock.fileno(), EVENT_WRITE, self.connected)

    def connected(self, key, mask):
        # 连接完成后发起请求来抓取页面
        print('connected!')
        selectors.unregister(key.fd)
        
        request = "GET {} HTTP/1.0\r\nHost: www.baidu.com\r\n\r\n".format(self.url)
        self.sock.send(request.encode("ascii"))

        selectors.register(key.fd, EVENT_READ, self.read_response)

    def read_response(self, key, mask):
        global stopped

        chunk = self.sock.recv(4096)
        if chunk:
            self.response += chunk
        else:
            print("read_response finish!\n" + self.response.decode())
            selectors.unregister(key.fd)
            
            # 该页面抓取完成后，分析页面中的其它链接，并创建一个新的 Fetcher 来抓取这些链接
            links = self.parse_links()
            for link in links.difference(seen_urls):
                urls_todo.add(link)
                Fetcher(link).fetch()
            
            seen_urls.update(links)
            urls_todo.remove(self.url)
            if not urls_todo:
                stopped = True

    def parse_links(self):
        return set([])

# 事件循环
def loop():
    while not stopped:
        events = selectors.select()
        for event_key, event_mask in events:
            callback = event_key.data
            callback(event_key, event_mask)

# 开始抓取 www.baidu.com/duty/ 页面
fetcher = Fetcher('/duty/')
fetcher.fetch()

# 启动事件循环
loop()
```

上面的例子暴露出了异步编程的一大问题：意大利面代码。

为了接受异步事件，你不得不编写许多回调函数。同时还需要在类成员中记录状态信息，也就是 url, sock, response 这些数据。而如果我们用同步代码来完成上述逻辑，那么将非常简单直白：

```py
#coding:utf-8

import socket

urls_todo = set([])

def parse_links(response):
    return set([])

def fetch(url):
    # 创建 socket 并连接
    sock = socket.socket()
    sock.connect(('www.baidu.com', 80))

    # 发起请求
    request = 'GET {} HTTP/1.0\r\nHost: www.baidu.com\r\n\r\n'.format(url)
    sock.send(request.encode('ascii'))

    # 接收数据
    response = b''
    chunk = sock.recv(4096)
    while chunk:
        response += chunk
        chunk = sock.recv(4096)
    print(response.decode())

    # 处理数据
    links = parse_links(response)
    for link in links:
        urls_todo.add(link)

    urls_todo.remove(url)

while not urls_todo:
    url = '/duty/'
    urls_todo.add(url)
    fetch(url)
```

上面的同步代码无需保存 url, sock, response 这些数据，这是因为这些数据被保存到了函数的栈上，在 socket 操作结束后，会继续执行对应的处理代码，我们认为这样的代码具有“连续性”，这种连续性是由编译器(解释器)来实现的。而回调函数的栈是分离的，没有连续性。

在我们的异步代码中，我们把 url, sock, response 这些变量存储到了 Fetcher 对象里面，相当于解释器把局部变量存储到了栈上。在代码中注册了 `connected` 和 `read_response` 回调函数，相当于解释器的指令指针。只不过这些代码都是我们手动实现的，不像同步代码那样由编译器或解释器来帮助我们实现。

另外，异步代码这种模型还有一个缺点，假如发生了异常，因为栈信息不连续，我们无法在异常信息里看到栈回溯信息 —— 最多只能看到 event loop 调用了它，不知道是谁发起的调用，很难定位到问题。

### Coroutines 异步代码看起来像同步一样

协程是 python 提供的一种机制，简单来说协程就是 "协作的子程序", 跟一般的函数不一样，协程不是一下就执行到底，而是可以中断后继续执行：

```py
def foo():
    # ...
    yield	# 中断执行，让其他协程有机会运行
    # ...
    yield   # 中断执行，让其他协程有机会运行
    # ...
```

python 3.4 的 asyncio 模块里有一个叫 aiohttp 的包，它利用协程这一机制，可以把异步操作写得跟同步一样：

```py
@asyncio.coroutine
def fetch(self, url):
    response = yield from self.session.get(url)
    body = yield from response.read()
```

看起来是不是很简单？相对于线程而言，协程的开销也很小，python 中每个线程需要占用 50k 内存，而协程只需要 3k。


## 协程是如何工作的

在理解协程如何工作之前，你需要知道普通的 python 函数是如何工作的。

### python 的函数栈在堆中分配

标准的 Python 解释器是 C 语言编写的。一个 python 函数对应的 C 函数是 `PyEval_EvalFrameEx`。这个函数会获得一个 python 栈帧结构，然后在这个栈帧的上下文中执行 python 子节码。

以下是函数 foo 和它的子节码：

```py
import dis

def foo():
    bar()

def bar():
    pass

dis.dis(foo)

# 输出
#  2           0 LOAD_GLOBAL              0 (bar)
#              3 CALL_FUNCTION            0 (0 positional, 0 keyword pair)
#              6 POP_TOP
#              7 LOAD_CONST               0 (None)
#             10 RETURN_VALUE
```

- 第 1 行，foo 函数在它的栈中加载 bar。
- 第 2 行，调用 bar 函数；
- 第 3 行，把 bar 函数的返回值从栈中弹出；
- 第 4 行，加载 None 值；
- 第 5 行，返回 None 值；

注意第 2 行 `CALL_FUNCTION` 这个子节码，它会创建一个新的栈帧，并利用这个栈帧递归地调用 `PyEval_EvalFrameEx` 来执行 bar 函数的子节码。如果你对 C 语言中函数调用栈帧的概念有了解，你会发现 python 实现函数调用的方法与 C 语言是一样的，都是利用了栈。

然而，跟 C 语言中的函数栈不一样的是，**python 中的栈是在堆中分配的**。你可以把 python 的函数栈理解成是在堆中 new 出来的一块内存，这块内存可以留着不释放。这跟 C 语言的函数栈不一样，C 的函数栈是通过栈顶指针来控制栈大小的，退出函数的时候栈空间一定会被释放掉。

下面的代码通过 `inspect` 模块来获取到 bar 函数的栈帧，然后打印它的两个成员 `f_code` 和 `f_back`：

```py
>>> import inspect
>>> frame = None
>>> def foo():
...     bar()
...
>>> def bar():
...     global frame
...     frame = inspect.currentframe()
...
>>> foo()
>>> # The frame was executing the code for 'bar'.
>>> frame.f_code.co_name
'bar'
>>> # Its back pointer refers to the frame for 'foo'.
>>> caller_frame = frame.f_back
>>> caller_frame.f_code.co_name
'foo'
```

在 C 语言编写的 python 解释器中，用 PyFrameObject 来表示一个函数的栈帧，它的 `f_back` 成员指向调用者的 PyFrameObject 对象， `f_code` 指向该栈帧的子节码：

![]( {{site.url}}/asset/python-asyncio-pyframeobject.png )

如上图所示，PyCodeObject 代表了函数的字节码，相当于 C 语言中一个函数编译后的二进制代码；而 PyFrameObject 代表了该函数的栈帧，相当于 C 语言在运行时动态产生的函数栈。PyCodeObject 必须利用 PyFrameObject 才能正确运行，因为 PyFrameObject 中包含了 PyCodeObject 中所需要使用的参数、局部变量、返回值等内容。

### python 生成器的原理

```py
>>> def gen_fn():
...     result = yield 1
...     print('result of yield: {}'.format(result))
...     result2 = yield 2
...     print('result of 2nd yield: {}'.format(result2))
...     return 'done'
...     
```

python 在把上述 `gen_fn` 编译成字节码的过程中，一旦它看到 yield 语句就知道这是一个生成器函数，而不是普通的函数，它会设置一个标记记录在 `gen_fn.__code__.co_flags` 变量中，你可以通过如下代码来验证该标记：

```py
>>> # The generator flag is bit position 5.
>>> generator_bit = 1 << 5
>>> bool(gen_fn.__code__.co_flags & generator_bit)
True
```

当你调用 `gen_fn` 时，python 看到这个标记，就不会运行它而是创建一个生成器：

```py
>>> gen = gen_fn()
>>> type(gen)
<class 'generator'>
```

创建的生成器在 C 中用 PyGenObject 来表示，它包含了一个 PyFrameObject 对象，和一个 PyCodeObject 对象：

![]( {{site.url}}/asset/python-asyncio-pygenobject.png )

如果多次调用 `gen_fn` 来创建新的生成器，那么每个生成器都指向同一个 PyCodeObject 对象，但每个生成器都指向各自的 PyFrameObject 对象。在生成器函数彻底执行结束前，PyFrameObject 将一直存在，换句话说，PyCodeObject 所需要的局部变量、参数、返回值等信息将一直保存在生成器的 PyFrameObject 对象中。

PyFrameObject 中有一个成员 `f_lasti` 记录了当前指令指针的位置，类似于汇编中的 EIP 指针。每次调用 yield 的时候，`f_lasti` 都会指向最后运行的那行字节码，下次调用 send 就可以根据 `f_lasti` 的值来恢复到上次运行的位置。

正因为这样，生成器可以在任何时候，任何函数中恢复运行。

### 通过生成器来优化基于 callback 的意大利面代码

正如本文最开始介绍的，要使用 I/O 多路复用，需要有一个 event loop:

```py
def loop():
    while not stopped:
        events = selector.select()
        for event_key, event_mask in events:
            callback = event_key.data
            callback(event_key, event_mask)
```

在之前的做法中，每当 selector 有返回(代表连接可用)，就调用对应的 callback。因为向 selector 注册的事件类型有好几种 (连接完毕、接收到数据等), 我们不得不为每种事件都设置对应的 callback 函数，并且需要通过类封装来记录中间状态。

现在有了生成器，它可以被暂停，被恢复，并且在暂停和恢复期间中间状态会保存到栈中。根据这些特点，我们可以用一个生成器来完成之前 Fetcher 的工作：
- 每次 `selector.select()` 返回后，都将事件分发到同一个生成器上，分发的方式是调用生成器的 `send()` 函数来恢复生成器的执行；
- 生成器中通过 yield 将操作分为多个步骤，例如 connect 完成后 yield, 接着执行 request, 完成后 yield, 接着完成解析，对于解析后的链接，创建新的生成器，交给 event loop 继续分发；

按照上述思路可以编写出下面的代码：

```py
#coding:utf-8

from selectors import DefaultSelector, EVENT_WRITE, EVENT_READ
import socket

selector = DefaultSelector()
stopped = False

the_gen = None
undo_urls = set([])
seen_urls = set([])

def parse_links(response):
    return set(['/duty/baozhang.html'])

def fetch(url):
    
    # 创建 socket 并连接
    sock = socket.socket()
    sock.setblocking(False)

    try:
        sock.connect(("www.baidu.com", 80))
    except BlockingIOError:
        pass

    # 连接完成后回调此方法，在这里调用生成器的 send 方法，让生成器继续执行
    def connect_finish():
        try:
            global the_gen
            the_gen.send(None)
        except StopIteration:
            pass

    selector.register(sock.fileno(), EVENT_WRITE, connect_finish)

    # 返回
    yield

    print("connected!")
    selector.unregister(sock.fileno())

    request = "GET {} HTTP/1.0\r\nHost: www.baidu.com\r\n\r\n".format(url)
    sock.send(request.encode("ascii"))

    # 请求完成后回调此方法，在这里调用生成器的 send 方法，让生成器继续执行
    def request_finish():
        try:
            global the_gen
            the_gen.send(None)
        except StopIteration:
            pass
    selector.register(sock.fileno(), EVENT_READ, request_finish)

    # 返回
    yield

    response = b''

    while True:
        chunk = sock.recv(4096)
        if chunk:
            response += chunk
            # 返回
            yield
        else:
            break

    print('response: \n' + response.decode())
    selector.unregister(sock.fileno())

    undo_urls.remove(url)
    seen_urls.add(url)

    links = parse_links(response)
    for link in links.difference(seen_urls):
        # 创建一个新的生成器来处理解析出的链接
        undo_urls.add(link)
        global the_gen
        the_gen = fetch(link)
        the_gen.send(None)

    if not undo_urls:
        global stopped
        stopped = True

def loop():
    while not stopped:
        events = selector.select()
        for event_key, event_mask in events:
            callback = event_key.data
            callback()

# 启动生成器
undo_urls.add('/duty/')
the_gen = fetch('/duty/')
the_gen.send(None)

# 启动事件循环
loop()
```

上面的代码看起来很长，但是没有了 Fetcher 类 (因为无需记录中间状态)，没有了多个回调函数，如果忽略 `yield`, `register`, `unregister` 等代码，fetch 函数就像是编写同步代码一样。

### 另一种写法

上一小节的代码是我根据自己的理解写的，比较直白，就是将 select 的事件通过 send 方法分发到对应的生成器上。[原文](http://ls-a.me/translation/crawler/#building-coroutines-with-generators) 的写法不太一样。

对比一下代码，发现我的代码里有一个不好的地方，`the_gen` 是一个全局变量。之所以要有这个全局变量，是因为在生成器内部无法直接获取到生成器本身，所以不得不通过全局变量来记录它，这样才能在 `connect_finish` 回调中调用它。

原文首先实现了一个 Future 类，这个类用来表示协程正在等待的事件，每次协程进入等待的时候，都会通过 `yield future` 来返回一个新的 Future 对象，外部可以通过向这个对象注册回调函数的方式来接收到事件触发的通知：

```py
class Future:
    def __init__(self):
        self.result = None
        self._callbacks = []

    def add_done_callback(self, fn):
        self._callbacks.append(fn)

    def set_result(self, result):
        self.result = result
        for fn in self._callbacks:
            fn(self)
```

协程内部收到事件后，会调用 `set_result` 来通知外部：

```py
def fetch():
    sock = socket.socket()
    sock.setblocking(False)
    try:
        sock.connect(('xkcd.com', 80))
    except BlockingIOError:
        pass

    Future future

    def connect_finish():
        # 这里不再直接调用 send, 而是通过 future 把事件告诉外部，由外部去调用 send
        future.set_result(None)
    
    selector.register(sock.fileno(), EVENT_WRITE, connect_finish)

    # 把 future 传给外部，外部可以借由它来得到事件通知
    yield future

    selector.unregister(sock.fileno())
    print('connected!')

    # ...
```

另外还需要定义一个 Task 类，这个类接收协程返回的 Future 对象，监听 Future 对象发来的事件，并调用协程的 send 方法来驱动协程继续运行：

```py
class Task:
    def __init__(self, coro):
        self.coro = coro
        self.step(None)

    def step(self, future):
        try:
            # 驱动协程运行到下一条 yield 的位置
            # 协程会返回一个新的 future 对象，以便 Task 对这个对象进行监听
            result = (future and future.result) or None
            next_future = self.coro.send(result)
        except StopIteration:
            return
        
        # 监听 future 对象，收到事件后调用 step 继续驱动协程
        next_future.add_done_callback(self.step)

# 通过 Task 类来驱动协程运行
Task(fetch('/duty/'))

loop()
```

可以看到上述写法中无需使用全局变量来记录生成器，因为调用生成器、监听事件的代码都放到了 Task 类中。

需要注意的是，协程每次调用 `yield future` 都应该返回一个新的 future 对象，因为我们设计的 Future 对象没有 `remove_done_callback()` 类似的接口，不能重复使用。

### 通过 yield from 提取子协程

`yield from` 语法可以让一个生成器委托另一个生成器。

例如下面这个简单的生成器：

```py
def gen_fn():
    result = yield 1
    print('result of yield: {}'.format(result))
    resutl2 = yield 2
    print('result of 2nd yield: {}'.format(result2))
    return 'done'
```

通过 `yield from`, 你可以在另一个生成器中调用它：

```py
def caller_fn():
    gen = gen_fn()
    rv = yield from gen
    print("return value of yield-from: {}".format(rv))

caller = caller_fn()
```

`yield from gen` 将把对 `caller_fn` 的 `send()` 调用委托给 `gen_fn` 来执行：

```py
caller.send(None)
# 返回 1

caller.gi_frame.f_lasti
# 返回 15

caller.send('hello')
# 输出 result of yield: hello
# 返回 2

caller.gi_frame.f_lasti  # 注意 caller_fn 一直停留在 yield from gen 这个语句处
# 返回 15
```

`caller_fn()` 执行到 `yield from gen` 之后，会一直停在这里，直到 `gen_fn` 这个生成器进行了 `return` 操作，`gen_fn` 的返回值 `'done'` 就是 `yield from gen` 的返回值 `rv`：

```py
# 继续调用 send, gen_fn 返回，caller_fn 的 yield from gen 语句也返回
caller.send('goodbye')

# 输出 result of 2nd yield: goodbye
# 输出 return value of yield-from: done
#
# Traceback (most recent call last):
#  File "<input>", line 1, in <module>
# StopIteration
```

可以看到 `yield from` 就像是一个管道，把对 `caller_fn` 的 `send` 调用都转给了 `gen_fn`, 同时 `gen_fn` 的 `yield` 操作也都转发给了外部。

借由 `yield from`, 我们可以像提取子函数那样提取子协程 —— 调用方将一直停在 `yield from` 语句上，直到子协程返回，并且调用方的 `send` 都会转发给子协程去处理。

对于我们的 fetch 函数：

```py
def fetch(url):
    # ... connection logic from above, then:
    sock.send(request.encode('ascii'))

    while True:
        f = Future()

        def on_readable():
            f.set_result(sock.recv(4096))

        selector.register(sock.fileno(),
                            EVENT_READ,
                            on_readable)
        chunk = yield f
        selector.unregister(sock.fileno())
        if chunk:
            self.response += chunk
        else:
            # Done reading.
            break
```

我们先提取一个读取一块数据的子协程 `read`：

```py
def read(sock):
    f = Future()

    def on_readable():
        f.set_result(sock.recv(4096))
    
    selector.register(sock.fileno(), EVENT_READ, on_readable)

    chunk = yield f

    selector.unregister(sock.fileno())
    return chunk
```

在 `read` 的基础上，`read_all` 读取整个信息：

```py
def read_all(sock):
    response = []

    chunk = yield from read(sock)

    while chunk:
        response.append(chunk)
        chunk = yield from read(sock)
    
    return b''.join(response)
``` 

对 `read_all` 调用 `send` 相当于对 `read` 调用，这时 `read` 会返回一个 future, 跟我们之前的逻辑一样，交给外部的 task 去监听。

换一个角度看上面的代码，它们看起来就像是在做阻塞 I/O 的普通函数一样。

现在，我们可以在 fetch 中直接调用 `read_all` ：

```py
def fetch(url):
    # ...

    sock.send(request.encode('ascii'))
    response = yield from read_all(sock)
```

### 这就是 asyncio

我们已经完成了对asyncio协程探索。我们深入观察了生成器的机制，实现了简单的future和task。我们指出协程是如何利用两个世界的优点：比线程高效，比回调清晰。当然真正的asyncio比我们这个简化版本要复杂的多。真正的框架需要处理 zero-copy I/O, 公平调度，异常处理和其他大量特性。

使用asyncio编写协程代码比你现在看到的要简单的多。在前面的代码中，我们从基本原理去实现协程，所以你看到了回调，task和future，甚至非阻塞套接字和select调用。但是当用asyncio编写应用，这些都不会出现在你的代码中。我们承诺过，你可以像这样下载一个网页：

```py
@asyncio.coroutine
def fetch(self, url):
    response = yield from self.session.get(url)
    body = yield from response.read()
```


## 使用 asyncio 写一个网络爬虫

[500lines](https://github.com/aosabook/500lines) 中有一个 crawl 项目，利用 asyncio, aiohttp 模块编写了一个网络爬虫。也正是本文要分析的对象。

### asyncio 的基本使用

在开始编写爬虫前，必须先简单介绍一下 asyncio 库的使用方法，一边写一边介绍不太好。

正如我们之前介绍的内容，通过 yield from 语法可以委托一个生成器来完成某项操作，因此我们可以像封装子函数那样封装子协程。asyncio 库主要被设计用来完成网络 IO 操作，它提供的功能大多是通过子协程的方式来封装的。下述代码使用 asyncio, aiohttp 库实现了拉取指定网页的功能：

```py
#coding:utf-8

import asyncio
import aiohttp

# async def 用于声明一个协程
async def Fetch(url):
    loop = asyncio.get_event_loop()

    # 创建一个 session, 后续进行 http 请求都使用这个 session 来进行
    session = aiohttp.ClientSession(loop=loop)

    # 发起 GET 请求
    response = await session.get(url)

    # 读取 body
    body = await response.read()

    # 退出前释放 response
    await response.release()

    # 退出前关闭 session
    session.close()

    return body

# 在事件循环中运行 Fetch 协程
loop = asyncio.get_event_loop()
loop.run_until_complete(Fetch("http://baike.baidu.com/item/python"))
```

上述代码中的 `async def` 关键字用于声明一个协程，`await` 关键字等价于 `yield from` 也就是调用子协程。可以看到代码中所使用的 asyncio, aiohttp 模块的功能全部是使用协程而非子函数来封装的，必须通过 `await` 来调用。这是 asyncio 库有别于普通库的一大特点。

注意上面代码中并没有我们之前提到过的 Future, Task 类，实际上 `session.get()`, `response.read()` 子协程在执行过程中会对外抛出 Future 对象，无须我们自己抛出 Future。

### asyncio.Task

上一小节的代码只能获取一个网页，效果跟同步代码没啥区别。我们希望能够同时抓取多个网页，这时候可以使用 asyncio.Task 对象：

```py
import asyncio

async def Work():
    LOGGER.debug("Work Enter")
    # ...

async def Main():
    loop = asyncio.get_event_loop()
    workers = [asyncio.Task(Work(), loop=loop) for _ in range(10)]
    await asyncio.sleep(1)

    for worker in workers:
        worker.cancel()

loop = asyncio.get_event_loop()
loop.run_until_complete(Main())
```

上述代码中创建了 10 个 asyncio.Task 对象，这意味着 10 个 Work 子协程 “同时” 运行。这里的同时指的是，当某个 Work 协程因为进行 IO 操作而阻塞的时候，事件循环会继续运行，把 I/O 事件分发到其它协程上。

正如我们之前自己封装的 Task 对象一样， asyncio.Task 需要有一个 coroutine 协程来初始化，它会监听这个 coroutine 返回的 Future 所抛出的事件，从而驱动协程继续运行。

创建多个 asyncio.Task 对象，意味着创建多个协程，每个协程都有各自的函数栈，彼此独立。这些 Task 对象会接收到自己的 coroutine 返回的 Future 对象，然后监听 Future 的完成事件，并在事件完成时调用 coroutine 的 send 方法驱动协程执行下一步操作。这一系列操作都是独立的，不会彼此影响。

### asyncio.Queue

Queue 是 asyncio 中提供的一个阻塞队列，它也是基于协程技术实现的，因此 get,put 方法都需要使用 await 语法来等待。当队列为空时，`await q.get()` 将阻塞当前协程，当队列满时，`await q.put(xxx)` 也会阻塞协程。

此外它还提供了 `join()` 和 `task_done()` 方法，这两个方法需要配合使用，每次从队列中 `get()` 一个数据，并处理完这个数据后，都调用 `task_done()` 方法来递减内步计数。可以在协程里使用 `await q.join()` 方法来等待，直到队列中所有数据都处理完毕时，代码才会继续执行。

请看如下代码：

```py
import asyncio
from asyncio import Queue

async def Work(q):
    await asyncio.sleep(1)
    num = await q.get()
    q.task_done()

async def Main():
    loop = asyncio.get_event_loop()

    q = Queue(loop=loop)
    await q.put(1)
    q.put_nowait(2)

    workers = [asyncio.Task(Work(q), loop=loop) for _ in range(10)]

    await q.join()

    for worker in workers:
        worker.cancel()

loop = asyncio.get_event_loop()
loop.run_until_complete(Main())
```

注意一定要用 await 来调用 get, put, task_done, join 等方法。

Queue 还有一个 `q.put_nowait()` 方法，它不经过协程的处理，直接把数据 put 到队列中。

### 500lines crawler

明白了上述几点，500lines 的 [crawler](https://github.com/aosabook/500lines/tree/master/crawler) 项目就非常好理解了。

这里暂时就不介绍 crawler 的代码了，有兴趣可以 clone 下来看一下，非常简洁直白。

