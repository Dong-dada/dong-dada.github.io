---
layout: post
title:  "洗牌算法"
date:   2017-04-22 15:12:30 +0800
categories: algorithm
---

* TOC
{:toc}

洗牌 Shuffle，也就是将一堆数字打乱顺序。

## 随机数生成器

洗牌算法必然需要使用到随机数生成器，一般我们使用的随机数发生器都是伪随机数发生器，它通过 [线性同余法](https://zh.wikipedia.org/zh-cn/%E7%B7%9A%E6%80%A7%E5%90%8C%E9%A4%98%E6%96%B9%E6%B3%95) 等方法生成概率均等的随机数序列。

随机数序列是否可用，有以下几点需要考虑：
- 是不是够乱；
- 一个随机数序列中，各数字出现的概率是不是均等的；
- 随机数的种子取值范围是否够大；

最后一点可能不太好理解，举个例子：一副牌(54张)具有多达 54!(2.3084 * 10^71) 种排序可能。对伪随机数发生器而言，如果使用的种子只有 32 位，那么只能产生 N^32(2.7327 * 10^55) 种可能的随机数序列。

```cpp
void shuffle(std::vector<unsigned int>& cards)
{
    // 先把原来的牌复制一遍
    std::vector<unsigned int> temp_cards = cards;

    // 把原来的牌清空，准备接收打乱后的牌
    cards.clear();

    // 每次洗牌都重设种子
    srand(GetTickCount());

    while(!temp_cards.empty())
    {
        int index = rand() % (temp_cards.size());
        cards.push_back(temp_cards[index]);
        temp_cards.erase(temp_cards.begin() + index);
    }
}
```

因为种子只有 32 位，所以洗牌的结果也只会有 N^32 种。当然，对于扑克牌程序来说这么多结果已经完全够用了。

另外注意不要每次循环都设置新的种子，假如用时间来作为种子的话，因为循环的速度很快，每次的种子其实都是同一个，并且获取到的值也都是一样的。

Windows 中提供了 RtlGenRandom 函数，这个函数的实现选取了 CPU 信息，进程线程 ID，QueryPerformanceCounter 等值作为随机数的熵进行计算，能够产生真随机数。

C++11 中提供了 `random_device` 方法用于生成真随机数。

一些在线服务也能够生成真随机数，例如 https://www.random.org/

另外还要考虑的一点是随机数序列的重复周期，常用的伪随机数生成器 MT19937 的周期为 2^19937 - 1，由于 2080! < 2^19937 - 1 < 2081!, 所以它最多能生成 n = 2080 个元素的所有排列。

## 洗牌算法

### Fisher–Yates Shuffle

也就是上一节展示的洗牌算法。它的基本思想是 **原始数组中随机抽取一个新的数字到新数组中**。

这种算法有个问题，每次从原始数组中移除元素，都需要移位操作，所以算法的复杂度是 O(N^2)。

### Knuth-Durstenfeld Shuffle

Knuth 和 Durstenfeld 等人对 Fisher 算法进行了改进，其基本思想是 **每次从未处理的数据中随机取出一个数字，然后把该数字和数组尾部的数字交换，即数组尾部存放的是已经处理过的数字** 它是一种原地交换的算法，不需要再开辟一个数组，也不需要进行移位，所以算法的复杂度是 O(N)：

```cpp
void shuffle(std::vector<unsigned int>& cards)
{
    srand(GetTickCount());

    for (int i = cards.size() - 1; i >= 0; --i)
    {
        int index = rand() % (i+1);
        std::swap(cards[index], cards[i]);
    }
}
```

## 参考

- [洗牌算法shuffle](http://www.tuicool.com/articles/qIjqQzb)
- [当随机不够随机：一个在线扑克游戏的教训](http://blog.jobbole.com/64897/)
- [如何测试洗牌算法](http://coolshell.cn/articles/8593.html)
- [音乐随机播放的算法是怎样的](https://www.zhihu.com/question/37302731)

如何测试洗牌算法 这篇文章中提到了另外几种洗牌的方式，我不是很理解为什么会出现的结果不均匀的现象。