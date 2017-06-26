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
import logging
import asyncio
from aiohttp import web

logging.basicConfig(level=logging.DEBUG)
LOGGER = logging.getLogger(__name__)

def get_ip_address():
    """ 获取本机 ip 地址 """
    s = socket.socket(socket.AF_INET, socket.SOCK_DGRAM)
    s.connect(("8.8.8.8", 80))
    ip =s.getsockname()[0]
    s.close()

    return ip

def index(request):
    """处理 '/' 请求"""
    return web.Response(body=b'<h1>Hello world!</h1>', content_type='text/html')

async def init(loop):
    """程序入口"""
    app = web.Application(loop=loop)
    app.router.add_route('GET', '/', index)

    ip_addr = get_ip_address()
    server = await loop.create_server(app.make_handler(), ip_addr, 9000)
    LOGGER.info('server started at http://' + ip_addr + ':9000 ...')

    return server

loop = asyncio.get_event_loop()
loop.run_until_complete(init(loop))
loop.run_forever()
```

上述代码运行起来后，在浏览器上输入树莓派的 ip 加端口，就会显示一个 Hello 页面。

上面的代码非常简单，其核心是利用 asyncio 库的 `loop.create_server()` 方法创建一个 server, 接着利用 aiphttp 的 `web.Application` 类进行 url 的路由。

值得一提的是 `get_ip_address()` 函数，这种获取 ip 的思路来自于 [Stack Overflow 的一个回答](http://stackoverflow.com/questions/166506/finding-local-ip-addresses-using-pythons-stdlib)。


## 编写 ORM

所谓 ORM(Object-Relational Mapping), 其作用是将表结构与类对象映射起来，能够更加直白简单地表示对表的操作。例如下面代码用 User 对象来封装对 User 表的操作，避免手动编写 SQL 语句，简化了逻辑：

```py
# 创建实例:
user = User(id=123, name='Michael')
# 存入数据库:
user.save()
# 查询所有User对象:
users = User.findAll()
```

按照最直观的写法，我们可以这样来编写 User 类，从而封装对 user 表的操作：

```py
class User():
    def __init__(self, **kw):
        self.id = kw.get('id', None)
        self.name = kw.get('name', None)
    
    def save(self):
        sql = 'insert into user (id, name) values (%s, %s)' % (self.id, self.name)
        # 数据库操作，略
        pass

    def update(self):
        sql = 'update user set name=%s where id=%s' % (self.name, self.id)
        # 数据库操作，略
        pass
```

## 封装 Model 基类

上述做法有一个缺点，如果我们需要操作 blog 表，就需要另外封装一个 Blog 类，我们得再为这个 Blog 类编写 save, update 等方法，这造成了代码的重复。

这一小节尝试编写一个基类 Model, 统一提供 save, update 等方法，这样 User, Blog 等派生类就不需要额外编写这些方法了。

要编写基类 Model, 首先就面临一个问题：基类 Model 不知道子类 User 有哪些表项，这样的话它没办法在 sql 语句中填充表项信息。

### 元类 Meta Class

要解决上述问题，需要用到 元类 (metaclass) 这个工具。对于元类的原理，这里不多做介绍，可以参考 [这篇文章](http://blog.jobbole.com/21351/)。这里只需要知道，**我们可以通过元类来访问类属性**，从而了解到子类的属性信息：

```py
class ModelMetaclass(type):
    def __new__(cls, name, bases, attrs):
        pass

class Model(dict, metaclass=ModelMetaclass):
    pass

class User(Model):
    id = StringField(primary_key=True, default=next_id, ddl='varchar(50)')
    email = StringField(ddl='varchar(50)')
    passwd = StringField(ddl='varchar(50)')
    admin = BooleanField()
    name = StringField(ddl='varchar(50)')
    image = StringField(ddl='varchar(500)')
    created_at = FloatField(default=time.time)
```

注意上述代码中，我们在 User 类中定义了一系列类成员变量，如 id, email, passwd, name 等。它们都被赋值为 `XXXField` 类型的值。

在代码 `clss User(Model):` 定义 User 类时，python 解释器会首先从 User 类的定义中查找 metaclass, 找不到的话就去基类 Model 里找，找到了，就使用`ModelMetaclass.__new__()` 方法来创建 User **类**。

注意，上述过程不涉及对象的创建，metaclass 的作用是干涉类的创建过程，与对象的创建过程无关。即使你没有创建对象，`ModelMetaclass.__new__()` 方法也会执行，因为你定义了 User 类。

`__new__()` 函数的第 4 个参数 `attrs` 中包含了要创建类的所有属性 (类的成员变量，成员函数等)，因此你可以在 `__new__()` 中通过 `attrs` 参数来了解到子类 User 中定义的 id, name 等类成员变量的信息：

```py
class ModelMetaclass(type):
    def __new__(cls, name, bases, attrs):

        # 通过 attrs 参数访问 User 类的类成员变量
        for key, value in attrs.items():
            if isinstance(value, Field):
                print(key)

class Model(dict, metaclass=ModelMetaclass):
    pass

class User(Model):
    pass

# 输出
# id
# email
# passwd
# admin
# name
# image
# created_at
```

从上述代码可以看出，在 `ModelMetaclass.__new__()` 方法中可以访问到 User 类的类成员变量，但 Model 里是无法访问到这些变量的。

### 用 Field 封装表的列信息

`XXXField` 封装了表的列信息，其定义如下：

```py
class Field(object):

    def __init__(self, name, column_type, primary_key, default):
        self.name = name
        self.column_type = column_type
        self.primary_key = primary_key
        self.default = default

    def __str__(self):
        return '<%s, %s:%s>' % (self.__class__.__name__, self.column_type, self.name)

class StringField(Field):

    def __init__(self, name=None, primary_key=False, default=None, ddl='varchar(100)'):
        super().__init__(name, ddl, primary_key, default)

class IntegerField(Field):

    def __init__(self, name=None, primary_key=False, default=0):
        super().__init__(name, 'bigint', primary_key, default)

# 其他类型，略
```

`id = StringField(primary_key=True, default=next_id, ddl='varchar(50)')` 就表示 User 表中有一个 id 列，这一列存储了表的主键，类型为 `varchar(50)`.

### 编写 Model 类

`ModelMetaclass.__new__()` 函数中可以获取到表的列信息，接下来就可以利用这些信息构建 sql 语句，供 Model 类使用：

```py
class ModelMetaclass(type):

    def __new__(cls, name, bases, attrs):

        # 如果子类中包含了 __table__ 成员变量，就用它作为数据库表名，否则使用类名来作为表名
        table_name = attrs.get()

        # 把 User 表的列信息映射关系存储到 mappings 字典中
        mappings = dict()
        for key, value in attrs.items():
            if isinstance(value, Field):
                mappings[key] = value
        
        # 从 attrs 中移除这些属性，见下文
        for key in mappings.keys():
            attrs.pop(key)
        
        # 记录到 `__mappings__` 属性中，供 Model 类使用
        attrs['__mappings__'] = mappings

        # 找到主键名和其他属性名
        primary_key = None
        field_keys = []
        for key, value in mappings.items():
            if isinstance(value, Field):
                if value.primary_key == true:
                    if primary_key:
                        raise StandardError('Duplicate primary key for field : %s' % key)
                    else:
                        primary_key = key
                else
                    field_keys.append(key)

        # 拼接出 select, insert, update, delete 等 sql 语句，并记录到 attrs 中，供 Model 类使用
        escaped_fields = list(map(lambda f: '`%s`' % f, field_keys))
        attrs['__primary_key__'] = primary_key
        attrs['__field_keys__'] = field_keys
        attrs['__select__'] = 'select `%s`, %s from `%s`' % (primary_key, ', '.join(escaped_fields), table_name)
        attrs['__insert__'] = 'insert into `%s` (%s, `%s`) values (%s)' % (table_name, ', '.join(escaped_fields), primary_key, create_args_string(len(escaped_fields) + 1))
        attrs['__update__'] = 'update `%s` set %s where `%s`=?' % (table_name, ', '.join(map(lambda f: '`%s`=?' % (mappings.get(f).name or f), fields)), primary_key)
        attrs['__delete__'] = 'delete from `%s` where `%s`=?' % (table_name, primary_key)

        # 使用经过修改的 attrs 参数来创建子类
        return type.__new__(cls, name, bases, attrs)
```

`ModelMetaclass.__new__()` 利用 attrs 读取到类成员信息之后，将表的列信息记录到了 `__mappings__` 属性里，并拼接出 sql 语句，记录到了 `__select__`, `__insert__`, `__update__`, `__delete__` 等属性中。接着就可以在 Model 类中使用这些信息了：

