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
- 使用 [network address translation](https://en.wikipedia.org/wiki/Network_address_translation)(NAT) 的路由器需要维护一张来源和目标地址的关系表。通常家庭路由器只能有 2000 个表项。BitTorrent 的客户端通常每秒就连接 20-30 个服务，因此会很快占用完 NAT 表，导致家庭路由器无法正常工作。


Routers that use network address translation (NAT) must maintain tables of source and destination IP addresses and ports. Typical home routers are limited to about 2000 table entries[citation needed] while some more expensive routers have larger table capacities. BitTorrent frequently contacts 20–30 servers per second, rapidly filling the NAT tables. This is a known cause of some home routers ceasing to work correctly.[74][75]


# 搜索 torrent 文件

BitTorrent 协议本身没有规定 torrent 文件如何分发。一般情况下会有一些网站负责存放、分发 torrent 文件，这被称为 torrent index site, 比如 [The Pirate Bay](https://en.wikipedia.org/wiki/The_Pirate_Bay), [Torrentz](https://en.wikipedia.org/wiki/Torrentz), [isoHunt](https://en.wikipedia.org/wiki/IsoHunt), [BTDigg](https://en.wikipedia.org/wiki/BTDigg)。

通常这些网站也支持 BitTorrent Tracker 功能，但分发 torrent 文件和作为 tracker 这两件事是不相关的，你完全可以让 torrent 文件被某个 tracker site 追踪，随后把这个 torrent 文件上传到其他 index site 上。

有一些 tracker site 是私有的，它们会限制注册用户的权限，并追踪每个用户的上传和下载数量，减少 "吸血" 的情况。


# 下载 torrent 随后共享文件

下载到 torrent 文件后，BitTorrent 客户端会连接到文件中指定的 tracker，从 tracker 获取到当前正在传输 pieces 的 peers 列表。随后客户端连接到这些 peers 来下载不同的 pieces. 如果 BT 网络(被称为 swarm) 中只有一个初始的 seeder, 客户端就会直接连到这个 seeder 上，开始请求 pieces.

数据交换的有效性很大程度上取决于客户端采取何种策略来决定要把数据发送给哪个 peer. 一种策略是基于公平交易的原则，把数据发送给那些会返回数据给自己的 peer，但这种策略对新加入的 peer 很不友好，因为这些 peer 没有数据可以返回给发送方，另外也可能导致两个 peer 之间的连接情况虽然很好，但就因为谁都不愿意主动发送数据，导致数据交换建立不起来。BitTorrent 的官方客户端里使用了一种称为 "optimistic unchoking" 的机制，客户端会保留一部分可用带宽，使用这部分带宽向随机的 peer 发送数据，以便发现更好的 peer, 也给了新的 peer 机会来加入 swarm.

虽然 BitTorrent 可以降低流行资源的成本，但对于小众资源来说没有多大用处，这类资源只在一开始会有人下载，后面就没什么人下载和做种了，这种情况下后加入的 peer 需要一直等待 seed 的加入，否则就无法完成下载。对于做种的服务商来说，为小众资源做种需要高昂的带宽成本，这与他们使用 BitTorrent 替换传统 Client-Server 模式的出发点背道而驰。一些资源的发布者为了解决这种问题，会把多个文件捆绑在一起，让做种的人多一些。此外还有一些人提出了更复杂的方案，他们使用跨种子的机制，让多个种子合作来更好的分享内容。


# 创建和发布 torrent

Peer 会把文件分割成一堆相同大小的 piece 来发送，这个大小是 2 的幂次，一般在 32KB 到 16MB 之间。Peer 使用 SHA-1 为每个 piece 生成一个 hash 值，记录到 torrent 文件里。大于 512KB 的 piece 可以降低 torrent 文件的大小，但有可能降低 BitTorrent 协议的效率。另一个 peer 收到 piece 之后，会计算这个 piece 的 hash 值，并与 torrent 文件中记录的原始 hash 值比较，从而校验 piece 是否被篡改。能够提供完整文件的 peer 被称为 seeders，首先发布文件的 peer 被称为 initial seeder.

一般来说，torrent 文件的后缀是 ".torrent". Torrent 文件有个 announce 字段，其中指定了 tracker 的 URL；此外 torrent 文件里还有个 info 字段，包含了建议的文件名、文件大小、每个 piece 的大小，以及每个 piece 的 SHA-1 hash 值，客户端会借助这些数据来校验收到的数据的合法性。虽然 SHA-1 的加密强度不足，但开发者 Bram Cohen 一开始没有考虑到安全性问题，导致协议无法向后兼容，改为更加安全的 SHA-3 等方式。同时 BitTorrent v2 已经把加密方式升级到了 SHA-256.

早期 torrent 文件通常会发布到 torrent index site 上，并注册至少一个 tracker，tracker 会管理当前加入 swarm 的 client 列表。除此之外，也有一种被称为 trackerless system(去中心化的 tracking)，在这种系统中，每个 peer 都是一个 tracker，Azureus 是第一个实现了这种系统的 BT 客户端，它使用的方式是 [distributed hash table](https://en.wikipedia.org/wiki/Distributed_hash_table)(DHT)。还有另外一种不兼容的 DHT 系统，被称为 [Mainline DHT](https://en.wikipedia.org/wiki/Mainline_DHT)，由 Mainline BT 客户端发布，随后被其他几种客户端采用。

在 DHT 被采用之后，一些人非正式地引入了一个 "private" 标志，告诉其他客户端不要去使用 DHT 等去中心化的追踪技术。这个字段特意被放在 torrent 文件的 info 字段里面，这样的话除非修改 torrent 文件，否则无法被修改或移除。这个标志的目的是阻止 torrent 被分享给无权访问特定 tracker 的客户端。这个标志于 2008 年被要求加入到官方规范当中，但目前还未被接受。忽略这个字段的客户端被许多 Tracker 禁止了。


# 匿名性

BitTorrent 自身没有提供匿名功能，一个客户端通常可以看到 swarm 当中所有 peer 的 IP 地址，这可能导致用户暴露在攻击之下。在一些国家，版权组织会爬取 peer 列表，然后把下载版权保护内容的 ip 发给 ISP，让他们删除这些 ip。在一些地方，版权持有者会使用法律武器来对抗不法的上传者和下载者。

有很多方式可以提高匿名性，比如 [Tribler](https://en.wikipedia.org/wiki/Tribler) 这个客户端使用了一个 [Tor](https://en.wikipedia.org/wiki/Tor_(anonymity_network))-like 的洋葱网络，可以借由其它 peer 来获取要下载的数据。在 swarm 中可以看到一些节点，但这些节点都是由 Tribler 提供的，真正下载的节点是无法被看到的。Tribler 的一个优势是它只需要额外的一跳就可以下载到文件，对下载速度的影响不大。

[i2p](https://en.wikipedia.org/wiki/I2p) 提供了一个类似的匿名层，用户可以下载上传到 i2p 网络的文件。Vuze 这种 BT 客户端既支持获取普通网络中的文件，也支持从 i2p 网络中下载文件。

大部分 BitTorrent 没有因为匿名性对软件做特殊的设计，此外在 Tor 网络当中使用 BT 软件是否会拖慢网络速度也是存疑的。

私有的 torrent tracker 通常是邀请制的，并且会要求成员都参与上传，但这种 tracker 存在单点故障的缺点。[Oink's Pink Palace](https://en.wikipedia.org/wiki/Oink%27s_Pink_Palace) 和 [What.cd](https://en.wikipedia.org/wiki/What.cd) 是两个私有 tracker 的例子，不过他们都已经关闭了。

[Seedbox](https://en.wikipedia.org/wiki/Seedbox) 服务支持把 BT 文件下载到公司服务器，用户可以直接从公司服务器下载 BT 文件，这样用户 IP 仅对公司服务器可见，三方是不可见的。

[Virtual private networks](https://en.wikipedia.org/wiki/Virtual_private_network) 网络会加密通信，并隐藏用户的真实 IP，这样用户的真实 IP 不会暴露在 swarm 里面。


# BitTorrent V2

BitTorrent V2 协议被设计为能够跟 V1 版本的客户端一起工作。这次协议升级的主要目的是更新加密方式，从 SHA-1 升级到了 SHA-256。为了保证向后兼容，V2 版本的 torrent 文件支持混合模式，在这种模式下，会使用新方式和旧方式来对文件做 hash 处理，这样文件既可以被分享给 v1 的 peer, 也可以被分享给 v2 的 peer。另外一个更新是增加了一个 hash 树，从而加快从添加种子文件到下载文件之间的时间，并且允许对文件损坏进行更详细的检查。

此外每个文件都被独立地进行哈希，这样 swarm 中的重复数据可以删除掉，如果多个种子包含相同的文件，同时 seeder 仅为其中一些种子播种，那么其它种子的下载者仍然可以下载到这个文件。

V2 版本的磁力链接也支持混合模式，以确保支持旧客户端。
