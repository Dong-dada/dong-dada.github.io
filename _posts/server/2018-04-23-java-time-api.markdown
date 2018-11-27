---
layout: post
title:  "Java8 时间"
date:   2018-04-23 17:59:30 +0800
categories: server
---

* TOC
{:toc}


## 时间的表示

现在见到过的时间表示形式有两种，一种是 `2018-06-15T11:44:16+08:00` 这种年月日形式，一种是 `1529033507624` 这种秒数或毫秒数形式。

时间可以看成是时间线上的一个点，要表示这个点，需要定义一个原点，比如大爆炸开始的瞬间，耶稣诞生的瞬间。

对于地球人而言，看时间不只是在时间线上比较谁先谁后，还得知道白天黑夜，表示这种时间还需要加上时区信息，也就是一个 offset, offset 的参照物是那一瞬间的格林威治时间。为啥是格林威治呢，因为本初子午线——也就是 0 度经线经过那里，为啥 0 度经线经过那里呢，可能是因为英国以前航海很厉害。

`2018-06-15T11:44:16+08:00` 这串时间的原点是 `0000-00-00T00:00:00Z` (`Z` 即 Zero), 他相对于格林威治时间


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


Instant 
代表时间线上的一个瞬间(An instantaneous point on the time-line). 可以用于表示时间戳.
内部维护了存储了一个 long 值来存储 epoch seconds, 以及一个 int 值来存储 nanosecond-of-second.

Instant 中存储 epoch seconds 是从 `1970-01-01T00:00:00Z`(Z 表示 Zero, 即相对于 UTC 是 0 偏移) 开始的秒数.

从这点上来说，Instant 隐含了时区是 UTC 这一信息。
