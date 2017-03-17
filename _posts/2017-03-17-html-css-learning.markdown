---
layout: post
title:  "HTML-CSS 基础知识学习"
date:   2017-03-17 10:05:30 +0800
categories: web
---

## 基本概念

网页使用 HTML 对页面元素进行布局，使用 CSS 对页面元素进行装饰：

{% highlight html %}
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
{% endhighlight %}


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
{% highlight html %}
<ul>
    <li>第一列</li>
    <li>第二列</li>
</ul>
{% endhighlight%}

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
{% highlight html %}
<form method="post" action="save.php">
    <label for="username">用户名：</label>
    <input type="text" name="username" />
    <label for="pass">密码：</label>
    <input type="password" name="pass"/>
    <input type="submit" value="确定"  name="submit" />
    <input type="reset" value="重置" name="reset" />
</form>
{% endhighlight %}

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




