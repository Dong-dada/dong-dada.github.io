---
layout: post
title:  "Chromium 多进程资源加载"
date:   2017-09-12 17:44:30 +0800
categories: chromium
---

 
 

本文参考自 [官方文档](http://www.chromium.org/developers/design-documents/multi-process-resource-loading) 以及 [这篇文章](http://blog.csdn.net/yl02520/article/details/21285745)


## 背景

Chromium采用多进程架构，本文主要介绍浏览器进程和渲染进程，由于渲染进程是运行在沙盒中，没有直接访问系统资源的能力（如网络资源）。渲染进程中的WebKit/Blink在渲染网页时，首先需要从网络上加载相应的资源文件，比如HTML文档以及文档中的图片、JS脚本、CSS脚本等子资源文件，那么渲染进程就需要先把这些请求通过IPC（进程间通信）的方式发送给浏览器进程，由浏览器进程统一管理和网络数据传输。

这样做有一下几个好处：
1. 浏览器可以统一管理所有的资源请求；
2. 浏览器进程可以缓存已经下载完毕的资源文件和Cookie，当有新的请求到达时，如果请求的资源以及存在于缓存中，就直接从缓存中提取，没必须要重新从网络上下载，这样可以节省网络带宽和资源下载的时间。
3. 自从HTTP/1.1之后，浏览器要求不能为同一域名的主机打开太多的连接，因为如果一个标签页打开太多的Socket链接，可能会导致系统资源消耗过多，这可能会影响到其他标签页或者其他应用程序的运行，这在资源受限的移动平台上显得尤为重要。Chromium目前允许同一主机最多打开6个Socket链接。


## 概览

![]( {{site.url}}/asset/chromium-resource-loading.png )

chromium 的多进程架构可以看作有三层：
- 最底层是用于渲染页面的 Blink 层；
- 中间是 render process (简单地理解为每个 tab 对应一个 render process), 每个 renderer process 都包含了一个 Blink 实例；
- 最上层的 browser process 管理所有的 render process, 并且控制着所有网络访问；


## Blink

Blink 中有一个 `ResourceLoader` 对象来负责拉取数据，每个 Loader 都有一个 `WebURLLoader` 来执行实际的请求，这些相关的头文件包含在 Blink 仓库里；

`ResourceLoader` 实现了 `WebURLLoaderClient` 接口, renderer 会通过这个接口中的回调函数来将请求到的数据和其它事件分发给 Blink.


## Renderer

renderer 实现了 `WebURLLoader`, 名为 `WebURLLoaderImpl`, 代码位于 `content/child/` 中。请求发起前，它会使用全局的单例 `ResourceDispatcher` 来创建一个唯一的 request ID, 然后把这个 ID 通过 IPC 传给 browser. browser 给的回复里包含了这个 request ID, 收到回复后，resource dispatcher 可以通过这个 ID 来得到一个 `ResourcePeer` 对象。


## Browser

browser 内部的 `RenderProcessHost` 接收到来自 renderer 的 IPC 请求。它会把这些请求转发给全局的 `ResourceDispatcherHost`, 它们通过一个指向 render process host 的指针以及一个 request ID 来唯一标识一个请求。

每个请求都会被转化为 `URLRequest` 对象，它内部的 `URLRequestJob` 会实现指定协议的请求操作。在 `URLRequest` 产生通知后，他会把通知发送给正确的 `RenderProcessHost` 对象，后者会把通知发送给 renderer.


## Cookies

所有的 Cookie 都是由 `/net/base/` 目录下的 `CookieMonster` 来处理的，chromium 不会把 Cookie 共享给其它浏览器的网络栈(比如 WinINET). `CookieMonster` 对象运行于 browser 进程中，因为 Cookie 需要被多个 tab 所共享。

页面可以通过 `document.cookie` 来请求 cookie, 这种情况下 renderer 会发送一个同步的 IPC 请求给 browser. 在 browser 处理这个请求的过程中, Blink 是被阻塞的，直到 renderer 的 I/O 线程接收到 browser 的处理结果，它才会被唤醒，并把请求结果传回给 JavaScript 引擎。
