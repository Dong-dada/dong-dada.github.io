---
layout: post
title:  "Rust - 异步编程"
date:   2021-02-03 21:59:59 +0800
categories: rust
---

* TOC
{:toc}

本文翻译自 [Asynchronous Programming in Rust](https://rust-lang.github.io/async-book/01_getting_started/01_chapter.html)


# 简介

## 为什么需要 Async?

在普通的多线程程序中，如果你想下载两个网页，你需要把下载任务放到两个线程中执行：

```rust
fn get_two_sites() {
    // Spawn two threads to do work.
    let thread_one = thread::spawn(|| download("https://www.foo.com"));
    let thread_two = thread::spawn(|| download("https://www.bar.com"));

    // Wait for both threads to complete.
    thread_one.join().expect("thread one panicked");
    thread_two.join().expect("thread two panicked");
}
```

多线程程序看起来挺好的，但线程切换和线程之间共享数据会涉及到许多开销。一个线程即使处于什么都不做的睡眠状态，也会占用系统资源。异步代码的作用就是消除这些成本。你可以使用 Rust 的 `async/await` 关键字来重写上述代码，重写之后我们可以在单个线程中运行多个任务：

```rust
async fn get_two_sites_async() {
    // Create two different "futures" which, when run to completion,
    // will asynchronously download the webpages.
    let future_one = download_async("https://www.foo.com");
    let future_two = download_async("https://www.bar.com");

    // Run both futures to completion at the same time.
    join!(future_one, future_two);
}
```

异步程序有可能让程序执行得更快，并节省资源，但也有其代价。线程是由操作系统本身提供支持的，使用线程时你并不需要任何特殊的语言模型 —— 你可以在任意函数中创建线程，调用这些会创建线程的函数与调用普通函数相比没有什么不同。然而异步函数需要有特殊的语言模型或者库来支持。在 Rust 当中，`async fn` 会创建一个异步函数，异步函数会返回一个 `Future`. 要执行异步函数体中的代码，你必须运行函数返回的 `Future`。

注意传统的多线程程序也可以很有效率，异步编程给程序带来的复杂度不一定总是能够带来价值。你需要考虑哪种模型更适合你的程序。

注意同步和异步代码不能直接混用，你不能在 sync function 里直接调用一个 async function, 同步和异步是两种不同的编程模式，很难混用。甚至异步代码之间有时候也不能混用，因为不同的 crate 可能依赖不同的 async 运行时。


