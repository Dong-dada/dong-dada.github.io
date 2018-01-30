---
layout: post
title:  "Java 集合"
date:   2018-01-30 16:38:30 +0800
categories: java
---

* TOC
{:toc}

## 集合框架中的接口

Java 中的集合框架，做了一个设计，把接口和实现相分离。比如 `List<T>` 是个接口，`ArrayList<T>` 则是这个接口的实现。

先来看看 Java 集合框架中定义的接口有哪些：

![]( {{site.url}}/asset/java-collection-interface.png )

集合有两个基本接口，类似于数组的 Collection 和类似于 KV 表的 Map。


## 具体的集合类

![]( {{site.url}}/asset/java-collection-implementation.png )

上图中展示了集合框架中提供的集合类。一系列 AbstractXXX 看起来比较奇怪，它只是提供了 `Collection` 等接口的一些默认实现。比如 `contains` 方法，不依赖于具体存储是怎么样的，都可以通过遍历的方法来实现这个方法。 AbstractXXX 系列类就是帮你把这些可以实现的接口先用通用的方法实现一遍，如果你有更高效的方法，直接复写就可以了。

在 C++ 中我们常用 `std::vector`, `std::map`, `std::queue`, `std::list`，Java 中都提供了对应的类，可以按需选用。
