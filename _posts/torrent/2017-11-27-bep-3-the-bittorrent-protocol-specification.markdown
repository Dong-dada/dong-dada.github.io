---
layout: post
title:  "BEP3 The BitTorrent Protocol Specification"
date:   2017-11-27 23:47:30 +0800
categories: torrent
---

* TOC
{:toc}


本文参考自 [官方文档](http://bittorrent.org/beps/bep_0003.html)

BitTorrent 是一个用于文件分发的协议。通过用户间互相上传文件的机制，它可以支持大量用户同时下载同一个文件，而不会增加文件发布者的负担。

## 通过 BitTorrent 协议发布文件时会涉及到以下几个角色

- 一个原始的 web server;
- 一个静态的 'metainfo' 文件;
- 一个 BitTorrent tracker;
- 一个原始的 downloader;
- 终端用户的浏览器；
- 终端用户的下载器；


## 要发布一个文件，发布者需要按照如下步骤：

- 启动 tracker;
- 启动 web server, 例如 apache;
- 在 web server 上简历 .torrent 扩展名与 mimetype application/x-bittorrent 之间的关联；
- 通过 目标文件 和 trakcer url 来生成一个 .torrent 文件，也就是之前所说的 metainfo 文件；
- 把 metainfo 文件放到 web server 上；
- 在某网页上放上这个 .torrent 文件的下载链接；
- 启动原始的 downloader, 它应该已经有了完整文件，通过它来下载这个 .torrent 文件；


## 要下载一个文件，用户需要按照如下步骤：

- 安装 BitTorrent;
- 连接网络；
- 点击一个 .torrent 文件；
- 选择文件保存位置，或者选择一个 partial download 来恢复下载；
- 等待下载完成；
- 停止下载器(停止之前它会持续进行上传工作)；


## bencoding

bencoding 是 BitTorrent 协议中使用到的一种编码格式，它的规则如下：
- 字符串由十进制的长度前缀来修饰，后面加一个 `:` 冒号，例如 `4:spam` 代表字符串 `spam`；
- 整形用 `i + 数字 + e` 的方式来表示，例如 `i3e` 表示数字 `3`, `i-3e` 表示 `-3`, `i0e` 表示 `0`;
- 列表用 `l + item + item + e` 表示，例如 `l4:spami3ee` 表示 `['spam', 3]`;
- 字典用 `d + item + item + e` 表示，例如 `d4:spaml1:a1:bee` 表示 `{'spam':['a', 'b']}`, 其中 key 必须是字符串，且按字母大小排序；


## metainfo files

也就是种子文件 .torrent.

metainfo files 由 bencoding 方式进行编码，它是一个字典的结构，包含了如下 key:

- announce: tracker 的 url
- info:
    + name: 建议名；
    + piece length: 表示每个 piece 的字节长度。原始文件会按照这个大小被分割为多个 piece, 除了最后一个 piece 可能被裁剪以外，其它 piece 的长度都是一样的。piece length 的大小必须是 2^n, 一般是 2^18=256KB, 或者是 2^20=1MB;
    + pieces: 包含了多个长度为 20 的字符串，每个字符串都对应一个 piece 的 SHA1 hash 值；
    + length: 如果出现该字段，说明是单文件的情况，它表示了该文件的大小，此时文件的名称由 name 字段指定；
    + files: 如果没有出现 length 字段，则说明是多文件的情况，此时 name 字段表示文件夹名称，files 字段说明了文件的信息，它是一个列表，列表中的每一项都包含如下字段：
        * length: 该文件的长度；
        * path: 该文件的路径；


## trackers

tracker 的 GET 请求包含了如下字段：
- **info_hash**: 表示 metainfo 文件中 info 字段的 SHA1 hash 值，是一个 20 字节的字符串；
- **peer_id**: downloader 的唯一标识，每个 downloader 在启动一个新的下载任务时，都应当随机生成一个新的 `peer_id`；
- **ip**: 可选参数，用来表示该 peer 对应的 ip.
- **port**: 该 peer 监听的端口号，通常情况下 downloader 应当监听 6881, 如果不行就尝试 6882, 直到 6889.
- **uploaded**: 迄今为止已经上传的总量；
- **downloaded**: 迄今为止已经下载的总量；
- **left**: 剩余需要下载的大小；
- **event**: 可选参数，取值可以是：`started`, `completed`, `stopped`, (或者 `empty`, 相当于没有这个参数)。

Tracker 会回复一个由 bencode 编码的字典，如果此次请求失败，则 tracker 返回 failure reason 字段来说明失败原因。否则，它将返回两个字段：
- **interval**: 告知 downloader 隔一段时间后再来查询；
- **peers**: 返回一个 peer 列表，其中每一项都由 `id`, `ip`, `port` 组成；


## peer protocal

BitTorrent 的 peer 协议通过 TCP 或 [uTP](http://bittorrent.org/beps/bep_0029.html) 来实现。

peer 的连接是对称的，双方都可以向对方发送消息，数据也可以双向流动。

peer 协议通过索引来引用 piece, 这个索引由 metainfo 文件给出，由 0 开始。一旦一个 peer 完成了某个 piece 的下载并校验通过，他就会向它的所有 peers 公告(announce)这件事情。

在连接的两端都维护了两个状态位：`choked`, `interested`. choking 是一个通知，用于表示接下来不会再发送数据了，除非 unchoking 发生。



