---
layout: post
title:  "Java8 时间"
date:   2018-04-23 17:59:30 +0800
categories: server
---

* TOC
{:toc}


## Java8 转换 UTC 时间到 Date

Java8 以前的 java.util.Date 类没有提供时区相关的能力，过去得靠 java.util.Calendar, java.util.TimeZone 来转换。

在 Java8 中，有两个类可以用来表示时间：
- LocalDateTime: 不包含时区的时间类；
- ZonedDateTime: 包含了时区的时间类；

理论上来说用 ZonedDateTime 表示时间是比较科学的，因为其中包含了时区信息，在国际化的时候不会有问题。不过有时候没办法，得转成 java.util.Date, 转换方式如下：

```java
// 获取当前的 UTC 时间，不带时区，比如 2018-04-24-00:00:00
LocalDateTime time = LocalDateTime.now(ZoneOffset.UTC);

// 给时间加上当前时区，比如 2018-04-24-00:00:00 UTC+8
ZonedDateTime zonedTime = time.atZone(ZoneId.systemDefault());

// 转成 instant, 转换时会加上时区
Instant zonedInstant = zonedTime.toInstant();

// 转成 Date
Date date = Date.from(zonedInstant);
```

上述转换的关键在于，好像没有办法重设 ZonedDateTime 的时区，也就是说，你不能直接把它的时区从 `UTC+08:00` 变成 `UTC+00:00`. 所以做法是先构造一个按照 UTC 计时的 LocalDateTime, 它的值相当于 