---
layout: post
title:  "JavaScript 库 underscore"
date:   2017-04-05 18:36:30 +0800
categories: javascript
---

* TOC
{:toc}

在之前的文章中我们已经介绍过，JavaScript 支持函数式编程，支持高阶函数和闭包。函数式编程非常强大，可以写出非常简洁的代码。例如 `Array` 的 `map()` 和 `filter()` 方法：

```js
'use strict';

var a1 = [1, 4, 9, 16];
var a2 = a1.map(Math.sqrt); // [1, 2, 3, 4]
var a3 = a2.filter((x) => { return x % 2 === 0; }); // [2, 4]
```

问题是只有 `Array` 有这些方法，Object 就没有这些方法。underscore 库的作用就是提供这样一套完善的函数式编程接口，让我们更方便地在 JavaScript 中实现函数式编程。

jQuery 在加载的时候，会把自身绑定到 `$` 这个全局变量上，underscore 与此类似，它会把自身绑定到 `_` 上，这也是它叫做 underscore 的原因。

用 underscore 实现 `map()` 操作如下：

```js
'use strict';
_.map([1, 2, 3], (x) => x * x); // [1, 4, 9]
```

乍一看比 `Array` 的 `map()` 方法更麻烦，但它可以作用于 Object:

```js
'use strict';
_.map({ a: 1, b: 2, c: 3 }, (v, k) => k + '=' + v); // ['a=1', 'b=2', 'c=3']
```

接下来我们介绍 underscore 的一系列功能。

## Collections

unserscore 为集合类对象提供了一系列接口，这里的集合类指 Array 和 Object, 暂不支持 Map 和 Set

### map/filter

当 underscore 的 `map/filter` 作用于 Object 时，传入的函数为 `function(value, key)`。这个函数会作用到 Object 的每个 key,value 上。

`_.map()` 函数返回一个数组，数组的每个元素都是由传入的函数计算得来。

### every/some

当集合的所有元素都满足条件时，`_.every()` 返回 true, 当集合的至少一个元素满足条件时，`_.some()` 返回 true:

```js
'use strict';
// 所有元素都大于0？
_.every([1, 4, 7, -3, -9], (x) => x > 0); // false
// 至少一个元素大于0？
_.some([1, 4, 7, -3, -9], (x) => x > 0); // true
```

它同样可以作用于 Object 上：

```js
'use strict';
var obj = {
    name: 'bob',
    school: 'No.1 middle school',
    address: 'xueyuan road'
};

// 判断key和value是否全部是小写：
var r1 = _.every(obj, function (value, key) {
    return _.every(key+value, (x) => x >= 'a' && x <= 'z');
});

var r2 = _.some(obj, function (value, key) {
    return _.every(key+value, (x) => x >= 'a' && x <= 'z');
});
```

### max/min

返回集合中最大和最小的数： 

```js
'use strict';
var arr = [3, 5, 7, 9];
_.max(arr); // 9
_.min(arr); // 3

// 空集合会返回-Infinity和Infinity，所以要先判断集合不为空：
_.max([]) // -Infinity
_.min([]) // Infinity

// 如果是 Object, 则只会比较 value
_.max({ a: 1, b: 2, c: 3 }); // 3
```

### groupBy

把集合的元素按照 key 归类：

```js
'use strict';

var scores = [20, 81, 75, 40, 91, 59, 77, 66, 72, 88, 99];
var groups = _.groupBy(scores, function (x) {
    if (x < 60) {
        return 'C';
    } else if (x < 80) {
        return 'B';
    } else {
        return 'A';
    }
});
// 结果:
// {
//   A: [81, 91, 88, 99],
//   B: [75, 77, 66, 72],
//   C: [20, 40, 59]
// }
```

### shuffle/sample

`shuffle()` 用洗牌算法随机打乱一个集合：

```js
'use strict';
// 注意每次结果都不一样：
_.shuffle([1, 2, 3, 4, 5, 6]); // [3, 5, 4, 6, 2, 1]
```

`sample()` 随机选择一个或多个元素：

```js
'use strict';
// 注意每次结果都不一样：
// 随机选1个：
_.sample([1, 2, 3, 4, 5, 6]); // 2
// 随机选3个：
_.sample([1, 2, 3, 4, 5, 6], 3); // [6, 1, 4]
```


## Array

underscore 为 `Array` 提供了许多工具类方法，可以更方便地操作 `Array`：

### first/last

顾名思义，取 Array 的第一个和最后一个元素。

```js
'use strict';
var arr = [2, 4, 6, 8];
_.first(arr); // 2
_.last(arr); // 8
```

### flatten

把多维数组变成一维数组：

```js
'use strict';

_.flatten([1, [2], [3, [[4], [5]]]]); // [1, 2, 3, 4, 5]
```

### zip/unzip

`zip()` 接受两个或多个数组，它会把两个数组中的元素按照索引对其，然后按照索引合并成新数组，比如你有一个 `Array` 保存了名字，另外一个数组保存了名字对应的分数, `zip()` 可以让你很轻松地把名字和分数对齐：

```js
'use strict';

var names = ['Adam', 'Lisa', 'Bart'];
var scores = [85, 92, 59];
_.zip(names, scores);
// [['Adam', 85], ['Lisa', 92], ['Bart', 59]]
```

unzip 则是相反操作：

```js
'use strict';
var namesAndScores = [['Adam', 85], ['Lisa', 92], ['Bart', 59]];
_.unzip(namesAndScores);
// [['Adam', 'Lisa', 'Bart'], [85, 92, 59]]
```

### object

`object()` 函数接受两个数组，它按照类似 `zip()` 的规则，将两个数组合并成一个 object:

```js
'use strict';

var names = ['Adam', 'Lisa', 'Bart'];
var scores = [85, 92, 59];
_.object(names, scores);
// {Adam: 85, Lisa: 92, Bart: 59}
```

### range

`range()` 用于快速生成序列：

```js
'use strict';

// 从0开始小于10:
_.range(10); // [0, 1, 2, 3, 4, 5, 6, 7, 8, 9]

// 从1开始小于11：
_.range(1, 11); // [1, 2, 3, 4, 5, 6, 7, 8, 9, 10]

// 从0开始小于30，步长5:
_.range(0, 30, 5); // [0, 5, 10, 15, 20, 25]

// 从0开始大于-10，步长-1:
_.range(0, -10, -1); // [0, -1, -2, -3, -4, -5, -6, -7, -8, -9]
```


## Functions

unserscore 中提供了大量 JavaScript 本身没有的高阶函数：

### bind

首先我们看一个错误用法：

```js
'use strict';

console.log('Hello, world!');
// 输出'Hello, world!'

var log = console.log;
log('Hello, world!');
// Uncaught TypeError: Illegal invocation
```

你想把 console.log 函数赋值给 log, 以后直接用 `log()` 来打印，像上面的代码是不行的，因为直接调用 `log()` 传入的 `this` 指针是 `undefined` 而不是 `console`。

```js
'use strict';

var log = _.bind(console.log, console);
log('Hello, world!');
// 输出Hello, world!
```

上述代码把 `console` 对象绑定在 `log()` 的 this 指针上，就可以直接调用 log 了。

### partial

`partial()` 的作用是为一个函数创建一个偏函数。

所谓偏函数可以看做是设定了默认参数的函数。比如我们已经有一个 `Math.pow(x, y)` 可以计算 x 的 y 次方，我们可以为他创建一个偏函数 `cube(x)` 默认的幂为 3, 专门计算立方：

```js
'use strict';

var cube = _.partial(Math.pow, _, 3);
cube(3); // 27
cube(5); // 125
cube(10); // 1000
```

### once

`once()` 返回一个函数，该函数无论调用多少次，函数体都只执行一次：

```js
'use strict'

var register = _.once(function () {
    alert('Register ok!');
});

// 只会执行第一次
register()
register()
register()
```

### delay

`delay()` 让一个函数延迟执行，效果和 `setTimeout()` 一样，但简单很多：

```js
'use strict';

var log = _.bind(console.log, console);
_.delay(log, 2000, 'Hello,', 'world!');
// 2秒后打印'Hello, world!':
```


## Objects

unserscore 也提供了大量针对于 Object 的函数：

### keys/allKeys

`keys()` 返回一个 Object 的所有 key, 不包括从原型链继承下来的：

```js
'use strict';

function Student(name, age) {
    this.name = name;
    this.age = age;
}

var xiaoming = new Student('小明', 20);
_.keys(xiaoming); // ['name', 'age']
```

`allKeys()` 除了自身的 key, 还有原型链继承下来的 key:

```js
'use strict';

function Student(name, age) {
    this.name = name;
    this.age = age;
}
Student.prototype.school = 'No.1 Middle School';
var xiaoming = new Student('小明', 20);
_.allKeys(xiaoming); // ['name', 'age', 'school']
```

### values

`values()` 返回自身所有的 value:

```js
'use strict';

var obj = {
    name: '小明',
    age: 20
};

_.values(obj); // ['小明', 20]
```

### mapObject

`mapObject()` 就是针对 Object 的 `map()`:

```js
'use strict';

var obj = { a: 1, b: 2, c: 3 };
// 注意传入的函数签名，value在前，key在后:
_.mapObject(obj, (v, k) => 100 + v); // { a: 101, b: 102, c: 103 }
```

### invert

`invert()` 把 Object 的 key 和 value 进行交换：

```js
'use strict';

var obj = {
    Adam: 90,
    Lisa: 85,
    Bart: 59
};
_.invert(obj); // { '59': 'Bart', '85': 'Lisa', '90': 'Adam' }
```

### extend/extendOwn

`extend()` 把多个 object 的 key-value 合并到第一个 object 并返回：

```js
'use strict';

var a = {name: 'Bob', age: 20};
_.extend(a, {age: 15}, {age: 88, city: 'Beijing'}); // {name: 'Bob', age: 88, city: 'Beijing'}
// 变量a的内容也改变了：
a; // {name: 'Bob', age: 88, city: 'Beijing'}
```

### clone

`clone()` 方法将复制一个 object：

```js
'use strict';
var source = {
    name: '小明',
    age: 20,
    skills: ['JavaScript', 'CSS', 'HTML']
};

var copied = _.clone(source);
```

### isEqual

`isEqual()` 对两个 object 进行深度比较，内容完全相同则返回 true：

```js
'use strict';

var o1 = { name: 'Bob', skills: { Java: 90, JavaScript: 99 }};
var o2 = { name: 'Bob', skills: { JavaScript: 99, Java: 90 }};

o1 === o2; // false
_.isEqual(o1, o2); // true
```


## Chaining

jQuery 支持链式调用，underscore 能够将一组操作转化为链式操作：

```js
_.filter(_.map([1, 4, 9, 16, 25], Math.sqrt), x => x % 2 === 1);
// [1, 3, 5]

_.chain([1, 4, 9, 16, 25])
 .map(Math.sqrt)
 .filter(x => x % 2 === 1)
 .value();
// [1, 3, 5]
```