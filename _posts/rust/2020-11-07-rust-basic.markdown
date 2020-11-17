---
layout: post
title:  "Rust - 基础知识"
date:   2020-10-27 14:50:30 +0800
categories: rust
---

* TOC
{:toc}


# Cargo

Cargo 是个构建系统，能够帮我们管理 rust 的依赖。

```bash
# 创建新项目
$cargo new hello_cargo

$cd hello_cargo

# 编译项目
$cargo build

# 编译并运行项目
$cargo run

# 编译，但不生成可执行文件，可以快速检查代码有没有编译错误
$cargo check
```

Cargo 项目的目录结构:

```
.
├── Cargo.toml         // 项目配置文件
├── Cargo.lock
├── src                // 代码目录
│   └── main.rs
└── target             // 编译结果保存在这里
```

一个简单的 Cargo.toml 文件

```toml
[package]
name = "rust-basic"
version = "0.1.0"
authors = ["dongyu"]
edition = "2018"

[dependencies]
rand = "0.5.5"         # 像这样就可以添加依赖，只需要 crate 名称以及版本号
```


# 打印

```rust
let x = 5;
let y = "hello";
println!("x = {}, y = {}", x, y);   // x = 5, y = hello
```


# 变量

变量默认不可变，加上 mut 修饰符，才能变：

```rust
let foo = 10;
foo = 20;        // error

let mut bar = 10;
bar = 20;        // Ok
```

在同一作用域可以多次定义同名变量，并未变量指定不同的类型，这被称为 shadowing.

```rust
let mut guess = String::new();
io::stdin().read_line(&mut guess).expect("Failed to read line");

// 再次定义同名变量，注意这里改变了类型和可变性
let guess: u32 = guess.trim().parse().expect("Please type a number!");
```

变量包含以下几种基本类型:
- 整形，包括 i8, u8, i16, u16, i32, u32, i64, u64, i128, u128, isize, usize 这几种
- 浮点型，包括 f32, f64 这两种
- 布尔型，也就是 bool
- 字符型，也就是 char, 注意 rust 的 char 类型有 4 字节这么长，它代表的是一个 Unicode 值，可以包含中文、emoji 等符号。

变量还包含一下几种复合类型:
- tuple, 类似一个定长数组，但是每个元素的类型可以不同。
- array, 定长数组，每个元素的类型都相同

```rust
// tuple 初始化
let tup: (i32, f64, u8) = (500, 6.4, 1);

// 获取 tup 中的单个元素
let (x, y, z) = tup;
println!("The value of y is {}", y);

// 也可以用下表来获取
let x = tup.0
let y = tup.1
let z = tup.2

// tuple 作为函数参数
fn area(dimensions: (u32, u32)) -> u32 {
    dimensions.0 * dimensions.1
}


// array 初始化
let a = [1, 2, 3, 4, 5];
let month = ["January", "February", "March", "April", "May", "June", "July",
              "August", "September", "October", "November", "December"];

// 另一种初始化方式，i32 指定元素类型，5 指定数组长度
let a: [i32; 5] = [1, 2, 3, 4, 5];

// 另一种初始化方式，数组长度为 5，每个元素的值都是 3
let a = [3; 5];

// 使用下标获取数组元素
let first = a[0];
let second = a[1];
```


# 常量

主要用处是避免魔数之类的，比如用常量来定义错误码，或者一些 limit 之类的东西。

```rust
const MAX_POINTS: u32 = 100000;
```


# if 语句

```rust
fn main() {
    let number = 3;

    // rust 中 if 语句的条件必须是 bool 值，rust 不会做隐式转换
    if number < 5 {
        println!("too small!");
    } else if number > 10 {
        println!("too big!");
    } else {
        println!("correct.");
    }
}
```

if 可以配合 let 语句使用：

```rust
fn main() {
    let condition = true;

    // 注意这里 if else 里都应该是一个表达式，且表达式的返回值类型应该一致
    let number = if condition { 5 } else { 6 };

    println!("The value of number is: {}", number);
}
```


# 循环

```rust
// 无限循环
loop {
    if xxx {
        continue;
    }

    if xxx {
        break;
    }

    xxx
}

// while 循环
let a = [10, 20, 30, 40, 50];
let mut index = 0;

while index < 5 {
    println!("the value is: {}", a[index]);

    index += 1;
}

// for in 循环，可以用来遍历数组
let a = [1, 2, 3, 4, 5];
for element in a.iter() {
    println!("the value is: {}", element);
}

// for in range, 可以用来对一个区间进行遍历
for number in (1..4).rev() {
    println!("{}!", number);
}
```

循环里的 break 可以返回值，随后跟 let 一起配合使用：

```rust
fn main() {
    let mut counter = 0;

    let result = loop {
        counter += 1;

        if counter == 10 {
            break counter * 2;
        }
    };
}
```


# 函数

```rust
fn add(x: i32, y: i32) -> i32 {
    return x + y;
}

fn sub(x: i32, y: i32) -> i32 {
    // rust 是基于表达式的语言，如果不加分号，那么就是一个表达式
    // 函数的最后一个表达式的执行结果，就是这个函数的返回值，所以这里可以不写 return
    x - y
}
```


# Ownership

所有权规则:
- Rust 中的所有值都有一个 owner。
- 一个值同一时刻仅属于一个 owner。
- 如果 owner 离开了作用域，这个值将被销毁。

Rust 语言所提供的几种基本类型(包括复合类型)，都是在栈上分配内存的。标准库中定义了一些类型，这些类型是从堆上分配内存的，比如 String。

```rust
{
    // 在堆上分配内存
    let mut s = String:from("hello");
    s.push_str(", world!");
    println!("{}", s);
}   // 变量 s 离开了作用域，为其分配的内存将被释放
```

从实现上讲，变量 s 离开作用域的时候，其 drop() 方法将被自动调用，来销毁为其分配的内存。类似于 C++ 的 RAII 模式。

看起来很简单，不过考虑到其他情况，事情就会变得有点复杂。

## 赋值时发生的移动

比如以下代码:

```rust
fn main() {
    let s1 = String::from("hello");
    let s2 = s1;
    println!("{}, world!", s1);
}
```

变量 s1 被赋值给了 s2，当 s1 和 s2 都离开作用域的话会发生什么呢？ drop() 方法会被调用两次吗？

其实不会，rust 在赋值的时候会检查变量是否实现了 Copy trait：如果实现了 Copy trait，那么变量应该做一次深拷贝，这样新旧变量就都可以用；如果没有实现 Copy trait，那么 s1 对应的内存会被 move 到 s2 上，然后 s1 变成了无效状态。

因此上面的代码无法编译通过，会报以下错误：

```
error[E0382]: borrow of moved value: `s1`
 --> src\main.rs:5:28
  |
3 |     let s1 = String::from("hello");
  |         -- move occurs because `s1` has type `std::string::String`, which does not implement the `Copy` trait
4 |     let s2 = s1;
  |              -- value moved here
5 |     println!("{}, world!", s1);
  |                            ^^ value borrowed here after move
```

如果你希望进行深拷贝，那么应该调用 clone() 方法：

```rust
fn main() {
    let s1 = String::from("hello");
    let s2 = s1.clone();
    println!("s1 = {}, s2 = {}", s1, s2);
}
```

## 函数参数和返回值

向函数传参时发生的情况跟赋值类似，要么 move, 要么 copy：

```rust
fn main() {
    let s = String::from("hello");
    takes_ownership(s);     // 因为 String 没有实现 Copy trait, 因此 s 被 move 到函数内

    let x = 5;
    makes_copy(x);          // 因为 i32 实现了 Copy trait，因此 x 做了一次拷贝
}

fn takes_ownership(some_string: String) {
    println!("{}", some_string);
}   // 变量 some_string 离开了作用域，因此被释放

fn makes_copy(some_integer: i32) {
    println!("{}", some_integer);
}   // 变量 some_integer 离开了作用域，因此被释放
```

函数返回值的情况也赋值类似，要么 move，要么 copy:

```rust
fn main() {
    let s1 = give_ownership();

    let s2 = String::from("hello");

    let s3 = takes_and_gives_back(s2);
}

fn give_ownership() -> String {
    let some_string = String::from("hello");
    some_string
}   // 因为 String 没有实现 Copy trait, 因此 some_string 被 move 给了调用方

fn takes_and_gives_back(a_string: String) -> String {
    a_string
}   // 变量先被 move 给该方法，然后又被 move 给调用方
```

## 引用

有的时候你有这种需求：既希望把变量传给函数使用，又不希望函数把这个变量的所有权夺走。

按照上述规则，你只能在返回值里面重新得到这个变量的所有权：

```rust
fn main() {
    let s1 = String::from("hello");

    let (s2, len) = calculate_length(s1);

    println!("The length of '{}' is {}.", s2, len);
}

fn calculate_length(s: String) -> (String, usize) {
    let length = s.len(); // len() returns the length of a String

    (s, length)
}   // 再把所有权返回给调用方
```

这种写法比较繁琐，Rust 提供了名为引用的概念，可以达到你想要的效果：

```rust
fn main() {
    let s1 = String::from("hello");

    // &s1 用这种方式就可以获取到变量的引用
    let s2 = &s1;

    // &s1 用这种方式获取到变量的引用，随后把引用作为参数传给函数
    let len = calculate_length(&s1);
    println!("The length of '{}' is {}.", s1, len);

    // 如果希望通过引用修改原对象，那么需要用 mut 修饰，能够这么做的前提是源对象是 mutable 的
    let mut s1 = s1;
    let s2 = &mut s1;
    change(&mut s1);
    println!("The length of '{}' is {}.", s1, s1.len());
}

// 通过引用的方式，可以达到 "借用" 所有权的效果
fn calculate_length(s: &String) -> usize {
    s.len()
}

fn change(s: &mut String) {
    s.push_str(", world");
}
```

引用的实现跟 C++ 类似，也是通过指针来实现的：

![]({{site.url}}/asset/rust-reference.svg)

因此上述代码中的变量 s 退出作用域，只是销毁了指针，不会销毁字符串对象。

不过 rust 对于引用的使用有一些限制，以避免 data race 问题：
- 两个以上的线程同时访问同一块内存；
- 其中一个或多个线程对这块内存做修改；
- 某个线程没有做同步机制；

总的来说就是有一个线程修改了内存，其它线程不知道，就会读到还没有被写完的数据，或者同时写入，导致内存里的数据错乱。

所以 rust 要求，在引用的作用范围内：
- 最多只能有一个可变引用(避免多个可变引用修改同一块内存)。
- 如果有了可变引用，那就不能有其它不可变引用(避免一个可变引用修改内存时，其它不可变引用读取到的数据不完整)。
- 不可变引用可以有多个(只读的话，引用再多也没关系)。

实验发现 rust 似乎会跟踪引用的作用区间，检测到两个引用的作用区间重复的话就会报错。比如以下两例代码：

```rust
// 以下代码能够编译通过
fn main() {
    let mut s1 = String::from("hello");     // s1 作用区间开始
    let s2 = &mut s1;                       // s2 作用区间开始
    s2.push_str(", world");                 // s2 作用区间结束
    s1.push_str("!");                       // s1 作用区间结束
}
```

```rust
// 以下代码编译失败
fn main() {
    let mut s1 = String::from("hello");     // s1 作用区间开始
    let s2 = &mut s1;                       // s2 作用区间开始
    s1.push_str("!");                       // s1 作用区间结束
    s2.push_str(", world");                 // s2 作用区间结束
}
```

第二例代码的编译提示如下，可以看到因为 s1 和 s2 的 作用区间是重合的，所以编译失败了：

```
 --> src/main.rs:4:5
  |
3 |     let s2 = &mut s1;
  |              ------- first mutable borrow occurs here
4 |     s1.push_str("!");
  |     ^^ second mutable borrow occurs here
5 |     s2.push_str(", world");
  |     -- first borrow later used here
```

另外一种情况是，如果有了可变引用，就不能再有其它的不可变引用，比如下面的例子：

```rust
// 以下代码编译失败
fn main() {
    let mut s = String::from("hello");

    let r1 = &s;            // r1 作用区间开始
    let r2 = &mut s;        // r2 作用区间开始

    // 已经有了可变引用 r2 了，这个时候就不能再使用不可变引用 r1 了
    println!("{}, {}", r1, r2);     // r1,r2 作用区间结束
}
```

此外，rust 能够检查出来引用作为返回值时，是否会导致空悬指针(dangling pointer)问题：

```rust
fn dangle() -> &String {
    let mut s = String::from("hello");
    &s
}   // s 离开作用域之后会销毁，因此返回 &s 会导致空悬指针问题
```

以上代码报错如下：

```
error[E0106]: missing lifetime specifier
 --> src/main.rs:6:16
  |
6 | fn dangle() -> &String {
  |                ^ expected named lifetime parameter
  |
  = help: this function's return type contains a borrowed value, but there is no value for it to be borrowed from
help: consider using the `'static` lifetime
  |
6 | fn dangle() -> &'static String {
  |
```

以下代码就能工作正常，因为改代码没有空悬指针问题：

```rust
fn main() {
    let s = String::from("Hello world!");
    let r = safe(&s);
    println!("{}", r);
}

fn safe(s: &String) -> &String {
    &s
}
```


# Slice 类型

与 Reference 类似，Slice 这种数据类型也没有所有权。Slice 的作用是让你能够引用一个集合的子集：

```rust
fn main() {
    let s = String::from("hello world");

    let hello = &s[0..5];       // 获取 s 的 [0~5) 子集。这里的 0 可以省略不写: [..5]
    let world = &s[6..11];      // 获取 s 的 [6~11) 子集。这里的 11 可以省略不写: [6..]

    println!("{} {}", hello, world);
}
```

以下是 slice type 的内存结构：

![]({{site.url}}/asset/rust-slice.svg)

可以看到 slice 实际上是指针加上长度。

Slice 类型的定义为 &str，以下是一个把 slice 作为返回值的例子：

```rust
// 这段代码没问题
fn first_word(s: &String) -> &str {
    let bytes = s.as_bytes();

    for (i, &item) in bytes.iter().enumerate() {
        if item == b' ' {
            return &s[0..i];
        }
    }

    &s[..]
}

// 这段代码会编译失败
fn main() {
    let mut s = String::from("hello world");
    let word = first_word(&s);
    s.clear();
    println!("{}", word);   // s 已经被释放了，仍然尝试访问 word slice，因此编译出错
}
```

上面编译错误的原因跟引用是类似的，也是作用区间的问题。

一种更好的编码实践是使用 &str 而不是 &String 来作为函数的参数，这样函数既可以接收 slice 参数，也可以接收 String 参数：

```rust
fn first_word(s: &str) -> &str {
    let bytes = s.as_bytes();

    for (i, &item) in bytes.iter().enumerate() {
        if item == b' ' {
            return &s[0..i];
        }
    }

    &s[..]
}

fn main() {
    let s = String::from("hello world");
    let word = first_word(&s[..]);          // 通过 &s[..] 把 String 转为 slice
    println!("{}", word);

    let word = first_word("hello world");   // 直接传入 slice
    println!("{}", word);
}
```

Slice 也可以用在数组之类的地方：

```rust
let a = [1, 2, 3, 4, 5];
let slice = &a[1..3];
```


# 结构体

```rust
// 结构体定义
#[derive(Debug)]
struct User {
    username: String,
    email: String,
    sign_in_count: u64,
    active: bool,
}

fn build_user(email: &str, username: &str) -> User {
    // 创建实例
    User {
        email: String::from(email),
        username: String::from(username),
        active: true,
        sign_in_count: 1,
    }
}

fn main() {
    // rust 不允许将用 mut 修饰特定 field, 只能用 mut 修饰整个结构体
    let mut user = build_user("someone@example.com", "someone");

    // 访问结构体成员
    user.email = String::from("anotheremail@example.com");

    // 打印结构体信息，注意结构体必须添加 #[derive(Debug)] 注解，才能打印出来
    println!("{:#?}", user);

    // 根据已有实例，创建一个新的实例
    let user2 = User {
        email: String::from("another@example.com"),
        username: String::from("another"),
        ..user
    };
    println!("{:#?}", user2);

    // tuple struct, 类似于 tuple, 但是可以起名字
    struct Color(i32, i32, i32);
    struct Point(i32, i32, i32);

    let black = Color(0, 0, 0);
    let origin = Point(0, 0, 0);
}
```

对于上述代码中声明的 User 结构体而言，其成员的生命周期与 User 自身的生命周期是一致的。如果希望让结构体保存一个引用，需要用到 rust 的 lifetime 机制。下述代码会编译错误：

```rust
struct User {
    username: &str,     // 需要使用 lifetime 机制，不能直接使用引用
    email: &str,
    sign_in_count: u64,
    active: bool
}
```

以下例子展示了如何为结构体定义成员函数：

```rust
#[derive(Debug)]
struct Rectangle {
    width: u32,
    height: u32,
}

// 为结构体添加 method
impl Rectangle {
    // 使用 &self, 以只读方式访问 Rectangle 实例
    fn area(&self) -> u32 {
        self.width * self.height
    }

    // 使用 &mut self, 以读写方式刚问 Rectangle 实例
    fn scale(&mut self, radio: f32) {
        self.width = (self.width as f32 * radio) as u32;
        self.height = (self.height as f32 * radio) as u32;
    }
}

// 注意同一个结构体，可以定义多个 impl block.
impl Rectanble {
    // 定义 associated function, 类似于静态成员函数
    fn square(size: u32) -> Rectangle {
        Rectangle {
            width: size,
            height: size,
        }
    }
}

fn main() {
    let rect = Rectangle {
        width: 30,
        height: 50,
    };

    // 调用成员函数
    println!("The area of the rectangle is {} square pixels.", rect.area());

    // 调用成员函数修改结构体成员
    let mut rect = rect;
    rect.scale(0.5);
    println!("{:#?}", rect);

    // 调用 associated function
    let rect = Rectangle::square(3);
    println!("{:#?}", rect);
}
```



# 枚举

简单使用:

```rust
// 定义枚举
#[derive(Debug)]
enum IpAddrKind {
    V4,
    V6,
}

#[derive(Debug)]
struct IpAddr {
    // 枚举作为结构体成员
    kind: IpAddrKind,
    address: String,
}

fn main() {
    let home = IpAddr {
        // 使用枚举
        kind: IpAddrKind::V4,
        address: String::from("127.0.0.1"),
    };
    let loopback = IpAddr {
        kind: IpAddrKind::V6,
        address: String::from("::1"),
    };
    println!("home: {:#?}", home);
    println!("loopback: {:#?}", loopback);
}
```

与 C++/Java 等语言不同，Rust 的枚举值可以被赋值：

```rust
#[derive(Debug)]
enum IpAddr {
    V4(u8, u8, u8, u8),
    V6(String),
}

fn main() {
    // 把 127, 0, 0, 1 赋值给 V4
    let home = IpAddr::V4(127, 0, 0, 1);
    println!("home: {:#?}", home);

    // 把 ::1 赋值给 V6
    let loopback = IpAddr::V6(String::from("::1"));
    println!("loopback: {:#?}", loopback);
}
```

看起来 rust 里面的枚举与 C++/Java 里的枚举很不一样。不过这种语法让 Optional 等类型实现起来更自然。


# Option Enum

Rust 语言中没有 null 的概念，但标准库提供了一个 Option 枚举：

```rust
enum Option<T> {
    Some(T),
    None,
}
```

以下是使用 Option 的一些简单例子：

```rust
fn main() {
    let some_number = Some(5);              // some_number 类型为 Option<i32>
    let some_string = Some("a string");     // some_string 类型为 Option<&str>
    let absent_number: Option<i32> = None;  // 如果要初始化为 None, 那么必须显式指明变量类型
    println!("{:?}, {:?}, {:?}", some_number, some_string, absent_number);

    let x: i8 = 5;
    let y: Option<i8> = Some(5);
    // let z = x + y;                       // 编译错误，Option 和 i8 是不同的类型，不能直接相加
    let z = x + y.unwrap_or(0);             // 使用 unwrap 获取实际的值

    // 使用 is_some, is_none 方法检查 Option 是否有值
    println!("y is some? {}", y.is_some());
    println!("y is none? {}", y.is_none());
}
```


# match 操作符

match 操作符用来检查传入的值符合哪种模式，随后执行相应模式的代码，看起来跟 C++/Java 的 switch 差不多。

```rust
#[derive(Debug)]
enum Coin {
    Penny,
    Nickel,
    Dime,
    Quarter,
}

fn value_in_cents(coin: Coin) -> u8 {
    match coin {
        Coin::Penny => {
            println!("Lucky penny!");
            1
        },
        Coin::Nickel => 5,
        Coin::Dime => 19,
        Coin::Quarter => 25,
    }
}

fn main() {
    println!("The value of {:?} is {}.", Coin::Quarter, value_in_cents(Coin::Quarter));
}
```

除此之外 match 操作符还可以用来获取 enum 中保存的值：

```rust
#[derive(Debug)]
enum UsState {
    Alabama,
    Alaska,
    // --snip--
}

#[derive(Debug)]
enum Coin {
    Penny,
    Nickel,
    Dime,
    Quarter(UsState),
}

fn main() {
    let coin = Coin::Quarter(UsState::Alaska);
    match coin {
        Coin::Quarter(state) => {
            // 使用 match 可以获取到 enum 中保存的值
            println!("State quarter from {:?}!", state);
        },

        // 使用 _ 占位符，可以添加默认行为，类似于 C++/Java 中的 default
        _ => {}
    };
}
```

这种技巧也可以用在 Option 上：

```rust
fn main() {
    let five = Some(5);
    let six = match five {
        None => None,
        Some(i) => Some(i + 1),     // 获取到 Option 内的值，加一之后生成一个新的 Option
    };
    println!("{:?}, {:?}", five, six);
}
```


# if let 操作符

看起来 if let 是一种语法糖，可以比较方便地获取 enum 里面的值。

```rust
fn main() {
    // match 写法
    let some_u8_value = Some(0u8);
    match some_u8_value {
        Some(3) => println!("three"),
        _ => ()
    }

    // if let 写法
    if let Some(3) = some_u8_value {
        println!("three");
    }

    let mut count = 0;
    let coin = Coin::Quarter(UsState::Alaska);
    // match 写法
    match coin {
        Coin::Quarter(ref state) => println!("State quarter from {:?}!", state),
        _ => count += 1,
    }

    // if let 写法
    if let Coin::Quarter(ref state) = coin {
        println!("State quarter from {:?}!", state);
    } else {
        count += 1;
    }
}
```


# module system

- crate 是一个可执行文件或者一个库。
    - crate root 是一个源文件，rust 编译器从这个源文件开始编译。
    - crate root 构成了 crate 的 root module.
- package 是一个或多个 crate 构成的包。
    - package 包含类一个 Cargo.toml 文件，这个文件描述了如何编译 package 所包含的 crate。
    - package 包含 crate 的规则为：
        - 只能包含 0 或 1 个 library crates
        - 可以包含任意多个二进制箱 binary crates
        - 至少有一个 crate, 可以是 library crates, 也可以是 binary crates
    - src/main.rs 是 binary crate 的 crate root
    - src/lib.rs 是 library crate 的 crate root, 该 crate 与 package 同名
    - 如果需要多个 binary crate, 在 src/bin 目录下创建 .rs 文件，每个文件都对应一个 binary crate.
- package 对 crate 规则的限制看起来很奇怪。为什么 package 下最多只能有一个 library crate，但是可以有多个 binary crate 呢？
    - 可以参考 [这篇文章](https://stackoverflow.com/questions/54843118/why-can-a-cargo-package-only-have-one-library-target)
    - 总的来说，cargo 是一个包管理工具，对于一个 package, 当然希望是这个 package 里只有一个 library，不然的话引用一个库还需要 package.library 这样分两层，搞的就很复杂了。
    - 而对于 binary crate 来说，反正它不是包管理的一部分(不会被其他人依赖)，所以有几个都没有关系。

```s
# 创建 binary crate
$cargo new my-project
     Created binary (application) `my-project` package
$tree my-project
my-project
├── Cargo.toml
└── src
    └── main.rs

# 创建 library crate
$cargo new --lib my-lib
     Created library `my-lib` package
$tree my-lib
my-lib
├── Cargo.toml
└── src
    └── lib.rs
```

现在我们知道 package 内包含一个或多个 crate, 在 Cargo.toml 文件中定义了依赖。Cargo 会帮助我们下载依赖，构建当前的 package.那么 crate 内部的代码怎么组织呢？rust 怎么从 crate root 找到其他文件中的代码呢？这就需要 mod 出马了。

Rust 的 module system 与其它语言不太一样，它需要你手动来构建 module tree. 比如你代码的文件树如下：

```txt
.
├── Cargo.toml
└── src
    ├── main.rs
    └── order.rs
```

在 order.rs 当中定义了一些结构体、枚举，你想要在 main.rs 当中访问这些代码：

```rust
// order.rs 文件内容

use chrono::Local;

enum OrderState {
    Init,
    PendingPayment,
    PendingDeliver,
    Finish,
}

#[derive(Debug)]
struct Order {
    id: i64,
    order_state: OrderState,
    create_timestamp_ms: i64,
    modified_timestamp_ms: i64,
}

impl Order {
    fn new() -> Order {
        Order {
            id: 0,
            order_state: OrderState::Init,
            create_timestamp_ms: Local::now().timestamp_millis(),
            modified_timestamp_ms: Local::now().timestamp_millis(),
        }
    }
}
```

你需要在 main.rs 中添加以下代码，让 module tree 中包含 order.rs 中的内容：

```rust
// 将 order.rs 加入到 modules tree
pub mod order;

fn main() {
    // ...
}
```

此外，你还需要修改 order.rs，把你想要对外露出的结构体、枚举修改为 public:

```rust
use chrono::Local;

pub enum OrderState {
    Init,
    PendingPayment,
    PendingDeliver,
    Finish,
}

#[derive(Debug)]
pub struct Order {
    id: i64,
    order_state: OrderState,
    create_timestamp_ms: i64,
    modified_timestamp_ms: i64,
}

impl Order {
    pub fn new() -> Order {
        Order {
            id: 0,
            order_state: OrderState::Init,
            create_timestamp_ms: Local::now().timestamp_millis(),
            modified_timestamp_ms: Local::now().timestamp_millis(),
        }
    }
}
```

这样就可以在 main.rs 中访问到这些代码了：

```rust
pub mod order;

fn main() {
    let order = order::Order::new();
    println!("{:#?}", order);
}
```

main.rs 中 `pub mod order;` 这行代码，是告诉 rust 载入同级目录中的 order.rs 文件，或者载入 order/mod.rs 文件，比如你想把文件结构改成下面这种：

```txt
.
├── Cargo.toml
└── src
    ├── main.rs
    └── order
       └── order.rs
```

想要让 main.rs 访问到 order/order.rs，就需要创建一个 order/mod.rs 文件：

```rust
// order/mod.rs 文件

pub mod order;
```

这样 main.rs 中的 `pub mod order;` 这行代码，就可以定位到 order/mod.rs 文件，随后载入其中的 modules.

因为添加了一个文件夹，所以 path 需要修改一下：

```rust
pub mod order;

fn main() {
    // path 里多了一个 order
    let order = order::order::Order::new();
    println!("{:#?}", order);
}
```

上面的代码提到了 path 的概念，意思是访问 module 的时候需要指明这个 module 在 module tree 中的位置。你可以使用 use 关键字来导入 path 下的内容，这样就不需要每次都写完整 path 了：

```rust
pub mod order;

// 使用 use 关键字导入 module 中的内容
use crate::order::order::Order;

fn main() {
    let order = Order::new();
    println!("{:#?}", order);
}
```

更多信息可以参考 [这篇文章](http://www.sheshbabu.com/posts/rust-module-system/?spm=ata.13261165.0.0.7b6861b6u5SVuw)