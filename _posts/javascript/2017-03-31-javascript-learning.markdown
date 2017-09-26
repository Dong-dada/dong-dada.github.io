---
layout: post
title:  "JavaScript 基础知识学习"
date:   2017-03-31 16:08:30 +0800
categories: javascript
---

 
 

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


## 函数

JS 中定义函数的语法如下：

```js
function abs(x) {
    if (x >= 0) {
        return x;
    } else {
        return -x;
    }
}
```

如果函数没有 return 语句，那么返回值是 `undefined` 而不是 `null`;

由于函数在 JS 中是第一类值，因此可以有如下写法：

```js
var abs = function (x) {
    if (x >= 0) {
        return x;
    } else {
        return -x;
    }
};
```

### 函数参数

JS 中的函数参数可以是可变参数，这时候可以通过 `arguments` 关键字来获取所有参数：

```js
function abs() {
    if (arguments.length === 0) {
        return 0;
    }
    var x = arguments[0];
    return x >= 0 ? x : -x;
}

abs(); // 0
abs(10); // 10
abs(-9); // 9
```

使用 `arguments` 关键字有一个比较麻烦的地方：

```js
function foo(a, b) {
    var i, rest = [];
    if (arguments.length > 2) {
        for (i = 2; i<arguments.length; i++) {
            rest.push(arguments[i]);
        }
    }
    console.log('a = ' + a);
    console.log('b = ' + b);
    console.log(rest);
}
```

上述例子中，`a`, `b` 这两个参数可以直接获取，但剩下的参数需要通过 `arguments` 来循环获取，并且还得把前两个排除。这种写法很别扭，为此 ES6 标准引入了 rest 参数，可以写为：

```js
function foo(a, b, ...rest) {
    console.log('a = ' + a);
    console.log('b = ' + b);
    console.log(rest);
}

foo(1, 2, 3, 4, 5);
// 结果:
// a = 1
// b = 2
// Array [ 3, 4, 5 ]
```

### 变量作用域

与其他语言一样，`var` 定义的局部变量，其作用域位于 `{ ... }` 内部。

JavaScript 函数定义有个特点，它会先扫描整个函数体的语句，把它声明的所有变量“提升”到函数顶部：

```js
'use strict';

function foo() {
    var x = 'Hello, ' + y;
    alert(x);
    var y = 'Bob';
}

foo();
```

虽然在 `var x = 'Hello, ' + y` 这一句中引用了未定义的变量 `y`, 但这句话并不会报错，因为 `y` 在之后定义过了，它相当于：

```js
function foo() {
    var y; // 提升变量y的申明
    var x = 'Hello, ' + y;
    alert(x);
    y = 'Bob';
}
```

注意，这时只是提升了 `y` 的声明，并没有提升它的赋值操作，因此 `y` 的值是 `undefined`.

由于 JS 的这个奇怪特性，我们在函数内部定义变量的时候，请严格遵守 “在函数内部首先申明所有变量” 这一原则：

```js
function foo() {
    var
        x = 1, // x初始化为1
        y = x + 1, // y初始化为2
        z, i; // z和i为undefined
    // 其他语句:
    for (i=0; i<100; i++) {
        ...
    }
}
```

### 全局作用域

不在任何函数内定义的变量就具有全局作用域。实际上，JS 默认有一个全局对象 `window`, 全局作用域的变量实际上被绑定到了 `window` 的一个属性上：

```js
'use strict';

var course = 'Learn JavaScript';
alert(course); // 'Learn JavaScript'
alert(window.course); // 'Learn JavaScript'
```

类似地，在顶层定义的函数也是一个全局变量，也被绑定到了 `window` 对象上：

```js
'use strict';

function foo() {
    alert('foo');
}

foo(); // 直接调用foo()
window.foo(); // 通过window.foo()调用
```

### 名字空间

全局变量会绑定到 `window` 上，同一页面不同的 JS 文件会共享同一个 window, 如果定义了相同名字的函数或者变量，就会造成名字冲突。为了解决这个问题，你可以把自己所有的变量和函数都绑定到一个全局变量上：

```js
// 唯一的全局变量MYAPP:
var MYAPP = {};

// 其他变量:
MYAPP.name = 'myapp';
MYAPP.version = 1.0;

// 其他函数:
MYAPP.foo = function () {
    return 'foo';
};
```

### 局部作用域

JS 中变量作用域在函数内部，因此在 `for` 语句中定义的变量可能会被外部引用到：

```js
'use strict';

function foo() {
    for (var i=0; i<100; i++) {
        //
    }
    i += 100; // 仍然可以引用变量i
}
```

为了解决块级作用域，ES6 引入了新的关键字 `let`, 用 `let` 替代 `var` 可以申明一个块级作用域的变量：

```js
'use strict';

function foo() {
    var sum = 0;
    for (let i=0; i<100; i++) {
        sum += i;
    }
    i += 1; // SyntaxError
}
```

### 常量

ES6 标准引入了新的关键字 `const` 来定义常量，`const` 和 `let` 都具有块级作用域。

```js
'use strict';

const PI = 3.14;
PI = 3; // 某些浏览器不报错，但是无效果！
PI; // 3.14
```

### 方法

给一个对象绑定函数，称为这个对象的方法：

```js
var xiaoming = {
    name: '小明',
    birth: 1990,
    age: function () {
        var y = new Date().getFullYear();
        return y - this.birth;
    }
};

xiaoming.age; // function xiaoming.age()
xiaoming.age(); // 今年调用是25,明年调用就变成26了
```

注意上述代码中使用了 `this` 关键字。`this` 的语义和 C/C++ 中的 `this` 类似，用来代表当前对象，也就是上述例子中的 `xiaoming` 对象。

然而这里的 `this` 有点奇葩，你必须用 `obj.xxx()` 的形式来调用，`this` 才能代表当前对象，否则 `this` 将指向 `window` 对象！(ECMA 规定，如果使用 strict 模式，这种情况下 `this` 将指向 `undefined`)。这意味着下面的代码都有问题：

```js
'use strict';

var xiaoming = {
    name: '小明',
    birth: 1990,
    age: function () {
        var y = new Date().getFullYear();
        return y - this.birth;
    }
};

// 先拿到 age 函数，再进行调用，不行，因为没有提供 obj, this 找不到 obj 就会指向 window 或 undefined
var fn = xiaoming.age;
fn(); // Uncaught TypeError: Cannot read property 'birth' of undefined
```

```js
'use strict';

var xiaoming = {
    name: '小明',
    birth: 1990,
    age: function () {
        // 不行，函数内部定义的函数，没有提供 obj, this 找不到 obj 就会指向 window 或 undefined
        function getAgeFromBirth() {
            var y = new Date().getFullYear();
            return y - this.birth;
        }
        return getAgeFromBirth();
    }
};

xiaoming.age(); // Uncaught TypeError: Cannot read property 'birth' of undefined
```

如果你真要使用 `this`, 首先避免上面的第一种写法，必须用 `obj.xxx()` 的方式来调用函数，其次，你可以在函数内部先获取到 `this`, 这样函数内部定义的函数就能访问到正确的 `this` 了：

```js
'use strict';

var xiaoming = {
    name: '小明',
    birth: 1990,
    age: function () {
        var that = this; // 在方法内部一开始就捕获this
        function getAgeFromBirth() {
            var y = new Date().getFullYear();
            return y - that.birth; // 用that而不是this
        }
        return getAgeFromBirth();
    }
};

xiaoming.age(); // 25
```

此外，JS 提供了 `apply` 和 `call` 方法，也可以部分解决上述问题，它的作用请看下述代码：

```js
function getAge() {
    var y = new Date().getFullYear();
    return y - this.birth;
}

var xiaoming = {
    name: '小明',
    birth: 1990,
    age: getAge
};

xiaoming.age(); // 25
getAge.apply(xiaoming, []); // 25, this指向xiaoming, 参数为空
```

`call` 和 `apply` 功能一样，只不过 `apply` 把函数参数用数组的方式进行打包，而 `call` 只是把参数按顺序传入。

利用 `apply` 还可以动态修改一个函数的定义：

```js
var count = 0;
var oldParseInt = parseInt; // 保存原函数

window.parseInt = function () {
    count += 1; // 统计 parseInt 调用次数
    return oldParseInt.apply(null, arguments); // 调用原函数
};

// 测试:
parseInt('10');
parseInt('20');
parseInt('30');
count; // 3
```

### 高阶函数

高阶函数 (Higher-order function), 意思是接收其他函数作为参数的函数。高阶函数定义了某种规则，利用其他函数来实现某种计算。

JS 内置了几个高阶函数，接下来我们分别介绍它们。

#### map/reduce

高阶函数 map 接受两个参数，一个数组 array, 一个函数 `f(x)`, map 会把 `f(x)` 作用到数组的每个元素上，计算结束后返回一个新的数组。

![]( {{ site.url }}/asset/javascript-learning-map.png)

JS 中调用 map 方法需要使用 `Array` ：

```js
function pow(x) {
    return x * x;
}

var arr = [1, 2, 3, 4, 5, 6, 7, 8, 9];
arr.map(pow); // [1, 4, 9, 16, 25, 36, 49, 64, 81]
```

高阶函数 reduce 接受两个参数，一个数组 array, 一个函数 `f(x, y)`, reduce 会把数组的元素通过 `f(x, y)`进行累计运算，返回累积运算之后的结果。其效果就是：

```
[x1, x2, x3, x4].reduce(f) = f(f(f(x1, x2), x3), x4)
```

例如数组的求和运算可以用 reduce 表达为：

```js
var arr = [1, 3, 5, 7, 9];
arr.reduce(function (x, y) {
    return x + y;
}); // 25
```

要把 `[1, 3, 5, 7, 9]` 变换成整数 `13579`，`reduce()`也能派上用场：

```js
var arr = [1, 3, 5, 7, 9];
arr.reduce(function (x, y) {
    return x * 10 + y;
}); // 13579
```

#### filter

高阶函数 filter 接受两个参数，一个数组 array, 一个函数 `f(x)`, filter 会根据 `f(x)` 返回的布尔值决定保留或丢弃数组中的指定元素：

```js
var arr = [1, 2, 4, 5, 6, 9, 10, 15];
var r = arr.filter(function (x) {
    return x % 2 !== 0;
});
r; // [1, 5, 9, 15]
```

#### sort

之前我们提到的 sort 也是一个高阶函数，它接受两个参数，一个数组 array, 一个函数 `f(x, y)`, sort 会根据 `f(x, y)` 返回的布尔值决定排序的顺序。

```js
// 忽略大小写的排序

var arr = ['Google', 'apple', 'Microsoft'];
arr.sort(function (s1, s2) {
    x1 = s1.toUpperCase();
    x2 = s2.toUpperCase();
    if (x1 < x2) {
        return -1;
    }
    if (x1 > x2) {
        return 1;
    }
    return 0;
}); // ['apple', 'Google', 'Microsoft']
```

值得一提的是，sort 如果没有提供排序函数，默认的排序规则会先把元素转换为字符串，然后进行排序。

### 闭包

闭包的概念很简单，就是可以把 函数 和 函数外的局部变量 打包在一起，这个整体就称为闭包。函数实际上是闭包的一种特殊情况。同时，由于函数在 JS 里是第一类值，因此可以作为返回值返回。

下面的代码返回了一个闭包，并且使用这个闭包进行计算：

```js
function count() {
    var arr = [];
    for (var i=1; i<=3; i++) {
        arr.push(function () {
            return i * i;
        });
    }
    return arr;
}

var results = count();
var f1 = results[0];
var f2 = results[1];
var f3 = results[2];

f1(); // 16
f2(); // 16
f3(); // 16
```

上面的代码有一个问题，函数 count 返回了三个闭包，当 count 返回时，实际上 `i` 的值已经变成了 4，因此每个函数的计算结果都是 16, 而不是我们所设想的，三个函数分别返回 `1, 4, 9`。

上述问题的根本原因在于，我们返回的三个闭包都引用了同一个变量 `i`, 并且这个变量 `i` 随后被修改了。因此在使用闭包时要注意，不要在闭包函数里使用那些 **后续** 会被修改的变量：

```js
function count() {
    var arr = [];
    for (var i=1; i<=3; i++) {
        arr.push((function (n) {
            return function () {
                return n * n;
            }
        })(i));
    }
    return arr;
}

var results = count();
var f1 = results[0];
var f2 = results[1];
var f3 = results[2];

f1(); // 1
f2(); // 4
f3(); // 9
```

上述代码中多创建了一个函数，将 `i` 作为参数传给了这个函数，这种方法避免了之前的问题。值得注意的是，如果要创建一个匿名函数并让函数立刻执行，应当像下面这样写：

```js
(function (x) {
    return x * x;
})(3);
```

用括号把匿名函数的定义括起来，后面再跟上函数的参数。

### 箭头函数

ES6 中新增了一种箭头函数 (Arrow Function)：

```js
x => x * x
```

它相当于

```js
function (x) {
    return x * x;
}
```

箭头函数的意义在于简化函数定义，可以看到它省略了 `function` 关键字、参数列表、大括号小括号 这些东西。你可以像如下代码那样定义含有多条语句的箭头函数：

```js
// 两个参数:
(x, y) => x * x + y * y

// 无参数:
() => 3.14

// 可变参数:
(x, y, ...rest) => {
    var i, sum = x + y;
    for (i=0; i<rest.length; i++) {
        sum += rest[i];
    }
    return sum;
}
```

箭头函数相比匿名函数而言有个明显的区别：箭头函数内部的 `this` 是词法作用域，由上下文确定：

```js
var obj = {
    birth: 1990,
    getAge: function () {
        var b = this.birth; // 1990
        var fn = () => new Date().getFullYear() - this.birth; // this指向obj对象
        return fn();
    }
};
obj.getAge(); // 25
```

### generator

ES6 标准引入了新的数据类型 generator, 它是一种可以多次返回的函数：

```js
function* fib(max) {
    var
        t,
        a = 0,
        b = 1,
        n = 1;
    while (n < max) {
        yield a;
        t = a + b;
        a = b;
        b = t;
        n ++;
    }
    return a;
}

var f = fib(5);
f.next(); // {value: 0, done: false}
f.next(); // {value: 1, done: false}
f.next(); // {value: 1, done: false}
f.next(); // {value: 2, done: false}
f.next(); // {value: 3, done: true}
```

generator 相比于普通函数的优势在于，它能记住 **状态**，yield 使得函数执行之后能够停下来，把当前的计算结果先返回，下一次再从这个状态开始继续计算。这里多说无用，以后遇到了实际的代码，就能体会它的妙处了。


## 标准对象

我们先用 `typeof` 操作符打印一下现在用过的几种数据类型：

```js
typeof 123; // 'number'
typeof NaN; // 'number'
typeof 'str'; // 'string'
typeof true; // 'boolean'
typeof undefined; // 'undefined'
typeof Math.abs; // 'function'
typeof null; // 'object'
typeof []; // 'object'
typeof {}; // 'object'
```

在 JS 中，一切都是对象，除了几个基本数据类型 `number`, `string`, `boolean`, `undefined`, `function` 以外。

有点奇葩的是 `null` 居然也是 object 类型，我有点懵逼。

除了这些之外，JS 还提供了包装对象：

```js
var n = new Number(123); // 123,生成了新的包装类型
var b = new Boolean(true); // true,生成了新的包装类型
var s = new String('str'); // 'str',生成了新的包装类型
```

这些包装对象是 object 类型，因此使用 `===` 进行比较的话将会报错：

```js
typeof new Number(123); // 'object'
new Number(123) === 123; // false
```

**不要使用包装类型**，这个东西看不出来有啥用，还会造成困惑。

更奇葩的是如果你不写 new 的话，那么 `Number()` 是一个函数，函数的作用是将一个变量转换为基本类型 `number`：

```js
var n = Number('123'); // 123，相当于parseInt()或parseFloat()
typeof n; // 'number'

var b = Boolean('true'); // true
typeof b; // 'boolean'

var b2 = Boolean('false'); // true! 'false'字符串转换结果为true！因为它是非空字符串！
var b3 = Boolean(''); // false

var s = String(123.45); // '123.45'
typeof s; // 'string'
```

你可以用下述方法来判断全局变量或局部变量是否存在：

```js
function foo() {
    if (typeof window.myVar === 'undefined') {
        // 全局变量 myVar 不存在
    }

    if (typeof myVar === 'undefined') {
        // 局部变量 myVar 不存在
    }
}
```

接下来介绍一下 JS 中提供的几个标准对象：

### Date

Data 对象用来表示当前时间：

```js
var now = new Date();
now; // Wed Jun 24 2015 19:49:22 GMT+0800 (CST)
now.getFullYear(); // 2015, 年份
now.getMonth(); // 5, 月份，注意月份范围是0~11，5表示六月
now.getDate(); // 24, 表示24号
now.getDay(); // 3, 表示星期三
now.getHours(); // 19, 24小时制
now.getMinutes(); // 49, 分钟
now.getSeconds(); // 22, 秒
now.getMilliseconds(); // 875, 毫秒数
now.getTime(); // 1435146562875, 以number形式表示的时间戳
```

上述代码有个非常坑爹的地方，它的月份是从 0 数到 11 的，因此 5 表示的是六月！

```js
// 将时间解析为时间戳
var d = Date.parse('2015-06-24T19:49:22.875+08:00');
d; // 1435146562875

// 从时间戳反推回时间
var d = new Date(1435146562875);
d; // Wed Jun 24 2015 19:49:22 GMT+0800 (CST)
```

时间戳是一个自增的整数，它表示从1970年1月1日零时整的GMT时区开始的那一刻，到现在的毫秒数。假设浏览器所在电脑的时间是准确的，那么世界上无论哪个时区的电脑，它们此刻产生的时间戳数字都是一样的，所以，时间戳可以精确地表示一个时刻，并且与时区无关。

所以，我们只需要传递时间戳，或者把时间戳从数据库里读出来，再让JavaScript自动转换为当地时间就可以了。

### RegExp

JS 中对正则表达式也提供了支持，这需要通过 RegExp 标准对象来使用：

```js
// 写正则表达式
var re = /^\d{3}\-\d{3,8}$/;

re.test('010-12345'); // true
re.test('010-1234x'); // false
re.test('010 12345'); // false
```

正则表达式有两种写法，一种是用斜杠括起来： `/正则表达式/` 另一种是通过 `new RegExp('正则表达式')`。

正则表达式的 `test()` 方法用于判断字符串是否匹配。

之前提到过 string 的 `split()` 方法可以对字符串进行切分，你可以给 `split()` 方法传入正则表达式来进行更准确的切分：

```js
// 普通的切分无法识别连续多个空格
'a b   c'.split(' '); // ['a', 'b', '', '', 'c']

// 正则表达式可以识别连续的空格
'a b   c'.split(/\s+/); // ['a', 'b', 'c']

// 除了空格还可以根据逗号切分
'a,b, c  d'.split(/[\s\,]+/); // ['a', 'b', 'c', 'd']
```

JS 中的正则表达式同样提供了分组的功能，或者可以称它为捕获：

```js
var re = /^(\d{3})-(\d{3,8})$/;
re.exec('010-12345'); // ['010-12345', '010', '12345']
re.exec('010 12345'); // null
```

可以看到，使用 `exec()` 进行分组之后会返回一个数组，数组的第一个是匹配的字符串，随后是分组的各个子串。

JS 正则默认是贪婪匹配，加上 `?` 可以变为非贪婪：

```js
var re = /^(\d+?)(0*)$/;
re.exec('102300'); // ['102300', '1023', '00']
```

JS 的正则表达式还可以指定一些特殊的标志，例如 `i` 表示忽略大小写、`m` 表示执行多行匹配、`g` 表示进行全局匹配：

```js
var s = 'JavaScript, VBScript, JScript and ECMAScript';
var re=/[a-zA-Z]+Script/g;

// 使用全局匹配，每次匹配都会记录上次匹配到的位置，下次调用 exec 会从此位置继续匹配:
re.exec(s); // ['JavaScript']
re.lastIndex; // 10

re.exec(s); // ['VBScript']
re.lastIndex; // 20

re.exec(s); // ['JScript']
re.lastIndex; // 29

re.exec(s); // ['ECMAScript']
re.lastIndex; // 44

re.exec(s); // null，直到结束仍没有匹配到
```

### JSON

JSON 是 JavaScript Object Notation 的缩写，是 JavaScript 支持的一种数据交换格式。

下面的代码将一个对象序列化为 JSON 字符串：

```js
var xiaoming = {
    name: '小明',
    age: 14,
    gender: true,
    height: 1.65,
    grade: null,
    'middle-school': '\"W3C\" Middle School',
    skills: ['JavaScript', 'Java', 'Python', 'Lisp']
};

JSON.stringify(xiaoming, null, '  '); 

/*
{
  "name": "小明",
  "age": 14,
  "gender": true,
  "height": 1.65,
  "grade": null,
  "middle-school": "\"W3C\" Middle School",
  "skills": [
    "JavaScript",
    "Java",
    "Python",
    "Lisp"
  ]
}
*/
```

`JSON.stringify()` 的第二个参数用来筛选对象的键，如果只想序列化指定的属性，可以传入一个数组：

```js
JSON.stringify(xiaoming, ['name', 'skills'], '  ');

/*
{
  "name": "小明",
  "skills": [
    "JavaScript",
    "Java",
    "Python",
    "Lisp"
  ]
}
*/
```

你还可以给第二个参数传入一个函数，这样每个 key-value 都会被这个函数处理一遍：

```js
function convert(key, value) {
    if (typeof value === 'string') {
        return value.toUpperCase();
    }
    return value;
}

JSON.stringify(xiaoming, convert, '  ');

/*
{
  "name": "小明",
  "age": 14,
  "gender": true,
  "height": 1.65,
  "grade": null,
  "middle-school": "\"W3C\" MIDDLE SCHOOL",
  "skills": [
    "JAVASCRIPT",
    "JAVA",
    "PYTHON",
    "LISP"
  ]
}
*/
```

从字符串反序列化出对象也很简单：

```js
JSON.parse('[1,2,3,true]'); // [1, 2, 3, true]
JSON.parse('{"name":"小明","age":14}'); // Object {name: '小明', age: 14}
JSON.parse('true'); // true
JSON.parse('123.45'); // 123.45
```


## 面向对象编程

JS 中的面向对象与 Java/C++ 等语言不一样，它没有提供 class 这个概念，而是提供了一种叫做 原型(prototype) 的东西来完成面向对象功能：

```js
var Student = {
    name: 'Robot',
    height: 1.2,
    run: function () {
        console.log(this.name + ' is running...');
    }
};

var xiaoming = {
    name: '小明'
};

xiaoming.__proto__ = Student;
```

如上，我们通过 `xiaoming.__proto__ = Student;` 这行语句，将 `xiaoming` 的原型设定为 `Student`，其效果就类似于 `xiaoming` 继承自 `Student`，这样就可以通过 `xiaoming` 来调用 `Student` 的 `run` 方法了：

```js
xiaoming.name; // '小明'
xiaoming.run(); // 小明 is running...
```

直接用 `__proto__` 指向的方式来修改原型不太好，标准的做法是使用 `Object.create()` 方法：

```js
// 原型对象:
var Student = {
    name: 'Robot',
    height: 1.2,
    run: function () {
        console.log(this.name + ' is running...');
    }
};

function createStudent(name) {
    // 基于Student原型创建一个新对象:
    var s = Object.create(Student);
    // 初始化新对象:
    s.name = name;
    return s;
}

var xiaoming = createStudent('小明');
xiaoming.run(); // 小明 is running...
xiaoming.__proto__ === Student; // true
```

### 创建对象

除了直接用 `{ ... }` 创建一个对象以外，JS 还支持用构造函数来创建对象：

```js
function Student(name) {
    this.name = name;
    this.hello = function () {
        alert('Hello, ' + this.name + '!');
    }
}

// 必须用 new 操作符来调用构造函数
var xiaoming = new Student('小明');
xiaoming.name; // '小明'
xiaoming.hello(); // Hello, 小明!
```

新创建的 xiaoming 对象的原型链是：

```js
xiaoming ----> Student.prototype ----> Object.prototype ----> null
```

使用构造函数来创建对象，有一个弊端是对象中的函数不是共享的，假如你又创建了一个 `xiaohong` 对象，那么 `xiaoming.hello` 函数和 `xiaohong.hello` 函数是两个不同的函数。这造成了内存的浪费。

你可以用如下写法来让创建出来的对象共享同一个函数：

```js
function Student(name) {
    this.name = name;
}

Student.prototype.hello = function () {
    alert('Hello, ' + this.name + '!');
};
```

上述写法还有一个比较坑的地方，如果你忘记写 new 了，也不一定会报错，而是会产生意外的结果：

```js
var xiaoming = Student('小明');
xiaoming.name; // '小明'
xiaoming.hello(); // Hello, 小明!
```

这时候 Student 变成了一个普通的函数，它内部的 this 指向的可能会是 `undefined` (strict 模式下) 也可能是 `window`。前者会报错，后者不会报错但比报错更加糟糕。

为此我们可以创建一个 `createStudent()` 函数，在内部封装 new 操作，一个常用的编程模式是：

```js
function Student(props) {
    this.name = props.name || '匿名'; // 默认值为'匿名'
    this.grade = props.grade || 1; // 默认值为1
}

Student.prototype.hello = function () {
    alert('Hello, ' + this.name + '!');
};

function createStudent(props) {
    return new Student(props || {})
}
```

这种写法非常灵活，你不需要用 new 就可以创建出对象，同时 `createStudent()` 的参数可以是空，也可以是键值对。

### 原型继承

JS 的原型是个很奇葩的东西，我们来看看上一节代码的原型链：

```js
function Student(props) {
    this.name = props.name || 'Unnamed';
}

Student.prototype.hello = function () {
    alert('Hello, ' + this.name + '!');
}
```

![]( {{ site.url }}/asset/javascript-learning-prototype-chain.png)

关于原型我没彻底想明白，以 class 的概念来推断原型的理论，这样是不可行的，它们俩思路不一样。这个以后再说，这里先略过它。

### class 继承

新的关键字 class 从 ES6 开始正式引入到了 JavaScript 中。`class` 的目的就是让类的定义变得更简单。

```js
class Student {
    constructor(name) {
        this.name = name;
    }

    hello() {
        alert('Hello, ' + this.name + '!');
    }
}

xiaoming = new Student("小明");
```

ES6 中的继承也变得非常简单：

```js
class PrimaryStudent extends Student {
    constructor(name, grade) {
        super(name); // 记得用super调用父类的构造方法!
        this.grade = grade;
    }

    myGrade() {
        alert('I am at grade ' + this.grade);
    }
}

xiaohong = new PrimaryStudent("小红", 90);
xiaohong.hello();
xiaohong.myGrade();
```

ES6 引入的 class 操作符，其作用是让 JavaScript 引擎去实现原来需要我们来编写的一大堆继承代码。让类的编写更简单，更容易理解。

因为并不是所有的主流浏览器都支持 class 语句，如果你需要在旧版本浏览器上使用，你可以使用 [babel](https://babeljs.io/) 这个工具，它能够把 class 定义的类转换成使用原型来定义的类。


## 错误处理

JS 提供了 `try ... catch ... finally` 这样的异常处理语句：

```js
`use strict`

var r1, r2, s = null;
try {
    r1 = s.length; // 此处应产生错误
    r2 = 100; // 该语句不会执行
} catch (e) {
    alert('出错了：' + e);
} finally {
    console.log('finally');
}
console.log('r1 = ' + r1); // r1应为undefined
console.log('r2 = ' + r2); // r2应为undefined
```

其中 `e` 是一个异常对象，表示出现异常的具体信息，JS 中的异常对象都从 `Error` 派生，包括 `TypeError`, `ReferenceError` 等错误：

```js
try {
    ...
} catch (e) {
    if (e instanceof TypeError) {
        alert('Type error!');
    } else if (e instanceof Error) {
        alert(e.message);
    } else {
        alert('Error: ' + e);
    }
}
```

程序也可以主动抛出错误：

```js
`use strict`

var r, n, s;
try {
    s = prompt('请输入一个数字');
    n = parseInt(s);
    if (isNaN(n)) {
        throw new Error('输入错误');
    }
    // 计算平方:
    r = n * n;
    alert(n + ' * ' + n + ' = ' + r);
} catch (e) {
    alert('出错了：' + e);
}
```

### 错误传播

如果代码发生了错误，又没有用 `try ... catch` 捕获，那么错误会被抛到外层调用函数，如果外层函数也没有捕获错误，那么错误会沿着函数调用链一直往外抛，直到抛给 JavaScript 引擎，代码中止执行。

### 异步错误处理

涉及到异步代码时需要注意，我们直接捕获异步函数中的错误往往是不行的，需要在回调函数中捕获错误：

```js
function printTime() {
    throw new Error();
}

// 这里捕获错误是没用的，因为错误不是 setTimeout 抛出的，是回调函数抛出的
try {
    setTimeout(printTime, 1000);
    console.log('done');
} catch (e) {
    alert('error');
}
```