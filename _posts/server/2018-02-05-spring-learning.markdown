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

### 容器

在 Spring 框架中，对象生存于 Spring 容器中。Spring 容器负责创建对象，装配它们，配置它们并管理它们的整个生存周期：

![]( {{site.url}}/asset/spring-container.png )

Spring 容器并不是只有一个，Spring 带有多个容器实现，可以归纳为两种不同的类型：
- bean 工厂是最简单的容器，提供基本的 DI 支持。
- 应用上下文基于 BeanFactory 构架，提供应用框架级别的服务。

bean 工厂比较低级，应用上下文更受欢迎。

### 应用上下文

Spring 自带了多种类型的应用上下文：
- AnnotationConfigApplicationContext: 从一个或多个基于 Java 的配置类中加载 Spring 上下文；
- AnnotationConfigWebApplicationContext: 从一个或多个基于 Java 的配置类中加载 Spring Web 上下文；
- ClassPathXmlApplicationContext: 从类路径下的一个或多个 XML 配置文件中加载上下文定义，把应用上下文的定义文件作为类资源；
- FileSystemXmlApplicationContext: 从文件系统下的一个或多个 XML 配置文件中加载上下文定义；
- XmlWebApplicationContext: 从 Web 应用下的一个或多个 XML 配置文件中加载上下文定义；

下列代码展示了如何加载应用上下文：

```java
// 从不同位置加载应用上下文
ApplicationContext context = new FileSystemXmlApplicationContext("C:/knight.xml");
ApplicationContext context = new ClassPathXmlApplicationContext("knight.xml");

ApplicationContext context = new AnnotationConfigApplicationContext(com.dada.login.config.LoginConfig.class);

// 从应用上下文(容器)中获取 bean
LoginService loginService = context.getBean(LoginService.class);
```

### Spring 模块

![]( {{site.url}}/asset/spring-modules.png )


## 装配 beans

Spring 提供了三种装配机制：
- 在 XML 中进行显式配置；
- 在 Java 中进行显式配置；
- 隐式的 bean 发现机制和自动装配；

书中建议优先使用自动化装配方式，然后选择 JavaConfig 的方式，因为它类型安全且功能丰富，最后考虑选择 XML 装配方式。

### 自动化装配

Spring 从两个角度来实现自动化装配：
- 组件扫描(component scanning): Spring 会自动发现应用上下文中所创建的 bean;
- 自动装配(autowiring): Spring 自动满足 bean 之间的依赖；

要让 Spring 完成上述操作，你需要在合适的地方加上注解。比如 `@Component` 注解将告诉 Spring, 依据当前的类来创建 bean, 而 `@Autowired` 注解则告诉 Spring, 寻找合适的 bean 来满足依赖：

```java
@Component
public class SgtPeppers implements CompactDisc {
    private String title = "Sgt. Pepper's Lonely Hearts Club Band";
    private String artist = "The Beatles";

    public void play() {
        System.out.println("Playing " + title + " by " + artist);
    }
}

@Component
public class CDPlayer implements MediaPlayer {
    private CompactDisc cd;

    @Autowired
    public CDPlayer(CompactDisc cd) {
        this.cd = cd;
    }

    public void play() {
        cd.play();
    }
}
```

上述代码中通过 `@Component` 注解，创建了两个 bean, 而 `@Autowired` 注解则告诉 Spring, 在创建 CDPlayer bean 的时候，自动绑定一个实现了 CompactDisc 接口的 bean, 也就是 SgtPeppers bean.

你可能会好奇，当没有 bean 能够满足自动装配的时候，会怎样呢？当有多个 bean 都能满足自动装配的时候，会怎样呢？这一点以后再介绍。

### 通过 Java 代码装配

对于一些第三方库，没办法在其中添加自动装配的注解，这时候可以使用 JavaConfig 代码来进行装配。

JavaConfig 跟普通的 Java 代码有所区别，它是配置代码，不应该包含任何业务代码。所以通常我们会把 JavaConfig 放到单独的包中，与其他业务代码区分开来。

上一小节介绍自动装配时，我们漏掉了一段代码，Spring 并不会自动完成组建的扫描，而需要你在某个类中指定 `@ComponentScan` 注解，我们可以把这个注解放到 JavaConfig 中：

```java
package com.dada.learning;

import org.springframework.context.annotation.ComponentScan;
import org.springframework.context.annotation.Configuration;

@Configuration
@ComponentScan
public class CDPlayerConfig {
}
```

`@Configuration` 告诉 Spring, 被修饰的类是配置类，该类应该包含在 Spring 应用上下文中如何创建 bean 的细节。我们现在不使用自动装配，因此可以移除 `@ComponentScan` 注解，然后加入显式配置的代码：

```java
@Configuration
public class CDPlayerConfig {

    @Bean
    public CompactDisc sgtPeppers() {
        return new SgtPeppers();
    }

    @Bean
    public CDPlayer cdPlayer(CompactDisc cd) {
        return new CDPlayer(cd);
    }
}
```

上述代码中通过 `@Bean` 注解来修饰一个方法，Spring 会自动调用该方法，该方法返回的对象会被视为一个 bean 加入到 Spring 容器中。

在 `cdPlayer()` 方法创建 bean 的时候，会接受一个实现了 CompactDisc 接口的对象，Spring 会自动装配 SgtPeppers bean 给 `cdPlayer()` 方法，这也就实现了依赖的绑定。

### 通过 XML 装配 bean

在文章开头我们已经介绍过使用 XML 装配 bean 的方法。例如如下代码：

```xml

<beans xmlns="http://www.springframework.org/scheme/beans"
    xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
    xsi:schemaLocation="http://www.springframework.org/schema/beans http://www.springframework.org/schema/beans/spring-beans.xsd">

    <bean id="compactDisc" class="soundsystem.SgtPeppers" />

    <bean id="cdPlayer" class="soundsystem.CDPlayer">
        <constructor-arg ref="compactDisc" />
    </bean>
</beans>
```

### 导入和混合配置

有的时候，你可能觉得某个 JavaConfig 配置类过于复杂，而希望将其拆分开来，拆分完成后，你需要把其中一个配置类导入到另一个配置类中，以确保配置仍然会生效。这时候可以使用 `@Import` 注解：

```java
@Configuration
@Import({CDPlayerConfig.class, CDConfig.class})
@ImportResource("classpath:cd-config.xml")
public class SoundSystemConfig {
}
```

`@ImportResource` 注解用于导入 XML 配置。

类似的，在 XML 配置中可以使用 import 标签来导入配置。
