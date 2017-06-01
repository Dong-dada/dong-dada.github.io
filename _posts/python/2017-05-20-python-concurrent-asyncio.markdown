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
- [asyncio](https://docs.python.org/3/library/asyncio.html)
- [一个使用 asyncio 协程的爬虫(一)](http://mp.weixin.qq.com/s?__biz=MzA3NDk1NjI0OQ==&mid=2247483875&idx=1&sn=66b10f67c8f2cb2002fa16d9022540be&chksm=9f76ad55a8012443cd1c58c82a5f8a40178b29e04179ae7122f122116531bf1d8a96e736a275#rd)
- [一个使用 asyncio 协程的爬虫(二)](http://mp.weixin.qq.com/s?__biz=MzA3NDk1NjI0OQ==&mid=2247483875&idx=2&sn=753243326c3486228881c9810eb43f0e&chksm=9f76ad55a80124439cdcd5a37e6d4ce8ec6bc95a4848474f4e089f56039b82d8c8d9516fbc42#rd)
- [单台服务器并发 TCP 连接数到底可以有多少](http://www.52im.net/thread-561-1-1.html)
- [高性能网络编程(二)：上一个10年，著名的C10K并发连接问题](http://www.52im.net/thread-566-1-1.html)

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

可以看到上面的逻辑非常简单，就是用一个 while 循环不断等待 `poll` 返回，返回后就调用对应的 handler。

### python 的 selector

上文提到的 `::poll` 是 linux 提供的一个系统函数，你把一系列 文件描述符 作为参数传给他，它会阻塞住，直到某个文件描述符变为可用，或者超时。事实上很久以前 Unix 提供了一个 select 函数，poll 函数是它的改进版，性能更好。

python 中提供了 selector 来实现类似的功能，请看如下代码：

```py
# coding:utf-8

from selectors import DefaultSelector, EVENT_WRITE
import socket

def connected():
    """ selector 的回调函数 """
    selector.unregister(sock.fileno())
    print('connected!')

def loop():
    """ Event Loop """
    while True:
        # 调用 selector.select() 会阻塞，直到事件被触发
        events = selector.select()
        for event_key, event_mask in events:
            # 从 event 里取出 callback 并回调它
            callback = event_key.data
            callback()

if __name__ == '__main__':

    selector = DefaultSelector()

    # 创建非阻塞的 socket, 这样调用 sock.connect 时不会阻塞
    sock = socket.socket()
    sock.setblocking(False)
    try:
        sock.connect(('127.0.0.1', 80))
    except BlockingIOError:
        pass

    # 将 文件描述符 和 回调函数注册给 selector
    selector.register(sock.fileno(), EVENT_WRITE, connected)

    # 启动 loop
    loop()
```

相比于原先的 Event Loop 的实例代码，这里增加了一个回调函数的逻辑，便于将 selector 返回的事件分发给对应的 handler 函数。

### callback 编程模型的困难

基于 callback 的编程是一种很常见的手法，然而当嵌套层次比较深的时候编写这样的代码会很困难。

例如编写一个爬虫代码，`socket.connect()` 只是第一步。随后我们还得向服务器发送一个 GET 请求，为了避免这一步被阻塞，我们得注册另外一个回调函数。GET 之后，它不能一次读取完所有请求，所以还得再一次注册，如此反复。

我们把这个过程写出来，看看实际的代码会有多难写。

首先，我们有一个未获取 url 的集合，和一个已获取 url 的集合：

```py
urls_todo = set(['/'])
seen_urls = set(['/'])
```

之前说过，会有很多回调，我们把这些回调放在一个 Fetcher 对象中：

```py
class Fetcher:
	def __init__(self, url):
		self.url = url
		self.sock = None
		self.response = b''

	def fetch(self):
		self.sock = socket.socket()
		self.sock.setBlocking(False)

		try:
			self.sock.connect(('xkcd.com', 80))
		catch BlockingIOError:
			pass
		
		selector.register(self.sock.fileno(), EVENT_WRITE, self.connected)
	
	def connected(self, key, mask):
		pass
```

connected 回调中需要发送 GET 请求：

```py
def connected(key, mask):
	print('connected!')
	# 先注销之前注册的 connected 回调
	selector.unregister(key.fd)

    # 发送 GET 请求
	request = 'GET {} HTTP/1.0\r\nHost: xkcd.com\r\n\r\n'.format(self.url)
	self.sock.send(request.encode('ascii'))

	# 注册回调，用于处理 GET 请求
	selector.register(key.fd, EVENT_READ, self.read_response)
```

接着编写 `read_response` 回调，来处理服务器对 GET 请求的响应：

```py
def read_response():
	global stopped

	chunk = self.sock.recv(4096)
	if chunk:
		self.response += chunk
	else:
		# 同样的，先注销 read_response 回调
		selector.unregister(key.fd)

		# 解析出新的链接
		links = self.parse_links()
		for link in links.difference(seen_urls):
			urls_todo.add(link)
			# 用 Fetcher 抓取这个链接中的内容
			Fetcher(link).fetch()
		
		seen_urls.update(links)
		urls_todo.remove(self.url)
		if not urls_todo:
			# 没有需要处理的链接了，将 stopped 设为 True
			stopped = True
```

stopped 变量用于控制程序何时退出：

```py
stopped = False

def loop():
	while not stopped:
		events = selector.select()
		for event_key, event_mask in events:
			callback = event_key.data
			callback(event_key, event_mask)
```

上面的例子暴露出了异步编程的一大问题：意大利面代码。

为了接受异步事件，你不得不编写许多回调函数。同时还需要在类成员中记录状态信息，也就是 url, sock, response 这些数据。而如果我们用同步代码来完成上述逻辑，那么将非常简单直白：

```py
# Blocking version.
def fetch(url):
    sock = socket.socket()
    sock.connect(('xkcd.com', 80))
    request = 'GET {} HTTP/1.0\r\nHost: xkcd.com\r\n\r\n'.format(url)
    sock.send(request.encode('ascii'))
    response = b''
    chunk = sock.recv(4096)
    while chunk:
        response += chunk
        chunk = sock.recv(4096)
    # Page is now downloaded.
    links = parse_links(response)
    q.add(links)
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

然而，跟 C 语言中的函数栈不一样的是，**python 中的栈是在堆中分配的**。你可以把 python 的函数栈理解成是在堆中 new 出来的一块内存，这块内存可以留着不释放。这跟 C 语言的函数栈不一样，C 的函数栈是通过栈顶置针来控制栈大小的，所以退出函数的时候栈空间一定会被释放掉。

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

在 C 语言编写的 python 解释器中，用 PyFrameObject 来表示一个函数的栈帧，它的 `f_back` 成员指向调用者， `f_code` 指向子节码：

![]( {{site.url}}/asset/python-asyncio-pyframeobject.png )

### python 生成器的原理

