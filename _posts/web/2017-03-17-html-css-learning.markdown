---
layout: post
title:  "HTML-CSS 基础知识学习"
date:   2017-03-17 10:05:30 +0800
categories: web
---

 
 

## 基本概念

网页使用 HTML 对页面元素进行布局，使用 CSS 对页面元素进行装饰：

```html
<!DOCTYPE HTML>
<html>
    <head>
        <meta http-equiv="Content-Type" content="text/html; charset=utf-8">
        <title>Html和CSS的关系</title>
        <style type="text/css">
        h1{
            font-size:12px;
            color:#930;
            text-align:center;
        }
        </style>
    </head>
    <body>
        <h1>Hello World!</h1>
    </body>
</html>
```


## HTML 基础知识

### HTML 文件的基本结构

HTML 文件必须有一个 html 根标签。

一般而言 html 根标签下会有两个标签 head, body. 
- head 标签不会显示到页面上，它用来对网页进行一些配置；
- body 标签下存放了页面的内容，也是 html 文件的主题；

head 和 body 标签下还会有子标签，例如 head 标签下的 title 标签表示网页的标题，将会显示在浏览器 tab 头上，而 body 标签下的 h1 标签表示一个一级标题。

### HTML 常用标签

- `<p>` 标签：标签内的内容表示一个段落；
- `<hx>` 标签：其中的 x 标示 1-6 之间的数字，用来表示一个标题，例如 h1 表示一级标题，在页面上显示的最大, h6 则为最小；
- `<em>` 和 `<strong>` 标签： 表示强调，一般而言 `<em>` 标签中的内容将会以斜体展示，`<strong>` 标签中的内容将会以粗体标示；
- `<q>` 标签：表示引用，在某些浏览器上面，`<q>` 中包裹的内容将会被加上双引号；
- `<blockquote>` 标签：表示长引用，如果想要引用一大段文字，可以使用这个标签；
- `<br/>` 标签：表示分行，它是一个空标签，不需要写成 `<br></br>` 的形式，只需要 `<br/>` 就可以将一行文字分成两行。要注意：**html 中会忽略空格和回车**，因此必须使用 `<br/>` 标签来达到分行的目的；
- `&nbsp;` : 这不是一个标签，它的作用是输入空格，前面说了 html 会忽略空格，如果遇到需要输入空格的场合，可以使用 `&nbsp;` ；
- `<hr/>` 标签：表示分割线，它是一个空标签，不要写成 `<hr></hr>` 的形式；

- `<address>` 标签：表示地址；

- `<code>` 标签：表示一段代码，它只能修饰一行代码，不能修饰多行代码；
- `<pre>` 标签：用来修饰多行代码，修饰的内容中，**空格和换行都会保留**；

- `ul-li` 标签：用来表示无序列表，列表的每一项前面都没有序号，只有一个点；
- `ol-li` 标签：用来表示有序列表，列表的每一项前面都有按照顺序的序号；列表的示例代码如下：
```html
<ul>
    <li>第一列</li>
    <li>第二列</li>
</ul>
```

- `<table>` 标签：表示一个表格，需要和 `<tr>`, `<th>`, `<td>`, `<tbody>` 这几个标签来配合使用。其中：
    - `<tr>` 表示一行 (table row)；
    - `<th>` 表示表格的头 (table head)，需要写在 `<tr>` 里面：`<tr> <th>姓名</th> <th>学号</th> <th>班级</th> </tr>`
    - `<td>` 表示一个单元格 (table data cell)，需要写在 `<tr>` 里面：`<tr> <td>董哒哒</td> <td>233333</td> <td>233</td> </tr>`
    - `<tbody>` 当表格内容非常多时，表格会下载一点显示一点，但如果加上<tbody>标签后，这个表格就要等表格内容全部下载完才会显示；这个标签需要加在 `<table>` 标签里面，然后包裹其他标签，形如：`<table> <tbody> ... </tbody> </table>` ；
    - `<caption>` 标签写在 `<table>` 的最开始处，表示这个表格的标题；
    - `<table>` 标签中可以通过 summary 属性添加摘要，形如 `<table summary="这个表格用来记录班级人员的信息">`, summary 中的内容不会显示到页面上，它的作用是增加表格的可读性；

- `<div>` 标签：用来划分独立的逻辑区域，不会显示到页面上。你可以为它指定不同的 id, 从而区分不同的 div : `<div id="task-list">...</div>` ；

- `<a>` 标签：表示超链接，例如 `<a href="http://www.baidu.com" title="百度网站" target="_blank">点击这里</a>` 其中 href 表示点击后要跳转的链接，title 表示鼠标 hover 时弹出的 tips, target 表示打开超链接的方式，`_blank` 表示在新窗口中打开，`<a>` 标签包裹的内容是点击的文字；

- `<img>` 标签：表示图片，例如 `<img src="myimage.gif" alt="my image" title="My Image" />` 其中 src 表示图像的位置; alt 表示图像的描述，当图片下载失败时，可以看到该属性指定的文本；title 表示图片可见时鼠标 hover 显示的 tips; 

- `<span>` 标签：这个标签 **没有语义**, 它仅用来占位，以后可以用 css 来专门修改 span 中包括的内容；

- `<form>` 标签：表单标签，表单可以把用户输入的数据发送到服务器端，这样服务器端程序就可以处理表单传过来的数据，且看下面的例子：
```html
<form method="post" action="save.php">
    <label for="username">用户名：</label>
    <input type="text" name="username" />
    <label for="pass">密码：</label>
    <input type="password" name="pass"/>
    <input type="submit" value="确定"  name="submit" />
    <input type="reset" value="重置" name="reset" />
</form>
```

- `<form>` 标签中可以包含如下表单控件：
    - method 属性表示数据传送的方式 get/post;
    - action 属性表示用户输入的数据要传送到什么地方，这里是一个 php 页面；
    - `<label>`, `<input>` 等标签是表单控件，必须写到 `<form>` 标签里面，否则可能提交数据失败；
    - `<input>` 标签中如果设定 type 为 text, 则显示一个编辑框，如果设定 type 为 password, 则显示一个密码编辑框，你还可以在 `<input>` 中输入默认值：`<input type="text" value="董哒哒">`，标签中的 name 属性是编辑框的名字，可以给服务端程序使用，以分辨数据的来源；
    - `<input>` 标签中如果设定 type 为 radio 则显示单选框，如果设定 type 为 checkbox, 则显示一个复选框。这时候可使用 `checked` 属性来表示是否选中它，另外，对于单选框而言，你需要把同一组 radio 设定为相同的 name, 这样才能实现单选效果；对于单选框和复选框而言, value 属性表示要提交到服务器的数据；
    - `<input>` 标签中如果设定 type 为 submit, 则显示一个提交按钮，这个按钮可以将表单数据提交到服务器上；
    - `<input>` 标签中如果设定 type 为 reset, 则显示一个重置按钮，这个按钮可以将表单恢复到初始状态；
    - `<label>` 标签是静态文本框，它有一个 for 属性，其中指定的是另一个表单控件的 id, 如果指定了 for 属性，那么当点击 label 的时候，将同时作用到另外一个控件上，例如 `<label for="man">男</label> <input type="radio" id="man" name="gender">`, 这时点击 label, 则后面的单选框也会被选中；
    - `<textarea>` 也是一个表单控件，可以输入大段文本，形如：`<textarea col="50" row="10">在这里输入内容</textarea>`，其中 col, row 可以用 width,height 代替；
    - `<select>` 也是一个表单控件，表示下拉列表框，可以容纳多个选项，每个选项用 `<option>` 标签表示，形如： `<select multiple="multiple"> <option value="看书">看书</option> <option value="旅游">旅游</option> <option value="运动">运动</option> </select>` 其中 multiple 属性表示多选，如果设定了这个属性，则 window 下可以用 Ctrl + mouse click 的方式来多选项目；


## CSS 基础知识

CSS 用于对 html 中的标签进行修饰。例如下面的代码：

```html
<!DOCTYPE HTML>
<html>
    <head>
        <meta http-equiv="Content-Type" content="text/html; charset=utf-8">
        <title>CSS样式的优势</title>
        <style type="text/css">
            span{
            font-size:20px; /*设置文字字号*/
            font-weight:bold; /*设置为粗体*/
            color:blue; /*设置文字为蓝色*/
            }
        </style>
    </head>
    <body>
        <p>慕课网，<span>超酷的互联网</span>、IT技术免费学习平台，创新的网络一站式学习、实践体验；<span>服务及时贴心</span>，内容专业、<span>有趣易学</span>。专注服务互联网工程师快速成为技术高手！</p>
    </body>
</html>
```

上述 CSS 代码将导致该 html 中所有 `<span>` 标签包裹的内容都变为蓝色，字号设置为 20，并加粗文字。

### CSS 代码的结构

![CSS 代码结构]({{ site.url }}/asset/52fde5c30001b0fe03030117.jpg)

**选择符：** 指网页中要应用样式规则的元素，上图中将会把所有 `<p>` 元素中的文字设定为蓝色。

**声明：** 在 `{}` 之间的内容就是声明，多条声明之间用分号 `;` 分隔。可以用 `/* */` 的方式为声明填写注释。

CSS 代码不一定要写在 `<head>` 中，也可以写在别的地方：
- 内联式：把 css 代码直接写在标签中，形如： `<p style="color:red; font-size:12px">这是一个段落</p>` ；
- 嵌入式：也就是本章开头所介绍的将 CSS 代码写在 `<head>` 标签下的做法；
- 外部式：写在单独的一个文件中：

以下是外部式写法的一个例子：

```html
<!DOCTYPE HTML>
<html>
    <head>
        <meta http-equiv="Content-Type" content="text/html; charset=utf-8">
        <title>外部式css样式</title>
        <!-- 指定 css 文件 -->
        <link href="style.css" rel="stylesheet" type="text/css" />
    </head>
    <body>
        <p>慕课网，<span>超酷的互联网</span>、IT技术免费学习平台，创新的网络一站式学习、实践体验；<span>服务及时贴心</span>，内容专业、<span>有趣易学</span>。专注服务互联网工程师快速成为技术高手！</p>
    </body>
</html>
```

```css
/* style.css 文件 */
span{
   color:red;  
}
```

注意 `<link href="style.css" rel="stylesheet" type="text/css" />` 这行 HTML 代码，其中 href 属性指定了 css 文件的位置，rel 和 type 属性是固定写法。

这里有一个问题，如果我们在一个 HTML 里同时使用了这三种写法来修饰相同的元素，会按照那种写法为准？答案是根据优先级决定，默认情况下 **内联式 > 嵌入式 > 外部式**；

之所以说是默认情况，主要有两种意外：
- 如果外部式的 `<link>` 标签存在于嵌入式的 `<style>` 标签后面，那么 `<link>` 的优先级会大于 `<style>`，不过实际工程中一般都会先写 `<link>`；
- CSS 有权值的概念，这会在之后介绍；

### 选择器

之前我们介绍的选择器只是标签选择器，它的优势在于可以一次性设置所有同类型标签的样式，其实选择器不只它一种：
- 标签选择器：设置所有同类型标签的样式；
- 类选择器：设置指定类标签的样式；
- ID 选择器：设置指定 id 标签的样式，id 在整个 HTML 中是唯一的，但 class 可以指定到多个标签上；
- 子选择器：可以选择指定标签的 **第一代** 子元素；
- 包含选择器：可以选择指定标签的 **所有** 子元素；
- 通用选择器：可以选择所有标签；

#### 类选择器

类选择器设置的是某个类的效果，这些类需要提前设置到标签上面：

```html
<span class="address">挪威国布宜诺斯艾利斯市</span>
```

设置了类之后，就可以在 css 代码中为 address 这个类来设定样式了：

```css
.address {
    color:red;
    font-size:14px;
}
```

如此一来，所有设置了 address 类的元素都会被设为红色；

#### ID 选择器

ID 选择器设置的是某个 id 的效果，它的写法与 类选择器 很像，不同之处在于 id 在整个 HTML 文件中是唯一的，而 class 则可以设定到多个标签上；

```html
<!DOCTYPE HTML>
<html>
    <head>
        <meta http-equiv="Content-Type" content="text/html; charset=utf-8">
        <title>认识html标签</title>
        <style type="text/css">
        #stress{
            color:red;
        }
        </style>
    </head>
    <body>
        <h1>勇气</h1>
        <p>三年级时，我还是一个<span id="stress">胆小如鼠</span>的小女孩，上课从来不敢回答老师提出的问题，生怕回答错了老师会批评我。就一直没有这个勇气来回答老师提出的问题。学校举办的活动我也没勇气参加。</p>
    </body>
</html>
```

写法上的不同之处在于标签的属性是 id 而不是 class, 同时 css 里选择器的格式是 `#选择器` 而不是 `.选择器`；

类选择器 和 ID 选择器还有一个不同在于，你可以为同一个标签设置多个类，但不能设置多个 id:

```html
.stress{
    color:red;
}
.bigsize{
    font-size:25px;
}
<p>到了<span class="stress bigsize">三年级</span>下学期时，我们班上了一节公开课...</p>
```

上面代码中为 `<span>` 标签设置了两个类 stress, bigsize;

#### 子选择器

选择指定标签下的第一代子元素；

语法形如 `.food>li{border:1px solid red;}`， `.food>li` 将 food 类下所有的 第一代 li 子元素设置为红色边框；

#### 包含选择器

选择指定标签下的所有子元素(不只第一代)；

语法形如 `.food li{border:1px solid red;}`, `.food li` 将 food 类下所有 li 元素设置为红色边框；

与子选择器的不同之处在于使用空格而非 `>` 符号来选择；

```html
<!DOCTYPE HTML>
<html>
    <head>
        <meta http-equiv="Content-Type" content="text/html; charset=utf-8">
        <title>后代选择器</title>
        <style type="text/css">
            .food li{
                border:1px solid red;/*添加边框样式（粗细为1px， 颜色为红色的实线）*/	
            }
        </style>
    </head>
    <body>
        <ul class="food">
            <li>水果
                <ul>
                    <li>香蕉</li>
                    <li>苹果</li>
                    <li>梨</li>
                </ul>
            </li>
            <li>蔬菜
                <ul>
                    <li>白菜</li>
                    <li>油菜</li>
                    <li>卷心菜</li>
                </ul>
            </li>
        </ul>
    </body>
</html>
```

在上述例子中，如果使用的是子选择器，则之后 food 类下第一代子元素 `<li>水果</li>` 和 `<li>蔬菜</li>` 会被选择，而使用后代选择器则将选择 food 类下所有的 `li` 元素；

#### 通用选择器

通用选择器的功能非常粗暴，它直接选择所有元素：

语法形如 `* { color:red; }`;

#### 伪类选择符

这种选择符可以为标签的某种状态设置样式，比如 `a:hover { color:red; font-size:20px; }` 会设置 `<a>` 标签 hover 时超链接的样式；

目前浏览器兼容最通用的是 `a:hover` 这种伪类选择器，像 `p:hover` 原理上也是可以的，但浏览器支持并不完善；

#### 分组选择符

你可以把多个选择器合在一起写：

```css
h1,span{color:red;}
```

上述代码等价于：

```css
h1{color:red;}
span{color:red;}
```

### CSS 的继承、层叠、和特殊性

#### CSS 的继承

CSS 的某些样式是有继承性的，这里继承的意思是说这个样式不仅会应用到选择的元素上，还会应用到它的子元素上。比如下面的代码中，CSS 的颜色样式不仅会应用到 p 标签，它的子标签 span 也会应用：

```css
p {
    color:red;
    border: 1px solid green;
}

<p>哈哈哈哈<span>呵呵呵呵</span></p>
```

但有些 CSS 样式并不具有继承性，例如上述例子中的 border 属性，不会被 span 标签所继承。

#### CSS 的权值

如果我们为同一个标签设置了不同的 CSS 样式代码。这种时候会采用哪一个呢？

```css
p { color:red; }
.first { color:green; }

<p class="first">哈哈哈哈<span>呵呵呵呵</span></p>
```

答案是会采用 `.first` 样式，这是因为 CSS 样式代码是有权值的，同时出现的情况下会使用权值高的那一个。

权值的规则为：标签的权值为 1, 类选择器的权值为 10, ID 选择器的权值为 100. 例如下面的代码：

```css
p {color:red;} /* 权值为 1 */
p span{color:green;} /* 权值为 1+1=2 */
.warning{color:blue;} /* 权值为 10 */
p span.warning{color:white;} /* 权值为 1+1+10=12 */
#footer .note p{color:black} /* 权值为 100+10+1=111 */
```

可以看到使用包含选择器时权值需要相加。

#### CSS 的层叠

如果有相同权值的 CSS 作用到同一个元素上，这时候会采用哪一个呢？

```css
p {color:red;}
p {color:green;}
```

答案是会采用后一个，层叠的意思就是说当有相同权值的 CSS 作用在同一个元素上时，会采用后一个覆盖前一个的样式，也就是说，会采用最后一个 CSS 样式来装饰元素。

这也就解释了之前所说的不同 CSS 优先级的顺序为什么是 内联式>嵌入式>外部式；其实只是后面的属性覆盖了前面而已；

#### !important 标识符

有的时候我们希望某个 CSS 样式无法被其他人覆盖，也不会因为权值或者用户的设置而被修改。这时候可以使用 `!important` 标识符来修饰这个样式：

```html
p {color:red!important;}
p.first {color:green;}

<p class="first">哈哈哈啊</p>
```

这时候虽然 p.first 的样式权值比较高，并且位于 p 样式的后面，但仍然会使用红色样式，因为这个样式被指定为 important.

另外还有一点需要说明，用户可以自己修改网页的样式，比如调大字号等，一般情况下 **用户自己设置的样式>网页中指定的样式>浏览器默认的样式**，但 important 样式比这些级别都要高。

### CSS 格式化排版

CSS 中用于文字排版的属性有：
- font-family : 字体；
- font-size : 字号，以 px 为单位；
- color : 颜色，可以使用 red,green 等默认颜色，也可以使用 #666 这样的色值；
- font-weight : 字体粗细，可以设置为 normal, bold, border, lighter
- font-style : 可以设置 normal, italic(斜体), oblique(倾斜);
- text-decoration : 可以设置文本上的线，包括 none, underline(下划线), overline(上划线), line-through(删除线), blink(文本闪烁)；
- text-indent : 设置文字行首的空白，例如 `p { text-indent:2em; }` 表示每个段落行首都留两个字的空白，其中 2em 表示两个字的宽度；
- line-height : 行间距，例如 `p {line-height:1.5em; }` 表示段落中每行之间的距离是 1.5 个字的高度；
- letter-spacing : 每个字母或中文字之间的距离，例如 `p {letter-spacing:50px;}`；
- word-spacing : 每个英文单词之间的距离，例如 `p {word-spacing:50px;}`；
- text-align : 设定 **块状元素** 中文本、图片的对齐形式，可以为 left, center, right, 例如 `h1 {text-aligh:center;}` 将使标题居中；块状元素的定义将在之后讲解。

### CSS 盒模型

#### HTML 元素分类

在 HTML 中，元素大致分为三种类型：
- 块状元素，包括： `<div>, <p>, <h1>...<h6>, <ol>, <ul>, <dl>, <table>, <address>, <blockquote>, <form>`
- 内联元素(又叫行内元素)，包括：`<a>, <span>, <br>, <i>, <em>, <strong>, <label>, <q>, <var>, <cite>, <code>`
- 内联块状元素，包括：`<img>, <input>`

块状元素的特点：
1. 每个块级元素都从新的一行开始，并且其后的元素也另起一行，换句话说，块级元素独占一行；
2. 元素的宽、高、行高、顶和底边距都可以设置；
3. 元素宽度在不设置的情况下，是它父容器宽度的 100%；

注意：**可以用 CSS 将内联元素设置为块元素** 形如：`a {display:block;}`;

内联元素的特点：
1. 和其它元素都在同一行上；
2. 元素的高度、宽度以及顶部和底部边距不可以设置；
3. 元素的宽度就是它包含的文字或图片的宽度，不可改变；

注意：**可以用 CSS 将块元素设置为内联元素** 形如： `div {display:inline;}`;

内联块状元素的特点：
1. 和其它元素都在同一行上；
2. 元素的高度、宽度、行高、以及顶和底边距都可以设置；

注意：**可以用 CSs 将块元素或内联元素设置为内联块状元素** 形如： `div {display:inline-block;}`;

#### 盒模型

盒模型也就是 margin, pandding, border 这套玩意儿，这里就不细说了，很容易理解的；

需要注意的是盒模型只适用于 块级元素，例如 div,p,h1,table,form 这些；

下面逐个介绍盒模型的每个属性可以应用哪些 CSS 样式：

border:
- border-width: 边框宽度，以 px 为单位；
- border-style: 样式，常见的有 dashed(虚线), dotted(点线), solid(实线)；
- border-color: 颜色，可以是 red,green 等内置颜色，也可以是 16 进制颜色值例如 #888；

这些样式可以简化为形如 `div {border:2px solid #123;}` 的形式，等价于 `div {border-width:2px; border-style:solid; border-color:#123;}`

CSS 还允许只设置某条边的样式，形如 `div {border-top:1px solid red;}`

注意：CSS 里有宽高的属性 width,height, **这两个属性是不包括 margin,pandding,border 的，它表示的是内容物的宽度**。一个元素所占的尺寸，需要通过 width/height + margin, border, padding 来计算得出。

pandding:
- pandding 只能应用边框宽度这一个属性，同 border 一样，它也可以各自定义每一条边的宽度，并且它可以简写为 `div {pandding:10px 20px 30px 40px;}`，需要注意的 **边的顺序并不是 ltrb 而是 trbl 是从 t 开始顺时针排列的**。

margin:
- margin 只能应用边框宽度这一个属性，它也可以定义每一条边的宽度，并且它可以简写为 `div {margin:10px 20px 30px 40px;}`;

### CSS 布局模型

在网页中，有三种布局模型：
1. flow 流动模型；
2. float 浮动模型；
3. layer 层模型；

下面逐个介绍这三种布局模型：

flow:
- 流动模型是默认的模型，这种模型非常简单：每个块状元素都各自独占一行，从上到下排列，每个内联元素都在当前行上，从左到右排列；

float:
- 我们可以使用 CSS 代码将任何元素定义为浮动，形如：`div {float:left;}`, 这一操作会导致 div 不再独占一行，而是浮动起来，让其他元素可以跟在这个 div 后面；
- float 还可以定义为 `float:right` 这样后面的其他元素会跟在这个 div 左侧排列；
- float 属性可以实现文本环绕图片的效果；

layer:
- 层模型有三种形式：
    - position:absolute 绝对定位。这种方式会把对应的元素从文档流中拖出来，它的位置不再是从上到下或从左到右，而是一个相对于父元素的位置，同时由于它已经脱离了原来的文档流，就相当于它不存在了一样，它随后的元素会替代它之前的位置；
    - position:relative 相对定位。这种方式 **不会** 把对应的元素从文档流中拖出来，它的新位置是相对于原位置的偏移，但原位置上还残留了一个虚像，它随后的元素会跟着它之前的位置进行从上到下或从左到右的排列。
    - position:fixed 固定定位。这种方式与 absolute 方式类似，它也会把对应的元素从文档流中拖出来，所不同的是，**它的位置是相对于整个页面窗口的**，即使滚动条滚动，它的位置也不会发生改变。

下面我们看一下 layer 模型下三种形式的具体代码和效果：

下面是绝对定位的效果：
![absolute 绝对定位的效果]({{ site.url }}/asset/css-layer-absolute.png)

下面是相对定位的效果：
![relative 相对定位的效果]({{ site.url }}/asset/css-layer-relative.png)

下面是固定定位的效果：
![fixed 固定定位的效果]({{ site.url }}/asset/css-layer-fixed.png)

另外，上述几种形式是可以混用的，从而实现复杂的页面效果。但需要注意的是, absolute 默认情况下是相对于 HTML 最外层 body 的位置，如果你希望它能够相对于其他元素来设定位置，那么你需要把父元素设定上 `position:relative` 样式：

```html
<div id="box1" style="position:relative; border:1px solid red; width:100px; height:100px" >
    <div id="box2" style="position:absolute; left:10px; top:20px; border:1px solid green; width:100px; height:100px"></div>
</div>
```

这样就可以实现 box2 相对于父元素 box1 的左上角偏移一定距离的效果了。

需要注意的是，如果元素有多个层级，那么子元素在这是 absolute 位置时，所对应的参照物，是它前辈元素中最近的那个具有 relative 样式的元素。例如下面的代码：

```html
<div id="box1" style="position:relative;">
    <div id="box2">
        <div id="box3" style="position:absolute;"></div>
    </div>
</div>
```

由于 box2 没有设置 relative 样式，因此 box3 的参照物是设置了 relative 的 box1.


### CSS 中的单位和值

#### 颜色值

CSS 中设置颜色值有如下几种方法：
- 内置颜色：形如 `p{color:red;}`;
- RGB 颜色：形如 `p{color:rgb(123,233,100);}` 其中的数字是 0~255，也可以是百分数，形如：`p{color:rgb(1%, 50%, 40%);}`；
- 十六进制颜色：形如 `p{color:#00ffff;}`

下面是颜色表，可以方便地选取颜色：

![RGB 颜色表]({{ site.url }}/asset/css-color-table.jpg)

#### 长度值

CSS 中设置元素或文字长度常用的单位有以下几种：
- px: 像素，CSS 假设 90 像素 = 1 英寸；
- em: 代表了本元素的 font-size 值，比如 `p {font-size:12px; text-indent:2em; }` 设定了 font-size 是 12px, 那么 1em=12px. 如果你直接把 em 设定到了 font-size 上，那么 em 是以父元素的 font-size 为准的；
- 百分比: 这个比较难懂，不确定到底是父元素的百分比，还是本元素的百分比；

### CSS 中的居中

居中又分为水品居中和垂直居中。

#### 水平居中

水平居中是一个很常见的需求，这里我们需要分两种情况来讨论。一种是行内元素，另一种是块状元素。

对于行内元素(a, img) 而言，由于它自身的宽高都是固定的，所以你需要对他的父元素设定 `text-align:center` 样式来达到本身居中的效果：

```html
<div style="text-align:center">
    <a href="http://www.baidu.com">点这里</a>
</div>
```

对于块状元素(div, table) 而言，有以下方法可以做到水平居中：

比较简单的一种方法是给块状元素设定一个固定的长度，然后设置它的水平方向上的 margin 为 auto, 形如：`<div style="width=200; margin:20px auto;"></div>`。由于水平方向上的 margin 是自动调节的，这时候块状元素会自动居中。这种方法的原理在于给块状元素一个固定的宽度，剩下的 margin 由它自己来适应；

上述方法对于某些宽度不确定的块状元素就不适应了。默认情况下块状元素会占领一整行的宽度，这时候即使设定了 `margin:0 auto` 也不会实现居中效果。我们称这种没有设定具体 width 的块状元素为 不定宽块状元素。

对于不定宽块状元素而言，有以下方法可以实现水平居中效果：
- 我们可以用一个 table 标签来包裹目标标签，然后对 table 标签进行 `margin:0 auto` 设定来完成水平居中的效果。其原理是 table 标签具有长度自适应的特性，它的宽度不会延展为父 body 的宽度，而是与内容的大小保持一致。这样就可以设置 `margin:0 auto` 了，形如 `<table style="margin:0 auto;><tbody><tr><td> <div>设置居中</div> </tbody></table>`。
- 将元素的 `display` 属性修改为 `inline`，接着就可以像行内属性一样为它的父元素设置 `text-align:center` 来实现自身居中的效果了。
- 给父元素设置 `float:left`, 让它悬浮起来，然后设置 `position:relative`, `left:50%` 让父元素相对于原位置偏移到中间；接着子元素设置 `position:relative` 和 `left:-50%` 来实现水平居中；

#### 垂直居中

对于单行文本而言，设置垂直居中非常简单，只需要将父元素的 height 和 `line-height` 属性设为一致就可以了，形如：`<div style="height:100px; line-height:100px;"> <p>哈哈哈哈</p> </div>`

对于多行文本而言，你可以借助 table 的 `vertical-align:middle` 属性来实现，形如：

```
<body>
<table><tbody><tr><td style="height:300px; background:#333;">
<div>
    <p>看我是否可以居中。</p>
    <p>看我是否可以居中。</p>
    <p>看我是否可以居中。</p>
    <p>看我是否可以居中。</p>
</div>
</td></tr></tbody></table>
</body>
```

上述代码实际上是把我们的多行文本当做表格的一个表项 (table-cel) 来实现的，在 IE8 以上可以设置 `display:table-cell` 属性来表示。


## Tips

### CSS 中的代码简写

我们之前介绍盒模型的时候已经说过 margin,border,padding 都可以写成简写形式，书写的顺序是 trbl 也就是顺时针从 t 开始数；需要注意的是，如果 l 和 r 一样，t 和 b 一样，那么可以进一步缩写成只有两个数字，例如 `p p{margin:10px 20px 10px 20px;}` 可以简写成 `p{margin:10px 20px;}`

此外我们一般用 `#123456` 这样的颜色值来表示 RGB 色彩，如果每两位的数字一样，可以简写为一个，例如 `#112233` 可以简写为 `#123` ；

我们之前介绍了许多设置字体的代码，例如：

```css
body{
    font-style:italic;
    font-variant:small-caps; 
    font-weight:bold; 
    font-size:12px; 
    line-height:1.5em; 
    font-family:"宋体",sans-serif;
}
```

上述代码可以简写为：

```css
body{
    font:italic  small-caps  bold  12px/1.5em  "宋体",sans-serif;
}
```

需要注意的是 `font-size` 和 `line-height` 之间要用 `/` 而不是空格来分隔；

上述一些属性我们可以省略，一般中文网站会写为如下形式：

```css
body{
    font:12px/1.5em  "宋体",sans-serif;
}
```




