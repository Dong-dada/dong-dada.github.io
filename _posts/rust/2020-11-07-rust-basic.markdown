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


