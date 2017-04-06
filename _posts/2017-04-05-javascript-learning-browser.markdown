---
layout: post
title:  "JavaScript 浏览器操作"
date:   2017-04-05 09:56:30 +0800
categories: javascript
---

* TOC
{:toc}

本文参考自 [廖雪峰的官方网站](http://www.liaoxuefeng.com/wiki/001434446689867b27157e896e74d51a89c25cc8b43bdb3000)


## 浏览器对象

JavaScript 可以获取浏览器提供的很多对象，并进行操作；

### window

`window` 对象我们在之前介绍过，它充当了全局作用域。不仅如此，它还代表了浏览器窗口。

`window` 对象有 `innerWidth` 和 `innerHeight` 属性，可以获取浏览器窗口的内部宽度和高度。内部宽度是指去除了菜单栏、工具栏、边框等占位元素后，用于显示网页的净宽高。对应的，还有 `outerWidth`, `outerHeight` 属性，可以获取浏览器窗口的整个宽高。

其他的属性我们之后再介绍。

### navigator

`navigator` 表示浏览器的信息，常用的属性包括：

- navigator.appName: 浏览器名称；
- navigator.appVersion: 浏览器版本；
- navigator.language : 浏览器设置的语言；
- navigator.platform: 操作系统类型；
- navigator.userAgent: 浏览器设定的 `User-Agent` 字符串；

需要注意的是，由于上面的这些信息可以很容易地被用户修改，所以 js 读取的值不一定是正确的。例如下面根据内核版本来使用不同代码的方法：

```js
var width;
if (getIEVersion(navigator.userAgent) < 9) {
    width = document.body.clientWidth;
} else {
    width = window.innerWidth;
}
```

这样写可能判断不准确，正确的做法是利用 JS 对不存在属性返回 `undefined` 的特性，直接用短路运算符 `||` 计算：

```js
// 如果浏览器不支持 innerWidth 属性，就使用 document.body.clientWidth 属性
var width = window.innerWidth || document.body.clientWidth;
```

### screen

`screen` 属性表示屏幕的信息，常用的属性有：

- screen.width, screen.height : 屏幕宽高，以像素为单位；
- screen.colorDepth : 返回颜色的位数；

### location

`location` 对象表示当前页面的 URL 信息，例如下面的代码获取 url 各部分的信息：

```js
// http://www.example.com:8080/path/index.html?a=1&b=2#TOP

location.protocol; // 'http'
location.host; // 'www.example.com'
location.port; // '8080'
location.pathname; // '/path/index.html'
location.search; // '?a=1&b=2'
location.hash; // 'TOP'
```

要加载一个新页面，可以调用 `location.assign()`. 如果要重新加载当前页面，可以使用 `location.reload()`

### document

`document` 对象表示当前页面。由于 HTML 在浏览器中以 DOM 形式表现为树结构，`document` 对象就是整个 DOM 树的根节点。

`document` 对象的 `title` 属性是从 HTML 文档中的 `<title>xxxx</title>` 读取的，但是可以动态改变。

要查找 DOM 树的某个节点，需要从 `document` 对象开始查找。最常用的查找是根据 ID 和 Tag Name：

```html
<dl id="drink-menu" style="border:solid 1px #ccc;padding:6px;">
    <dt>摩卡</dt>
    <dd>热摩卡咖啡</dd>
    <dt>酸奶</dt>
    <dd>北京老酸奶</dd>
    <dt>果汁</dt>
    <dd>鲜榨苹果汁</dd>
</dl>
```

由 `document` 对象提供的 `getElementById()` 和 `getElementByTagName()` 可以按 ID 获得一个 DOM 节点和按 Tag Name 获得一组 DOM 节点：

```js
`use strict`

var menu = document.getElementById('drink-menu');
var drinks = document.getElementsByTagName('dt');
var i, s, menu, drinks;

menu = document.getElementById('drink-menu');
menu.tagName; // 'DL'

drinks = document.getElementsByTagName('dt');
s = '提供的饮料有:';
for (i=0; i<drinks.length; i++) {
    s = s + drinks[i].innerHTML + ',';
}
alert(s);
```

`document` 对象还有一个 `cookie` 属性，可以获取当前页面的 Cookie. 

Cookie 是由服务器发送的 key-value 标识符。因为 HTTP 协议是状态无关的，但是服务器要区分到底是哪个用户发来的请求，就可以用 Cookie 来区分。当一个用户登录成功后，服务器发送一个 Cookie 给浏览器，例如 `user=xxxxxx`，此后，浏览器访问该网站时，会在请求头上附上这个 Cookie, 服务器根据 Cookie 即可区分出用户。

Cookie 还可以存储网站的一些设置，例如页面显示的语言等。

JS 可以通过 `document.cookie` 属性获取到当前的 cookie:

```js
document.cookie; // 'v=123; remember=true; prefer=zh'
```

由于 cookie 可以被 JS 读取，而网页中难免会嵌入第三方的 JS 代码，这就造成了巨大的安全隐患。为了解决这个问题，服务器在设置 Cookie 时可以使用 `httpOnly`, 设定了这个标记的 Cookie 不能被 JS 读取，只能由浏览器来自动完成 cookie 的发送。

为了确保安全，服务器端在设置 cookie 时，应该始终坚持使用 `httpOnly`.

### history

`history` 保存了浏览器的历史记录，JS 可以调用 `history` 对象的 `back()` 或 `forward()`, 相当于用户点击了后退或前进按钮。

这个对象属于历史遗留对象，任何情况下你都不应该使用 `history` 对象了。


## 操作 DOM

对于 DOM 树的操作，无非分为增删改查这几种。

### 获取 DOM 节点

在操作一个 DOM 节点之前，我们需要先拿到这个 DOM 节点。常用的方法是 `document.getElementById()` 和 `document.getElementsByTagName()` 以及 CSS 选择器 `document.getElementsByClassName`：

```js
// 返回ID为'test'的节点：
var test = document.getElementById('test');

// 先定位ID为'test-table'的节点，再返回其内部所有tr节点：
var trs = document.getElementById('test-table').getElementsByTagName('tr');

// 先定位ID为'test-div'的节点，再返回其内部所有class包含red的节点：
var reds = document.getElementById('test-div').getElementsByClassName('red');

// 获取节点test下的所有直属子节点:
var cs = test.children;

// 获取节点test下第一个、最后一个子节点：
var first = test.firstElementChild;
var last = test.lastElementChild;
```

第二种方法是使用 `querySelector()` 和 `querySelectorAll()`，这需要了解 selector 的语法，然后使用条件来获取节点，更加方便：

```js
// 通过querySelector获取ID为q1的节点：
var q1 = document.querySelector('#q1');

// 通过querySelectorAll获取q1节点内的符合条件的所有节点：
var ps = q1.querySelectorAll('div.highlighted > p');
```

严格地讲，我们这里的 DOM 节点是指 `Element`, 但是 DOM 节点实际上是 `Node`, 在 HTML 中，`Node` 包括 `Element`, `Comment`, `CDATA_SECTION` 等很多种，以及根节点 `Document` 类型。但是绝大多数时候我们仅关心 `Element` 也就是实际控制页面结构的 `Node`，其他类型的 `Node` 忽略即可。

### 更新 DOM 节点

拿到 DOM 节点之后，你可以对它的属性进行修改，方法有两种。

一种是修改 `innerHTML` 属性，这种方式非常强大，不但可以修改 DOM 节点的文本内容，还可以直接通过 HTML 片段修改 DOM 节点内部的子树：

```js
// 获取<p id="p-id">...</p>
var p = document.getElementById('p-id');
// 设置文本为abc:
p.innerHTML = 'ABC'; // <p id="p-id">ABC</p>
// 设置HTML:
p.innerHTML = 'ABC <span style="color:red">RED</span> XYZ';
// <p>...</p>的内部结构已修改
```

用 `innerHTML` 时要注意，如果写入的 HTML 内容是通过网络获取的，要注意对字符串进行编码来避免 XSS 攻击。

第二种是修改 `innerText` 或 `textContent` 属性，这个属性只能用于修改文本，不会插入新的 HTML 标签：

```js
// 获取<p id="p-id">...</p>
var p = document.getElementById('p-id');
// 设置文本:
p.innerText = '<script>alert("Hi")</script>';
// 插入的是文本，无法设置一个<script>节点:
// <p id="p-id">&lt;script&gt;alert("Hi")&lt;/script&gt;</p>
```

要修改 CSS，可以通过 节点的 `style` 属性来完成：

```js
// 获取<p id="p-id">...</p>
var p = document.getElementById('p-id');
// 设置CSS:
p.style.color = '#ff0000';
p.style.fontSize = '20px';
p.style.paddingTop = '2em';
```

### 插入 DOM 节点

之前介绍的 `innerHTML` 属性，会把原来的节点删除掉，如果我们想插入一个新的节点，有两个方法：

第一种是使用 `appendChild()` 方法，它会把一个子节点添加到父节点的最后一个节点，例如：

```html
<!-- HTML结构 -->
<p id="js">JavaScript</p>
<div id="list">
    <p id="java">Java</p>
    <p id="python">Python</p>
    <p id="scheme">Scheme</p>
</div>
```

下面代码把 `<p id="js">JavaScript</p>` 添加到 `<div id="list">` 的最后一项：

```js
var
    js = document.getElementById('js'),
    list = document.getElementById('list');
list.appendChild(js);
```

上面的代码会导致 list 节点先被删除，然后再插入到新的位置，一般我们会从零创建一个新的节点，然后插入到指定位置：

```js
// 实现的效果和上一段代码一样，但这样不会导致节点删除后插入的浪费
var
    list = document.getElementById('list'),
    haskell = document.createElement('p');
haskell.id = 'haskell';
haskell.innerText = 'Haskell';
list.appendChild(haskell);
```

我们可以利用上述方法来插入一个 `<style>` 标签，从而给文档添加新的 CSS 定义：

```js
var d = document.createElement('style');
d.setAttribute('type', 'text/css');
d.innerHTML = 'p { color: red }';
document.getElementsByTagName('head')[0].appendChild(d);
```

如果我们要把子节点插入到指定的位置，可以使用 `parentElement.insertBefore(newElement, referenceElement);` 这样子节点会插入到 `referenceElement` 节点之前：

```js
var
    list = document.getElementById('list'),
    ref = document.getElementById('python'),
    haskell = document.createElement('p');
haskell.id = 'haskell';
haskell.innerText = 'Haskell';
list.insertBefore(haskell, ref);
```

要使用 `insertBefore` 的关键在于要拿到一个 “参考子节点” 的引用，很多时候我们需要循环父节点的所有子节点，这可以通过迭代 `children` 属性来实现：

```js
var
    i, c,
    list = document.getElementById('list');
for (i = 0; i < list.children.length; i++) {
    c = list.children[i]; // 拿到第i个子节点
}
```

### 删除 DOM 节点

要删除一个 DOM 节点，首先要获得它的父节点，然后调用父节点的 `removeChild` 把自己删除掉：

```js
// 拿到待删除节点:
var self = document.getElementById('to-be-removed');
// 拿到父节点:
var parent = self.parentElement;
// 删除:
var removed = parent.removeChild(self);
removed === self; // true
```

值得注意的是，虽然子节点已经从 DOM 树中删除了，但它作为一个节点仍然存在于内存中，可以随时添加到 DOM 树里。


## 操作表单

用 JS 操作表单与操作 DOM 是类似的，因为表单本身也是 DOM 树：

HTML 表单的输入控件主要有以下几种：
- 文本框：对应 `<input type="text">` 用于输入文本；
- 口令框：对应 `<input type="password>"` 用于输入口令；
- 单选框：对应 `<input type="radio">` 用于选择一项；
- 复选框：对应 `<input type="checkbox">`；
- 下拉框：对应 `<select>`；
- 隐藏文本：对应 `<input type="hidden">` 用户不可见，但表单提交时会把隐藏文本发送到服务器；

### 获取和设置 value

如果我们获取到一个 `<input>` 节点，就可以直接通过 `value` 属性来获取用户输入的值：

```js
// <input type="text" id="email">
var input = document.getElementById('email');
input.value; // '用户输入的值'
```

对于 radio 和 checkbox 我们应该用 `checked` 来判断：

```js
// <label><input type="radio" name="weekday" id="monday" value="1"> Monday</label>
// <label><input type="radio" name="weekday" id="tuesday" value="2"> Tuesday</label>
var mon = document.getElementById('monday');
var tue = document.getElementById('tuesday');
mon.value; // '1'
tue.value; // '2'
mon.checked; // true或者false
tue.checked; // true或者false
```

设置值很简单，直接对 `value` 属性赋值就可以了：

```js
// <input type="text" id="email">
var input = document.getElementById('email');
input.value = 'test@example.com'; // 文本框的内容已更新
```

对于 radio 和 checkbox, 设置 `checked` 属性即可。

### 提交表单

JS 有两种方式来处理表单的提交（AJAX 方式在后面文章介绍）。

第一种是通过 `<form>` 元素的 `submit()` 方法提交一个表单，例如，响应一个 `<button>` 的 `click` 事件，在 JS 代码中提交表单：

```html
<!-- HTML -->
<form id="test-form">
    <input type="text" name="test">
    <button type="button" onclick="doSubmitForm()">Submit</button>
</form>

<script>
function doSubmitForm() {
    var form = document.getElementById('test-form');
    // 可以在此修改form的input...
    // 提交form:
    form.submit();
}
</script>
```

这种方式的缺点是扰乱了浏览器对 form 的正常提交，浏览器默认点击 `<button type="submit">` 时提交表单，或者用户在最后一个输入框按回车键。因此第二种方法是响应 `<form>` 本身的 `onsubmit` 事件：

```html
<!-- HTML -->
<form id="test-form" onsubmit="return checkForm()">
    <input type="text" name="test">
    <button type="submit">Submit</button>
</form>

<script>
function checkForm() {
    var form = document.getElementById('test-form');
    // 可以在此修改form的input...
    // 继续下一步:
    return true;
}
</script>
```

注意你需要 `return true;` 来告诉浏览器继续提交。

在检查和修改 `<input>` 时，要充分利用 `<input type="hidden">` 来传递数据：

```html
<!-- HTML -->
<form id="login-form" method="post" onsubmit="return checkForm()">
    <input type="text" id="username" name="username">
    <input type="password" id="input-password">
    <input type="hidden" id="md5-password" name="password">
    <button type="submit">Submit</button>
</form>

<script>
function checkForm() {
    var input_pwd = document.getElementById('input-password');
    var md5_pwd = document.getElementById('md5-password');
    // 把用户输入的明文变为MD5:
    md5_pwd.value = toMD5(input_pwd.value);
    // 继续下一步:
    return true;
}
</script>
```


## 操作文件

在 HTML 表单中，唯一可以上传文件的控件就是 `<input type="file">`。

注意：当一个表单包含 `<input type="file">` 时，表单的 `enctype` 必须指定为 `multipart/form-data`，`method` 必须指定为 `post` ，浏览器才能正确编码并以 `multipart/form-data` 格式发送表单的数据。

出于安全考虑，浏览器只允许用户点击 `<input type="file">` 来选择本地文件，用 JavaScript 对 `<input type="file">` 的 `value` 赋值是没有任何效果的。当用户选择了上传某个文件后， JavaScript 也无法获得该文件的真实路径。

```js
var f = document.getElementById('test-file-upload');
var filename = f.value; // 'C:\fakepath\test.png'
if (!filename || !(filename.endsWith('.jpg') || filename.endsWith('.png') || filename.endsWith('.gif'))) {
    alert('Can only upload image file.');
    return false;
}
```

### FILE API

由于JavaScript对用户上传的文件操作非常有限，尤其是无法读取文件内容，使得很多需要操作文件的网页不得不用Flash这样的第三方插件来实现。

随着HTML5的普及，新增的File API允许JavaScript读取文件内容，获得更多的文件信息。

HTML5的File API提供了File和FileReader两个主要对象，可以获得文件信息并读取文件。

下面的例子演示了如何读取用户选取的图片文件，并在一个 `<div>` 中预览图像：

```html
<form method="post" action="http://localhost/test" enctype="multipart/form-data">
    <p>图片预览：</p>
    
    <p>
        <div id="test-image-preview" style="border: 1px solid #ccc; width: 100%; height: 200px; background-size: contain; background-repeat: no-repeat; background-position: center center;"></div>
    </p>
    
    <p>
        <input type="file" id="test-image-file" name="test">
    </p>
    
    <p id="test-file-info"></p>
</form>
```

```js
var
    fileInput = document.getElementById('test-image-file'),
    info = document.getElementById('test-file-info'),
    preview = document.getElementById('test-image-preview');

    // 监听change事件:
    fileInput.addEventListener('change', function () {
        // 清除背景图片:
        preview.style.backgroundImage = '';
        // 检查文件是否选择:
        if (!fileInput.value) {
            info.innerHTML = '没有选择文件';
            return;
        }
        // 获取File引用:
        var file = fileInput.files[0];
        // 获取File信息:
        info.innerHTML = '文件: ' + file.name + '<br>' +
                        '大小: ' + file.size + '<br>' +
                        '修改: ' + file.lastModifiedDate;
        if (file.type !== 'image/jpeg' && file.type !== 'image/png' && file.type !== 'image/gif') {
            alert('不是有效的图片文件!');
            return;
        }
        // 读取文件:
        var reader = new FileReader();
        reader.onload = function(e) {
            var
                data = e.target.result; // 'data:image/jpeg;base64,/9j/4AAQSk...(base64编码)...'            
            preview.style.backgroundImage = 'url(' + data + ')';
        };
        // 以DataURL的形式读取文件:
        reader.readAsDataURL(file);
    });
```


## AJAX

AJAX 并不是 JavaScript 的规范，它是 Asynchronous JavaScript and XML 的缩写，意思就是用 JavaScript 执行异步网络请求。

如果仔细观察一个Form的提交，你就会发现，一旦用户点击“Submit”按钮，表单开始提交，浏览器就会刷新页面，然后在新页面里告诉你操作是成功了还是失败了。如果不幸由于网络太慢或者其他原因，就会得到一个404页面。

这就是Web的运作原理：一次HTTP请求对应一个页面。

如果要让用户留在当前页面中，同时发出新的HTTP请求，就必须用JavaScript发送这个新请求，接收到数据后，再用JavaScript更新页面，这样一来，用户就感觉自己仍然停留在当前页面，但是数据却可以不断地更新。

在现代浏览器上写 AJAX 主要依靠 `XMLHttpRequest` 对象：

```js
`use strict`

function success(text) {
    var textarea = document.getElementById('test-response-text');
    textarea.value = text;
}

function fail(code) {
    var textarea = document.getElementById('test-response-text');
    textarea.value = 'Error code: ' + code;
}

var request = new XMLHttpRequest(); // 新建XMLHttpRequest对象

request.onreadystatechange = function () { // 状态发生变化时，函数被回调
    if (request.readyState === 4) { // 成功完成
        // 判断响应结果:
        if (request.status === 200) {
            // 成功，通过responseText拿到响应的文本:
            return success(request.responseText);
        } else {
            // 失败，根据响应码判断失败原因:
            return fail(request.status);
        }
    } else {
        // HTTP请求还在继续...
    }
}

// 发送请求:
request.open('GET', '/api/categories');
request.send();

alert('请求已发送，请等待响应...');
```

### 安全限制

上面代码中的 URL 使用的是相对路径，如果你把它改成 `http://www.sina.com.cn/` 的话，就会报错，这是因为浏览器的同源策略导致的，默认情况下，JS 在发送 AJAX 请求时，URL 的域名必须和当前页面完全一致。

完全一致的意思是说，域名、端口、协议 都需要完全一样才行。

那如果你一定要请求外域 (其他网站) 的 URL 怎么办呢？有下面这些方法可以考虑：
- 使用 Flash 来发送 HTTP 请求，这种做法可以绕过浏览器的安全限制，但是必须安装 Flash, 用起来比较麻烦；
- 在同源域名下架设一个代理服务器来转发，JS 先把请求发送到代理服务器，由代理服务器请求完成后返回给浏览器，这时候 url 可以改为：`'/proxy?url=http://www.sina.com.cn'`
- 这种方式称为 JSONP，这种方式的原理在于浏览器允许跨域引用 JavaScript 资源，你可以在 JS 中动态增加一个 `<script>` 节点，让浏览器帮你去请求这个外域的 JS 资源，这样一来就完成了外域的访问：

```js
function refreshPrice(data) {
    var p = document.getElementById('test-jsonp');
    p.innerHTML = '当前价格：' +
        data['0000001'].name +': ' + 
        data['0000001'].price + '；' +
        data['1399001'].name + ': ' +
        data['1399001'].price;
}

function getPrice() {
    var
        js = document.createElement('script'),
        head = document.getElementsByTagName('head')[0];
    js.src = 'http://api.money.126.net/data/feed/0000001,1399001?callback=refreshPrice';
    head.appendChild(js);
}
```

### CORS

如果浏览器支持 HTML5，那么可以通过 CORS(Cross-Origin Resource Sharing) 机制来一劳永逸地解决跨域访问问题。

Origin 表示本域，也就是浏览器当前页面的域，当 JS 向外域发起请求，浏览器收到请求后，首先检查 `Access-Control-Allow-Origin` 是否包含本域，如果是则此次跨域请求成功，如果不是，JavaScript 将无法获取到响应的任何数据：

![]( {{ site.url }}/asset/javascript-learning-cors.png )

假设本域是 my.com, 外域是 sina.com 只要响应头 `Access-Control-Allow-Origin` 为 `http://my.com`, 或者是 `*`, 本次请求就可以成功。

可见，跨域能否成功，取决于对方服务器是否愿意给你设置一个正确的 `Access-Control-Allow-Origin`, 决定权始终在对方手中。

需要深入了解 CORS 请参考 [W3C文档](http://www.w3.org/TR/cors/)。


## Promise

在 JavaScript 的世界中，所有的代码都是在单线程中运行的。

JS 引擎提供了一套异步框架，可以让网络操作等异步执行，我们在 JS 中发起异步操作，然后用回调函数接收异步处理的结果。AJAX 就是典型的异步操作，以上一节的代码为例：

```js
request.onreadystatechange = function () {
    if (request.readyState === 4) {
        if (request.status === 200) {
            return success(request.responseText);
        } else {
            return fail(request.status);
        }
    }
}
```

把回调函数 `success`, `fail` 写到一个 AJAX 操作里是很常见的做法，但是不太好看。如果能把回调跟请求分开写，这样会好看一些。

ES6 中对 Promise 做出了规范，使得我们可以写出更漂亮的异步请求代码。

我们先写一个异步函数：

```js
function test(resolve, reject) {
    var timeOut = Math.random() * 2;
    log('set timeout to: ' + timeOut + ' seconds.');
    setTimeout(function () {
        if (timeOut < 1) {
            log('call resolve()...');
            resolve('200 OK');
        }
        else {
            log('call reject()...');
            reject('timeout in ' + timeOut + ' seconds.');
        }
    }, timeOut * 1000);
}
```

如果 test 执行成功，将调用 `resolve()` 函数，如果失败将执行 `reject()` 函数，现在看看怎么利用 Promise 来调用 test：

```js
var p1 = new Promise(test);
var p2 = p1.then(function (result) {
    console.log('成功：' + result);
});
var p3 = p2.catch(function (reason) {
    console.log('失败：' + reason);
});
```

当 `resolve` 被调用后，将自动执行 `p1.then` 里面的代码，当 `reject` 被调用后，将自动执行 `p2.cache` 里面的代码。

Promise 对象可以串联起来，所以上述代码可以简化为：

```js
new Promise(test).then(function (result) {
    console.log('成功：' + result);
}).catch(function (reason) {
    console.log('失败：' + reason);
});
```

Promise 最大的好处在于分离了 执行代码 和 处理结果的代码：

![]( {{ site.url }}/asset/javascript-learning-promise.png )

Promise 还可以做到串行执行一系列任务，先做任务 1，如果成功再做任务 2，任何任务失败则执行错误处理函数：

```js
job1.then(job2).then(job3).catch(handleError);
```

Promise 还可以做到并行执行异步任务：

```js
var p1 = new Promise(function (resolve, reject) {
    setTimeout(resolve, 500, 'P1');
});
var p2 = new Promise(function (resolve, reject) {
    setTimeout(resolve, 600, 'P2');
});
// 同时执行p1和p2，并在它们都完成后执行then:
Promise.all([p1, p2]).then(function (results) {
    console.log(results); // 获得一个Array: ['P1', 'P2']
});
```

有时候并行执行是为了容错，这个时候我们只需要得到一个任务的返回结果即可，先得到哪个就用哪个，这个时候可以用 `promise.race()`：

```js
var p1 = new Promise(function (resolve, reject) {
    setTimeout(resolve, 500, 'P1');
});
var p2 = new Promise(function (resolve, reject) {
    setTimeout(resolve, 600, 'P2');
});
Promise.race([p1, p2]).then(function (result) {
    console.log(result); // 'P1'
});
```


## Canvas

Canvas 是 HTML5 新增的组件，它就像一块幕布，可以用 JavaScript 在上面绘制各种图表、动画等。

```html
<canvas id="test-canvas" width="300" height="200"></canvas>
```

`getContext('2d')` 方法让我们拿到一个 `CanvasRenderingContext2D` 对象，所有的绘图操作都需要通过这个对象完成。

如果要绘制 3D 效果，则需要使用 `gl = canvas.getContext("webgl");` 来获取一个 webgl 对象。

要在 Canvas 中进行绘制，首先需要知道 Canvas 的坐标系统：

![]( {{ site.url }}/asset/javascript-learning-canvas.png )

接着，我们可以使用获取到的 `CanvasRenderingContext2D` 对象来在 Canvas 上绘制图形：

```js
'use strict';

var
    canvas = document.getElementById('test-shape-canvas'),
    ctx = canvas.getContext('2d');

ctx.clearRect(0, 0, 200, 200); // 擦除(0,0)位置大小为200x200的矩形，擦除的意思是把该区域变为透明
ctx.fillStyle = '#dddddd'; // 设置颜色
ctx.fillRect(10, 10, 130, 130); // 把(10,10)位置大小为130x130的矩形涂色
// 利用Path绘制复杂路径:
var path=new Path2D();
path.arc(75, 75, 50, 0, Math.PI*2, true);
path.moveTo(110,75);
path.arc(75, 75, 35, 0, Math.PI, false);
path.moveTo(65, 65);
path.arc(60, 65, 5, 0, Math.PI*2, true);
path.moveTo(95, 65);
path.arc(90, 65, 5, 0, Math.PI*2, true);
ctx.strokeStyle = '#0000ff';
ctx.stroke(path);
```

我们还可以在 Canvas 上绘制文本：

```js
'use strict';

var
    canvas = document.getElementById('test-text-canvas'),
    ctx = canvas.getContext('2d');

ctx.clearRect(0, 0, canvas.width, canvas.height);
ctx.shadowOffsetX = 2;
ctx.shadowOffsetY = 2;
ctx.shadowBlur = 2;
ctx.shadowColor = '#666666';
ctx.font = '24px Arial';
ctx.fillStyle = '#333333';
ctx.fillText('带阴影的文字', 20, 40);
```

Canvas 除了能绘制基本的形状和文本，还可以实现动画、缩放、各种滤镜和像素转换等高级操作。如果要实现非常复杂的操作，可以考虑以下优化方案：
- 通过创建一个不可见的Canvas来绘图，然后将最终绘制结果复制到页面的可见Canvas中；
- 尽量使用整数坐标而不是浮点数；
- 可以创建多个重叠的Canvas绘制不同的层，而不是在一个Canvas中绘制非常复杂的图；
- 背景图片如果不变可以直接用 `<img>` 标签并放到最底层。