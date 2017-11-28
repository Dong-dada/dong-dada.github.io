---
layout: post
title:  "[总结] Chromium 开发小知识点"
date:   2017-11-28 11:41:30 +0800
categories: chromium
---

* TOC
{:toc}


## 如何在 chromium 源码下新建项目

许多时候都是在原有 chromium 代码的基础上新增或修改一些代码。如果需要新建一个项目，使用 chromium 的一些基础设施来进行开发的话，需要按照以下步骤：

- 在 chromium 源码目录下新建一个目录，用于存放你自己的项目代码，比如我新建了一个目录：`src/dada/test/`；
- 在该目录下创建一个 `BUILD.gn` 文件，该文件描述了这个项目如何被构建，你可以把它理解成 VS 里的项目描述文件，稍后我会介绍如何编写该文件；
- 把项目目录加到一个已有的 group 里面，以便 `gn` 能够找到我们新加的项目，你可以把这个步骤理解为在 VS 里把一个项目加入到解决方案里，这里我们把项目目录加到 `src/BUILD.gn` 的 `gn_only` 这个 group 里，不要问我为什么是它，我也不知道 :P

具体来说，我们在 `src/BUILD.gn` 的 `group("gn_only")` 里添加如下内容：

```
group("gn_only") {

  // blah blah ...

  deps += [
    "//dada/test:dada_test",
  ]
}
```

接下来说明一下 `src/dada/test/BUILD.gn` 文件如何编写，以下是一个例子：

```
// executable 表示该项目的生成目标是一个可执行文件
// 类似还有 static_library 表示生成目标是静态库
executable("dada_test") {
  output_name = "dada_test"

  // 指定该项目使用到的源码
  sources= [
    "main.cc",
  ]

  // 指定该项目依赖于 chromium 的 base 库
  deps = [
    "//base",
  ]

  // 指定第三方库的目录
  lib_dirs = [
    "$root_gen_dir/dada/third_party/lib",
  ]

  // 指定该项目所依赖的第三方库
  libs = [
    "third_party.lib",
  ]

  // 编译相关选项
  cflags = [
    "/wd4245",  # disable warning C4245
  ]

  cflags_cc = [
    "/EHsc",    # catch C++ exceptions
  ]
}
```

BUILD.gn 里还有许多值可以用来对构建过程进行控制，具体可以参考 [GN Reference](https://chromium.googlesource.com/chromium/src/+/master/tools/gn/docs/reference.md)。


### 在新项目里使用 MessageLoop

如果我们新建了一个项目，想要在这个项目里使用 chromium 的 Message Loop 等相关机制，要怎么做呢？

很简单，先创建 `base::MessageLoop`, 然后调用 `base::RunLoop().Run()` 就可以了：

```cpp
#include "base/at_exit.h"
#include "base/message_loop/message_loop.h"
#include "base/run_loop.h"

int main() {
  base::MessageLoop loop;
  base::AtExitManager at_exit;

  // ...

  base::RunLoop().Run();
  return 0;
}
```

注意上述代码中的 `base::AtExitManager`, 它用在某些 chrome 的单例对象上。我在使用 `base::RepeatingTimer` 的时候发现如果不加这行代码，会断言错误。

记得在 BUILD.gn 里加上 `//base` 的依赖，这样才能正确链接到 base 静态库：

```
// 自建项目的 BUILD.gn 文件

deps = [
  "//base",
]
```