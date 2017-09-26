---
layout: post
title:  "设计模式小知识点"
date:   2017-09-06 11:00:30 +0800
categories: pattern
---

* TOC
{:toc}


## 观察者模式和委托模式的区别
这两个模式是有点儿相似的，都是某个特定情形发生的时候，把这件事告诉别人。包括他们的代码实现也是类似的，比如：

```cpp
// 观察者模式的例子，下面三个类处于三个不同的文件中

// 定义登陆状态的 Observer 接口，抽象用户状态相关的事件
class LoginStateObserver
{
public:
    virtual void OnLogin() = 0;
    virtual void OnLogout() = 0;
}

// 用户登陆类，负责完成实际的用户登陆逻辑
class UserLogin
{
public:
    void AddObserver(LoginStateObserver* observer);
    void RemoveObserver(LoginStateObserver* observer);

    void Login()
    {
        // ... 登陆逻辑

        // 登陆成功后通知所有 observer 登陆状态发生了变化
        for (auto& observer : observers_)
        {
            observer->OnLogin();
        }
    }
}

// 负责展示用户头像，需要观察登陆状态，从而切换用户头像
class UserHeaderView : public LoginStateObserver
{
    void OnLogin() override
    {
        // ... 用户登陆了，设置用户头像
    }

    void OnLogout() override
    {
        // ... 用户登出了，设为未登录状态下的默认头像
    }
}

// 负责展示 VIP 信息，需要观察登陆状态，切换界面显示的内容
class VipInfoView : public LoginStateObserver
{
    void OnLogin() override
    {
        // ... 用户登陆了，设置 VIP 信息
    }

    void OnLogout() override
    {
        // ... 用户登出了，隐藏 VIP 信息
    }
}
```

```cpp
// 委托模式的例子

// 定义弹出广告的 Delegate 接口，抽象弹出广告的相关接口
class AdPopupDelegate
{
    virtual bool PopupAd(const AdInfo& ad_info) = 0;
    virtual bool CancelPopupAd() = 0;
}

// 广告系统的 Controller, 收到广告数据后需要将其弹出
class AdController
{
public:
    void SetAdPopupDelegate(AdPopupDelegate* ad_popup_delegate);

    void OnReceiveAd()
    {
        // 处理该广告信息

        // 通过 delegate 委托给其他类来弹出广告
        if (!ad_popup_delegate_->PopupAd(ad_info))
        {
            // 广告弹出失败，执行一些错误处理逻辑，比如过段时间尝试再弹
        }
    }
}

// 桌面左下角形式弹出广告
class ButtomAdPopup : public AdPopupDelegate
{
    bool PopupAd(const AdInfo& ad_info) = 0;
}

// 桌面中心形式弹出广告
class CenterAdPopup : public AdPopupDelegate
{
    bool PopupAd(const AdInfo& ad_info) = 0;
}
```

可以看到，对于观察者模式来说，我只是告诉你某件事情发生了，你怎么处理跟我没关系，因此观察者模式下，Observer 可以有许多个，比如上面的代码中有两个观察者 UserHeaderView 和 VipInfoView，它们根据登陆状态来改变界面显示效果，但这对登陆操作本身不会有影响。

而对于委托模式来说，是把本来应该由我来做的事情交给你来做，你做事儿的结果会影响我下一步的操作，因此在委托模式下，被委托者只能有一个，上面的代码中定义了两个被委托者 ButtomAdPopup 和 CenterAdPopup, 这两个对象只能有一个被设置到 AdController 里面。

委托模式可以用在代码复用上，比如上面的例子中，也许一开始只有一个 AdController 类，它只提供了从底部弹出广告的功能，有一天你需要开发一个新模块，这个模块里除了广告需要从中间而不是底部弹出以外，其他的逻辑都是一样的。这种情况下，为了复用  AdController 的代码，就可以把从底部弹出的这部分功能封装为 BottomAdPopup 类，把从中间弹出的功能封装为 CenterAdPopup 类，对于不同的模块，只需要通过 SetAdPopupDelegate 设置不同的 delegate 就可以了，AdController 是在模块间复用的。

从代码的形式上看，观察者模式的接口定义没有返回值，因为观察者的行为不应当影响被观察者，另外观察者可以由很多个，所以接口往往是 AddObserver 这样的写法；而委托模式的接口定义可以有返回值，委托者发布委托之后，需要根据被委托者处理委托的情况来进行后续的操作，另外被委托者通常只有一个，因此接口往往是 SetDelegate 这样的写法；