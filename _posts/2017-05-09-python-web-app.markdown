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
    return web.Response(body = b'<h1>Awesome</h1>', content_type='text/html')

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