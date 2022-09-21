---
layout: post
title:  "C++ - TaskRunner 的销毁"
date:   2022-09-21 10:40:30 +0800
categories: cpp
---

* TOC
{:toc}


# 背景

以下是一个简单的 TaskRunner 实现，一个 TaskRunner 对象包含一个线程和一个任务队列，你可以使用 submit 方法向任务队列中投递任务，TaskRunner 内的线程会每隔 10ms 执行你的任务。

```cpp
#include <atomic>
#include <iostream>
#include <memory>
#include <mutex>
#include <functional>
#include <thread>
#include <vector>
#include <list>

using std::cout;
using std::endl;

class TaskRunner {
public:
  TaskRunner() {
    thread_ = std::make_unique<std::thread>(&TaskRunner::threadFunc, this);
  }

  ~TaskRunner() {
    thread_stop_flag_ = true;
    thread_->join();
    thread_.reset();
  }

  using Task = std::function<void()>;
  void submit(Task&& task) {
    std::lock_guard<std::mutex> lock_guard(tasks_mutex_);
    tasks_.push_back(std::move(task));
  }

private:
  void threadFunc() {
    cout << "threadFunc begin" << endl;

    while (!thread_stop_flag_) {
      std::this_thread::sleep_for(std::chrono::milliseconds(10));

      std::lock_guard<std::mutex> lock_guard(tasks_mutex_);
      if (tasks_.empty()) {
        continue;
      }
      auto task = tasks_.front();
      tasks_.pop_front();

      task();
    }

    cout << "threadFunc end" << endl;
  }

private:
  std::unique_ptr<std::thread> thread_;
  std::atomic_bool thread_stop_flag_{false};

  std::list<Task> tasks_;
  std::mutex tasks_mutex_;
};

int main(int argc, char** argv) {
  auto runner = std::make_unique<TaskRunner>();

  for (int i = 0; i < 5; ++i) {
    runner->submit([i]() {
      cout << "task " << i << " running..." << endl;
    });
  }

  std::this_thread::sleep_for(std::chrono::seconds(1));
  cout << "main exit" << endl;
  return 0;
}
```

以上程序运行后的输出是：

```
threadFunc begin
task 0 running...
task 1 running...
task 2 running...
task 3 running...
task 4 running...
main exit
threadFunc end
```

重点关注 TaskRunner 对象的销毁过程，当主函数作用于结束后，TaskRunner 对象被销毁，在 TaskRunner 对象的析构函数中会先设置一个标记，然后通过 `std::thread::join()` 方法等待线程退出：

```cpp
class TaskRunner {
  // ...

  ~TaskRunner() {
    thread_stop_flag_ = true;
    thread_->join();
    thread_.reset();
  }

  void threadFunc() {
    cout << "threadFunc begin" << endl;

    while (!thread_stop_flag_) {
      // ...
    }

    cout << "threadFunc end" << endl;
  }
};
```

这种做法看起来没啥问题，但如果 TaskRunner 对象是在其自身线程被销毁的，就会出现问题，比如我们向消息队列中投递一个任务，这个任务就是销毁 TaskRunner 自身：

```cpp
int main(int argc, char** argv) {
  auto runner = std::make_unique<TaskRunner>();

  runner->submit([&runner]() {
    runner.reset();
  });

  std::this_thread::sleep_for(std::chrono::seconds(1));
  cout << "main exit" << endl;
  return 0;
}
```

此时运行程序会崩溃，其输出如下：

```
threadFunc begin
terminate called after throwing an instance of 'std::system_error'
  what():  Resource deadlock avoided
```

这是因为 `std::thread::join()` 方法无法在当前线程中执行，很显然自己等待自己执行完成会导致死锁，因此 `std::thread::join()` 方法在这种情况下会抛出 deadlock 错误。


# 一种解决方法

出现上面问题，比较直观的思路是对 "当前线程中析构 TaskRunner" 这种情况做特殊处理，避免调用 `std::thread::join()` 函数：

```cpp
class TaskRunner {
  // ...

  ~TaskRunner() {
    thread_stop_flag_ = true;

    if (std::this_thread::get_id() == thread_->get_id()) {
      // 如果是当前线程，就不等待线程退出了
      thread_->detach();
      thread_.release();
    } else {
      thread_->join();
      thread_.reset();
    }
  }

  void threadFunc() {
    cout << "threadFunc begin" << endl;

    // 这里的 thread_stop_flag_ 可能已经被析构
    while (!thread_stop_flag_) {
      std::this_thread::sleep_for(std::chrono::milliseconds(10));

      std::lock_guard<std::mutex> lock_guard(tasks_mutex_);
      if (tasks_.empty()) {
        continue;
      }
      auto task = tasks_.front();
      tasks_.pop_front();

      // 此处可能析构 TaskRunner 对象
      task();
    }

    cout << "threadFunc end" << endl;
  }
};
```

这种做法带来的问题是 TaskRunner 被析构之后，其线程函数仍然在运行，可能会继续访问 TaskRunner 的成员变量，导致未定义的行为。

为此可以考虑把线程函数运行时需要访问的成员变量包装到结构体里，这个结构体的生命周期由线程函数维持，TaskRunner 只是引用这个结构体，这样 TaskRunner 对象的销毁就不会影响线程函数的执行了：

```cpp
#include <atomic>
#include <iostream>
#include <memory>
#include <mutex>
#include <functional>
#include <thread>
#include <vector>
#include <list>

using std::cout;
using std::endl;

class TaskRunner {
public:
  TaskRunner() {
    auto context = std::make_unique<TaskRunnerContext>();
    context_ = context.get();

    thread_ = std::make_unique<std::thread>(&TaskRunner::threadFunc, std::move(context));
  }

  ~TaskRunner() {
    context_->thread_stop_flag_ = true;

    if (thread_->get_id() == std::this_thread::get_id()) {
      thread_->detach();
      thread_.release();
    } else {
      thread_->join();
    }
  }

  using Task = std::function<void()>;
  void submit(Task&& task) {
    std::lock_guard<std::mutex> lock_guard(context_->tasks_mutex_);
    context_->tasks_.push_back(std::move(task));
  }

private:
  struct TaskRunnerContext {
    std::atomic_bool thread_stop_flag_{false};
    std::list<Task> tasks_;
    std::mutex tasks_mutex_;
  };

  // 因为 threadFunc 不再依赖 TaskRunner，所以可以是静态方法
  static void threadFunc(std::unique_ptr<TaskRunnerContext> context) {
    cout << "threadFunc begin" << endl;

    while (!context->thread_stop_flag_) {
      std::this_thread::sleep_for(std::chrono::milliseconds(10));

      std::lock_guard<std::mutex> lock_guard(context->tasks_mutex_);
      if (context->tasks_.empty()) {
        continue;
      }
      auto task = context->tasks_.front();
      context->tasks_.pop_front();

      task();
    }

    cout << "threadFunc end" << endl;
  }

private:
  TaskRunnerContext* context_;
  std::unique_ptr<std::thread> thread_;
};

int main(int argc, char** argv) {
  auto runner = std::make_unique<TaskRunner>();

  for (int i = 0; i < 5; ++i) {
    runner->submit([i]() {
      cout << "task " << i << " running..." << endl;
    });
  }

  runner->submit([&runner]() {
    runner.reset();
  });

  std::this_thread::sleep_for(std::chrono::seconds(1));
  cout << "main exit" << endl;
  return 0;
}
```

