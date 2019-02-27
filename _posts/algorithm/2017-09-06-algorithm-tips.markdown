---
layout: post
title:  "算法小知识点"
date:   2017-09-06 11:04:30 +0800
categories: algorithm
---

* TOC
{:toc}

## 有向无环图
[有向无环图](https://zh.wikipedia.org/wiki/%E6%9C%89%E5%90%91%E6%97%A0%E7%8E%AF%E5%9B%BE)

如果一个有向图从任意定点出发无法经过若干条边回到该点，则这个图是一个 有向无环图 (directed acyclic graph, 简称 DAG);

值得注意的是，如下图一个点 7 可以向两个方向出发到达另一个点 9，虽然形成了一个圈，但这并不是环(无法回到原来的点)，因此它还是有向无环图。

![]( {{site.url}}/asset/algorithm-directed-acyclic-graph.png )

这也说明有向无环图未必能够转化为树，但任何有向树都是有向无环图；