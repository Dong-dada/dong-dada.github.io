---
layout: post
title:  "Chromium 插件架构"
date:   2017-09-13 11:56:30 +0800
categories: chromium
---

 
 

本文参考自 [官方文档](http://www.chromium.org/developers/design-documents/plugin-architecture)

chromium 里的插件 (plugin) 和扩展 (extension) 是两个不同的概念，这里介绍的是插件而非扩展；


## 概览

插件是导致浏览器不稳定的主要问题来源。另外，把插件引入到 renderer 进程会导致沙箱机制无法启用，因为插件往往需要访问操作系统的资源，开启沙箱的话插件就无法正常运行了。因此 chromium 把插件放到了独立的进程中去运行。


## 细节设计

### 进程中插件

chromium 既可以让插件在进程内，也可以让插件在进程外运行。这两种方式都依赖于 chromium 所实现的多进程无关的 WebKit 内嵌层，可以通过实现这个内嵌层的 `WebKit::WebPlugin` 接口来完成插件的引入。

`WebKit::WebPlugin` 接口由 `WebPluginImpl` 来实现，它通过一个 `WebPluginDelegate` 接口来与外界对话。对于进程内插件来说，该接口由 `WebPluginDelegateImpl` 来实现。后者会与 NPAPI wrapper layer 来对话：

![]( {{site.url}}/asset/chromium-plugin-architecture-in-process.png )

### 进程外插件

进程外插件的结构请看下图。

chromium 通过切换下图点线 (dotted line) 以上的部分来支持进程外插件。从下图可以看出，在 `WebPluginImpl` 层和 `WebPluginDelegateImpl` 层之间插入了一个 IPC 层，这使得 NPAPI 代码可以在两种模式下共享，旧的 `WebPluginDelegateImpl` 和 NPAPI 的代码现在都可以在独立的插件进程中执行。

在 plugin/renderer 之间的通信 channel 由 `PluginChannel` 和 `PluginChannelHost` 来实现。chromium 中有许多 renderer 进程(由打开的页面决定)，而每**种** plugin 对应一个 plugin 进程(比如 Flash 插件可能有多个实例，每个实例对应某个 renderer, 这些实例都运行在同一个 plugin 进程里。而所有 Windows Media Player 插件的实例都运行在另一个 plugin 进程中)。 

这意味着，在 renderer 进程中，需要为每种插件都创建一个 `PluginChannelHost` 实例，每个实例与某个 plugin 进程通信。而在 plugin 进程中，每个使用了该插件的 renderer 都对应有一个 `PluginChannel` 实例。这个结论很好理解，因为一个 channel 只能负责一对一的进程通信，如果一个 renderer 使用了两种 plugin, 那么它需要与两个 plugin 进程通信，它就会为每个 plugin 进程创建一个 `PluginChannelHost` 实例；而对于 plugin 来说，如果它需要为多个 renderer 服务，那么它也需要创建多个 `PluginChannel`。

一个页面中可以运行同一个插件的多个实例，比如在一个网页中嵌入了两个 Flash 视频，那么在 renderer 这一端会有两个 `WebPluginDelegateProxies`, 在 Flash plugin process 这一端则会有两个 `WebPluginDelegateStubs` 及相关设施。

对于 `PluginChannel` 来说，它需要在一个 IPC 连接上完成多个对象的通信(把 renderer 到 plugin 的一对一进程间通信，变为两个进程中多个对象的一对一通信)。

![]( {{site.url}}/asset/chromium-plugin-architecture-out-process.png )

### windowless plugin

windowless plugin 用于直接在 rendering pipeline(渲染管线)中运行。当 WebKit 需要绘制一个涉及到插件的区域的时候，它会调用插件的代码，传给它一个 drawing context(绘图上下文) 由插件来完成绘制。这种 windowless plugin 一般用在插件希望能够有一部分透明地叠加在页面上的情形 (我觉得就是异形窗口)，这需要由插件自己来决定如何合成画面。

剩下的看不懂了，这里就不写了。。。

### 整体示意图

下图展示了有着两个 render process, 一个 flash plugin process 的整体示意图。plugin process 里运行了三个 flash 实例。注意下图中的 `WebPluginStub` 在实际代码中已经被合并到了 `WebPluginDelegateProxy` 里面。

![]( {{site.url}}/asset/chromium-plugin-architecture-overall-system.png )
