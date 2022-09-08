---
layout: post
title:  "C++ - 避免使用单例"
date:   2022-09-07 21:06:30 +0800
categories: cpp
---

* TOC
{:toc}


在使用单例过程中遇到的主要问题是它让写单侧变得比较麻烦，比如下面的代码无法测试对单例的调用：

```cpp
void foo() {
  // 调用单例
  Singleton::getInstance().bar();

  // ...
}
```

但是怎么避免写单例呢？一般写单例的情况是对象的所有权不属于任何人，又想像个工具类那样在各个地方调用。可以考虑用这样的方式来处理：

为项目整体维护一个 Instance 对象，其中提供了各种方法可以拿到所需的 "单例":

```cpp
// Instance 接口，定义了获取各种单例对象的方法
class Instance {
public:
  virtual PluginManager& pluginManager() = 0;
  virtual ConfigManager& configManager() = 0;  
};

// Instance 接口的实现类
class InstanceImpl : public Instance {
public:
  // 提供 init 和 shutdown 方法，用于创建、销毁各个单例对象
  bool init() {
    plugin_manager_ = std::make_unique<PluginManager>();
    config_manager_ = std::make_unique<ConfigManager>();
  }

  PluginManager& pluginManager() override {
    return *plugin_manager_;
  }

  ConfigManager& configManager() override {
    return *config_manager_;
  }

private:
  std::unique_ptr<PluginManager> plugin_manager_;
  std::unique_ptr<ConfigManager> config_manager_;
};

// Instance 接口的 mock 类，方便做单元测试
class MockInstance : public Instance {
  MOCK_METHOD0(pluginManager, PluginManager&());
  MOCK_METHOD0(configManager, ConfigManager&());
};
```

Instance 对象应当在代码开始处创建，如果有对象需要用到单例，就把 Instance 一路传递下去：

```cpp
class Connection {
public:
  Connection(Instance& instance) : instance_(instance) {}

private:
  Instance& instance_;
};

class ConnectionPool {
public:
  ConnectionPool(Instance& instance) : instance_(instance) {}

  std::unique_ptr<Connection> newConnection(std::string ip, uint16_t port) {
    // ...

    return std::make_unique<Connection>(instance_);
  }

private:
  Instance& instance_;
};
```

这样的话 "单例" 对象就很容易 mock 出来，而且单例都被包含在 Instance 接口上面了，一路往下传的成本也比较低，只需要传一个 Instance 引用下去就行，不用传很多单例对象下去。

如果单例之间有依赖关系，也比较容易在代码里控制。