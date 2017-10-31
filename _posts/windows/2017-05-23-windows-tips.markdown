---
layout: post
title:  "Windows 编程小知识点"
date:   2017-05-23 11:49:30 +0800
categories: windows
---

* TOC
{:toc}


## 获取用户上次输入的时间，判断用户长时间无输入

有时候需要判断用户多长时间内没有输入，然后执行一些操作，比如锁屏、开启某些功能之类的。

这个可以通过 [GetLastInputInfo](https://msdn.microsoft.com/en-us/library/windows/desktop/ms646302.aspx) 这个 API 来完成。

这个 API 返回用户最后一次输入操作发生的时间，这个时间是距系统启动到用户最后一次输入的毫秒数。

有了这个毫秒数，就可以用 [GetTickCount](https://msdn.microsoft.com/en-us/library/ms724408.aspx) 减去它来计算出用户无输入的时间：

```cpp
#include <iostream>
#include <Windows.h>

int main()
{
    LASTINPUTINFO last_input_info;
    last_input_info.cbSize = sizeof(LASTINPUTINFO);
    last_input_info.dwTime = 0;

    for (int i = 0; i < 100; ++i)
    {
        BOOL ret = GetLastInputInfo(&last_input_info);
        DWORD tick_count = GetTickCount();
        std::cout << "ret = " << ret << ", dwTime = " << last_input_info.dwTime << ", interval = " << tick_count - last_input_info.dwTime << std::endl;
        Sleep(1000);
    }

    return 0;
}
```


## Windows 文字转语音 TTS

Windows 提供了将文字转换为语音的库，这个库包含在 Speak API (SAPI) 库中。

相关内容可以参考 [MSDN](https://msdn.microsoft.com/en-us/library/ms720571.aspx)。

简单说来你需要调用 ISPVoice 接口的 Speak 方法，向它传入一个字符串，他就会将这个字符串用语音引擎播放出来。

语音引擎可以在控制面板中看到：

![]( {{site.url}}/asset/windows-tips-tts.png )

需要特别说明的是，你可以在传入 Speak 方法的字符串中设置 xml 标签，来对语音播放的情况进行定制。比如默认情况下 12345 会被读为 “一万两千三百四十五”，如果你希望它一个一个数字读出来，那么可以用标签将它标记起来：

```xml
<context ID="number_digit">123456</context>
```

对于手机号码，则可以用另外的标签：

```xml
<context ID="CHS_phone_postal">123456</context>
```

相关内容可以参考 [MSDN](https://msdn.microsoft.com/en-us/library/ms723628.aspx#CHS_Context_Number)

如何使用 SAPI 就不介绍了，用到的时候再研究好了。


## WM_LBUTTONDBLCLK 事件

之前一直以为 `WM_LBUTTONDBLCLK` 消息就是单独的一条消息。今天遇到一个问题，查了一下 [MSDN](https://msdn.microsoft.com/en-us/library/windows/desktop/ms645606.aspx) 才发现这个消息有一些细节需要注意下：

> which the system generates whenever the user presses, releases, and again presses the left mouse button within the system's double-click time limit. Double-clicking the left mouse button actually generates a sequence of four messages: `WM_LBUTTONDOWN`, `WM_LBUTTONUP`, `WM_LBUTTONDBLCLK`, and `WM_LBUTTONUP`。

以前一直以为双击只会产生一条消息，实际上双击会产生四条消息：`WM_LBUTTONDOWN`, `WM_LBUTTONUP`, `WM_LBUTTONDBLCLK`, `WM_LBUTTONUP`。也就是说，第二次按下鼠标左键的时候，就会生成 `WM_LBUTTONDBLCLK` 消息，这时候鼠标弹起，还会继续生成 `WM_LBUTTONUP` 消息。换句话说，产生双击消息需要三步而不是四步。

我所遇到的问题就来源于此：

首先，有两个窗口 LeftWnd, RightWnd :

![]( {{site.url}}/asset/windows-tips-dbclick-1.png )

双击 LeftWnd 左侧窗口会扩大，把 RightWnd 盖住：

![]( {{site.url}}/asset/windows-tips-dbclick-2.png )

接着在原来 RightWnd 的区域内再次双击 LeftWnd, 这时候 LeftWnd 会缩回去。

这个时候，实际上双击操作的最后一个 `WM_LBUTTONUP` 是由 RightWnd 接收并处理的。RightWnd 收到鼠标弹起事件后会做一些响应操作。从现象上看来，明明双击了 LeftWnd, RightWnd 却响应了鼠标弹起事件，问题就这样出现了。

解决方法我想了一下有两种：
- RightWnd 处理的 `WM_LBUTTONUP` 消息，实际上目的是处理 “鼠标按下，移动一段，再弹起” 这个操作，最后的 `WM_LBUTTONUP` 事件是一个孤立的事件，那么可以在它收到 `WM_LBUTTONDOWN` 消息的时候标记一下，如果 `WM_LBUTTONUP` 的时候这个标记不是 true, 那么说明这个 `WM_LBUTTONUP` 消息是一个孤立的消息，不应当被处理；
- 另一种是对 LeftWnd 的双击操作做一下处理，当收到 `WM_LBUTTONDBLCLK` 事件的时候，通过 `SetCapture()` 设置一下让 LeftWnd 去接受 `WM_LBUTTONUP` 消息，这样 RightWnd 就不会收到这个消息了，这种方法我还没试过，不过应该是可行的；


## 在程序启动时弹出  UAC 提示

有时候调用某些接口，需要获取管理员权限，比如通过 `CreateFile()` 获取盘符的句柄的时候。

这就要求程序以管理员权限启动。换句话说，在程序启动的时候弹出一个 UAC 提示，从用户那里获取到运行的权限。

这需要在程序清单文件中配置如下内容：

```xml
<?xml version="1.0" encoding="UTF-8" standalone="yes"?>
<assembly xmlns="urn:schemas-microsoft-com:asm.v1" manifestVersion="1.0">
  <trustInfo xmlns="urn:schemas-microsoft-com:asm.v3">
    <security>
      <requestedPrivileges>
        <requestedExecutionLevel level="requireAdministrator" uiAccess="false"></requestedExecutionLevel>
      </requestedPrivileges>
    </security>
  </trustInfo>
</assembly>
```

也可以在 visual studio 的设置面板中进行设置：

![]( {{site.url}}/asset/debug-tips-required-administrator.png )

只需将 UAC 执行级别设置为 requiredAdministrator 即可。
