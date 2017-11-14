---
layout: post
title:  "编译 libtorrent 并运行 demo"
date:   2017-11-09 00:08:30 +0800
categories: libtorrent
---

* TOC
{:toc}


最近对 libtorrent 比较感兴趣，本文介绍如何编译 libtorrent, 并且运行官网提供的示例代码。

先说一下版本：
- boost: 1.65.1
- libtorrent: 1.1.5
- 使用 VS2017 编译

## 编译 boost

libtorrent 依赖于 boost 库，所以在编译 libtorrent 之前，得先把 boost 编译出来。

编译 boost 的方法可以参考 [官方文档](http://www.boost.org/doc/libs/1_65_1/more/getting_started/windows.html)，我说一下我的步骤。

### Boost.Build

下载 boost 源代码并解压，比如我解压到了 `D:/code/boost_1_65_1` 这个文件夹下。

boost 根目录下有一个 bootstrap.bat 文件，用命令行执行它。如果执行成功，那么会在根目录生成 b2.exe, bjam.exe 这两个程序。他俩其实是一样的可执行文件，只不过名字不一样。

b2 或者 bjam 是一个解释器程序，专门用来解释运行 Boost.Build 脚本。Boost.Build 简单来说就是类似于 make 之类的程序构建方案。它的脚本里说明了要编哪些文件，互相之间的依赖关系是咋样的，如何链接得到可执行文件等，解释器读取脚本之后，就会根据配置去调用编译器、链接器来完成程序的编译工作。

boost 源码本身支持通过 Boost.Build 这种方式来编译。你可以看到 boost 的根目录有一个 jamroot 文件，他就是 Boost.Build 的"脚本"(用脚本这个词不太合适，大概就类似于 makefile 那样的东西吧)。

**注意：**

在执行 bootstrap.bat 的时候，我遇到了如下错误：

```
D:\code\boost_1_65_1>bootstrap.bat
Building Boost.Build engine

Failed to build Boost.Build engine.
Please consult bootstrap.log for further diagnostics.

You can try to obtain a prebuilt binary from

   http://sf.net/project/showfiles.php?group_id=7586&package_id=72941

Also, you can file an issue at http://svn.boost.org
Please attach bootstrap.log in that case.
```

bootstrap.log 中的内容为：

```
**********************************************************************
** Visual Studio 2017 Developer Command Prompt v15.4.1
** Copyright (c) 2017 Microsoft Corporation
**********************************************************************
[vcvarsall.bat] Environment initialized for: 'x86'
###
### Using 'vc141' toolset.
###

C:\Users\dongy\source>if exist bootstrap rd /S /Q bootstrap

C:\Users\dongy\source>md bootstrap

C:\Users\dongy\source>cl /nologo /RTC1 /Zi /MTd /Fobootstrap/ /Fdbootstrap/ -DNT -DYYDEBUG -wd4996 kernel32.lib advapi32.lib user32.lib /Febootstrap\yyacc0 yyacc.c
yyacc.c
c1: fatal error C1083: 无法打开源文件: “yyacc.c”: No such file or directory
```

解决方法是使用 VS2017 自带的 `x86 Native Tools Command Prompt for VS 2017` 这个命令行工具来执行 bootstrap.bat

### 编译

在 boost 根目录执行 `b2 --help` 就能看到 b2 程序的使用方法。根据其中的内容，我执行了如下命令:

```
b2 --stagedir=D:\code\boost_1_65_1\stage --build-type=complete variant=debug link=static threading=multi runtime-link=shared debug-symbols=on debug-store=database stage
```

上述命令中：
- stagedir: 表示按照 stage 方式编译时，编译结果的输出目录；
- build-type: complete 表示完全编译；
- link: 表示最后生成的库是 静态库(static) 还是 动态库(shared)；
- runtime-link: 表示以何种方式使用运行时库，是 链接静态库(static) 还是 链接动态库(shared)；
- debug-symbols=on, debug-store=database: 用于指定生成 pdb 文件
- stage: boost 支持的编译方式有两种，install 表示把头文件和库文件都生成到输出目录里；stage 表示只把库文件生成到输出目录里。因为 boost 目录里已经有头文件的代码了，所以我使用了 stage 方式。

上述命令执行成功之后，boost 就编译完成了，编译好的 `.lib` 文件保存在我所指定的 `D:\code\boost_1_65_1\stage` 文件夹中。

pdb 文件没有自动放到 stage 文件夹里，而是放在 `bin.v2` 的子文件夹里。搜了一下应该没有单独导出 pdb 的命令，不过问题不大，当程序链接 `.lib` 的时候，会自动查找到 pdb 的位置。如果确实需要的话，可以写个批处理来做这件事。

随便建个项目，把 boost 根目录加入到附加包含目录里，把库目录加入到附加库目录里。然后编写如下代码来测试是否能够正确调用到 boost:

```cpp
#include <boost/regex.hpp>
#include <iostream>
#include <string>

int main()
{
    std::string line;
    boost::regex pat( "^Subject: (Re: |Aw: )*(.*)" );

    while (std::cin)
    {
        std::getline(std::cin, line);
        boost::smatch matches;
        if (boost::regex_match(line, matches, pat))
            std::cout << matches[2] << std::endl;
    }
}
```

**注意：**
之后发现我的项目没有开启 rtti, 而 boost 默认编译是开启了 rtti 的，导致 API 不一致。解决方法是加上如下参数关闭掉 boost 的 rtti:

```
define=BOOST_NO_RTTI define=BOOST_NO_TYPEID --without-log

完整命令：
b2 --stagedir=D:\code\boost_1_65_1\stage --build-type=complete variant=debug link=static threading=multi runtime-link=shared debug-symbols=on debug-store=database define=BOOST_NO_RTTI define=BOOST_NO_TYPEID --without-log stage
```

前面两个宏可以参考 [这份文档](http://www.boost.org/doc/libs/1_46_1/libs/exception/doc/configuration_macros.html)。 `--without-log` 目的是不编译 boost.log 库，因为这个库依赖了 rtti.

### 小结

Boost 的编译分为两步：
1. bootstrap.bat, 用于生成 Boost.Build 的解释器 b2.exe;
2. b2.exe, 使用解释器编译 boost;


## 编译 libtorrent

下载并解压 libtorrent 库。之后你可以看到它的根目录里有 jamfile, jamroot.jam 这两个文件，它们就是使用 Boost.Build 编译 libtorrent 时所需要的构建脚本。

### 设置环境变量

首先得把 boost 的根目录添加到环境变量的 PATH 里，方便在命令行里找到 b2.exe.

然后设置 `BOOST_ROOT` 环境变量，其值为 boost 的根目录，这个环境变量是 libtorrent 的 jamfile 里需要的。

### 编译

在 libtorrent 根目录下执行以下命令：

```
b2 --libdir=%s --includedir=%s --build-type=complete link=static runtime-link=static boost-link=static variant=debug dht=on debug-symbols=on debug-store=database install
```

指定了 install 方式来编译，这样可以通过定义 `libdir`, `includedir` 的方式来指定 lib 的输出位置。

具体可以参考 [官方文档](http://www.libtorrent.org/building.html)。

上述命令执行完毕后，会在 libdir 指定的位置生成编译结果，其中就包含了 libtorrent.lib.

### 运行 demo

[官方文档](http://www.libtorrent.org/tutorial.html) 中提供了一个下载磁力链的简单例子：

```cpp
#include <iostream>
#include <thread>
#include <chrono>

#include <libtorrent/session.hpp>
#include <libtorrent/add_torrent_params.hpp>
#include <libtorrent/torrent_handle.hpp>
#include <libtorrent/alert_types.hpp>

namespace lt = libtorrent;
int main(int argc, char const* argv[])
{
    if (argc != 2) {
        std::cerr << "usage: " << argv[0] << " <magnet-url>" << std::endl;
        return 1;
    }
    lt::settings_pack pack;
    pack.set_int(lt::settings_pack::alert_mask
        , lt::alert::error_notification
        | lt::alert::storage_notification
        | lt::alert::status_notification);

    lt::session ses(pack);

    lt::add_torrent_params atp;
    atp.url = argv[1];
    atp.save_path = "."; // save in current dir
    lt::torrent_handle h = ses.add_torrent(atp);

    for (;;) {
        std::vector<lt::alert*> alerts;
        ses.pop_alerts(&alerts);

        for (lt::alert const* a : alerts) {
            std::cout << a->message() << std::endl;
            // if we receive the finished alert or an error, we're done
            if (lt::alert_cast<lt::torrent_finished_alert>(a)) {
                goto done;
            }
            if (lt::alert_cast<lt::torrent_error_alert>(a)) {
                goto done;
            }
        }
        std::this_thread::sleep_for(std::chrono::milliseconds(200));
    }
done:
    std::cout << "done, shutting down" << std::endl;
}
```

把上述代码粘贴到工程里，再把 libtorrent 的 include 文件夹加入到附加包含目录，把库文件夹加入到附加库目录，把 libtorrent.lib 加入到附加依赖项里。尝试编译即可。

**注意：**

编译之后出现了如下错误：

```
1>------ 已启动生成: 项目: libtorrent_test, 配置: Debug Win32 ------
1>libtorrent.lib(assert.obj) : error LNK2019: 无法解析的外部符号 __imp__UnDecorateSymbolName@16，该符号在函数 "class std::basic_string<char,struct std::char_traits<char>,class std::allocator<char> > __cdecl demangle(char const *)" (?demangle@@YA?AV?$basic_string@DU?$char_traits@D@std@@V?$allocator@D@2@@std@@PBD@Z) 中被引用
1>libtorrent.lib(assert.obj) : error LNK2019: 无法解析的外部符号 __imp__StackWalk64@36，该符号在函数 "void __cdecl print_backtrace(char *,int,int,void *)" (?print_backtrace@@YAXPADHHPAX@Z) 中被引用
1>libtorrent.lib(assert.obj) : error LNK2019: 无法解析的外部符号 __imp__SymFunctionTableAccess64@12，该符号在函数 "void __cdecl print_backtrace(char *,int,int,void *)" (?print_backtrace@@YAXPADHHPAX@Z) 中被引用
1>libtorrent.lib(assert.obj) : error LNK2019: 无法解析的外部符号 __imp__SymGetModuleBase64@12，该符号在函数 "void __cdecl print_backtrace(char *,int,int,void *)" (?print_backtrace@@YAXPADHHPAX@Z) 中被引用
1>libtorrent.lib(assert.obj) : error LNK2019: 无法解析的外部符号 __imp__SymGetLineFromAddr64@20，该符号在函数 "void __cdecl print_backtrace(char *,int,int,void *)" (?print_backtrace@@YAXPADHHPAX@Z) 中被引用
1>libtorrent.lib(assert.obj) : error LNK2019: 无法解析的外部符号 __imp__SymInitialize@12，该符号在函数 "void __cdecl print_backtrace(char *,int,int,void *)" (?print_backtrace@@YAXPADHHPAX@Z) 中被引用
1>libtorrent.lib(assert.obj) : error LNK2019: 无法解析的外部符号 __imp__SymFromAddr@20，该符号在函数 "void __cdecl print_backtrace(char *,int,int,void *)" (?print_backtrace@@YAXPADHHPAX@Z) 中被引用
1>libtorrent.lib(assert.obj) : error LNK2019: 无法解析的外部符号 __imp__SymRefreshModuleList@4，该符号在函数 "void __cdecl print_backtrace(char *,int,int,void *)" (?print_backtrace@@YAXPADHHPAX@Z) 中被引用
1>libtorrent.lib(broadcast_socket.obj) : error LNK2019: 无法解析的外部符号 _if_nametoindex@4，该符号在函数 __catch$?supports_ipv6@libtorrent@@YA_NXZ$0 中被引用
1>D:\code\vs2017\libtorrent_test\Debug\libtorrent_test.exe : fatal error LNK1120: 9 个无法解析的外部命令
1>已完成生成项目“libtorrent_test.vcxproj”的操作 - 失败。
========== 生成: 成功 0 个，失败 1 个，最新 0 个，跳过 0 个 ==========
```

有几个符号找不到，搜索发现是没有导入 dbghelp.lib, Iphlpapi.lib 这两个库。把它们添加到附加依赖项里之后，就可以正确编译了。

运行的时候需要输入一个磁力链作为命令行参数。运行起来之后如果一切正常，会把文件下载到当前目录下。
