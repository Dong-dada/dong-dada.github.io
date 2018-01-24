---
layout: post
title:  "Java 基础知识"
date:   2018-01-22 16:08:30 +0800
categories: java
---

* TOC
{:toc}

最近在学 Java, 这篇文章大多来自于 《Java 核心技术》(第 10 版) 这本书。

## 基本数据类型

Java 中有 8 种基本数据类型：

| type    | size   | range                            |
| :------ | :----- | :------------------------------- |
| int     | 4 byte | -2147483648 - 2147483647 (20 亿) |
| short   | 2 byte | -32768 ~ 32767                   |
| long    | 8 byte | 很大                             |
| byte    | 1 byte | -128 ~ 127                       |
| float   | 4 byte | ± 3.40282347E+38F                |
| double  | 8 byte | ± 1.79769313486231570E+308       |
| char    | 2 byte | \u0000 ~ \uffff                  |
| boolean | 1 bit  | true or false                    |

注意：
- Java 中没有无符号 (unsigned) 形式的 int, long, short, byte 类型；
- Java 中可以用 0xFF 来表示十六进制数 ，用 0b101010 来表示二进制数；
- 一般而言，一个 char 代表一个 Unicode 字符，某些特殊的 Unicode 字符需要两个 char 来表示，具体可以参考之后的介绍；
- boolean 和 int 类型之间不能进行相互转换，也就是说 `if (1)` 这样的语句不能通过编译。

### char 与 Unicode

在 Unicode 设计之初，人们认为两个字节的代码宽度足以对世界上各种语言的所有字符进行编码，因此 1991 年发布的 Unicode 1.0 中只占用了 65536 个代码值中的不到一半。

所以设计 Java 的时候，将 char 用两个字节来表示。

经过一段时间后， Unicode 的字符超过了 65536 个，其主要原因是增加了大量的汉语、日语和韩语中的表意文字。因此原有的 16 位的 char 就不能满足描述所有 Unicode 字符的需要了。

在 Unicode 标准中，**码点** 用来表示某个字符在编码表中的位置。 Unicode 把所有码点分为 17 个代码级别 (code plane)，也称为 代码平面。

第一个代码级别被称为 **基本的多语言级别(basic multilingual plane)**，码点范围是 `U+0000 ~ U+FFFF`。剩下的 16 个代码级别，其范围是 `U+10000 ~ U+10FFFF`。

需要注意的是，在基本级别中，并不是所有的编码值都对应于一个 Unicode 码点，其中还包含了一些辅助字符。 比如 `D800 ~ DBFF` 表示的是 UTF-16 的高半区，`DC00-DFFF` 表示的是 UTF-16 的低半区。

UTF-16 编码采用不同长度的编码表示所有 Unicode 码点。对于基本的多语言级别，每个字符用 2 个字节表示，通常被称为 **代码单元**(code unit)。 对于更高级别，则需要采用两个连续的代码单元(也就是 4 个字节)来进行编码。这时候之前说过的 `D800 ~ DBFF`, `DC00 ~ DFFF` 这些空闲区域就派上用场了，比如 `U+1D546` 这个特殊字符，可以用两个代码单元 `U+D835` 和 `U+DD46` 来表示。

对于 Java 而言，一个 char 字符固定为 2 个字节大小，它表示了 UTF-16 编码中的一个代码单元。换句话说，对于高级别的字符来说，需要两个 char 来表示。

我们强烈建议不要在程序中使用 char 类型，除非确实需要处理 UTF-16 代码单元。

上述内容可参考原文，以及 [维基百科](https://zh.wikipedia.org/wiki/Unicode%E5%AD%97%E7%AC%A6%E5%B9%B3%E9%9D%A2%E6%98%A0%E5%B0%84)

## 字符串

从概念上讲，Java 字符串就是 Unicode 字符序列 （UTF-16 编码）。

Java 没有内置的字符串类型，而是通过标准库提供了一个 `String` 类(类似于 C++ 的 `std::string`)。

```java
// String 对象
String s;       // s == null
String e = "";  // e != null, 空字符串
String greeting = "Hello";

// 子串
s = greeting.substring(0, 3);

// 拼接
String expletive = "Expletive";
String PG13 = "deleted";
String message = expletive + PG13;
message = "Expletive" + PG13;
message = "Hello" + " " + "World!";

// join
String all = String.join("/", "S", "M", "L", "XL");
// all 的值为 S/M/L/XL

// 检查字符串是否相等
if (greeting.equals(message)) {}
if (greeting.equals("Hello")) {}
if (greeting.equalsIgnoreCase("hello")) {} // 忽略大小写
if ("Hello".equals(greeting)) {}           // 字符串也可以调用 .equals

// 不要使用 == 来比较字符串
if ("hello" == greeting) {
    // probably false
}

// 返回代码单元数量，也就是 char 的数量
int n = greeting.length();

// 获取指定位置的代码单元，也就是 char
char first = greeting.charAt(0);
char last = greeting.charAt(greeting.length() - 1);

// 返回码点数量，可能小于 length()
int cpCount = greeting.codePointCount(0, greeting.length());

// 获取第 n 个码点，码点用 int 而不是 char 来表示
int offset = greeting.offsetByCodePoints(0, n);
int cp = greeting.codePointAt(offset);

// 返回一个代表码点的 int 数组
int[] codePoints = str.codePoints().toArray();

// 用码点数组构造 String
String str = new String(codePoints, 0, codePoints.length);
```

注意：
- String 对象是不可变的，一旦构建了一个 String 对象，那么其包含的字符串就无法被修改；
- 不要用 `==` 来比较两个字符串，因为 `==` 只会判断两个变量是否引用了同一位置的字符串，有可能两个字符串的内容相同，但存储在不同的地方，此时 `==` 仍然会返回否；
- 码点和代码单元的区别，上一小节已经介绍过了，这里不再重复；

### 使用 StringBuilder 来构建字符串

有时候，需要由
