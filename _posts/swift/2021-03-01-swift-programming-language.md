---
layout: post
title:  "Swift - 基础知识"
date:   2021-03-01 12:49:59 +0800
categories: swift
---

* TOC
{:toc}

Swift 采用了以下现代编程模式来避免错误：
- 变量必须经过初始化方能使用；
- 数组索引会进行越界检查；
- 整数会进行越界检查；
- Optionals 保证了 nil 值必须被显式处理；
- 自动管理内存；
- 错误处理机制允许程序从 unexpected failures 中恢复；


# 特点

Swift 里的语句不需要分号结尾：

```swift
print("Hello world")
```

Swift 是强类型语言，换句话说变量是有类型的，声明为整型的变量，不能保存字符串值。

```swift
var count: Int32 = 10
count = "Hello"           // 编译错误
```


# 简单值

使用 `let` 声明常量，使用 `var` 声明变量。注意这里说的常量(constant), 是说声明了之后不可以被改变，并不要求一定要在编译期确定常量的值。

```swift
var myVariable = 42;
myVariable = 43;

let myConstant = 42;
```

上面的代码没有显式指明变量类型，类型是由编译器推断出来的。你也可以参考以下代码指明类型：

```swift
let implicitInteger = 70
let implicitDouble = 70.0
let explicitDouble: Double = 70
```

Swift 里面没有隐式类型转换的能力，必须进行显式转换：

```swift
let label = "The width is "
let width = 94
let widthLabel = label + String(width)

print(widthLabel)      // The width is 94
```

下面的语法可以让你比较简单地拼接字符串：

```swift
let apples = 3
let oranges = 5
let appleSummary = "I have \(apples) apples."
let fruitSummary = "I have \(apples + oranges) pieces of fruit."
```

使用三个双引号(`"""`) 可以比较方便地声明多行字符串：

```swift
let quotation = """
I said "I have \(apples) apples."
And then I said "I have \(apples + oranges) pieces of fruit."
"""
```