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

### 在 ubuntu 上安装及配置 mysql

搬家后没带 hdmi 线，连不上树莓派了，只好虚拟机里装了一个 Ubuntu 来开发。

现在我想在 Ubuntu 虚拟机上安装 mysql, 在本机(Windows) 上通过 python 的 mysql connector 连接到 ubuntu 的 mysql 来进行开发。之所以这么搞主要是刚装了新电脑，不想在电脑上安装 mysql 这个大家伙。

在 Ubuntu 上安装 mysql 的方法很简单，只需使用如下几个命令即可：

```
sudo apt-get install mysql-server
sudo apt-get install mysql-client
```

安装完毕后可以使用 `mysql -u root -p` 命令来登陆到 mysql 中。

为了测试，可以使用 `create database test;` 命令来创建一个名为 test 的数据库；

接下来就是从主机连接到虚拟机的 mysql 上了。这需要在主机上安装 python 的 mysql connector, 去官网看了一下目前支持的 python 版本最高到 3.4, 而我的 python 版本是 3.6. 所以官网安装这种方法就用不了了。

不过可以改为安装 pymysql，它的作用跟 msql connector 类似，只需通过 pip 安装即可：`pip install pymysql`

接下来编写如下测试代码来尝试从主机上连接虚拟机中的 mysql:

```py
import logging
import pymysql

logging.basicConfig(level=logging.DEBUG)
LOGGER = logging.getLogger(__name__)

if __name__ == '__main__':
    # 连接到 test 数据库
    connect = pymysql.connect(
        host='192.168.125.121',
        port=3306,
        user='dongdada',
        passwd='password',
        database='test',
        charset='utf8')

    cursor = connect.cursor()

    # 在数据库中创建一个 user 表
    cursor.execute('create table user (id varchar(20) primary key, name varchar(20))')

    # 在表中插入一行数据
    cursor.execute('insert into user (id, name) values (%s, %s)', ['1', 'dongdada'])

    LOGGER.info('row count = %d' % cursor.rowcount)

    # 提交数据库请求
    connect.commit()
    connect.close()
```

运行上述代码后出现了如下提示：

```
Traceback (most recent call last):
  File "D:\code\py\pytest\pytest\pytest.py", line 36, in <module>
    charset='utf8')
  File "C:\Program Files\Python36\lib\site-packages\pymysql\__init__.py", line 90, in Connect
    return Connection(*args, **kwargs)
  File "C:\Program Files\Python36\lib\site-packages\pymysql\connections.py", line 706, in __init__
    self.connect()
  File "C:\Program Files\Python36\lib\site-packages\pymysql\connections.py", line 963, in connect
    raise exc
pymysql.err.OperationalError: (2003, "Can't connect to MySQL server on '192.168.125.121' ([WinError 10061] 由于目标计算 机积极拒绝，无法连接。)")
```

这是因为 ubuntu 上安装的 mysql 还不支持远程连接，需要进行一些修改。在 ubuntu 上执行以下命令：

```
sudo vim /etc/mysql/mysql.conf.d/mysqld.cnf
```

打开 mysqld.cnf 之后，将 `bind-address = 127.0.0.1` 这行配置注释掉，保存退出。

接着使用 `mysql -u root -p` 进入 mysql 执行环境，然后执行如下命令，设置远程访问 mysql 时的用户名和密码：

```
mysql -u root -p
grant all privileges on *.* to dongdada@"%" identified by "password" with grant option;
flush privileges;
quit;
```

我把 dongdada 设为连接数据库时的用户名，password 设置为密码。随后使用 `flush privileges;` 指令来刷新设置；最后退出 mysql 环境。

接着还得使用 `sudo /etc/init.d/mysql restart` 命令来重启 mysql 服务。如此一来就可以远程连接到 Ubuntu 虚拟机中的 mysql 数据库了。

执行之前编写的 python 代码，成功把数据插入到了数据库中。你可以进入到 mysql 环境中，然后通过命令查看 mysql 的信息：

```
mysql> use test;
Reading table information for completion of table and column names
You can turn off this feature to get a quicker startup with -A

Database changed
mysql> show columns from user from test;
+-------+-------------+------+-----+---------+-------+
| Field | Type        | Null | Key | Default | Extra |
+-------+-------------+------+-----+---------+-------+
| id    | varchar(20) | NO   | PRI | NULL    |       |
| name  | varchar(20) | YES  |     | NULL    |       |
+-------+-------------+------+-----+---------+-------+
2 rows in set (0.00 sec)
```

### 使用 aiomysql 操作数据库

再编写 ORM 之前，我们先把基本的数据库操作封装一下，便于接下来的操作。

首先创建一个连接池，避免频繁打开和关闭数据库连接：

```py
async def create_pool(loop, **kw):
    logging.info('create database connection pool...')
    global __pool
    __pool = await aiomysql.create_pool(
        host=kw.get('host', 'localhost'),
        port=kw.get('port', 3306),
        user=kw['user'],
        password=kw['password'],
        db=kw['db'],
        charset=kw.get('charset', 'utf8'),
        autocommit=kw.get('autocommit', True),
        maxsize=kw.get('maxsize', 10),
        minsize=kw.get('minsize', 1),
        loop=loop
    )
```

接着定义 select 语句：

```py
async def select(sql, args, size=None):
    log(sql, args)
    global __pool
    with (await __pool) as conn:
        cur = await conn.cursor(aiomysql.DictCursor)
        await cur.execute(sql.replace('?', '%s'), args or ())
        if size:
            rs = await cur.fetchmany(size)
        else:
            rs = await cur.fetchall()
        await cur.close()
        logging.info('rows returned: %s' % len(rs))
        return rs
```

insert, update, delete 语句可以定义一个通用的 execute() 函数：

```py
async def execute(sql, args):
    log(sql)
    with (await __pool) as conn:
        try:
            cur = await conn.cursor()
            await cur.execute(sql.replace('?', '%s'), args)
            affected = cur.rowcount
            await cur.close()
        except BaseException as e:
            raise
        return affected
```

### ORM 介绍

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

### 封装 Model 基类

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

```py
class Model(dict, metaclass=ModelMetaclass):

    def __init__(self, **kw):
        super(Model, self).__init__(**kw)

    def __getattr__(self, key):
        try:
            return self[key]
        except KeyError:
            raise AttributeError(r"'Model' object has no attribute '%s'" % key)
    
    def __setattr__(self, key, value):
        self[key] = value

    def get_value(self, key):
        return getattr(self, key, None)
    
    def get_value_or_default(self, key):
        value = getattr(self, key, None)
        if value is None:
            field = self.__mappings__[key]
            if field.default is not None:
                value = field.default() if callable(field.default) else field.default
                setattr(self, key, value)
        return value
    
    @classmethod
    async def find_all(cls, where=None, args=None, **kw):
        sql = [cls.__select__]
        if where:
            sql.append('where')
            sql.append(where)
        if args is None:
            args = []
        
        order_by = kw.get('order_by', None)
        if order_by:
            sql.append('order by')
            sql.append(order_by)
        
        limit = kw.get('limit', None)
        if limit is not None:
            sql.append('?')
            args.append(limit)
        elif isinstance(limit, tuple) and len(limit) == 2:
            sql.append('?, ?')
            args.extend(limit)
        else:
            raise ValueError('Invalid limit value: %s' % str(limit))
        
        rs = await select(' '.join(sql), args)
        return [cls(**r) for r in rs]
    
    # 其他方法，略
```

注意这里的 Model 类继承自 dict, 因此可以用初始化字典的方式来初始化 User 类。