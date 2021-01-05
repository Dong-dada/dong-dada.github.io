---
layout: post
title:  "远程机器 Connection Refused 错误"
date:   2021-01-05 21:01:30 +0800
categories: network
---

* TOC
{:toc}

最近调试一个东西，发现了个奇怪的问题：

这个程序部署在远程机器上，可以在远程机器上通过 curl 指令来调用这个程序的一些功能，比如 `curl localhost:15000/help`。但是我在本地机器上尝试 `curl 11.11.11.11:15000/help` 却会返回 Connection Refused 错误。

之后发现是因为远程机器上的这个程序，监听的是 127.0.0.1 这个 ip，所以只有在远程机器上通过 localhost 才能访问到这个程序。改成监听 0.0.0.0 这种 ip 之后，就能在本地机器上访问到这个程序了。