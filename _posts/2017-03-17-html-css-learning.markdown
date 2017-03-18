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

CSS 用于对 html 中的标签进行修饰。例如下面的代码：

{% highlight html %}
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
{% endhighlight %}

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

{% highlight HTML %}
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
{% endhighlight %}

{% highlight css %}
/* style.css 文件 */
span{
   color:red;  
}
{% endhighlight %}

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

{% highlight HTML %}
<span class="address">挪威国布宜诺斯艾利斯市</span>
{% endhighlight %}

设置了类之后，就可以在 css 代码中为 address 这个类来设定样式了：

{% highlight CSS %}
.address {
    color:red;
    font-size:14px;
}
{% endhighlight %}

如此一来，所有设置了 address 类的元素都会被设为红色；

#### ID 选择器

ID 选择器设置的是某个 id 的效果，它的写法与 类选择器 很像，不同之处在于 id 在整个 HTML 文件中是唯一的，而 class 则可以设定到多个标签上；

{% highlight HTML %}
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
{% endhighlight %}

写法上的不同之处在于标签的属性是 id 而不是 class, 同时 css 里选择器的格式是 `#选择器` 而不是 `.选择器`；

类选择器 和 ID 选择器还有一个不同在于，你可以为同一个标签设置多个类，但不能设置多个 id:

{% highlight HTML %}
.stress{
    color:red;
}
.bigsize{
    font-size:25px;
}
<p>到了<span class="stress bigsize">三年级</span>下学期时，我们班上了一节公开课...</p>
{% endhighlight %}

上面代码中为 `<span>` 标签设置了两个类 stress, bigsize;

#### 子选择器

选择指定标签下的第一代子元素；

语法形如 `.food>li{border:1px solid red;}`， `.food>li` 将 food 类下所有的 第一代 li 子元素设置为红色边框；

#### 包含选择器

选择指定标签下的所有子元素(不只第一代)；

语法形如 `.food li{border:1px solid red;}`, `.food li` 将 food 类下所有 li 元素设置为红色边框；

与子选择器的不同之处在于使用空格而非 `>` 符号来选择；

{% highlight HTML %}
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
{% endhighlight %}

在上述例子中，如果使用的是子选择器，则之后 food 类下第一代子元素 `<li>水果</li>` 和 `<li>蔬菜</li>` 会被选择，而使用后代选择器则将选择 food 类下所有的 `li` 元素；

#### 通用选择器

通用选择器的功能非常粗暴，它直接选择所有元素：

语法形如 `* { color:red; }`;

#### 伪类选择符

这种选择符可以为标签的某种状态设置样式，比如 `a:hover { color:red; font-size:20px; }` 会设置 `<a>` 标签 hover 时超链接的样式；

目前浏览器兼容最通用的是 `a:hover` 这种伪类选择器，像 `p:hover` 原理上也是可以的，但浏览器支持并不完善；

#### 分组选择符

你可以把多个选择器合在一起写：

{% highlight CSS %}
h1,span{color:red;}
{% endhighlight %}

上述代码等价于：

{% highlight CSS %}
h1{color:red;}
span{color:red;}
{% endhighlight %}

### CSS 的继承、层叠、和特殊性

#### CSS 的继承

CSS 的某些样式是有继承性的，这里继承的意思是说这个样式不仅会应用到选择的元素上，还会应用到它的子元素上。比如下面的代码中，CSS 的颜色样式不仅会应用到 p 标签，它的子标签 span 也会应用：

{% highlight CSS %}
p {
    color:red;
    border: 1px solid green;
}

<p>哈哈哈哈<span>呵呵呵呵</span></p>
{% endhighlight %}

但有些 CSS 样式并不具有继承性，例如上述例子中的 border 属性，不会被 span 标签所继承。

#### CSS 的权值

如果我们为同一个标签设置了不同的 CSS 样式代码。这种时候会采用哪一个呢？

{% highlight CSS %}
p { color:red; }
.first { color:green; }

<p class="first">哈哈哈哈<span>呵呵呵呵</span></p>
{% endhighlight %}

答案是会采用 `.first` 样式，这是因为 CSS 样式代码是有权值的，同时出现的情况下会使用权值高的那一个。

权值的规则为：标签的权值为 1, 类选择器的权值为 10, ID 选择器的权值为 100. 例如下面的代码：

{% highlight CSS %}
p {color:red;} /* 权值为 1 */
p span{color:green;} /* 权值为 1+1=2 */
.warning{color:blue;} /* 权值为 10 */
p span.warning{color:white;} /* 权值为 1+1+10=12 */
#footer .note p{color:black} /* 权值为 100+10+1=111 */
{% endhighlight %}

可以看到使用包含选择器时权值需要相加。

#### CSS 的层叠

如果有相同权值的 CSS 作用到同一个元素上，这时候会采用哪一个呢？

{% highlight CSS %}
p {color:red;}
p {color:green;}
{% endhighlight %}

答案是会采用后一个，层叠的意思就是说当有相同权值的 CSS 作用在同一个元素上时，会采用后一个覆盖前一个的样式，也就是说，会采用最后一个 CSS 样式来装饰元素。

这也就解释了之前所说的不同 CSS 优先级的顺序为什么是 内联式>嵌入式>外部式；其实只是后面的属性覆盖了前面而已；

#### !important 标识符

有的时候我们希望某个 CSS 样式无法被其他人覆盖，也不会因为权值或者用户的设置而被修改。这时候可以使用 `!important` 标识符来修饰这个样式：

{% highlight HTML %}
p {color:red!important;}
p.first {color:green;}

<p class="first">哈哈哈啊</p>
{% endhighlight %}

这时候虽然 p.first 的样式权值比较高，并且位于 p 样式的后面，但仍然会使用红色样式，因为这个样式被指定为 important.

另外还有一点需要说明，用户可以自己修改网页的样式，比如调大字号等，一般情况下 **用户自己设置的样式>网页中指定的样式>浏览器默认的样式**，但 important 样式比这些级别都要高。

### CSS 格式化排版

CSS 中用于文字排版的属性有：
- font-family : 字体；
- font-size : 字号，以 px 为单位；
- color : 颜色，可以使用 red,green 等默认颜色，也可以使用 #666 这样的色值；
- font-weight : 字体粗细，可以设置为 normal, bold, bolder, lighter
- font-style : 可以设置 normal, italic(斜体), oblique(倾斜);
- text-decoration : 可以设置文本上的线，包括 none, underline(下划线), overline(上划线), line-through(删除线), blink(文本闪烁)；
- text-indent : 设置文字行首的空白，例如 `p { text-indent:2em; }` 表示每个段落行首都留两个字的空白，其中 2em 表示两个字的宽度；
- line-height : 行间距，例如 `p {line-height:1.5em; }` 表示段落中每行之间的距离是 1.5 个字的高度；
- letter-spacing : 每个字母或中文字之间的距离，例如 `p {letter-spacing:50px;}`；
- word-spacing : 每个英文单词之间的距离，例如 `p {word-spacing:50px;}`；
- text-align : 设定 **块状元素** 中文本、图片的对齐形式，可以为 left, center, right, 例如 `h1 {text-aligh:center;}` 将使标题居中；块状元素的定义将在之后讲解。

