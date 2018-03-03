---
layout: post
title:  "Spock 测试框架"
date:   2018-03-03 14:53:30 +0800
categories: java
---

* TOC
{:toc}


Spock 是一个 Java 测试框架，使用 groovy 脚本语言，我之前没接触过 groovy, 但是看上去它跟 Java 挺像。

这片文章介绍一下 spock 的特点。

## 结构可读性

优秀的 junit 测试，其结构有一种模式，被称为 arrange-act-assert 模式。它将测试分为三个阶段：

```java
@Test
public void loginWithPassword() {
    // 准备各种参数
    String userName = "dongdada";
    String password = "panda";
    LoginService loginService = LoginService.getInstance();

    // 执行测试
    loginService.doLoginWithPassword(userName, password);

    // 判断执行结果
    assertEquals("login should be success", true, loginService.isLogin());
}
```

Spock 的一个优点在于，它对上述不同的阶段进行了显式的标识，使得这一结构具有很好的可读性：

```groovy
public void "login with password - happy path" () {
    setup:
    def userName = "dongdada"
    def password = "panda"

    when:
    loginService.doLoginWithPassword(userName, password)

    then:
    loginService.isLogin() == true
}
```

上述例子中涉及到两个概念，specification(说明) 和 phases(阶段).

对于 specification 而言，需要明白一件事情——单元测试应该先于编码来进行。正常的流程应该是：
1. 编写接口，规定需要做的事情有哪些；
2. 编写单元测试，详细说明每件事情要达到哪些要求；
3. 编写实现类来实现这些要求；

不过实际开发中经常会出现 1-3-2, 甚至 3-1-2 这样的顺序。这就有可能造成对需求理解不清晰而返工的情况。

Spock 将单元测试中的不同阶段用相应的 block 来进行标识，对应关系如下：

![]( {{site.url}}/asset/spock-blocks-to-phases.png )

- `setup` : 一些准备工作，初始化、参数之类的。也可以用 `given`, 两者等价；
- `when` 和 `then` : 两者必须配合使用，`when` 表示执行操作，`then` 表示该操作所期望的结果。一个方法中可以包含多个 `when`, `then` 对，用来描述不同情况下的测试工作；
- `expect` : 可以看作同时执行了 `when` 和 `then`, 有的判断只需要一个表达式既可以完成，也就不需要分成 `when` 和 `then` 两个步骤了；
- `cleanup` : 执行清理工作；
- `where` : 这个 block 比较特别，它用于编写 data-driven 的测试函数，data-driven 可以意会一下，以后再去介绍。

你还可以给每个 block 加上说明，使其意义更加清晰，把上述所有 block 用到一个例子里：

```groovy
def "some complex feature" () {
    setup "prepare some account":
    // ...

    // 进行多种不同的测试
    when "login with email account":
    // ...
    then "login success":
    // ...

    when "login with phone number account:
    // ...
    then "login success":
    // ...

    expect "login with invalid account should be failed:
    // ...

    cleanup "clean login status":
    // ...
}

def "some data drive feature" () {
    expect "get correct maximum":
    Math.max(a, b) == c

    where:
    a << [5, 3]
    b << [1, 9]
    c << [5, 9]
}
```


## 参数化测试

待补充。。。


## 参考

- [Spock Primer](http://spockframework.org/spock/docs/1.0/spock_primer.html)