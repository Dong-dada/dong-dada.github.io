---
layout: post
title:  "JavaScript 基础知识学习"
date:   2017-03-31 16:08:30 +0800
categories: lang javascript
---

* TOC
{:toc}

本文参考自 [廖雪峰的官方网站](http://www.liaoxuefeng.com/wiki/001434446689867b27157e896e74d51a89c25cc8b43bdb3000)

## JavaScript 基础知识

JavaScript 代码可以直接嵌在网页的任何地方，不过通常我们都把 JavaScript 代码放到 `<head>` 中：

```html
<html>
<head>
    <script>
        alert('Hello, world');
    </script>
</head>
<body>
  ...
</body>
</html>
```

第二种方法是把 JavaScript 代码放到一个单独的 `.js` 文件中，然后在 HTML 中通过 `<script src="..."></script>` 引入这个文件：

```html
<html>
<head>
    <script src="/static/js/abc.js"></script>
</head>
<body>
    ...
</body>
</html>
```

你可以在同一个页面中引入多个 `.js` 文件，还可以在页面中多次编写 `<script> js 代码 ... </script>`, 浏览器会按照顺序依次执行。

有时候你会看到 `<script>` 标签还设置了一个 `type` 属性：

```html
<script type="text/javascript">
    ...
</script>
```

但这是没有必要的，因为默认的 `type` 就是 JavaScript。

JavaScript 的语法和 Java 类似，每个语句都以 `;` 结尾，语句块使用大括号 `{...}`，但是 JavaScript 并不强制要求每个语句的结尾加 `;`，因为浏览器中的 JS 引擎会自动在每个语句的结尾加上 `;`，所以你见到没有以分号结尾的 js 代码，不要觉得奇怪。

### 注释

JavaScript 的注释与 Java 一样，可以使用 `//` 或 `/* ... */`。

### 数据类型

JavaScript 中的基本数据类型和其他语言没有什么两样：

#### Number

JS 中的数字不区分整数和浮点数，统一称为 Number。有两个例外：
- `NaN` 表示 `Not a Number` 无法计算结果时会用 NaN 表示；
- `Infinity` 表示无限大，如果数字超过了 JS 所能表示的最大值，就会用 `Infinity` 来表示；

Number 支持常见的四则运算：

```js
1 + 2;              // 3
(1 + 2) * 5 / 2;    // 7.5
2 / 0;              // Infinity
0 / 0;              // NaN
10 % 3;             // 1
10.5 % 3;           // 1.5
```

注意 JavaScript 中用 `%` 表示取余运算。

#### 字符串

JS 中的字符串和 lua, python 一样，可以用单引号或者双引号来表示：

```js
alert('abc');
alert("xyz");
```

JS 中的字符串也有转义：

```js
'hello \n world';   // \n,\t 这些都是有的
'\x41';             // 十六进制也是支持的
'\u4e2d';           // Unicode 字符也是支持的
```

JS 中表示多行字符串是通过反引号来做的：

```js
var text = `
<root>
    <item>
    </item>
</root>
`
```

字符串可以用 `+` 连接起来：

```js
var name = '董哒哒';
var age = 20;
var message = '你好, ' + name + ', 你今年' + age '岁了!';
alert(message);
```

ES6 还提供了叫做模板字符串的写法，相对使用 `+` 来说简单一些：

```js
var name = '小明';
var age = 20;
var message = `你好, ${name}, 你今年${age}岁了!`;
alert(message);
```

JS 中的字符串可以像 C/C++ 那样进行下标操作：

```js
var s = 'Hello, world!';

s[0]; // 'H'
s[6]; // ' '
s[7]; // 'w'
s[12]; // '!'
s[13]; // undefined 超出范围的索引不会报错，但一律返回undefined
```

值得注意的是使用下标来修改字符串不会达到效果，JS 中的字符串是不可变的：

```js
var s = 'Hello, world!';

s[0] = 'A'; // 不会起作用
```

JS 还提供了一些方法可以对字符串进行操作：

```js
var s = 'Hello';
s.length;       // 获得字符串长度
s.toUpperCase();// 转大写
s.toLowerCase();// 转小写
s.indexOf('e'); // 返回指定子串的位置，没找到返回 -1
s.substring(0, 3);  // 返回指定区间的子串
s.substring(3);     // 返回 3 到结尾的字符串，而不是前三个
```

#### 布尔值

JS 中布尔值只有 `true`, `false` 两种，它支持 与或非 这样的布尔运算：

```js
true && true;
false || true;
! false;
```

值得一提的是 JS 有两个比较运算符 `==` 和 `===`，这很奇葩。`==` 会对数据类型自动转换，使得 `false == 0`, 而 `===` 不会转换，一旦比较两个不同的数据类型，`===` 必然会返回 false.

实际使用中 **不要** 使用 `==`；

另外 `NaN` 这个玩意儿跟谁都不相等，包括它自己。。。 `NaN === NaN` 为 `false`，你只能通过 `isNaN()` 函数来判断一个值是不是 `NaN`，我知道这有点奇葩，不过好在一般不会遇到数字的值是 `NaN` 的情况，谁会写出 `0/0` 这样的代码呢？

与 C/C++ 一样，因为浮点运算有误差，你不能直接比较浮点运算的结果，：

```js
1/3 === (1-2/3); // false
```

#### null 和 undefined

`null` 表示一个空值，类似于 Python 中的 `None`, Lua 中的 `nil`。

`undefined` 表示 “未定义”，设计者希望区分 `null` 和 `undefined` 这两种不同的语义，实际上这个没啥卵用，多数情况下，都应该使用 `null`, 只有在判断函数参数是否传递的情况下才使用 `undefined`。

#### 数组

JS 中的数组用中括号表示，与 Python 类似：

```js
[1, 2, "Hello", null, true];
```

你可以用 `Array()` 来动态创建出数组：

```js
new Array(1, 2, 3);
```

不过一般建议用 `[...]` 的形式来表示，这样比较直观：

```js
var arr = [1, 2, "Hello", null, true];
arr[0];
arr[4];
arr[6];     // 超出范围，返回的是 undefined
```

可以看到 JS 的数组也支持下表访问，不过超出范围之后返回 `undefined` 这个让我有点蒙，这样区分 `null` 或者 `undefined` 感觉没啥意义。。。

JS 也提供了一系列操作数组的方法：

```js
var arr = [10, 20, '30', 'Hello', null, true];
arr.length;             // 返回数组长度
arr.length = 2;         // 奇葩的是你可以给 length 赋值来改变数组的长度
arr.[10] = 'x';         // 数组会变长到有 11 个元素，中间以 undefined 来填充
arr.indexOf(20);        // 返回元素所在的索引
arr.slice(0, 3);        // 返回一个数组切片：[10,20,'30']
arr.slice(3);           // 相当于 arr.slice(3, arr.length), 而不是 arr.slice(0, 3)
arr.push('A', 'A');     // 向数组尾端添加数据
arr.pop();              // 返回数组尾端的数据，并删掉它
arr.unshift('A', 'B');  // 向数组头部添加数据
arr.shift();            // 返回数组头部的数据，并删除它
arr.sort();             // 按照默认规则对数组进行排序
arr.reserve();          // 反转数组
arr.splice(2, 3, 'Google', 'FaceBook');     // 从索引 2 开始删除 3 个元素，然后插入两个新元素
arr.splice(2, 3);       // 从索引 2 开始删除 3 个元素，不插入
arr.splice(2, 0, 'Google', 'FaceBook');     // 只添加，不删除

var arr2 = [1, 2, 3];
var arr3 = arr.concat(arr2);    // 连接两个数组，返回一个新数组

arr.join('-');          // 把数组中所有的元素都用指定的符号连接起来，返回连接之后的字符串
```

如果数组的元素也是数组，那么就构成了多维数组：

```js
var arr = [[1, 2, 3], [400, 500, 600], '-'];
arr[1][1];      // 500
```

#### 对象

JS 的对象是由键值对组成的无序集合：

```js
var person = {
    name : 'bob',
    age : 20,
    tags : ['js', 'web', 'mobile'],
    city : 'beijing',
    hasCar : true,
    zipcode : null
};
```

注意对象的 key 必须是字符串类型，JS 也支持用键来访问：

```js
person.name;    // bob
person.zipcode; // null
person.afasdfasdf;  // undefined
```

如果键是一个不合法的变量名，需要使用字符串作为键来访问：

```js
xiaohong['middle-school']; // 'No.1 Middle School'
```

直接赋值可以新增一个键，使用 `delete` 操作符可以删除一个键：

```js
var xiaoming = {
    name: '小明'
};
xiaoming.age;           // undefined
xiaoming.age = 18;      // 新增一个age属性
xiaoming.age;           // 18
delete xiaoming.age;    // 删除age属性
xiaoming.age;           // undefined
delete xiaoming['name'];// 删除name属性
xiaoming.name;          // undefined
delete xiaoming.school; // 删除一个不存在的school属性也不会报错
```

`in` 操作符可以检查对象是否具有某个属性：

```js
var xiaoming = {
    name: '小明',
    birth: 1990,
    school: 'No.1 Middle School',
    height: 1.70,
    weight: 65,
    score: null
};
'name' in xiaoming; // true
'grade' in xiaoming; // false
```

JS 中的对象具有继承的特性，所以 `in` 操作符检查的结果可能是它的父对象的某个属性。如果你想检查某个对象本身是否具有某种属性，可以使用 `hasOwnProperty()` 方法：

```js
var xiaoming = {
    name: '小明'
};
xiaoming.hasOwnProperty('name'); // true
xiaoming.hasOwnProperty('toString'); // false
```

#### 变量

上面代码中已经展示了 `var` 语句，它的作用是声明一个变量。

值得一提的是 JS 中变量名里可以包含 `$` 这个符号，跟其他语言不太一样：

```js
var $b = 1;
var s$_007 = 10;
```

`var` 在 JS 里有点像 lua 的 `local`，也就是说在 JS 里你也可以不用 `var` 来声明变量，但这个变量的作用域是全局的：

```js
i = 10;     // i 是全局变量
```

值得注意的是：**JS 里全局变量的范围不只是当前 JS 文件，而是整个 HTML 页面！**, 也就是说如果同一页面的两个 JS 文件里都使用了上述例子中的 `i`, 那么会互相产生影响！

你可以在 JS 代码的第一行写上：

```js
`use strict`;
```

这会告诉浏览器开启 strict 模式，这样浏览器检测到不符合规范的代码后将会报运行错误，就比如上述例子中的全局变量。你应当始终打开 strict 模式来编写代码，避免造成意料之外的问题。

### 判断和循环

JS 中的判断与 C 完全一样：

```js
var age = 3;
if (age >= 18) {
    alert('adult');
} else if (age >= 6) {
    alert('teenager');
} else {
    alert('kid');
}
```

JS 支持与 C 完全一样的循环：

```js
var x = 0;
var i;
for (i=1; i<10; i++) {
    x = x + i;
}

var x = 0;
var n = 99;
while (n > 0){
    x = x + n;
    n = n - 2;
}

var n = 0;
do {
    n = n + 1；
} while (n < 100);
```

此外 JS 还支持 `for ... in` 循环，它可以作用于数组和对象：

```js
var o = {
    name: 'Jack',
    age: 20,
    city: 'Beijing'
};
for (var key in o) {
    alert(key); // 'name', 'age', 'city'
}

var a = ['A', 'B', 'C'];
for (var i in a) {
    alert(i); // '0', '1', '2'
    alert(a[i]); // 'A', 'B', 'C'
}
```

### Map 和 Set

JS 中的对象很像是其他语言中的 `map` 或者 `directory`，即键值对。但是 JS 对象的 key 必须是字符串，这个有时候不太方便。为了解决这个问题，ES6 规范引入了新的数据类型 `Map`:

```js
// 用二维数组来初始化一个 Map
var m = new Map([['Michael', 95], ['Bob', 75], ['Tracy', 85]]);
m.get('Michael'); // 95
```

Map 中包含了一下几个方法：

```js
var m = new Map();  // 空Map
m.set('Adam', 67);  // 添加新的key-value
m.set('Bob', 59);
m.has('Adam');      // 是否存在key 'Adam': true
m.get('Adam');      // 67
m.delete('Adam');   // 删除key 'Adam'
m.get('Adam');      // undefined
```

`Set` 和 `Map` 类似，只不过它不存储 value, set 的特点是没有重复的 key:

```js
var s1 = new Set(); // 空Set

// 用数组来初始化 set
var s2 = new Set([1, 2, 3]); // 含1, 2, 3

s2.add(4);  // 添加一个元素
s2.add(3);  // 重复的 key，添加不会有效果

s2.delete(3);   // 删除一个元素
``` 

### iterable

ES6 标准引入了新的 `iterable` 类型，`Array`, `Map`, `Set` 都属于 `iterable` 类型。它们都可以通过 `for ... of` 的方式来进行遍历：

```js
var a = ['A', 'B', 'C'];
for (var x of a) { // 遍历Array
    alert(x);
}

var s = new Set(['A', 'B', 'C']);
for (var x of s) { // 遍历Set
    alert(x);
}

var m = new Map([[1, 'x'], [2, 'y'], [3, 'z']]);
for (var x of m) { // 遍历Map
    alert(x[0] + '=' + x[1]);
}
```

看起来 `for ... in` 和 `for ... of` 没两样，实际上还是有个区别。`for ... in` 会把属性也给遍历进来：

```js
var a = ['A', 'B', 'C'];

// 你可以给一个数组动态添加属性
a.name = 'Hello';

// for in 循环遍历的时候会把属性也算进来
for (var x in a) {
    alert(x); // '0', '1', '2', 'name'
}
```

而 `for ... of` 没有这种副作用，它就是单纯地遍历元素。

此外, `iterable` 类型还支持 `forEach` 方法：

```js
var a = ['A', 'B', 'C'];
a.forEach(function (element, index, array) {
    // element: 指向当前元素的值
    // index: 指向当前索引
    // array: 指向Array对象本身
    alert(element);
});

var s = new Set(['A', 'B', 'C']);
s.forEach(function (element, sameElement, set) {
    // element 都是当前元素
    // set 是 set 对象本身
    alert(element);
});

var m = new Map([[1, 'x'], [2, 'y'], [3, 'z']]);
m.forEach(function (value, key, map) {
    alert(value);
});

// 你可以忽略不需要的参数
var a = ['A', 'B', 'C'];
a.forEach(function (element) {
    alert(element);
});
```
