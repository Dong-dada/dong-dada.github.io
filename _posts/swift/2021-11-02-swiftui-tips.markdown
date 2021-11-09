---
layout: post
title:  "SwiftUI - 小知识点"
date:   2021-11-02 20:03:59 +0800
categories: swift
---

* TOC
{:toc}


# @State 属性包装器

View 都是 struct 类型，因为 SwiftUI 框架会经常性地销毁和重建 View，因此需要其尽量简单，搞成 class 的话可能会有初始化之类的特殊逻辑，导致 SwiftUI 重新创建它的成本变高。

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



# 双向绑定

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


# Modifier 的顺序

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


# Environment Modifier

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