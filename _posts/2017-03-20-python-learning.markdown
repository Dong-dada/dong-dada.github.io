---
layout: post
title:  "Python 基础知识学习"
date:   2017-03-20 10:53:30 +0800
categories: lang python
---

* TOC
{:toc}

## Python 基础知识

### 输入和输出

python 中使用 print 函数来进行打印：

```python
print 'The quick brown fox', 'jumps over', 'the lazy dog'
```

`print` 会依次打印每个字符串，遇到逗号“,”会输出一个空格，因此，输出的字符串是这样拼起来的：

![]({{ site.url }}/asset/python-print.png)

python 中使用 raw_input 函数来获取用户的输入：

```python
name = raw_input('please enter your name:')

print `hello`, name
```

### 数据类型和变量

python 支持常见的数据类型，包括 布尔值(True/False)、整数、浮点数、字符串、空值(None)。

python 中的布尔值只有 True/False 这两种，它们可以使用 `and`, `or`, `not` 布尔运算。

```python
# 根据年龄判断是否是成年人
if age >= 18:
    print 'adult'
else:
    print 'teenager'
```

另外在这里提一句，python 使用缩进来表示作用域块。在冒号 `:` 之后必须新开一行增加缩进。

python 中使用 `#` 来进行注释

#### 字符串

python 中的字符串需要用 `''` 或 `""` 括起来，例如 `"I'm OK"` 表示一个字符串。python 同样支持类似于 C 的转义字符，例如 `\n` 等。另外，python 允许用 r'' 表示 '' 内部的字符串默认不转义：

```python
print r'C:\note\text.txt'
```

python 还支持多行字符串写法 '''...''' ：

```python
print '''line1
... line2
... line3'''
```

上述代码的打印结果是：

```python
line1
line2
line3
```

python 中默认的字符串是 ASCII 编码，它提供了 `u'哈哈哈'` 这样的写法来表示这个字符串是 Unicode 字符串。这时候采用的编码实际上是 UTF16 编码(我猜的)，例如：

```python
print u'中', u'\u4e2d'
```

上述代码中的 `'中'` 与 `'\u4e2d'` 是一样的，都表示一个 Unicode 字符。

你可以使用 `encode('utf-8')` 方法来将 Unicode 字符串转换为 UTF8 编码格式的字符串：

```python
a = u'这是一行中文'
b = a.encode('utf-8')
```

你可以使用 `decode('utf-8')` 方法来将 UTF-8 字符串转换为 Unicode 编码格式：

```python
a = '\xe4\xb8\xad\xe6\x96\x87'
b = a.decode('utf-8')
```

多说一句，你可以在 python 文件的开头加上以下代码，这样 python 解释器会把这篇文件当做 UTF-8 编码格式进行读取：

```python
# -*- coding: utf-8 -*-
```

python 中提供了类似于 C 的字符串格式化方法。例如下面代码：

```python
time = '%d-%d-%d/%d:%d:%d' % (2017, 3, 20, 11, 55, 10)
print time # 2017-3-20/11:55:10
```

#### 变量

python 中的变量没有类型，例如下面代码：

```python
# age 这个变量可以存储任何类型的值

age = 123
age = '123' 
```

### 内置数据结构

python 内置了两种列表型数据结构 list 和 tuple:

list 类似于 C++ 中的 `std::vector` 或 `std::list`

```python
# list 中可以存储各种类型的值
a = ['Michale', 'Bob', 'Tracy', 123, 10.01, True, False]

# list 中可以存储 list
classmates = ['Michale', 'Bob', 'Tracy', ['Tom', 'Jerry']]

# len 方法可以获取 list 当前的长度
print len(classmates)

# list 的下标从 0 开始
print classmates[0]

# 如果超过 list 下标，会抛出 index out of range 错误
print classmates[10000]

# 可以用 -1 作为下标，表示倒数第一个
print classmates[-1]

# 可以使用 append 方法附加新的元素到末尾
classmates.append('Tim')

# 可以使用 insert 方法插入元素
classmates.insert(1, 'Jack')

# 可以使用 pop 方法删除元素
classmates.pop(1)
```

tuple 与 list 不同，它一旦初始化之后就不能修改：

```python
# 初始化 tuple
classmates = ('Michale', 'Bob', 'Tracy')

# 不能修改 tuple, 只能访问它
classmates[1] = 10  # 出错

# tuple 中可以包含 list 等数据结构，tuple 中的 list 元素是可以被修改的
a = (123, True, ['Michale', 'Bob'])
a[2].append('Tom')
```

如果可能的话，尽量用 tuple 而不是 list, 这样可以对数据进行保护。

python 内置了两种字典型数据结构 dict 和 set:

dict 类似于 C++ 中的 `std::map`

```python
# 创建一个 dict
d = {'Michael':95, 'Bob':75, 'Tracy':85}

# 使用 key 来访问 dict
num = d['Michael']
d['Michael'] = 10

# 访问不存在的 key 会报错
print d['Thomas']  # KeyError : 'Thomas'

# 可以通过 key 来添加新的元素
d['new item'] = 10
print d['new item']

# 可以通过 pop 方法删除元素
d.pop('new item')
print d['new item'] # 报错

# 通过 in 操作符可以判断 key 是否存在
if 'Thomas' in d:
    d['Thomas'] = 10

# 通过 get() 方法可以判断 key 是否存在
num = d.get('Thomas')  # 不存在时返回 None
num = d.get('Thomas', -1) # 不存在时返回 -1

# 可变的值如 list 不能作为 key, 但不可变的值如 整形、字符串、tuple 可以作为 key
d[['asdf', 123]] = 10  # 报错 unhashable type : 'list'
d[('asdf', 123)] = 10
```

set 类似于 C++ 中的 `std::set`, 与 dict 不同，它只存储 key, 不存储 value

```python
# 创建 set 需要提供一个 list 作为输入，然后使用 set() 方法来创建
s = set([1, 2, 3])

# 重复的元素自动被过滤
s = set([1, 1, 2, 2, 3, 3]) # s 中只会有 1,2,3 这三个元素

# 通过 add 方法可以添加新的元素
s.add(4)

# 通过 remove 方法可以移除旧元素
s.remove(3)

# set 可以看做数学上无序和无重复元素的集合，因此可以进行交集并集等操作：
s1 = set([1, 2, 3])
s2 = set([2, 3, 4])

s3 = s1 & s2 # s3 中包含 2, 3 两个元素
s4 = s1 | s2 # s4 中包含 1,2,3,4 四个元素
```

### 条件判断和循环

python 中的判断使用 `if`, `elif`, `else` 这三个操作符

```python
age = 20
if age >= 20:
    print 'adult'
elif age >= 6:
    print 'teenager'
else:
    print 'kid'
```

注意不要少些冒号 `:`

判断条件可以简写：

```python
# 非零数字、非空字符串、非空 list 等，会被判断为 True
if x:
    print 'True'
```

python 中的循环使用 `for...in` 来遍历 list,tuple,dict,set 中的元素：

```python
# 遍历 list
names = ['Machael', 'Tom', 'Tracy']
for name in names:
    print name

# 遍历 tuple
names = ('Machael', 'Tom', 'Tracy')
for name in names:
    print name

# 使用 range 可以生成一个整数序列：
sum = 0
for num in range(101):  # 生成 0~100 之间的整数序列
    sum = sum + num

# 遍历 dict
students = {'Machael':12, 'Tom':13, 'Jack':123}
for name in students:
    print students[name]

# 遍历 set
students = {'Machael', 'Tom', 'Jack'}
for name in students:
    print name
```

python 中也可以使用 while 循环：

```python
sum = 0
n = 99
while n > 0:
    sum = sum + n
    n = n-2
```

对于 dict,set 而言，还可以使用 items, iteritems 方法来遍历:

```python
# 使用 items() 方法得到一个序列
for (k, v) in d.items():
    print k, v

# 使用 iteritems() 方法得到一个序列
for k, v in d.iteritems():
    print k, v
```


## 函数

### 内置函数

你可以去 [这里](https://docs.python.org/2/library/functions.html) 查看 python 内置函数的列表。

以下函数可以用于数据类型转换：

```python
>>> int('123')
123
>>> int(12.34)
12
>>> float('12.34')
12.34
>>> str(1.23)
'1.23'
>>> unicode(100)
u'100'
>>> bool(1)
True
>>> bool('')
False
```

### 定义函数

定义函数需要使用 def 操作符：

```python
def my_abs(x) : # 注意这里要加冒号
    if x >= 0:
        return x
    else:
        return -x
```

检查传入参数的类型可以使用 `isinstance()` 函数：

```py
def my_abs(x) :
    if not isinstance(int, float):
        raise TypeError('bad oper and type') # 抛异常，以后会介绍
    if x >= 0:
        return x
    else:
        return -x
```

python 中的函数可以有多个返回值：

```
import math

def move(x, y, step, angle = 0):
    nx = x + step * math.cos(angle)
    ny = y - step * math.sin(angle)
    return nx, ny

x, y = move(100, 10, 60, math.pi / 6)
print x, y
```

实际上这时候返回的是一个 tuple, python 提供了语法糖可以同时接受多个结果。

如果你希望一个函数什么都不做，可以用 `pass` 语句:

```py
def foo():
    # 暂时 pass, 以后对应代码
    pass
```

对于 if, elif, else 也可以使用 `pass`：

```py
if age > 60:
    pass
```

函数在 python 中是第一类值，可以存储到变量中：

```py
a = abs
print a(-1)
```

### 函数的参数

python 中函数的参数可以有默认值：

```py
def power(x, n=2): 
    sum = 1
    while n > 0:
        n = n - 1
        sum = sum * x
    return sum
```

有默认值的参数必须放到参数列表后面；

要注意参数的默认值必须是不可变的值，而不能是 list 等可变值，下面的代码在多次调用时会出现错误：

```py
def add_end(list_param = [])
    list_param.append('END')
    return list_param

print add_end() # END
print add_end() # END, END 
```

这是因为默认参数也是一个变量，如果你不断调用，那么会不断操作同一个变量。所以要注意默认参数最好是不可变的值，避免引起意外错误。

python 中可以传递可变参数：

```py
def calc(*numbers):
    sum = 0
    for num in numbers:
        sum = sum + num
    return sum

# 传入多个参数
print calc(1, 2, 3, 4)

# 如果参数是 list 或 tuple, 可以在参数前加一个 * 来传入
numbers = [1, 2, 3, 4, 5]
print calc(*numbers)
```

python 中的可变参数实际上是通过 tuple 来实现的，python 自动帮你把传入的多个参数转换为一个 tuple。

python 中还提供了一种叫做关键字参数的用法，这种参数类似于 dict：

```py
def person(name, age, **kw):
    print name, age, kw

person('Dongdada', 18, city='hangzhou') # 输出 Dongdada 18 {'city': 'Beijing'}

def student(name, age, **info):
    # 关键字参数实际上是一个 dict, 可以直接使用
    for key, value in info:
        print key value

#类似的，你可以把 dict 当做关键字参数直接传进去，只需在参数前加两个 *
info = {'city'='hangzhou'}
student('Dongdata', 18, **info)
```

关键字参数的作用在于它可以扩展函数的功能。

上述几种参数可以组合使用，其先后顺序必须是： 必选参数、默认参数、可变参数、关键字参数。

```py
def func(a, b, c=0, *args, **kw):
    print 'a =', a, 'b =', b, 'c =', c, 'args =', args, 'kw =', kw
```

如果某一项参数没有填，python 会自动填充为空的 tuple 或 dict:

```py
>>> func(1, 2)
a = 1 b = 2 c = 0 args = () kw = {}
>>> func(1, 2, c=3)
a = 1 b = 2 c = 3 args = () kw = {}
>>> func(1, 2, 3, 'a', 'b')
a = 1 b = 2 c = 3 args = ('a', 'b') kw = {}
>>> func(1, 2, 3, 'a', 'b', x=99)
a = 1 b = 2 c = 3 args = ('a', 'b') kw = {'x': 99}
```

### 递归函数

python 中可以定义递归函数： 

```py
def fact(n):
    if n==1:
        return 1
    return n * fact(n - 1)
```

上述递归可能会导致栈溢出，解决栈溢出的方式是使用 尾递归优化，事实上尾递归和循环的效果是一样的。

尾递归是指：在函数返回的时候，返回函数自身，并且 return 语句中不能包含表达式，这样编译器就可以对尾递归优化掉，不管递归都多少次，都使用同一个栈帧：

```
def fact(n):
    return fact_iter(n, 1)

def fact_iter(num, product):
    if num == 1:
        return product
    return fact_iter(num - 1, num * product)
```

上述代码符合尾递归的定义方法，可惜的是 python 的解释器没有对尾递归做出优化，也会导致栈溢出；