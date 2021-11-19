---
layout: post
title:  "SwiftUI - 小知识点"
date:   2021-11-02 20:03:59 +0800
categories: swift
---

* TOC
{:toc}


# 小知识点

## @State 属性包装器

View 都是 struct 类型，因为 SwiftUI 框架会经常性地销毁和重建 View，因此需要其尽量简单，搞成 class 的话可能会因为继承等原因导致 class 体积很大，使得 SwiftUI 重新创建它的成本变高。struct 是不允许继承的，没有隐式的成本。

这导致没法在 View 内部修改它的属性:

```swift
struct ContentView: View {
    var tapCount = 0

    var body: some View {
        Button("Tap Count: \(tapCount)") {
            // 报错: Left side of mutating operator isn't mutable: 'self' is immutable
            tapCount += 1
        }
    }
}
```

对于普通的 swift 代码来说，可以在 struct 里创建一个 `mutating func doSomeWork()` 方法，由 `mutating` 修饰的方法允许修改属性。但我们这里的 body 是个 computed property, swift 不支持在 computed property 上添加 `mutating` 修饰符，所以上面代码没法工作。

```swift
struct ContentView: View {
    @State var tapCount = 0

    var body: some View {
        Button("Tap Count: \(tapCount)") {
            self.tapCount += 1
        }
    }
}
```

只需为属性加上 `@State` 包装器，上面的代码就能正常工作了。这并没有破坏 swift 语言的规则，这个属性包装器会在其它地方存储这个属性，这样 SwiftUI 就能够修改它了。

更为重要的是，被 `@State` 所包装的属性一旦发生变化, body 这个 computed property 就会被调用，重新渲染这个 View。

值得注意的是， 如果你自己定义的 computed property 依赖了被 `@State` 包装的属性，每当后者发生变化，computed property 也会发生变化，并反映到视图上。换句话说，虽然你不能用 `@State` 去包装 computed property，但 computed property 的变化也能反映到视图上。

```swift

struct ContentView: View {
    @State private var checkAmount = 0.0
    @State private var numberOfPeople = 2
    @State private var tipPercentage = 20
    
    let tipPercentages = [10, 15, 20, 25, 0]
    var totalPerPerson: Double {
        // checkAmount 被 @State 所包装，发生变化时先重新计算 totalPerPerson，然后再调用 body 属性刷新界面
        let tipValue = checkAmount * Double(tipPercentage) / 100
        let grandTotal = checkAmount + tipValue
        return grandTotal / Double(numberOfPeople + 2)
    }
    
    var body: some View {
        NavigationView {
            Form {
                Section {
                    TextField("Amount", value: $checkAmount, format: .currency(code: Locale.current.currencyCode ?? "USD"))
                        .keyboardType(.decimalPad)
                }
                
                Section {
                    // totalPerPerson 这个 computed property 的变化将反映到界面上
                    Text(totalPerPerson, format: .currency(code: Locale.current.currencyCode ?? "USD"))
                }
            }
            .navigationTitle("WeSplit")
        }
    }
}
```

`@State` 修饰的属性只能是基本类型或者是结构体，对 class 无效。比如以下代码使用 `@State` 修饰了 User 类，当 User 的属性发生变化，Text View 并不会被更新。只有将 User 类型改为结构体 `@State` 才能生效。对于结构体而言，每当 TextField 修改了结构体中的一个字段，Swift 都会创建一个新的结构体实例，然后赋值给 user 属性。换句话说，虽然 TextField 修改的是结构体内部的字段，但实际上整个结构体都被重建了，user 属性也发生了变化，所以 `@State` 才能监听到这种变化。而如果 User 是 class 类型，那么 Swift 不会重建它，而是直接修改了它的字段，此时 user 属性本身在 `@State` 看来是没有发生变化的，因此界面不会被更新。

```swift
class User {
    var firstName = "Bilbo"
    var lastName = "Baggins"
}

struct ContentView: View {
    @State private var user = User()

    var body: some View {
        VStack {
            Text("Your name is \(user.firstName) \(user.lastName)")
            
            TextField("First name", text: $user.firstName)
            TextField("Last name", text: $user.lastName)
        }
    }
}
```


## @StateObject

使用 struct 作为 View 的属性有个问题，就是数据没法在多个 View 之间共享。如果有需要共享的数据，那么应该用 class 来描述，并且使用 `@StateObject` 来包装：

```swift
// 希望 User 存储的数据能够在多个 View 间共享，那么需要对这个 class 做两件事：
// 1. 实现 ObservableObject 协议
// 2. 按需要使用 '@Published' 包装属性，被包装的属性发生变化时，将通知外部(也就是 @StateObject 或 @ObservedObject)
class User: ObservableObject {
    @Published var firstName = "Bilbo"
    @Published var lastName = "Baggins"
}

struct ContentView: View {
    // 创建 User 实例
    // 如果是创建实例，那么用 @StateObject 包装，这样就可以监听实例的属性变化
    @StateObject var user = User()

    var body: some View {
        VStack {
            Text("Your name is \(user.firstName) \(user.lastName)")
            
            // 把实例传给其它 View
            UserNameTextField(user: user)
        }
    }
}

struct UserNameTextField: View {
    // 使用已有实例
    // 如果是使用已有实例，那么用 @ObservedObject 包装，这样就可以监听实例的属性变化
    @ObservedObject var user: User
    
    var body: some View {
        VStack {
            TextField("First name", text: $user.firstName)
            TextField("Last name", text: $user.lastName)
        }
    }
}
```

总的来说，`@Publish` 承担了原先 `@State` 的一半功能：发送变化事件给监听者；`@StateObject`, `@ObservedObject` 承担了 `@State` 的另一半功能：监听事件，随后更新界面。

`@StateObject` 和 `@ObservedObject` 功能差不多，区别只在于前者用于创建实例的时候，后者用于接收已有实例的时候。


## 双向绑定

除了根据属性显示内容，也可以反过来根据用户输入修改属性，这就被称为 "双向绑定"。

比如简单的输入框:

```swift
struct ContentView: View {
    
    private var name = ""
    
    var body: some View {
        Form {
            // 报错: Cannot convert value of type 'String' to expected argument type 'Binding<String>'
            TextField("Enter your name", text: name)
            Text("your name is \(name)")
        }
    }
}
```

TextField 的 text 参数期望是一个 `Binding<String>` 类型，以便将界面显示与 text 参数建立双向绑定。需要做两处修改来让普通属性变为 `Binding<T>` 类型：

```swift
struct ContentView: View {
    
    // 加入 @State 包装器，以便让 TextField 修改这个值
    @State private var name = ""
    
    var body: some View {
        Form {
            // 加上 '$' 符号，以便将属性转换为 Binding<T> 类型
            TextField("Enter your name", text: $name)

            // 不需要双向绑定，所以不用加 $
            Text("your name is \(name)")
        }
    }
}
```

## 结构体数组中元素属性的双向绑定

类似于以下代码的情况，有个结构体数组，使用这个结构体数组构建了一个 View 列表，每个 View 需要去双向绑定结构体数组元素的一个属性:

```swift
struct Question : Identifiable {
    var question: String
    var possibleAnswers : [UInt]
    var correctAnswer: UInt
    var userAnswer: UInt
    
    // 如果不想在 List() 里面声明 id 参数，就可以实现 Identifiable 协议，然后把某个属性定义成 id，这也是个小技巧
    var id: String {
        question
    }
}

struct ContentView: View {
    @State private var questions = [
        Question(question: "1 x 1 =", possibleAnswers: [1, 2, 3], correctAnswer: 1, userAnswer: 0),
        Question(question: "1 x 2 =", possibleAnswers: [2, 3, 4], correctAnswer: 2, userAnswer: 0),
        Question(question: "1 x 3 =", possibleAnswers: [3, 4, 5], correctAnswer: 3, userAnswer: 0),
        Question(question: "1 x 4 =", possibleAnswers: [4, 5, 6], correctAnswer: 4, userAnswer: 0)
    ]
    
    var body: some View {
        VStack {
            // 注意这里的语法，为了能够双向绑定数组元素的某个属性，需要在数组、元素上都加上 $ 符号
            List($questions) { $question in
                HStack {
                    Text("\(question.question)")

                    // 注意这里的语法，这样就可以双向绑定元素的某个属性
                    Picker("", selection: $question.userAnswer) {
                        ForEach(question.possibleAnswers, id: \.self) { answer in
                            Text("\(answer)")
                        }
                    }
                    .pickerStyle(.menu)
                    
                    Spacer()
                    Text(question.self.userAnswer == 0 ? "?" : question.self.userAnswer == question.self.correctAnswer ? "√" : "X")
                }
            }
        }
    }
}
```


## Modifier 的顺序

Modifier 的顺序先后对最后的显示效果是有影响的，每个 modifier 都将在之前的基础上创建一个新的 struct，而不是直接修改原有 struct 的属性。比如下面的例子先将 button 的背景设置为红色，随后将 button 的大小设置为 200*200:

```swift
Button("Hello, world!") {
    print(type(of: self.body))
}    
.background(.red)
.frame(width: 200, height: 200)
```

![]( {{site.url}}/asset/swiftui-modifier-order.png )

可以看到使用 `.frame()` 调整大小后，之前设置的红色并不会随之扩展。原因就是因为每个 modifier 都是在之前的基础上创建一个新的 struct, modifier 起到的并不是设置某个属性的效果，而是在原有基础上施加影响的效果。

查看 Button 点击后显示的类型为 `ModifiedContent<ModifiedContent<Button<Text>, _BackgroundStyleModifier<Color>>, _FrameLayout>`。其中的 ModifiedContent 也是一个实现了 View 协议的 struct. 也就是说，第一个 modifier `.background(red)` 创建了一个具有特定大小，为红色的 ModifiedContent View，并将原有的 Button 纳入其中。第二个 modifier `.frame()` 又创建了一个新的 ModifiedContent View，其大小为 200*200，但背景是默认的白色。

更明显的例子是，你可以多次使用相同的 modifier, 此时呈现的结果是这些 modifier 依次对之前的 struct 施加影响，而非覆盖之前的结果:

```swift
Button("Hello world") {
    print(type(of: self.body))
}
.padding()
.background(.red)
.padding()
.background(.blue)
.padding()
.background(.green)
.padding()
.background(.yellow)
```

![]( {{site.url}}/asset/swiftui-modifier-order-2.png )


## Environment Modifier

下面代码中为 VStack 设置的 `.font()` modifier 是一个 environment modifier，这意味着 VStack 中的每个 Text 都将被这个 modifier 所修改。这样可以节省在每个 Text 上都设置 modifier 的代码量。你也可以为某个 Text 特殊指定 modifier, 此时它将有更高的优先级:

```swift
struct ContentView: View {
    var body: some View {
        VStack {
            Text("Gryffindor")
                .font(.largeTitle)
            Text("Hufflepuff")
            Text("Ravenclaw")
            Text("Slytherin")
        }
        .font(.title)
    }
}
```

并不是所有的 modifier 都是 environment modifier, 比如下面代码的 `.blur()` 就不是。因此 Text 中的 `.blur()` 并不能影响 VStack 中的 `.blur()`:

```swift
VStack {
    Text("Gryffindor")
        .blur(radius: 0)
    Text("Hufflepuff")
    Text("Ravenclaw")
    Text("Slytherin")
}
.blur(radius: 5)
```

除了阅读文档或者亲自实验，没啥有效的办法可以直接区分出一个 modifier 是否是 environment modifier.


## 条件式的 modifier

可以通过三目元算符来让 modifier 根据条件选择不同的参数

```swift
struct ContentView: View {
    @State private var useRedText = false

    var body: some View {
        Button("Hello World") {
            // flip the Boolean between true and false
            useRedText.toggle()            
        }
        .foregroundColor(useRedText ? .red : .blue)
    }
}
```

也可以在 body 里面使用 `if ... else` 表达式，但这样做性能没有三目表达式好，三目表达式的话允许 SwiftUI 直接渲染原有 button 来提升性能，但 `if ... else` 的话就必须把原有对象销毁再创建一个新的。

```swift
var body: some View {
    if useRedText {
        Button("Hello World") {
            useRedText.toggle()
        }
        .foregroundColor(.red)
    } else {
        Button("Hello World") {
            useRedText.toggle()
        }
        .foregroundColor(.blue)
    }
}
```

## @ViewBuilder

虽然 body 属性的类型是 `some View`, 但是你可以直接在 body 里面放置两个 View，这是因为 body 属性经过了 `@ViewBuilder` 的修饰。

```swift
struct ContentView: View {
    var body: some View {
        Text("hello")
        Text("world")
    }
}
```

`@ViewBuilder` 能够自动用合适的容器把多个 View 包装起来。


## 将 View 作为属性

你可以把 View 作为属性，并在需要的位置引用它:

```swift
struct ContentView: View {
    let motto1 = Text("Draco dormiens")
    let motto2 = Text("nunquam titillandus")

    var body: some View {
        VStack {
            motto1
            motto2
        }
    }
}
```

还可以把 view 作为 computed property，这是一个小技巧，普通的 property 是不能引用其它 property 的，但是 computed property 可以。

```swift
struct ContentView: View {
    let name = "DongDada"

    var spells: some View {
        VStack {
            Text("Hello, \(DongDada)")
            Text("Hi~")
        }
    }

    var body: some View {
        spells
    }
}
```

## 自定义 View

其实 ContentView 就是个自定义的 View，你可以用类似的方式自定义出其它 View:

```swift
struct CapsuleText: View {
    var text: String

    var body: some View {
        Text(text)
            .font(.largeTitle)
            .padding()
            .foregroundColor(.white)
            .background(.blue)
            .clipShape(Capsule())
    }
}

struct ContentView: View {
    var body: some View {
        VStack(spacing: 10) {
            CapsuleText(text: "First")
            CapsuleText(text: "Second")
                .foregroundColor(.yellow)
        }
    }
}
```

## 自定义 modifier

通过实现 ViewModifier 接口，可以自定义出新的 modifier。然后通过 View 的 `modifier()` 方法，将这个 modifier 应用上去

```swift
struct Title: ViewModifier {
    func body(content: Content) -> some View {
        content
            .font(.largeTitle)
            .foregroundColor(.white)
            .padding()
            .background(.blue)
            .clipShape(RoundedRectangle(cornerRadius: 10))
    }
}

struct ContentView: View {
    var body: some View {
        Text("hello")
            .modifier(Title())
    }
}
```

可以看到 ViewModifier 的 `body()` 方法，其返回值是一个 View, 这意味着你不止可以在原始 view 的基础上施加 modifier, 还可以直接添加别的 View:

```swift
struct Watermark: ViewModifier {
    var text: String

    func body(content: Content) -> some View {
        ZStack(alignment: .bottomTrailing) {
            content
            Text(text)
                .font(.caption)
                .foregroundColor(.white)
                .padding(5)
                .background(.black)
        }
    }
}
```

为了书写简单，可以通过 extension 机制，把自定义 modifier 包装成 View 的一个方法

```swift
extension View {
    func watermarked(with text: String) -> some View {
        // 调用 View 的 modifier 方法
        modifier(Watermark(text: text))
    }
}

struct ContentView: View {
    var body: some View {
        Color.blue
            .frame(width: 200, height: 200)
            .watermarked(with: "Dong dada")
    }
}
```

## 自定义容器

容器也是一种 View, 所以可以通过类似于自定义 View 的方式来定义自己的容器.

```swift
struct GridStack<Content: View>: View {
    let rows: Int
    let columns: Int
    // 一个回调函数，由调用方提供，通过它来创建容器内的子控件
    let content: (Int, Int) -> Content
    
    var body: some View {
        VStack {
            ForEach(0..<rows, id: \.self) { row in
                HStack {
                    ForEach(0..<columns, id: \.self) { column in
                        content(row, column)
                    }
                }
            }
        }
    }
}

struct ContentView: View {
    var body: some View {
        GridStack(rows: 10, columns: 5) { row, column in
            Color(red: Double(row) * 0.1, green: Double(column) * 0.1, blue: 0.0)
        }
    }
}
```

## 动画

### 为 View 变化添加动画

这种方式非常简单，只需要使用 `.animation()` modifier 包装一下 View，这个 View 的变化就会以动画的形式展现，你不需要考虑动画的每一帧是怎么渲染的，SwiftUI 框架会帮你完成这些事。

```swift
struct ContentView: View {

    @State private var animationAmount = 1.0
    @State private var bigger = true

    var body: some View {
        Button("Tap me") {
            if fabs(animationAmount.distance(to: 2.0)) <= 1e-15 {
                bigger = false
            }
            if fabs(animationAmount.distance(to: 1.0)) <= 1e-15 {
                bigger = true
            }
            
            animationAmount += bigger ? 0.2 : -0.2
        }
        .padding(50)
        .background(bigger ? .red : .green)
        .foregroundColor(.white)
        .clipShape(Circle())
        .scaleEffect(animationAmount)
        .animation(.easeInOut, value: animationAmount)
    }
}
```

以上代码有几点需要注意：
- `.animation()` 的第一个参数是指要附加的动画效果，它更像是描述了变化的快慢，这种效果可以影响颜色、大小、模糊程度等等。
- `.animation()` 的第二个参数 `value` 用于触发动画的执行，如果发生变化的属性不是传给 `value` 的那个属性，动画就不会被触发。比如上述代码如果把 `value` 改为 `bigger`，那么动画就只在 `bigger` 属性发生变化时才被执行。
- 如果动画被触发的时候，View 有多个状态发生变化，那么这些变化都会以动画的形式呈现。比如上述代码中，当 `bigger` 或 `animationAmount` 发生变化的时候，都会附加动画效果。


除了上述代码中展示的 `.easeInOut`, 还有其它效果的动画可以使用，并且可以为这些效果指定参数:

```swift
// 指定动画时长
.animation(.easeInOut(duration: 2), value: animationAmount)

// 延时一秒后开始执行
.animation(.easeInOut(duration: 2).delay(1), value: animationAmount)

// 动画重复 3 次，执行一次后返回原状，再执行下一次
.animation(.easeInOut(duration: 1).repeatCount(3, autoreverses: true), value: animationAmount)

// 动画一直重复
.animation(.easeInOut(duration: 1).repeatForever(autoreverse: true), value: animationAmount)

// .interpolatingSpring 弹跳效果
.animation(.interpolatingSpring(stiffness: 50, damping: 1), value: animationAmount)
```

此外，`.animation()` 只影响在这之前的 modifier, 比如下面的代码中，`.clipShape()` 因为写在了 `.animation()` 后面，因此不会有动画效果。这一点很容易理解，因为 modifier 是对原有结构的一种包装，那么 `.animation()` 当然只能原有结构生效，无法对新加入的 `.clipShape()` 生效。

```swift
struct ContentView: View {
    @State private var enabled = false
    
    var body: some View {
        Button("Tap me") {
            enabled.toggle()
        }
        .frame(width: 200, height: 200)
        .background(enabled ? .blue : .red)
        .foregroundColor(.white)
        .animation(.default, value: enabled)
        // 必须放在 .animation() 前面，才会有动画效果。
        .clipShape(RoundedRectangle(cornerRadius: enabled ? 60 : 0))
    }
}
```

如果添加了多个 `.animation()` modifier，这些 animation 会 **同时执行**，但每个 `.animation()` 只对之前的 modifier 生效，这类似于把属性修改分了段落，具体可以看看下列代码，注意第一个 animation 只影响 `.background()`, 第二个 animation 只影响 `.clipShape()`，**就像把属性变化分成了不同的组一样**。

```swift
struct ContentView: View {
    @State private var enabled = false
    
    var body: some View {
        Button("Tap me") {
            enabled.toggle()
        }
        .frame(width: 200, height: 200)
        .background(enabled ? .blue : .red)
        // 背景颜色切换的动画执行 5 遍
        .animation(.easeInOut(duration: 0.2).repeatCount(5, autoreverses: true), value: enabled)
        .foregroundColor(.white)
        .clipShape(RoundedRectangle(cornerRadius: enabled ? 60 : 0))
        // 形状裁剪的动画执行 1 遍
        .animation(.easeInOut(duration: 1), value: enabled)
    }
}
```


### 为 Binding 属性变化添加动画

这种方式下，动画的作用目标是 binding 属性，而不是 View。当 binding 属性发生变化时，动画效果会作用在绑定了这个属性的 view 上面。比如下面的例子，Stepper 修改了 `$animationAmount` 的值之后，Button 会按照指定的动画效果进行缩放。

在动画过程中，Binding 属性的值并不会逐渐修改，动画的作用目标还是 View。

```swift
struct ContentView: View {

    @State private var animationAmount = 1.0

    var body: some View {
        VStack {
            // Binding 属性支持 `.animation()` 方法，可以将属性的变化以动画效果来呈现
            Stepper("Scale amount", value: $animationAmount.animation(.easeOut), in: 0.6...10, step: 0.2)
            
            Spacer()
            
            Button("Tap me") {
            }
            .padding(50)
            .background(.red)
            .foregroundColor(.white)
            .clipShape(Circle())
            .scaleEffect(animationAmount)
        }
    }
}
```

### 为普通属性变化添加动画

为 View 添加 `.animation()` modifier, 可以让 View 的所有变化以动画的方式呈现。为 Binding 属性添加 `.animation()` modifier，可以让该属性的变化以动画方式呈现。还有一种方法是使用 `withAnimation()` closure 来让普通属性的变化以动画方式呈现:

```swift
struct ContentView: View {

    @State private var animationAmount = 0.0

    var body: some View {
        Button("Tap me") {
            withAnimation(.easeInOut) {
                animationAmount += 180
            }
        }
        .padding(50)
        .background(.red)
        .foregroundColor(.white)
        .clipShape(Circle())
        .rotation3DEffect(.degrees(animationAmount), axis: (x: 0, y: 1, z: 0))
    }
}
```


## Bundle

SwiftUI 项目中的 Assets 主要是用来存放图片的。把图片拖进 Assets, 然后起个名字，就可以直接在 Image 控件中引用它:

```swift
Image("Germany")
```

如果希望将文本文件等内容放到项目里，就需要使用到 Bundle.

首先把文件拖动到项目里，然后使用以下代码读取文件内容：

```swift
guard let fileURL = Bundle.main.url(forResource: "start", withExtension: "txt") else { return }

guard let fileContents = try? String(contentsOf: fileURL) else { return }

// 操作 fileContents ...
```

这里需要注意，不论你把文件放到了哪个文件夹，xcode 在编译的时候都会把这些文件放在同一个文件夹下。这意味着你没法导入两个同名文件到 xcode 里面，同时在使用上述代码访问这些文件的时候，也就不需要提供目录名，只需要填入文件名就可以了。



## View 加载时执行指定操作

使用 `onAppear()` modifer 就可以指定在视图加载时需要指定的方法:

```swift
struct ContentView: View {
    @State private var newWord = ""
    
    var body: some View {
        TextField("Enter your word", text: $newWord)
            .autocapitalization(.none)
            .onSubmit { text in
                // onSubmit 将在按下回车键后被调用
                print(text)
            }
            .onAppear {
                // onAppear 将在 View 被加载时调用
                // ...
            }
    }
}
```


## .transition() modifier

`.transition()` modifier 用于控制 View 在显示和隐藏时候的切换效果。比如下面的代码:

```swift
struct ContentView: View {
    
    @State private var isShowingRed = false
    
    var body: some View {
        VStack {
            Button("Tap Me") {
                withAnimation {
                    isShowingRed.toggle()
                }
            }
            
            if isShowingRed {
                Rectangle()
                    .fill(.red)
                    .frame(width: 200, height: 200)
                    // 默认情况下，View 会以透明度变化来完成显示和隐藏状态的切换，
                    // 但通过 `.transition(.scale)` modifier，可以将显示和隐藏过程更改为缩放效果
                    .transition(.scale)
            }
        }
    }
}
```

默认情况:

![]( {{site.url}}/asset/swiftui-transition-default.gif )

修改后的效果:

![]( {{site.url}}/asset/swiftui-transition-scale.gif )

你还可以通过 `.asymmetric()` 为显示和隐藏分别指定不同的切换效果:

```swift
.transition(.asymmetric(insertion: .scale, removal: .opacity))
```


## 手势

### DragGesture

DragGesture 可以使用 `.gesture()` modifier 添加到 View 上，以下代码利用 DragGesture 实现了拖动 View 的效果:

```swift
struct ContentView: View {
    
    @State private var dragAmount = CGSize.zero
    
    var body: some View {
        LinearGradient(gradient: Gradient(colors: [.yellow, .red]),
                       startPoint: .topLeading, endPoint: .bottomTrailing)
            .frame(width: 300, height: 200)
            .clipShape(RoundedRectangle(cornerRadius: 10))
            .offset(dragAmount)
            .gesture(DragGesture()
                        // 每当手指位置变化，onChanged 事件就会触发
                        .onChanged() { dragAmount = $0.translation }
                        // 每当手指离开屏幕，onEnded 事件就会触发                        
                        .onEnded() { _ in dragAmount = .zero })
            // 使用动画效果让拖动过程更平滑
            .animation(.spring(), value: dragAmount)
    }
}
```

### TapGesture

TapGesture 可以使用 `.onTapGesture()` modifier 添加到 View 上，使 View 能够相应点击事件:

```swift
struct ContentView: View {
    
    @State private var isShowingRed = false
    
    var body: some View {
        ZStack {
            Rectangle()
                .fill(.blue)
                .frame(width: 200, height: 200)
            
            if isShowingRed {
                Rectangle()
                    .fill(.red)
                    .frame(width: 200, height: 200)
                    .transition(.scale)
            }
        }
        .onTapGesture {
            withAnimation {
                isShowingRed.toggle()
            }
        }
    }
}
```


# 常见控件

## Section

Section 跟 Form 配合使用，能够把 Form 分成不同的部分，这里主要演示一下 closure 的简写方法:

```swift
Form {
    // 第一个段落
    Section {
        Text(checkAmount, format: .currency(code: Locale.current.currencyCode ?? "USD"))
    }

    // 第二个段落
    Section {
        Picker("Tip percentage", selection: $tipPercentage) {
            ForEach(tipPercentages, id: \.self) {
                Text($0, format: .percent)
            }
        }
        .pickerStyle(.segmented)
    } header: {
        Text("How much tip do you want to leave?")
    }
}
```

注意第二个 Section 中添加了 `header:` 参数，这其实是 closure 的一种简便写法：

```swift
// 有多个 closure 参数时的简写方式
func doImportantWork(first: () -> Void, second: () -> Void, third: () -> Void) {
    print("About to start first work")
    first()
    print("About to start second work")
    second()
    print("About to start third work")
    third()
    print("Done!")
}
doImportantWork {
    print("This is first work")
} second: {
    print("This is second work")
} third: {
    print("This is third work")
}
```

## ForEach

ForEach 控件本身是个 View，它能够循环创建 View。

它的构造函数有两个参数，第一个是 Range，第二个是 closure:

```swift
Form {
    // 最后一个参数是 closure, 因此可以把大括号写到后面
    ForEach(0..<100) { number in
        Text("Row \(number)")
    }
}
```

需要注意的是 ForEach 只管把 View 创建出来，它最后长啥样取决于外面的容器。比如上面的例子中外面的容器是 Form, 因此最后长下面这样:

![]({{site.url}}/asset/swiftui-foreach-view-with-form.jpg)

如果把外面的容器改成 HStack，那么最后会长下面这样:

```swift
HStack {
    // 最后一个参数是 closure, 因此可以把大括号写到后面
    ForEach(0..<100) { number in
        Text("Row \(number)")
    }
}
```

![]({{site.url}}/asset/swiftui-foreach-hstack.jpg)

在 ForEach 中访问数组需要指定 id:

```swift
struct ContentView: View {
    let colors: [Color] = [.red, .green, .blue]

    var body: some View {
        VStack {
            ForEach(colors, id: \.self) { color in
                Text(color.description.capitalized)
                    .padding()
                    .background(color)
            }
        }
    }
}
```


## Picker

Picker 提供选项给用户选择。它长啥样还跟是放在哪里有关系，比如下面的代码把 Picker 放在 NavigationView 和 Form 里面，此时 Picker 显示为一个新的页面。如果把 Picker 直接放在最外面，那么 Picker 就会显示为一个滚轮。

```swift
struct ContentView: View {
    let students = ["Harry", "Hermione", "Ron"]
    @State private var selectedStudent = "Harry"

    var body: some View {
        NavigationView {
            Form {
                // 参数 1: 标题
                // 参数 2: 需要双向绑定，用于输出用户选择的结果
                // 参数 3: 用于绘制多个选择项
                Picker("Select your student", selection: $selectedStudent) {

                    // 注意这里为 ForEach 额外设置了 id 参数，因为 Picker 需要唯一确定每个选择项
                    ForEach(students, id: \.self) {
                        Text($0)
                    }
                }
            }
        }
    }
}
```

![]( {{site.url}}/asset/swiftui-picker.jpg )


## TextField

TextField 也就是文本输入框，值得注意的是它所绑定的变量不一定是 String，也可以是 Double 等类型，这可以通过 `format` 参数来改变:

```swift
struct ContentView: View {
    @State private var checkAmount = 0.0
    
    var body: some View {
        Form {
            // 参数1: 提示
            // 参数2: 用于接收输入结果
            // 参数3: 指定文本的格式，这里指定的是 `.currency` 货币
            TextField("Amount", value: $checkAmount, format: .currency(code: Locale.current.currencyCode ?? "USD"))
                // keyboardType() modifier 可以指定键盘类型
                .keyboardType(.decimalPad)

            Text(checkAmount, format: .currency(code: Locale.current.currencyCode ?? "USD"))
        }
    }
}
```


## Color

Color 也是 View，它也有对应的大小。

```swift
struct ContentView: View {
    var body: some View {
        VStack {
            // 在安全区渲染颜色
            Color.gray
                .ignoresSafeArea()
            
            // Color 是个有大小的 View
            Color.mint
                .frame(width: .infinity, height: 200, alignment: .center)
            
            ZStack {
                Color.indigo
                    .frame(width: 200, height: 200, alignment: .center)
                Color.primary
                    .frame(width: 150, height: 150, alignment: .center)
                Color.secondary
                    .frame(width: 100, height: 100, alignment: .center)
                Color(red: 1, green: 0.8, blue: 0)
                    .frame(width: 50, height: 50, alignment: .center)
            }
            
            Color.red
        }
    }
}
```

![]( {{site.url}}/asset/swiftui-color.png )


## Gradient

类似 Color, Gradient(渐变) 也是一种 View，它也有对应的大小。

```swift
struct ContentView: View {
    var body: some View {
        VStack {
            // 线性渐变
            LinearGradient(gradient: Gradient(colors: [.white, .black]),
                           startPoint: .top, endPoint: .bottom)
            
            // 线性渐变可以指定渐变的起始位置
            LinearGradient(gradient: Gradient(stops: [
                Gradient.Stop(color: .white, location: 0.45),
                Gradient.Stop(color: .black, location: 0.55),
            ]), startPoint: .top, endPoint: .bottom)
            
            // 放射形渐变
            RadialGradient(gradient: Gradient(colors: [.blue, .black]),
                           center: .center, startRadius: 20, endRadius: 200)
            
            // 锥形渐变
            AngularGradient(gradient: Gradient(colors: [.red, .yellow, .green, .blue, .purple, .red]),
                            center: .center)
        }
    }
}
```

![]( {{site.url}}/asset/swiftui-gradient.png )


## Button

```swift
struct ContentView: View {
    var body: some View {
        VStack {
            Button("Delete selection", action: executeDelete)
            
            // .buttonStyle() 修饰器可以改变按钮风格
            Button("Button 1") { }
                .buttonStyle(.bordered)
            // role 参数可以影响按钮的显示，比如 .destructive 表示点击按钮是高危动作
            Button("Button 2", role: .destructive) { }
            // .tint() 修饰器用于改变按钮的颜色
            Button("Button 3") { }
                .buttonStyle(.borderedProminent)
                .tint(.mint)
            Button("Button 4", role: .destructive) { }
                .buttonStyle(.borderedProminent)
            
            // 使用 label 参数可以更丰富地自定义按钮样式
            Button {
                print("Button was tapped")
            } label: {
                Text("Tap me!")
                    .padding()
                    .foregroundColor(.white)
                    .background(.red)
            }
            
            // 图片
            Button {
                print("Edit button was tapped")
            } label: {
                Image(systemName: "pencil")
            }
            
            // 文字 + 图片
            Button {
                print("Edit button was tapped")
            } label: {
                Label("Edit", systemImage: "pencil")
            }
        }
    }
    
    func executeDelete() {
        print("Now deleting...")
    }
}
```

![]( {{site.url}}/asset/swiftui-button.png )


## Alert

Alert 用于显示对话框

```swift
struct ContentView: View {
    
    // 用一个属性来表示是否需要展示对话框
    @State private var showingAlert = false
    
    var body: some View {
        VStack {
            Button("Show Alert") {
                showingAlert = true
            }
            // 当对话框消失，isPresented 会自动通过双向绑定修改 showingAlert 属性
            // 默认情况下任何按钮按下都将关闭对话框
            .alert("Important message", isPresented: $showingAlert) {
                Button("Delete", role: .destructive) { }
                Button("Cancel", role: .cancel) { }
            } message: {
                // message 参数用于在对话框中显示文本等内容
                Text("Please read this.")
            }
            
            Spacer()
        }
    }
    
    func executeDelete() {
        print("Now deleting...")
    }
}
```

![]( {{site.url}}/asset/swiftui-alert.png )


## Stepper

Stepper 控件用于选择数字:

```swift
struct ContentView: View {
    @State private var sleepAmount = 8.0
    
    var body: some View {
        // 第一个参数是要显示的内容
        // 第二个参数用于接收输出
        // 第三个参数用于限制数字范围
        // 第四个参数用于设置步长
        Stepper("\(sleepAmount.formatted())", value: $sleepAmount, in: 4...12, step: 0.25)
    }
}
```

![]( {{site.url}}/asset/swiftui-stepper.png )


## DatePicker

DatePicker 控件用于选择日期:

```swift
struct ContentView: View {
    @State private var wakeUp = Date.now
    
    var body: some View {
        // 第一个参数是个标题，可以用 `.labelsHidden()` 这个 modifier 隐藏标题
        // 第二个参数用于接收输出
        // 第三个参数用于指定日期范围
        // 第四个参数用于选择是否要展示日期或时间
        DatePicker("Please enter a date", selection: $wakeUp, in: Date.now..., displayedComponents: .hourAndMinute)
            .labelsHidden()
    }
}
```

![]( {{site.url}}/asset/swiftui-date-picker.png )


## List

之前提到的 Form View, 其实也是一种 List, 你可以在 List 里面使用 Section, ForEach:

```swift
struct ContentView: View {
    var body: some View {
        List {
            Section("Section 1") {
                Text("Static row 1")
                Text("Static row 2")
            }
            
            Section("Section 2") {
                ForEach(0..<5) {
                    Text("Dynamic row \($0)")
                }
            }
            
            Section("Section 3") {
                Text("Static row 3")
                Text("Static row 4")
            }
        }
        .listStyle(.grouped)
    }
}
```

![]( {{site.url}}/asset/swiftui-list-view.png )

不同于 Form 的是，List 不需要使用 ForEach 就能动态的创建出列表项:

```swift
struct ContentView: View {
    let people = ["Finn", "Leia", "Luke", "Rey"]

    var body: some View {
        // 跟 ForEach 一样，你需要指定将列表项的一个属性设置为 id
        List(people, id: \.self) {
            Text($0)
        }
    }
}
```