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


# 打印

Swift 里的 print 可以打印各种类型：

```swift
print(42)
print("Hello world")
print(true)
print(["hello", "world"])
```

基于字符串拼接方法，可以比较简单地拼接要打印的内容：

```swift
let apples = 3
let oranges = 5
print("I have \(apples) apples.")
print("I have \(apples + oranges) pieces of fruit.")
```


# 变量

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

# Optional

Swift 中默认情况下变量必须有值，除非使用 `?` 将其声明为 Optional

```swift
var str: String
// 以下代码未经初始化而使用一个值，将导致编译错误
// print(str)

var optionalString: String?
print(optionalString)           // nil
print(optionalString == nil)    // true
```

## 解包

使用 `if let` 语法可以获取到 option 中的值(如果有的话):

```swift
var optionalName: String? = "John Appleseed"
if let name = optionalName {
    print("Hello, \(name)")     // Hello, John Appleseed
}

// `if let` 获取到的值只在 if 的作用域内有效，外面是用不了的
// print(name)
```

使用 `??` 语法可以实现 “如果有值则获取，否则返回默认值” 的逻辑：

```swift
let nickname: String? = nil
let fullName: String = "John Appleseed"
print("Hi \(nickname ?? fullName)")     // Hi John Appleseed
```

使用 `?.` 语法可以获取 optional 对象中的属性，或者调用成员函数。如果对象本身是 nil, 则什么都不做然后返回 nil；如果对象不为 nil, 则执行相应的操作，并将结果以 optional 返回:

```swift
let shape: NamedShape? = NamedShape(name: "optional shape")
print(shape?.name)              // Optional("optional shape")
print(shape?.description())     // Optional("A shape named `optional shape`")
```

使用 `!` 语法可以在你能确定 `optional` 一定有值的情况下，获取到 `optional` 中的值:

```swift
let str = "10"
let number = Int(str)
print(number!)      // 10
```

## 隐式解包

使用 `String?` 可以声明一个 optional，使用 `String!` 也可以声明一个 optional，后者的不同之处在于 swift 会尝试自动解包出其中的变量，不需要你手动解包，换句话说，你可以把 `String!` 当 `String` 来使用，尽管它是个 optional:

```swift
let assumedString: String! = "An implicitly unwrapped optional string."
let implicitString: String = assumedString // no need for an exclamation point
```

隐式解包有个前提，就是你必须确保这个变量在使用的时候一定是有值的，否则自动解包失败后，会抛出一个运行时错误。看起来有点儿奇怪，如果你能确定变量一定有值，干嘛要把它声明成 optional 类型呢？实际上有一些场景下，你必须把变量声明成 optional 类型，尽管它一定会有值，典型的场景就是类的构造:

```swift
class Country {
    let name: String
    var capitalCity: City

    init(name: String, capitalName: String) {
        self.name = name
        // error: 'self' used before all stored properties are initialized
        self.capitalCity = City(name: capitalName, country: self)
    }
}
```

以上代码会编译失败，因为 `self` 中的 `capitalCity` 尚未初始化，此时 `self` 不能作为参数传给 `City()`. 此时就可以把 `capitalCity` 声明为 `City!`, 让其成为一个 optional, 并且在后续可以像普通变量那样使用它。

```swift
class Country {
    let name: String
    var capitalCity: City!

    init(name: String, capitalName: String) {
        self.name = name
        // error: 'self' used before all stored properties are initialized
        self.capitalCity = City(name: capitalName, country: self)
    }
}
```

# 类型别名

也就是给类型起另外一个名字，有点像 C++ 里的 `using StringPtr = std::unique_ptr<std::string>;` 这种用法。使用 `typealias` 关键字就可以给类型起别名:

```swift
typealias AudioSample = UInt16
```


# 字符串

## Unicode 和 Character

Swift 的 String 类型基于 Unicode Scalar Value 来构建，每个 Unicode Scalar Value 在内存中占 21 bit 内存。这里的 Unicode Scalar Value 与 Character 类型不是一回事儿，并不是说一个 Character 就一定对应一个 Unicode 码。因为有一些字符是由两个 Unicode 码拼接起来的，比如字符 `é` 既可以用 "\u{E9}" 表示，也可以用 "\u{65}\u{301}" 表示:

```swift
let eAcute: Character = "\u{E9}"                // é
let combinedEAcute: Character = "\u{65}\u{301}" // e followed by ́
                                                // eAcute is é, combinedEAcute is é
```

String 提供的访问 Character 的方法，都考虑了上面的情况，即使是两个 Unicode Scalar Value 表示的字符，也只会对应一个 Character 值。

## 字符串拼接

可以使用 `+` 来拼接字符串:

```swift
var variableString = "Horse"
variableString += " and carriage"
```

下面的语法可以让你比较简单地拼接字符串：

```swift
let apples = 3
let oranges = 5
let appleSummary = "I have \(apples) apples."
let fruitSummary = "I have \(apples + oranges) pieces of fruit."
```

## 多行字符串

使用三个双引号(`"""`) 可以比较方便地声明多行字符串：

```swift
let quotation = """
I said "I have \(apples) apples."
And then I said "I have \(apples + oranges) pieces of fruit."
"""
```

下面这张图展示了多行字符串中空格的忽略方式:

![]( {{site.url}}/asset/multilineStringWhitespace.png )

## 转义字符

字符串字面值当中可以包含以下两种特殊字符
- 转义字符: `\0`, `\\`, `\t`, `\n`, `\r` , `\"`, `\'`
- Unicode 码: `\u{2665}`，`\u{24}`

你可以使用 `#" "#` 把字符串括起来，这样可以在字面量中禁用转义:

```swift
print(#"Line 1\nLine 2"#)   // Line 1\nLine 2
```

多行字符串中也可以使用 `# #` 语法:

```swift
let jsonStr = #"""
{
  "name":"dongdada",
  "age":12
}
"""#
print(jsonStr)
```


# 复合类型

## 数组

创建和访问数组元素，使用 `[]` 方式：

```swift
// 直接初始化 array
var beatles = ["John", "Paul", "George", "Ringo"]
let numbers = [4, 8, 15, 16, 23, 42]
let temperatures = [25.3, 28.2, 26.4]


// 使用下标访问数组元素
print(beatles[0])
print(numbers[1])
print(temperatures[2])


// 数组是可扩展的
beatles.append("Adrian")


// 显示声明数组内元素的类型
var scores = Array<Int>()
scores.append(100)
scores.append(80)


// 也可以写成以下语法
var albums = [String]()
albums.append("Folklore")


// 使用 count 属性获取数组内元素数量
print(beatles.count)


// 使用 remove(at:) 方法移除指定位置的元素
beatles.remove(at: 0)

// 使用 removeAll() 方法移除所有元素
beatles.removeAll()


// 使用 contains() 方法判断数组是否包含某个元素
let bondMovies = ["Casino Royale", "Spectre", "No Time To Die"]
print(bondMovies.contains("Frozen"))


// 使用 sorted() 方法对数组排序，该方法返回一个新的数组
let cities = ["London", "Tokyo", "Rome", "Budapest"]
let sortedCities = cities.sorted()
print(type(of: sortedCities))       // Array<String>


// 使用 reversed() 方法将元素逆序排列，注意这时候返回的并不是一个新的数组，而是数组的一个视图
let reversedCities = cities.reversed()
print(type(of: reversedCities))     // ReversedCollection<Array<String>>


// 使用 filter() 方法来过滤数组中的元素，将返回一个新的数组
let team = ["Gloria", "Suzanne", "Piper", "Tiffany", "Tasha"]
print(team.filter{ $0.hasPrefix("T") })


// 使用 map() 方法来对数组中的每个元素做转换，将返回一个新的数组
print(team.map{ $0.uppercased() })


// 将 filter(), sorted(), map() 等方法连在一起使用
let luckyNumbers = [7, 4, 38, 21, 16, 15, 12, 33, 31, 49]
let lines = luckyNumbers.filter{ $0 % 2 == 1 }.sorted().map{ "\($0) is a lucky number" }
lines.forEach{print($0)}
```

数组是支持自动扩容的，而不是固定长度：

```swift
shoppingList.append("blue paint")
```

## 字典

创建和访问字典，也是用 `[]` 方式：

```swift
// 创建空字典
let emptyDictionary = [String: Float]()

// 可以简写为以下形式，字典的类型会根据后续添加的元素类型推到出来
var nameToScoreDict = [:]
nameToScoreDict["dong-dada"] = 61

// 创建字典并初始化
var occupations = [
    "Malcolm": "Captain",
    "Kaylee": "Mechanic",
]

// 访问字典元素
occupations["Jayne"] = "Public Relations"

print(occupations)
```

## 集合

```swift
// 初始化 Set
let actors = Set(["Denzel Washington", "Tom Cruise", "Nicolas Cage", "Samuel L Jackson"])
print(actors)


// 显式指定 Set 中元素的类型
var names = Set<String>()
names.insert("Dongdada")
names.insert("Jhann")


// 使用 contains() 方法判断元素是否存在
print(names.contains("Dongdada"))
print(names.contains("PangPang"))


// 使用 sorted() 方法返回一个经过排序的数组
// 输出: ["Dongdada", "Jhann"]
print(names.sorted())
```

## 元组

创建和访问元组(tuple)，使用 `()` 方式，注意 tuple 中元素类型可以不同:

```swift
let http404Error = (404, "Not Found")

// 使用序号来访问元素
print(http404Error.0)
print(http404Error.1)

// 也可以使用以下语法来获取元素，并将元素保存到变量中
let (code, message) = http404Error
print(code)
print(message)
```

还可以在声明 tuple 的时候，指定每个元素的名称:

```swift
let http200Status = (statusCode: 200, description: "Ok")

// 可以使用名称来访问元素
print(http200Status.statusCode)
print(http200Status.description)
```

还可以在声明的时候，指定每个元素的类型:

```swift
var httpStatus: (Int, String) = (200, "Hello")
httpStatus = (404, "Not Found")
```


# 控制流

## if

可参考以下语法。

```swift
let score = 59
if score >= 100 {
    print("Perfect!")
} else if score >= 80 {
    print("Good!")
} else if score >= 60 {
    print("Okay~")
} else if score >= 30 {
    print("Umm...")
} else {
    print("Get out!")
}
```

值得注意的是 swift 的 condition 必须是个布尔表达式，其它类型(比如数字 0 或 1)是不能作为 condition 的。

## switch

Swift 中的 `switch` 关键字有点特别：
- 可以适合于各种类型，而不仅仅是整型。
- 每个 case 默认是 break 的。
- 可以用逗号把两个 case 写在一起，也就是这两个 case 都执行相同操作。
- 可以在 case 中使用 `let x where x.hasSuffix("xxx")` 这样的语法，来自定义比较条件。

```swift
let vegetable = "red pepper"
switch vegetable {
case "celery":
    print("Add some raisins and make ants on a log.")
case "cucumber", "watercress":
    print("That would make a good tea sandwich.")
case let x where x.hasSuffix("pepper"):
    print("Is it a spicy \(x)?")
default:
    print("Everything tastes good in soup.")
}
// Prints "Is it a spicy red pepper?"
```

## for in

使用 `for in` 语法可以遍历数组或字典：

```swift
let interestingNumbers = [
    "Prime": [2, 3, 5, 7, 11, 13],
    "Fibonacci": [1, 1, 2, 3, 5, 8],
    "Square": [1, 4, 9, 16, 25],
]
var largest = 0
for (_, numbers) in interestingNumbers {
    for number in numbers {
        if number > largest {
            largest = number
        }
    }
}
print(largest)  // 25
```

注意上述代码在遍历的过程中使用了 `_` 来忽略字典中的 key。


## for in range

在 `for in` 中可以使用 `..<` 来构造一个数值区间：

```swift
var total = 0
for i in 0..<4 {
    total += i
}
print(total)    // 6
```

`..<` 表示的是左闭右开区间，`...` 则可以表示左闭右闭区间。


## for in one-sided range

遍历数组元素时，可以使用 one-sided range 来指定数组的范围:

```swift
for name in names[2...] {
    print(name)
}

for name in names[...2] {
    print(name)
}

for name in names[..<2] {
    print(name)
}
```


## while 和 repeat while

```swift
var n = 2
while n < 100 {
    n *= 2
}
print(n)    // 128

var m = 2
repeat {
    m *= 2
} while m < 100
print(m)    // 128
```


# 函数和闭包

使用 `func` 关键字可以声明一个函数:

```swift
func greet(person: String, day: String) -> String {
    return "Hello \(person), today is \(day)."
}
let result = greet(person: "Bob", day: "Tuesday")
print(result)       // Hello Bob, today is Tuesday.
```

## 参数标签 (argument label)

有点特别的是在 swift 里面，函数的参数是有标签的，参数的标签放在参数名前面：
- 参数名只用在函数内部，标签则用在调用方。
- 你可以把标签设置为 `_`，这时候调用函数就不需要指定标签了。

```swift
func greet(_ person: String, on day: String) -> String {
    return "Hello \(person), today is \(day)."  // 函数内部使用参数名
}
let result = greet("John", on: "Wednesday")     // 调用函数时使用标签名
print(result)
```

简单来说标签这个东西的好处在于可以让函数参数在内部使用和外部使用的时候更符合语义，比如下面的代码:

```swift
func setAge(for person: String, to value: Int) {
    print("\(person) is now \(value)")
}

setAge(for: "Paul", to: 14)
```

`setAge(for: "Paul", to: 14)` 相比于 `setAge(person: "Paul", value: 14)` 更容易读懂。

[这篇文档](https://swift.org/documentation/api-design-guidelines/#argument-labels) 中针对参数标签的命名提供了一些建议，目的是让调用方调用这个函数时语义更清晰：
- 如果没有必要在语义上对参数做区分，那么不要有标签，比如:
    - `min(1, 2)` 比 `min(number1, number2)` 更好
- 如果要执行类型转换，可以考虑省略第一个参数的标签，比如:
    - `String(10, radix: 16)` 比 `String(number: 10, radix: 16)` 更好
- 但也有例外，如果所做的类型转换会 "缩小" 类型，就不要省略掉参数标签，比如:
    - `UInt32(UInt16(10))` 这个没什么问题，`UInt16` -> `UInt32` 不会截断。
    - `UInt32(truncating: UInt64(10))` 注意这里加上了 `truncating` 标签，指出这次转换会截断。
    - `UInt32(saturating: UInt64(10))` 注意这里加上了 `saturating` 标签，指出这次转换会取最接近的值。
- 如果第一个参数是介词短语的一部分，那么给这个参数一个标签，比如:
    - `a.moveTo(x: b, y: c)`
    - `x.removeBoxes(havingLength: 12)` 表示移除掉长度包含 12 的 box。
- 如果第一个参数是语法短语的一部分，那么不要加标签：
    - `x.addSubview(y)`
- 换言之如果第一个参数不是语法短语的一部分，那么就应该加上标签：
    - `view.dismiss(animated: false)`
    - `let text = words.split(maxSplits: 12)`
- 其它参数都应该加上标签


## 使用 tuple 返回多个值

对于多个返回值，可以用索引或者名称来访问。

```swift
func calculateStatistics(scores: [Int]) -> (min: Int, max: Int, sum: Int) {
    var min = scores[0]
    var max = scores[0]
    var sum = 0

    for score in scores {
        if score > max {
            max = score
        } else if score < min {
            min = score
        }
        sum += score
    }

    return (min, max, sum)
}
let statistics = calculateStatistics(scores: [5, 3, 100, 3, 9])
print(statistics.sum)   // 120
print(statistics.2)     // 120
```


## 内嵌函数

Swift 允许在函数体里内嵌函数：

```swift
func returnFifteen() -> Int {
    var y = 10
    func add() {
        y += 5
    }
    add()
    return y
}
returnFifteen()
```


## 函数作为返回值

Swift 里的函数是一等公民，你可以把内嵌函数作为返回值返回：

```swift
func makeIncrementer() -> ((Int) -> Int) {
    func addOne(number: Int) -> Int {
        return 1 + number
    }
    return addOne
}
var increment = makeIncrementer()
increment(7)
```


## 函数作为参数

也可以把函数作为参数:

```swift
func hasAnyMatches(list: [Int], condition: (Int) -> Bool) -> Bool {
    for item in list {
        if condition(item) {
            return true
        }
    }
    return false
}
func lessThanTen(number: Int) -> Bool {
    return number < 10
}
var numbers = [20, 19, 7, 12]
hasAnyMatches(list: numbers, condition: lessThanTen)
```


## 闭包

函数实际上是闭包的一种特例。闭包就是一堆可以在创建之后延迟调用的代码块。闭包有个重要的特性：闭包内的代码可以访问创建闭包的作用域内的变量和函数，即使调用闭包的时候已经离开了创建闭包的作用域。

```swift
func makeCounter() -> () -> Int32 {
    var count: Int32 = 0
    func addOne() -> Int32 {
        count += 1
        return count
    }
    return addOne
}

let counter = makeCounter()
print(counter())    // 1
print(counter())    // 2
print(counter())    // 3
```

## 匿名闭包

可以使用 `{}` 来创建一个匿名的闭包:

```swift
let numbers = [1, 2, 3, 4, 5]
let new_numbers = numbers.map({ (number: Int) -> Int in
    return number * 3
})
print(new_numbers)      // [3, 6, 9, 12, 15]
```

如果闭包的类型已经确定，那么你可以省略它的参数类型、返回值类型。如果闭包内只有一条语句，那么 `return` 关键字都可以省略：

```swift
let numbers = [1, 2, 3, 4, 5]
let new_numbers = numbers.map({ number in number * 3 })
print(new_numbers)      // [3, 6, 9, 12, 15]
```

甚至有更简单的方法，你可以用数字而不是名称来引用闭包的参数；如果闭包作为函数的最后一个参数，那么可以把闭包写在括号后面；如果闭包作为函数的唯一一个参数，那么括号可以省略：

```swift
var numbers:[Int] = [1, 2, 3, 4, 5]

// 使用数字来应用闭包的参数
let sortedNumbers1 = numbers.sorted(by: { $0 > $1  })
print(sortedNumbers1)

// 闭包作为函数的最后一个参数，可以放在括号的后面
let sortedNumbers2 = numbers.sorted() { $0 > $1 }
print(sortedNumbers2)

// 闭包作为函数唯一的参数，可以省略括号
let sortedNumbers3 = numbers.sorted { $0 > $1 }
print(sortedNumbers3)
```


# 类和对象

使用 `class` 关键字可以定义一个类，定义类成员的方法和定义变量、函数的方法一样。通过 `类名 + ()` 的方式可以实例化一个对象出来，默认情况下对象的成员都是 public 的:

```swift
class Shape {
    var numberOfSides = 0

    func simpleDescription() -> String {
        return "A shape with \(numberOfSides) sides."
    }
}

let shape = Shape()
shape.numberOfSides = 10

let description = shape.simpleDescription()
print(description)
```

类成员函数(包括 init 构造函数)允许同名，但参数列表不一样：

```swift
class NamedShape {
    var numberOfSides: Int = 0
    var name: String

    init(name: String) {
        self.name = name
    }

    func description() -> String {
        return "A shape with \(numberOfSides) sides."
    }

    func description(withName: Bool) -> String {
        if withName {
            return "A shape named `\(name)` with \(numberOfSides) sides."
        } else {
            return description()
        }
    }
}

let shape = NamedShape(name: "foo")
shape.numberOfSides = 10
print(shape.description())
print(shape.description(withName: false))
print(shape.description(withName: true))
```

构造函数通过 `init()` 进行定义，析构函数通过 `deinit` 进行定义(注意没有括号)，构造函数没有返回值，除此之外和普通函数的定义方式没有什么不同：

```swift
class NamedShape {
    var numberOfSides: Int = 0
    var name: String

    init(name: String) {
        // 如果成员变量没有在声明时初始化，那么必须在构造函数中初始化它，否则编译会失败
        // 使用 self 可以引用对象自身
        self.name = name
    }

    init(name: String, numberOfSides: Int) {
        self.name = name
        self.numberOfSides = numberOfSides
    }

    deinit {
        // 清理一些东西...
    }

    func simpleDescription() -> String {
        return "A shape with \(numberOfSides) sides."
    }
}

let shape = NamedShape(name: "triangle")
shape.numberOfSides = 3
print(shape.simpleDescription())
```

## 类的继承

继承的语法跟 C++, Java 是类似的，基类的函数默认就是可以覆写的，不像 C++ 那样需要有 virtual 关键字。覆写后的函数必须加上 `override` 关键字，否则会编译不过。

```swift
class NamedShape {
    var numberOfSides: Int = 0
    var name: String

    init(name: String) {
        self.name = name
    }

    func simpleDescription() -> String {
        "A shape with \(numberOfSides) sides."
    }
}

class Square : NamedShape {
    var sideLength: Double

    init(sideLength: Double, name: String) {
        self.sideLength = sideLength
        super.init(name: name)
    }

    func area() -> Double {
        sideLength * sideLength
    }

    override func simpleDescription() -> String {
        "A square with \(sideLength) side length."
    }
}

let shape = Square(sideLength: 4.2, name: "square")
print(shape.simpleDescription())
print(shape.area())
```

## Getter 和 Setter

不添加成员变量，也可以通过设置 getter, setter 来构造出新的属性:

```swift
class NamedShape {
    var numberOfSides: Int = 0
    var name: String

    init(name: String) {
        self.name = name
    }

    func simpleDescription() -> String {
        "A shape with \(numberOfSides) sides."
    }
}

class EquilateralTriangle : NamedShape {
    var sideLength: Double

    init(sideLength: Double, name: String) {
        self.sideLength = sideLength
        super.init(name: name)
        numberOfSides = 3
    }

    var perimeter: Double {
        get {
            return 3.0 * sideLength
        }
        set {
            // 在 setter 当中，有一个隐式的变量 newValue, 代表了要设置给属性的值
            // 也可以写成 `set(newPerimeter: Double) {}` 这样的形式，来显式命名这个变量
            sideLength = newValue / 3.0
        }
    }

    override func simpleDescription() -> String {
        "An equilateral triangle with sides of length \(sideLength)."
    }
}

var triangle = EquilateralTriangle(sideLength: 3.1, name: "a triangle")
print(triangle.perimeter)   // 9.3

triangle.perimeter = 9.9
print(triangle.sideLength)  // 3.3000000000000003
```

## willSet 和 didSet

可以按照以下语法，给成员的某个属性设置 `willSet` 和 `didSet`。属性发生变化前会触发 `willSet`，此时可以通过 `newValue` 拿到将要变成的新值；属性发生变化后会触发 `didSet`，此时可以通过 `oldValue` 拿到变化前的旧值：

```swift
class NamedShape {
    var numberOfSides: Int = 0
    var name: String {
        willSet {
            // 设置前该方法被调用，此时 name 表示当前值, newValue 表示新值
            print("Name will be change from \(name) to \(newValue)")
        }
        didSet {
            // 设置后该方法被调用，此时 name 表示当前值，oldValue 表示修改前的值
            print("Name has been change from \(oldValue) to \(name)")
        }
    }

    init(name: String) {
        self.name = name
    }
}

let shape = NamedShape(name: "hello")
shape.name = "dongdada"     // Name will be change from hello to dongdada
                            // Name has been changed from hello to dongdada
```


## 多继承

Swift 不支持多继承：

```swift
class Shape {
    var numberOfSides: Int = 0
}

class Color {
    var rgb = "#FFFFFF"
}

// 以下代码将导致编译失败
class ColorfulShape : Shape, Color {

}
```


# 枚举

使用以下语法可以创建一个枚举。

```swift
enum Gender {
    case male
    case female
}
```

多个枚举值可以用逗号分隔写在一个 case 后面：

```swift
enum Gender {
    case male, female
}
```

## rawValue

你可以为枚举值指定一个 rawValue，rawValue 的类型可以是整型、浮点型、字符串。指定的方式是在枚举名后面跟上 rawValue 类型，在枚举值后面跟上具体的值。如果是整型，那么 rawValue 默认从 0 开始；如果是浮点型，那么 rawValue 默认从 0.0 开始；如果是字符串，那么 rawValue 默认与字段名称一样。如果指定了 rawValue，这个枚举类型会提供一个 `init?(rawValue:)` 方法，供你从 rawValue 中构造出枚举值。

```swift
enum Gender : Int {
    case male = 0
    case female = 1
}
print(Gender.male.rawValue)         // 0
print(Gender(rawValue: 1))          // Optional(swift_demo.Gender.female)

enum BodyHeight : Float {
    case tall
    case short
}
print(BodyHeight.tall.rawValue)     // 0.0
print(BodyHeight(rawValue: 1.0))    // Optional(swift_demo.BodyHeight.short)

enum BodyType : String {
    case thin
    case fat
}
print(BodyType.thin.rawValue)       // thin
print(BodyType(rawValue: "fat"))    // Optional(swift_demo.BodyType.fat)
```

## 成员函数

枚举中可以有成员函数：

```swift
enum Suit {
    case spades, hearts, diamonds, clubs

    func simpleDescription() -> String {
        switch self {
        case .spades:
            return "spades"
        case .hearts:
            return "hearts"
        case .diamonds:
            return "diamonds"
        case .clubs:
            return "clubs"
        }
    }
}
```

上面的代码还展示了另外一个技巧，如果上下文中已经可以确定枚举类型，那么用到枚举值的时候可以把类型省略掉。

## 枚举值保存变量

此外，枚举值还能够用来保存变量：

```swift
enum ServerResponse {
    case result(String, String)
    case failure(String)
}

// 把 "6:00 am", "8:09 pm" 这两个字符串，保存到 ServerResponse.success 这个枚举值中
let success = ServerResponse.result("6:00 am", "8:09 pm")
let failure = ServerResponse.failure("Out of cheese.")
```

使用 `if case` 语法可以从枚举值中解析出相应变量:

```swift
let success = ServerResponse.result("6:00 am", "8:09 pm")
if case ServerResponse.result(let sunrise, let sunset) = success {
    print("Sunrise is at \(sunrise) and sunset is at \(sunset).")
}
```

使用 `case let` 语法，可以在 switch 语句中解析出相应变量:

```swift
func printResponse(_ response: ServerResponse) {
    switch response {
    case let .result(sunrise, sunset):
        print("Sunrise is at \(sunrise) and sunset is at \(sunset).")
    case let .failure(message):
        print("Failure...  \(message)")
    }
}

let failure = ServerResponse.failure("Out of cheese.")
printResponse(failure)
```


# 结构体

结构体跟类的形式差不多，但他们有一些关键的差别：
- 结构体在代码中总是以值传递，而类以引用传递。
- 结构体不支持继承，而 class 可以。

```swift
struct Card {
    var rank: Rank
    var suit: Suit
    func simpleDescription() -> String {
        return "The \(rank.simpleDescription()) of \(suit.simpleDescription())"
    }
}
let threeOfSpades = Card(rank: .three, suit: .spades)
let threeOfSpadesDescription = threeOfSpades.simpleDescription()
```

此外结构体还有一个限制，默认情况下类的成员函数都是可以修改类成员变量的，但结构体不行。如果需要修改成员变量，必须加上 `mutating` 关键字:

```swift
struct ServiceIdentifier {
    var name: String
    var version: String

    // 如果要修改成员变量，必须加上 mutating 关键字，否则编译失败
    mutating func setVersion(newVersion: String) {
        version = newVersion
    }
}

var id = ServiceIdentifier(name: "com.dada.ProductService", version: "1.0.0")
id.setVersion(newVersion: "1.0.1")
```


# Protocols 和 Extensions

这里翻译为协议和扩展。

## 协议

协议类似于 Java 里面的 interface, Rust 里面的 trait：

```swift
protocol ExampleProtocol {
    // 实现该协议的类，必须有一个 simpleDescription 属性，该属性必须可读
    var simpleDescription: String { get }

    // 实现该协议的类，必须有一个 adjust() 方法，该方法可以修改成员变量
    mutating func adjust()
}
```

类、结构体、枚举都可以实现协议：

```swift
class SimpleClass : ExampleProtocol {
    var simpleDescription: String = "A very simple class."

    // 类成员函数默认就可以修改成员变量，所以不需要加 mutating 关键字
    func adjust() {
        simpleDescription += "  Now 100% adjusted."
    }
}

struct SimpleStruct : ExampleProtocol {
    var simpleDescription: String = "A simple structure"

    mutating func adjust() {
        simpleDescription += " (adjusted)"
    }
}
```

## 扩展

你可以通过扩展的方式为一个已有的类型添加额外的功能，而不需要对已有类型的代码做修改。扩展看起来跟在类定义的地方实现协议很像，只是不需要把代码写到类定义上了。

你甚至可以扩展库里面的类型：

```swift
extension Int: ExampleProtocol {
    var simpleDescription: String {
        return "The number \(self)"
    }
    mutating func adjust() {
        self += 42
    }
}
print(7.simpleDescription)  // "The number 7"
```


# 错误处理

Swift 提供了一个名为 `Error` 的 protocol, 你可以用它来表示错误:

```swift
enum PrinterError : Error {
    case outOfPaper
    case noToner
    case onFire
}
```

你可以在函数中使用 `throw` 关键字来抛出错误，抛出错误的函数必须在声明中标记为 `throws`:

```swift
func send(job: Int, toPrinter printerName: String) throws -> String {
    if printerName == "Never Has Toner" {
        throw PrinterError.noToner
    }

    return "Job sent"
}
```

会抛出错误的函数，必须显式使用 `try` 关键字来调用:

```swift
let result = try send(job: 1, toPrinter: "Never Has Toner")
```

如果函数抛出了错误，但没有 `catch` 这个错误，那么程序会异常退出。可以使用 `do catch` 语法来捕获错误：

```swift
do {
    let result = try send(job: 1, toPrinter: "Bi Sheng")
    print(result)
} catch {
    print(error)
}
```

你可以指定要 `catch` 的异常类型。默认情况下捕获到的错误会保存在 `error` 这个隐式变量里面，你也可以显式指定变量名称:

```swift
do {
    let printerResponse = try send(job: 1440, toPrinter: "Gutenberg")
    print(printerResponse)
} catch let printerError as PrinterError {
    print("Printer error: \(printerError).")
} catch {
    print(error)
}
// Prints "Job sent"
```

另外 swift 还提供了 `try?` 语法，实现了 "如果没有错误就获取这个值，如果有错误就返回 nil" 的效果:

```swift
let printerSuccess = try? send(job: 1884, toPrinter: "Mergenthaler")
print(printerSuccess)       // Optional("Job sent")

let printerFailure = try? send(job: 1885, toPrinter: "Never Has Toner")
print(printerFailure)       // nil
```

此外还有类似的 `try!` 语法，表示 "一定没有错误，可以获取到目标值":

```swift
listener = try! NWListener(using: .tcp, on: 12345)
```

## defer

`defer` 关键字允许你在当前作用于的所有代码执行完毕后再执行一些清理逻辑。无论作用域内的代码是否抛出了错误，`defer` 代码都会被执行，有点像 C++/Java 里的 `finally`，但是 `defer` 不止可以用在异常上面，普通代码也可用:

```swift
var fridgeIsOpen = false
let fridgeContent = ["milk", "eggs", "leftovers"]

func fridgeContains(_ food: String) -> Bool {
    fridgeIsOpen = true
    defer {
        fridgeIsOpen = false
    }

    let result = fridgeContent.contains(food)
    return result
}

fridgeContains("banana")
print(fridgeIsOpen)     // false
```


# 泛型

使用以下语法可以声明泛型函数:

```swift
func makeArray<Item>(repeating item: Item, numberOfTimes: Int) -> [Item] {
    var result = [Item]()
    for _ in 0..<numberOfTimes {
        result.append(item)
    }
    return result
}

print(makeArray(repeating: "knock", numberOfTimes: 4))
```

类、枚举、结构体都可以设置泛型:

```swift
enum OptionalValue<Wrapped> {
    case none
    case some(Wrapped)
}

var possibleInteger: OptionalValue<Int> = .none
possibleInteger = .some(100)
```

在设置泛型的时候，可以使用 `where` 关键字对泛型类型做一些限制，比如要求必须实现某个协议，两个泛型类型必须相同，或者要求某个类必须继承另外一个类：

```swift
func anyCommonElements<T: Sequence, U: Sequence>(_ lhs: T, _ rhs: U) -> Bool
    where T.Element: Equatable, T.Element == U.Element {
    for lhsItem in lhs {
        for rhsItem in rhs {
            if lhsItem == rhsItem {
                return true
            }
        }
    }

    return false
}

print(anyCommonElements([1, 2, 3], [3]))
```


# Assertions 和 Preconditions

Assertions 和 Preconditions 都是断言，只不过前者只在 Debug 版本中生效，而后者在 Debug 和 Production 版本中都会生效。

你可以使用 `assert()` 来进行断言：

```swift
let age = -3
assert(age >= 0, "A person's age can't be less than zero.")
// This assertion fails because -3 isn't >= 0.
```

你可以使用 `precondition()` 来进行断言：

```swift
// In the implementation of a subscript...
precondition(index > 0, "Index must be greater than zero.")
```


# 实现 Comparable 协议

```swift
struct User: Identifiable, Comparable {
    let id = UUID()
    let firstName: String
    let lastName: String
    
    static func < (lhs: User, rhs: User) -> Bool {
        lhs.lastName < rhs.lastName
    }
}
```