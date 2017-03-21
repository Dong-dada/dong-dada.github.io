---
layout: post
title:  "Python 基础知识学习"
date:   2017-03-20 10:53:30 +0800
categories: lang python
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

