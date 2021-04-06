---
layout: post
title:  "Swift - 泛型"
date:   2021-04-06 13:18:59 +0800
categories: swift
---

* TOC
{:toc}

本文参考自 [这篇文章](https://docs.swift.org/swift-book/LanguageGuide/Generics.html)


# 泛型函数

你可以在函数当中使用泛型:

```swift
func swapTwoValues<T>(_ a: inout T, _ b: inout T) {
    let temporary = a
    a = b
    b = temporary
}

var a = 10
var b = 20
swapTwoValues(&a, &b)
print(a, b)     // 20 10
```


# 泛型类型

你也可以在结构体、类、枚举等类型上使用泛型:

```swift
struct Stack<Element> {
    var items = [Element]()
    mutating func push(_ item: Element) {
        items.append(item)
    }
    mutating func pop() -> Element {
        return items.removeLast()
    }
}

extension Stack {
    // 扩展里可以直接使用泛型类型
    var topItem: Element? {
        return items.isEmpty ? nil : items[items.count - 1]
    }
}

var stackOfStrings = Stack<String>()
stackOfStrings.push("uno")
stackOfStrings.push("dos")
stackOfStrings.push("tres")
stackOfStrings.push("cuatro")
```


限制范型类型

```swift
// 要求范型类型必须实现 Equatable 协议
func findIndex<T: Equatable>(of valueToFind: T, in array:[T]) -> Int? {
    for (index, value) in array.enumerated() {
        if value == valueToFind {
            return index
        }
    }
    return nil
}
```


# Associated Types

协议上是不允许使用泛型的，但你可以在协议上使用 associated type，它和泛型有点儿像，只是它被用在协议上：

```swift
protocol Container {
    // Item 由实现者指定
    associatedtype Item

    mutating func append(_ item: Item)
    var count: Int { get }
    subscript(i: Int) -> Item { get }
}
```

遵从上述协议的类型，必须指明 Associated type 具体是什么类型：

```swift
struct IntStack: Container {
    // original IntStack implementation
    var items = [Int]()
    mutating func push(_ item: Int) {
        items.append(item)
    }
    mutating func pop() -> Int {
        return items.removeLast()
    }

    // 实现 Container 协议，并指明 Item 为 Int
    // 下面这一行不是必须的，只是为了看着方便，Swift 可以根据函数参数之类的信息自己推导出 Item = Int
    typealias Item = Int
    mutating func append(_ item: Int) {
        self.push(item)
    }
    var count: Int {
        return items.count
    }

    subscript(i: Int) -> Int {
        return items[i]
    }
}
```

Associated type 也可以和泛型一起使用：

```swift
struct Stack<Element>: Container {
    // original Stack<Element> implementation
    var items = [Element]()
    mutating func push(_ item: Element) {
        items.append(item)
    }
    mutating func pop() -> Element {
        return items.removeLast()
    }

    // 实现 Container 协议，Item 类型与泛型类型 Element 一样
    mutating func append(_ item: Element) {
        self.push(item)
    }
    var count: Int {
        return items.count
    }
    subscript(i: Int) -> Element {
        return items[i]
    }
}
```

另外你还可以给 associated type 添加一些限制：

```swift
protocol Container {
    // 要求 Item 必须遵照 Equatable 协议
    associatedtype Item: Equatable

    mutating func append(_ item: Item)
    var count: Int { get }
    subscript(i: Int) -> Item { get }
}
```


# Where 语句

## Where 语句与 Associated Type

where 语句可以让你对 Associated type 施加更多限制，比如要求必须遵守某种协议，两种类型必须相等：

```swift
// C1 和 C2 必须遵守 Container 协议
func allItemsMatch<C1: Container, C2: Container>
    (_ someContainer: C1, _ anotherContainer: C2) -> Bool
    // C1 和 C2 的 Item 类型必须相同
    // C1 的 Item 类型必须遵守 Equatable 协议
    where C1.Item == C2.Item, C1.Item: Equatable {

        // Check that both containers contain the same number of items.
        if someContainer.count != anotherContainer.count {
            return false
        }

        // Check each pair of items to see if they're equivalent.
        for i in 0..<someContainer.count {
            if someContainer[i] != anotherContainer[i] {
                return false
            }
        }

        // All items match, so return true.
        return true
}
```

以下是调用 allItemsMatch 泛型函数的实例，请注意它的两个参数尽管类型不同，但都满足了函数的限制要求：

```swift
var stackOfStrings = Stack<String>()
stackOfStrings.push("uno")
stackOfStrings.push("dos")
stackOfStrings.push("tres")

var arrayOfStrings = ["uno", "dos", "tres"]

if allItemsMatch(stackOfStrings, arrayOfStrings) {
    print("All items match.")
} else {
    print("Not all items match.")
}
// Prints "All items match.
```

## Where 语句与扩展

Where 语句也可以用在扩展当中：

```swift
extension Stack where Element: Equatable {
    func isTop(_ item: Element) -> Bool {
        guard let topItem = items.last else {
            return false
        }
        return topItem == item
    }
}
```

不难理解，你也可以在 Where 条件中要求 associated type 必须是特定类型：

```swift
// 只对 Item == Double 的情况做扩展
extension Container where Item == Double {
    func average() -> Double {
        var sum = 0.0
        for index in 0..<count {
            sum += self[index]
        }
        return sum / Double(count)
    }
}
print([1260.0, 1200.0, 98.6, 37.0].average())
```