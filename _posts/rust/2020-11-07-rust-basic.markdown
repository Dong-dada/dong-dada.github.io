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

## 使用 Release Profiles

```s
# 执行 release 版本
$ cargo build --release
```

你可以在 Cargo.toml 中对 profile.* 文件进行修改，来控制编译过程:

```toml
[profile.dev]
opt-level = 0

[profile.release]
opt-level = 3
```

# 文档注释

```rust
/// Divide two numbers.
///
/// # Examples
///
/// ```
/// let result = my_crate::div(10, 2);
/// assert_eq!(result, 5);
/// ```
///
/// # Panics
///
/// If second number is zero, panic occurred.
///
/// ```should_panic
/// // panics on division by zero
/// my_crate::div(10, 0);
/// ```
pub fn div(a: i32, b: i32) -> i32 {
    if b == 0 {
        panic!("Divide-by-zero error");
    }

    a / b
}
```

按照以上格式对 pub fn 进行注释，随后输入以下命令，就能够生成文档：

```s
$ cargo doc --open
```

生成的文档如下图所示:

![]({{ site.url }}/asset/rust_doc_example.png)

文档注释有个好处，其中的测试用例会在每次 `cargo test` 的时候被运行，从而保证文档和代码的一致性。

```
PS D:\code\rust\rust-lib-demo> cargo test
    Finished test [unoptimized + debuginfo] target(s) in 0.01s
     Running target\debug\deps\my_crate-aaae9c20b6eaebe3.exe

running 0 tests

test result: ok. 0 passed; 0 failed; 0 ignored; 0 measured; 0 filtered out

   Doc-tests my_crate

running 2 tests
test src\lib.rs - div (line 14) ... ok
test src\lib.rs - div (line 5) ... ok

test result: ok. 2 passed; 0 failed; 0 ignored; 0 measured; 0 filtered out
```

你可以参考 [官网文档](https://doc.rust-lang.org/book/ch14-02-publishing-to-crates-io.html) 来了解文档注释的更多用法。

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


## Workspaces

有时候你希望可以统一管理多个 package，这时候可以使用 workspace 功能。Workspace 中可以包含一系列 packages，这些 packages 共享同一个 Cargo.lock 文件和输出目录。

Workspace 的使用很简单，只要在文件夹根目录定义一个 Cargo.toml 文件，输入以下内容即可:

```rust
[workspace]

members = [
    "rust-demo-app",
    "rust-demo-lib"
]
```

members 中的内容就是要包含在这个 workspace 中的 package.

# 常用集合

## Vec 

```rust
fn main() {
    // 创建
    let v: Vec<i32> = Vec::new();
    // 创建 Vec 的另一种写法(使用 vec! 宏)
    let v = vec![1, 2, 3];

    // 添加值，注意需要声明为 mut
    let mut v = Vec::new();
    v.push(5);
    v.push(6);
    v.push(7);
    v.push(8);

    // 将元素 push 到 Vec 时，所有权将转移给 Vec，这时候需要注意可变引用的作用域问题：
    let mut v = vec![1, 2, 3, 4, 5];
    let first = &v[0];
    v.push(6);
    // 下面一行代码将导致编译错误，因为 first 的作用区间和 v 重合了
    // 当你向 Vec 添加元素时，Vec 可能会重新分配内存，导致 first 的引用失效。
    // println!("The first element is: {}", first);

    // 使用索引访问元素
    let v = vec![1, 2, 3, 4, 5];
    let third: &i32 = &v[2];
    println!("The third element is {}", third);
    // 如果超出范围，程序将 panic
    // let nine = &v[8];

    // 使用 get() 方法获取元素，其返回值类型为 Option<&i32>
    let third = v.get(2);
    match third {
        None => println!("There is no third element."),
        Some(third) => println!("The third element is {}", third),
    }

    // 遍历 Vec 的元素，只读方式
    let v = vec![100, 32, 57];
    for i in &v {
        println!("{}", i);
    }

    // 遍历 Vec 的元素，读写方式
    let v = vec![100, 32, 57];
    for i in &mut v {
        *i += 50;
    }
}
```

Vec 的元素必须是同一种类型，但你可以用 Enum 来做一层包装，因为在 rust 中 Enum 可以附加指定类型的数据。

```rust
enum SpreadsheetCell {
    Int(i32),
    Float(f64),
    Text(String),
}

// 利用 Enum 在 Vec 中放置不同类型的元素
let row = vec![
    SpreadsheetCell::Int(3),
    SpreadsheetCell::Text(String::from("blue")),
    SpreadsheetCell::Float(10.12),
];
```

## String

String 也是一种集合类型，它底层是通过 Vec<u8> 来实现的。

String 和 &str 都是 utf8 编码的字符串。

```rust
fn main() {
    // 创建字符串
    let mut st = String::new();

    // 实现了 Display trait 的类型，都可以调用 to_string() 方法来获取该类型的字符串表示
    let data = "initial contents";
    let s = data.to_string();
    let s = "initial contents".to_string();

    // 通过 String::from 方法来创建字符串
    let s = String::from("initial contents");

    // 因为 String 是 UTF-8 编码，所以可以传入各种 Unicode 字符:
    let hello = String::from("السلام عليكم");
    let hello = String::from("Dobrý den");
    let hello = String::from("Hello");
    let hello = String::from("שָׁלוֹם");
    let hello = String::from("नमस्ते");
    let hello = String::from("こんにちは");
    let hello = String::from("안녕하세요");
    let hello = String::from("你好");
    let hello = String::from("Olá");
    let hello = String::from("Здравствуйте");
    let hello = String::from("Hola");

    // 添加一个字符串到字符串尾部
    let mut s1 = String::from("foo");
    let s2 = "bar";
    s1.push_str(s2);
    println!("s2 is {}", s2);   // 注意 push_str 不会夺取参数的所有权，所以此时仍然可以访问 s2

    // 添加一个字符到字符串尾部
    s1.push('l');

    // 使用 + 操作符来拼接字符串
    let s1 = String::from("Hello, ");
    let s2 = String::from("world!");
    let s3 = s1 + &s2;
    // 下面一行代码会编译失败，因为 + 操作符会夺取 s1 的所有权，然后生成一个新的变量 s3
    // 所以使用了 + 操作符拼接字符串之后，第一个字符串就不能被使用了
    // println!("s1 is {}", s1);

    // 使用 format!() 宏来拼接字符串
    let s1 = String::from("tic");
    let s2 = String::from("tac");
    let s3 = String::from("toe");
    let s = format!("{}-{}-{}", s1, s2, s3);

    // 你可以通过 [..] 操作符，来获取字符串的字节序列的一部分，这种啊方法不推荐，因为 string 是 UTF-8 编码的，有可能会截取到不完整的字符串
    let hello = "Здравствуйте";
    // Здравствуйте 的 utf-8 编码里，前 4 个字节是 Зд, 所以 s 的值是 Зд
    let s = &hello[0..4];
    println!("s is {}", s);

    // 遍历字符串中的字符(而不是字节)
    for c in "नमस्ते".chars() {
        println!("{}", c);
    }

    // 遍历字符串中的字节：
    for b in "नमस्ते".bytes() {
        println!("{}", b);
    }
}
```

注意使用 + 操作符拼接字符串时，第一个字符串的所有权会被夺走，+ 操作符实际上对应了以下方法：

```rust
fn add(self, s: &str) -> String {
    // ...
}
```

可以看到参数生命中 self 并不是引用，因此它会把第一个字符串的所有权夺走。

另外可以看到第二个参数实际上是 &str 类型，但是你传入的参数实际上是 &String。之所以可以将 String 和 &String 进行拼接，是因为 Rust 做了自动装包解包的操作，把 &String 转换成了 &str.


## HashMap

```rust
use std::collections::HashMap;

fn main() {
    // 创建 HashMap
    let mut scores: HashMap<String, u32> = HashMap::new();

    // 插入元素
    scores.insert(String::from("Blue"), 10);
    scores.insert(String::from("Yellow"), 50);

    // 通过 collect 函数将 Vec 转换为 HashMap
    let teams = vec![String::from("Blue"), String::from("Yellow")];
    let initial_scores = vec![10, 50];
    let mut scores: HashMap<_, _> = teams.into_iter().zip(initial_scores.into_iter()).collect();

    // 插入元素时将夺取参数的所有权
    let field_name = String::from("Favorite color");
    let field_value = String::from("Blue");
    let mut map = HashMap::new();
    map.insert(field_name, field_value);
    // 以下代码将导致编译失败，因为 HashMap.insert(key, value) 将会夺取参数的所有权。
    // println!("field_name = {}, field_value = {}", field_name, field_value);

    // 访问 HashMap 中的元素
    // get 的返回值是一个 Optional<&i32>
    let team_name = String::from("Blue");
    let score = scores.get(&team_name);
    match score {
        None => println!("The {} team is not existed.", team_name),
        Some(score) => println!("The {} team score is {}.", team_name, score),
    }

    // 遍历 HashMap 中的元素
    for (key, value) in &scores {
        println!("{}: {}", key, value);
    }

    // 使用 entry() 函数，只有 key 不存在时才插入元素
    let mut scores = HashMap::new();
    scores.insert(String::from("Blue"), 10);
    scores.entry(String::from("Blue")).or_insert(50);

    // entry() 函数会返回一个 value 引用，可以用这个引用来修改值
    let score = scores.entry(String::from("Blue")).or_insert(0);
    *score += 1;
}
```


# 错误处理

Rust 里包含两种错误 recoverable 和 unrecoverable. 比如文件找不到就是一种 recoverable 错误，你可以创建一个新文件来修复这个错误。unrecoverable 错误通常是 bug 导致，比如访问一块非法内存。

Rust 的错误处理机制并不是基于异常，而是使用 `Result<T, E>` 枚举来表示 recoverable 错误，使用 `panic!` 宏来处理 unrecoverable 错误。

## panic! 宏

panic! 宏执行时，程序将打印出错信息，随后退出。

Unwinding:
> 默认情况下，panic 发生时程序会进行 unwinding 处理，此时 rust 会退出函数堆栈，并清理每个函数调用中使用到的数据。这个过程可能会比较耗时，你可以在 Cargo.toml 的 `[profile.release]` 中添加 `panic = 'abort'` 来避免 release 版本的包执行 unwinding 操作。

以下例子在代码中调用了 `panic!` 宏：

```rust
fn main() {
    panic!("crash and burn");
}
```

执行后将打印以下内容：

```s
$ cargo run
   Compiling panic v0.1.0 (file:///projects/panic)
    Finished dev [unoptimized + debuginfo] target(s) in 0.25s
     Running `target/debug/panic`
thread 'main' panicked at 'crash and burn', src/main.rs:2:5
note: run with `RUST_BACKTRACE=1` environment variable to display a backtrace.
```

可以看到出错信息中提示了 `src/main.rs:2:5` 这行代码处发生了 panic. 你还可以通过设置 `RUST_BACKTRACE=1` 环境变量来打印堆栈信息。

打印堆栈信息的前提是编译结果带有符号。带有符号的前提是编译时没有设置 `--release` 标记。

## Result<T, E> 枚举

```rust
enum Result<T, E> {
    Ok(T),
    Err(E),
}
```

例如打开文件时，如果文件不存在，将返回 Err:

```rust
use std::fs::File;

fn main() {
    let f = File::open("hello.txt");

    let f = match f {
        Ok(file) => file,
        Err(error) => panic!("Problem opening the file: {:?}", error)
    }
}
```

你还可以通过 error.kind() 来确定出现问题的具体原因：

```rust
use std::fs::File;
use std::io::ErrorKind;

fn main() {
    let f = File::open("hello.txt");
    let f = match f {
        Ok(file) => file,
        Err(error) => match error.kind() {
            ErrorKind::NotFound => match File::create("hello.txt") {
                Ok(fc) => fc,
                Err(e) => panic!("Problem creating the file: {:?}", e),
            },
            other_error => {
                panic!("Problem opening the file: {:?}", other_error)
            }
        },
    };
}
```

rust 还提供了一些简单写法：

```rust
use std::fs::File;

fn main() {
    // unwrap(): 能打开的话，返回 File, 打不开的话调用 panic!
    let f = File::open("hello.txt").unwrap();

    // expect(): 能打开的话，返回 File, 打不开的话调用 panic!, 并设置出错信息
    let f = File::open("hello.txt").expect("Failed to open hello.txt");
}
```

有的时候我们希望把错误信息往上抛：

```rust
fn read_username_from_file() -> Result<String, io::Error> {
    let f = File::open("hello.txt");

    // 打开文件失败的话，把错误往上抛
    let mut f = match f {
        Ok(file) => file,
        Err(e) => return Err(e),
    };

    let mut s = String::new();

    // 读取文件失败的话，把错误往上抛
    match f.read_to_string(&mut s) {
        Ok(_) => Ok(s),
        Err(e) => Err(e),
    }
}
```

rust 也为这种情况提供了简单写法：

```rust
fn read_username_from_file() -> Result<String, io::Error> {
    // 注解结尾的 ?, 表示如果发生了错误，那么直接返回
    let mut f = File::open("hello.txt")?;

    let mut s = String::new();
    // 注解结尾的 ?, 表示如果发生了错误，那么直接返回
    f.read_to_string(&mut s)?;

    // 返回正确结果
    Ok(s)
}
```

上面的代码甚至可以连起来写成一行：

```rust
fn read_username_from_file() -> Result<String, io::Error> {
    let mut s = String::new();

    // 连起来写成一行
    File::open("hello.txt")?.read_to_string(&mut s)?;

    Ok(s)
}
```

## 什么时候使用 !panic ?

默认情况下返回 Result 会比较好，因为可以让上层来决定是否要 panic。不过在一些情况下使用 panic! 可能会让事情更简单：
- 编写原型代码、测试代码时，不需要考虑完善的错误处理机制，直接调用 panic! 宏可以让代码编写更容易。
- 有的时候你知道代码一定不会出错，比如 `let home: IpAddr = "127.0.0.1".parse().unwrap();` 这行代码中，你知道你传入的一定是一个有效的 ip 地址，因此不可能发生地址解析错误。

有的时候你希望对外提供的函数参数遵守一定的规则，比如传入的 pageSize 不能大于 200。如果传了 500 过来，你应该立刻调用 panic! 宏，这样调用方可以在开发期间就知道传入 500 是不对的。

创建对象的时候，你可以把类似的规则封装在 new 函数里，而不是让调用方直接操作成员变量来指定：

```rust
pub struct Guess {
    value: i32,
}

impl Guess {
    pub fn new(value: i32) -> Guess {
        if value < 1 || value > 100 {
            panic!("Guess value must be between 1 and 100, got {}.", value);
        }

        Guess { value }
    }

    pub fn value(&self) -> i32 {
        self.value
    }
}
```


# 泛型, Trait, Lifetime

## 泛型

```rust
// 在函数中使用泛型
// 注意需要 <T: PartialOrd>, 对 T 进行约束，让它支持比较
fn largest<T: PartialOrd>(list: &[T]) -> &T {
    let mut largest = &list[0];
    for item in list {
        if item > largest {
            largest = item;
        }
    }

    largest
}

fn main() {
    let number_list = vec![1, 2, 3, 4, 5];
    println!("largest number is {}.", largest(&number_list));

    let char_list = vec!['a', 'b', 'c', 'd', 'e'];
    println!("largest char is {}.", largest(&char_list));
}
```

```rust
// 在结构体中使用泛型
struct Point<T> {
    x: T,
    y: T,
}

// 在结构体成员函数中使用泛型
impl<T> Point<T> {
    fn x(&self) -> &T {
        &self.x
    }

    fn y(&self) -> &T {
        &self.y
    }
}

// 也可以只针对特定类型提供成员函数的实现
impl Point<f32> {
    fn distance_from_origin(&self) -> f32 {
        (self.x.powi(2) + self.y.powi(2)).sqrt()
    }
}

fn main() {
    let both_integer = Point { x: 5, y: 10 };
    println!("x: {}, y: {}", both_integer.x(), both_integer.y());

    let both_float = Point { x: 1.0, y: 1.0 };
    println!("distance from origin: {}", both_float.distance_from_origin());

    // 以下代码将报错，因为没有对 i32 类型提供 distance_from_origin() 方法
    //println!("distance from origin: {}", both_integer.distance_from_origin());
}
```

```rust
// 在枚举中使用泛型
enum Option<T> {
    Some(T),
    None,
}

enum Result<T, E> {
    Ok(T),
    Err(E),
}
```

## Trait

Trait 跟其他语言中的 interface 概念类似。

```rust
use std::fmt::{Display, Formatter};
use std::fmt;

// 定义 trait
trait Summary {
    fn summarize(&self) -> String;
}

struct NewsArticle {
    headline: String,
    location: String,
    author: String,
    content: String,
}

// 实现 trait
impl Summary for NewsArticle {
    fn summarize(&self) -> String {
        format!("{}, by {} ({})", self.headline, self.author, self.location)
    }
}

struct Tweet {
    username: String,
    content: String,
    reply: bool,
    retweet: bool,
}

// 实现 Trait
impl Summary for Tweet {
    fn summarize(&self) -> String {
        format!("{}: {}", self.username, self.content)
    }
}

impl Display for Tweet {
    fn fmt(&self, f: &mut Formatter<'_>) -> fmt::Result {
        write!(f, "Tweet (username: {}, content: {}, reply: {}, retweet: {})", self.username, self.content, self.reply, self.retweet)
    }
}

// Trait 作为函数参数
fn notify(item: &impl Summary) {
    println!("Breaking news! {}", item.summarize());
}

// 要求参数必须实现多个 trait
fn another_notify(item: &(impl Summary + Display)) {
    println!("Breaking news! {}", item);
}

// 返回值中使用 Trait
fn returns_summarizable() -> impl Summary {
    Tweet {
        username: "dong-dada".to_string(),
        content: "hello rust!".to_string(),
        reply: false,
        retweet: false,
    }
}

// 注意下面的写法会编译出错，trait 作为返回值时，只能返回某种特定的类型
// fn returns_summarizable(switch: bool) -> impl Summary {
//     if switch {
//         NewsArticle {
//             headline: String::from(
//                 "Penguins win the Stanley Cup Championship!",
//             ),
//             location: String::from("Pittsburgh, PA, USA"),
//             author: String::from("Iceburgh"),
//             content: String::from(
//                 "The Pittsburgh Penguins once again are the best \
//                  hockey team in the NHL.",
//             ),
//         }
//     } else {
//         Tweet {
//             username: String::from("horse_ebooks"),
//             content: String::from(
//                 "of course, as you probably already know, people",
//             ),
//             reply: false,
//             retweet: false,
//         }
//     }
// }

fn main() {
    let tweet = Tweet {
        username: "dong-dada".to_string(),
        content: "hello rust!".to_string(),
        reply: false,
        retweet: false,
    };

    let news_article = NewsArticle {
        headline: "Chang'e 5 lifts off".to_string(),
        location: "Beijing".to_string(),
        author: "dong-dada".to_string(),
        content: "balabala...".to_string(),
    };

    notify(&tweet);
    notify(&news_article);
    another_notify(&tweet);

    let summary = returns_summarizable().summarize();
    println!("summary: {}", summary);
}
```


## Lifetime

只有引用才涉及到了生命周期的概念，其他类型不涉及。

Rust 当中的所有引用都有一个生命周期，这个生命周期表示了引用的作用域。大多情况下你不需要关心引用的作用域，编译期会帮你做各种检查。但有时候有一些必要的信息编译器没办法知道，就需要你手动标记生命周期。

```rust
// 以下代码将编译失败
{
    let r;                  // ---------+-- 'a 
                            //          |
    {                       //          |
        let x = 5;          // -+-- 'b  |
        r = &x;             //  |       |
    }                       // -+       |
                            //          |
    println!("r: {}", r);   //          |
}                           // ---------+
```

以上代码将编译失败，上述代码分别标识除了变量 r 的生命周期 'a 和变量 x 的生命周期 'b，编译器在编译的时候发现 r 引用了 b，但 r 的生命周期比 b 还要大，因此认为编译失败。

```rust
// 以下代码可以编译成功
{
    let x = 5;              // ----------+-- 'b
                            //           |
    let r = &x;             // --+-- 'a  |
                            //   |       |
    println!("r: {}", r);   //   |       |
                            // --+       |
}                           // ----------+
```

以上代码不会编译失败，编译器编译的时候发现 r 引用了 b, 并且 b 的生命周期比 r 大，因此认为编译成功。

### 函数中使用 lifetime

在上面的两种情况下，编译器都能判断出来生命周期是否合法。但有一些情况下编译器是判断不出来的，比如下面的代码，要在运行期才能做出判断。

```rust
fn longest(x: &str, y: &str) -> &str {
    if x.len() > y.len() {
        x
    } else {
        y
    }
}
```

上面的代码中，编译器和我们都不知道要返回的引用是 x 的引用还是 y 的引用。此时可以使用 lifetime annotation 对参数、返回值的生命周期进行标注，告知编译器他们生命周期之间的关系。

```rust
// 使用 lifetime 注解，对参数和返回值的生命周期进行标注
// 下面的注解表示参数 x, y 和返回值，三者在函数内必须有相同的生命周期。
// 话句话说返回值的 lifetime 与参数中比较小的那个 lifetime 一样。
fn longest<'a>(x: &'a str, y: &'a str) -> &'a str {
    if x.len() > y.len() {
        x
    } else {
        y
    }
}


fn main() {
    let string1 = String::from("long string is long");

    // result 生命周期和 string2 一致，因此下面代码可以编译通过
    {
        let string2 = String::from("xyz");
        let result = longest(string1.as_str(), string2.as_str());
        println!("The longest string is {}", result);
    }

    // result 生命周期和 string2 一致，因此下面代码无法编译通过
    let result;
    {
        let string2 = String::from("xyz");
        result = longest(string1.as_str(), string2.as_str());
    }
    println!("The longest string is {}", result);
}
```

注意 lifetime annotation 标记是用来告诉编译器变量之间的生命周期关系的，如果你写出的代码实际上不能满足这个关系，那么会编译失败。

```rust
// 以下代码将编译失败

// 告知编译器 返回值的生命周期是 x 和 y 当中比较小的那一个
fn longest<'a>(x: &str, y: &str) -> &'a str {
    // 实际上返回值的生命周期是局部变量的生命周期
    let result = String::from("really long string");
    result.as_str()
}
```

### 结构体中使用 lifetime

在结构体中使用引用作为成员变量，此时要设置 lifetime annotation:

```rust
// 表示成员变量 part 的生命周期，必须大于等于 ImportantExcerpt 对象的生命周期
struct ImportantExcerpt<'a> {
    part: &'a str,
}

fn main() {
    let novel = String::from("Call me Ishmael. Some years ago...");
    let first_sentence = novel.split('.').next().expect("Could not find a '.'");
    let i = ImportantExcerpt {
        part: first_sentence,
    };
}
```

### Lifetime 什么情况下可以省略

如果函数只有一个参数、一个返回值，那么 lifetime annotation 标记可以省略，编译器实际上会自动帮我们加上注解:

```rust
// 一个参数、一个返回值的函数，编译器会默认加上注解，表示返回值的生命周期至少与参数的生命周期相同
fn first_word<'a>(s: &'a str) -> &'a str {
    // ...
}
```

如果没有设置 lifetime annotation，编译器会按照以下三个规则来进行设置：
1. 函数中的每个引用类型的参数，都各自有一个 lifetime annotation，比如有两个引用参数的函数，会自动增加 lifetime annoation，变成类似 `fn foo<'a, 'b>(x: &'a i32, y: &'b i32)` 的样子，也就是表示参数的的生命周期是彼此独立的。
2. 如果函数只有一个引用类型的参数，那么它的 lifetime 会被赋值给所有返回值：`fn foo<'a>(x: &'a i32) -> &'a i32`。也就是表示返回值的生命周期默认与参数的一样。
3. 如果函数有多个引用类型的参数，但其中一个参数是 `&self` 或者 `&mut self`。此时 `&self` 的 lifetime 将被赋值给所有返回值。也就是返回值的生命周期默认与结构体对象本身一样。


```rust
impl<'a> ImportantExcerpt<'a> {
    // 如果是结构体成员函数，那么返回值的 lifetime 默认与 &self 一样
    fn announce_and_return_part(&self, announcement: &str) -> &str {
        println!("Attention please: {}", announcement);
        self.part
    }
}
```

### The Static Lifetime

`'static` 表示引用在整个应用生命周期内都可用。所有的 string 字面量都是 `'static` 的:

```rust
let s: &'static str = "I have a static lifetime.";
```


# 单元测试

使用 `#[test]` 属性修饰方法，即可将该方法标记为测试用例。

```rust
pub fn add_two(a: i32) -> i32 {
    internal_adder(a, 2)
}

fn internal_adder(a: i32, b: i32) -> i32 {
    return a + b;
}

// #[cfg(test)] 表示 module 内的代码仅会在 cargo test 时被编译
#[cfg(test)]
mod tests {
    use super::*;

    #[test]
    fn exploration() {
        // 在测试用例中使用断言
        assert_eq!(2 + 2, 4);
    }

    #[test]
    fn another() {
        // 在测试用例中使用 panic
        panic!("Make this test fail");
    }

    // 使用 #[ignore] 属性来跳过用例的执行
    #[test]
    #[ignore]
    fn expensive_test() {
        // 耗时操作
    }

    #[test]
    fn internal() {
        // rust 中可以对私有方法进行测试
        assert_eq!(4, internal_adder(2, 2));
    }
}
```

随后即可使用 `cargo test` 命令来运行所有用例。


## Cargo 命令

```s
# 默认情况下单元测试是并行跑的，可以通过 --test-threads 参数来设置线程数量
$ cargo test -- --test-threads=1

# 默认情况下，测试通过的用例，其 stdout 不会展示出来，通过 --show-outpu 参数可以展示这些内容
$ cargo test -- --show-output

# 默认情况下 cargo test 将运行项目中的所有用例，可以传入用例的名称，这样只会执行一部分用例。
# 只执行名称中包含 `add` 的用例
$ cargo test add

# 单独运行一下被跳过的用例
$ cargo test -- --ignored
```


# 函数式语言的特性 - Closure 和 Iterator

## Closure

Closure 是一种可以保存变量或者参数的匿名函数。你可以把方法及其上下文保存起来，然后在另外一个地方执行，达到类似于延迟执行的效果。

先来看一个简单的例子:

```rust
use std::thread;
use std::time::Duration;

// 模拟一个耗时的调用，它的业务含义是根据输入的强度，输出锻炼计划
fn simulated_expensive_calculation(intensity: u32) -> u32 {
    println!("calculating slowly...");
    thread::sleep(Duration::from_secs(2));
    return intensity;
}

fn generate_workout(intensity: u32, random_number: u32) {
    expensive_result = simulated_expensive_calculation(intensity);

    if intensity < 25 {
        // 如果强度较低，那么先做若干次俯卧撑，再做若干次仰卧起坐
        println!("Today, do {} push-ups!", expensive_result);
        println!("Next, do {} sit-ups", expensive_result);
    } else {
        // 如果强度比较高，随机休息一下，其它时间再按照较高强度来练习
        if random_number == 3 {
            println!("Take a break today! Remember to stay hydrated!");
        } else {
            println!("Today, run for {} minutes!", expensive_result);
        }
    }
}

fn main() {
    let simulated_user_specified_value = 10;
    let simulated_random_number = 7;

    generate_workout(simulated_user_specified_value, simulated_random_number);
}
```

上述代码中，耗时调用是在进入 generate_workout 的时候立刻被执行的，我们可以使用 closure 对其重构，使耗时调用在需要的时候被延迟执行:

```rust
use std::thread;
use std::time::Duration;

fn generate_workout(intensity: u32, random_number: u32) {
    // 把耗时调用放到 closure 里面，让它延迟执行
    let expensive_closure = |num| {
        println!("calculating slowly...");
        thread::sleep(Duration::from_secs(2));
        return intensity;
    };

    if intensity < 25 {
        // 如果强度较低，那么先做若干次俯卧撑，再做若干次仰卧起坐
        println!("Today, do {} push-ups!", expensive_closure(intensity));
        println!("Next, do {} sit-ups", expensive_closure(intensity));
    } else {
        // 如果强度比较高，随机休息一下，其它时间再按照较高强度来练习
        if random_number == 3 {
            println!("Take a break today! Remember to stay hydrated!");
        } else {
            println!("Today, run for {} minutes!", expensive_closure(intensity));
        }
    }
}

fn main() {
    let simulated_user_specified_value = 10;
    let simulated_random_number = 7;

    generate_workout(simulated_user_specified_value, simulated_random_number);
}
```

不过重构之后耗时调用被执行了两次，我们可以构造一个 Cacher，把计算的结果缓存起来:

```rust
use std::thread;
use std::time::Duration;

struct Cacher<T>
where T: Fn(u32) -> u32,
{
    calculation: T,

    // 缓存计算结果
    value: Option<u32>,
}

impl<T> Cacher<T>
where T: Fn(u32) -> u32,
{
    fn new(calculation: T) -> Cacher<T> {
        Cacher {
            calculation,
            value: None
        }
    }

    fn value(&mut self, arg: u32) -> u32 {
        // 已经计算过的话直接返回，没有计算过的话调用 calculation 方法进行计算
        match self.value {
            Some(v) => v,
            None => {
                let v = (self.calculation)(arg);
                self.value = Some(v);
                return v;
            }
        }
    }
}

fn generate_workout(intensity: u32, random_number: u32) {
    // 把耗时调用放到 closure 里面，让它延迟执行
    let mut expensive_result = Cacher::new(|num: u32| -> u32 {
        println!("calculating slowly...");
        thread::sleep(Duration::from_secs(2));
        return num;
    });

    if intensity < 25 {
        // 如果强度较低，那么先做若干次俯卧撑，再做若干次仰卧起坐
        println!("Today, do {} push-ups!", expensive_result.value(intensity));
        println!("Next, do {} sit-ups", expensive_result.value(intensity));
    } else {
        // 如果强度比较高，随机休息一下，其它时间再按照较高强度来练习
        if random_number == 3 {
            println!("Take a break today! Remember to stay hydrated!");
        } else {
            println!("Today, run for {} minutes!", expensive_result.value(intensity));
        }
    }
}

fn main() {
    let simulated_user_specified_value = 10;
    let simulated_random_number = 7;

    generate_workout(simulated_user_specified_value, simulated_random_number);
}
```

不过上面的代码还是有点问题，首先 Cacher 只能缓存一个值，如果传给 Cacher::value() 的 intensity 不一样，它不会再次计算；其次 Cacher 的 calculation 只能接受 u32 类型的参数，输出 u32 类型的返回值，不够通用。

第一个问题比较好解决，用一个 HashMap 来存储参数和返回值就好了，第二个问题需要用到泛型，试了几回没有成功。

```rust
struct Cacher<T>
where T: Fn(u32) -> u32
{
    calculation: T,
    values: HashMap<u32, u32>,
}

impl<T> Cacher<T>
where T: Fn(u32) -> u32
{
    fn new(calculation: T) -> Cacher<T> {
        Cacher {
            calculation,
            values: HashMap::new(),
        }
    }

    fn value(&mut self, arg: u32) -> u32 {
        let value = self.values.get(&arg);
        match value {
            Some(v) => v.clone(),
            None => {
                let v = (self.calculation)(arg);
                self.values.insert(arg, v);
                return v;
            }
        }
    }
}
```

总的来说上面的例子都可以用普通的函数来实现，不过 Closure 还有一个普通函数不具备的能力：它可以把局部变量等内容存起来，然后延迟地去访问它：

```rust
fn main() {
    let x = 4;

    let equal_to_x = |z| {
        // 判断传入的值与局部变量 x 是否相等
        return z == x;
    };

    let y = 4;

    assert!(equal_to_x(y));
}
```

对于捕获的变量，Clusure 需要使用额外的内存来存储它。Cluster 捕获变量的方式有以下几种：
- FnOnce: 将获取变量的所有权；
- FnMut: 将会借用变量(获取变量的可变引用)，并且修改这个变量
- Fn: 将会借用变量(获取变量的不可变引用)

如果你希望让 closure 强制获取到变量的所有权，可以使用 `move` 关键字：

```rust
fn main() {
    let x = vec![1, 2, 3];

    // 使用 move 关键字，让 closure 夺取变量 x 的所有权
    let equal_to_x = move |z| z == x;

    // 以下代码将导致报错，因为 x 的所有权已经被 closure 夺走了
    println!("can't use x here: {:?}", x);

    let y = vec![1, 2, 3];

    assert!(equal_to_x(y));
}
```


## Iterators

示例:

```rust
#[test]
fn test_iterator_basic() {
    let vec = vec![1, 2, 3];

    // 获取 Vector 的迭代器
    let iter = vec.iter();

    // 使用迭代器遍历 Vector 的所有元素
    for value in iter {
        println!("Got: {}", value);
    }

    // 迭代器返回的是元素的引用类型，因此可以通过迭代器来修改元素
    let mut vec = vec;
    let iter = vec.iter_mut();
    for value in iter {
        *value += 1;
    }
    println!("{:?}", vec);
}

#[test]
fn test_iterator_next() {
    let vec = vec![1, 2, 3];

    // 也可以调用 next() 方法来获取迭代器的下一个元素，注意需要用 mut 修饰，因为 next() 方法将修改迭代器内的数据
    let mut iter = vec.iter();

    // next() 方法返回的是一个 Option
    assert_eq!(iter.next(), Some(&1));
    assert_eq!(iter.next(), Some(&2));
    assert_eq!(iter.next(), Some(&3));
    assert_eq!(iter.next(), None);
}

#[test]
fn test_iterator_sum() {
    let vec = vec![1, 2, 3];
    let iter = vec.iter();

    // sum() 方法会把每个元素加起来
    let total: i32 = iter.sum();

    assert_eq!(total, 6);
}

#[test]
fn test_iterator_map() {
    let vec: Vec<i32> = vec![1, 2, 3];

    // 迭代器可以通过 map().collect() 实现类似于 java Stream API 的效果。
    // 也就是依次对每个元素执行 map(), filter() 等操作，最后再把这些元素合并成一个新的 Vector 返回
    let vec: Vec<i32> = vec.iter()
        .map(|x| x + 1)
        .filter(|x| x % 2 == 0)
        .collect();

    assert_eq!(vec, vec![2, 4]);
}
```

## 通过实现 Iterator Trait 来创建自己的迭代器

Iterator Trait 的定义如下，你需要实现 next() 方法:

```rust
pub trait Iterator {
    type Item;

    fn next(&mut self) -> Option<Self::Item>;

    // methods with default implementations elided
}
```

```rust
struct Counter {
    count: u32,
}

impl Counter {
    fn new() -> Counter {
        Counter { count: 0 }
    }
}

impl Iterator for Counter {
    type Item = u32;

    fn next(&mut self) -> Option<Self::Item> {
        // 实现 next() 方法，返回的 count 每次加 1，最大值为 5
        if self.count < 5 {
            self.count += 1;
            Some(self.count)
        } else {
            None
        }
    }
}

#[test]
fn calling_next_directly() {
    let mut counter = Counter::new();

    assert_eq!(counter.next(), Some(1));
    assert_eq!(counter.next(), Some(2));
    assert_eq!(counter.next(), Some(3));
    assert_eq!(counter.next(), Some(4));
    assert_eq!(counter.next(), Some(5));
    assert_eq!(counter.next(), None);
}

#[test]
fn using_other_iterator_trait_methods() {
    // 实现 Iterator trait 之后，就可以调用 Iterator 的各种方法了

    // Iterator.skip(n) 方法, 能够跳过迭代器的前 n 个元素
    // Iterator.zip() 方法，能够将两个迭代器合并在一起，每次迭代都返回一个 tuple, 其中包含了两个迭代器的迭代结果

    let sum: u32 = Counter::new()
        .zip(Counter::new().skip(1))    // (1, 2), (2, 3), (3, 4), (4, 5)
        .map(|(a, b)| a * b)    // 2, 6, 12, 20
        .filter(|x| x % 3 == 0) // 6, 12
        .sum();
    assert_eq!(18, sum);
}
```


# 智能指针

Rust 中的引用是一种通用的指针，你可以用 `&` 符号来借用它所指向的值。相比引用而言，智能指针具有更多的功能。

Rust 标准库中提供了以下几种智能指针:
- `Box<T>` 用于在堆上分配内存;
- `Rc<T>` 能够让一块内存被多个 owner 持有;
- `Ref<T>` 和 `RefMut<T>` 这两种类型通过 `RefCell<T>` 进行访问，它可以在运行期而非编译期保障 borrowing rules.

## Box<T>

`Box<T>` 类似于 C++ 当中的 `std::unique_ptr<T>`，通常用在以下场合：
- 类型的大小在编译期无法确定，要在运行期才能确定，但是你又必须在确定大小的场合下访问该类型的变量，也就是递归类型。
- 类型的数据量很大，因此希望在转移所有权时不要进行拷贝。
- 希望持有基类指针，也就是希望持有 Trait 而非具体实现。

简单示例:

```rust
fn main() {
    let b = Box::new(5);
    println!("b = {}", b);
}
```

递归类型：

```rust
use crate::List::{Cons, Nil};

// 递归类型无法在编译时知道大小
enum List {
    Cons(i32, List),
    Nil,
}

fn main() {
    // 以下代码将导致编译错误，因为出现了递归类型
    let list = Cons(1, Cons(2, Cons(3, Nil)));
}
```

```rust
use crate::List::{Cons, Nil};

// 用智能指针来引用递归类型，这样就可以在编译期让类型有确定的大小。
#[derive(Debug)]
enum List {
    Cons(i32, Box<List>),
    Nil,
}

fn main() {
    let list = Cons(1, Box::new(Cons(2, Box::new(Cons(3, Box::new(Nil))))));
    println!("{:#?}", list);
}
```

`Box<T>` 实现了 `Deref` Trait，因此可以像使用引用那样来使用 `Box<T>`。此外 `Box<T>` 还实现了 `Drop` Trait，它会在指针离开作用域的时候销毁相关内容。

## Deref Trait

实现 `Deref` Trait 可以让你自定义 *解引用操作符(dereference operator)* `*` 的行为。`Box<T>` 实现了 `Deref` Trait，因此你可以像使用普通引用那样来使用 `Box<T>`。

```rust
fn main() {
    let x = 5;
    let y = &x;

    // 对普通的引用类型使用解引用操作符
    assert_eq!(5, *y);

    let x = 5;
    let y = Box::new(x);

    // 对 Box<T> 变量使用解引用操作符
    assert_eq!(5, *y);
}
```

`Deref` Trait 的实现关键在于，其中定义的 `deref()` 方法返回了实际对象的引用：

```rust
use std::ops::Deref;

struct MyBox<T>(T);

impl<T> MyBox<T> {
    fn new(x: T) -> MyBox<T> {
        MyBox(x)
    }
}

impl<T> Deref for MyBox<T> {
    type Target = T;

    // 注意 deref() 的返回值是 T 的引用
    fn deref(&self) -> &T {
        &self.0
    }
}

fn hello(name: &str) {
    println!("Hello, {}!", name);
}

fn main() {
    let x = 5;
    let y = MyBox::new(x);

    // *y 实际上是调用了 *(y.deref())
    assert_eq!(5, *y);

    let m = MyBox::new(String::from("Rust"));
    // 注意因为这里的 MyBox 实现了 Deref Trait, 因此 Rust 能够进行隐式转换
    // 它首先将 &m 从 &MyBox<String> 类型转换为 &String 类型，
    // 因为 String 也实现了 Deref Trait，因此 Rust 随后将 &String 类型转换为 &str 类型
    hello(&m);
}
```

## Drop Trait

```rust
struct CustomSmartPointer {
    data: String,
}

// Drop trait 类似于 C++ 里的析构函数，你可以在指针及其内容被析构时做一些事情
impl Drop for CustomSmartPointer {
    fn drop(&mut self) {
        // 打一行日志记录对象析构
        println!("Dropping CustomSmartPointer with data `{}`", self.data);
    }
}

fn main() {
    let c = CustomSmartPointer {
        data: String::from("my stuff"),
    };
    let d = CustomSmartPointer {
        data: String::from("other stuff"),
    };
    println!("CustomSmartPointers created.");

    // 使用 std::mem::drop() 方法可以提前析构一个对象
    // 注意 drop(c) 会夺取所有权，所以调用了 drop(c) 之后就不能再访问这个变量了。
    std::mem::drop(c);
    println!("CustomSmartPointers dropped the end of main.")

    // 输出结果为:
    // CustomSmartPointers created.
    // Dropping CustomSmartPointer with data `my stuff`
    // CustomSmartPointers dropped the end of main.
    // Dropping CustomSmartPointer with data `other stuff`
}
```

## RC<T> 引用计数智能指针

有时候某个值可能会有多个 owner, 比如在图数据结构中，可能会有多条边指向相同的节点，从概念上说这些节点应该被所有的边拥有，直到没有边在拥有节点时，该节点才能被释放。
> 注意 Rc<T> 是非线程安全的。

```rust
use crate::List::{Cons, Nil};
use std::rc::Rc;

enum List {
    Cons(i32, Rc<List>),
    Nil
}

fn main() {
    // 使用 Rc<T>::new() 方法来创建一个 Rc 指针，使用 Rc::strong_count() 来获取当前的引用计数
    let a = Rc::new(Cons(5, Rc::new(Cons(10, Rc::new(Nil)))));
    println!("count after creating a = {}", Rc::strong_count(&a));

    // 使用 Rc::clone() 方法来复制一个 Rc 指针，此时 a, b 都指向同样的内容
    let b = Cons(3, Rc::clone(&a));
    println!("count after creating b = {}", Rc::strong_count(&a));

    {
        let c = Cons(4, Rc::clone(&a));
        println!("count after creating c = {}", Rc::strong_count(&a));
    }
    println!("count after c goes out of scope = {}", Rc::strong_count(&a));

    // 输出以下内容:
    // count after creating a = 1
    // count after creating b = 2
    // count after creating c = 3
    // count after c goes out of scope = 2
}
```

## RefCell<T>

重新回忆一下 Rust 的 borrowing rules:
- 任何时候，你只能拥有 一个可变引用 **或者** 多个不可变引用。
- 引用必须总是有效的。

对于基本的引用类型以及 `Box<T>` 来说，borrowing rules 是在编译期被保障的。而 `RefCell<T>` 的作用则是在运行期保障 borrowing rules，如果运行期发生了 borrowing rules 破坏的情况，程序会 panic.

总的来说，`RefCell<T>` 主要用在这样的一种场景下：编译器认为你的代码没有遵守 borrowing rules 因此拒绝了编译, 但你知道你的代码遵守了 borrowing rules, 你希望你的代码能够编译通过。

以下是选择智能指针的原因概述：
- `Rc<T>` 允许一个值有多个所有者；`Box<T>` 和 `RefCell<T>` 只允许有一个所有者；
- `Box<T>` 支持编译期的 immutable 和 mutable borrows 检查；`Rc<T>` 仅支持编译期的 immutable borrows 检查；`RefCell<T>` 支持运行期的 immutable 和 mutable borrows 检查。
- 因为 `RefCell<T>` 允许运行期的 mutable borrows 检查，所以即使 `RefCell<T>` 是不可变的，你也可以通过 `RefCell<T>` 来修改变量。

以下是一个使用 `RefCell<T>` 的例子，在这个例子中 `Messenger` Trait 的 `sent(&self, message: &str)` 方法指明了 self 是不可变的，但我们希望 MockMessenger 可以把消息记录下来。

```rust
pub trait Messenger {
    fn send(&self, msg: &str);
}

pub struct LimitTracker<'a, T: Messenger> {
    messenger: &'a T,
    value: usize,
    max: usize
}

impl<'a, T> LimitTracker<'a, T>
where T: Messenger,
{
    pub fn new(messenger: &T, max: usize) -> LimitTracker<T> {
        LimitTracker {
            messenger,
            value: 0,
            max,
        }
    }

    pub fn set_value(&mut self, value: usize) {
        self.value = value;
        let percentage_of_max = self.value as f64 / self.max as f64;
        if percentage_of_max >= 1.0 {
            self.messenger.send("Error: You are over your quota!");
        } else if percentage_of_max >= 0.9 {
            self.messenger.send("Urgent warning: You've used up over 90% of your quota!");
        } else if percentage_of_max >= 0.75 {
            self.messenger.send("Warning: You've used up over 75% of your quota!");
        }
    }
}

#[cfg(test)]
mod tests {
    use super::*;
    use std::cell::RefCell;

    struct MockMessenger {
        sent_messages: RefCell<Vec<String>>,
    }

    impl MockMessenger {
        fn new() -> MockMessenger {
            MockMessenger {
                sent_messages: RefCell::new(vec![]),
            }
        }
    }

    impl Messenger for MockMessenger {
        // send() 方法的 &self 是不可变的，但这时候我们希望修改 sent_messages 变量的内容
        // 为了达成这个目的，可以使用 RefCell<T> 指针指向 sent_messages
        fn send(&self, msg: &str) {
            // 尽管 self 是不可变的，但是可以通过 RefCell::borrow_mut() 方法来获取可变引用
            self.sent_messages.borrow_mut().push(String::from(msg));
        }
    }

    #[test]
    fn it_sends_an_over_75_percent_warning_message() {
        let mock_messenger = MockMessenger::new();
        let mut limit_tracker = LimitTracker::new(&mock_messenger, 100);

        limit_tracker.set_value(80);

        // 通过 RefCell::borrow() 来获取不可变引用
        assert_eq!(mock_messenger.sent_messages.borrow().len(), 1);
    }
}
```

注意 `RefCell<T>` 是在运行期维护了 borrows rule, 因此以下代码会在运行期出现 panic 错误：

```rust
impl Messenger for MockMessenger {
    fn send(&self, msg: &str) {
        // 同一个变量有两个可变引用，在运行期报 BorrowMutError 错误
        let mut one_borrow = self.sent_messages.borrow_mut();
        let mut two_borrow = self.sent_messages.borrow_mut();
        one_borrow.push(String::from(msg));
        two_borrow.push(String::from(msg));
    }
}
```

报错内容如下:

```
thread 'main' panicked at 'already borrowed: BorrowMutError'
```

## 结合 Rc<T> 和 RefCell<T> 来让多个所有者持有可变引用

`Rc<T>` 让一个变量可以有多个所有者，但这些所有者持有的都是不可变引用。`RefCell<T>` 可以让你修改不可变的变量。把这两个结合起来，就可以让一份数据被多个 owner 持有，并且这些 owner 都能修改这个变量。

```rust
use std::rc::Rc;
use std::cell::RefCell;
use crate::List::{Cons, Nil};

#[derive(Debug)]
enum List {
    Cons(Rc<RefCell<i32>>, Rc<List>),
    Nil
}

fn main() {
    let value = Rc::new(RefCell::new(5));
    let a = Rc::new(Cons(Rc::clone(&value), Rc::new(Nil)));
    let b = Cons(Rc::new(RefCell::new(3)), Rc::clone(&a));
    let c = Cons(Rc::new(RefCell::new(4)), Rc::clone(&a));

    *value.borrow_mut() += 10;

    println!("a after = {:?}", a);
    println!("b after = {:?}", b);
    println!("c after = {:?}", c);

    // 输出以下内容，注意 value 的值被修改后也会影响到其它变量
    // a after = Cons(RefCell { value: 15 }, Nil)
    // b after = Cons(RefCell { value: 3 }, Cons(RefCell { value: 15 }, Nil))
    // c after = Cons(RefCell { value: 4 }, Cons(RefCell { value: 15 }, Nil))
}
```

## 使用 Weak<T> 循环引用导致的内存泄漏

虽然 rust 有很多机制来帮助我们实现内存安全，但还是有些情况 rust 没法检查出来，比如循环引用：

```rust
use std::rc::Rc;
use std::cell::RefCell;
use crate::List::{Cons, Nil};

#[derive(Debug)]
enum List {
    // Cons 中的第二个变量，指向另一个节点，从而形成链表
    Cons(i32, RefCell<Rc<List>>),
    Nil
}

impl List {
    // tail 函数用于获取下个节点
    fn tail(&self) -> Option<&RefCell<Rc<List>>> {
        match self {
            Cons(_, item) => Some(item),
            Nil => None
        }
    }
}

fn main() {
    // 创建一个节点 a
    let a = Rc::new(Cons(5, RefCell::new(Rc::new(Nil))));
    println!("a initial rc count = {}", Rc::strong_count(&a));
    println!("a next item = {:?}", a.tail());

    // 创建一个节点 b, 并让节点 b 指向节点 a
    let b = Rc::new(Cons(10, RefCell::new(Rc::clone(&a))));
    println!("a rc count after b creation = {}", Rc::strong_count(&a));
    println!("b initial rc count = {}", Rc::strong_count(&b));
    println!("b next item = {:?}", b.tail());

    // 让 a 节点指向 b 节点，从而形成循环引用
    if let Some(link) = a.tail() {
        *link.borrow_mut() = Rc::clone(&b);
    }

    // 此时 a 和 b 的引用计数都是 2
    // main 函数执行结束后，因为 a 和 b 离开了作用域，因此引用计数变为 1，变量不会被销毁
    println!("b rc count after changing a = {}", Rc::strong_count(&b));
    println!("a rc count after changing b = {}", Rc::strong_count(&a));

    // 以下代码将导致 stack overflow, 因为 println 会一直尝试打印 next 节点的内容，不断循环
    println!("a next item = {:?}", a.tail());

    // 输出结果:
    // a initial rc count = 1
    // a next item = Some(RefCell { value: Nil })
    // a rc count after b creation = 2
    // b initial rc count = 1
    // b next item = Some(RefCell { value: Cons(5, RefCell { value: Nil }) })
    // b rc count after changing a = 2
    // a rc count after changing b = 2
    // a next item = Some(RefCell { value: Cons(10, RefCell { ....
    // thread 'main' has overflowed its stack
}
```

使用 `Weak<T>` 来避免循环引用问题：

```rust
use std::cell::RefCell;
use std::rc::{Rc, Weak};

#[derive(Debug)]
struct Node {
    value: i32,
    // 在 children 成员中记录子节点列表，如此一来可以构造出树形结构
    children: RefCell<Vec<Rc<Node>>>,
    // 在 parent 成员中记录父节点，如此一来可以方便搜索，注意此处使用了 Weak<T> 指针
    parent: RefCell<Weak<Node>>,
}

fn main() {
    // 创建一个节点 leaf
    let leaf = Rc::new(Node {
        value: 3,
        parent: RefCell::new(Weak::new()),
        children: RefCell::new(vec![]),
    });
    // 此时提权会失败
    println!("leaf parent = {:?}", leaf.parent.borrow().upgrade());

    // 创建一个节点，把 leaf 作为它的子节点
    let branch = Rc::new(Node {
        value: 5,
        parent: RefCell::new(Weak::new()),
        children: RefCell::new(vec![Rc::clone(&leaf)]),
    });
    // 把 branch 作为 leaf 的父节点
    *leaf.parent.borrow_mut() = Rc::downgrade(&branch);

    // 此时提权可以成功
    println!("leaf parent = {:#?}", leaf.parent.borrow().upgrade());

    // 输出结果
    // leaf parent = None
    // leaf parent = Some(
    //     Node {
    //         value: 5,
    //         children: RefCell {
    //             value: [
    //                 Node {
    //                     value: 3,
    //                     children: RefCell {
    //                         value: [],
    //                     },
    //                     parent: RefCell {
    //                         value: (Weak),
    //                     },
    //                 },
    //             ],
    //         },
    //         parent: RefCell {
    //             value: (Weak),
    //         },
    //     },
    // )
}
```

最后说下 `Rc<T>` 和 `Weak<T>` 的实现原理：
- `Rc<T>` 会记录两个指针，一个指向数据，另一个指向共享数据块。在共享数据块里记录了引用计数。每次调用 `Rc<T>::clone()` 时，除了复制这两份指针，还会将共享数据块中的引用计数加一。每次 `Rc<T>` 被 drop 时，会检查共享数据快中的引用计数是否为 0, 如果为 0, 则删除原始数据。
- `Weak<T>` 需要通过 `Rc<T>::downgrade()` 来创建，它和 `Rc<T>` 指向同一个共享数据块。共享数据块里除了记录引用计数 reference count 之外，还会记录一个 weak count, 表示弱引用的计数。reference count 变成 0 之后，`Rc<T>` 指向的原始资源会被销毁，如果此时 weak count 不为 0，那么共享数据块不会被销毁。直到所有 `Weak<T>` 被释放，weak count 变成 0 之后，共享数据块才会被销毁。
- 通过 `Weak<T>::upgrade()` 进行提权时，会检查其中的 reference count 是不是已经变成 0 了，如果是，则提权失败；如果大于 0，则会将 reference count 加一，然后创建一个新的 `Rc<T>` 出来。