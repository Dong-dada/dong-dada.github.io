---
layout: post
title:  "Chromium 多进程架构"
date:   2017-09-06 11:40:30 +0800
categories: chromium
---

 
 


本文参考自 [官方文档](http://www.chromium.org/developers/design-documents/multi-process-architecture)


## 架构总览

Chromium 的最大特点是多进程架构，这一点在许多文章里也提到了。它把负责渲染页面的渲染引擎放到单独的进程执行，渲染进程将页面渲染完毕后，再把渲染后的结果返回给主进程，由主进程将渲染结果展示出来。

主进程被称为 browser process, 渲染进程被称为 renderer process.

browser 和 renderer 之间是一对多的关系，然而不一定一个 renderer 就对应于一个网页 (tab)，因为 [同源策略](https://developer.mozilla.org/zh-CN/docs/Web/Security/Same-origin_policy) 的存在，一个 renderer 有可能对应多个同源的网页。

多进程架构至少有两个好处：
- 安全： renderer 进程启动后，完成一些必要的初始化操作之后，就会进入沙箱 (sandbox) 模式，这种模式下的 renderer 进程无法进行文件操作，无法访问窗口，无法调起子进程。这样就能防御来自 web 的一些攻击行为，即使 renderer 被注入了，也无法对系统造成破坏；
- 稳定： 某个 renderer 进程崩溃后，不会影响 browser 进程，不会影响其它 renderer 进程；

多进程架构的架构图如下：

![]( {{site.url}}/asset/chromium-architecture.png )

上图中有几个需要注意的地方：
- 每个 renderer 里都有一个 `RenderProcess` 全局对象，同时 browser 里有多个 `RenderProcessHost` 对象(与每个 renderer 的 `RenderProcess` 对象一一对应)。这两个对象之间通过 [Chromium's IPC System](http://www.chromium.org/developers/design-documents/inter-process-communication) 进行通信。
- renderer 里的每个 tab 都对应于一个 `RenderView` 对象，该对象由 `RenderProcess` 来管理；同样的，在 browser 来看，每个 tab 都对应与一个 `RenderViewHost` 对象，由 `RenderProcessHost` 来管理；
- renderer 里会用 view ID 来标识每个 `RenderView`, 这个 ID 在单个 renderer 进程里唯一，但在 browser 进程看来是不唯一的(不同 renderer 的不同 view 可能有相同的 view ID)，因此在 browser 里标识一个 view, 需要有 `RenderProcessHost` 和 view ID 两个东西；
- 在 browser 里访问页面内容可以通过 `RenderViewHost` 对象来完成，它会通过对应的 `RenderProcessHost` 与 renderer 进行通信；


## 组件和接口

renderer process 有以下组件：
- `RenderProcess`: 负责与 browser process 的 `RenderProcessHost` 进行 IPC 通信，每个 renderer process 只有一个 `RenderProcess` 全局对象，而 browser process 中对于每个 `RenderProcess` 都有一个对应的 `RenderProcessHost` 对象；
- `RenderView`: 负责与 browser process 中的 `RenderViewHost` 进行通信，代表了 renderer 中的一个页面；

browser process 有以下组件：
- `Browser`: 代表一个 top-level browser window (顶层浏览器窗口)，chromium 浏览器可以有多个 top-level 窗口，每个窗口中又有多个页面，每个 `Browser` 对象代表一个 top-level 窗口；
- `RenderProcessHost`: 负责与某个 renderer 的 `RenderProcess` 进行通信；每个 `RenderProcessHost` 对应一个 `RenderProcess`；
- `RenderViewHost`: 负责与 renderer 中的 `RenderView` 进行通信;


## 共享 render process

通常情况下，每打开一个新的页面就会创建一个新的 renderer process, 然而有的时候 tab 或者 window 必须共享同一个 renderer process, 比如在 js 里通过 `window.open` 创建打开一个新页面的时候，当前页面需要同步地访问新页面的一些属性，这时候新开的页面就需要与原来的页面在同一个 renderer process 里面；

具体的共享 render process 的策略可以参考 [Process Models](http://www.chromium.org/developers/design-documents/process-models)


## 监视 renderer process 的崩溃和无响应情况

browser process 会监视 renderer process 的进程句柄，从而判断 renderer 是否崩溃，如果崩溃，界面上会展示一个出错页面，用户可以点击重载按钮来尝试重新载入这个页面；
