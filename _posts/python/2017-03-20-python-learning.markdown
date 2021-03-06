---
layout: post
title:  "Python 基础知识学习"
date:   2017-03-20 10:53:30 +0800
categories: python
---

* TOC
{:toc}

本文参考自 [廖雪峰的官方网站](http://www.liaoxuefeng.com/wiki/001374738125095c955c1e6d8bb493182103fac9270762a000)

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

# python 也可以使用 break, continue
n = 0
while True
    n = n + 1
    if n = 2:
        break
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


## 高级特性

### 切片

取 list 或 tuple 中一部分元素是一个非常常见的操作。比如一个 list 如下：

```py
students = ['Machael', 'Sarah', 'Tracy', 'Bob', 'Jack']
```

你想取其中的前三个元素，该怎么取呢？很直观可以想到使用 `for...in ` 循环来取。但这样代码有点多。

python 提供了 slice 操作符，能大大简化这种操作。例如：

```
# 取前三个元素
somebody = students[0:3]

# 从索引 1 开始，取到索引 3 (4-1)
somebody = students[1:4]

# 如果第一个索引是 0，可以省略，例如取前 5 个元素
somebody = student[:5]

# 取倒数两个元素
somebody = students[-2:]

# 还支持每隔 n 隔取一个
somebody = students[::5]

# 取所有元素，相当于复制
somebody = students[:]

# 字符串也可是使用切片操作
sub_str = 'ABCDEFG'[:3]
sub_str = 'ABCDEFG'[::2]
```

### 迭代

迭代 dict 时，默认迭代的是 key, 可以通过 itervalues, iteritems 来获取 value:

```py
for key in d:
    print key

for value in d.itervalues():
    print value

for key, value in d.iteritems():
    print key value
```

在 python 中，只要一个对象是可迭代对象，就可以用 `for...in` 的方式进行迭代。

```py
# string 也是可迭代对象
str = 'ABC'
for ch in str:
    print ch
```

要判断一个对象是否可迭代，可以通过 collections 模块的 Iterable 类型来判断：

```py
from collection import Iterable

if isinstance('abc', Iterable) :
    # ...
```

### 列表生成式

之前我们提过，使用 range 方法可以生成整数序列：

```py
# 生成 1~10 整数序列
numbers = range(1, 11)
```

基于 range 方法，列表生成式可以生成更加复杂的序列：

```py
# 生成 1*1, 2*2, 3*3 ... 10*10 list
[x*x for x in range(1, 11)]
```

把要生成的元素 x*x 放到前面，后面跟 for 循环，就可以把 list 生成出来，非常方便。

上述方法还可以嵌套使用，生成全排列：

```py
# 生成 ABC/ABC 的全排列
print [m + n for m in 'ABC' for n in 'ABC']
# ['AX', 'AY', 'AZ', 'BX', 'BY', 'BZ', 'CX', 'CY', 'CZ']
```

运用列表生成式，可以写出非常简洁的代码。例如：

```py
# 列出当前目录下所有文件和文件名
import os
file_names = [dir for dir in os.listdir('.')]
```

列表生成式还可以利用 for 循环可以遍历 key,value 的特点，把 dict 中的 key,value 都记录到 list 中：

```py
d = {'x': 'A', 'y': 'B', 'z': 'C' }
print [k + '=' + v for k, v in d.iteritems()]
# ['y=B', 'x=A', 'z=C']
```

下述代码把 list 中所有字符串变成小写

```py
L = ['Hello', 'World', 'IBM', 'Apple']
print [s.lower() for s in L]
# ['hello', 'world', 'ibm', 'apple']
```

### 生成器

之前的列表生成器，生成的是整个列表，有可能我们暂时需要的仅仅是列表的前面几个元素，后面的用到了再生成可以节约内存。python 提供了生成器这项功能，可以逐个生成，避免一次性生成所有元素：

```py
# 只需要把之前的 [] 换成 (), 就能变成生成器：
generator = (x*x for x in range(1, 11))

# 通过 next 生成下一个结果
print generator.next()
# 0
```

生成器可以使用 `for...in` 来迭代：

```py
for num in generator:
    print num
```

抽象点说，生成器保存的是一种算法，每次调用 next, 就会返回下一个元素的值，没有更多元素时，抛出 StopIteration 错误。

如果推算的方式比较复杂，使用列表生成式的循环方式写不出来的话，我们可以自己来定义一个生成式，例如下面的代码定义了一个斐波那契数列生成式：

```py
def fib(max):
    n, a, b = 0, 0, 1
    while n < max:
        yield b # 注意这个 yield 操作符
        a, b = b, a+b
        n = n+1

# fib(6) 是一个生成器
generator = fib(6)

for num in generator:
    print num
```

注意上述代码中的 yield 操作符，就是它让 fib 这个函数变成了一个生成器。此时的 fib(6) 是一个能生成 1~6 斐波那契数列的生成器，每次调用 next 时，生成器都会执行，执行到 `yield b` 这一句之后返回。再次调用 next 时，会从 yield 的位置接着执行。

例如下面的例子：

```py
def odd():
    print 'step 1'
    yield 1
    print 'step 2'
    yield 3
    print 'step 3'
    yield 5

o = odd()
print o.next()
# step 1
# 1

print o.next()
# step 2
# 3

print o.next()
# step 3
# 5

print o.next()
#Traceback (most recent call last):
#  File "<stdin>", line 1, in <module>
#StopIteration
```


## 函数式编程

函数式编程就是一种抽象程度很高的编程范式，纯粹的函数式编程语言编写的函数没有变量，因此，任意一个函数，只要输入是确定的，输出就是确定的，这种纯函数我们称之为没有副作用。而允许使用变量的程序设计语言，由于函数内部的变量状态不确定，同样的输入，可能得到不同的输出，因此，这种函数是有副作用的。

函数式编程的一个特点就是，允许把函数本身作为参数传入另一个函数，还允许返回一个函数！

Python对函数式编程提供部分支持。由于Python允许使用变量，因此，Python不是纯函数式编程语言。

### 高阶函数

函数在 python 里是第一类值，它与普通的值比如 int, string 一样可以传递给变量、可以作为函数参数、可以作为返回值。

例如下面的代码：

```py
# abs 只是一个变量，它存储的是计算绝对值函数的这个值，因此我们可以把函数传给别的变量
abs(-10)
my_abs = abs
my_abs(-10)

# abs 这个变量中的函数值可以作为参数传给其他函数
def add(x, y, fun):
    return fun(x) + fun(y)

add(10, -10, abs)

# abs 这个变量中的函数值可以作为返回值返回
def get_fun()
    return abs

fun = get_fun()
fun(-10)
```

高阶函数(High-order function) 就类似于上面例子中的 `get_fun` 函数，它接受另外一个函数作为参数，这样的函数就称为高阶函数。

我们接下来看一下 python 中提供了哪些高阶函数，由此来体会高阶函数的魅力。 

#### map/reduce

Python 内建了 map/reduce 高阶函数。

如果你读过Google的那篇大名鼎鼎的论文“[MapReduce: Simplified Data Processing on Large Clusters](http://research.google.com/archive/mapreduce.html)”，你就能大概明白map/reduce的概念。

`map(f, list)` 有两个参数，第一个参数是一个函数，另一个是一个序列。map 会把 f 依次作用到 list 的每个元素上，然后返回一个新的 list. 例如下面的例子将一串数字依次计算 x*x ：

```py
def f(x):
    return x*x

ret = map(f, [1, 2, 3, 4, 5, 6, 7, 8, 9])
```

你当然也可以用循环的方式来完成上述计算，但是这种方法没能清晰地表达出 “把f(x)作用在list的每一个元素并把结果生成一个新的list” 这一过程。

map 作为一个高阶函数，事实上它把运算规则抽象了，因此我们不但可以计算 `f(x) = x^2` 还可以计算任意复杂的函数，比如下面的代码把 list 中所有数字转换为字符串：

```py
char_list map(str, [1, 2, 3, 4, 5, 6])
print "".join(char_list) # "".join() 可以把字符列表转换为字符串
```

`reduce(f, list)` 有两个参数，第一个参数是一个函数 f, 另一个是一个序列 list. 其中 f 必须接受两个参数，并且只有一个返回值。 recuce 会先将 f 作用到 list 的前两个元素上，然后将 f 的返回值和 list 的下一个元素作为参数传给 f 继续进行计算，从而把计算结果累积起来。例如我们熟悉的 sum 操作可以这样来实现：

```py
def add(x, y):
    return x+y

reduce(add, [1, 2, 3, 4, 5, 6])
```

上面的例子没啥用处，结合 map/reduce 我们可以写出 str 转 int 的函数：

```py
# str 转 int 分为两步：
# 第一步是把 str 中的每个字符都转换为 int 数字。这可以用 map 来实现；
# 第二步是把 每个 int 数字叠加起来，计算出最终的数字，这可以用 reduce 来实现；

def str_to_num(str):
    def char_to_num(ch):
        return {'0':0, '1':1, '2':2, '3':3, '4':4, '5': 5, '6': 6, '7': 7, '8': 8, '9': 9}[ch]

    def fn(x, y):
        return x*10 + y
    return reduce(fn, map(char_to_num, str))

print str_to_num('123456')
```

reduce 抽象的运算规则是对一个列表中的元素进行累积计算。

#### filter

python 内置的 `filter()` 高阶函数用于过滤序列

`filter(f, list)` 也有两个参数， f 会作用到 list 的每个元素上，根据 f 的返回值为 True/False, filter 决定保留或删除这个元素。例如：

```py
def not_emptys(ch):
    return ch and ch.strip()

print filter(not_empty, ['A', '', 'B', None, 'C', ' '])
# 输出 ['A', 'B', 'C']
```

#### sorted

python 内置的排序函数 sorted 也是高阶函数。它可以接受一个比较函数来实现自定义排序：

```py
# 默认的字符串比较方法中， Z 会排在 a 前面，我们修改比较函数，使他忽略大小写
def cmp_ignore_case(s1, s2):
    u1 = s1.upper()
    u2 = s2.upper()
    if u1 < u2:
        return -1
    if u1 > u2:
        return 1
    return 0

print sorted(['bob', 'about', 'Zoo', 'Credit'], cmp_ignore_case)
# 输出：['about', 'bob', 'Credit', 'Zoo']
```

### 返回函数

python 中的函数是第一类值，因此可以作为返回值返回：

```py
def lazy_sum(*args):
    def sum()
        ax = 0
        for n in args:
            ax = ax + n
        return ax
    return sum

# 得到一个函数
f = lazy_sum(1,2,3,4,5)

# 使用函数进行计算
print f()
```

注意上述代码中, `lazy_sum` 在返回函数时，将局部变量 args 和 sum 函数一起作为返回值返回了，这种把局部变量和函数打包在一起的方式叫做 “闭包”。

### 匿名函数

python 中对匿名函数提供了有限支持，在 python 中, `lambda` 表示匿名函数，且看下面的例子：

```py
print map (lambda : x*x, [1, 2, 3, 4, 5, 6])
```

上述代码中的 `lambda` 就是匿名函数。在 python 中，匿名函数中只能有一个表达式，并且没有返回值，这个表达式的计算结果就是返回值。

### 装饰器

假如我们希望在运行期间增强某个函数的功能，比如在函数调用前后自动打印日志，但是又不希望修改函数原先的定义，这种在代码运行期间动态增加功能的方式，称之为“装饰器”(Decorator).

本质上， decotator 就是一个返回函数的高阶函数。例如下面的代码定义了一个能够自动打印日志的装饰器：

```py
def log(func)
    def wrapper(*args, **kw)
        print "%s Enter" % func.__name__
        return func(*args, **kw)
    return wrapper
```

`log` 函数就是我们定义的装饰器，那么怎么使用它来装饰另外一个函数呢？请看下面的代码：

```py
@log
def now()
    print '2017-03-21' 

print now()
# 输出内容：
# now Enter
# 2017-03-21
```

要使用我们定义的装饰器，只需要在函数定义的前面加上 `@log` 即可，相当于 `now = log(now)`。

如果我们希望 `log()` 这个装饰器能够接收参数要怎么做呢？因为默认的 `@log` 语法并不支持传入参数，所以我们可以先定义一个高阶函数，这个高阶函数的作用是返回真正的装饰器：

```py
def log(text):
    def decorator(func):
        def wrapper(*args, **kw):
            print "%s Enter text = %s" % func.__name__, text
            return func(*args, **kw)
        return wrapper
    return decorator

@log('adsf')
def now():
    return '2017-03-21'

print now()
# 输出内容：
# now Enter text = asdf
# 2017-03-21
```

`@log('asdf')` 相当于 `now = log('asdf')(now)`

上述代码已经说明了装饰器的作用，但存在一个问题，如果我们这个时候查看 `now.__name__` 属性，会发现它的名字是 `wrapper` 而不是 `now`, 对于一些依赖于函数名的代码可能会有问题。这时候我们可以利用 python 内置的 `functools.wraps` 这个装饰器来复制 `now()` 的属性到 `wrapper()` 中：

```py
import functools

def log(func)
    @functools.wraps(func)
    def wrapper(*args, **kw)
        print 'call %s' % func.__name__
        return func(*args, **kw)
    return wrapper
```

### 偏函数

functools 模块还提供了一个偏函数(partial function)的功能。偏函数可以为函数中的参数设定一个默认值，然后生成一个新的函数：

```py
# int 函数用于字符串转整形，它有一个额外的参数，可以指定目标进制
int('123456', 16)

# 每次转换为 16 进制都要多填一个参数，有点麻烦，偏函数可以直接为我们生成一个新的函数，并帮我们填好 16 这个参数：
import functools
int16 = functools.partial(int, base = 16)

# 直接调用即可，不需要写目标进制这个参数了
int16('1234566')
```

偏函数可以生成一个新的函数，并向原函数传入固定的值，从而方便我们对函数的调用。


## 模块

python 支持以模块的方式对代码进行管理。对于 python 而言，一个 .py 文件就是一个模块，我们可以使用 import 关键字来导入其他的 .py 文件：

```py
# my_module.py 文件

# -*- coding: utf-8 -*-

# 文档注释，任何模块代码的第一行字符串都被视为模块的文档注释
' a test module '

# 模块的作者
__author__ = 'Dong dada'

# 我们模块中的一个函数
def foo():
    print 'foo'
```

`my_module.py` 文件就是我们所编写的模块，现在我们看看怎么在别的文件中使用它：

```py
# test.py 文件

# 导入 my_module 模块
import my_module

# 使用 my_module 模块中的 foo 函数
my_module.foo()
```

可以看到模块的定义和使用都非常简单。

这里需要考虑一个问题，如果我们的模块名 `my_module` 和别的模块冲突了怎么办？python 引入了按照目录来组织模块的方法，称为包(package). 我们需要把自己的模块放到一个文件夹中，例如`dongdada_modules`, 然后在这个文件夹中放入我们的 `.py` 模块文件：

```
dongdada_modules
|
+-- __init__.py
|
+-- my_module.py
|
+-- abc.py
```

可以看到上述目录中还有一个 `__init__.py` 文件，它的作用是告诉 python 这个文件夹是一个 package. 这个文件可以使空文件，也可以包含一些代码，当使用者调用 `import dongdada_modules` 的时候，会导入 `__init__.py` 文件中的内容，用户这时候如果需要导入 `my_module.py`，需要在 import 的时候加上包名：`import dongdada_modules.my_module`。

包的下面可以不只有一个文件夹，文件夹可以有多个层级，导入的时候按照层级来逐层导入。

### 使用模块

#### 别名

导入模块时，可以使用别名，这样可以根据当前环境选择最合适的模块。比如 python 一般会提供 StringIO 和 cStringIO 这两个库，cStringIO 用 C 语言编写，速度更快，但不是所有的环境上都有这个库，这个时候就可以通过别名的方式先尝试导入 cStringIO, 如果导入失败则使用 StringIO:

```py
try:
    import cStringIO as StringIO
except ImportError:
    import StringIO
```

通过别名的方式，不管我们导入的模块是 cStringIO 还是 StringIO, 后续的代码都能够正常工作。

#### 作用域

python 模块中定义的变量或函数，具有作用域的概念。对于普通的变量或函数而言，其作用域是公开的，如果你希望隐藏某些变量或函数，可以使用类似 `__xxx`, `_xxx` 这样的变量名可以表示该变量为非空开的变量，避免外部使用到它；

### 使用第三方模块

在 python 中，安装第三方模块是通过 setuptools 这个工具来完成的。python 封装了两个 setuptools 工具： `easy_install` 和 `pip`, 官方推荐使用 `pip` 工具。你可在安装 python 环境时注意一下，勾选 pip 这个工具。

python 下有一个非常强大的第三方库 python image library. 我们可以使用下述命令来下载安装它：

```
pip install PIL
```

安装完毕后，我们就可以在代码中使用它了：

```py
import Image

img = Image.open('test.png')
print img.format img.size img.mode

# 生成图片的缩略图
img.thumbnail((200, 100))
img.save('thumb.jpg', 'JPEG')
```

上述代码在 64 位 windows 下会执行失败，这是因为 PIL 只提供了 32 位版本，你可以自行下载 PIL 的安装包来手动安装这个库。

#### 模块搜索路径

当我们试图加载模块时，python 会按照 当前目录-->内置模块目录-->第三方模块目录 这样的顺序来搜索模块。如果我们的模块没有在这些目录里，搜索就会失败。

这些路径都记录在 `sys.path` 这个变量里面：

```py
import sys
print sys.path

# 输出
# ['', 'C:\\Windows\\system32\\python27.zip', 'D:\\Python27\\DLLs', 'D:\\Python27\
# \lib', 'D:\\Python27\\lib\\plat-win', 'D:\\Python27\\lib\\lib-tk', 'D:\\Python27
#', 'D:\\Python27\\lib\\site-packages', 'D:\\Python27\\lib\\site-packages\\PIL']
```

我们可以通过修改这个路径来添加自己的模块路径。

如果觉得修改 `sys.path` 不太好，也可以设置 `PYTHONPATH` 环境变量来添加。

### 使用 __future__

python 提供了 `__future__` 模块，可以把新版本的特性导入到当前版本。例如：

```py
# still running on Python 2.7

from __future__ import unicode_literals

print '\'xxx\' is unicode?', isinstance('xxx', unicode)
print 'u\'xxx\' is unicode?', isinstance(u'xxx', unicode)
print '\'xxx\' is str?', isinstance('xxx', str)
print 'b\'xxx\' is str?', isinstance(b'xxx', str)

# 输出如下：
# 'xxx' is unicode? True
# u'xxx' is unicode? True
# 'xxx' is str? False
# b'xxx' is str? True
```

`unicode_literals` 这个模块可以使用 python3.x 中关于字符串的新语法。在 python2.x 中，unicode 字符串前面需要加上 u, 普通的字符串不加 u. 而在 python3.x 中，字符串默认就是 Unicode 的，使用 `b'xxx'` 这样的形式来表示窄字节版本。

类似的，在 python2.x 中，整数间的除法得到的结果也是整数，而在 python3.x 中，整数除法如果不能整除，得到的是一个浮点数：

```py
from __future__ import division

print '10 / 3 =', 10 / 3
print '10.0 / 3 =', 10.0 / 3
print '10 // 3 =', 10 // 3

# 输出如下：
# 10 / 3 = 3.33333333333
# 10.0 / 3 = 3.33333333333
# 10 // 3 = 3
```

`//` 表示之前的 “地板除” 方式，得到的仍然是一个整数。


## 面向对象编程

python 支持面向对象方式的编程：

```py
# 定义类
class Students(object):
    def __init__(self, name, score):
        self.name = name
        self.score = score
    def print_score(self):
        print '%s: %s' % (self.name, self.score)

# 创建对象
bart = Student('Bart Simpson', 59)
lisa = Student('Lisa Simpson', 87)

# 访问对象的成员变量
print bart.name
print lisa.name

# 调用对象的成员函数
bart.print_score()
lisa.print_score()
```

上述代码中有一些需要注意的地方：
- `class Student(object):` 这一句中的 `object` 是 Student 类的基类；
- `__init__` 可以看做这个类的构造函数；
- 类的所有成员函数的第一个参数都是 self, self 表示创建的实例本身，类似于 C++ 中的 this, 我们可以把各种属性设定到 self 上面；

### 成员的可见性

如果我们希望隐藏成员变量的可见性，可以将成员变量命名为 `__xxx` 的形式，例如：

```py
class Students(object):
    def __init__(self, name, score):
        self.__name = name
        self.__score = score

    def print_score(self):
        print '%s: %s' % (self.__name, self.__score)
    
    def set_name(self, name):
        self.__name = name
    
    def get_name(self)
        return self.__name
```

### 继承和多态

正如之前所说, python 中的类也支持继承：

```py
class Animal(object):
    def run(self):
        print 'Animal is running...'

class Cat(Animal):
    pass

class Dog(Animal):
    def run(self):
        print 'Dog is running...'

cat = Cat()
dog = Dog()

cat.run()  # 输出 Animal is running ...
dog.run()  # 输出 Dog is running ... 
```

可以看到，因为 Cat 类没有复写基类 Animal 的 run 方法，因此调用 `Cat.run()` 时执行的是基类的方法；而 Dog 复写了 Animal 的 run 方法，调用 `Dog.run()` 时执行的是 Dog 类的 run 方法。

### 获取对象信息

当我们获取到一个对象的引用时，可以通过下述方法来得到关于这个对象的一些信息：

#### type() 方法获取对象类型

```py
type(123)
# <type 'int'>

type('str')
# <type 'str'>

type(None)
#<type 'NoneType'>

type(abs)
# <type 'builtin_function_or_method'>

a = Animal()
type(a)
# <class '__main__.Animal'>
```

#### isinstance() 判断继承关系

```py
isinstance(dog, Animal)
# True

isinstance(animal, Dog)
# False
```

#### dir() 获取一个对象的所有属性和方法

```py
dir('ABC')
# ['__add__', '__class__', '__contains__', '__delattr__', '__doc__', '__eq__', '__format__', '__ge__', '__getattribute__', '__getitem__', '__getnewargs__', '__getslice__', '__gt__', '__hash__', '__init__', '__le__', '__len__', '__lt__', '__mod__', '__mul__', '__ne__', '__new__', '__reduce__', '__reduce_ex__', '__repr__', '__rmod__', '__rmul__', '__setattr__', '__sizeof__', '__str__', '__subclasshook__', '_formatter_field_name_split', '_formatter_parser', 'capitalize', 'center', 'count', 'decode', 'encode', 'endswith', 'expandtabs', 'find', 'format', 'index', 'isalnum', 'isalpha', 'isdigit', 'islower', 'isspace', 'istitle', 'isupper', 'join', 'ljust', 'lower', 'lstrip', 'partition', 'replace', 'rfind', 'rindex', 'rjust', 'rpartition', 'rsplit', 'rstrip', 'split', 'splitlines', 'startswith', 'strip', 'swapcase', 'title', 'translate', 'upper', 'zfill']
```

除了可以获取所有属性，还可以使用 `getattr()`, `setattr()`, `hasattr()` 这几个方法来对属性进行访问：

```py
def readImage(fp):
    if hasattr(fp, 'read'):
        return readData(fp)
    return None
```

## 面向对象高级编程

### 使用 __slots__

在 python 中，我们可以对 类实例、类本身 的成员进行修改：

```py
class Student(object):
    pass

bart = Student()

# 为 bart 这个实例新增一个成员变量 name
bart.name = 'Bart'

def print_name(self):
    print self.name

# 为 bart 这个实例新增一个成员函数 print_name
bart.print_name = print_name

def set_score(self, score):
    self.score = score

# 为 Student 类新增一个成员函数 set_score
Student.set_score = set_score

lisa = Student()
lisa.set_score(100)
```

可以看到 python 不仅支持动态修改实例的成员，还支持动态修改类本身的成员。

这个功能虽然很强大，但有可能引入我们不需要的成员，如果我们想要限制使用者不能随意添加成员，可以在定义类的时候添加一个特殊的变量 `__slots__`：

```py
class Student(object):
    __slots__('name', 'score')
    pass

bart = Student()
bart.age = 10 # 出错！不允许添加 age 成员
```

`__slots__` 限制了外部添加属性的能力，有了它之后外部只能添加固定的几个成员。

### 使用 @property

我们常常会在类中添加 get/set 方法，以便对类成员进行访问：

```py
class Student(object):
    
    def get_score(self)
        return self.__score
    
    def set_score(self, score)
        if not isinstance(score, int):
            raise ValueError('score must be an integer!')
        
        if score < 0 or score > 100:
            raise ValueError('score must between 0 ~ 100!')

        self.__score = score
```

set 方法中常常会包括一些参数合法性检查，避免用户直接访问 `self.__score` 时输入不合法的数据。

python 内置了一个叫做 `@property` 的装饰器，可以自动生成 get/set 方法，使得用户可以通过 `self.score` 来访问属性，同时又能有边界检查的功能：

```py
class Student(object):

    @property
    def score(self):
        return self.__score
    
    @score.setter
    def score(self, value)
        if not isinstance(value, int):
            raise ValueError('score must be an integer!')
        
        if value < 0 or value > 100:
            raise ValueError('score must between 0 ~ 100!')

        self.__score = value

# 外部可以直接通过 self.score 属性来访问
bart = Student()
bart.score = 100
bart.score = 9999 # 错误！'score must between 0 ~ 100!'
```

如果你没有定义 `@score.setter` 那么 score 就会变成一个只读属性。

### 多重继承

python 中的类也支持多重继承：

```py
class Runable(object):
    pass

class Flyable(object):
    pass

# 多重继承
class Chicken(Animal, Runable, Flyable)
    pass
```

可以看到 Chicken 多重继承了 Runable 和 Flyable 这两个基类。python 中的多重继承与 C++ 中的多重继承并无不同，特殊的地方在于这里可以使用一种叫 Mixin 的设计模式，如果我们希望为一个类添加更多的功能，就可以通过上述例子中的方法，只需要把这些功能的类添加到继承列表里就可以了。

### 定制类

我们之前介绍了 `__slots__` 这个函数。遇到这种形如 `__xxx__` 格式的函数需要注意，它们具有特殊的用途。python 的 class 还有许多这样具有特殊用途的函数。

#### __str__

自定义 `__str__` 函数可以让我们返回一个自定义的字符串，来标识这个 class:

```py
class Student(object):
    def __init__(self, name):
        self.name = name
    def __str__(self)
        return 'Student object (name : %s)' % self.name

print Student('Tracy')
# 输出： Student object (name : Tracy)
```

`__repr__` 函数的功能与 `__str__` 类似，它是给程序调试用的。

#### __iter__

自定义 `__iter__` 可以让类实例能够被 `for...in` 来遍历：

```py
class Fib(object):
    def __init__(self):
        self.a, self.b = 0, 1
    
    def __iter__(self):
        return self # 实例本身就是迭代对象，所以返回自己
    
    def next(self):
        self.a, self.b = self.b, self.a + self.b
        if self.a > 100000:
            raise StopIteration()
        return self.a

for n in Fib():
    print n
```

#### __getitem__

自定义 `__getitem__` 可以让类实例能够像 list 那样用下标取数据：

```py
class Fib(object):
    def __getitem__(self, n):
        if isinstance(n, int):
            a, b = 1, 1
            for x in range(n):
                a, b = b, a + b
            return a
        if isinstance(n, slice): # 如果传入的是一个 slice, 那么需要支持切片
            start = n.start
            stop = n.stop
            a, b = 1, 1
            L = []
            for x in range(stop):
                if x >= start:
                    L.append(a)
                a, b = b, a + b
            return L
```

#### __getattr__

如果自定义了 `__getattr__` 方法，那么当使用者尝试访问一个不存在的属性时，可以返回一个我们自己定义的内容：

```py
class Student(object):

    def __init__(self):
        self.name = 'Michael'

    def __getattr__(self, attr):
        if attr=='score':
            return 99
        else:
            raise AttributeError('\'Student\' object has no attribute \'%s\'' % attr)

stu = Student()
print stu.score # 输出 99
```

#### __call__

如果自定义了 `__call__` 方法，那么使用者可以直接对实例调用方法：

```py
class Student(object):
    def __init__(self, name):
        self.name = name

    def __call__(self):
        print('My name is %s.' % self.name)

stu = Student('Tracy')
stu() # 输出 My name is Tracy.
```

### 使用元类

这一节有点复杂，又不怎么会用到，这里先略过吧，具体可以参考 [原文](http://www.liaoxuefeng.com/wiki/001374738125095c955c1e6d8bb493182103fac9270762a000/001386820064557c69858840b4c48d2b8411bc2ea9099ba000)


## 错误、调试和测试

### 错误处理

在程序运行过程中，可以通过返回错误码的方式来表示代码是否运行正常：

```py
def foo():
    r = some_function()
    if r==(-1):
        return (-1)
    # do something
    return r

def bar():
    r = foo()
    if r==(-1):
        print 'Error'
    else:
        pass
```

可以看到，这种错误处理方式不太方便，因为错误码会跟正常的结果混在一起，调用者必须通过大量代码来判断是否出错。

#### try

python 提供了 `try` 机制：

```py
try:
    print 'try...'
    r = 10 / int('a')
    print 'result:', r
except ValueError, e:
    print 'ValueError:', e
except ZeroDivisionError, e:
    print 'ZeroDivisionError:', e
else:
    print 'no error!'
finally:
    print 'finally...'
print 'END'
```

当我们觉得某段代码可能抛出异常时，可以通过 `try` 来包裹这段代码，如果代码出错，程序会运行到 `except` 处，无论是否成功，都会执行 `finally` 代码。如果错误没有发生，你可以在 `except` 后面跟一个 `else` 专门来处理这种情况。

注意，一般情况下，程序发生异常后会立刻停止运行，但如果我们用 `try...except` 包裹住了出现异常的代码，那么即使发生异常，程序也会在执行完 except 之后继续执行。

python 中的异常也是一个 class, 所有的异常都是从 BaseException 继承的，你可以从 [这里](https://docs.python.org/2/library/exceptions.html#exception-hierarchy) 查看到常见的异常类型和继承关系。

`try...except` 这种机制还有一个巨大的好处，当存在多个层次的调用时，深层的异常会一层层抛到外部，你只需要在合适的层次捕获这个异常就可以了，不需要在每层都写 `try..except` 代码：

```py
def foo(s):
    return 10 / int(s)

def bar(s):
    return foo(s) * 2

def main():
    try:
        bar('0')
    except StandardError, e:
        print 'Error!'
    finally:
        print 'finally...'
```

#### 记录错误

当异常发生导致程序终止时，python 解释器会在终端上打印错误的调用堆栈。方便我们定位异常发生的位置。

python 提供了一个 logging 模块，可以让我们在程序出现异常时，将异常信息打印出来：

```py
# err.py
import logging

def foo(s):
    return 10 / int(s)

def bar(s):
    return foo(s) * 2

def main():
    try:
        bar('0')
    except StandardError, e:
        logging.exception(e)

main()
print 'END
```

#### 抛出错误

我们也可以在自己编写的函数中抛出异常：

```py
# err.py
class FooError(StandardError):
    pass

def foo(s):
    n = int(s)
    if n==0:
        raise FooError('invalid value: %s' % s)
    return 10 / n
```

上述代码中自定义了一个异常类型 FooError, 并在异常发生时抛出了这个异常。

一般情况下我们应当尽量选择 python 内置的异常类型，默认的异常类型具有更高的辨识度，方便我们快速确定造成异常的原因。

接着请看下述代码：

```py
# err.py
def foo(s):
    n = int(s)
    return 10 / n

def bar(s):
    try:
        return foo(s) * 2
    except StandardError, e:
        print 'Error!'
        raise

def main():
    bar('0')

main()
```

程序捕获到一个异常后，打印了一下，接着又把它抛了出去。这种做法相当常见，因为异常发生时，当前函数可能不知道该如何处理这个异常，因此在这里记录一下，抛给上层来处理。

### 调试

#### assert 断言

下面是 python 中使用断言的一个例子：

```py
# err.py
def foo(s):
    n = int(s)
    assert n != 0, 'n is zero!'
    return 10 / n

def main():
    foo('0')
```

#### logging

下面是 python 中使用 logging 模块的一个例子：

```py
# err.py
import logging
logging.basicConfig(level=logging.INFO)

s = '0'
n = int(s)
logging.info('n = %d' % n)
print 10 / n
```

#### 单步调试和设置断点

在运行 python 脚本时加上 `-m pdb` 参数，就能以单步的方式来运行程序：

```
$ python -m pdb err.py
```

进入调试后，有以下命令来对调试过程进行控制：
- n : 单步执行代码；
- l : 查看代码；
- `p 变量名` : 查看变量的值；
- q : 结束调试，退出程序；

利用 pdb 模块的 `pdb.set_trace()` 方法，可以在代码中设置断点：

```py
# err.py
import pdb

s = '0'
n = int(s)
pdb.set_trace() # 运行到这里会自动暂停
print 10 / n
```


## IO 编程

IO在计算机中指Input/Output，也就是输入和输出。由于程序和运行时数据是在内存中驻留，由CPU这个超快的计算核心来执行，涉及到数据交换的地方，通常是磁盘、网络等，就需要IO接口。

IO编程中，Stream（流）是一个很重要的概念，可以把流想象成一个水管，数据就是水管里的水，但是只能 **单向流动**。Input Stream就是数据从外面（磁盘、网络）流进内存，Output Stream就是数据从内存流到外面去。对于浏览网页来说，浏览器和新浪服务器之间至少需要建立两根水管，才可以既能发数据，又能收数据。

IO 操作分为同步和异步两种模式，本章介绍的都是同步 IO

### 文件读写

下面的代码展示了如何读取一个文件中的内容：

```py
try:
    f = open('/path/to/file', 'r')
    print f.read()
finally:
    if f:
        f.close()
```

要对文件进行操作，首先要打开它，文件操作结束后，需要将这个文件关闭。`f.read()` 会返回一个 `str` 对象，由于我们使用 `r` 的方式来打开文件，str 中存储的是文本形式的数据。如果我们用 `rb` 的形式来打开文件，那么 str 中存储的就是二进制数据。

上述代码写起来有些长， python 提供了 `with...as` 这样的简写形式，效果和上面的代码一样：

```py
with open('/path/to/file', 'r') as f:
    f.close()
```

调用 `read()` 会一次性读取文件的全部内容，如果文件有10G，内存就爆了，所以，要保险起见，可以反复调用 `read(size)` 方法，每次最多读取 size 个字节的内容。另外，调用 `readline()` 可以每次读取一行内容，调用`readlines()` 一次读取所有内容并按行返回list。因此，要根据需要决定怎么调用。

如果文件很小，`read()` 一次性读取最方便；如果不能确定文件大小，反复调用 `read(size)` 比较保险；如果是配置文件，调用 `readlines()` 最方便：

#### file-like Object

像 `open()` 函数返回的这种有个 `read()` 方法的对象，在 Python 中统称为 `file-like Object` 。除了 file 外，还可以是内存的字节流，网络流，自定义流等等。 `file-like Object` 不要求从特定类继承，只要写个 `read()` 方法就行。

`StringIO` 就是在内存中创建的 `file-like Object`，常用作临时缓冲。

#### 字符编码

要读取非 ASCII 编码的文本文件，就必须以二进制模式打开，再解码。比如 GBK 编码的文件：

```py
f = open('/Users/michael/gbk.txt', 'rb')
u = f.read().decode('gbk')
```

python 还提供了 `codecs` 模块来方便地执行上述操作：

```py
import codecs
with codecs.open('/Users/michael/gbk.txt', 'r', 'gbk') as f:
    f.read() # u'\u6d4b\u8bd5'
```

在 python 中，使用 `with...as` 来操作文件是一个好习惯。

### 操作文件和目录

python 中与操作系统打交道的函数封装在 os 模块中。

`os.environ` 和 `os.getenv()` 可以访问环境变量：
```py
import os

print os.environ
print os.getenv('PATH')
```

`os.path` 模块封装了一些与路径相关的函数：

```py
# 获取绝对路径
print os.path.abspath('.')

# 拼接路径
print os.path.join('/Users/michael', 'testdir') # 输出 /Users/michael/testdir

# 分割路径得到文件名
print os.path.split('/Users/michael/testdir/file.txt') # 输出 ('Users/michael/testdir/', 'file.txt')

# 分割路径得到扩展名
print os.path.splitext('/path/to/file.txt') # 输出 ('Users/michael/testdir/file', '.txt')
```

上述函数不会考虑文件是否存在，它只是对字符串进行分割。

`os` 模块中还有一些函数可以对文件、文件夹进行创建、删除、重命名等操作：

```py
# 创建一个目录
os.mkdir('/Users/michael/testdir')

# 删掉一个目录:
os.rmdir('/Users/michael/testdir')

# 对文件重命名:
os.rename('test.txt', 'test.py')

# 删掉文件:
os.remove('test.py')
```

python 没有提供复制文件的操作，这是因为这个函数在操作系统上没有直接提供。`shutil` 提供了 `copyfile()` 操作，你可以在 `shutil` 模块中找到许多实用的函数，它可以被看做是 `os` 模块的补充。

结合列表生成式，我们可以写出非常简单的代码来过滤文件夹中的内容：

```py
print [x for x in os.listdir('.') if os.path.isfile(x) and os.path.splitext(x)[1]=='.py']
```

### 序列化

python 中把序列化称为 pickling, 反序列化则称为 unpickling.

python 中提供了两个模块来实现序列化: `pickle` 和 `cPickle`, 这两个模块的功能是一样的，区别仅在于 `cPickle` 由 C 语言编写，速度比较快。

```py
try:
    import cPickle as pickle
except ImportError:
    import pickle

d = dict(name='Bob', age=20, score=88)

# 序列化到字符串
string = pickle.dumps(d)

# 序列化到文件
f = open('dump.txt', 'wb')
pickle.dump(d, f)
f.close()

# 从文件中反序列化出对象
f = open('dump.txt', 'rb')
d = pickle.load(f)
f.close()
```

#### JSON

如果希望序列化出的内容能够交给不同语言处理，就必须把对象序列化为标准格式，例如 xml, json.

```py
import json

d = dict(name='Bob', age=20, score=88)

# 序列化
print json.dumps(d) # 输出 '{"age": 20, "score": 88, "name": "Bob"}'

# 反序列化
json_str = '{"age": 20, "score": 88, "name": "Bob"}'
d = json.loads(json_str)
```

有一点需要注意的是，反序列化得到的 dict 中，所有字符串对象都是 unicode 编码，类似于 `u'age'` 这样。

#### JSON 进阶

如果我们希望序列化一个类对象为 json, 会遇到错误：

```py
import json

class Student(object):
    def __init__(self, name, age, score):
        self.name = name
        self.age = age
        self.score = score

s = Student('Bob', 20, 88)
print(json.dumps(s)) # 错误! TypeError: <__main__.Student object at 0x10aabef50> is not JSON serializable
```

可以看到类对象不是可以序列化的类型。

为了解决这个问题，我们需要编写一个函数，告诉 json 模块如何对这个类对象进行序列化：

```py
def student2dict(std):
    return {
        'name': std.name,
        'age': std.age,
        'score': std.score
    }

print(json.dumps(s, default=student2dict))
```

每个类都这样写感觉有点麻烦，我们可以通过下面的代码来支持任意 class 的序列化：

```py
print(json.dumps(s, default=lambda obj: obj.__dict__))
```

通常每个 class 的实例都有一个 `__dict__` 属性，用来存储成员变量。也有少数例外，例如定义了 `__slots__` 的 class.

同样的，我们也需要传入一个函数来支持将 json 文本反序列化为一个类实例：

```py
def dict2student(d):
    return Student(d['name'], d['age'], d['score'])

json_str = '{"age": 20, "score": 88, "name": "Bob"}'
print(json.loads(json_str, object_hook=dict2student))
```


## 进程和线程

python 提供了多进程和多线程的支持，接下来我们会讨论如何编写这两种多任务程序。

### 多进程

Unix/Linux 系统提供了一个 `fork()` 系统调用，它能够创建一个子进程, python 的 os 模块中也包含了 `fork()` 函数：

```py
# multiprocessing.py
import os

print 'Process (%s) start...' % os.getpid()
pid = os.fork()
if pid==0:
    print 'I am child process (%s) and my parent is %s.' % (os.getpid(), os.getppid())
else:
    print 'I (%s) just created a child process (%s).' % (os.getpid(), pid)
```

有了 fork 调用，一个进程在接到一个新任务时就可以复制出一个子进程来处理任务，常见的 Apache 服务器就是由父进程监听端口，每当有新 http 请求时，就 fork 出子进程来处理新的 http 请求。

#### multiprocessing 模块

由于 windows 没有 `fork` 调用，上述代码无法在 windows 下运行. python 的 multiprocessing 模块提供了跨平台的多进程支持：

```py
from multiprocessing import Process
import os

# 子进程要执行的代码
def run_proc(name):
    print 'Run child process %s (%s)...' % (name, os.getpid())

if __name__=='__main__':
    print 'Parent process %s.' % os.getpid()
    p = Process(target=run_proc, args=('test',)) # 一个 Process 对象就代表一个进程
    print 'Process will start.'
    p.start() # 启动进程
    p.join()  # 等待进程执行结束
    print 'Process end.'
```

#### 进程池

如果要启动大量的子进程，可以用进程池的方式来批量创建子进程：

```py
from multiprocessing import Pool
import os, time, random

def long_time_task(name):
    print 'Run task %s (%s)...' % (name, os.getpid())
    start = time.time()
    time.sleep(random.random() * 3)
    end = time.time()
    print 'Task %s runs %0.2f seconds.' % (name, (end - start))

if __name__=='__main__':
    print 'Parent process %s.' % os.getpid()
    p = Pool() # 一个 Pool 对象代表一个进程池，进程池中的进程数默认与 CPU 内核数有关，你可以自己指定进程数，只需要在创建时传入一个数字即可，例如 p = Pool(5)
    for i in range(5):
        p.apply_async(long_time_task, args=(i,)) # 在进程池中执行任务
    print 'Waiting for all subprocesses done...'
    p.close() # 调用 close 表示以后不能再向这个池中添加新的任务
    p.join()  # 等待进程池中所有进程都执行结束，必须在 close 之后才能调用
    print 'All subprocesses done.'
```

#### 进程间通信

`multiprocessing` 模块包装了进程间通信的底层机制，提供了 `Queue`, `Pipes` 等多种方式来交换数据。

下面的代码中有一个读进程，一个写进程。分别对同一个 Queue 进行操作

```py
from multiprocessing import Process, Queue
import os, time, random

# 写数据进程执行的代码:
def write(q):
    for value in ['A', 'B', 'C']:
        print 'Put %s to queue...' % value
        q.put(value)
        time.sleep(random.random())

# 读数据进程执行的代码:
def read(q):
    while True:
        value = q.get(True)
        print 'Get %s from queue.' % value

if __name__=='__main__':
    # 父进程创建Queue，并传给各个子进程：
    q = Queue()
    pw = Process(target=write, args=(q,))
    pr = Process(target=read, args=(q,))
    # 启动子进程pw，写入:
    pw.start()
    # 启动子进程pr，读取:
    pr.start()
    # 等待pw结束:
    pw.join()
    # pr进程里是死循环，无法等待其结束，只能强行终止:
    pr.terminate()
```

### 多线程

python 标准库中提供了两个模块用于对线程的操作：`thread` 和 `threading`, `thread` 是比较底层的模块，`threading` 比较高层，对 `thread` 进行了封装。一般我们只需要使用 `threading` 模块即可。

下面是创建一个线程的例子：

```py
import time, threading

# 新线程执行的代码:
def loop():
    print 'thread %s is running...' % threading.current_thread().name
    n = 0
    while n < 5:
        n = n + 1
        print 'thread %s >>> %s' % (threading.current_thread().name, n)
        time.sleep(1)
    print 'thread %s ended.' % threading.current_thread().name

print 'thread %s is running...' % threading.current_thread().name
t = threading.Thread(target=loop, name='LoopThread') # 创建一个新线程，执行 loop 函数，线程名为 LoopThread
t.start() # 启动新线程
t.join() # 等待新线程执行完毕
print 'thread %s ended.' % threading.current_thread().name
```

#### 锁

使用 `threading.Lock()` 函数可以获得一把互斥锁，用来保护线程间共享的数据：

```py
balance = 0
lock = threading.Lock()

def run_thread(n):
    for i in range(100000):
        # 先要获取锁:
        lock.acquire()
        try:
            # 放心地改吧:
            change_it(n)
        finally:
            # 改完了一定要释放锁:
            lock.release()
```

#### 多核

下面的代码在 python 中运行，并不能让 cpu 的所有核心都跑满 100%, 这是为什么呢？

```py
import threading, multiprocessing

def loop():
    x = 0
    while True:
        x = x ^ 1

for i in range(multiprocessing.cpu_count()):
    t = threading.Thread(target=loop)
    t.start()
```

因为Python的线程虽然是真正的线程，但解释器执行代码时，有一个GIL锁：Global Interpreter Lock，任何Python线程执行前，必须先获得GIL锁，然后，每执行100条字节码，解释器就自动释放GIL锁，让别的线程有机会执行。这个GIL全局锁实际上把所有线程的执行代码都给上了锁，所以，**多线程在Python中只能交替执行，即使100个线程跑在100核CPU上，也只能用到1个核**。

GIL是Python解释器设计的历史遗留问题，通常我们用的解释器是官方实现的CPython，要真正利用多核，除非重写一个不带GIL的解释器。

所以，在Python中，可以使用多线程，但不要指望能有效利用多核。如果一定要通过多线程利用多核，那只能通过C扩展来实现，不过这样就失去了Python简单易用的特点。

不过，也不用过于担心，Python虽然不能利用多线程实现多核任务，但可以通过多进程实现多核任务。多个Python进程有各自独立的GIL锁，互不影响。

### ThreadLocal

我们往往需要在子线程中设定一些私有数据。这种时候本应当使用局部变量，但这样会比较麻烦，你不得不把这些局部变量一层一层传给需要它的函数。

python 的 ThreadLocal 可以解决这个问题。

首先 `ThreadLocal` 是一个全局变量，但是它又不太像一个全局变量，倒更像是一个字典。每个线程都可以访问到全局的 ThreadLocal 对象，但是这些线程在访问这个对象的时候，访问的是一个属于当前线程的私有数据，例如下面的代码：

```py
import threading

# 创建全局ThreadLocal对象:
local_school = threading.local()

def process_student():
    print 'Hello, %s (in %s)' % (local_school.student, threading.current_thread().name)

def process_thread(name):
    # 绑定ThreadLocal的student:
    local_school.student = name
    process_student()

t1 = threading.Thread(target= process_thread, args=('Alice',), name='Thread-A')
t2 = threading.Thread(target= process_thread, args=('Bob',), name='Thread-B')
t1.start()
t2.start()
t1.join()
t2.join()
```

`local_school` 是一个全局对象，但它更像是在每个线程中都存在一份拷贝，线程 A 访问的 `local_school` 与线程 B 访问的并不是同一个，`local_school` 在不同线程间是相互独立的内容。你可以认为在每个线程中都有一个自己的 `local_school` 对象。因为 `local_school` 是全局的，因此你不需要像局部对象那样把它一层一层传入给调用方。

### 进程 vs 线程

要实现多任务，我们往往会设计 Master-Worker 模式, Master 负责分配任务, Worker 负责执行任务。

多进程的优势在于稳定性高，一个进程挂掉不会影响另外一个；缺点则在于创建进程的开销比较大，另外进程间的通信相比线程间要低效一些。

多线程的优势在于效率高，但存在稳定性问题。

任务分为 计算密集型 和 IO 密集型。
- 对于计算密集型而言，主要的工作都由 CPU 来承担，因此最好让线程的数量等于 CPU 的核数；
- 对于 IO 密集型而言, CPU 参与的工作并不多，主要时间耗费在 IO 上面，因此线程或进程数多一些也没关系，因为 CPU 有充分的空闲可以去对它们进行调度；

### 分布式进程

请参考 [原文](http://www.liaoxuefeng.com/wiki/001374738125095c955c1e6d8bb493182103fac9270762a000/001386832973658c780d8bfa4c6406f83b2b3097aed5df6000)


## 正则表达式

我们先看看 python 支持的正则表达式语法：

![]({{ site.url }}/asset/python-regex.png)

上图来源于 [这篇文章](http://www.cnblogs.com/huxi/archive/2010/07/04/1771073.html)

可以看到, python 的正则表达式当中 `\` 是一个特殊字符，而 `\` 在 python 字符串中表达了转义的意思，为了避免混淆，我们最好使用 `r'...'` 这样的方式来书写正则表达式，避免 `\` 符号造成困扰。

python 中的 `re` 模块提供了正则表达式的支持，我们逐个介绍它提供的功能。

### 判断是否匹配：

```py
import re

# 匹配则返回一个 match object
match_obj = re.match(r'^\d{3}\-\d{3,8}$', '010-12345')

# 不匹配则返回 None
match_obj =  re.match(r'^\d{3}\-\d{3,8}$', '010 12345')

# 可以直接判断
if re.match(r'表达式', '字符串'):
    print 'ok'
else:
    print 'failed'
```

### 切分字符串：

```py
print re.split(r'[\s\,]+', 'a,b, c  d')
# 输出: ['a', 'b', 'c', 'd']
```

### 提取分组：

在 python 的正则表达式中，用 `()` 括起来的内容就表示要捕获的分组，**如果字符串能够匹配上正则表达式**，那么这些分组中的内容也能够被提取出来：

```py
m = re.match(r'^(\d{3})-(\d{3,8})$', '010-12345')

print m.group(0) # group 的第一个是原始字符串 '010-12345'

print m.group(1) # group 的第二个是匹配到的第一个分组 '010'

print m.group(2) # group 的第三个是匹配到的第二个分组 '12345'
```

### 贪婪匹配

最后需要特别指出的是，正则匹配默认是贪婪匹配，也就是匹配尽可能多的字符。举例如下，匹配出数字后面的0：

```py
re.match(r'^(\d+)(0*)$', '102300').groups()
# ('102300', '')
```

由于 `\d+` 采用贪婪匹配，直接把后面的 0 全部匹配了，结果 `0*` 只能匹配空字符串了。

必须让 `\d+` 采用非贪婪匹配（也就是尽可能少匹配），才能把后面的0匹配出来，加个 `?` 就可以让 `\d+` 采用非贪婪匹配：

```py
re.match(r'^(\d+?)(0*)$', '102300').groups()
# ('1023', '00')
```

### 编译

re 模块在进行匹配的过程中，会首先把正则表达式编译一下，然后用编译过后的数据结构去进行实际的匹配操作。如果我们用同一个正则表达式匹配多次，那么可以先编译这个表达式，接下来都使用编译过后的表达式来进行匹配，这样做可以提高程序的性能：

```py
import re

# 编译:
re_telephone = re.compile(r'^(\d{3})-(\d{3,8})$')

# 使用：
re_telephone.match('010-12345').groups()
# ('010', '12345')
re_telephone.match('010-8086').groups()
# ('010', '8086')
```


## 常用内建模块

python 内置了许多模块可供我们使用，接下来逐个介绍比较有用的模块：

### collections 

`collections` 模块中封装了许多有用的集合类。

#### namedtuple

tuple 表示不可变的数据集合，但它不能通过属性来引用。`namedtuple` 可以很方便地定义出一种数据类型，这种数据类型是 tuple, 但同时可以用属性的方式来访问 tuple 里的数据：

```py
fron collections import namedtuple

# 定义 Point 这种数据类型
Point = namedtuple('Point', ['x', 'y'])

# 创建实例
p = Point(1, 2)

# 通过属性来访问
print p.x, p.y
```

#### deque

`list` 内部的实现其实是数组方式，插入和删除的效率比较低，索引的效率比较高。

如果我们需要对大量数据进行插入删除操作，则可以使用 `deque`, 它还可以方便地用于队列和栈：

```py
from collections import deque
q = deque(['a', 'b', 'c'])
q.append('x')
q.appendleft('y')

print q
# deque(['y', 'a', 'b', 'c', 'x'])
```

#### defaultdict

`defaultdict` 与 `dict` 的不同之处在于，`dict` 中访问某个 key 如果失败，会抛出异常，而 `defaultdict` 则会返回一个默认值：

```py
from collections import defaultdict
dd = defaultdict(lambda: 'N/A')
dd['key1'] = 'abc'
print dd['key1'] # key1存在
# 'abc'

print dd['key2'] # key2不存在，返回默认值
# 'N/A'
```

#### OrderedDict

与 `dict` 不同，`OrderedDict` 中的 key 是按照插入顺序排序的：

```py
from collections import OrderedDict

od = OrderedDict()
od['z'] = 1
od['y'] = 2
od['x'] = 3

print od.keys() # 按照插入的Key的顺序返回
# ['z', 'y', 'x']
```

#### Counter

Counter 是一个简单的计数器：

```py
from collections import Counter

c = Counter()
for ch in 'programming':
    c[ch] = c[ch] + 1

print c
# Counter({'g': 2, 'm': 2, 'r': 2, 'a': 1, 'i': 1, 'o': 1, 'n': 1, 'p': 1})
```

### base64

Base64 是一种编码格式，他把二进制转换为几个标准的字符。二进制数据经过 base64 编码后可以放到 url 里面。

```py
import base64

# 编码
print base64.b64encode('binary\x00string')
# 'YmluYXJ5AHN0cmluZw=='

# 解码
print base64.b64decode('YmluYXJ5AHN0cmluZw==')
# 'binary\x00string'
```

注意 `YmluYXJ5AHN0cmluZw==` 这样的字符串，用在 url 里的时候可能会跟 url 本身的 `=` 符号冲突，所以一般会把它去掉。因为 base64 字符串的长度一定是 4 的倍数，可以根据这一点，在解码的时候吧去掉的 `=` 补上。

### struct

这个模块用于 字节 与 整形、浮点型 等类型的转换。感觉用到的地方不多，需要的话请参考 [原文](http://www.liaoxuefeng.com/wiki/001374738125095c955c1e6d8bb493182103fac9270762a000/0013994173393204e80af1f8fa94c8e9d094d229395ea43000)

### hashlib

`hashlib` 模块提供了常用的摘要算法，如 MD5, SHA1 等。

摘要算法又称哈希算法、散列算法。它通过一个函数，把任意长度的数据转换为一个长度固定的数据串（通常用16进制的字符串表示）。

```py
import hashlib

md5 = hashlib.md5()
md5.update('how to use md5 in ')
md5.update('python hashlib?')    # 可以分批加入要 md5 的字符串
print md5.hexdigest()
# d26a53750bc40b38b65a520292f69306
```

可以看到 hashlib 中的 md5 算法支持分批计算，这样就避免了在计算大文件 md5 时可能造成的内存占用高问题。

sha1 算法类似：

```py
import hashlib

sha1 = hashlib.sha1()
sha1.update('how to use sha1 in ')
sha1.update('python hashlib?')
print sha1.hexdigest()
```

### itertools

用于操作迭代对象，感觉用的不多，需要的话请参考 [原文](http://www.liaoxuefeng.com/wiki/001374738125095c955c1e6d8bb493182103fac9270762a000/001415616001996f6b32d80b6454caca3d33c965a07611f000)

### XML

操作 XML 有两种方法 DOM 和 SAX, DOM 会一次性载入 XML 文件并将它解析为树结构，而 SAX 则是边解析边抛事件出来。

SAX 方法比较节省内存，我们介绍一下这种方法：

```py
from xml.parsers.expat import ParserCreate

class DefaultSaxHandler(object):
    def start_element(self, name, attrs):
        print('sax:start_element: %s, attrs: %s' % (name, str(attrs)))

    def end_element(self, name):
        print('sax:end_element: %s' % name)

    def char_data(self, text):
        print('sax:char_data: %s' % text)

xml = r'''<?xml version="1.0"?>
<ol>
    <li><a href="/python">Python</a></li>
    <li><a href="/ruby">Ruby</a></li>
</ol>
'''

# 创建 parser
parser = ParserCreate()

# 设置为 returns_unicode, 这样返回的所有节点名和数据都是 unicode 格式
parser.returns_unicode = True

# 创建 handler, 用于处理 SAX 解析过程中的事件
handler = DefaultSaxHandler()

# 设置解析过程中的各个事件处理函数
parser.StartElementHandler = handler.start_element
parser.EndElementHandler = handler.end_element
parser.CharacterDataHandler = handler.char_data

# 开始解析
parser.Parse(xml)
```

至于生成 xml 的方法，文中只介绍了最粗暴的字符串拼接法：

```py
L = []
L.append(r'<?xml version="1.0"?>')
L.append(r'<root>')
L.append(encode('some & data'))
L.append(r'</root>')

return ''.join(L)
```

### HTMLParser

由于 HTML 网页的格式没有 xml 那么严格，所以不能用 XML 模块来解析 HTML 文件。好在 python 提供了 HTMLParser 来方便地解析 HTML:

```py
from HTMLParser import HTMLParser
from htmlentitydefs import name2codepoint

# 定义 MyHTMLParser 类，继承自 HTMLParser
class MyHTMLParser(HTMLParser):

    # 标签头
    def handle_starttag(self, tag, attrs):
        print('<%s>' % tag)

    # 标签尾
    def handle_endtag(self, tag):
        print('</%s>' % tag)

    # 形如 <x/> 的标签
    def handle_startendtag(self, tag, attrs):
        print('<%s/>' % tag)

    # 标签中的数据
    def handle_data(self, data):
        print('data')

    # 注释
    def handle_comment(self, data):
        print('<!-- -->')

    # 处理一些特殊字符，以&开头的
    def handle_entityref(self, name):
        print('&%s;' % name)

    # 处理特殊字符串，就是以&#开头的，一般是内码表示的字符
    def handle_charref(self, name):
        print('&#%s;' % name)

parser = MyHTMLParser()

# 开始解析
parser.feed('<html><head></head><body><p>Some <a href=\"#\">html</a> tutorial...<br>END</p></body></html>')
```


## 常用第三方模块

原文只介绍了 PIL 这一个第三方库，现在还没有用到，需要的话可以参考 [原文](http://www.liaoxuefeng.com/wiki/001374738125095c955c1e6d8bb493182103fac9270762a000/00140767171357714f87a053a824ffd811d98a83b58ec13000)


## 图形界面

python 支持多种图形界面的第三方库，包括：
- Tk
- wxWidgets
- Qt
- GTK

python 自带的是 Tk 的 Tkinter, 无需任何安装包即可使用，本章介绍使用 Tkinter 来进行 GUI 编程。

下面的例子展示了使用 Tkinter 编写一个 GUI 小程序的例子：

```py
# -*- coding: UTF-8 -*-

from Tkinter import *

# 从 Frame 派生一个 Application 类，作为所有 widgets 的容器
class Application(Frame):

    def __init__(self, master=None):
        Frame.__init__(self, master)

        # pack 将会把这个 widget 布局到界面中，现在 Application 是这个 widget 树的根节点
        self.pack()
        self.createWidgets()
    
    def createWidgets(self):
        # label 是一个 widget, 我们先创建一个 label, 然后使用 pack 将它布局到界面中
        self.helloLabel = Label(self, text='Hello, world!')
        self.helloLabel.pack()

        self.quitButton = Button(self, text='Quit', command=self.quit)
        self.quitButton.pack()

# 创建 Application 实例
app = Application()

# 设置标题
app.master.title('Hello, world')

# 启动消息泵
app.mainloop()
```

上述内容对 Tkinter GUI 编程做了简单的介绍，更多细节请参考相关文章。


## 网络编程

### TCP 编程

#### 客户端

如下代码创建一个基于 TCP 连接的 socket:

```py
# 导入socket库:
import socket

# 创建一个socket:
s = socket.socket(socket.AF_INET, socket.SOCK_STREAM)

# 建立连接:
s.connect(('www.sina.com.cn', 80))

# 发送数据:
s.send('GET / HTTP/1.1\r\nHost: www.sina.com.cn\r\nConnection: close\r\n\r\n')

# 接收数据:
buffer = []
while True:
    # 每次最多接收1k字节:
    d = s.recv(1024)
    if d:
        buffer.append(d)
    else:
        break
data = ''.join(buffer)

# 关闭连接:
s.close()

# 获取 HTTP 头和返回的 html 页面
header, html = data.split('\r\n\r\n', 1)
print header

# 把接收的数据写入文件:
with open('sina.html', 'wb') as f:
    f.write(html)
```

上述代码中的 `AF_INET` 指定使用 ipv4 协议，如果要使用 ipv6, 请填写为 `AF_INET6`, `SOCK_STREAM` 指定使用面向流的 TCP 协议。

`s.connect(('www.sina.com.cn', 80))` 这段代码表示将 socket 连接到 `www.sina.com.cn` 的 80 端口。

Tips: 端口号小于 1024 的是 Internet 标准服务的端口，端口号大于 1024 的则是自定义端口。

#### 服务器

服务端的程序比较麻烦一些，因为服务端需要接收并相应多个客户端的请求，所以每一个连接都需要一个新的进程或者新的线程来处理。

下面是一个简单的服务器程序的例子，它接受客户端连接，把客户端发过来的字符串加上 `Hello` 再发回去。

```py
# 导入socket库:
import socket

# 创建一个socket:
s = socket.socket(socket.AF_INET, socket.SOCK_STREAM)

# 绑定端口:
s.bind(('127.0.0.1', 9999))

# 监听端口
s.listen(5)
print 'Waiting for connection...'

while True:
    # 接受一个新连接:
    sock, addr = s.accept()
    # 创建新线程来处理TCP连接:
    t = threading.Thread(target=tcplink, args=(sock, addr))
    t.start()

def tcplink(sock, addr):
    print 'Accept new connection from %s:%s...' % addr
    sock.send('Welcome!')
    while True:
        # 接受来自客户端的数据
        data = sock.recv(1024)
        time.sleep(1)
        if data == 'exit' or not data:
            break
        # 向客户端发送数据
        sock.send('Hello, %s!' % data)
    sock.close()
    print 'Connection from %s:%s closed.' % addr
```

### UDP 编程

使用 UDP 编程的方法很简单，这里就不介绍了，具体可以参考 [原文](http://www.liaoxuefeng.com/wiki/001374738125095c955c1e6d8bb493182103fac9270762a000/0013868325264457324691c860044c5916ce11b305cb814000)


## 电子邮件

这部分内容不太常用，这里就不介绍了，有需要的话可以参考 [原文](http://www.liaoxuefeng.com/wiki/001374738125095c955c1e6d8bb493182103fac9270762a000/0013868325601402299d1e941914a21990ac7861ef4bc2d000)


## 访问数据库

### 使用 SQLite

SQLite是一种嵌入式数据库，它的数据库就是一个文件。由于SQLite本身是C写的，而且体积很小，所以，经常被集成到各种应用程序中，甚至在iOS和Android的App中都可以集成。

Python就内置了SQLite3，所以，在Python中使用SQLite，不需要安装任何东西，直接使用。

在使用SQLite前，我们先要搞清楚几个概念：
- 表是数据库中存放关系数据的集合，一个数据库里面通常都包含多个表，比如学生的表，班级的表，学校的表，等等。表和表之间通过外键关联。
- 要操作关系数据库，首先需要连接到数据库，一个数据库连接称为Connection；
- 连接到数据库后，需要打开游标，称之为Cursor，通过Cursor执行SQL语句，然后，获得执行结果。
- Python定义了一套操作数据库的API接口，任何数据库要连接到Python，只需要提供符合Python标准的数据库驱动即可。
- 由于SQLite的驱动内置在Python标准库中，所以我们可以直接来操作SQLite数据库。

下面是在 python 中创建 SQLite 数据库的一个例子：

```py
# 导入SQLite驱动:
import sqlite3

# 连接到SQLite数据库
# 数据库文件是test.db
# 如果文件不存在，会自动在当前目录创建:
conn = sqlite3.connect('test.db')

# 创建一个Cursor:
cursor = conn.cursor()

# 执行一条SQL语句，创建user表:
cursor.execute('create table user (id varchar(20) primary key, name varchar(20))')

# 继续执行一条SQL语句，插入一条记录:
cursor.execute('insert into user (id, name) values (\'1\', \'Michael\')')

# 通过rowcount获得插入的行数:
print cursor.rowcount
# 1

# 关闭Cursor:
cursor.close()

# 提交事务:
conn.commit()

# 关闭Connection:
conn.close()
```

下面是在 python 中查询 sqlite 数据库的例子：

```py
conn = sqlite3.connect('test.db')
cursor = conn.cursor()

# 执行查询语句:
cursor.execute('select * from user where id=?', ('1',))

# 获得查询结果集:
values = cursor.fetchall()
print values
# [(u'1', u'Michael')]

cursor.close()
conn.close()
```

使用Python的DB-API时，只要搞清楚Connection和Cursor对象，打开后一定记得关闭，就可以放心地使用:
- 使用Cursor对象执行insert，update，delete语句时，执行结果由rowcount返回影响的行数，就可以拿到执行结果。
- 使用Cursor对象执行select语句时，通过featchall()可以拿到结果集。结果集是一个list，每个元素都是一个tuple，对应一行记录。
- 如果SQL语句带有参数，那么需要把参数按照位置传递给execute()方法，有几个?占位符就必须对应几个参数，例如：
    `cursor.execute('select * from user where name=? and pwd=?', ('abc', '123456'))`

SQLite 支持常见的标准 SQL 语句以及几种常见的数据类型。具体文档请参阅 SQLite 官方网站。

### 使用 MySQL

MySQL是Web世界中使用最广泛的数据库服务器。SQLite的特点是轻量级、可嵌入，但不能承受高并发访问，适合桌面和移动应用。而MySQL是为服务器端设计的数据库，能承受高并发访问，同时占用的内存也远远大于SQLite。

此外，MySQL内部有多种数据库引擎，最常用的引擎是支持数据库事务的InnoDB。

由于 MySQL 服务器以独立的进程运行，并通过网络对外服务，所以需要支持 Python 的 MySQL 驱动来连接到 MySQL 服务器。目前可选的驱动有以下两个：
- mysql-connector-python：是MySQL官方的纯Python驱动；
- MySQL-python：是封装了MySQL C驱动的Python驱动。

安装驱动的方法和安装第三方模块一样，这里就不赘述了。

下面使用 `mysql-connector-python` 驱动作为例子，演示如何使用 MySQL.

```py
# 导入MySQL驱动:
import mysql.connector

# 注意把password设为你的root口令:
conn = mysql.connector.connect(user='root', password='password', database='test', use_unicode=True)

cursor = conn.cursor()

# 创建user表:
cursor.execute('create table user (id varchar(20) primary key, name varchar(20))')

# 插入一行记录，注意MySQL的占位符是%s:
cursor.execute('insert into user (id, name) values (%s, %s)', ['1', 'Michael'])
print cursor.rowcount
# 1

# 提交事务:
conn.commit()
cursor.close()

# 运行查询:
cursor = conn.cursor()
cursor.execute('select * from user where id = %s', ('1',))
values = cursor.fetchall()
print values
# [(u'1', u'Michael')]

# 关闭Cursor和Connection:
cursor.close()
conn.close()
```

可以看到，由于Python的DB-API定义都是通用的，所以，操作MySQL的数据库代码和SQLite类似。

Tips:
- MySQL的SQL占位符是%s；
- 通常我们在连接MySQL时传入use_unicode=True，让MySQL的DB-API始终返回Unicode。

### 使用 SQLAlchemy

python 的 DB-API 返回的查询结果是一个 tuple 列表：

```py
[
    ('1', 'Michael'),
    ('2', 'Bob'),
    ('3', 'Adam')
]
```

这样很难看出表的结构，如果能够把这个结构转换为一个类对象，也就是说把关系数据库的表结构映射到对象上，这样就能很好理解了。

这种技术叫做 ORM(Object-Relational Mapping) 技术，进行这种转换需要 ORM 框架，在 python 中，最有名的 ORM 框架是 SQLAlchemy. 这节我们介绍 SQLAlchemy 的用法。

首先通过 `easy_install` 或者 `pip` 安装 SQLAlchemy:

```s
$ easy_install sqlalchemy
```

接下来我们看看如何在代码中使用它，将表结构映射到对象上：

```py
# 导入:
from sqlalchemy import Column, String, create_engine
from sqlalchemy.orm import sessionmaker
from sqlalchemy.ext.declarative import declarative_base

# 创建对象的基类:
Base = declarative_base()

# 定义User对象:
class User(Base):
    # 表的名字:
    __tablename__ = 'user'

    # 表的结构:
    id = Column(String(20), primary_key=True)
    name = Column(String(20))

# 初始化数据库连接，连接信息为：'数据库类型+数据库驱动名称://用户名:口令@机器地址:端口号/数据库名'
engine = create_engine('mysql+mysqlconnector://root:password@localhost:3306/test')
# 创建DBSession类型:
DBSession = sessionmaker(bind=engine)

### 以下代码展示如何向表中插入数据

# 创建session对象:
session = DBSession()

# 创建新User对象:
new_user = User(id='5', name='Bob')

# 添加到session:
session.add(new_user)

# 提交即保存到数据库:
session.commit()

# 关闭session:
session.close()

### 以下代码展示如何从表中查询数据，查询的结果是一个 object

# 创建Session:
session = DBSession()

# 创建Query查询，filter是where条件，最后调用one()返回唯一行，如果调用all()则返回所有行:
user = session.query(User).filter(User.id=='5').one()

# 打印类型和对象的name属性:
print 'type:', type(user)
print 'name:', user.name

# 关闭Session:
session.close()
```

从上述代码中可以看到 ORM 的作用：它把表结构与类对象映射起来，能够更加直白简单地表示对表的操作。


## Web 开发

最早的软件都是运行在大型机上的，软间使用者通过 “亚终端” 登录到大型机上运行。后来随着 PC 机的兴起，软件开始主要运行在桌面上，而数据库这样的软件运行在服务器端，这种 Client/Server 模式简称 CS 架构。

随着互联网的兴起，人们发现， CS 架构不适合 Web ，最大的原因是 Web 应用程序的修改和升级非常迅速，而 CS 架构需要每个客户端逐个升级桌面 App ，因此, Browser/Server 模式开始流行，简称 BS 架构。

在 BS 架构下，客户端只需要浏览器，应用程序的逻辑和数据都存储在服务器端。浏览器只需要请求服务器，获取 Web 页面，并把 Web 页面展示给用户即可。

当然， Web 页面也具有极强的交互性。由于 Web 页面是用 HTML 编写的，而 HTML 具备超强的表现力，并且，服务器端升级后，客户端无需任何部署就可以使用到新的版本，因此， BS 架构迅速流行起来。

用 python 开发 web, 也就是在服务器端动态生成 HTML.

### HTTP 协议简介

HTTP 请求的流程：
- 浏览器向服务器发送 HTTP 请求，请求包括：
    - 方法: GET 或者 POST, GET 仅仅请求资源，POST 会附带用户数据；
    - 路径: /full/url/path;
    - 域名: 由 host 头指定，例如: host:www.sina.com.cn
    - 其他 Header;
    - 如果是 POST 方法，还包括一个 body, 包含了用户数据;
- 服务器向浏览器返回 HTTP 响应，响应包括：
    - 状态码: 200 表示成功, 3xx 表示重定向, 4xx 表示客户端发送的请求有错误, 5xx 表示服务器端处理的时候发生了错误；
    - 响应类型: 由 Content-Type 指定；
    - 其他 Header;
    - 通常会有一个 body, 包含了响应的内容, HTML 源码就包含在 body 中;
- 如果浏览器还需要继续向服务器请求其它资源，比如图片，就再次发出 HTTP 请求，重复步骤 1,2;

Web 采用的 HTTP 协议采用了非常简单的请求-响应模式，从而大大简化了开发。当我们编写一个页面时，我们只需要在 HTTP 请求中把 HTML 发送出去，不需要考虑如何附带图片、视频等，浏览器如果需要请求图片和视频，它会发送另一个 HTTP 请求，因此，一个 HTTP 请求只处理一个资源。

每个 HTTP 请求和响应都遵循相同的格式，一个 HTTP 包含 Header 和 Body 两个部分，其中 Body 是可选项；

HTTP 协议是一种文本协议，所以它的格式也非常简单，下面是 HTTP GET 的请求格式：

```
GET /path HTTP/1.1
Header1: Value1
Header2: Value2
Header3: Value3
```

每个Header一行一个，换行符是 `\r\n`.

HTTP POST 的请求格式：

```
POST /path HTTP/1.1
Header1: Value1
Header2: Value2
Header3: Value3

body data goes here...
```

当遇到连续两个`\r\n`时, Header 部分结束，后面的数据全部是 Body.

HTTP 响应的格式：

```
200 OK
Header1: Value1
Header2: Value2
Header3: Value3

body data goes here...
```

HTTP响应如果包含body，也是通过\r\n\r\n来分隔的。请再次注意，Body的数据类型由Content-Type头来确定，如果是网页，Body就是文本，如果是图片，Body就是图片的二进制数据。

当存在Content-Encoding时，Body数据是被压缩的，最常见的压缩方式是gzip，所以，看到Content-Encoding: gzip时，需要将Body数据先解压缩，才能得到真正的数据。压缩的目的在于减少Body的大小，加快网络传输。

要详细了解HTTP协议，推荐 “HTTP: The Definitive Guide” 一书，非常不错，有中文译本：[HTTP权威指南](https://www.amazon.cn/dp/B008XFDQ14)

### WSGI 接口

了解了HTTP协议和HTML文档，我们其实就明白了一个Web应用的本质就是：
- 浏览器发送一个 HTTP 请求；
- 服务器收到请求，生成一个 HTML 文档；
- 服务器把 HTML 文档作为 HTTP 响应的 Body 发送给浏览器；
- 浏览器收到 HTTP 响应，从 HTTP Body 取出 HTML 文档并显示。

所以，最简单的Web应用就是先把 HTML 用文件保存好，用一个现成的 HTTP 服务器软件，接收用户请求，从文件中读取 HTML, 返回。 Apache,Nginx,Lighttpd 等这些常见的静态服务器就是干这件事情的。

如果要动态生成 HTML, 就需要把上述步骤自己来实现。不过，接受 HTTP 请求、解析 HTTP 请求、发送 HTTP 响应都是苦力活，如果我们自己来写这些底层代码，还没开始写动态 HTML 呢，就得花个把月去读 HTTP 规范。

正确的做法是底层代码由专门的服务器软件实现，我们用 Python 专注于生成 HTML 文档。因为我们不希望接触到 TCP 连接, HTTP 原始请求和响应格式，所以，需要一个统一的接口，让我们专心用 Python 编写 Web 业务。

这个接口就是 WSGI：Web Server Gateway Interface. WSGI 接口定义非常简单，它只要求 Web 开发者实现一个函数，就可以响应 HTTP 请求。我们来看一个最简单的 Web 版本的 "Hello, web!":

```py
def application(environ, start_response):
    # 组装 header 部分
    start_response('200 OK', [('Content-Type', 'text/html')])
    # 组装 body 部分
    return '<h1>Hello, web!</h1>'
```

上面的 `application` 函数就是符合 WSGI 标准的一个 HTTP 处理函数，它接收两个参数：
- environ : 一个包含 HTTP 请求信息的 dict 对象；
- start_response: 一个发送 HTTP 响应的函数；

整个 `application()` 函数本身没有涉及到任何解析 HTTP 的部分，也就是说，底层代码不需要我们自己编写，我们只负责在更高层次上考虑如何响应请求就可以了。

为了让 WSGI 调用上述函数，我们首先得运行 WSGI 服务. python 内置了一个 WSGI 服务器，这个模块叫做 wsgiref, 它是用纯 python 编写的 WSGI 服务，它完全符合 WSGI 标准，但是不考虑任何运行效率，仅供开发和测试使用。

下面代码展示了如何运行 python 内置的 WSGI 服务：

```py
# server.py

# 从 wsgiref 模块导入:
from wsgiref.simple_server import make_server
# 导入我们自己编写的application函数:
from hello import application

# 创建一个服务器， IP 地址为空，端口是 8000，处理函数是 application:
httpd = make_server('', 8000, application)
print "Serving HTTP on port 8000..."
# 开始监听 HTTP 请求:
httpd.serve_forever()
```

### 使用 Web 框架

有了 WSGI 我们就只需要编写 application 函数，让它根据不同的 url 返回不同的 HTML 文本就可以了。

然而这样还是很麻烦，因为你不得不在 application 函数里添加许多 `if...elif` 判断，这样才能对不同的 url 进行处理：

```py
def application(environ, start_response):
    method = environ['REQUEST_METHOD']
    path = environ['PATH_INFO']
    if method=='GET' and path=='/':
        return handle_home(environ, start_response)
    if method=='POST' and path='/signin':
        return handle_signin(environ, start_response)
    ...
```

这些一路写下去，代码会越来越长，越来越难以维护。如果能够有一个框架自动地把 url 和它对应的处理函数一一映射起来，代码就会清晰许多。

也正因为这个原因，诞生了 web 框架这个东西。由于用 python 开发一个 web 框架非常容易，因此这样的框架有许多，我们这里直接选取一个比较流行的 web 框架 -- `Flask` 来使用。

首先安装 Flask:

```s
$ pip install flask
```

接着就可以在 python 中使用 flask 了：

```py
from flask import Flask
from flask import request

# 创建一个 flask 对象
app = Flask(__name__)

# 处理首页，也就是 localhost:5000/
@app.route('/', methods=['GET', 'POST'])
def home():
    return '<h1>Home</h1>'

# 处理登录页，也就是 localhost:5000/signin
@app.route('/signin', methods=['GET'])
def signin_form():
    # 返回登录表单
    return '''<form action="/signin" method="post">
              <p><input name="username"></p>
              <p><input name="password" type="password"></p>
              <p><button type="submit">Sign In</button></p>
              </form>'''

# 处理登录表单，也就是 localhost:5000/sigin
@app.route('/signin', methods=['POST'])
def signin():
    # 需要从request对象读取表单内容：
    if request.form['username']=='admin' and request.form['password']=='password':
        return '<h3>Hello, admin!</h3>'
    return '<h3>Bad username or password.</h3>'

if __name__ == '__main__':
    # 运行 flask 对象，这将启动 WSGI 服务器
    app.run()
```

flask 默认监听的端口是 5000. 我们运行上述 python 代码后，就可以看到 flask 框架运行了起来：

```
C:\Users\dongyu\Desktop>python serve.py
 * Running on http://127.0.0.1:5000/ (Press CTRL+C to quit)
```

之后在浏览器中输入 `localhost:5000/` 就可以访问首页，输入 `localhost:5000/signin` 就可以显示登录页，在登录页上输入 admin, password 之后就可以提交表单，之后页面会显示一个 Hello, admin! 字符串。

从上述代码可以看出, Flask 通过 python 的装饰器在内部自动地把 url 和处理函数关联起来。无需我们调用 WSIG 那样写许多 `if...elif` 判断。

除了Flask，常见的Python Web框架还有：
- Django：全能型Web框架；
- web.py：一个小巧的Web框架；
- Bottle：和Flask类似的Web框架；
- Tornado：Facebook的开源异步Web框架。

### 使用模板

上面的代码中还有一个问题，我们在函数中直接返回了 HTML 页面，简单的页面这么处理当然没问题，但对于动辄几千上万行的复杂页面来说，太复杂了。

在 python 里用拼字符串的方式来生成 HTML 是不现实的，所以，模板技术出现了。

使用模板，我们需要预先准备一个HTML文档，这个HTML文档不是普通的HTML，而是嵌入了一些变量和指令，然后，根据我们传入的数据，替换后，得到最终的HTML，发送给用户：

![]({{site.url}}/asset/python-html-template.png)

这就是传说中的 MVC: Model-View-Controller 中文名“模型-视图-控制器”：
- 上面例子中的 `['name': 'Michael']` 就是 model, 它是数据的来源；
- 包含变量 &#123;&#123; name &#125;&#125; 的模板就是 view, view 负责显示逻辑，通过简单地替换一些变量, view 的最终输出就是 HTML;
- python 中处理 url 的代码就是 controller, controller 负责业务逻辑，比如检查用户名是否存在，取出用户信息等；

我们使用 MVC 模式来修改一下上一章中的例子：

```py
from flask import Flask, request, render_template

app = Flask(__name__)

@app.route('/', methods=['GET', 'POST'])
def home():
    # 渲染 home.html 模板，将得到一个 HTML
    return render_template('home.html')

@app.route('/signin', methods=['GET'])
def signin_form():
    # 渲染 form.html 模板，将得到一个 HTML
    return render_template('form.html')

@app.route('/signin', methods=['POST'])
def signin():
    # 逻辑部分，通过用户名和密码来判断需要渲染哪个模板
    username = request.form['username']
    password = request.form['password']
    if username=='admin' and password=='password':
        return render_template('signin-ok.html', username=username)
    return render_template('form.html', message='Bad username or password', username=username)

if __name__ == '__main__':
    app.run()
```

上述例子中的 `render_template()` 是 flask 的一个函数，它会完成模板的渲染操作。实际上模板的渲染有许多种, flask 默认支持的是 [jinja2](http://jinja.pocoo.org/), 所以我们需要先安装它，否则 flask 无法帮助我们完成模板渲染操作。

```s
$ pip install jinja2
```

安装好 jinja2 之后，我们就可以编写 HTML 模板了：

这是首页的模板：

```html
<html>
<head>
  <title>Home</title>
</head>
<body>
  <h1 style="font-style:italic">Home</h1>
</body>
</html>
```

这是用来显示登录表单的模板：

![]({{site.url}}/asset/python-jinja-code1.png)

这是登录成功的模板：

![]({{site.url}}/asset/python-jinja-code2.png)

随后，我们创建一个 `templates` 目录来存放这些 html 模板，`templates` 和 `app.py` 在同一目录下。

启动 `app.py` 就可以查看模板的效果了。

注意上述代码中，我们用 `form.html` 代表了登录失败的模板，如果登录失败，会给模板传一个 message 变量，这个时候就会显示出失败的原因。在 jinja2 中，使用 &#123;&#123; name &#125;&#125; 来表示一个需要替换的变量，使用 &#123;&#37; ... &#37;&#125; 来表示循环、条件判断等指令。例如循环输出页码：

![]({{site.url}}/asset/python-jinja-code3.png)

在 python 中传入 `page_list = [1, 2, 3, 4, 5]`, 那么上述代码将输出五个超链接。

除了 jinja2 之外，常见的模板还有：
- Mako：用 `<% ... %>` 和 `${xxx}` 的一个模板；
- Cheetah：也是用 `<% ... %>` 和 `${xxx}` 的一个模板；
- Django：Django 是一站式框架，内置一个用 &#123;&#37; ... &#37;&#125; 和 &#123;&#123; xxx &#125;&#125; 的模板。