---
layout: post
title:  "JavaScript 库 jQuery"
date:   2017-04-05 14:36:30 +0800
categories: javascript
---

* TOC
{:toc}

本文参考自 [廖雪峰的官方网站](http://www.liaoxuefeng.com/wiki/001434446689867b27157e896e74d51a89c25cc8b43bdb3000)

你可以把 jQuery 看做是一个 JS 库，它有如下好处：
- 消除浏览器差异；
- 简洁的操作 DOM 的方法：写 `$('#test')` 肯定比 `document.getElementById('test')` 来得简洁；
- 轻松实现动画、修改 CSS 等操作；

目前 jQuery 有 1.x 和 2.x 版本，区别在于 2.x 移除了古老的 IE6,7,8 的支持，因此 2.x 的代码更精简。选择哪个版本主要取决于你是否想支持更老的 IE 版本。

从 [jQuery 官网](http://jQuery.com/download/) 可以下载最新版本，jQuery 只是一个 `jquery-xxxx.js` 文件，但你会看到有 compressed 和 uncompressed 版本，使用时完全一样。

要使用 jQuery 只需要在页面的 `<head>` 标签中引入 jQuery 文件即可：

```html
<html>
<head>
    <script src="//code.jquery.com/jquery-1.11.3.min.js"></script>
    ...
</head>
<body>
    ...
</body>
</html>
```

`$` 是著名的 jQuery 符号，实际上，jQuery 把所有的功能全部封装在一个全局变量 `jQuery` 中，`$` 是这个变量的别名：

```js
window.jQuery; // jQuery(selector, context)
window.$; // jQuery(selector, context)
$ === jQuery; // true
typeof($); // 'function'
```

`$` 本质上就是一个函数，但是函数也是对象，于是 `$` 除了可以直接调用外，也可以有很多其他属性。


## 选择器

选择器时 jQuery 的核心，一个选择器写出来类似于 `$('#dom-id')`。

选择器可以帮助我们快速定位到一个或多个 DOM 节点。

### 按照 ID 查找

按照 ID 查找时，需要在 ID 前面加上 `#`：

```js
// 查找<div id="abc">:
var div = $('#abc');
```

选择器返回的是一个 jQuery 对象，它类似于数组，如果查找不到，将返回一个空数组，而不是 `null/undefined`。

jQuery 对象和 DOM 对象之间可以互相转化：

```js
var div = $('#abc'); // jQuery对象
var divDom = div.get(0); // 假设存在div，获取第1个DOM元素
var another = $(divDom); // 重新把DOM包装为jQuery对象
```

### 按照 tag 查找

按照 tag 查找直接写 tag 名就可以了：

```js
var ps = $('p'); // 返回所有<p>节点
ps.length; // 数一数页面有多少个<p>节点
```

### 按照 class 查找

按照 clas 查找需要在 class 名前面加上 `.`：

```js
var a = $('.red'); // 所有节点包含`class="red"`都将返回
// 例如:
// <div class="red">...</div>
// <p class="green red">...</p>
```

如果节点有多个 class, 可以同时查找：

```js
var a = $('.red.green'); // 注意没有空格！
// 符合条件的节点：
// <div class="red green">...</div>
// <div class="blue green red">...</div>
```

### 按照属性查找

把属性用 `[]` 括起来，就可以查找了：

```js
var email = $('[name=email]'); // 找出<??? name="email">
var passwordInput = $('[type=password]'); // 找出<??? type="password">
var a = $('[items="A B"]'); // 找出<??? items="A B">
```

还可以指定按照属性的前缀或后缀进行查找：

```js
var icons = $('[name^=icon]'); // 找出所有name属性值以icon开头的DOM
// 例如: name="icon-1", name="icon-2"
var names = $('[name$=with]'); // 找出所有name属性值以with结尾的DOM
// 例如: name="startswith", name="endswith"
```

### 组合查找

可以同时把上述几种选择器组合起来使用：

```js
var emailInput = $('input[name=email]'); // 不会找出<div name="email">

var tr = $('tr.red'); // 找出<tr class="red ...">...</tr>
```

### 多项选择器

组合查找是一个 “与” 的关系，我们可以在查找器之间加入 `,` 表示 “或” 的关系：

```js
$('p,div'); // 把<p>和<div>都选出来
$('p.red,p.green'); // 把<p class="red">和<p class="green">都选出来
```

要注意的是，选出来的元素是按照它们在 HTML 中出现的顺序排列的，而且不会有重复元素。


### 层级选择器

因为 DOM 的结构就是层级结构，所以我们经常要使用层级关系来进行选择，可以在选择器之间加入 ` ` 空格，来进行层级选择：

```html
<!-- HTML结构 -->
<div class="testing">
    <ul class="lang">
        <li class="lang-javascript">JavaScript</li>
        <li class="lang-python">Python</li>
        <li class="lang-lua">Lua</li>
    </ul>
</div>
```

```js
$('ul.lang li.lang-javascript'); // [<li class="lang-javascript">JavaScript</li>]
$('div.testing li.lang-javascript'); // [<li class="lang-javascript">JavaScript</li>]
```

### 子选择器

子选择器类似于层级选择器，但是限定了层级关系必须是父子关系，子选择器需要在选择器之间加入 `>` 符号：

```js
$('ul.lang>li.lang-javascript'); // 可以选出[<li class="lang-javascript">JavaScript</li>]
$('div.testing>li.lang-javascript'); // [], 无法选出，因为<div>和<li>不构成父子关系
```

### 过滤器

过滤器一般不单独使用，它通常附加在选择器上，对选择到的元素进行过滤：

```js
$('ul.lang li'); // 选出JavaScript、Python和Lua 3个节点

$('ul.lang li:first-child'); // 仅选出JavaScript
$('ul.lang li:last-child'); // 仅选出Lua
$('ul.lang li:nth-child(2)'); // 选出第N个元素，N从1开始
$('ul.lang li:nth-child(even)'); // 选出序号为偶数的元素
$('ul.lang li:nth-child(odd)'); // 选出序号为奇数的元素
```

### 表单相关

针对表单元素，jQuery 还有一组特殊的选择器：

- `:input`：可以选择 `<input>`，`<textarea>`，`<select>` 和 `<button>`；
- `:file`：可以选择 `<input type="file">` ，和 `input[type=file]` 一样；
- `:checkbox`：可以选择复选框，和 `input[type=checkbox]` 一样；
- `:radio`：可以选择单选框，和 `input[type=radio]` 一样；
- `:focus`：可以选择当前输入焦点的元素，例如把光标放到一个 `<input>` 上，用 `$('input:focus')` 就可以选出；
- `:checked` ：选择当前勾上的单选框和复选框，用这个选择器可以立刻获得用户选择的项目，如 `$('input[type=radio]:checked')`；
- `:enabled`：可以选择可以正常输入的 `<input>`、`<select>` 等，也就是没有灰掉的输入；
- `:disabled`：和 `:enabled` 正好相反，选择那些不能输入的。


## 查找和过滤

使用选择器获取到某个或某组 jQuery 对象后，我们可以用这个对象来进行查找和过滤：

```html
<!-- HTML结构 -->
<ul class="lang">
    <li class="js dy">JavaScript</li>
    <li class="dy">Python</li>
    <li id="swift">Swift</li>
    <li class="dy">Scheme</li>
    <li name="haskell">Haskell</li>
</ul>
```

用 `find()` 查找子元素：

```js
var ul = $('ul.lang'); // 获得<ul>
var dy = ul.find('.dy'); // 获得JavaScript, Python, Scheme
var swf = ul.find('#swift'); // 获得Swift
var hsk = ul.find('[name=haskell]'); // 获得Haskell
```

用 `parent()` 查找父元素：

```js
var swf = $('#swift'); // 获得Swift
var parent = swf.parent(); // 获得Swift的上层节点<ul>
var a = swf.parent('div.red'); // 从Swift的父节点开始向上查找，直到找到某个符合条件的节点并返回
```

用 `next()` 和 `prev()` 查找同级元素：

```js
var swift = $('#swift');

swift.next(); // Scheme
swift.next('[name=haskell]'); // Haskell，因为Haskell是后续第一个符合选择器条件的节点

swift.prev(); // Python
swift.prev('.js'); // JavaScript，因为JavaScript是往前第一个符合选择器条件的节点
```

用 `filter` 过滤掉不符合选择器条件的节点：

```js
var langs = $('ul.lang li'); // 拿到JavaScript, Python, Swift, Scheme和Haskell
var a = langs.filter('.dy'); // 拿到JavaScript, Python, Scheme
```

也可以传入函数：

```js
var langs = $('ul.lang li'); // 拿到JavaScript, Python, Swift, Scheme和Haskell
langs.filter(function () {
    return this.innerHTML.indexOf('S') === 0; // 返回S开头的节点
}); // 拿到Swift, Scheme
```

可以用 `map()` 方法把一个 jQuery 对象中包含的 DOM 节点转化成其他对象：

```js
var langs = $('ul.lang li'); // 拿到JavaScript, Python, Swift, Scheme和Haskell
var arr = langs.map(function () {
    return this.innerHTML;
}).get(); // 用get()拿到包含string的Array：['JavaScript', 'Python', 'Swift', 'Scheme', 'Haskell']
```

`first()`, `last()`, `slice()` 方法可以返回一个新的 jQuery 对象，把不需要的 DOM 节点过滤掉：

```js
var langs = $('ul.lang li'); // 拿到JavaScript, Python, Swift, Scheme和Haskell
var js = langs.first(); // JavaScript，相当于$('ul.lang li:first-child')
var haskell = langs.last(); // Haskell, 相当于$('ul.lang li:last-child')
var sub = langs.slice(2, 4); // Swift, Scheme, 参数和数组的slice()方法一致
```


## 操作 DOM

在我们之前的介绍中，修改 DOM 的 CSS，文本，设置 HTML 有些麻烦，而且存在浏览器兼容问题，有了 jQuery 对象后，做这些事情就很方便了：

### 修改 Text 和 HTML

```html
<!-- HTML结构 -->
<ul id="test-ul">
    <li class="js">JavaScript</li>
    <li name="book">Java &amp; JavaScript</li>
</ul>
```

```js
$('#test-ul li[name=book]').text(); // 'Java & JavaScript'
$('#test-ul li[name=book]').html(); // 'Java &amp; JavaScript'
```

`text()` 方法的返回值是我们要获取的文本，你传入一个新的字符串参数给 `text()` 就能设置新的文本：

```js
'use strict';
var j1 = $('#test-ul li.js');
var j2 = $('#test-ul li[name=book]');
j1.html('<span style="color: red">JavaScript</span>');
j2.text('JavaScript & ECMAScript');
```

一个 jQuery 对象可能会对应多个 DOM 节点，一旦对这个 jQuery 对象进行设置，就会反映到所有 DOM 节点上。也就是说 jQuery 对象具有 “批量修改” 的特点。

### 修改 CSS

对于下面的 html 结构，我们可以用 jQuery 批量修改它的 CSS 属性：

```html
<!-- HTML结构 -->
<ul id="test-css">
    <li class="lang dy"><span>JavaScript</span></li>
    <li class="lang"><span>Java</span></li>
    <li class="lang dy"><span>Python</span></li>
    <li class="lang"><span>Swift</span></li>
    <li class="lang dy"><span>Scheme</span></li>
</ul>
```

```js
$('#test-css li.dy>span').css('background-color', '#ffd351').css('color', 'red');
```

jQuery 还提供了 css 属性，可以让我们修改节点的 CSS 样式：

```js
var div = $('#test-div');
div.css('color'); // '#000033', 获取CSS属性
div.css('color', '#336699'); // 设置CSS属性
div.css('color', ''); // 清除CSS属性
```

### 显示和隐藏 DOM

要隐藏一个 DOM 节点，我们可以设置 CSS 的 `display` 属性为 `none` 利用 `css()` 方法就可以实现。不过这样做要恢复起来需要我们自己记录原有的 `display` 属性到底是 `block` 还是 `inline` 还是别的值。

jQuery 提供了 `show()` 和 `hide()` 方法，可以方便地完成这一工作：

```js
var a = $('a[target=_blank]');
a.hide(); // 隐藏
a.show(); // 显示
```

### 获取 DOM 信息

```js
// 浏览器可视窗口大小:
$(window).width(); // 800
$(window).height(); // 600

// HTML文档大小:
$(document).width(); // 800
$(document).height(); // 3500

// 某个div的大小:
var div = $('#test-div');
div.width(); // 600
div.height(); // 300
div.width(400); // 设置CSS属性 width: 400px，是否生效要看CSS是否有效
div.height('200px'); // 设置CSS属性 height: 200px，是否生效要看CSS是否有效
```

`attr()` 和 `removeAttr()` 方法用于操作 DOM 节点的属性：

```js
// <div id="test-div" name="Test" start="1">...</div>
var div = $('#test-div');
div.attr('data'); // undefined, 属性不存在
div.attr('name'); // 'Test'
div.attr('name', 'Hello'); // div的name属性变为'Hello'
div.removeAttr('name'); // 删除name属性
div.attr('name'); // undefined
```

### 操作表单

对于表单元素，jQuery 对象统一提供 `val()` 方法获取和设置对应的 `value` 属性：

```js
/*
    <input id="test-input" name="email" value="">
    <select id="test-select" name="city">
        <option value="BJ" selected>Beijing</option>
        <option value="SH">Shanghai</option>
        <option value="SZ">Shenzhen</option>
    </select>
    <textarea id="test-textarea">Hello</textarea>
*/
var
    input = $('#test-input'),
    select = $('#test-select'),
    textarea = $('#test-textarea');

input.val(); // 'test'
input.val('abc@example.com'); // 文本框的内容已变为abc@example.com

select.val(); // 'BJ'
select.val('SH'); // 选择框已变为Shanghai

textarea.val(); // 'Hello'
textarea.val('Hi'); // 文本区域已更新为'Hi'
```

### 修改 DOM 结构

要添加新的 DOM 节点，除了通过 jQuery 的 `html()` 这种方法外，还可以使用 `append()` 方法，例如：

```html
<div id="test-div">
    <ul>
        <li><span>JavaScript</span></li>
        <li><span>Python</span></li>
        <li><span>Swift</span></li>
    </ul>
</div>
```

```js
var ul = $('#test-div>ul');
ul.append('<li><span>Haskell</span></li>');
```

除了接受字符串，`append()` 方法还接受传入原始的 DOM 对象、jQuery 对象和函数对象：

```js
// 创建DOM对象:
var ps = document.createElement('li');
ps.innerHTML = '<span>Pascal</span>';
// 添加DOM对象:
ul.append(ps);

// 添加jQuery对象:
ul.append($('#scheme'));

// 添加函数对象:
ul.append(function (index, html) {
    return '<li><span>Language - ' + index + '</span></li>';
});
```

`append()` 把节点添加到最后，`prepend()` 则把元素添加到最前。如果要把新节点插入到指定位置，可以先定位到之前的一个节点，然后利用 `after()` 方法：

```js
var js = $('#test-div>ul>li:first-child');
js.after('<li><span>Lua</span></li>');
```

也就是说，同级节点可以使用 `before()` 或 `after()` 方法。

要删除 DOM 节点，直接调用 `remove()` 方法就可以了：

```js
var li = $('#test-div>ul>li');
li.remove(); // 所有<li>全被删除
```


## 事件

利用 jQuery 可以方便地监听 DOM 节点上的各种事件，无需考虑不同浏览器间的差异：

```js
/* HTML:
 *
 * <a id="test-link" href="#0">点我试试</a>
 *
 */

// 获取超链接的jQuery对象:
var a = $('#test-link');
a.on('click', function () {
    alert('Hello!');
});
```

`on` 方法用来绑定一个事件，我们需要传入事件名称和对应的处理函数。另一种更简化的方法是直接调用 `click()` 方法：

```js
a.click(function () {
    alert('Hello!');
});
```

jQuery 能够绑定的事件主要包括：
- 鼠标事件：
    - click;
    - dbclick;
    - mouseenter;
    - mouseleave;
    - mousemove;
    - hover;
- 键盘事件：(仅作用在当前焦点的 DOM 上，通常是 `<input>` 和 `<textarea>`)
    - keydown;
    - keyup;
    - keypress;
- 其他事件：
    - focus;
    - blur: 当 DOM 失去焦点后触发；
    - change: 当 `<input>`, `<select>` 或 `<textarea>` 的内容改变时触发；
    - submit: 当 `<form>` 提交时触发；
    - ready: 当页面被载入并且 DOM 树完成初始化后触发；

其中 `ready` 事件仅作用于 `document` 对象。由于 `ready` 事件在 DOM 完成初始化后触发，且仅触发一次，所以非常适合用来编写初始化代码。假设我们想给一个 `<form>` 表单绑定 `submit` 事件，下面的代码将不能生效：

```html
<html>
<head>
    <script>
        // 代码有误:
        $('#testForm).on('submit', function () {
            alert('submit!');
        });
    </script>
</head>
<body>
    <form id="testForm">
        ...
    </form>
</body>
```

因为 `<script>` 标签中的代码执行时，`<form>` 尚未载入浏览器，所以 `$('#testForm)` 返回 `[]`，并没有绑定事件到任何 DOM 上。所以我们需要在收到 `ready` 事件之后再去做绑定操作：

```html
<html>
<head>
    <script>
        $(document).on('ready', function () {
            $('#testForm).on('submit', function () {
                alert('submit!');
            });
        });
    </script>
</head>
<body>
    <form id="testForm">
        ...
    </form>
</body>
```

由于 `ready` 操作非常普遍，所以可以这样简化：

```js
$(document).ready(function () {
    // on('submit', function)也可以简化:
    $('#testForm).submit(function () {
        alert('submit!');
    });
});
```

甚至可以进一步简化为：

```js
$(function () {
    // init...
});
```

上面这种简化的写法最为普遍。如果你遇到 `$(function() { ... })` 这种形式，牢记这是 `document` 对象的 `ready` 事件处理函数。

你可以多次绑定同一个事件，它的事件处理函数会依次执行：

```js
$(function () {
    console.log('init A...');
});
$(function () {
    console.log('init B...');
});
$(function () {
    console.log('init C...');
});
```

### 事件参数

有些事件，例如 `mousemove` 和 `keypress` 会传入 `Event` 对象作为参数，可以从 `Event` 对象中获取到更多的信息：

```js
$(function () {
    $('#testMouseMoveDiv').mousemove(function (e) {
        $('#testMouseMoveSpan').text('pageX = ' + e.pageX + ', pageY = ' + e.pageY);
    });
});
```

### 取消绑定

取消绑定可以通过 `off('click', function)` 这样的写法来实现：

```js
function hello() {
    alert('hello!');
}

a.click(hello); // 绑定事件

// 10秒钟后解除绑定:
setTimeout(function () {
    a.off('click', hello);
}, 10000);
```

### 事件触发条件

一个需要注意的问题是，事件的触发总是由用户引起的，例如，我们监视文本框的内容改动：

```js
var input = $('#test-input');
input.change(function () {
    console.log('changed...');
});
```

当用户在文本框输入内容时，我们会收到 `change` 事件，但如果我们在 js 代码改变文本框内容，将不会触发 `change` 事件。

```js
var input = $('#test-input');
input.val('change it!'); // 无法触发change事件
```

如果我们希望在代码中触发 `change` 事件，可以使用 `change()` 函数：

```js
var input = $('#test-input');
input.val('change it!');
input.change(); // 触发change事件
```

`input.change()` 相当于 `input.trigger('change')` 它是 `trigger()` 方法的简写。

### 浏览器安全限制

在浏览器中，有些 JavaScript 代码只能在用户触发的情况下执行，例如 `window.open()` 函数：

```js
// 无法弹出新窗口，将被浏览器屏蔽:
$(function () {
    window.open('/');
});
```

这些 “敏感代码” 只能由用户操作来触发：

```js
var button1 = $('#testPopupButton1');
var button2 = $('#testPopupButton2');

function popupTestWindow() {
    window.open('/');
}

button1.click(function () {
    popupTestWindow();
});

button2.click(function () {
    // 不立刻执行popupTestWindow()，100毫秒后执行:
    setTimeout(popupTestWindow, 100);
});
```

上述例子中，延迟执行的 button2 代码将被浏览器拦截。


## 动画

JavaScript 中实现动画的原理非常简单，只需要以固定的时间间隔，每个把 DOM 元素的 CSS 样式修改一点，看起来就像动画了。

但是在 JS 中手动实现动画效果非常麻烦，jQuery 封装了常见的几种动画效果，只需简单的一行代码就可以实现动画。

### show/hide

之前我们用 `show/hide` 无参数的方式来显示和隐藏元素，只需要传一个时间参数进去，就会给显示和隐藏效果加上动画：

```js
var div = $('#test-show-hide');
div.hide(3000); // 在3秒钟内逐渐消失

var div = $('#test-show-hide');
div.show('slow'); // 在0.6秒钟内逐渐显示
```

`toggle()` 会记录动画当前的状态，决定调用时是 `show()` 还是 `hide()`。

### slideUp/slideDown

`show/hide` 是从左上角逐渐展开或收缩的，而 `slideUp/slideDown` 是在垂直方向上展开或收缩的。

同理，`slideToggle()` 会根据 DOM 节点当前的状态来决定展开还是收缩。

### fadeIn/fadeOut

`fadeIn/fadeOut` 的动画效果是淡入淡出，也就是通过不断设置 DOM 元素的 `opacity` 属性来实现的。

同理，`fadeToggle()` 会根据 DOM 节点当前的状态来决定淡入或者淡出。

```js
var div = $('#test-fade');
div.fadeOut('slow'); // 在0.6秒内淡出
```

### 自定义动画

jQuery 的 `animate()` 方法可以实现自定义动画，我们需要传入的参数是 DOM 元素最终的 CSS 状态、以及所花费的时间，jQuery 会在这段时间内不断调整 CSS 直到达到我们设定的值：

```js
var div = $('#test-animate');
div.animate({
    opacity: 0.25,
    width: '256px',
    height: '256px'
}, 3000); // 在3秒钟内CSS过渡到设定值
```

`animate()` 还可以再传入一个函数作为参数，当动画执行完毕后，该函数将被调用：

```js
var div = $('#test-animate');
div.animate({
    opacity: 0.25,
    width: '256px',
    height: '256px'
}, 3000, function () {
    console.log('动画已结束');
    // 恢复至初始状态:
    $(this).css('opacity', '1.0').css('width', '128px').css('height', '128px');
});
```

### 串行动画

jQuery 的动画还可以串行执行，通过 `delay()` 方法可以实现动画的暂停：

```js
var div = $('#test-animates');
// 动画效果：slideDown - 暂停 - 放大 - 暂停 - 缩小
div.slideDown(2000)
   .delay(1000)
   .animate({
       width: '256px',
       height: '256px'
   }, 2000)
   .delay(1000)
   .animate({
       width: '128px',
       height: '128px'
   }, 2000);
}
```

你可能会遇到，有的动画如 `slideUp()` 根本没有效果。这是因为 jQuery 动画的原理是逐渐改变 CSS 的值，如 height 从 100px 逐渐变为 0。但是很多不是 block 性质的 DOM 元素，对它们设置 height 根本就不起作用，所以动画也就没有效果。

此外，jQuery 也没有实现对 `background-color` 的动画效果，用 `animate()` 设置 `background-color` 也没有效果。这种情况下可以使用 CSS3 的 `transition` 实现动画效果。


## AJAX

我们之前已经介绍过用 JavaScript 写 AJAX 的方法，它存在着浏览器兼容问题，并且状态和错误处理写起来也很麻烦。

### ajax 函数

jQuery 在全局对象 `jQuery` (也就是 `$`) 上绑定了一个 `ajax()` 函数，可以处理 `ajax()` 请求。`ajax(url, setting)` 函数接受一个 url 和一个可选的 `settings` 对象，常用选项如下:
- async：是否异步执行 AJAX 请求，默认为 `true` ，千万不要指定为 `false`；
- method：发送的 Method，缺省为'GET'，可指定为'POST'、'PUT'等；
- contentType：发送 POST 请求的格式，默认值为 `'application/x-www-form-urlencoded; charset=UTF-8'`，也可以指定为 `text/plain`、`application/json`；
- data：发送的数据，可以是字符串、数组或 object。如果是 `GET` 请求，data 将被转换成 query 附加到 URL 上，如果是 POST 请求，根据 contentType 把 data 序列化成合适的格式；
- headers：发送的额外的 HTTP 头，必须是一个 object；
- dataType：接收的数据格式，可以指定为'html'、'xml'、'json'、'text'等，缺省情况下根据响应的 Content-Type 猜测。

下面的例子发送一个 GET 请求，并返回一个 JSON 格式的数据：

```js
var jqxhr = $.ajax('/api/categories', {
    dataType: 'json'
});
// 请求已经发送了
```

请求结束后，可以使用类似于 Promise 的方法来处理结果：

```js
'use strict';

function ajaxLog(s) {
    var txt = $('#test-response-text');
    txt.val(txt.val() + '\n' + s);
}

$('#test-response-text').val('');

var jqxhr = $.ajax('/api/categories', {
    dataType: 'json'
}).done(function (data) {
    ajaxLog('成功, 收到的数据: ' + JSON.stringify(data));
}).fail(function (xhr, status) {
    ajaxLog('失败: ' + xhr.status + ', 原因: ' + status);
}).always(function () {
    ajaxLog('请求完成: 无论成功或失败都会调用');
});
```

### get 函数

由于 GET 请求最为常见，所以 jQuery 提供了 `get()` 函数，可以这么写：

```js
var jqxhr = $.get('/path/to/resource', {
    name: 'Bob Lee',
    check: 1
});
```

第二个参数如果是 object, jQuery 会自动把它转换成 query string 附加到 url 后面：

```
/path/to/resource?name=Bob%20Lee&check=1
```

这样我们就不用关心如何用 URL 编码并构造一个 query string 了。

### post 函数

`post()` 和 `get()` 类似，但是传入的第二个参数默认被序列化为 `application/x-www-form-urlencoded` ：

```js
var jqxhr = $.post('/path/to/resource', {
    name: 'Bob Lee',
    check: 1
});
```

实际构造的数据 `name=Bob%20Lee&check=1` 作为 post 的 body 发送。

### getJSON

由于 JSON 用得越来越普遍，所以 jQuery 也提供了 `getJSON()` 方法来快速通过 GET 获取一个 JSON 对象：

```js
var jqxhr = $.getJSON('/path/to/resource', {
    name: 'Bob Lee',
    check: 1
}).done(function (data) {
    // data已经被解析为JSON对象了
});
```

### 安全限制

jQuery 中的 AJAX 操作完全是封装了 JS 的 AJAX 操作，所以它的安全限制和前面讲的用 JavaScript 写 AJAX 时完全一样，也存在着同源策略的限制。

如果要使用 JSONP，可以在 `ajax()` 中设置 `jsonp:'callback'` 让 jQuery 实现 JSONP 跨域加载数据。


## 扩展

当我们使用 jQuery 对象的方法时，由于 jQuery 对象可以操作一组 DOM, 而且支持链式操作，所以用起来很方便。

但是 jQuery 内置的方法不可能满足我们所有的需求，有时候我们还是得写出下面这样的代码：

```js
$('span.hl').css('backgroundColor', '#fffceb').css('color', '#d85030');

$('p a.hl').css('backgroundColor', '#fffceb').css('color', '#d85030');
```

要高亮某些元素，需要重复写许多代码，能不能统一起来，写一个 `highlight()` 方法呢？

我们可以扩展 jQuery 来实现自定义方法，这种方式也称为编写 jQuery 插件。

给 jQuery 对象绑定一个新方法是通过扩展 `$.fn` 方法来实现的：

```js
$.fn.highlight1 = function () {
    // this已绑定为当前jQuery对象:
    this.css('backgroundColor', '#fffceb').css('color', '#d85030');
    return this;
}

$('#test-highlight1 span').highlight1();
```

注意上述代码中的 `return this;` 这是为了让我们编写的 `highlight` 方法能够支持链式操作：

```js
$('span.hl').highlight1().slideDown();
```

我们还能给 `highlight` 传入参数，并为参数设定默认值：

```js
$.fn.highlight = function (options) {
    // 合并默认值和用户设定值:
    var opts = $.extend({}, $.fn.highlight.defaults, options);
    this.css('backgroundColor', opts.backgroundColor).css('color', opts.color);
    return this;
}

// 设定默认值:
$.fn.highlight.defaults = {
    color: '#d85030',
    backgroundColor: '#fff8de'
}
```

最终，我们得出编写一个 jQuery 插件的原则：
- 给 `$.fn` 绑定函数，实现插件的代码逻辑；
- 插件函数最后要 `return this;` 以支持链式调用；
- 插件函数要有默认值，绑定在 `$.fn.<pluginName>.defaults` 上；
- 用户在调用时可传入设定值以便覆盖默认值；

### 针对特定元素的扩展

我们知道 jQuery 对象的有些方法只能作用在特定 DOM 元素上，比如 `submit()` 方法只能针对 `form`。如果希望我们编写的扩展只针对某些类型的 DOM 元素，可以借助 jQuery 选择器支持的 `filter()` 方法：

```js
// 给所有指向外链的超链接加上跳转提示
$.fn.external = function () {
    // return返回的each()返回结果，支持链式调用:
    return this.filter('a').each(function () {
        // 注意: each()内部的回调函数的this绑定为DOM本身!
        var a = $(this);
        var url = a.attr('href');
        if (url && (url.indexOf('http://')===0 || url.indexOf('https://')===0)) {
            a.attr('href', '#0')
             .removeAttr('target')
             .append(' <i class="uk-icon-external-link"></i>')
             .click(function () {
                if(confirm('你确定要前往' + url + '？')) {
                    window.open(url);
                }
            });
        }
    });
}
```