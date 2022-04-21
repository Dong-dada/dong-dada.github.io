---
layout: post
title:  "C++ - 所有权"
date:   2022-04-21 11:00:30 +0800
categories: cpp
---

* TOC
{:toc}


# 所有权转移

什么场景适合这种形式？
- 自己不是变量的 owner, 但是具有构造变量的职责


```cpp
void consumeTask(Task&& task) {
  // ...
}

int main() {
  Task task;

  // 所有权转移给函数
  consumeTask(std::move(task));

  // 转移之后对象就不能继续使用了
  // task.getId();
}
```


# 引用

什么场景适合这种形式？
- 被调用的函数不是变量的 owner, 不负责变量的生命周期
- 被调用的函数不会保存引用
- 能够保证变量的生命周期大于引用者

```cpp
// 工具函数，不会保存引用
int sum(const std::vector<int>& numbers) {
    int sum = 0;
    for (int number : numbers) {
        sum += number;
    }
    return sum;
}

int main() {
    std::vector<int> numbers = {1, 2, 3, 4, 5};

    cout << sum(numbers) << endl;

    return 0;
}
```

```cpp
class Context {
};

class Router {
public:
    // 能够保证 Context 的生命周期大于 Router
    explicit Router(Context& context) : context_(context) {}

private:
    Context& context_;
};
```


# 拷贝

什么场景适合这种形式：
- 变量是值类型，没有 ownership 概念