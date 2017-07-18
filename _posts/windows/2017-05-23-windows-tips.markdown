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