---
layout: post
title:  "Swift - Opaque Type(不透明类型)"
date:   2021-04-06 13:18:59 +0800
categories: swift
---

* TOC
{:toc}

本文参考自 [这篇文章](https://docs.swift.org/swift-book/LanguageGuide/OpaqueTypes.html)

Opaque Type(不透明类型) 用在函数返回值上。所谓不透明的意思是，函数的返回值不需要指明具体的类型，只需指明它必须实现某个协议即可。


# Opaque type 用法

以下例子中，先定义了 `Shape` protocol, 随后实现了几种不同的 `Shape`:

```swift
protocol Shape {
    func draw() -> String
}

// 三角形
struct Triangle: Shape {
    var size: Int
    func draw() -> String {
        var result = [String]()
        for length in 1...size {
            result.append(String(repeating: "*", count: length))
        }
        return result.joined(separator: "\n")
    }
}

// 正方形
struct Square: Shape {
    var size: Int
    func draw() -> String {
        let line = String(repeating: "*", count: size)
        let result = Array<String>(repeating: line, count: size)
        return result.joined(separator: "\n")
    }
}

// 可以将指定形状翻转
struct FlippedShape<T: Shape>: Shape {
    var shape: T
    func draw() -> String {
        let lines = shape.draw().split(separator: "\n")
        return lines.reversed().joined(separator: "\n")
    }
}

// 可以拼接两个形状
struct JoinedShape<T: Shape, U: Shape>: Shape {
    var top: T
    var bottom: U
    func draw() -> String {
        return top.draw() + "\n" + bottom.draw()
    }
}
```

接着，利用以上形状编写一个可以够造出梯形的函数，并且使用 `some` 关键字指明该函数的返回值是一个实现了 `Shape` 协议的不透明类型:

```swift
func makeTrapezoid() -> some Shape {
    // 用一个三角形、一个翻转三角形、一个正方形 来构造出梯形
    let top = Triangle(size: 2)
    let middle = Square(size: 2)
    let bottom = FlippedShape(shape: top)
    let trapezoid = JoinedShape(
        top: top,
        bottom: JoinedShape(top: middle, bottom: bottom)
    )
    return trapezoid
}
let trapezoid = makeTrapezoid()
print(trapezoid.draw())
// *
// **
// **
// **
// **
// *
```

# 返回 OpaqueType 和返回 Protocol 的区别

## 提供者角度: Opaque Type 要求函数只能返回一种类型

OpaqueType 和 Protocol 在代码编写上有一个很大的不同，前者要求函数只会返回某种类型，而不会返回多种类型:

```swift
func flip<T: Shape>(_ shape: T) -> some Shape {
    // 编译通过，因为只会返回 FlippedShape 一种类型
    return FlippedShape(shape: shape)
}

func invalidFlip<T: Shape>(_ shape: T) -> some Shape {
    // 编译出错，因为这个函数既会返回 Square, 也会返回 FlippedShape
    if shape is Square {
        return shape
    }
    return FlippedShape(shape: shape)
}

func protocolFlip<T: Shape>(_ shape: T) -> Shape {
    // 编译成功，返回 protocol 的话，允许类型不一样
    if shape is Square {
        return shape
    }
    return FlippedShape(shape: shape)
}
```

从函数提供者的角度去看，protocol 比 opaque type 更灵活一些。


## 调用者角度: Opaque type 具有类型标识，而 protocol 没有

Opaque type 与 protocol 相比本质上的不同在于 opaque type 保留了类型标识，而 protocol 没有。如果你希望让返回的类型支持 `Self`, `associated type` 等依赖类型标识的特性，那就需要使用 opaque type.

比方说下面的例子，在 `Container` 这个 protocol 中定义了一个 associated type:

```swift
protocol Container {
    associatedtype Item
    var count: Int { get }
    subscript(i: Int) -> Item { get }
}

extension Array: Container { }
```

因为 `Container` 当中包含 associated type, 因此它无法作为返回值:

```swift
// 编译出错:
// Protocol 'Container' can only be used as a generic constraint 
// because it has Self or associated type requirements
func makeProtocolContainer<T>(item: T) -> Container {
    return [item]
}

// 编译成功
func makeOpaqueContainer<T>(item: T) -> some Container {
    return [item]
}
let opaqueContainer = makeOpaqueContainer(item: 12)
let twelve = opaqueContainer[0]
print(type(of: twelve))
// Prints "Int"
```



