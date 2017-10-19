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

这个示例扩展的入口在地址栏右侧，点击图标后弹一个框，在框里可以选择一个颜色，选择了之后就会修改当前页面的背景颜色。

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

### 实现插件的界面和功能

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

### 加载插件

首先打开 `chrome://extensions`, 勾选上 "开发者模式"，然后点击 "加载已解压的扩展程序"，然后选择我们的 `extension_test` 目录就可以了 :

![]( {{site.url}}/asset/chrome-extension-how-to-load.png )

如果你修改了扩展的相关代码，需要点击 "重新加载"， 或者刷新整个扩展页面。
