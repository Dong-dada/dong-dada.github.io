---
layout: post
title:  "Python 实战 Web App 开发"
date:   2017-05-09 10:19:30 +0800
categories: python
---

* TOC
{:toc}

之前通过爬虫练习了一下 python, 这次跟着 [廖雪峰python](http://www.liaoxuefeng.com/wiki/0014316089557264a6b348958f449949df42a6d3a2e542c000/001432170876125c96f6cc10717484baea0c6da9bee2be4000) 学习一下开发 Web App。

这个教程的实战目标是一个 blog 网站，包含日志、用户和评论三大部分。

## 搭建开发环境

刚买了树莓派，这次刚好试试在树莓派上用 python 开发的感觉。

在树莓派上搭环境跟别的系统区别不大，毕竟树莓派的官方系统就是 linux。

树莓派自带了 python2 和 python3，这次要用的是 python3.

安装异步框架 aiphttp :

```
$ pip3 install aiohttp
```

前端模版引擎 jinja2 :

```
$ pip3 install jinja2
```

MySQL :

```
$ sudo apt-get install mysql-server python-mysqldb
```

MySQL 的异步驱动 :

```
$ pip3 install aiomysql
```

建立开发目录：

```
dada_blog/  <-- 根目录
|
+- backup/               <-- 备份目录
|
+- conf/                 <-- 配置文件
|
+- dist/                 <-- 打包目录
|
+- www/                  <-- Web目录，存放.py文件
|  |
|  +- static/            <-- 存放静态文件
|  |
|  +- templates/         <-- 存放模板文件
|
+- ios/                  <-- 存放iOS App工程
|
+- LICENSE               <-- 代码LICENSE
```


## 编写 Web App 骨架

在 www 目录下新建一个 app.py 文件，然后输入以下代码：

```py
import logging; logging.basicConfig(level=logging.INFO)

import asyncio, os, json, time
from datetime import datetime

import socket
from aiohttp import web

def get_ip_address():
    s = socket.socket(socket.AF_INET, socket.SOCK_DGRAM)
    s.connect(("8.8.8.8", 80))
    ip =s.getsockname()[0]
    s.close()

    return ip

def index(request):
    return web.Response(body = b'<h1>Hello</h1>', content_type='text/html')

@asyncio.coroutine
def init(loop):
    ip = get_ip_address()

    app = web.Application(loop = loop)
    app.router.add_route('GET', '/', index)
    srv = yield from loop.create_server(app.make_handler(), ip, 9000)
    logging.info('server start at http://' + ip + ':9000...')
    return srv

loop = asyncio.get_event_loop()
loop.run_until_complete(init(loop))
loop.run_forever()
```

上述代码运行起来后，在浏览器上输入树莓派的 ip 加端口，就会显示一个 Hello 页面。

详细看看这段代码做了什么：

### get_ip_address

`get_ip_address()` 这个函数是我自己加上去的，其作用是获取设备的 ip 地址。关于在 python 中获取 ip 地址的方法，在网上搜了一圈，比较普遍的解决方案是这样：

```py
import socket
ip = socket.gethostbyname(socket.gethostname())
print(ip)
```

这种写法在电脑上是没问题的，在树莓派上运行则会获取到一个奇怪的 ip : `127.0.1.1`, 搜了一下发现这个 ip 好像跟 `/etc/hosts` 文件有关，打开这个文件，果然发现有一行：

```
raspberry 127.0.1.1
```

尝试了一下把它删掉，用上面的方法就获取不到 ip 了，如果使用 `gethostbyname_ex()` 函数，则会获取到一个外网 ip. 不是我想要的结果。据 [这篇文章](http://openwares.net/linux/debian_hosts_127_0_1_1.html) 的解释，`127.0.1.1` 这个 ip 是用来获取全限定域名的，如果设备有一个静态 ip, 就不会返回 `127.0.1.1`。因为不知道家里路由器的管理员密码，所以我的树莓派没有固定 ip. 

继续搜索，在 [Stack Overflow](http://stackoverflow.com/questions/166506/finding-local-ip-addresses-using-pythons-stdlib) 上找到了一段代码，也就是我所写的 `get_ip_address()` 函数中的内容。回答中描述说这种方法假定设备能够连接到网络，并且没有设置代理。现在暂时对网络这块儿还不熟悉，这里就不深究原理了。

### index

`index()` 这个函数的作用很好理解，它就像一个 handler, 专门用来处理 `'/'` 这个路由消息。无论传入了什么参数，都直接返回一个 `<h1>Hello</h1>` 页面。

### init

`init()` 这个函数一眼看上去有些不明朗，它有一个装饰器 `@asyncio.coroutine`, 其中还使用了 `yield from` 语法。

`init()` 是一个 generator 而不是普通的函数, 因为它的定义中包含了 `yield` 关键字。也就是说，直接调用 `init()` 并不会执行它，而是会返回一个 generator 对象，对这个 generator 对象调用 `next()` 方法，才会真正执行这个 `init()` 里面的内容：

```py
foo = init(loop)
foo.next()
```

`@asyncio.coroutine` 将 `init()` 这个 generator 标记为 coroutine 类型，这样就可以把 `init()` 扔到 `asyncio` 的 EventLoop 中执行。

在 `init()` 内部，首先创建一个 `web.Application` 对象，然后把之前定义的 `index()` 函数加入到路由里面：

```py
app = web.Application(loop = loop)
app.router.add_route('GET', '/', index)
```

接着，调用 loop 的 `create_server` 方法创建 server, 并等待他执行结束：

```py
srv = yield from loop.create_server(app.make_handler(), ip, 9000)
```

`create_server` 方法本身也是一个 coroutine, `yield from` 关键字的作用是等待这个 coroutine 执行完毕。代码执行到这一句的时候并不会阻塞，而是中断下来，让 EventLoop 可以执行下一次循环，当 `create_server` 这个 coroutine 执行完毕之后，`yield from` 会返回它创建的 server.

最后的这几行代码很好理解，它首先创建一个 EventLoop, 然后把 `init()` 扔到 EventLoop 中执行：

```py
loop = asyncio.get_event_loop()
loop.run_until_complete(init(loop))
loop.run_forever()
```

`run_forever()` 将一直运行 EventLoop, 以便处理其他异步 IO 事务。

asyncio 这个库内部的运行原理看起来很有意思，很像之前在 《linux 多线程服务器端编程》 中看到的 non-blocking IO + EventLoop 模式，有空写篇文章来专门讨论一下。