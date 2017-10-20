---
layout: post
title:  "Chrome 扩展开发"
date:   2017-10-19 11:32:30 +0800
categories: chromium
---

* TOC
{:toc}

本文参考自 [官方文档](https://developer.chrome.com/extensions)


## 入门

chrome extension 是啥就不介绍了。这一章介绍一个简单扩展的开发过程，建立起扩展开发中的一些基本概念。

先看下这个简单扩展最后的效果：

![]( {{site.url}}/asset/chrome_extension_getstarted_example.png )

这个示例的入口在地址栏右侧，点击图标后弹一个框，在框里可以选择一个颜色，选择了之后就会修改当前页面的背景颜色。

### manifest.json

要开发上面那个扩展，先创建一个文件夹，比如 `extension_test`，用来存储我们的扩展代码。

首先你得创建一个 `manifest.json` 文件，也就是扩展的清单文件，其中包含了扩展名、描述、版本号之类的东西，也包含了扩展程序的入口、权限信息。

按照如下内容编写 `manifest.json` ：

```json
{
    "manifest_version": 2,
    "name": "Getting started example",
    "description": "This extension allows the user to change the background color of the current page.",
    "version": "1.0",

    "browser_action": {
        "default_icon": "icon.png",
        "default_popup": "popup.html",
        "default_title": "click here!"
    },

    "permissions": [
        "activeTab",
        "storage"
    ]
}
```

`browser_action` 字段指定了扩展的入口信息。`permissions` 指定了扩展所需要的权限。

### browser_action

扩展在 chrome 上的入口——也就是右上角的那个小图标，有两种表现形式，一种是 `browser_action`, 表示这个扩展针对所有网页适用；另一种是 `page_action`, 表示这个扩展只在某些网页下显示；

`default_icon` 字段是入口图标，图标文件也要放在咱们的扩展文件夹 `extension_test` 里。

`default_popup` 字段是点击入口图标之后弹出框里要加载的 html 页面。

### 实现扩展的界面和功能

入口配置好了之后，接下来编写 `popup.html`:

```html
<!DOCTYPE html>
<html>
<head>
    <script src="popup.js"></script>
</head>

<body>
    <h1>Background Color Changer</h1>
    <div id="container">
        <span>Choose a color</span>
        <select id="dropdown">
            <option selected="" disabled="" hidden="" value=""></option>
            <option value="white">White</option>
            <option value="pink">Pink</option>
            <option value="green">Green</option>
            <option value="yellow">Yellow</option>
        </select>
    </div>
</body>
</html>
```

接着编写 `popup.js`, 它的代码可以在 [这里](https://developer.chrome.com/extensions/examples/tutorials/getstarted/popup.js) 下载。逻辑非常简单，就是通过 `chrome.tabs.executeScript()` 接口来设置当前网页的背景颜色。

### 加载扩展

从 chrome web store 下载的扩展是打包后的 `.crx` 文件，如果在开发的时候，每次修改都打包的话就太麻烦了，所以 chrome 提供了直接加载扩展文件夹的方式：

首先打开 `chrome://extensions`, 勾选上 "开发者模式"，然后点击 "加载已解压的扩展程序"，然后选择我们的 `extension_test` 目录就可以了 :

![]( {{site.url}}/asset/chrome-extension-how-to-load.png )

如果你修改了扩展的相关代码，需要点击 "重新加载"， 或者刷新整个扩展页面。


## 概览

Extension 本质上是一个 web 页面，它们可以使用 [chrome 提供的一些 API](https://developer.chrome.com/extensions/api_other)。

Extension 可以通过 [content script](https://developer.chrome.com/extensions/content_scripts) 或者 [cross-origin XMLHttpRequests](https://developer.chrome.com/extensions/xhr) 来与 web 页面或者服务器进行交互。

Extension 也可以与浏览器的一些功能进行交互，比如 [书签](https://developer.chrome.com/extensions/bookmarks) 和 [tab](https://developer.chrome.com/extensions/tabs)。

### Extension 的界面

大部分的 Extension 都通过 `browser actions` 或者 `page actions` 来提供访问入口。每个 Extension 最多只能有一个 `browser action` 或者一个 `page action`.

Extension 也可以用其它方式来展现界面，比如增加一个菜单项、提供一个设置页面、通过 `content script` 来修改页面的样式。[这个页面](https://developer.chrome.com/extensions/devguide) 介绍了所有 extension 功能的列表。

### 文件

每个 Extension 都包含如下文件：
- 一个 manifest 文件；
- 一个或多个 HTML 文件(除非该扩展是一个 theme)
- 可选： 一个或多个 JavaScript 文件；
- 可选： 任何你需要的文件，比如图片；

要引用你的文件，一般可以通过相对路径来完成：

```html
<img src="images/myimage.png" />
```

### ExtensionID

如果你在 chrome debugger 里查看文件的路径，你会发现这些文件有它们自己的绝对路径：

```
chrome-extension://<extensionID>/<pathToFile>
```

在这个 URL 中， *extensionID* 表示的是系统为每个扩展生成的唯一 ID。

如果你没有把扩展打包，那么 extensionID 可能会变。具体说来，是在你改变扩展加载路径的时候，或者打包扩展的时候。

如果你需要在扩展内指定某个文件的全路径，那么可以通过 `@@extension_id` 这个 [预定义消息](https://developer.chrome.com/extensions/i18n#overview-predefined) 来完成。

一旦你发布了这个扩展(把它上传到 dashboard 上)，那么他的 ID 就变成永久的了，即使更新了扩展，这个 ID 也不会变。完事儿之后你就可以把所有 `@@extension_id` 替换成真实的 ID 了。

### 架构

大部分扩展都有一个 `background page`, 它不是用来做背景的，而是不可见的一个页面，用来存放扩展的主逻辑。

如果扩展需要与用户加载的 web 页面进行交互——而不是扩展自带的页面，那么需要使用 `content script`.

#### background page

Background pages 由 `background.html` 来定义，该文件里可以包含一些 JavaScript 代码。

有两种类型的 Background pages: [persistent background pages](https://developer.chrome.com/extensions/background_pages) 和 [event pages](https://developer.chrome.com/extensions/event_pages). Persistent background pages, 意为永久背景页，它将始终保持打开； Event pages 则根据需要来打开或关闭。显然后者更节省资源，是比较推荐的方式。

#### UI pages

扩展可以包含一些 HTML 页面来展示界面。例如， `browser action` 可以有一个 popup 页面； 扩展还可以有一个 设置页面，用户可以自定义扩展的工作方式； 另外还可以有 [override page](https://developer.chrome.com/extensions/override), 它可以覆盖在用户页面上； 还可以通过 `tabs.create()` 或 `window.open()` 以标签的形式展示扩展内部的页面；

扩展内部的 HTML 页面可以相互访问对方的 DOM，也可以相互调用对方的 JS 函数。

#### Content scripts

Content scripts 用于与浏览器中的 web 页面进行交互。它将在 web 页面的上下文中被执行，你可以认为 Content scripts 是 web 页面的一部分，而不是扩展的一部分。

Content scripts 可以读取页面浏览的细节，也可以修改页面。在下图中， content scripts 可以读取和修改 web 页面的 DOM, 然而它不能修改扩展的 background page (因为它不在 background page 的上下文中运行)：

![]( {{site.url}}/asset/chrome-extension-content-script.gif )

Content scripts 只能通过消息的方式与 background page 进行通信，比如它可以在找到 RSS feed 的时候向 background page 发送一条消息； background page 也可以发消息告诉 Content scripts 修改 web 页面的内容：

![]( {{site.url}}/asset/chrome-extension-content-script-send-message.gif )

### 使用 chrome.* APIs

Extension 除了可以使用 web page 本身已有的 API 以外，还可以使用 chrome 提供的专用 API (都是 `chrome.*` 格式)。

例如，你可以通过 `window.open()` 标准 API 来打开一个页面，但如果你需要指定页面打开的位置，那么可以使用专用 API `tabs.create()`.

大部分 chrome API 都是异步的——它们会立刻返回，如果你需要知道函数的执行结果，就得传入一个 callback:

```
chrome.tabs.create(object createProperties, function callback);
```

还有一些 chrome API 是同步的，这些 API 可以直接得到结果：

```
string chrome.runtime.getURL()
```

下面的例子展示了异步 API 的使用方法，它先获取当前选中的 tab ID (使用异步 API `tabs.query()`), 然后将该 tab 重新导航到一个新的网址：

```js
chrome.tabs.query({'active': true}, function(tabs) {
    chrome.tabs.update(tabs[0].id, {url: newUrl});
    });

someOtherFunction();
```

### 页面之间的通信

由于 extension 中的所有页面都处于同一进程的同一线程，因此他们可以直接相互调用。

要找到 extension 中的其他页面，可以使用 [chrome.extension](https://developer.chrome.com/extensions/extension) API，比如 `getView()` 和 `getBackgroundPage()`. 一旦一个页面引用了另一个页面，它就可以调用其他页面的 JS 函数，或者修改它的 DOM。

### 保存数据和隐私模式

Extension 可以使用 [storage](https://developer.chrome.com/extensions/storage) API，以及 [web storage API](http://dev.w3.org/html5/webstorage/) (例如 `localStorage`) 来存储数据。

一旦你试图存储数据，首先考虑下当前是不是正处于隐私模式 (incognito mode), 默认情况下， extension 是不允许在隐私模式下运行的，如果用户在隐私模式下开启了你的 extension, 你也不应该收集用户的数据。

要检查当前是否处于隐私模式，可以检查 `tabs.Tab` 的 `incognito` 属性，例如：

```js
function saveTabData(tab, data) {
  if (tab.incognito) {
    chrome.runtime.getBackgroundPage(function(bgPage) {
      bgPage[tab.url] = data;      // Persist data ONLY in memory
    });
  } else {
    localStorage[tab.url] = data;  // OK to store data
  }
}
```


## 总结

剩下的内容就不介绍了，直接上手做东西，有问题去官网查一下就好啦。
