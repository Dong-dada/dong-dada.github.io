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

### @State 的原理

上文提到 `@State` 实际上会把属性包装到另外一个地方，所以我们才能修改它。不过这也引起了一些问题，比如下面的代码:

```swift
struct ContentView: View {
    @State private var blurAmount = 0.0 {
        didSet {
            // 这行代码不会被执行到
            print("new value is \(blurAmount)")
        }
    }
    
    var body: some View {
        VStack {
            Text("Hello, world!")
                .blur(radius: blurAmount)
            
            Slider(value: $blurAmount, in: 0...20)
        }
    }
}
```

上述问题需要了解 `@State` 才能搞明白。Property wrapper 实际上会把我们的属性包装到一个 struct 里面，比如上述例子中，blurAmount 会被包装到 `State<Double>` 类型的结构体中。你可以通过 Cmd+Shift+O 打开 quick open 工具栏，然后输入 State 来查看这个 State 结构体的定义：

```swift
@@propertyWrapper public struct State<Value> : DynamicProperty {
    // ...

    public var wrappedValue: Value { get nonmutating set }

    // ...
}
```

`State` 结构体中的 `wrappedValue` 是个 compute property，真实的值并不是保存在 `wrappedValue` 里面，而是保存在其他地方。注意它的 set 被修改为 `nonmutating`，这表示设置 wrappedValue 的时候，不会改变结构体内的任何属性，实际上被改变的是保存在其它地方的一个可以自由修改的值。也就是说，在我们对 `@State` 属性做修改的时候，不仅 `blurAmount` 属性不会变化，`State<Double>` 结构体也不会变化，变化的是保存在其它地方的一个值。

回到开始的问题，以下代码中 `print()` 不被执行，是因为 `@State` 将 blurAmount 包装成了 `State<Double>` 结构体，只有 `State<Double>` 结构体发生变化，`didSet` 才会被触发，但根据上面的描述，`State<Double>` 是不会发生变化的。

要解决这一问题，可以通过自定义 Binding 的方式:

```swift
struct ContentView: View {
    @State private var blurAmount: CGFloat = 0

    var body: some View {
        // 引入一个自定义 Binding 对象，作为中间人，把 getter, setter 转发给原本的 Binding
        // 注意，Swift 不允许属性之间互相引用，所以不能把这个 Binding 对象作为 ContentView 的属性，需要放到 body 这个 computed property 里面
        let blur = Binding<CGFloat>(
            get: {
                self.blurAmount
            },
            set: {
                self.blurAmount = $0
                print("New value is \(self.blurAmount)")
            }
        )

        // 因为上面有了计算逻辑，所以这里需要把 return 写上
        return VStack {
            Text("Hello, World!")
                .blur(radius: blurAmount)

            // 传入自定义 Binding 而非原本的 Binding
            Slider(value: blur, in: 0...20)
        }
    }
}
```

可以看到，通过引入一个自定义 Binding 对象作为中间人，就能观察到 binding 值被修改的过程了。你可以在这里打日志、保存数据到 UserDefaults 之类的。


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


## @EnvironmentObject

`@EnvironmentObject` 的作用是从 "environment" 里获取相应类型的 ObservableObject 对象，并接收它发来的事件，更新视图。我们可以利用它把对象传递给所有子视图:

```swift
// 被获取的对象可以是实现了 ObservableObject 协议的对象
// 这样任何一个子视图修改了对象，其它视图都能收到事件做出反映
class User: ObservableObject {
    @Published var name = "Taylor Swift"
}

struct EditView: View {
    // 子视图可以从 environment 中根据类型获取到对象
    @EnvironmentObject var user: User
    
    var body: some View {
        // 在 EditView 里修改了 user.name 之后，DisplayView 将被更新
        TextField("Name", text: $user.name)
    }
}

struct DisplayView: View {
    // 子视图可以从 environment 中根据类型获取到对象
    @EnvironmentObject var user: User
    
    var body: some View {
        Text(user.name)
    }
}

struct ContentView: View {
    let user = User()
    
    var body: some View {
        VStack {
            EditView()
            DisplayView()
        }
        // 给 VStack 设置上 environment object
        .environmentObject(user)
    }
}
```

`@EnvironmentObject` 还兼具类似 `@ObservedObject` 的功能，当对象的内容发生变化时，View 可以感知到变化并自动更新。

注意，你必须确保在 `@EnvironmentObject` 生效时，对象已经通过 `.environmentObject()` 被设置到了环境里，否则会导致程序崩溃。


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


## @Binding

之前提到过双向绑定，`@Binding` 的作用就是将某个属性包装为双向绑定属性：这个属性所发生的变化可以被其他值所绑定

```swift
struct PushButton: View {
    let title: String
    // 将 isOn 包装为双向绑定属性
    @Binding var isOn: Bool
    
    var onColors = [Color.red, Color.yellow]
    var offColors = [Color(white: 0.6), Color(white: 0.4)]
    
    var body: some View {
        Button(title) {
            // 修改 isOn 属性，与 isOn 所绑定的值会同时被修改
            isOn.toggle()
        }
        .padding()
        .background(LinearGradient(gradient: Gradient(colors: isOn ? onColors : offColors),
                                   startPoint: .top, endPoint: .bottom))
        .foregroundColor(.white)
        .clipShape(Capsule())
        .shadow(radius: isOn ? 0 : 5)
    }
}

struct ContentView: View {
    @State private var rememberMe = false
    
    var body: some View {
        VStack {
            // PushButton 的 isOn 属性发生变化后，ContentView 的 rememberMe 属性也会发生变化
            PushButton(title: "Remember Me", isOn: $rememberMe)
            Text(rememberMe ? "On" : "Off")
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


### onTapGesture

TapGesture 可以使用 `.onTapGesture()` modifier 添加到 View 上，使 View 能够响应点击事件:

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

通过添加参数，可以让 `.onTapGesture()` 处理双击事件:

```swift
Text("Hello, World!")
    .onTapGesture(count: 2) {
        print("Double tapped!")
    }
```

### onLongPressGesture

`.onLongPressGesture()` 可以让 View 响应长按事件:

```swift
Text("Hello, World!")
    .onLongPressGesture {
        print("Long pressed!")
    }
```

通过添加参数，可以设置长按的时长:

```swift
Text("Hello, World!")
    .onLongPressGesture(minimumDuration: 2) {
        print("Long pressed!")
    }
```

通过添加参数，可以接收到 "处理中" 事件:
- 如果长按的时间不够长，比如没到 1 秒，那么以下代码中的 `inProgress` 参数会设置为 false
- 如果长按的时间足够长，那么首先 `inProgress` 会被设置为 false，随后将收到完成事件。

```
Text("Hello, world!")
    .onLongPressGesture(minimumDuration: 1, pressing: { inProgress in
        print("In progress: \(inProgress)!")
    }) {
        print("Long pressed!")
    }
```

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

### MagnificationGesture

MagnificationGesture 表示放大手势，需要使用 `.gesture()` modifier 来添加:

```swift
struct ContentView: View {
    @State private var currentAmount: CGFloat = 0
    @State private var finalAmount: CGFloat = 1
    
    var body: some View {
        Text("Hello, world!")
            .scaleEffect(finalAmount + currentAmount)
            .gesture(
                MagnificationGesture()
                    .onChanged{ amount in
                        self.currentAmount = amount - 1
                    }
                    .onEnded{ amount in
                        self.finalAmount += self.currentAmount
                        self.currentAmount = 0
                    })
    }
}
```

### RotationGesture

RotationGesture 表示旋转手势，需要使用 `.gesture()` modifier 来添加:

```swift
struct ContentView: View {
    @State private var currentAmount: Angle = .degrees(0)
    @State private var finalAmount: Angle = .degrees(0)
    
    var body: some View {
        Text("Hello, world!")
            .rotationEffect(finalAmount + currentAmount)
            .gesture(
                RotationGesture()
                    .onChanged{ angle in
                        self.currentAmount = angle
                    }
                    .onEnded{ angle in
                        self.finalAmount += self.currentAmount
                        self.currentAmount = .degrees(0)
                    })
    }
}
```

### 手势的优先级

在下面这种内外 View 都有 gesture 的情况下，child view 的优先级更高:

```swift
struct ContentView: View {
    var body: some View {
        VStack {
            Text("Hello, World!")
                .onTapGesture {
                    print("Text tapped")
                }
        }
        .onTapGesture {
            print("VStack tapped")
        }
    }
}
```

可以使用 `.highPriorityGesture()` modifier 为外层 view 设置更高的优先级:

```swift
struct ContentView: View {
    var body: some View {
        VStack {
            Text("Hello, World!")
                .onTapGesture {
                    print("Text tapped")
                }
        }
        .highPriorityGesture(
            TapGesture()
                .onEnded { _ in
                    print("VStack tapped")
                }
        )
    }
}
```

可以使用 `.simultaneousGesture()` modifier 为内外两层设置相同的优先级，这样两个手势都会被触发。

```swift
struct ContentView: View {
    var body: some View {
        VStack {
            Text("Hello, World!")
                .onTapGesture {
                    print("Text tapped")
                }
        }
        .simultaneousGesture(
            TapGesture()
                .onEnded { _ in
                    print("VStack tapped")
                }
        )
    }
}
```


### 手势序列

通过合并多个手势，可以组成手势序列，必须先完成前面的手势，才能继续捕获后面的手势:

```swift
struct ContentView: View {
    @State private var offset = CGSize.zero
    @State private var isDraging = false
    
    var body: some View {
        let pressGesture = LongPressGesture()
            .onEnded { value in
                withAnimation {
                    // 改变 isDraging 状态，以便观察到长按生效
                    self.isDraging = true
                }
            }
        
        let dragGesture = DragGesture()
            .onChanged { value in
                self.offset = value.translation
            }
            .onEnded { _ in
                withAnimation {
                    self.offset = .zero
                    self.isDraging = false
                }
            }
        
        // 要求先捕获到长按手势，才能捕获到拖动手势
        let combinedGesture = pressGesture.sequenced(before: dragGesture)
        
        return Circle()
            .fill(Color.red)
            .frame(width: 64, height: 64)
            .scaleEffect(isDraging ? 1.5 : 1)
            .offset(offset)
            .gesture(combinedGesture)
    }
}
```


## JSONEncoder, JSONDecoder

顾名思义用于 JSON 编解码:

```swift
// 结构体需要实现 Codable 协议，才能被 JSONEncoder, JSONDecoder 处理
struct GroceryProduct: Codable {
    var name: String
    var points: Int
    var description: String?
}

let pear = GroceryProduct(name: "Pear", points: 250, description: "A ripe pear.")

let encoder = JSONEncoder()
encoder.outputFormatting = .prettyPrinted

// encode 会返回一个 Data 类型的对象，而不是字符串，需要进行一次装换
let data = try encoder.encode(pear)
print(String(data: data, encoding: .utf8)!)

/* Prints:
 {
   "name" : "Pear",
   "points" : 250,
   "description" : "A ripe pear."
 }
*/



// 传给 decode 的对象，不是字符串，而是 Data 类型，需要进行一次转换
let json = """
{
    "name": "Durian",
    "points": 600,
    "description": "A fruit with a distinctive scent."
}
""".data(using: .utf8)!

let decoder = JSONDecoder()
let product = try decoder.decode(GroceryProduct.self, from: json)

print(product.name) // Prints "Durian"
```

利用泛型可以编写出直接从 bundle 中加载 json 的方法:

```swift
extension Bundle {
    func decodeJson<T: Codable>(_ file: String) -> T {
        guard let url = self.url(forResource: file, withExtension: nil) else {
            fatalError("Failed to locate \(file) in bundle.")
        }

        guard let data = try? Data(contentsOf: url) else {
            fatalError("Failed to load \(file) from bundle.")
        }

        let decoder = JSONDecoder()
        
        guard let loaded = try? decoder.decode(T.self, from: data) else {
            fatalError("Failed to decode \(file) from bundle.")
        }

        return loaded
    }
}


let astronauts: [Astronaut] = Bundle.main.decodeJson("astronauts.json")
```

如果 class 内所有属性都实现了 Codable 协议，那么这个 class 就可以直接自动实现 Codable 协议。否则的话就需要自己实现协议了。

如果 class 内有使用 `@Published` 包装的属性，那就需要自己来实现 Codable 协议:

```swift
class User: ObservableObject, Codable {
    @Published var name = "Dong Dada"
    
    // 实现 CodingKey 协议的枚举，描述有哪些字段需要被序列化
    enum CodingKeys: CodingKey {
        case name
    }
    
    // 新增构造函数，支持反序列化
    // required 表示如果继承当前类，则必须覆写该方法
    required init(from decoder: Decoder) throws {
        let container = try decoder.container(keyedBy: CodingKeys.self)
        name = try container.decode(String.self, forKey: .name)
    }
    
    // 覆写 encode 方法，支持序列化
    func encode(to encoder: Encoder) throws {
        var container = encoder.container(keyedBy: CodingKeys.self)
        try container.encode(name, forKey: .name)
    }
}
```


## URLSession

使用 URLSession 方法可以发起 HTTP 请求。不过这是一个异步方法，需要使用 `await` 关键字来发起调用:

```swift
struct Response: Codable {
    var results: [Result]
}

struct Result: Codable {
    var trackId: Int
    var trackName: String
    var collectionName: String
}

struct ContentView: View {
    @State private var results = [Result]()

    var body: some View {
        List(results, id: \.trackId) { item in
            Button("刷新") {
                // 不能直接调用 async 方法，需要包装到 Task 里面
                Task {
                    await loadData()
                }
            }
            .padding()

            VStack(alignment: .leading) {
                Text(item.trackName)
                    .font(.headline)
                Text(item.collectionName)
            }
        }
        // 无法在 .onAppear() modifier 里调用 async 方法，SwiftUI 提供了 .task() modifier 用于执行 async 任务
        .task {
            await loadData()
        }
    }

    // 由于内部需要访问异步方法，所以需要将 loadData() 声明为 async 方法
    func loadData() async {
        guard let url = URL(string: "https://itunes.apple.com/search?term=taylor+swift&entity=song") else {
            print("Invalid URL")
            return
        }

        do {
            // 使用 try await 关键字发起异步调用
            let (data, _) = try await URLSession.shared.data(from: url)

            if let decodedResponse = try? JSONDecoder().decode(Response.self, from: data) {
                results = decodedResponse.results
            }
        } catch {
            print("Invalid data")
        }
    }
}
```

也可以构造 URLRequest 发送 POST 请求：

```swift
func placeOrder() async {
    guard let encoded = try? JSONEncoder().encode(order) else {
        print("Failed to encode order")
        return
    }
    
    let url = URL(string: "https://reqres.in/api/cupcakes")!
    var request = URLRequest(url: url)
    request.setValue("application/json", forHTTPHeaderField: "Content-Type")
    request.httpMethod = "POST"
    
    do {
        let (data, _) = try await URLSession.shared.upload(for: request, from: encoded)
        
        let decodedOrder = try JSONDecoder().decode(Order.self, from: data)

        // ...
    } catch {
        print("Checkout failed.")
    }
}
```


## UserDefaults

UserDefaults 用于持久化数据，适合于存储用户配置之类的少量数据，以 KV 形式提供访问:

```swift
struct ContentView: View {
    // 从 UserDefaults 获取数据
    @State private var tapCount = UserDefaults.standard.integer(forKey: "Tap")
    
    var body: some View {
        Button("Tap count: \(tapCount)") {
            tapCount += 1

            // 将数据保存到 UserDefaults
            UserDefaults.standard.set(self.tapCount, forKey: "Tap")
        }
    }
}
```

SwiftUI 提供了一个包装器 `@AppStorage`, 被包装的属性会自动保存到 UserDefaults:

```swift
struct ContentView: View {
    @AppStorage("tapCount") private var tapCount = 0
    
    var body: some View {
        Button("Tap count: \(tapCount)") {
            tapCount += 1
        }
    }
}
```

默认情况下 UserDefaults 只能保存基本类型，如果想保存结构体，可以通过 JSONEncoder 把对象转换成 `Data`:

```swift
struct User: Codable {
    var firstName: String
    var lastName: String
}

struct ContentView: View {
    @State private var user = User(firstName: "Taylor", lastName: "Swift")
    
    var body: some View {
        VStack {
            TextField("FirstName", text: $user.firstName)
            TextField("LastName", text: $user.lastName)
            
            Button("Save User") {
                // 编码后保存到 UserDefaults
                let encoder = JSONEncoder()
                if let data = try? encoder.encode(user) {
                    UserDefaults.standard.set(data, forKey: "User")
                }
            }
        }
        .onAppear {
            // 从 UseDefaults 里获取 Data, 解码结构体对象
            if let data = UserDefaults.standard.data(forKey: "User") {
                let decoder = JSONDecoder()
                if let user = try? decoder.decode(User.self, from: data) {
                    self.user = user
                }
            }
        }
    }
}
```


## \.self 的工作原理

简单来说，`\.self` 会使用整个对象的 hash 值来唯一标识这个对象。这要求类型本身实现了 `Hashable` 协议。


## 在 View 内部使用条件判断

在 View 内部，可以使用 `if else` 来进行条件判断：

```swift
struct ContentView: View {
    @State private var image: Image?
    
    var body: some View {
        VStack {
            if image != nil {
                image?
                    .resizable()
                    .scaledToFit()
            } else {
                Text("Tap to select a picture")
            }
        }
    }
}
```

这是因为 SwiftUI 能够将 `if else` 转换成一种特殊的名为 `ConditionalContent` 的 View。

除了 `if else` 以外，其他的条件判断，比如 `if let`, `for`, `while`, `switch` 都是不能使用的，因为 SwiftUI 没有提供相应的转换逻辑。

这种条件式 View 很适合跟枚举结合起来，根据状态显示不同的 View:

```swift

enum LoadingState {
    case loading, success, failed
}

struct LoadingView: View {
    var body: some View {
        Text("Loading...")
    }
}

struct SuccessView: View {
    var body: some View {
        Text("Success!")
    }
}

struct FailedView: View {
    var body: some View {
        Text("Failed.")
    }
}

struct ContentView: View {
    var loadingState = LoadingState.loading
    
    var body: some View {
        Group {
            if loadingState == .loading {
                LoadingView()
            } else if loadingState == .success {
                SuccessView()
            } else if loadingState == .failed {
                FailedView()
            }
        }
    }
}
```



## 包装 UIViewController 到 SwiftUI 当中

UIView, UIViewController 是 UIKit 当中的概念，有一些功能因为还没有在 SwiftUI 上实现，所以需要桥接一下才能使用。

Apple 在 UIKit 以及其他框架中使用一种被称为 MVC 的模式。Model 用于访问数据，View 用于对界面做布局，Controller 是 Model 和 View 之间的胶水层。这里介绍一下怎么包装 Controller 到 SwiftUI 当中。

这里以 UIImagePickerController 为例展示桥接过程。

首先定义一个 ImagePicker 结构体，在其中完成 UIViewController 的包装。可以看到它主要利用了一个 Coordinator(中间人) 类来完成工作。这个类会处理 UIImagePickerController 发来的事件，最终将用户选择的图片保存到属性中。

值得注意的是: ImagePicker 结构体实现了 UIViewControllerRepresentable 协议，这个协议继承自 View，也就是说我们包装出的 ImagePicker 也是一个 View。

```swift
import Foundation
import SwiftUI

struct ImagePicker: UIViewControllerRepresentable {
    // 通过双向绑定，让调用方拿到用户选择的图片
    @Binding var image: UIImage?
    @Environment(\.presentationMode) var presentationMode
    
    func makeUIViewController(context: Context) -> UIImagePickerController {
        let picker = UIImagePickerController()
        
        // 将 picker controller 的事件委托给 Coordinator 处理
        picker.delegate = context.coordinator
        return picker
    }
    
    func updateUIViewController(_ uiViewController: UIImagePickerController, context: Context) {
    }
    
    // NSObject 是 UIKit 中所有对象的父类
    // UIImagePickerControllerDelegate 实现这个协议表示 Coordinator 可以处理来自 UIImagePickerController 的事件
    // UINavigationControllerDelegate 实现这个协议表示 Coordinator 可以处理导航相关的委托
    class Coordinator: NSObject, UIImagePickerControllerDelegate, UINavigationControllerDelegate {
        var parent: ImagePicker
        
        init(_ parent: ImagePicker) {
            // 保存 ImagePicker 到属性中，以便访问 ImagePicker 的属性
            self.parent = parent
        }
        
        // 实现子 UIImagePickerControllerDelegate 的接口，用于处理图片被选中的事件
        func imagePickerController(_ picker: UIImagePickerController, didFinishPickingMediaWithInfo info: [UIImagePickerController.InfoKey : Any]) {
            // 将选中的图片传给 ImagePicker 的 image 属性
            if let uiImage = info[.originalImage] as? UIImage {
                parent.image = uiImage
            }
            
            // 关闭 sheet
            parent.presentationMode.wrappedValue.dismiss()
        }
    }
    
    // 覆写这个方法，返回我们实现的 Coordinator 对象
    func makeCoordinator() -> Coordinator {
        Coordinator(self)
    }
}
```

包装好之后就可以在界面中使用了:

```swift
struct ContentView: View {
    @State private var image: Image?
    @State private var showingImagePicker = false
    @State private var inputImage: UIImage?
    
    var body: some View {
        VStack {
            image?
                .resizable()
                .scaledToFit()
            
            Button("Select Image") {
                self.showingImagePicker = true
            }
        }
        .sheet(isPresented: $showingImagePicker, onDismiss: loadImage) {
            // ImagePicker 将会把用户选择的图片保存到 inputImage 属性中
            ImagePicker(image: $inputImage)
        }
    }
    
    func loadImage() {
        // 将 inputImage 转换为 image
        guard let inputImage = inputImage else {
            return
        }
        
        image = Image(uiImage: inputImage)
    }
}
```


## 包装 UIView 到 SwiftUI 中

MKMapView 是 MapKit 中提供的一种 UIView，用于展示地图。这里演示一下怎么把 MKMapView 包装到 SwiftUI 当中。


```swift
import SwiftUI
import MapKit

// 要桥接的目标是 UIView 而非 UIViewController
// 所以需要实现的协议是 UIViewRepresentable 而非 UIViewControllerRepresentable
struct MapView: UIViewRepresentable {
    func makeUIView(context: Context) -> MKMapView {
        let mapView = MKMapView()
        
        // 将 MKMapView 的事件委托给 Coordinator 处理
        mapView.delegate = context.coordinator
        
        // 在地图上添加个小红点
        let annotation = MKPointAnnotation()
        annotation.title = "London"
        annotation.subtitle = "Capital of England"
        annotation.coordinate  = CLLocationCoordinate2D(latitude: 51.5, longitude: -0.13)
        mapView.addAnnotation(annotation)
        
        return mapView
    }
    
    func updateUIView(_ uiView: MKMapView, context: Context) {
        
    }
    
    func makeCoordinator() -> Coordinator {
        Coordinator(self)
    }
    
    // NSObject 是 UIKit 中所有对象的父类
    // MKMapViewDelegate 实现这个协议表示可以接受来自 MKMapView 的委托
    class Coordinator: NSObject, MKMapViewDelegate {
        var parent: MapView
        
        init(_ parent: MapView) {
            self.parent = parent
        }
        
        // 覆写这个方法可以接收到可视区域变化事件
        func mapViewDidChangeVisibleRegion(_ mapView: MKMapView) {
            print(mapView.centerCoordinate)
        }
        
        // 覆写这个方法可以修改小红点的样式
        func mapView(_ mapView: MKMapView, viewFor annotation: MKAnnotation) -> MKAnnotationView? {
            let view = MKPinAnnotationView(annotation: annotation, reuseIdentifier: nil)
            view.canShowCallout = true
            return view
        }
    }
}
```

包装好后就可以在界面中使用了:

```swift
struct ContentView: View {
    
    var body: some View {
        MapView()
            .ignoresSafeArea()
    }
}
```

![]( {{site.url}}/asset/swiftui-mapview.png )


## 保存图片到相册

要保存图片，首先需要给添加权限配置。按照下图所示打开项目设置后，在 "Custom iOS Target Properties" 处右键选择 "Add Row"，然后选择 Key 为 "Privacy - Photo Library Additions Usage Description"，接着在 value 处输入你的描述内容，比如 "We want to save the filtered photo."。添加了这个属性后，iOS 就会在合适的时候向用户索要相册权限。

![]( {{site.url}}/asset/swiftui-privacy-photo-library.png )

接着只要调用 `UIImageWriteToSavedPhotosAlbum()` 方法，就能把 `UIImage` 类型的图片保存到用户相册了:

```swift
// 第一个参数是要保存的图片
// 第二个参数是一个对象，用于接收处理结果，比如用户可能会拒绝保存。这个对象必须是实现了 NSObject 协议的 class 类型。
// 第三个参数是对象中的某个方法的名字。。。，用于接收处理结果。
// 第四个参数是个类似于上下文的东西，如果传入了这个参数，那么在接收处理结果时，可以再把这个值取出来。。。
UIImageWriteToSavedPhotosAlbum(inputImage, nil, nil, nil)
```

上面的代码只传入了第一个参数，因此无法获取处理结果。以下是可以处理结果的例子:

```swift
class ImageSaver: NSObject {
    var successHandler: (() -> Void)?
    var errorHandler: ((Error) -> Void)?
    
    func writeToPhotoAlbum(image: UIImage) {
        UIImageWriteToSavedPhotosAlbum(image, self, #selector(saveError), nil)
    }
    
    @objc func saveError(_ image: UIImage, didFinishSavingWithError error: Error?, contextInfo: UnsafeRawPointer) {
        // 把结果转发到 handler 上
        if let error = error {
            errorHandler?(error)
        } else {
            successHandler?()
        }
    }
}


// 每次使用的时候传建一个 ImageSaver() 实例
let imageSaver = ImageSaver()
imageSaver.successHandler = {
    print("Success!")
}
imageSaver.errorHandler = {
    print("Oops: \($0.localizedDescription)")
}
imageSaver.writeToPhotoAlbum(image: inputImage)
```


## Document 文件夹

Document 文件夹是随应用一起创建的文件夹，也就是所说的 "文稿" 文件夹。这里的文件会被 iCloud 自动备份。

```swift
// 使用 FileManager 获取 document 文件夹的路径，第一个路径就是我们需要的路径
let urls = FileManager.default.urls(for: .documentDirectory, in: .userDomainMask)
let documentDirUrl = urls[0]

let fileUrl = documentDirUrl.appendingPathComponent("message.txt")

// 写入文件
do {
    let str = "Test Message"

    // atomically 参数会让文件以原子的方式写入
    // 在这种模式下，iOS 会先把内容写入到一个临时文件，再将文件重命名为相应的文件名，从而保证不会有读写的线程安全问题
    try str.write(to: fileUrl, atomically: true, encoding: .utf8)
} catch {
    print(String(describing: error))
}

// 读取文件
do {
    let str = try String(contentsOf: fileUrl)
    print(str)
} catch {
    print(String(describing: error))
}
```


## 使用 TouchID 和 FaceID

使用 TouchID 和 FaceID 可以鉴定用户是否合法，目前这部分 API 是由 Objective-C 封装的，这里演示一下怎么包装给 SwiftUI 使用。

首先需要添加权限配置:

![]( {{site.url}}/asset/swiftui-faceid-privacy.png )

接着调用相关方法即可:

```swift
import SwiftUI
// 导入 LocalAuthentication 包
import LocalAuthentication

struct ContentView: View {
    @State private var canEvaluatePolicy = false
    @State private var isUnlocked = false
    
    var body: some View {
        VStack {
            if !canEvaluatePolicy {
                Text("Cannot evaluate policy")
            } else {
                if self.isUnlocked {
                    Text("Unlocked")
                } else {
                    Text("Locked")
                }
            }
        }
        .onAppear(perform: authenticate)
    }
    
    func authenticate() {
        // LAContext 用于执行鉴权操作
        let context = LAContext()
        var error: NSError?
        
        // 检查设备是否支持使用 TouchID 或 FaceID 鉴权
        if !context.canEvaluatePolicy(.deviceOwnerAuthenticationWithBiometrics, error: &error) {
            return
        }
        canEvaluatePolicy = true
        
        // 发起鉴权操作
        let reason = "We need to unlock your data."
        context.evaluatePolicy(.deviceOwnerAuthenticationWithBiometrics,
                               localizedReason: reason) { success, authenticationError in
            // 当前处于另一个线程，需要网主线程投递一个异步任务
            DispatchQueue.main.async {
                if success {
                    // 鉴权成功
                    self.isUnlocked = true
                } else {
                    // 鉴权失败
                    print(String(describing: authenticationError))
                }
            }
        }
    }
}
```

在模拟器中，需要通过菜单栏中的 Features -> FaceID -> Enrolled 来启用 FaceID 功能。


## 使用 '+' 连接多个 TextView

```swift
List(pages, id: \.pageid) { page in

    // 使用 '+' 连接多个 TextView, 可以让不同风格的文字显示在同一行
    Text(page.title)
        .font(.headline)
    + Text(": ") +
    Text("Page description here")
        .italic()
}
```


## 在实现 ObservableObject 协议的时候，让 computed property 主动发出事件

有时候需要为已有的类型包装 computed property, 并为其实现 ObservableObject 协议，这时候需要考虑怎么让 computed property 发出事件:

```swift
// 扩展 MKPointAnnotation, 使其支持 ObservableObject 协议
extension MKPointAnnotation: ObservableObject {

    // 因为原本的 title, subtitle 属性是 Optional 类型，无法绑定到 SwiftUI 上面
    // 为 MKPointAnnotation 添加两个 computed property, 便于做绑定
    public var wrappedTitle: String {
        get {
            self.title ?? "Unknown value"
        }
        
        set {
            // 通过 objectWillChange 来主动发出属性变化事件
            // 注意 objectWillChange.send() 必须在真正改变值之前调用，这样 SwiftUI 有机会执行动画
            objectWillChange.send()
            title = newValue
        }
    }
    
    public var wrappedSubtitle: String {
        get {
            self.subtitle ?? "Unknown value"
        }
        
        set {
            objectWillChange.send()
            subtitle = newValue
        }
    }
}
```

对于普通 property，也可以使用 `objectWillChange.send()` 来手动发送事件，即使这个 property 没有被 `@Published` 所修饰：

```swift
class DelayedUpdater: ObservableObject {
    // 注意 value 属性没有被 `@Published` 修饰，而是手动触发了事件
    var value = 0 {
        willSet {
            objectWillChange.send()
        }
    }
    
    init() {
        for i in 1...10 {
            DispatchQueue.main.asyncAfter(deadline: .now() + Double(i)) {
                self.value += 1
            }
        }
    }
}

struct ContentView: View {
    @StateObject var updater = DelayedUpdater()
    
    var body: some View {
        Text("value: \(updater.value)")
    }
}
```

还有一种常见的情况是，你有一个容器类型实现了 ObservableObject，这个容器类型中包含一个数组，并且为这个数组设置了 `@Published` 包装器。此时向数组里塞东西，UI 会自动刷新，但修改数组内的元素时，UI 不会自动刷新。此时就可以使用 `objectWillChange.send()` 方法来主动触发 UI 的刷新。

```swift
class Student: Identifiable {
    let id = UUID()
    var name: String
    var age: Int
    
    init(name: String, age: Int) {
        self.name = name
        self.age = age
    }
}

class Students: ObservableObject {
    @Published var all = [Student]()
}

struct ContentView: View {
    @StateObject var students = Students()
    
    var body: some View {
        VStack {
            List(students.all) { student in
                Text("\(student.name) (\(student.age))")
            }
        }
        
        HStack {
            Button("Add") {
                // 往数组里塞东西，因为数组设置了 @Published, 因此 UI 会自动刷新
                students.all.append(
                    Student(name: "Dongdada\(Int.random(in: 1...100))",
                            age: Int.random(in: 1...100))
                )
            }
            .padding()
            Spacer()
            
            Button("Modify") {
                // 如果直接设置数组中元素的属性，UI 不会被刷新
                // 需要手动调用以下 objectWillChange.send(), 触发 UI 刷新
                // students.objectWillChange.send()
                students.all.randomElement()?.name = "Jhann\(Int.random(in: 1...100))"
            }
            .padding()
        }
    }
}
```


## 自定义属性包装器

之前介绍 `@State` 的时候提过，property wrapper 其实会把属性包装成一个结构体实例。下面演示一下如何自定义 property wrapper:

```swift
// 通过 @propertyWrapper 自定义属性包装器
// 它能将原本的属性包装成一个结构体
@propertyWrapper
struct NonNegative<Value: BinaryInteger> {
    var value: Value
    
    // 构造函数需要传入属性的原始值
    init(wrappedValue: Value) {
        if wrappedValue < 0 {
            self.value = 0
        } else {
            self.value = wrappedValue
        }
    }
    
    // 另外需要提供一个名为 wrappedValue 的 computed property
    // 每次访问属性，都是通过访问 wrappedValue 来实现
    var wrappedValue: Value {
        get {
            value
        }
        
        set {
            if newValue < 0 {
                value = 0
            } else {
                value = newValue
            }
        }
    }
}

struct User {
    @NonNegative var score = 0
}

var user = User()
user.score += 10
print(user.score)       // 10

user.score -= 20
print(user.score)       // 0
```


## Result 类型

使用 Result 类型可以表示 “要么成功要么失败” 的意思。比如下面的代码，当 data 不为空表示成功，当 error 不为 nil 的时候表示失败，但是其它情况不知道怎么处理。

```swift
struct ContentView: View {
    var body: some View {
        Text("Hello, World!")
            .onAppear {
                let url = URL(string: "https://www.apple.com")!
                URLSession.shared.dataTask(with: url) { data, response, error in
                    if data != nil {
                        print("We got data!")
                    } else if let error = error {
                        print(error.localizedDescription)
                    }
                }
                .resume()
            }
    }
}
```

可以使用 Result 类型来包装一下:

```swift
enum NetworkError: Error {
    case badURL, requestFailed, unknown
}

struct ContentView: View {
    var body: some View {
        Text("Hello, World!")
            .onAppear {
                fetchData(from: "https://www.apple.com") { result in
                    switch result {
                    case .success(let str):
                        print(str)
                    case .failure(let error):
                        switch error {
                        case .badURL:
                            print("Bad URL")
                        case .requestFailed:
                            print("Network problems")
                        case .unknown:
                            print("Unknown error")
                        }
                    }
                }
            }
    }
    
    // 注意 completion 参数的类型使用了 @escaping 修饰，这表示这个参数可能会在这个方法的作用域以外被使用
    func fetchData(from urlString: String,
                   completion: @escaping (Result<String, NetworkError>) -> Void) {
        // 成功和失败的情况将分别使用 .success(), .failed() 来包装

        guard let url = URL(string: urlString) else {
            completion(.failure(.badURL))
            return
        }
        
        URLSession.shared.dataTask(with: url) { data, response, error in
            // 将执行结果抛到主线程
            DispatchQueue.main.async {
                if let data = data {
                    // 执行成功
                    let stringData = String(decoding: data, as: UTF8.self)
                    completion(.success(stringData))
                } else if error != nil {
                    // 执行失败
                    completion(.failure(.requestFailed))
                } else {
                    // 其它情况则认为是未知错误
                    completion(.failure(.unknown))
                }
            }
        }.resume()
    }
}
```


# 常见控件


## TextEditor

普通的 TextField 控件只能显示单行文字，如果需要编辑多行文字，可以使用 TextEditor 控件:

```swift
struct ContentView: View {
    @AppStorage("notes") private var notes = ""
    
    var body: some View {
        NavigationView {
            TextEditor(text: $notes)
                .navigationTitle("Notes")
                .padding()
        }
    }
}
```


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

以下代码展示了如何通过 `onDelete()` modifier 为 ForEach 增加删除功能，你只需要实现 `onDelete()` 所要求的方法来删除数据，SwiftUI 会帮你实现左滑删除的界面效果。

```swift
struct ContentView: View {
    @State private var numbers = [Int]()
    @State private var currentNumber = 1
    
    var body: some View {
        NavigationView {
            VStack {
                List {
                    ForEach(numbers, id: \.self) {
                        Text("Row \($0)")
                    }
                    .onDelete(perform: removeRows)
                }
                
                Button("Add Number") {
                    numbers.append(currentNumber)
                    currentNumber += 1
                }
            }
            .toolbar {
                // 添加一个编辑按钮，这样就可以批量删除
                EditButton()
            }
        }
    }
    
    func removeRows(at offsets: IndexSet) {
        // 删除实际数据
        numbers.remove(atOffsets: offsets)
    }
}
```

![]( {{site.url}}/asset/swiftui-foreach-ondelete.png )


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


## Sheet

Sheet 也是显示对话框，但长得跟 Alert 不一样:

```swift
struct SecondView: View {
    let name: String

    // @Environment 里包含了许多有用的工具，`.dismiss` 的作用是告诉 SwiftUI 将当前视图关闭
    @Environment(\.dismiss) var dismiss
    
    var body: some View {
        VStack {
            Text("Helllo, \(name)")
            Button("Dismiss") {
                dismiss()
            }
        }
    }
}

struct ContentView: View {
    @State private var showingSheet = false
    
    var body: some View {
        Button("Show Sheet") {
            showingSheet.toggle()
        }
        .sheet(isPresented: $showingSheet) {
            // 自定义对话框里的内容
            SecondView(name: "DongDada")
        }
    }
}
```

用户可以通过向下滑动的方式把 sheet 关闭，以上代码借助 `@Environment(.\dismiss)` 在 sheet 上提供了一个关闭按钮。

![]( {{site.url}}/asset/swiftui-sheet.png )


## ActionSheet

ActionSheet 跟 Sheet 有点像，只是可以显示许多按钮：

```swift
struct ContentView: View {
    @State private var showingActionSheet = false
    @State private var backgroundColor = Color.white
    
    var body: some View {
        Text("Hello, world!")
            .frame(width: 300, height: 300)
            .background(backgroundColor)
            .onTapGesture {
                self.showingActionSheet = true
            }
            .actionSheet(isPresented: $showingActionSheet) {
                ActionSheet(title: Text("Change background"),
                            message: Text("Select a new color"),
                            buttons: [
                                .default(Text("Red")) {
                                    self.backgroundColor = .red
                                },
                                .default(Text("Green")) {
                                    self.backgroundColor = .green
                                },
                                .default(Text("Blue")) {
                                    self.backgroundColor = .blue
                                },
                                .cancel()
                            ])
            }
    }
}
```

![]( {{site.url}}/asset/swiftui-action-sheet.png )


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


## Image

顾名思义，Image 用于加载图片。把你的素材拖进 Assets.xcassets 当中，随后就可以使用 Image 将其显示出来:

```swift
struct ContentView: View {
    var body: some View {
        ZStack {
            Color.gray
            
            Image("Sleeping")
                // frame() 只是设置了 view 本身的大小，图片的尺寸是不会被改变的。
                // 如果希望图片大小与 frame() 的尺寸一致，需要设置 resiable() modifier
                .resizable()
                .frame(width: 200, height: 300)
                .background(.red)
        }
        .ignoresSafeArea()
    }
}
```

![]( {{site.url}}/asset/swiftui-image-resizable.png )

可以看到图片本身是正方形，但 View 的尺寸设置成了长方形，所以图片被拉伸了。

为了避免图片被拉伸，可以使用 `scaledToFit()` modifier, 他会在图片不会拉伸的情况下，让图片尽量填充满 View 的空间。

```swift
struct ContentView: View {
    var body: some View {
        ZStack {
            Color.gray
            
            Image("Sleeping")
                .resizable()
                .scaledToFit()
                .frame(width: 200, height: 300)
                .background(.red)
        }
        .ignoresSafeArea()
    }
}
```

![]( {{site.url}}/asset/swiftui-image-scaletofit.png )


也可以使用 `scaleToFill` modifier，它会在图片不被拉伸的情况下，让自己填满整个 View 空间，即使这样做会让图片超出 View 空间限制。

![]( {{site.url}}/asset/swiftui-image-scalltofill.png )


### 调整图片的 插值 方式

如果把尺寸很小的图片放大到很大，SwiftUI 默认的插值方式会让像素边缘看起来比较模糊:

![]( {{site.url}}/asset/swiftui-interpolation-default.png )

只需使用 `.interpolation()` modifier 就可以修改图片放大时所采用的的插值方式。比如下面代码将插值方式设置为 `.none`，这样像素就会清晰地展示了：

```swift
struct ContentView: View {
    
    var body: some View {
        Image("example")
            .interpolation(.none)
            .resizable()
            .scaledToFit()
            .frame(maxHeight: .infinity)
            .background(.black)
            .ignoresSafeArea()
    }
}
```

![]( {{site.url}}/asset/swiftui-interpolation-none.png )


## GeometryReader

GeometryReader (几何读取器) 也是个 View，它能够让你获取一些关于尺寸大小的之类的信息，比如容器大小多大，当前 view 的位置在哪里，是否有 safe are 等等。

```swift
struct ContentView: View {
    var body: some View {
        ZStack {
            Color.gray
            
            GeometryReader { geo in
                Image("Sleeping")
                    .resizable()
                    .scaledToFit()
                    // 把 Image View 的宽度设置为容器的 80%，高度设置为 300
                    .frame(width: geo.size.width * 0.8, height: 300)
                .background(.red)
            }
        }
        .ignoresSafeArea()
    }
}
```

![]( {{site.url}}/asset/swiftui-geometryreader-default.png )

可以看到 Image View 按照我们的要求设置了合适的尺寸，但显示在了屏幕的左上角。这是因为 GeometryReader 会自动把内部的 View 放到左上角。


## ScrollView

ScrollView 可以为元素添加滚动效果。

```swift
struct ContentView: View {
    var body: some View {
        ScrollView {
            // 注意这里使用了 LazyVStack
            // LazyVStack 与普通 VStack 区别在于会按需创建 View 实例
            LazyVStack(spacing: 10) {
                ForEach(0..<100) {
                    Text("Item \($0)")
                        .font(.title)
                }
            }
            .frame(maxWidth: .infinity)
        }
    }
}
```

![]( {{site.url}}/asset/swifui-scrollview.gif )


## NavigationView

NavigationView 可以在 iOS 的左上角显示一个标题。此外，当它和 NavigationLink 一起使用的时候，可以实现页面跳转的效果：

```swift
struct ContentView: View {
    var body: some View {
        NavigationView {
            List(0..<100) { row in
                NavigationLink {
                    // 跳转后的页面内容
                    Text("Detail \(row)")
                } label: {
                    // 每一行的标题
                    Text("Row \(row)")
                }
            }
            .navigationTitle("SwiftUI")
        }
    }
}
```

![]( {{site.url}}/asset/swiftui-navigation-view.gif )


## LazyVGrid, LazyHGrid

顾名思义，这两个 view 用于显示表格:

```swift
struct ContentView: View {
    var body: some View {
        ScrollView {
            // 创建一个有 3 列的表格
            LazyVGrid(columns: [
                GridItem(.fixed(80)),
                GridItem(.fixed(80)),
                GridItem(.fixed(80))
            ]) {
                ForEach(0..<1000) {
                    Text("Item \($0)")
                }
            }
        }
    }
}
```

此外还可以为表格的列数指定为自定义数量，这样 SwiftUI 会按照屏幕大小展示合适的列数：

```swift
struct ContentView: View {
    var body: some View {
        ScrollView {
            LazyVGrid(columns: [
                GridItem(.adaptive(minimum: 80, maximum: 120))
            ]) {
                ForEach(0..<1000) {
                    Text("Item \($0)")
                }
            }
        }
    }
}
```

![]( {{site.url}}/asset/swiftui-lazyvgrid.png )


## ImagePaint

ImagePaint 不是空间，是一种 ShapeStyle. 它能够用图片来填充一块区域:

```swift
struct ContentView: View {
    
    var body: some View {
        VStack {
            Text("Lena")
                .frame(width: 128, height: 128, alignment: .topLeading)
                .foregroundColor(.white)
                // Image 可以直接设置到 background 上，不过没法控制缩放、位置参数
                .background(Image("Example"))
                .clipped()
            
            Text("Lena")
                .frame(width: 128, height: 128, alignment: .topLeading)
                .foregroundColor(.white)
                // ImagePaint 做背景的话，可以控制缩放比例
                .background(ImagePaint(image: Image("Example"), scale: 0.25))
            
            Capsule()
                // ImagePaint 还可以填充边 (stroke 是笔画的意思), Image 是做不到的
                .strokeBorder(ImagePaint(image: Image("Example"), scale: 0.1), lineWidth: 30)
                .frame(width: 200, height: 100)
            
            Text("Lena")
                .frame(width: 128, height: 128, alignment: .topLeading)
                .foregroundColor(.white)
                .background(
                    // 使用 sourceRect 参数，可以裁剪出图片的一部分用来做填充
                    ImagePaint(image: Image("Example"),
                               sourceRect: CGRect(x: 0, y: 0, width: 0.5, height: 0.5),
                               scale: 0.5)
                )
        }
    }
}
```

![]( {{site.url}}/asset/swiftui-imagepaint.png )


## AsyncImage

AsyncImage 能够自动下载并显示图片:

```swift
struct ContentView: View {

    var body: some View {
        VStack {
            AsyncImage(url: URL(string: "https://hws.dev/img/logo.png")) { phase in
                if let image = phase.image {
                    // 如果下载成功，则显示图片
                    image
                        .resizable()
                        .scaledToFit()
                } else if phase.error != nil {
                    // 如果下载失败，则显示错误信息
                    Text("There was an error loading the image.")
                } else {
                    // 下载过程中显示 loading 界面
                    ProgressView()
                }
            }
            .frame(width: 200, height: 200)
            .border(.red, width: 1)
        }
    }
}
```


## AnyView

以下代码会编译出错，因为 tossResult 的返回值类型是个不透明类型 `some View`, 不透明类型要求返回值类型是固定的，多次调用应该始终返回相同的类型:

```swift
struct ContentView: View {
    @Environment(\.managedObjectContext) var moc
    
    // 报错: Function declares an opaque return type, but has no return statements in its body from which to infer an underlying type
    var tossResult: some View {
        if Bool.random() {
            Image(systemName: "star")
        } else {
            Text("Better luck next time")
        }
    }

    var body: some View {
        tossResult
    }
}
```

一种解决方法是使用 `@ViewBuilder` 包装 tossResult 属性。`@ViewBuilder` 会把不同类型的 View 包装到单个 View 类型中。

```swift
@ViewBuilder var tossResult: some View {
    if Bool.random() {
        Image(systemName: "star")
    } else {
        Text("Better luck next time")
    }
}
```

View 的 body 属性已经经过了 `@ViewBuilder` 的包装，所以你也可以直接把 tossResult 直接拆分成一个自定义 View:

```swift
struct TossResult: View {
    var body: some View {
        if Bool.random() {
            Image("laser-show")
                .resizable()
                .scaledToFit()
        } else {
            Text("Better luck next time")
                .font(.title)
        }
    }
}
```

另一种解决方法是使用 `Group` 把不同类型放在一起，使得 tossResult 的类型固定为 Group:

```swift
var tossResult: some View {
    Group {
        if Bool.random() {
            Image("laser-show")
                .resizable()
                .scaledToFit()
        } else {
            Text("Better luck next time")
                .font(.title)
        }
    }
    .frame(width: 400, height: 300)
}
```

还有一种解决方法是使用 `AnyView` 来包装 View:

```swift
var tossResult: some View {
    if Bool.random() {
        return AnyView(Image(systemName: "star").resizable().scaledToFit())
    } else {
        return AnyView(Text("Better luck next time").font(.title))
    }
}
```

`AnyView` 和 `Group` 都能解决问题，不过 `AnyView` 是专门为解决这个问题设计的，语义更清晰一些。


## TabView

TabView 可以在屏幕底部放置 tab 按钮，如以下代码所示，你需要把各个 tab 的子视图放到 TabView 里面，然后使用 `.tabItem()` modifier 来设置这个子视图的 tab 按钮风格。按钮风格只支持图片和文字。

```swift
struct ContentView: View {
    var body: some View {
        TabView {
            ZStack {
                Color.red
                Text("Tap1")
                    .foregroundColor(.white)
            }
            .tabItem {
                Image(systemName: "star")
                Text("One")
            }
            
            ZStack {
                Color.green
                Text("Tab2")
                    .foregroundColor(.white)
            }
            .tabItem {
                Image(systemName: "star.fill")
                Text("Two")
            }
        }
    }
}
```

![]( {{site.url}}/asset/swiftui-tabview.png )

如果希望通过属性来控制 tab 的切换，则可以参考以下代码：

```swift
struct ContentView: View {
    // 追踪当前正在显示的 tab
    @State private var selectedTab = 0
    
    var body: some View {
        TabView(selection: $selectedTab) {
            ZStack {
                Color.red
                Text("Tap1")
                    .foregroundColor(.white)
            }
            .onTapGesture {
                // 切换到 Tab2
                self.selectedTab = 1
            }
            .tabItem {
                Image(systemName: "star")
                Text("One")
            }
            // 给这个子视图加上标签 0
            .tag(0)
            
            ZStack {
                Color.green
                Text("Tab2")
                    .foregroundColor(.white)
            }
            .tabItem {
                Image(systemName: "star.fill")
                Text("Two")
            }
            .onTapGesture {
                // 切换到 Tab1
                self.selectedTab = 0
            }
            // 给这个子视图加上标签 1
            .tag(1)
        }
    }
}
```

## ContextMenu 上下文菜单

使用 `.contextMenu()` modifier 可以为 View 添加上下文菜单:

```swift
struct ContentView: View {
    @State private var backgroundColor = Color.red
    
    var body: some View {
        VStack {
            Text("Hello, World!")
                .padding()
                .background(backgroundColor)
            
            Text("Change Color")
                .padding()
                .contextMenu {
                    Button(role: .destructive) {
                        backgroundColor = .red
                    } label: {
                        // 只能使用 systemImage 为上下文菜单显示图片
                        // 并且图片的颜色无法修改
                        // 只能通过设置 .destructive 来标识这个选项是危险操作
                        Label("Red", systemImage: "trash")
                    }
                    
                    Button("Green") {
                        backgroundColor = .green
                    }
                    
                    Button("Blue") {
                        backgroundColor = .blue
                    }
                }
        }
    }
}
```

![]( {{site.url}}/asset/swiftui-context-menu.png )



# 布局

## 布局规则

SwiftUI 的布局过程分为三个简单的步骤：
1. Parent View 给 Child View 一个建议的尺寸。
2. Child View 参考 Parent View 提供的尺寸，选择一个合适于自己的尺寸，对于这个尺寸，Parent View 必须接受。
3. 根据 Child View 给出的尺寸，Parent View 将其放到合适的位置。

来看一个简单的例子:

```swift
struct ContentView: View {
    var body: some View {
        Text("Hello, World!")
            .background(.red)
    }
}
```

由于 `.background()` modifier 实际上会产生一个新的 View，所以上述代码中涉及到了 ContentView, BackgroundView, TextView 三个 View，上述代码的布局过程是:

![]( {{site.url}}/asset/swiftui-layout.svg )

这个布局过程也解释了为什么 modifier 的顺序会影响布局结果，比如下面的代码:

```swift
Text("Hello, World!")
    .background(.red)
    .padding()
```

其布局过程中，由于 `.background()` 排在 `.padding()` 前面，因此 Background View 并没有附加 padding 带来的尺寸变化:

![]( {{site.url}}/asset/swiftui-layout-order.svg )


这里有个重要的概念叫做 *layout neutral(布局中立)*, 上述所提到的 ContentView, BackgroundView, PaddingView 都是布局中立的，它们自身对尺寸大小没有要求，因此只会询问 Child View 的意见。而 TextView 不是布局中立的，它会根据自己的文字数量、字体设置等决定自己需要多少空间。

如果整个 View 树都是布局中立的，那么最终它会占用所有的可用空间，比如 Color 也是一个布局中立的 View, 因此下面代码将导致 Color 占据整个可用空间:

```swift
var body: some View {
    Color.red
}
```

## 对齐规则

### .frame() modifier 中的 alignment 参数

`.frame()` modifier 会将原有 View 包装到一个固定大小的 Frame View 里面，默认情况下把原有 View 放到中心。不过你也可以通过它的 alignment 参数来控制对齐规则:

```swift
    var body: some View {
        Text("Hello, world!")
            .padding()
            .background(.red)
            // 将 Background View 包装到 300 x 300 的 Frame View 里面，随后把 Background View 放到左上角
            .frame(width: 300, height: 300, alignment: .topLeading)
            .background(.gray)
    }
```

### stack 中的 alignment 参数

这个很好理解，VStack 默认会在水平方向上居中显示 Child View，你可以通过 alignment 参数来控制是否需要左对齐:

```swift
struct ContentView: View {
    var body: some View {
        VStack(alignment: .leading) {
            Text("Hello, world!")
            Text("This is a longer line of text")
        }
        .background(.red)
        .frame(width: 400, height: 400)
        .background(.blue)
    }
}
```

### .alignmentGuide() modifier

每当 VStack 去对齐它的 Child View 的时候，它都会询问 Child View 需要按照那条边来对齐。默认情况下 Child View 会使用左侧的边来对齐(其实取决于语言，有一些国家的文字是从右往左的，这时候默认是使用右侧的边)。

你可以为 VStack 的 Child View 提供一个`.alignmentGuide()` modifier 来自定义对齐逻辑。

`.alignmentGuide()` 有两个参数，第一个参数表示你希望修改哪条边的对齐规则，第二个参数是个 closure, 它将返回修改后的对齐规则。closure 包含一个 ViewDimensions 的参数，它包含了当前 View 的宽和高，并且能够通过这个参数来获取 View 的各条边。

比如我们可以将左侧的边修改为右侧的边:

```swift
struct ContentView: View {
    var body: some View {
        // VStack 设置了按照左侧对齐
        VStack(alignment: .leading) {
            Text("Hello, world!")
            // 修改左侧边的对齐规则
            .alignmentGuide(.leading) { d in
                // 把右侧的边返回
                d[.trailing]
            }
            Text("This is a longer line of text")
        }
        .background(.red)
    }
}
```

此时 VStack 就会将第一个 Text 的右侧边和第二个 Text 的左侧边对齐:

![]( {{site.url}}/asset/swiftui-alignment-guide.png )


你也可以忽略 ViewDimensions 参数，自己计算对齐规则:

```swift
struct ContentView: View {
    var body: some View {
        VStack(alignment: .leading) {
            ForEach(0..<10) { position in
                Text("Number \(position)")
                    .alignmentGuide(.leading) { _ in
                        CGFloat(position) * -10
                    }
            }
        }
        .background(.red)
        .frame(width: 400, height: 400)
        .background(.blue)
    }
}
```

![]( {{site.url}}/asset/swiftui-alignment-guide-hardcode.png )



# 绘图

## Path

Path 也是一种 View，有点特别的是它的构造函数，它的构造函数中需要传入一个 closure, 这个 closure 只有一个参数，也是 Path 类型。SwiftUI 会创建一个空的 Path 实例作为这个参数，你只需要修改这个 path 参数就可以进行绘制了。

```swift
struct ContentView: View {

    var body: some View {
        Path { path in
            path.move(to: CGPoint(x: 200, y: 100))
            path.addLine(to: CGPoint(x: 100, y: 300))
            path.addLine(to: CGPoint(x: 300, y: 300))
            path.addLine(to: CGPoint(x: 200, y: 100))
        }
        // 指定线条的风格 (stroke 是笔画的意思)
        // - lineCap 表示如果线条端点没有与其它线条连接起来，应该如何绘制
        // - lineJoin 表示线条与线条之间发生连接时，应该如何绘制
        .stroke(.blue, style: StrokeStyle(lineWidth: 10, lineCap: .round, lineJoin: .round))
    }
}
```

![]( {{site.url}}/asset/swiftui-path.png )

## 通过实现 Shape 协议来自定义形状

通过 Path 可以自定义出你要的形状，Shape 的作用在于它可以为 Path 提供相对坐标，原点在 **左上角**。

```swift
struct Triangle: Shape {
    func path(in rect: CGRect) -> Path {
        var path = Path()
        
        path.move(to: CGPoint(x: rect.midX, y: rect.minY))
        path.addLine(to: CGPoint(x: rect.minX, y: rect.maxY))
        path.addLine(to: CGPoint(x: rect.maxX, y: rect.maxY))
        path.addLine(to: CGPoint(x: rect.midX, y: rect.minY))
        
        return path
    }
}

struct ContentView: View {

    var body: some View {
        Triangle()
            .stroke(.blue, style: StrokeStyle(lineWidth: 10, lineCap: .round, lineJoin: .round))
            .frame(width: 200, height: 200)
    }
}
```

![]( {{site.url}}/asset/swiftui-shape.png )


## InsettableShape 内缩的形状

Path 不止可以画直线，也可以画圆弧:

```swift
// 圆弧
struct Arc: Shape {
    var startAngle: Angle
    var endAngle: Angle
    var clockwise: Bool
    
    func path(in rect: CGRect) -> Path {
        // SwiftUI 里面，0 度的位置不是在正上方，而是在正右侧，看起来有点反人类，所以这里做了转换，把 0 度位置调整到了正上方
        let rotationAdjustment = Angle.degrees(90)
        let modifiedStart = startAngle - rotationAdjustment
        let modifiedEnd = endAngle - rotationAdjustment
        
        var path = Path()
        path.addArc(center: CGPoint(x: rect.midX, y: rect.midY), radius: rect.width / 2,
                    startAngle: modifiedStart, endAngle: modifiedEnd, 
                    // SwiftUI 里的顺逆时针是反的，所以这里做了调整
                    clockwise: !clockwise)
        
        return path
    }
}

struct ContentView: View {

    var body: some View {
        Arc(startAngle: .degrees(0), endAngle: .degrees(110), clockwise: true)
            .stroke(.blue, style: StrokeStyle(lineWidth: 10, lineCap: .round, lineJoin: .round))
            .frame(width: 200, height: 200)
            .background(.gray)
    }
}
```

![]( {{site.url}}/asset/swiftui-shape-arc.png )

可以看到上图中圆弧的一部分超过了边界。这是 `.stroke()` modifier 的正常表现。如果希望圆弧内缩到矩形内，需要让 Arc 实现 InsettableShape 协议，这里的 insettable 是可嵌入的意思。实现这个协议后，SwiftUI 会把需要内缩的尺寸告诉我们，我们可以在绘制图形的时候把内缩也考虑进去：

```swift
struct Arc: InsettableShape {
    var startAngle: Angle
    var endAngle: Angle
    var clockwise: Bool
    var insetAmount = 0.0
    
    // InsettableShape 需要实现一个 inset 方法，SwiftUI 会调用这个方法，把需要内缩的尺寸告诉我们
    func inset(by amount: CGFloat) -> some InsettableShape {
        var arc = self
        arc.insetAmount += amount
        return arc
    }
    
    func path(in rect: CGRect) -> Path {
        let rotationAdjustment = Angle.degrees(90)
        let modifiedStart = startAngle - rotationAdjustment
        let modifiedEnd = endAngle - rotationAdjustment
        
        var path = Path()

        // 在绘制的时候需要考虑到内缩尺寸，对于这个例子来说，需要把半径减去内缩尺寸
        path.addArc(center: CGPoint(x: rect.midX, y: rect.midY), radius: rect.width / 2 - insetAmount,
                    startAngle: modifiedStart, endAngle: modifiedEnd, clockwise: !clockwise)
        
        return path
    }
}

struct ContentView: View {

    var body: some View {
        Arc(startAngle: .degrees(0), endAngle: .degrees(110), clockwise: true)
            // 对于 InsettableShape, 可以使用 `.strokeBorder()` modifier, 此时绘制的边界将不会超出矩形框
            .strokeBorder(.blue, style: StrokeStyle(lineWidth: 10, lineCap: .round, lineJoin: .round))
            .frame(width: 200, height: 200)
            .background(.gray)
    }
}
```


## CGAffineTransform

CGAffineTransform 可以对 Path 施加位移，旋转等效果:

```swift
struct Flower: Shape {
    func path(in rect: CGRect) -> Path {
        var path = Path()
        
        for number in stride(from: 0, to: Double.pi * 2, by: Double.pi / 8) {
            // 现在左上角创建一个 Path
            let originalPetal = Path(ellipseIn: CGRect(x: petalOffset, y: 0, width: petalWidth, height: rect.width / 2))
            
            // 将这个 Path 旋转一定角度
            let rotation = CGAffineTransform(rotationAngle: number)
            // 将这个 Path 移动到矩形框的中心
            let position = CGAffineTransform(translationX: rect.width / 2, y: rect.height / 2)
            // 将 CGAffineTransform 应用到 Path 上面，注意这里使用了 concatenating() 方法来合并两个 CGAffineTransform
            let rotatedPetal = originalPetal.applying(rotation.concatenating(position))
            
            // 保存移动过的 Path
            path.addPath(rotatedPetal)
        }
        
        return path
    }
    
    var petalOffset: Double = -20
    var petalWidth: Double = 100
}

struct ContentView: View {
    @State private var petalOffset = -20.0
    @State private var petalWidth = 100.0

    var body: some View {
        VStack {
            Flower(petalOffset: petalOffset, petalWidth: petalWidth)
                // 这里通过 FillStyle 为颜色填充设置了奇偶规则: 奇数个图形叠加会显示颜色，偶数个图形叠加的话不显示颜色
                .fill(.red, style: FillStyle(eoFill: true, antialiased: false))
            
            Text("Offset")
            Slider(value: $petalOffset, in: -40...40)
                .padding([.horizontal, .bottom])
            
            Text("Width")
            Slider(value: $petalWidth, in: 0...100)
                .padding([.horizontal, .bottom])
        }
    }
}
```

![]( {{site.url}}/asset/swiftui-CGAffineTransform.png )


## 使用 drawGroup() modifier 启用 Metal 渲染

以下代码绘制了 100 个圆环，每个圆环都施加了 LinearGradient 变换，随后使用一个 slider 来切换圆环的颜色。

如果不使用 `.drawingGroup()` modifier 的话，slider 的推动会比较卡。

使用了 `.drawingGroup()` modifier 之后，SwiftUI 会利用 GPU 进行离屏渲染，也就是渲染完一整张图片后再把图片刷新到屏幕上，而不是每渲染一个 View 就做一次刷新，通过这种方式可以提高渲染效率。

不过 `.drawingGroup()` modifier 不是每次都要用的，它在 View 比较少的情况下反而更慢，所以只有在发现渲染效率比较低的情况下才推荐使用 `.drawingGroup()`.

```swift
struct ColorCyclingCircle: View {
    var amount = 0.0
    var steps = 100
    
    var body: some View {
        ZStack {
            // 绘制 100 个圆环，每个圆环的颜色不同
            ForEach(0..<steps) { value in
                Circle()
                    // inset() modifier 会把原有图形向内缩指定的尺寸
                    .inset(by: Double(value))
                    .strokeBorder(
                        // 给每个圆环都施加线性渐变效果
                        LinearGradient(
                            gradient: Gradient(colors: [
                                color(for: value, brightness: 1),
                                color(for: value, brightness: 0.5)
                            ]), startPoint: .top, endPoint: .bottom),
                        lineWidth: 2
                    )
            }
        }
        // 如果不使用 .drawingGroup() modifier, 做切换会比较卡
        .drawingGroup()
    }
    
    func color(for value: Int, brightness: Double) -> Color {
        var targetHue = Double(value) / Double(steps) + amount
        if targetHue > 1 {
            targetHue -= 1
        }
        
        return Color(hue: targetHue, saturation: 1, brightness: brightness)
    }
}


struct ContentView: View {
    @State private var colorCycle = 0.0
    
    var body: some View {
        VStack {
            ColorCyclingCircle(amount: colorCycle)
                .frame(width: 300, height: 300)
            
            Slider(value: $colorCycle)
        }
    }
}
```


## blend, blue, saturation 效果:

```swift
struct ContentView: View {
    @State private var amount = 0.0
    
    var body: some View {
        VStack {
            VStack {
                ZStack {
                    Circle()
                        .fill(.red)
                        .frame(width: 100 * amount)
                        .offset(x: -20, y: -30)
                        // blend 用于控制一个 View 叠加到另一个 View 上面之后，应该如何混合颜色
                        .blendMode(.screen)

                    Circle()
                        .fill(.green)
                        .frame(width: 100 * amount)
                        .offset(x: 20, y: -30)
                        .blendMode(.screen)

                    Circle()
                        .fill(.blue)
                        .frame(width: 100 * amount)
                        .blendMode(.screen)
                }
                .frame(width: 200, height: 200)
                .background(.black)
                
                Text("颜色叠加效果")
            }
            .frame(maxWidth: .infinity)
            .border(.black, width: 1)
            
            VStack {
                Image("Example")
                    .resizable()
                    .scaledToFit()
                    .frame(width: 200, height: 200)
                    // 饱和度，表示色彩的鲜艳程度，饱和度越低，颜色看起来越灰暗
                    // 如果饱和度为 0, 颜色就彻底变成灰色了
                    .saturation(amount)
                    // 模糊效果
                    .blur(radius: (1 - amount) * 20)
                Text("饱和度和模糊效果")
            }
            .frame(maxWidth: .infinity)
            .border(.black, width: 1)
            
            Slider(value: $amount)
                .padding()
        }
    }
}
```

![]( {{site.url}}/asset/swiftui-blend.gif )



## 使用 animatableData 属性来为 Shape 变化添加动画

Shape 协议继承了 Animatable 协议，后者定义了一个 animatableData 属性。如果希望 Shape 的变化能够被施加动画效果，就需要实现这个属性的代码:

```swift
struct Trapezoid: Shape {
    var insetAmount: Double
    
    // 实现 animatableData 属性
    // 在施加动画的时候，animatableData 属性会被一点点修改，对于这个例子来说，也就是 insetAmount 属性会被一点点修改
    var animatableData: Double {
        get { insetAmount }
        set { insetAmount = newValue }
    }
    
    func path(in rect: CGRect) -> Path {
        var path = Path()
        
        path.move(to: CGPoint(x: 0, y: rect.maxY))
        path.addLine(to: CGPoint(x: insetAmount, y: rect.minY))
        path.addLine(to: CGPoint(x: rect.maxX - insetAmount, y: rect.minY))
        path.addLine(to: CGPoint(x: rect.maxX, y: rect.maxY))
        path.addLine(to: CGPoint(x: 0, y: rect.maxY))
        
        return path
    }
}

struct ContentView: View {
    @State private var amount = 50.0
    
    var body: some View {
        VStack {
            Trapezoid(insetAmount: amount)
                .frame(width: 200, height: 100)
                .onTapGesture {
                    withAnimation {
                        amount = Double.random(in: 10...90)
                    }
                }
        }
    }
}
```

对于 Trapezoid 的 insetAmount 属性来说，它会立刻被设置为新的值。但是 SwiftUI 随后会在绘制动画的时候逐渐修改这个值来实现动画效果，但这种渐变式的修改对于我们的代码是不可见的。

如果影响动画的参数超过了 1 个，可以使用 `AnimatablePair<Double, Double>` 作为 animatableData 的类型，具体可以参考 [这篇文章](https://www.hackingwithswift.com/books/ios-swiftui/animating-complex-shapes-with-animatablepair)



# Core Data

Core Data 是个类似于数据库的东西，相比于 UserDefaults 更加强大和灵活。

使用 CoreData 首先需要需要创建 entity, 类似于定义表结构：

![]( {{site.url}}/asset/swiftui-coredata.gif )

上图中第一步创建了一个后缀为 `xcdatamodeld` 的 DataModel 文件，存储的就是我们定义的结构，可以在其中新增 entity, 或者为 entity 增加属性。

创建完 DataModel 之后，需要将其加到程序中:

```swift
class DataController: ObservableObject {
    // 定义一个 container, 通过这个 container 来访问 DataModel
    let container = NSPersistentContainer(name: "Bookworm")
    
    init() {
        // 调用 loadPersistentStores 加载 DataModel
        container.loadPersistentStores { description, error in
            if let error = error {
                print("Core Data failed to load: \(error.localizedDescription)")
            }
        }
    }
}
```

为了方便使用，一般会将 NSPersistentContainer 保存到 Enviroment 当中:

```swift
@main
struct BookwormApp: App {
    // 创建 DataController 实例，将自动加载 DataModel
    @StateObject private var dataController = DataController()
    
    var body: some Scene {
        WindowGroup {
            ContentView()
                // 将 DataModel 加载到 managed object context 当中
                // 并不是直接加载了 container, 而是将 container.viewContext 设置到了 environment.managedObjectContext 上
                // viewContext 会让你现在内存中处理数据，直到有需要时才将内存中的数据持久化到磁盘
                .environment(\.managedObjectContext, dataController.container.viewContext)
        }
    }
}
```

接着就可以在需要的地方，通过 `@Enviroment` 包装器，获取到 DataModel 了:

```swift
struct ContentView: View {
    // 获取 Environment.managedObjectContext, 也就是之前传入的 context.viewContext
    @Environment(\.managedObjectContext) var moc

    // @FetchRequest 用于从数据库中获取数据
    @FetchRequest(sortDescriptors: []) var students: FetchedResults<Student>
    
    var body: some View {
        VStack {
            List {
                ForEach(students) { student in
                    Text(student.name ?? "Unknown")
                }
                .onDelete(perform: deleteStudents)
            }
            
            Button("Add") {
                let firstNames = ["Ginny", "Harry", "Hermione", "Luna", "Ron"]
                let lastNames = ["Granger", "Lovegood", "Potter", "Weasley"]
                
                let chosenFirstName = firstNames.randomElement()!
                let chosenLastName = lastNames.randomElement()!
                
                // 在 managedObjectContext 中创建一个 Student 对象，此时对象保存在内存里面
                let student = Student(context: moc)
                student.id = UUID()
                student.name = "\(chosenFirstName) \(chosenLastName)"
                
                // 把内存中的数据持久化到磁盘
                // hasChanges 能够检查 moc 内是否有对象发生了改变
                // student 这种 ManagedObject 也有个 hasChanges 属性，可以检查单个对象是否发生了改变
                if moc.hasChanges {
                    try? moc.save()
                }
            }
        }
    }

    func deleteStudents(at offsets: IndexSet) {
        for offset in offsets {
            let student = students[student]

            // 从 managedObjectContext 中删除对象
            moc.delete(student)
        }

        try? moc.save()
    }
}
```

注意上述代码中的 Student, 它是由 Core Data 根据 DataModel 自动生成的一个 class 类型，继承于 NSManagedObject，表示这种类型被 Core Data 所管理。


## SortDescriptor

`@FetchRequest()` 的 sortDescritors 参数用于排序:

```swift
// 按照 title 逆序，其次按照 author 正序
@FetchRequest(sortDescriptors: [
    SortDescriptor(\.title, order: .reverse),
    SortDescriptor(\.author)
]) var books: FetchedResults<Book>
```


## 手动生成 Entity 类

首先通过菜单栏打开 Data Model Inspector: View -> Inspectors -> Data Model。打开后会在 XCode 的右侧看到一个窗口。

接着选中创建好的 Entity，在 Data Model Inspector 里将 Codegen 选项切换为 `Manual/None`。这样 XCode 就不会为我们自动生成 Entity 类了。

接着需要手动创建出 Entity 类，这需要通过菜单栏完成: Editor -> "Create NSManagedObject Subclass"。点击后会出现一些对话框，按照对话框提示就可以创建出我们需要的 Entity 类。

通过菜单栏会创建出两个文件，分别为 `{Entity名}+CoreDataClass.swift`, `{Entity名}+CoreDataProperties.swift`。你可以在其中增加一些代码来方便 Core Data 的访问：

```swift
import Foundation
import CoreData


extension Movie {

    @nonobjc public class func fetchRequest() -> NSFetchRequest<Movie> {
        return NSFetchRequest<Movie>(entityName: "Movie")
    }

    @NSManaged public var title: String?
    @NSManaged public var director: String?
    @NSManaged public var year: Int16

    // 增加一个 computed property, 这样调用方就不需要处理 optional 了
    public var wrappedTitle: String {
        title ?? "Unknown Title"
    }
}

extension Movie : Identifiable {

}
```


## 为 attribute 添加约束

为 attribute 添加约束，有点像给数据库建唯一索引，添加约束之后，CoreData 会保证这个 attribute 在所有 entity 对象中唯一。

添加方法是：
1. 在 DataModel 中选中特定 Entity
2. 通过菜单栏 View -> Inspectors -> Data Model 打开 Data Model Inspector。
3. 在 Data Model Inspector 的 Constraints 栏点击 "+" 按钮，XCode 会生成一个默认的样例。
4. 在样例上按下 Enter 键，输入你要添加约束的 attribute
5. Cmd + S

添加完成后，CoreData 就会为这个属性设置唯一约束。这个位移约束需要在 `save()` 动作执行的时候才会被校验。

比如下面的代码，多次按下 Add 按钮后，由于还没有执行 `save()` 操作，约束还没生效，所以 wizards 会不断增加。当按下 Save 按钮，执行 `moc.save()` 操作的时候，会执行失败。

```swift
struct ContentView: View {
    @Environment(\.managedObjectContext) var moc
    @FetchRequest(sortDescriptors: []) var wizards: FetchedResults<Wizard>

    var body: some View {
        VStack {
            List(wizards, id: \.self) { wizard in
                Text(wizard.name ?? "Unknown")
            }
            
            Button("Add") {
                let wizard = Wizard(context: moc)
                wizard.name = "Harry Potter"
            }
            
            Button("Save") {
                do {
                    try moc.save()
                } catch {
                    // 多次 Add 将导致这里报错
                    // The operation couldn’t be completed. (Cocoa error 133021.)
                    print(error.localizedDescription)
                }
            }
        }
    }
}
```

如果希望 `save()` 时不要报错，而是将对象合并，可以在 DataController 中指定合并策略。在按下 Save 按钮之后，之前添加的多个 Wizard 对象会被合并为 1 个，然后保存到数据库里。

```swift
class DataController {
    static let shared = DataController()
    
    let container = NSPersistentContainer(name: "PhoneDemo")
    
    init() {
        container.loadPersistentStores { description, error in
            if let error = error {
                print("Core Data failed to load: \(error.localizedDescription)")
            }
            
            // 使用内存中的对象覆盖磁盘中的对象
            self.container.viewContext.mergePolicy = NSMergePolicy.mergeByPropertyObjectTrump
        }
    }
}

```

## NSPredicate 过滤查询结果

`@FetchRequest` 包装器支持一个 predicate 参数，通过这个参数可以过滤查询结果:

```swift
// 使用 NSPredicate 过滤查询结果
// NSPredicate(format: "universe == %@", "Star Wars")
// NSPredicate(format: "name < %@", "F"))
// NSPredicate(format: "universe IN %@", ["Aliens", "Firefly", "Star Trek"])
// NSPredicate(format: "name BEGINSWITH %@", "E"))
// NSPredicate(format: "name BEGINSWITH[c] %@", "e"))
// NSPredicate(format: "NOT name BEGINSWITH[c] %@", "e"))
@FetchRequest(sortDescriptors: [],
              predicate: NSPredicate(format: "universe == %@", "Star Wars")) var ships: FetchedResults<Ship>
```

以上代码中的 `%@` 是个占位符，他会自动给 value 加上单引号，比如 `NSPredicate(format: "universe == %@", "Star Wars")` 内的字符串会被替换为 `"universe == 'Star Wars'`。

还有一种占位符是 `%K`，它用在 key 上面，比如 `NSPredicate(format: "%K BEGINSWITH %@", "universe", "Star Wars")` 会被替换为 `"universe == 'Star Wars'`。

如果没有特殊指定，过滤过程是大小写敏感的，通过以上代码中提到的 `[c]` 可以进行忽略大小写的匹配。


## 动态改变 @FetchRequest 的过滤条件

思路是创建一个新的 View 切换不同的 NSPredicate:

```swift
struct FilteredList: View {
    @FetchRequest var results: FetchedResults<Singer>
    
    init(filter: String) {
        // 注意这里赋值给了 '_results' 而非 'results'
        // @FetchRequest 实际上会创建一个 _results 属性，这个属性需要被覆写掉
        _results = FetchRequest<Singer>(sortDescriptors: [],
                                        predicate: NSPredicate(format: "lastName BEGINSWITH %@", filter))
    }
    
    var body: some View {
        List(results, id: \.self) { singer in
            Text("\(singer.wrappedFirstName) \(singer.wrappedLastName)")
        }
    }
}

struct ContentView: View {
    @Environment(\.managedObjectContext) var moc
    
    @State private var lastNameFilter = "A"

    var body: some View {
        VStack {
            FilteredList(filter: lastNameFilter)

            Button("Add Examples") {
                let taylor = Singer(context: moc)
                taylor.firstName = "Taylor"
                taylor.lastName = "Swift"

                let ed = Singer(context: moc)
                ed.firstName = "Ed"
                ed.lastName = "Sheeran"

                let adele = Singer(context: moc)
                adele.firstName = "Adele"
                adele.lastName = "Adkins"

                try? moc.save()
            }

            Button("Show A") {
                lastNameFilter = "A"
            }

            Button("Show S") {
                lastNameFilter = "S"
            }
        }
    }
}
```

结合泛型，可以把上述 FilteredList 改造为可以接收任意 Entity 的列表:

```swift
struct FilteredList<T: NSManagedObject, Content: View>: View {
    @FetchRequest var results: FetchedResults<T>
    
    let content: (T) -> Content
    
    // content 参数的 @ViewBuilder 包装器，用于支持多个子 View
    // content 参数的 @escaping 表示 closure 将被保存起来以后再用，SwiftUI 对于这种 closure 的内存会特殊处理
    init(filterKey: String, filterValue: String, @ViewBuilder content: @escaping (T) -> Content) {
        let predicate = NSPredicate(format: "%K BEGINSWITH %@", filterKey, filterValue)
        _results = FetchRequest<T>(sortDescriptors: [], predicate: predicate)
        
        self.content = content
    }
    
    var body: some View {
        List(results, id: \.self) { result in
            self.content(result)
        }
    }
}

struct ContentView: View {
    @Environment(\.managedObjectContext) var moc
    
    @State private var lastNameFilter = "A"

    var body: some View {
        VStack {
            FilteredList(filterKey: "lastName", filterValue: lastNameFilter) { (singer: Singer) in
                Text("\(singer.wrappedFirstName) \(singer.wrappedLastName)")
            }

            // ...
        }
    }
}
```

## 通过 relationship 关联 entities

CoreData 支持称为 relationship 的功能，可以把 Entity 关联起来。

![]( {{site.url}}/asset/swiftui-core-data-relationships-country.png )

![]( {{site.url}}/asset/swiftui-core-data-relationships-candy.png )

以上两图展示了建立 Country 和 Candy 两者之间一对多关系的过程：

首先在 Country Entity 里新增了一条名为 candy 的 relationship, 其目标为 Candy，类型为 'To Many'。这会让 Core Data 帮我们生成如下代码：

```swift
extension Country {

    @nonobjc public class func fetchRequest() -> NSFetchRequest<Country> {
        return NSFetchRequest<Country>(entityName: "Country")
    }

    @NSManaged public var fullName: String?
    @NSManaged public var shortName: String?

    // 这个多出来的 candy 属性，就是我们新增的 relationship, 可以看到它是一个集合类型
    @NSManaged public var candy: NSSet?
    
    // 这个是手动增加的一个 computed property, 用于将 candy 转换成对 SwiftUI 友好的数组类型
    public var candyArray: [Candy] {
        let set = candy as? Set<Candy> ?? []
        return set.sorted {
            $0.wrappedName < $1.wrappedName
        }
    }
}

// CoreData 还为我们访问 candy 属性生成了一些方法
extension Country {

    @objc(addCandyObject:)
    @NSManaged public func addToCandy(_ value: Candy)

    @objc(removeCandyObject:)
    @NSManaged public func removeFromCandy(_ value: Candy)

    @objc(addCandy:)
    @NSManaged public func addToCandy(_ values: NSSet)

    @objc(removeCandy:)
    @NSManaged public func removeFromCandy(_ values: NSSet)

}

extension Country : Identifiable {

}
```

接着在 Candy Entity 里新增了一条名为 origin 的 relationship, 表示这种糖果来自哪个国家。其目标为 Country, 类型为 'To One'。这会让 Core Data 帮我们生成如下代码：

```swift
extension Candy {

    @nonobjc public class func fetchRequest() -> NSFetchRequest<Candy> {
        return NSFetchRequest<Candy>(entityName: "Candy")
    }

    @NSManaged public var name: String?
    @NSManaged public var origin: Country?

    public var wrappedName: String {
        name ?? "Unknown Candy"
    }
}

extension Candy : Identifiable {

}
```

值得注意的是，在添加 relationship 的时候，还有个 inverse 选项，它表明了两个 relationship 间的双向绑定关系。Country 对 Candy 是一对多，反之 Candy 对 Country 就是一对一，因此需要设置其 inverse 选项。


## 批量删除表中的数据

```swift
// 从 NSFetchRequest 创建出一个 NSBatchDeleteRequest
let deleteRequest = NSBatchDeleteRequest(fetchRequest: NSFetchRequest(entityName: "User"))
deleteRequest.resultType = .resultTypeObjectIDs

// 执行批量删除动作
let deleteResult = try moc.execute(deleteRequest) as? NSBatchDeleteResult

// 获取到删除的 id
if let deletedIds = deleteResult?.result as? [NSManagedObjectID] {
let deletedObjects = [NSDeletedObjectsKey: deletedIds]

// 将改变合并到 ManagedObjectContext
NSManagedObjectContext.mergeChanges(
    fromRemoteContextSave: deletedObjects, 
    into: [moc]
)
```

[这篇文章](https://www.advancedswift.com/batch-delete-everything-core-data-swift) 还提到了其他几种批量删除的场景。



# Core Image

Core Image 是 Apple 提供的一种内置框架，用于对图片施加各种效果，类似于 锐化、模糊、晕影、像素化 之类的。Core Image 还没有很好地集成进 SwiftUI，所以需要做一些桥接工作。

除了 SwiftUI 提供的 Image 类型，在 Apple 的开发框架里还存在着其他几种类型的图片类型：
- `UIImage`, 来自于 UIKit，功能很强大，它之于 UIKit 里就类似于 Image 之于 SwiftUI。
- `CGImage`, 来自于 Core Graphics 框架，是一种简单的图片类型，只是个包含像素数据的二维数组。
- `CIImage`, 来自于 Core Image, 这种类型里包含了能够产生图片的所有信息，但不是图片本身，更像是图片的一个 "菜谱"。

这几种类型之间的转换关系是：

![]( {{site.url}}/asset/swiftui-core-image-convert-original.svg )

下面的例子展示了从 UIImage 加载图片，经过 CIImage 处理之后，再转换为 Image 的过程。

```swift
import SwiftUI
// 注意需要导入 CoreImage 的包
import CoreImage
import CoreImage.CIFilterBuiltins

struct ContentView: View {
    @State private var image: Image?
    // CIContext 用于将 CIImage 转换为 CGImage, 也就是根据 "菜谱" 加工出图像
    // CIContext 的创建代价比较高，所以如果要转换多张图片的话，最好只创建一次
    let context = CIContext()
    
    var body: some View {
        VStack {
            image?
                .resizable()
                .scaledToFit()
        }
        .onAppear(perform: loadImage)
    }
    
    func loadImage() {
        // 使用 UIImage 加载图片
        guard let inputImage = UIImage(named: "Example") else {
            return
        }

        // UIImage -> CIImage
        let beginImage = CIImage(image: inputImage)
        
        // 使用 CIImage 为图片施加 "结晶化" 效果
        let currentFilter = CIFilter.crystallize()
        currentFilter.setValue(beginImage, forKey: kCIInputImageKey)
        currentFilter.radius = 20
        
        guard let outputImage = currentFilter.outputImage else {
            return
        }
        
        // CIImage -> CGImage
        if let cgImage = context.createCGImage(outputImage, from: outputImage.extent) {

            // CGImage -> UIImage
            let uiImage = UIImage(cgImage: cgImage)
            
            // UIImage -> Image
            image = Image(uiImage: uiImage)
        }
    }
}
```

以下是转换过程，注意由于直接把 CGImage 转换为 Image 需要写比较多参数，所以从 UIImage 绕了一圈:

![]( {{site.url}}/asset/swiftui-core-image-convert.svg ) )

以下是施加 "结晶化" 效果后的图片：

![]( {{site.url}}/asset/swiftui-core-image-crystallize.png )


## 生成二维码

Core Image 内置了一个 filter，可以将文本转换为二维码:

```swift
import CoreImage.CIFilterBuiltins

func generateQRCode(from string: String) -> UIImage {
    let data = Data(string.utf8)
    filter.setValue(data, forKey: "inputMessage")
    
    if let outputImage = filter.outputImage {
        if let cgImage = context.createCGImage(outputImage, from: outputImage.extent) {
            return UIImage(cgImage: cgImage)
        }
    }
    
    return UIImage(systemName: "xmark.circle") ?? UIImage()
}
```


# 无障碍支持

下面列出一些为界面元素提供无障碍支持的方法:

```swift
struct ContentView: View {
    let pictures = [
        "ales-krivec-15949",
        "galina-n-189483",
        "kevin-horstmann-141705",
        "nicolas-tissot-335096"
    ]
    let labels = [
        "Tulips",
        "Frozen tree buds",
        "Sunflowers",
        "Fireworks"
    ]
    let hints = [
        "Tulips in the forest",
        "Frozen tree buds",
        "Sunflowers",
        "Fireworks"
    ]
    @State private var selectedPicture = Int.random(in: 0...3)
    
    var body: some View {
        Image(pictures[selectedPicture])
            .resizable()
            .scaledToFit()
            .onTapGesture {
                self.selectedPicture = Int.random(in: 0...3)
            }
            // label 提供一个简短的描述
            .accessibility(label: Text(labels[selectedPicture]))
            // hint 提供更长的描述
            .accessibility(hint: Text(hints[selectedPicture]))
            // 默认情况下会将这个 View 视为一张图片，通过 addTraits 可以将其描述为是一个按钮
            .accessibility(addTraits: .isButton)
            .accessibility(removeTraits: .isImage)
    }
}
```

除了为界面元素提供友好的提示，另外一点是尽可能减少界面元素，因为视障人士在使用 app 时会在屏幕上滑动来查看听取 VoiceOver 提供的信息，太多界面元素会产生干扰。

```swift
// decorative 参数可以告知 SwiftUI，这个界面元素只是装饰性的，对于这个元素 VoiceOver 只会展示非常少量的信息
Image(decorative: "character")
    // accessibility(hidden:) 可以彻底将这个元素从无障碍系统中移除
    .accessibility(hidden: true)
```

另外一种技巧是把多个界面元素归到一个组里，比如下面的例子把两段文本合并了起来，这样会连贯地读出文本内容。

```swift
VStack {
    Text("Your score is")
    Text("1000")
        .font(.title)
}
.accessibilityElement(children: .ignore)
.accessibility(label: Text("Your score is 1000"))
```


### 不以颜色区分

iPhone 的 设置 -> 辅助功能 -> 显示与文字大小 中有个选项叫 ”不以颜色区分“。可以帮助区分颜色有困难的人，用其他标识来区分界面元素。

```swift
struct ContentView: View {
    // 通过环境变量可以获取到是否设置了该选项
    @Environment(\.accessibilityDifferentiateWithoutColor) var differentiateWithoutColor;
    
    var body: some View {
        HStack {
            if differentiateWithoutColor {
                Image(systemName: "checkmark.circle")
            }
            
            Text("Success")
        }
        .padding()
        .background(differentiateWithoutColor ? Color.black : Color.green)
        .foregroundColor(Color.white)
        .clipShape(Capsule())
    }
}
```

以下是该选项打开后的显示效果:

![]( {{site.url}}/asset/swiftui-accessibility-differentiate-without-color.png )


类似的，辅助功能里还有 "降低透明度" 的选项，也可以从 environment 里获取到:

```swift
struct ContentView: View {
    @Environment(\.accessibilityReduceTransparency) var reduceTransparency
    
    var body: some View {
        Text("Hello, World!")
            .padding()
            .background(reduceTransparency ? Color.black : Color.black.opacity(0.5))
            .foregroundColor(Color.white)
            .clipShape(Capsule())
    }
}
```


# 其他

## UserNotifications

UserNotifications 框架用于向用户推送通知。有两种通知方式，一种是 remote notification, 顾名思义你需要向苹果的服务器 (Apple's push notification service, APNS) 推送消息，然后转发给用户；另一种是 local notification, 它是由 App 按计划发送的通知，不需要服务端的参与。

下面演示 local notification 的用法:

```swift
import SwiftUI
// 需要导入这个包
import UserNotifications

struct ContentView: View {
    
    var body: some View {
        VStack {
            Button("Request Permission") {
                // 请求通知权限
                UNUserNotificationCenter.current().requestAuthorization(options: [.alert, .badge, .sound]) { success, error in
                    if success {
                        print("All set!")
                    } else if let error = error {
                        print(error.localizedDescription)
                    }
                }
            }
            .padding()
            
            Button("Schedule Notification") {
                // 添加通知发送计划
                
                // 设置内容
                let content = UNMutableNotificationContent()
                content.title = "Feed the cat"
                content.subtitle = "It looks hungry"
                content.sound = UNNotificationSound.default
                
                // 设置触发事件(5 秒后)
                let trigger = UNTimeIntervalNotificationTrigger(timeInterval: 5, repeats: false)
                
                // 组装一个请求出来，注意需要设置一个 identifier
                let request = UNNotificationRequest(identifier: UUID().uuidString, content: content, trigger: trigger)
                
                // 把请求放提交到 NotificationCenter
                UNUserNotificationCenter.current().add(request)
            }
            .padding()
        }
    }
}
```


## 在 Xcode 里添加包依赖

通过菜单栏 File -> Add Package... 进入以下界面，然后在右上角的搜索框输入你要添加的包的地址。随后就可以添加了：

![]( {{site.url}}/asset/swiftui-add-package.png )


## 触觉反馈

UIKit 中提供了简单的方法来触发触觉反馈:

```swift
func simpleSuccess() {
    // 生成一个表示执行成功的触觉反馈
    let generator = UINotificationFeedbackGenerator()
    generator.notificationOccurred(.success)
}
```

此外，Apple 提供了 CoreHaptics 框架来支持更复杂的触觉反馈功能:

```swift
import SwiftUI
import CoreHaptics


struct ContentView: View {
    @State private var engine: CHHapticEngine?
    
    var body: some View {
        Text("Hello, World!")
            .onAppear(perform: prepareHaptics)
            .onTapGesture(perform: complexSuccess)
    }
    
    func prepareHaptics() {
        // 如果设备支持触觉反馈，则初始化 CHHapticEngine
        guard CHHapticEngine.capabilitiesForHardware().supportsHaptics else { return }

        do {
            self.engine = try CHHapticEngine()
            try engine?.start()
        } catch {
            print("There was an error creating the engine: \(error.localizedDescription)")
        }
    }
    
    func complexSuccess() {
        guard CHHapticEngine.capabilitiesForHardware().supportsHaptics else { return }
        var events = [CHHapticEvent]()

        // 创建一个强烈而尖锐的敲击事件
        let intensity = CHHapticEventParameter(parameterID: .hapticIntensity, value: 1)
        let sharpness = CHHapticEventParameter(parameterID: .hapticSharpness, value: 1)
        let event = CHHapticEvent(eventType: .hapticTransient, parameters: [intensity, sharpness], relativeTime: 0)
        events.append(event)
        
        // 敲击强度从小到大，再从大到小
//        for i in stride(from: 0, to: 1, by: 0.1) {
//            let intensity = CHHapticEventParameter(parameterID: .hapticIntensity, value: Float(i))
//            let sharpness = CHHapticEventParameter(parameterID: .hapticSharpness, value: Float(i))
//            let event = CHHapticEvent(eventType: .hapticTransient, parameters: [intensity, sharpness], relativeTime: i)
//            events.append(event)
//        }
//
//        for i in stride(from: 0, to: 1, by: 0.1) {
//            let intensity = CHHapticEventParameter(parameterID: .hapticIntensity, value: Float(1 - i))
//            let sharpness = CHHapticEventParameter(parameterID: .hapticSharpness, value: Float(1 - i))
//            let event = CHHapticEvent(eventType: .hapticTransient, parameters: [intensity, sharpness], relativeTime: 1 + i)
//            events.append(event)
//        }

        // 把敲击事件转换为 CHHapticPattern, 随后播放这个 pattern
        do {
            let pattern = try CHHapticPattern(events: events, parameters: [])
            let player = try engine?.makePlayer(with: pattern)
            try player?.start(atTime: 0)
        } catch {
            print("Failed to play pattern: \(error.localizedDescription).")
        }
    }
}
```


## .allowsHitTesting() modifier

SwiftUI 的 hit test 算法不仅作用于 View 的框架，也作用于它的内容，比如下面的例子，虽然矩形和圆形的边界一样，但是点击圆形外的矩形是可以命中，SwiftUI 忽略了圆形外部的一些透明区域。

```swift
struct ContentView: View {
    var body: some View {
        ZStack {
            Rectangle()
                .fill(Color.blue)
                .frame(width: 300, height: 300)
                .onTapGesture {
                    print("Rectangle tapped!")
                }
            
            Circle()
                .fill(Color.red)
                .frame(width: 300, height: 300)
                .onTapGesture {
                    print("Circle tapped!")
                }
        }
    }
}
```

`.allowsHitTesting()` modifier 用于表示 hit test 是否作用于这个 View，如果设置为 false，那么 hit test 就不会命中这个 View，比如修改上面的代码，使用 `.allowsHitTesting(false)` 来包装圆形，那么圆形将不会参与到 hit test 当中，即使你点击了圆形内部，hit test 也会直接忽略这个圆形，然后命中矩形。

另外还有一种情况是希望空白区域也能被 hit test 命中，比如下面 VSTack 中两个 Test 之间的 Spacer 默认是不会被命中的，但你可以使用 `.contentShape()` 来修改命中区域，这样 Spacer 也能被命中到:

```swift
VStack {
    Text("Hello")
    Spacer().frame(height: 100)
    Text("World")
}
.contentShape(Rectangle())
.onTapGesture {
    print("VStack tapped!")
}
```


## Timer

Timer 是 Swift Combine 框架里提供的东西。其实之前提到的 `ObservableObject`, `@Publisher` 也都是 Combine 框架里的东西。以下代码演示了 Timer 的发布、监听、取消过程:

```swift
struct ContentView: View {
    // Timer.publish() 方法将创建一个 Timer.Publisher
    // - 第一个参数表示每秒发送一次事件
    // - 第二个参数表示将事件抛到主线程
    // - 第三个参数表示应该把事件抛到 common run loop 里面
    // 使用 .autoconnect() 方法来立刻触发 Timer.Publisher 工作
    // - autoconnect() 返回的东西是个 auto connected publisher, 跟之前用 @Published 修饰的属性是类似的
    // - autoconnect() 返回的 Publisher 可以被 View 的 .onReceive() modifier 接收
    let timer = Timer.publish(every: 1, on: .main, in: .common).autoconnect()
    @State private var counter = 0
    
    var body: some View {
        Text("Hello, World!")
            .onReceive(timer) { time in
                if self.counter == 5 {
                    // 计时达到 5 次，关闭 timer
                    self.timer.upstream.connect().cancel()
                } else {
                    // time 是当前时间
                    print("The time is now \(time)")
                }
                
                self.counter += 1
            }
    }
}
```

另外 Timer 的 `publish()` 方法还有个 `tolerance` 参数，是 "公差" 的意思，在对精度要求不高的情况下设置这个参数，可以让 SwiftUI 把多个 Timer 的事件合并起来一起发送，节省系统资源。

```swift
let timer = Timer.publish(every: 1, tolerance: 0.5, on: .main, in: .common).autoconnect()
```


## 检测 App 切换

Apple 有个叫做 Notification Center 的框架，会基于 Combine 来向 App 发布各种系统相关的事件，下面展示了一些例子:

```swift
struct ContentView: View {
    var body: some View {
        Text("Hello, World!")
            // App 即将被切走
            .onReceive(NotificationCenter.default.publisher(
                for: UIApplication.willResignActiveNotification)) { _ in
                print("Moving to the background!")
            }
            // App 即将被切回来
            .onReceive(NotificationCenter.default.publisher(
                for: UIApplication.willEnterForegroundNotification)) { _ in
                print("Moving back to the foreground!")
            }
            // 用户在 App 界面进行了截图
            .onReceive(NotificationCenter.default.publisher(
                for: UIApplication.userDidTakeScreenshotNotification)) { _ in
                print("User took a screenshot!")
            }
    }
}
```


## 强制横屏显示

![]( {{site.url}}/asset/swiftui-device-orientation.png )

有时候上述设置会不生效，需要按照如下步骤删除掉不需要的 orientation:

![]( {{site.url}}/asset/swiftui-device-orientation-bug.png )