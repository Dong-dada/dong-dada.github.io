---
layout: post
title:  "读书笔记 - 单元测试的艺术"
date:   2019-02-27 21:45:30 +0800
categories: pattern
---

* TOC
{:toc}

- 优秀的单元测试应该有如下特征
    - 自动化的，可重复执行；
    - 容易编写；
    - 运行速度快；
    - 结果稳定；
    - 与环境隔离，没有真实依赖物(当前时间、真实数据之类的)；

- 要理解什么是一个单元，你需要弄清楚自己以前做的是什么类型的测试。你发现以前所作的是集成测试，因为他测试的是一组相互依赖的单元。

- 测试驱动开发
    - 先编写测试
    - 运行测试(这一步必然失败)
    - 编写代码，使测试通过

- 一个单元测试通常包含三个主要行为
    - Arrange(准备)对象, 创建对象，进行必要的设置；
    - Act(操作)对象；
    - Assert(断言)某件事情是预期的；

- 命名: `[方法名]_[Good/Bad]_[ReturnsTrue/ReturnsFalse]`, 某个方法的正/反检验返回True/False
    - IsInvalidLogFileName_GoodExtensions_ReturnsTrue
    - IsInvalidLogFileName_BadExtension_ReturnsFalse