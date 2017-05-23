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