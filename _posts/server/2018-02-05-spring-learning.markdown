---
layout: post
title:  "Spring 基础知识"
date:   2018-02-23 10:49:30 +0800
categories: server
---

* TOC
{:toc}


## 简介

Spring 是一个开源框架，它的目标是简化 Java 开发的复杂性。需要注意的它不仅仅用在服务器端开发上，任何 Java 应用都能在简单性、可测试性、松耦合等方面从 Spring 上收益。

Spring 采取了以下 4 种关键策略来简化 Java 开发：
- 基于 POJO 的轻量级和最小侵入性编程；
- 通过依赖注入和面向接口实现松耦合；
- 基于切面和惯例进行声明式编程；
- 通过切面和模版减少样板式代码；

第一点中的 POJO 意思是 Plain Old Java Object, 也就是普通的 Java 对象。所谓的最小侵入性编程，意思是 Spring 不会侵入你的代码，它不会强迫你实现一些接口或继承一些类。

### 依赖注入

第二点中的依赖注入(Dependency Injection, DI), 是一种编程技术，目的是为了减少对象之间的耦合。按照传统的做法，每个对象负责管理与自己相互协作的对象(即它所依赖的对象)的引用，这会导致高度耦合和难以测试的代码。比如下面的例子：

```java
public class DamselRescuingKnight implements Knight {
    // DamselRescuingKnight 持有 RescueDamselQuest 的引用，强耦合
    private RescueDamselQuest quest;

    public DamselRescuingKnight() {
        // 创建 RescueDamselQuest 对象，强耦合
        this.quest = new RescueDamselQuest();
    }

    public void embarkOnQuest() {
        // 要测试该函数，必须先创建 RescueDamselQuest 对象，不利于测试
        quest.embark();
    }
}
```

利用 DI 技术，Spring 会帮你解决对象间的耦合问题，你无需在代码中创建或管理两者的依赖关系。上述例子中的 RescueDamselQuest 会由 Spring 框架注入到 DamselRescuingKnight 对象中。你只需通过接口来获取所依赖的对象的引用，而不需要知道是谁实现了这个接口：

```java
public class BraveKnight implements Knight {
    // 使用接口来引用 quest 对象
    private Quest quest;

    public BraveKnight (Quest quest) {
        // 由 Spring 框架来负责创建 quest 对象；
        // 由 Spring 框架来负责把 quest 对象的引用传过来；
        this.quest = quest;
    }

    public void embarkOnQuest() {
        // 可以创建一个 mock 对象来进行测试，无需创建一个真实的 Quest 对象
        quest.embark();
    }
}
```

上述例子中，使用 Quest 接口来引用实际的对象，假定已经有一个 SlayDragonQuest 类实现了 Quest 接口。那么如何创建 SlayDragonQuest 对象，并将其与 BraveKnight 关联起来呢？Spring 把创建组件之间协作关系的行为叫做装配(wiring)，它提供了多种装配方式，例如使用 XML 来进行装配：

```xml
<beans xmlns="http://www.springframework.org/scheme/beans"
    xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
    xsi:schemaLocation="http://www.springframework.org/schema/beans http://www.springframework.org/schema/beans/spring-beans.xsd">

    <!-- 把 BraveKnight 变成一个 bean -->
    <bean id="knight" class="com.dada.knights.BraveKnight">
        <!-- 通过 ref 来配置 knight 和 quest 之间的依赖关系 -->
        <constructor-arg ref="quest"/>
    </bean>

    <bean id="quest" class="com.dada.knights.SlayDragonQuest">
        <constructor-arg value="#{T(System).out}" />
    </bean>
</beans>
```

接下来，只需要告诉 Spring 加载我们编写好的 xml 配置，它就能将 BraveKnight 和 SlayDragonQuest 对象创建并装配起来：

```java
public class KnightMain {
    public static void main(String[] args) throws Exception {
        // 加载 xml 配置
        ClassPathXmlApplicationContext context = new ClassPathXmlApplicationContext(
            "META-INF/spring/knights.xml");
        
        // 获取 bean
        Knight knight = context.getBean(Knight.class);

        // 调用 bean 的方法
        knight.embarkOnQuest();
        context.close();
    }
}
```

### 面向切面编程

第三点的面向切面编程 (Aspect-Oriented Programming, AOP) 是一种编程思想。它往往被定义为促使软件系统实现关注点的分离的一项技术。

大多数软件都有许多组件组成，每个组件负责一部分特定功能，我们总希望这些组件能有更高的内聚性，而不是分散在项目的各处，需要关注很多地方。比如针对登陆模块的日志功能，按照传统的思路，也许我们会写出以下的代码：

```java
public class LoginService {
    public void OnLoginSuccess() {
        // ...

        // 在项目代码的各个函数中记录登陆事件
        RecordLoginEvent("success");
    }

    public void OnLoginFailed() {
        // ...

        // 在项目代码的各个函数中记录登陆事件
        RecordLoginEvent("failed");
    }

    // ...
}
```

面向切面的思想，就是把分散在项目各处的这些功能分离出来，形成一个可重用的组件。对于上述例子来说，登陆事件的记录就是一个切面。类似的，登陆情况的统计也可以看作一个切面。

按照上述思路，我们可以封装 LoginEventRecoder, LoginEventRepoter 等类来实现这些切面：

```java
public class LoginEventRecoder {
    public void RecordLoginSuccess() {
    }

    public void RecordLoginFailed() {
    }

    // ...
}
```

有了 LoginEventRecoder 这个切面之后，我们还是得在 LoginService 的合适地方去调用它的方法才行。不过 Spring 提供了简单的方式来对切面进行配置，例如：

```xml
<beans xmlns="http://www.springframework.org/schema/beans"
    xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
    xmlns:aop="http://www.springframework.org/schema/aop
        http://www.springframework.org/schema/aop/spring-aop-3.2.xsd
        http://www.springframework.org/schema/beans
        http://www.springframework.org/schema/beans/spring-beans.xsd" >

    <beans id="loginService" class="com.dada.login.LoginService">
    </beans>

    <beans id="loginEventRecoder" class="com.dada.login.LoginEventRecoder">
    </beans>

    <aop:config>
        <!-- 把 loginEventRecoder 声明为一个切面 -->
        <aop:aspect ref="loginEventRecoder">
            <!-- 定义一个切点，它关心 OnLoginSuccess 函数的执行 -->
            <aop:poingcut id="loginSuccess"
                expression="execution(* *.OnLoginSuccess(..))" />

            <!-- 声明前置通知，当切点 loginSuccess 执行后，将触发 RecordLoginSuccess -->
            <aop:after pointcut-ref="loginSuccess" method="RecordLoginSuccess"/>

            <!-- 按照同样方法可以定义 loginFailed 切点 -->
        </aop:aspect>
    </aop:config>
</beans>
```

通过少量的配置，Spring 就可以帮你把 LoginEventRecoder 声明为一个切面，并将其与 LoginService 关联起来。从代码上看来，LoginEventRecoder 与 LoginService 完全独立，并且 LoginEventRecoder 只关注于如何记录登陆事件。

### 使用模板消除样板式代码

Spring 提供了一些模板，可以帮助你减少一些看起来差不多的代码。比如使用 JDBC 来访问数据库，你总是需要连接数据库、创建一个 SQL 语句对象、捕捉 SQLException 等等。而使用 JdbcTemplate，你就只需要关心要查询的内容是什么就可以了。

这一点就不写示例代码了，以后单独介绍吧。