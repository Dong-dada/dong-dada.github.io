---
layout: post
title:  "算法 - 一致性哈希"
date:   2021-05-08 20:18:30 +0800
categories: algorithm
---

* TOC
{:toc}


看起来这个算法在分布式 cache 场景下用得比较多。

在这种场景下，有N 个缓存服务器提供服务，每个服务器负责保存 1/N 份数据。

现在有一个 key ，怎么决定要把它保存到哪个服务器呢？一种做法是对这个 key 做 hash 然后对 N 取余:
> hash(object) % N

这种方法有个问题是，如果服务器数量发生了变化，不管是新增还是较少，所有的 hash 结果都会变化，导致所有缓存都失效了。

一致性哈希的作用是尽量减少失效情况：

![]( {{site.url}}/asset/consistent-hash-1.png )

如上图：
- 首先把整个 hash 值的范围 0 ~ 2^32 首尾相连构成一个环
- 然后对每个节点，也就是上面说的缓存服务器做一下 hash 运算，这个运算可以基于服务器名称、id 之类的标识。把运算结果映射到环上。
- 接着对 key 做 hash ，结果映射到环上
- 某个 key 要保存到哪个服务器，取决于它的在顺时针方向离哪个 node 最近。

有了上述结构之后，假设新增了一台服务器，那就把它加到环里：

![]( {{site.url}}/asset/consistent-hash.jpg )


可以看到这种场景下，受影响的只有 node2 和新增的 node5 之间的 key，其它key 不受影响，仍然有效。

总的来说这种算法在计算 hash 时能够排除节点数量 N 的影响，降低 key 失效的范围。

这个算法有个问题是，如果节点数量很少，或者节点之间的距离都很近，这样就会导致大多数 key 都被保存到了某个特殊节点上。不像之前提的那种 hash 方式，结果会比较均匀。解决这个问题的方式是搞几个虚拟节点出来，比如把 node1 加几个后缀，node1_a, node1_b, node1_c，通过增加虚拟节点的方式让结果变得更平均:

调整前:

![]( {{site.url}}/asset/consistent-hash-not-balance.png )

增加虚拟节点后:

![]( {{site.url}}/asset/consistent-hash-virtual-node.png )