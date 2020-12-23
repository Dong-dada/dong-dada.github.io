---
layout: post
title:  "设计模式 - MVC"
date:   2020-12-23 13:32:30 +0800
categories: pattern
---

* TOC
{:toc}

![]( {{site.url}}/asset/mvc-pattern.jpg)

上图摘自 stackoverflow 的 [这个提问](https://stackoverflow.com/questions/5217611/the-mvc-pattern-and-swing)。

简单来说：
1. View 是向用户展示 Model 的窗口。
  - 用户与 View 交互，这个交互包含两种，一种是查看 View 展示的内容，另一种是点击 View 上的按钮。
  - View 上的按钮之类的被点击之后，View 把这件事告知给 Controller
2. Controller 操作 Model, 修改 Model 的数据。
3. Controller 收到 View 发来的事件后，有可能需要对 View 进行修改。比如点击了登录按钮，Controller 收到此事件，将按钮置灰。
4. Model 的数据发生改变时，发事件给 View，View 根据数据的变化来更新界面。
5. View 在更新界面时，有时候需要从 model 那里拿数据。比如 View 收到了歌曲切换到下一首的事件，随后向 Model 发起请求，获取下一首歌的详细信息，然后显示到界面上。


从以往的开发经验来看，View 和 Controller 的职责往往分得不是很清楚。
- 一种常见的情况是 View 很薄，只负责布局和展示内容。下面的几件事都在 Controller 里面：
  - (V)处理 View 上用户触发的事件;
  - (V)监听 Model 事件，更新 View。
  - (C)根据用户触发的事件来修改 View。
  - (C)操作 Model;
- 另一种常见的情况是 View 很厚，不仅负责布局和展示。下面的几件事都在 View 里面：
  - (V)处理 View 上用户触发的事件；
  - (V)监听 Model 事件，更新 View。
  - (C)根据用户触发的事件来修改 View。
  - (C)操作 Model;


另外按照上面的思路的话，View 的更新有两个入口，一个是 View 自己，另一个是 Controller。写代码的时候不太容易分清楚更新 View 的代码到底应该在哪个入口做，最后的结果很可能是统一在某个地方做，然后打破 MVC 架构。