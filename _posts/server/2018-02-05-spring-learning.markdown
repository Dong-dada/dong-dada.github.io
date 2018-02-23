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

