---
layout: post
title:  "AES 加解密算法"
date:   2018-03-02 14:04:30 +0800
categories: algorithm
---

* TOC
{:toc}

## 简介

AES 全称 Advanced Encryption Standard, 高级加密标准。相对于 DES(Data Encryption Standard) 而言安全度更高。

AES 的密钥长度有三个版本，分别为 128 bit, 192 bit, 256 bit. 与此对应 DES 的密钥长度为 56 bit.

AES 属于对称加密，也就是双方使用同一个密钥来进行加解密。

AES 属于分组密码，也就是把明文按照一定大小分成若干组，然后对每组进行加密，最后把所有密文分组合并起来(合并不一定是直接合并，根据模式不同可能中间会有一些运算)。

AES 分组长度固定为 128 bit, 也就是 16 byte.

如果明文不是分组长度的整数倍，可能会用特殊字符进行填充。

AES 存在多种模式可以选择，模式不同，加密方法也不同。


## 参考
[密码算法详解--AES](https://www.cnblogs.com/luop/p/4334160.html)
