# 使用 jekyll 和 github pages 建立博客

## jekyll 和 github pages 分别是什么？

### jekyll 是什么？

jekyll 是一个生成静态页面的工具。之所以要有 jekyll, 是因为手写网页太辛苦了。jekyll 可以通过一些模式化的配置，从 markdown 中生成出静态网页。

### github pages 是什么？

github pages 是 github 提供的一项功能，可以让用户发布自己的站点。你可以把你自己编写的网页上传到 github 上，之后访问 http://*username*.github.io 就可以浏览你自己的站点。


## 怎样使用 jekyll 来生成静态页面？

先不管怎么把页面发布到 github 上，我们来看看怎么在本地生成静态网页。

### 安装相关环境

使用 jekyll 之前，你得先安装这些东西：

- git bash;
- ruby, ruby gems

具体安装方法就不说了，网上都有。

### 安装 jekyll 和 bundler

在 git bash 里执行下述命令即可安装 jekyll 和 bundler 这两个东西：

```
gem install jekyll bundler
```

jekyll 和 bundler 都是一种 ruby gem, 其中：
- jekyll 就是 jekyll 本身；
- bundler 用于管理 ruby 中的其他 gems, 它的主要作用是解决不同 gem 之间的依赖和兼容性问题。

jekyll 和 bundler 安装一次就好了，不用每次创建新站点都去安装它。

### 创建一个 jekyll 项目

在任意目录执行下述命令即可创建一个默认的 jekyll 项目：

```
jekyll new myblog
```

其中 myblog 是你站点的名字，你可以随意填写它。

如果要在一个已存在的文件夹里创建 jekyll 项目，可以用 `.` 来表示当前文件夹：

```
cd myblog
jekyll new .
```

创建完毕后，你会在 myblog 里发现 _posts 文件夹和 _config.yml 配置文件：
- _posts : 存储你的 markdown 文件，也就是你的 markdown 文章, jekyll 会根据这些 markdown 文件生成对应的网页；
- _config.yml : 站点配置文件，站点的风格、标题等内容可以在这里设定；

jekyll 默认会在 _posts 文件夹里创建一个 markdown 文件，它就是你默认的第一篇博客。你可以在 _posts 文件夹里添加新的 markdown 文章。添加时需要按照规定的格式：

```
YEAR-MONTH-DAY-title.MARKUP
```

也就是 `年-月-日-标题.markdown` 这样。

### 生成网页

在 myblog 路径下执行下述代码，就能将 markdown 生成为网页：

```
cd myblog
jekyll build
```

生成完毕后，你会在 myblog 里发现 _site 文件夹。这个文件夹里存储了 jekyll 根据 markdown 文件生成的网页。

你可以根据文章的标题在 _site 文件夹里搜索到对应的网页。

### 运行本地服务器，查看网页的效果

要查看网页效果，只需在 myblog 目录下执行下述命令：

```
bundle exec jekyll serve
```

这个命令会运行 jekyll 服务器，你可以在浏览器中导航 http://localhost:4000 来查看效果。

有时运行 `bundle exec jekyll serve` 命令会报如下错误：

```
Permission denied - bind(2) for 127.0.0.1:4000
```

这是因为 jekyll serve 使用的 4000 端口被占用导致的，你可以在 _config.yml 里添加下面配置来更改端口：

```
# 改为其他未被占用的端口
port: 5001
```

当你使用 jekyll serve 命令运行服务时，对 markdown 的修改会即可同步到页面中，你只需要在浏览器中刷新一下页面，就能看到最新的页面效果，而无需再次执行 `jekyll build` 命令。

### jekyll 项目的目录结构

jekyll 本质上是一个 文本转换引擎。它把你提供的文本，例如 markdown, textile, plain HTML 通过一系列 layout 文件组织起来，生成最终的静态网站。

一个 jekyll 站点基本上有如下的目录结构：

```
.
├── _config.yml
├── _data
|   └── members.yml
├── _drafts
|   ├── begin-with-the-crazy-ideas.md
|   └── on-simplicity-in-technology.md
├── _includes
|   ├── footer.html
|   └── header.html
├── _layouts
|   ├── default.html
|   └── post.html
├── _posts
|   ├── 2007-10-29-why-every-programmer-should-play-nethack.md
|   └── 2009-04-26-barcamp-boston-4-roundup.md
├── _sass
|   ├── _base.scss
|   └── _layout.scss
├── _site
├── .jekyll-metadata
└── index.html # can also be an 'index.md' with valid YAML Frontmatter
```

以下是各个 文件/文件夹 的具体作用：
- _config.yml : 站点配置文件；
- _drafts : drafts 表示未发布的 posts. 其中包含了一些 markdown 文件，与 _posts 中的 markdown 文件不同，它的文件名里不包括日期，你可以从 [work with drafts](http://jekyllrb.com/docs/drafts/) 中了解如何使用 drafts ；
- _includes : 包含了一些可以重用的内容，你可以用 `{% include file.ext %}` 这样的标签来导入 `_includes/file.ext` 中的内容；
- _layouts : layout 是包裹在 posts 外部的模板。`_layouts` 文件夹中包含了一系列的 layout, 你可以在 post 的头信息中选择这篇文章所需要使用的 layout. 这会在下一节中介绍；
- _posts : 这里存放的是你的文章。文章的文件名必须是 `YEAR-MONTH-DAY-title.MARKUP` 的格式；
- _data : 
- _sass : 存储了一些 sass 文件，这些文件可以作为素材导入到 `main.sass` 文件中，而 `main.sass` 又会被处理为 `main.css` 文件，从而对站点的 style 进行定义；
- _site : 存放由 jekyll 生成的站点。你最好把它加入到 `.gitignore` 文件里，因为它经常会发生变动；
- .jekyll-metadata : jekyll 自身的一些数据，用来追踪那些文件需要被重新生成。你最好把它加入到 `.gitignore` 文件里；
- index.html or index.md : 站点根目录中的所有 `.markdown`, `.md`, `.html`, `.textile` 文件都会被 jekyll 所处理；
- 其他文件和文件夹 : 例如 css, image 文件夹， favicon.ico 文件，等等，将被拷贝到最终生成的站点里面。你可以参考 [这些站点](http://jekyllrb.com/docs/sites/) 来了解这些文件夹的组织方式；







