---
layout: post
title:  "Protocol Buffer - Encoding"
date:   2020-05-09 11:00:30 +0800
categories: cpp
---

* TOC
{:toc}

翻译了一下 [这篇文档](https://developers.google.com/protocol-buffers/docs/encoding)

## 整形

首先需要理解 varints, 也就是用不同长度的 bytes 保存数字，数字越小所需的 byte 数量也越小。

varints 的规则如下：
- 对于一个 varints 来说，每个 byte 的最高位是一个标记位，剩余 7 个 bit 才存储了实际数据。最高位如果是 1，那么表示后面还有 byte，反之如果是 0，那么表示这就是最后一个 byte 了。
- 对于一个 varints 来说，多个 bytes 按照低位优先规则来排序，类似于小端字节序。

比如如下两个字节:

```
10101100 00000010
```

第一个字节的最高有效位是 1, 说明它后面还有字节，第二个字节的最高有效位是 0，表示这个字节就是最后一个字节了。

要得到它实际代表的数字，需要先去掉最高位，然后调整字节序，再把剩余的 7 个 bit 拼起来：

```
10101100 00000010
-> 0101100 0000010 // 去掉最高位
-> 0000010 0101100 // 调转方向，低位放后面
-> 10 ++ 0101100   // 拼接
-> 100101100       // 256 + 32 + 8 + 4 = 300
```


## Message 结构

Message 的二进制格式由一组 key value 构成。

其中的 key 包含了两个信息：
- 这个 key 的 field number 是多少。解码时需要根据 field number 以及 .proto 定义来决定这个 kv 是否能识别。
- value 的 wire type 是什么。解码时需要根据 wire type 来判断 value 的长度。

key 使用 varints 表示，其构成方式是 `(field_number << 3) | wire_type`，就是说最后三个 bit 表示了 wire type，剩余 bit 表示了 field_number。

举例来说，一个消息的 `.proto` 定义为：

```proto
message Test1 {
  optional int32 a = 1;
}
```

如果将 a 字段设置为 150，那么这个消息被序列化之后用二进制可以表示为：

```
00001000 10010110 00000001
```

首先解码 key:

```
00001000
-> 0001000            // 去掉最高位
-> 0001000 >> 3       // 右移 3 位，获取 field number，结果为 1
-> 0001000 & 0000111  // 截取低 3 位，获取 wire_type, 结果为 0
```

wire type 不同值代表的含义是：

| Type | Meaning | Used For |
|----|----|----|
| 0 | Varint | int32, int64, uint32, uint64, sint32, sint64, bool, enum |
| 1 | 64-bit | fixed64, sfixed64, double |
| 2 | Length-delimited | string, bytes, embedded messages, packed repeated fields |
| 3 | Start group | groups (deprecated) |
| 4 | End group | groups (deprecated) |
| 5 | 32-bit | fixed32, sfixed32, float |

可以看到 wire type 为 0 时，表示 value 是一个 varint 类型，接下来解码 value:

```
10010110 00000001
-> 0010110 0000001   // 去掉最高位
-> 0000001 0010110   // 调转方向，低位放前面
-> 10010110          // 128 + 16 + 4 + 2 = 150
```


## 有符号整形

对于存储有符号整形，ProtoBuf 中提供了两种实现:
- 一种就是普通的 int32, int64，其二进制格式是用 varints 存储的。这导致在保存负数的时候 varints 无法节省内存，因为负数的最高位是 1，那么对于 int64 来说，不管存储的数字是 -1 还是 -1000，总是需要用 10 个 bytes 的 varints 才能保存下这个数字。
- 另一种是 sint32, sint64，其二进制格式也是用 varints 存储的，但是会先进行 ZigZag 编码，把负数保存成正数。比如 -1 会先转换成 1，这样只需 1 个 byte 的 varints 就可以存储。

ZigZag 的编码方式是：

```
// sint32
(n << 1) ^ (n >> 31)

// sint64
(n << 1) ^ (n >> 63)
```

可以理解为把数字左移一位，然后把原来的最高位放到最低位上。下面是一些例子：

|Signed Original | Encoded As |
| --- | --- |
|0 | 0 |
|-1 | 1 |
|1 | 2 |
|-2 | 3 |
|2147483647 | 4294967294 |
|-2147483648 | 4294967295 |

根据以上原理，在保存负数的时候，int64 一定会占用 10 个 bytes, 而 sint64 在多数情况下会占用更少空间。

在保存正数的时候，sint64 因为会左移一位，所以在特定情况下会比 int64 多占用 1 byte 空间。


## 定长数字

`double`, `float`, `fixed32`, `fixed64` 这些类型，占用的空间是定长的。


## 字符串

字符串 value 由两部分组成：
- 开头是一个 varints，指明字符串长度
- 剩余字节是 UTF-8 编码的字符串

比如一个消息的 `.proto` 定义为

```proto
message Test2 {
  optional string b = 2;
}
```

将 b 字段设置为 "testing"，那么这个消息被序列化后使用十六进制可以表示为：

```
12 07 [74 65 73 74 69 6e 67]
```

首先解码 key:

```
0x12
-> 00010010           // 表示为二进制
-> 0010010            // 去掉最高位，只有一个字节，不需要调转方向和拼接
-> 0010010 >> 3       // 右移 3 位，获取 field number，结果为 2
-> 0010010 & 0000111  // 截取低 3 位，获取 wire_type, 结果为 2
```

wire_type == 2，表示 value 是个 string, 接着解码 string 的长度：

```
0x07
-> 00000111           // 表示为二进制
-> 0000111            // 去掉最高位，只有一个字节，不需要调转方向和拼接
```

得知 string 长度为 7，那么剩下的字节就是 UTF-8 编码的字符串内容了：

```
[74 65 73 74 69 6e 67]   // testing 的 UTF-8 编码
```


## 内嵌消息

内嵌消息的 value 跟字符串的实现方法是一样的：
- 开头是一个 varints，指明字符串长度
- 剩余字节内嵌消息的二进制内容

比如一个消息的 `.proto` 定义为

```proto
message Test3 {
  optional Test1 c = 3;
}
```

将 c 字段设置为一个 Test1 消息，就是我们之前在整形的例子里所使用的那个 Test1 消息。其十六进制格式为：

```
1a 03 [08 96 01]
```

从 `0x1a` 可以解析出 wire type 是 3，第二个字节 `0x03` 表示内嵌消息的长度也是 3，剩余的几个字节 `[08 96 01]` 就是内嵌消息的二进制内容了。



## Optional 和 Repeated

optional 的规则很简单，有值的话就写入 key value pair，没值的话就不写入。

repeated 则有两种规则：
- 列表里的每一项，都被编码为一个 key value pair，就是说在二进制格式中，同一个 repeated 字段在消息内会有多个相同的 key
  - 这种规则在 proto2 里是默认行为，可以通过 [packed=true] 选项修改。
- 一个 repeated 字段在消息内只会有一个 key value pair，所有的条目都被保存在一个 value 里。
  - 这种规则仅用于数字类型，可以减少重复 key 导致的内存占用
  - 这种规则在 proto2 里需要通过 [packed=true] 选项来指定。
  - 这种规则在 proto3 里是默认行为。

以下消息需要使用第二种规则进行编码：

```proto
message Test4 {
  repeated int32 d = 4 [packet = true];
}
```

假设将字段 d 设置为 [3, 270, 86942] 数组，那么这个消息编码后用十六进制表示为：

```
22        // key (field number 4, wire type 2)
06        // payload size (6 bytes)
03        // first element (varint 3)
8E 02     // second element (varint 270)
9E A7 05  // third element (varint 86942)
```


# Map

map 等价于 repeated message:

```proto
message MapFieldEntry {
  key_type key = 1;
  value_type value = 2;
}

repeated MapFieldEntry map_field = N;
```

# 字段顺序

字段的 field number 不影响序列化后的二进制格式，并不是 field number 越小，二进制中的字段就越靠前。