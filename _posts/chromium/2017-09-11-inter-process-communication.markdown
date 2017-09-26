---
layout: post
title:  "Chromium 的 IPC"
date:   2017-09-11 17:24:30 +0800
categories: chromium
---

 
 

本文参考自 [官方文档](https://www.chromium.org/developers/design-documents/inter-process-communication)


## 概览

chromium 的进程间通信用命名管道来实现，在 Linux & OSX 上，使用的是 `socketpair()`, 命名管道使用的是异步模式，以保证管道的两端都不会因为等待另一端而发生阻塞。

要了解如何编写安全的 IPC endpoints, 请参考 [Security Tips for IPC](https://www.chromium.org/Home/chromium-security/education/security-tips-for-ipc).

### IPC in browser

在 browser 中，与 renderers 的通信都在 I/O 线程中进行，向 views 收发消息必须使用 `ChannelProxy` 对象来代理到 browser 的主线程中。

这种方案的优点在于，所有的资源请求都发生在 I/O 线程中，这样可以避免影响用户界面。收到资源请求消息后，会被 `ChannelProxy::MessageFilter`(`RenderProcessHost` 会把 `ChannelProxy::MessageFilter` 插入到自己的 channel 中) 拦截掉，这个 filter 运行在 I/O 线程中，它拦截到资源请求消息后，会直接把他转给 resource dispatcher host 来处理。阅读 [Multi-process Resource Loading](http://dev.chromium.org/developers/design-documents/multi-process-resource-loading) 来了解资源加载的更多细节。

### IPC in renderer

renderer 的主线程用于处理通信，渲染和大部分操作都发生在其它线程中(可参考 [multi-process architecture](https://www.chromium.org/developers/design-documents/multi-process-architecture) 中的示意图)。大部分 browser 发给 WebKit 的消息都会通过 main renderer thread 完成，反之亦然。主线程还支持 render-to-browser 的同步消息。


## 消息

### 消息的类型

chromium 的消息分为两种：`routed`, `control`.

control 消息由创建管道的类本身来处理，而 `routed` 消息则由关心这个消息的其它对象来处理，这个路由操作是通过 `MessageRouter` 来完成的。

比如在进行渲染的时候，control 消息是指那些没有指定 view, 需要由 `RenderProcess` 或 `RenderProcessHost` 去处理的消息。比如请求资源、修改剪贴板，这些请求跟具体的 view 无关，因此是 control 消息。而要求某个 view 绘制某区域的请求则是 `routed` 消息；

消息的类型与这个消息是由 renderer 发给 browser, 或是 browser 发给 renderer 无关。

通常我们把 browser 发给 renderer 的与 frame 相关的消息称作 Frame 消息，因为它最终被发送给了 `RenderFrame`，类似的，renderer 发给 browser `RenderFrameHost` 的消息被称为 FrameHost 消息。你可以在 [frame_messages.h](https://code.google.com/p/chromium/codesearch#chromium/src/content/common/frame_messages.h) 里看到消息都被分为两个部分；

Plugins 也有独立的进程，与 renderer 类似，它也有 PluginProcess 消息和 PluginProcessHost 消息，这些消息都定义在 [plugin_process_messages.h](https://code.google.com/p/chromium/codesearch#chromium/src/content/common/plugin_process_messages.h) 里面；

browser 和 renderer 之间的其它消息都用类似的方式组织，例如 View 和 ViewHost 消息，它们定义在 [view_messages.h](https://code.google.com/p/chromium/codesearch#chromium/src/content/common/view_messages.h) 文件中；

### 声明消息

chromium 提供了一些宏来声明消息，要声明一个 renderer to browser 的 routed 消息，并且指定该消息包含一个 URL 和一个 integer 作为参数，可以用如下写法：

```c
IPC_MESSAGE_ROUTED2(FrameHostMsg_MyMessage, GURL, int)
```

要声明一个 browser to renderer 的 control 消息，消息中不带参数，可以用如下写法：

```c
IPC_MESSAGE_CONTROL0(FrameMsg_MyMessage)
```

### 消息参数的序列化和反序列化

消息的参数在跨进程传递的时候需要进行序列化和反序列化，`ParamTraits` 模版负责完成这件事情。大部分通用类型的模板特例化代码存储在 [ipc_message_utils.h](https://cs.chromium.org/chromium/src/ipc/ipc_message_utils.h) 文件中。如果你需要跨进程传递自己定义的类型，那么你需要为这个类型定义自己的 特例化 `ParamTraits` 模版；

`ParamTraits` 只能接受一个参数，如果一个消息附带了多个参数，则需要定义一个单独的结构体来存储这些参数。例如，对于 `FrameMsg_Navigate` 消息，使用 `CommonNavigateParams` 结构来存储参数，[frame_message.h](https://code.google.com/p/chromium/codesearch/#chromium/src/content/common/frame_messages.h) 文件中使用 [IPC_STRUCT_TRAITS](https://code.google.com/p/chromium/codesearch/#chromium/src/ipc/param_traits_macros.h) 系列宏定义了该结构的 `ParamTraits` 模版特例化；

### 发送消息

发送消息需要通过 channel 来完成。在 browser 中，`RenderProcessHost` 包含有一个可以用来从 UI 线程向 renderer 发送消息的 channel. `RenderWidgetHost` 提供了一个 `Send` 函数方便调用，它实际上还是通过 `RenderProcessHost` 来使用 channel 发送消息的。

`Send()` 函数的参数是一个 `IPC::Message*` 类型的指针，IPC 层接分发完这个消息后会自动去 delete 它，因此你只需要 new 一个消息传给 `Send()` 函数就好了：

```cpp
Send(new ViewMsg_StopFinding(routing_id_));
```

注意，如果需要让消息被路由到正确的 View/ViewHost 上面，你必须指定一个正确的 routing ID. `RenderWidgetHost` 和 `RenderWidget` 都包含有一个 `GetRoutingID()` 的成员函数，它会返回一个正确的 routing ID.

### 处理消息

要处理 IPC 消息，需要实现 `IPC::Listener` 接口，这个接口里最重要的函数是 `OnMessageReceived`, chromium 在这个函数里提供了几个宏来帮助消息分发：

```cpp
MyClass::OnMessageReceived(const IPC::Message& message) {
  IPC_BEGIN_MESSAGE_MAP(MyClass, message)
    // Will call OnMyMessage with the message. The parameters of the message will be unpacked for you.
    IPC_MESSAGE_HANDLER(ViewHostMsg_MyMessage, OnMyMessage)  
    ...
    IPC_MESSAGE_UNHANDLED_ERROR()  // This will throw an exception for unhandled messages.
  IPC_END_MESSAGE_MAP()
}

// This function will be called with the parameters extracted from the ViewHostMsg_MyMessage message.
MyClass::OnMyMessage(const GURL& url, int something) {
  ...
}
```

你还可以使用 `IPC_DEFINE_MESSAGE_MAP` 宏来完成上述操作，这种情况下不需要自己去定义 `OnMessageReceived` 函数，类似于 MFC 的消息映射宏一样，只需要在头文件里定义消息映射就行了：

```cpp
IPC_DEFINE_MESSAGE_MAP (RenderWidgetHost )
  IPC_MESSAGE_HANDLER( ViewHostMsg_RenderViewReady , OnMsgRenderViewReady )
  IPC_MESSAGE_HANDLER( ViewHostMsg_RenderViewGone , OnMsgRenderViewGone )
  IPC_MESSAGE_HANDLER( ViewHostMsg_Close , OnMsgClose )
  IPC_MESSAGE_HANDLER( ViewHostMsg_RequestMove , OnMsgRequestMove )
  IPC_MESSAGE_HANDLER( ViewHostMsg_PaintRect , OnMsgPaintRect )
  IPC_MESSAGE_HANDLER( ViewHostMsg_ScrollRect , OnMsgScrollRect )
  IPC_MESSAGE_HANDLER( ViewHostMsg_HandleInputEvent_ACK , OnMsgInputEventAck )  
  IPC_MESSAGE_HANDLER( ViewHostMsg_Focus , OnMsgFocus )
  IPC_MESSAGE_HANDLER( ViewHostMsg_Blur , OnMsgBlur )
  IPC_MESSAGE_HANDLER( ViewHostMsg_SetCursor , OnMsgSetCursor )
  IPC_MESSAGE_HANDLER( ViewHostMsg_ImeUpdateStatus , OnMsgImeUpdateStatus )
  IPC_MESSAGE_UNHANDLED_ERROR()
IPC_END_MESSAGE_MAP ()
```

`IPC_MESSAGE_FORWARD` 这个宏类似于 `IPC_MESSAGE_HADNLER`, 但它可以指定处理这个消息的类是哪个，而不一定非得是当前类才行：

```cpp
IPC_MESSAGE_FORWARD(ViewHostMsg_MyMessage, some_object_pointer, SomeObject::OnMyMessage)
```

`IPC_MESSAGE_HANDLER_GENERIC` 这个宏可以让你直接编写处理消息的代码，当然你得自己展开消息参数：

```cpp
IPC_MESSAGE_HANDLER_GENERIC(ViewHostMsg_MyMessage, printf("Hello, world, I got the message."))
```

### 安全性的注意事项

参考 [security for IPC](https://www.chromium.org/Home/chromium-security/education/security-tips-for-ipc) 这篇文章来了解更多细节。


## Channels

`IPC::Channel` 定义了通过管道进行通信的方法。`IPC::SyncChannel` 提供了同步等待某些消息的回应的方法。

Channels 不是线程安全的。如果 UI 线程里想通过 channel 发送消息，那么它必须通过 I/O 线程来完成。因此 chromium 提供了 `IPC::ChannelProxy`, 它的 API 和 channel 对象类似，但它会把消息代理到另一个线程(I/O 线程)上去发送，收到消息的时候则会把消息代理回来。

你的对象(通常运行在 UI 线程)可以在 channel 线程(通常是 I/O 线程)安装一个 `IPC::ChannelProxy::Listener` 对象，这样 I/O 线程中的 channel 收到消息后，可以由 `IPC::ChannelProxy::Listener` 过滤出想要的消息，并代理回 UI 线程。chromium 使用这种机制来接收那些由 I/O 线程完成的资源请求。`RenderProcessHost` 安装了一个 `RenderMessageFilter` 对象来完成这种过滤操作；

### 同步消息

在 renderer 看来，某些消息应该是同步发送的。这种情况大部分发生在 WebKit 需要我们提供一个返回值，但这个返回值必须在 browser 中提供。比如 JavaScript 获取 cookies 时。

browser-to-renderer 的同步消息是不允许的，以免影响界面响应。

**注意：** 不要在 UI 线程中处理同步消息！你只能在 I/O 线程中处理它。Otherwise, the application might deadlock because plug-ins require synchronous painting from the UI thread, and these will be blocked when the renderer is waiting for synchronous messages from the browser. (这句话没懂，不理解 plug-ins 的实现机制)

### 声明同步消息

同步消息可以用 `IPC_SYNC_MESSAGE_*` 系列宏来声明，这些宏包括了参数和返回值(异步消息是没有返回值的)，比如下面代码声明了一个包含 2 个参数 1 个返回值的 control 消息：

```cpp
IPC_SYNC_MESSAGE_CONTROL2_1(SomeMessage,  // Message name
                            GURL, //input_param1
                            int, //input_param2
                            std::string); //result
```

### 发送同步消息

WebKit 线程(渲染线程)发起一个同步的 IPC 请求，这个请求对象 (继承自 `IPC::SyncMessage`) 会通过 `IPC::SyncChannel` 被分发到 renderer 主线程(I/O 线程)上。`IPC::SyncChannel` 会阻塞 WebKit 线程，直到收到消息的回复。

在 WebKit 线程等待同步消息回复的过程中，renderer 主线程还在继续接收来自 browser 的消息，这些消息会被添加到 WebKit 线程的阻塞队列中，包括 browser 的消息回复也会增加到这个队列里，当收到消息回复后，WebKit 线程会被唤醒，开始处理队列里积压的消息。这意味着，如果在等待的过程中收到了别的消息，那么线程被唤醒后会先处理这些消息，最后才处理消息回复。

同步消息的发送方法跟普通消息类似，只是返回值需要作为消息构造函数的参数穿进去，用引用的方式来获取返回值：

```cpp
const GURL input_param("http://www.google.com/");
std::string result;
content::RenderThread::Get()->Send(new MyMessage(input_param, &result));
printf("The result is %s\n", result.c_str());
```

### 处理同步消息

同步消息和异步消息都可以使用 `IPC_MESSAGE_HANDLER` 等宏来处理。只需把返回值写入到参数里即可：

```cpp
void RenderProcessHost::OnMyMessage(GURL input_param, std::string* result) {
  *result = input_param.spec() + " is not available";
}
```

### 将 message type 转换为 message name

程序崩溃的时候，你可能希望把 message type 转换为 message name. message type 是一个 32-bit 的值，高 16-bit 是分类，低 16-bit 是 ID, 分类是基于 `ipc/ipc_message_start.h` 中的枚举，id 基于定义消息的行数。这意味着你必须根据代码的版本才能正确推断出 message name.

比如 `0x1c0098` 消息，其中 0x1c 代表它在 `ipc/ipc_message_start.h` 中枚举的第 29 个，也就是 `ChildProcessMsgStart`, 这个分类的消息定义在 `content/common/child_process_messages.h` 里面，他的第 0x98 行是 `ChildProcessHostMsg_ChildHistogramData` 消息。

如果你需要处理 `content::RenderProcessHostImpl::OnBadMessageReceived` 引起的崩溃，你可能会用到上面的技术。

