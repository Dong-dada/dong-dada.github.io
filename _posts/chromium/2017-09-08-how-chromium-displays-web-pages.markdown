---
layout: post
title:  "Chromium 如何展示网页"
date:   2017-09-08 17:21:30 +0800
categories: chromium
---

* TOC
{:toc}

本文参考自 [官方文档](https://www.chromium.org/developers/design-documents/displaying-a-web-page-in-chrome)


## 分层

![]( {{site.url}}/asset/chromium-conceptual-application -layers.png )

上图中的底层对高层没有依赖，我们从底层到高层来看：
- **WebKit :** 知名渲染引擎，Safari, Chromium 等 WebKit-based 的浏览器都使用了它，注意其中的 `WebKit Port` 是 `WebKit` 的一部分，其中包含了一些系统相关的服务，比如资源加载和绘图；
- **Glue :** 胶水层，把 `WebKit types` 转换成 `Chromium types`；
- **Renderer/Render host :** Chromium 的 `multi-process embedding layer`, 用来封装进程间通信；
- **WebContents :** Content module 中的一个重要的可重用组件，作为一个可嵌入的组件，用于使用多进程架构来渲染 HTML，具体可参考 [content module pages](https://www.chromium.org/developers/content-module) 来了解；
- **Browsers :** 代表一个浏览器窗口，其中可以包含多个 `WebContentses`；
- **TabHelpers :** 被 attach 到 `WebContentses` 上的独立对象. `Browser` 会把一系列 `Tab Helpers` 附加到它持有的 `WebContentses` 上面；

### WebKit

Chromium 使用开源项目 [WebKit](http://webkit.org/) 来布局 Web 页面。相关代码拉取自 Apple, 存储在 `/third_party/WebKit` 目录下。WebKit 主要由负责页面布局的 `WebCore`, 负责执行 `JavaScript` 代码的 `JavaScriptCore` 组成。其中的 `JavaScriptCore` 被替换为 V8, 另外在 Apple 的 Safari 浏览器中，会有一个叫做 `WebKit` 的层，这个层介于 `WebCore` 和 OSX 应用程序之间，Chromium 中并没有使用这一层，而是把来自于 Apple 的代码都称为 WebKit.

### The WebKit Port

WebCore 是平台独立的代码，其中有一些平台相关的代码用 Port(移植) 的方式来实现，Chromium 自己实现了这些平台相关代码，这些代码存储在 WebKit 目录下，包括 chromium 文件夹，以及一些以 chromium 为后缀的文件，Port 的内容包括：
- 网络传输通过 [multi-process resource loading](https://www.chromium.org/developers/design-documents/multi-process-resource-loading) 机制来实现，而不是直接交由操作系统处理；
- 绘图使用 Skia 绘图库来实现，它是一个跨平台的绘图库，但不包含文本绘制功能。Skia 的相关代码存储在 `/third_party/skia` 文件夹中，主要的入口点代码位于 `/webkit/port/platform/graphics/GraphicsContextSkia.cpp` 文件中，它会引用许多位于 `/base/gfx` 文件中的代码；

### The WebKit Glue

Chromium 使用的代码风格，数据类型跟 WebKit 不一样，因此封装了一个胶水层以便统一使用 Chromium 的风格来访问 WebKit 接口。例如我们使用 `std::string` 而不是 `WebCore::String`, 使用 `GURL` 而不是 `KURL`。胶水层的代码存储在 `/webkit/glue` 文件夹中。胶水层的对象名通常跟 WebKit 中的对象名类似，只是有一个 `Web` 前缀，比如 `WebCore::Frame` 会改名为 `WebFrame`.

胶水层的代码把 WebCore 的数据类型和 Chromium 代码隔离开来，这样可以尽量避免 WebCore 的变化对 chromium 产生影响, WebCore 的数据类型不会直接被 chromium 所使用。

chromium 项目中有一个 `test shell` 程序，它作为一个基本的浏览器专门用来测试 `WebKit port` 和 胶水层代码。它跟 chromium 浏览器一样通过相同的胶水层接口来调用 WebKit, 但它不包含多进程那套复杂的东西，因此也被用来进行 WebKit 内核本身的测试工作。


## The render process

![]( {{site.url}}/asset/chromium-rendering-in-the-renderer.png )

Chromium 的 render process 通过胶水层的接口来调用 WebKit port, 它的主要工作是作为 [IPC](https://www.chromium.org/developers/design-documents/inter-process-communication) channel 的 renderer 端与 browser 端进行通信；

renderer 中最重要的类是 `RenderView`, 代码位于 `/content/renderer/render_view_impl.cc` 中。一个 `RenderView` 对象代表了一个网页。与 browser 进程间的所有与导航有关的命令、消息都由它来处理。它继承自 `RenderWidget`，后者负责处理绘制和输入事件的处理。`RenderView` 通过全局对象 `RenderProcess` 与 browser process 进行通信；

**FAQ: RenderView 与 RenderWidget 之间的区别是什么？** `RenderWidget` 实现了胶水层的一个叫做 `WebWidgetDelegate` 的接口，从而映射到了一个 `WebCore::Widget` 对象上，它代表了屏幕上的一个窗口，用于绘制内容，接受输入事件。`RenderView` 继承自 `RenderWidget`, 除了处理绘图和输入事件外，它还负责处理导航命令。

只有一种情况下 `RenderWidget` 会独自存在——当它用于页面上的 select box 的时候。这些选择框需要弹出一个选项的列表，它必须使用一个本地窗口来进行渲染，这样才能够显示到其它所有内容的上面，甚至超出窗口边界。

### Threads in the renderer

每个 renderer 都有两个线程 (具体可以参考 [Threading and Tasks](https://chromium.googlesource.com/chromium/src/+/master/docs/threading_and_tasks.md)). 主要的对象都运行在 render 线程上，比如 `RenderView` 和 WebKit 代码。当这些对象想要与 browser 进行通信的时候，首先向主线程 (render 线程) 投递一个消息，主线程再把这个消息分发到 browser 进程。此外，它也允许从 renderer 中同步地发送消息给 browser, 因为一些消息必须在得到 browser 的响应结果后才能继续执行。比如 renderer 中的 javascript 代码想要获取页面的 cookie 的时候，renderer 的主线程会阻塞住，直到收到 browser 的正确响应，这个时候发送给 renderer 主线程的所有消息都会被存到队列里，等到线程恢复之后再继续处理。


## The browser process

![]( {{site.url}}/asset/chromium-browser-process.png )

### Low-level browser process objects (browser process 中的底层对象)

所有与 render process 的 [IPC](https://www.chromium.org/developers/design-documents/inter-process-communication) 通信都通过 browser 的 I/O 线程来完成。这个线程也会处理所有的 [network communication](https://www.chromium.org/developers/design-documents/multi-process-resource-loading), 以免干扰界面响应。

界面开始运行后，会在主线程中创建一个 `RenderProcessHost` 对象，随后 `RenderProcessHost` 会创建一个新的 render process 以及一个 `ChannelProxy` 对象，这个 `ChannelProxy` 对象中包含了一个通向 renderer 的命名管道。`ChannelProxy` 对象运行在 I/O 线程中，它监听命名管道的消息，自动把消息转发给 UI 线程中的 `RenderProcessHost` 对象。另外 I/O 线程中还运行了一个 `ResourceMessageFilter` 对象，它会把某些能够直接在 I/O 线程中处理的消息过滤下来，比如 renderer 发来的网络请求消息。这些过滤操作发生在 `ResourceMessageFilter::OnMessageReceived()` 函数中。

`RenderProcessHost` 会把视图相关的所有消息分发到对应的 `RenderViewHost` 上面(它自己也会处理一些视图无关的消息)。这个分发操作发生在 `RenderProcessHost::OnMessageReceived()` 函数中。

### High-level browser process objects (browser process 中的高层对象)

视图相关的消息会进入到 `RenderViewHost::OnMessageReceived()` 函数中，大部分消息都在这个函数里处理，还有一部分消息是有基类 `RenderWidgetHost` 来处理的。`RenderViewHost`, `RenderWidgetHost` 这两个对象与 renderer 中的 `RenderView`, `RenderWidget` 是映射关系。每个平台都有一个视图类(`RenderWidgetHostView[Aura|Gtk|Mac|Win]`) 来实现与本地视图系统的集成；

在 `RenderView/Widget` 的上层是 `WebContents` 对象，大部分的消息实际上会由该对象中的函数来处理，一个 `WebContents` 对象代表了一个网页中的 contents(内容)。`WebContents` 是 content 模型中的顶层对象，负责在一个矩形视图里显示一个网页，你可以通过 [content module pages] 这篇文章来了解详情；

`WebContents` 对象被包含在 `TabContentsWrapper` 中，后者保存在 `chrome/` 文件夹下，代表了一个 tab.


## Illustrative examples (示例说明)

[Getting Around the Chromium Source Code](https://www.chromium.org/developers/how-tos/getting-around-the-chrome-source-code) 这篇文章里有一些示例用于说明导航和启动过程。

### Life of a "set cursor" message

set cursor 是一个典型的由 renderer 发往 browser 的消息，以下是 renderer 中的步骤：
- set cursor 消息由 WebKit 内部产生，用来响应输入事件. 这个消息将在 `RenderWidget::SetCursor()`(代码位于 `content/renderer/render_widget.cc.`) 中发出；
- 它将调用 `RenderWidget::Send()` 函数来分发这个消息，`RenderView` 也会用这个函数来向 browser 发送其它消息，它将调用 `RenderThread::Send()`;
- 这将调用 `IPC::SyncChannel`, 它会产生一个代理消息，然后将这个消息投递给 renderer 主线程, 随后这个消息被发送到命名管道，从而把消息发送给 browser.

以下是 browser 中的步骤：
- `RenderProcessHost`(主线程) 中的 `IPC::ChannelProxy`(I/O 线程) 负责接受 I/O 线程中发来的所有消息。`IPC::ChannelProxy` 首先把消息发给 `ResourceMessageFilter` 来进行过滤，我们的 set cursor 消息不会被过滤掉，因此会分发到 UI 线程中进行处理；
- `RenderProcessHost::OnMessageReceived()` 会接受 render process 中所有 `RenderView` 发来的消息，它会直接处理一部分消息，剩下的消息会根据 `RenderView` 来寻找合适的 `RenderViewHost` 进行处理；
- `RenderViewHost::OnMessageReceived()` 接收到消息，许多消息会直接在这里进行处理，但 set cursor 比较特别(它是 `RenderWidget` 发来的), 它是由 `RenderWidgetHost` 进行处理的；
- 未处理的消息自动被投递给 `RenderWidgetHost` 处理，包括我们的 set cursor 消息；
- `RenderWidgetHost::OnMsgSetCursor` 收到消息，接着调用合适的 UI 函数来设置 cursor.

### Life of a "mouse click" message

mouse click 是一个典型的由 browser 发往 renderer 的消息，以下是 browser 中的步骤：
- `RenderWidgetHostViewWin::OnMouseEvent()` 收到 Windows 消息，接着调用 `ForwardMouseEventToRenderer()`;
- 该函数将鼠标事件打包为一个平台无关的 `WebMouseEvent` 事件，然后将其发送给相关联的 `RenderWidgetHost` 对象；
- `RenderWidgetHost::ForwardInputEvent()` 创建一个 IPC 消息 `ViewMsg_HandleInputEvent`, 将 `WebMouseEvent` 序列化后调用 `RenderWidgetHost::Send()`;
- 调用 `RenderProcessHost::Send()`, 随后消息被交给 `IPC::ChannelProxy`;
- `IPC::ChannelProxy` 将产生一个代理消息，然后将这个代理消息投递给 I/O 线程，接着把这个消息通过匿名管道发送给 renderer;

以下是 renderer 中的步骤：
- renderer 主线程中的 `IPC::Channel` 接收到消息，随后发给 `IPC::ChannelProxy`.
- `RenderView::OnMessageReceived()` 收到消息，许多类消息都直接在这个函数里处理了，但 mouse click 消息会被分发给 `RenderWidget::OnMessageReceived()`, 然后消息被分发给 `RenderWidget::OnHandleInputEvent()`;
- 事件被发给 `WebWidgetImpl::HandleInputEvent()`, 在这个函数里，事件会被转化给一个 `PlatformMouseEvent` 类型的 WebKit 对象，然后交给 WebKit 内部的 `WebCore::Widget` 去处理。


