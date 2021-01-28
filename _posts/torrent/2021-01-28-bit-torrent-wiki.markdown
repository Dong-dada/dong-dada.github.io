---
layout: post
title:  "BitTorrent Wiki"
date:   2021-01-28 12:26:30 +0800
categories: torrent
---

* TOC
{:toc}

[BitTorrent Wiki](https://en.wikipedia.org/wiki/BitTorrent)


# 简介

![]({{site.url}}/asset/torrent-description.gif)

- 上传者创建一个 torrent 文件，随后把这个文件上传到 torrent index site. 第一个上传者被称为 seed, 其它下载者被称为 peer, 当 peer 下载完文件后，它就变成了一个 seed.
- 下载者从 torrent index site 下载 torrent 文件.
- 下载者连接到 torrent 文件里的 tracker, 从 tracker 那里拿到一堆 seeds 和 peers 的 IP 地址。
- 文件会被分段为多个 piece，每当 peer 接收到了一个完整的 piece, 它就能作为这个 piece 的下载来源供其它 peer 下载，这样原始 seed 就不需要提供这个 piece 给每个 peer 了。
  - 每个 piece 都会计算一个 cryptographic hash 保存在 torrent 里面，这样可以保证从其它 peer 下载的 piece 不被篡改。
  - piece 的下载通常是无序的，每个 piece 的大小都一样，BitTorrent 客户端在下载到 piece 之后会对它进行正确的排序。这使得下载任务可以被暂停、恢复而不丢失之前已经下载到的 piece。这也使得下载器可以先去下载那些容易下载的 piece, 而不必等待前一个 piece 下载完毕。


# 运作方式

在 2005 年以前，分享文件必须通过 torrent 文件来完成；2005 年, [Vuze](https://en.wikipedia.org/wiki/Vuze) 提供了分布式追踪(distributed tracking)功能，使得网络中的客户端可以直接交换数据，不需要 torrent 文件；2006 年，增加了 peer 交换功能，允许客户端根据已连接节点上的数据来添加 peer.

BitTorrent 相对于 HTTP 等下载方式，可以减少 provider 的资源占用，避免流量高峰。然而它也有一些缺点：
- 下载者需要更多的时间才能到达完整的下载速度，因为它需要与足够的 peer 建立连接；
- BitTorrent 下载 piece 通常是无序的，这使得它对流媒体播放不太友好，在 2014 年 [https://en.wikipedia.org/wiki/Popcorn_Time] 这个客户端支持了 BitTorrent 视频文件的流媒体播放，后面许多客户端都支持了该选项。


# 搜索 torrent 文件

BitTorrent 协议本身没有规定 torrent 文件如何分发。一般情况下会有一些网站负责存放、分发 torrent 文件，这被称为 torrent index site, 比如 [The Pirate Bay](https://en.wikipedia.org/wiki/The_Pirate_Bay), [Torrentz](https://en.wikipedia.org/wiki/Torrentz), [isoHunt](https://en.wikipedia.org/wiki/IsoHunt), [BTDigg](https://en.wikipedia.org/wiki/BTDigg)。

通常这些网站也支持 BitTorrent Tracker 功能，但分发 torrent 文件和作为 tracker 这两件事是不相关的，你完全可以让 torrent 文件被某个 tracker site 追踪，随后把这个 torrent 文件上传到其他 index site 上。

有一些 tracker site 是私有的，它们会限制注册用户的权限，并追踪每个用户的上传和下载数量，减少 "吸血" 的情况。


# 下载 torrent 随后共享文件

下载到 torrent 文件后，BitTorrent 客户端会连接到文件中指定的 tracker，从 tracker 获取到当前正在传输 pieces 的 peers 列表。随后客户端连接到这些 peers 来下载不同的 pieces. 如果 BT 网络(被称为 swarm) 中只有一个初始的 seeder, 客户端就会直接连到这个 seeder 上，开始请求 pieces.

数据交换的有效性很大程度上取决于客户端采取何种策略来决定要把数据发送给哪个 peer. 一种策略是基于公平交易的原则，把数据发送给那些会返回数据给自己的 peer，但这种策略对新加入的 peer 很不友好，因为这些 peer 没有数据可以返回给发送方，另外也可能导致两个 peer 之间的连接情况虽然很好，但就因为谁都不愿意主动发送数据，导致数据交换建立不起来。BitTorrent 的官方客户端里使用了一种称为 "optimistic unchoking" 的机制，客户端会保留一部分可用带宽，使用这部分带宽向随机的 peer 发送数据，以便发现更好的 peer, 也给了新的 peer 机会来加入 swarm.

虽然 BitTorrent 可以降低流行资源的成本，但对于小众资源来说没有多大用处，这类资源只在一开始会有人下载，后面就没什么人下载和做种了，这种情况下后加入的 peer 需要一直等待 seed 的加入，否则就无法完成下载。对于做种的服务商来说，为小众资源做种需要高昂的带宽成本，这与他们使用 BitTorrent 替换传统 Client-Server 模式的出发点背道而驰。一些资源的发布者为了解决这种问题，会把多个文件捆绑在一起，让做种的人多一些。此外还有一些人提出了更复杂的方案，他们使用跨种子的机制，让多个种子合作来更好的分享内容。


# 创建和发布 torrent

