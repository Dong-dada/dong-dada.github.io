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

### 依赖注入

依赖注入(Dependency Injection, DI) 是 Spring 所支持的一种编程技术，其目的是为了解决模块间的依赖问题。

传统做法中，每个对象负责管理与自己协作的其它对象的引用，这会导致模块间的耦合和难以测试的代码。我们常常得考虑谁创建谁、谁持有谁之类的问题，很麻烦，测试的时候也不方便编写 mock 对象。

依赖注入解决这一问题的思路是面向接口，接着引入一个配置，来专门负责模块的装配。具体说来，你需要编写一个接口来描述需要实现哪些功能，然后在配置里创建实现了这些接口的类对象，最后再把这些对象装配起来。

对于实现类来说，它依赖其它类对象的方式是持有一个接口，而无需了解是谁实现了这个接口，模块间的装配由某个特定的配置来决定，这有利于模块间的松耦合。同时这也有利于编写 mock 对象进行测试，只需继承接口，然后把 mock 对象装配进去就可以了。

### 面向切面编程

面向切面编程(Aspect-Oriented Programming, AOP) 是一种编程思想，可以看成是对 OOP 的一种补充。它的作用在于实现关注点的分离。

传统做法里，有一些日志、统计、安全之类的模块，它的触发操作是分散在系统各处的。比如编写了一个统计模块，你必须在登陆的各个环节主动调用它，才能够去统计登陆情况。

面向切面编程的思想是，把这些分散在系统各处的触发点(AOP 中称之为切点)聚集起来形成一个切面，在切面里统一处理你所关注的操作。

举例来说，登陆操作分散在系统各处，你可以编写一个专门用来上报登陆统计的切面类，在这个切面类中定义你所关心的各种事件及对应的处理操作，比如登陆成功后上报 XXX 数据，登陆失败后上报 XXX 数据。

无需在系统各处来触发登陆统计，统计相关的内容都写在了切面类里面，原有代码不会受到任何干扰，关注点也聚焦起来了。

### 最小侵入性

Spring 框架极力避免侵入你的代码。这意味着你不需要继承各种接口，就可以使用 DI, AOP 等技术。


## 一些需要提前知道的事情

### bean

Bean 这个名字让人摸不着头脑，我理解它表示符合一些约定的 Java 类对象。比如：
- 这个类是 public 的；
- 有一个无参数的构造函数；
- 属性是 private 的，必须通过 get 和 set 方法进行访问；
- 实现 java.io.serializable 接口，支持序列化/反序列化；

与之对应概念是 POJO(Plain Old Java Object), 也就是普通的 Java 对象。

如果你希望某个类能够被 Spring 框架所使用，那么这个类的类对象应该是个 bean. 因为 Spring 必须通过一些约定来访问对象。

### 容器

在 Spring 框架中，bean 生存于 Spring 容器中。Spring 容器负责创建 bean，装配它们，配置它们并管理它们的整个生存周期：

![]( {{site.url}}/asset/spring-container.png )

我没搞清楚 Spring 里能不能同时存在多个容器，这个以后再来讨论。

### 应用上下文

应用上下文是 Spring 容器的一种实现。你可以从 Java 配置类, XML 配置里加载出应用上下文。加载完毕之后，配置里记录的 bean 将被添加到容器里，Spring 会根据配置来对 bean 进行装配。

### Spring 模块概览

下图列出了 Spring 所包含的主要模块。这里不过多介绍，以后会逐步介绍各个模块。

![]( {{site.url}}/asset/spring-modules.png )


## 装配 beans

这一小节详细介绍一下之前提到的依赖注入技术，来看看 Spring 怎么帮你完成 bean 的装配。

Spring 提供了三种装配机制：
- 在 XML 中进行显式配置；
- 在 Java 中进行显式配置；
- 隐式的 bean 发现机制和自动化装配；

### 自动化装配的例子

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

@Configuration
@ComponentScan
public class CDPlayerConfig {
}

public class Main {
    public static void main(String[] args) {
        // 加载应用上下文
        ApplicationContext context = new AnnotationConfigApplicationContext(com.dada.login.config.CDPlayerConfig.class);

        // 从应用上下文中获取 bean
        MediaPlayer player = context.getBean(CDPlayer.class);
    }
}
```

注意：
- `@Component` 注解告诉 Spring, 它所修饰的类是一个 JavaBean, 这样 Spring 在扫描的时候就会自动创建出这个 bean;
- `@Autowired` 注解告诉 Spring, 在创建 CDPlayer 的时候，为它绑定一个实现了 CompactDisc 的 bean;
- `@Configutation` 注解告诉 Spring, 它所修饰的类是一个 Java 配置类，其中包含了装配的细节；
- `@ComponentScan` 注解告诉 Spring, 启用组件发现功能，这样 Spring 才会开始扫描 `@Component` 注解；
- 最后，我们通过 AnnotationConfigApplicationContext 把配置类加载到应用上下文中；

### 通过 Java 配置类装配的例子

上一小节完全通过自动发现来装配 bean, 因此配置类里没有任何代码就完成了 bean 的创建和装配工作。

对于一些第三方库来说，你无法为其加上 `@Component` 注解，这个时候就可以在配置类里对其进行装配：

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

无需 `@Component`, `@Component` 注解，只需在配置类里编写 `@Bean` 修饰的方法，就可以告诉 Spring 来创建和装配这些 bean.

Java 配置类的好处是它比较灵活，你可以随意编写 `@Bean` 方法中创建和装配 bean 的方法。

### 通过 XML 进行装配的例子

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

### 环境和 profile

在开发软件的时候，一个很麻烦的事情就是把程序从一个环境迁移到另一个环境。比如数据库的配置，我们可能希望在开发环境中使用一个嵌入式的数据库、在生产环境中又期望通过 JDBC 从容器中获取一个 DataSource、在测试环境中则使用测试用的数据库。

如此一来，根据使用的数据库的不同，我们需要用不同的策略来生成一个 bean，Spring 提供了 profile 功能来支持这一点。你可以通过 profile 告知 Spring 当前的 bean 适用于哪种环境。之后再按照不同需求来激活对应的 profile。

例子：

```java
@Configuration
public class DataSourceConfig {

    @Bean(destroyMethod = "shutdown")
    @Profile("dev")
    public DataSource embeddedDataSource() {
        // ...
    }

    @Bean
    @Profile("prod")
    public DataSource jndiDataSource() {
        // ...
    }
}
```

接着，你得告诉 Spring 具体要激活哪种 profile, 激活过程依赖于不同的环境参数，这里不多介绍，用到的时候看书吧。

### bean 的作用域

之前我们创建的 bean 都是单例形式的，也就是说不管注入多少次，每次所注入的 bean 都是同一个对象。

Spring 定义了多种作用域，可以基于这些作用域来创建 bean，包括：
- 单例 (Singleton): 在整个应用中，只创建 bean 的一个实例；
- 原型 (Prototype): 每次注入或者通过 Spring 应用上下文获取的时候，都会创建一个新的 bean 实例；
- 会话 (Session) : 在 Web 应用中，为每个会话创建一个 bean 实例；
- 请求 (Request) : 在 Web 应用中，为每个请求创建一个 bean 实例；

默认情况的作用域是 Singleton ，你可以通过 `@Scope` 注解来指定其它的作用域：

```java
// 使用组件扫描
@Component
@Scope(ConfigurableBeanFactory.SCOPE_PROTOTYPE)
public class Notepat {
    // ...
}

// 在 JavaConfig 配置类里配置
@Bean
@Scope(ConfigurableBeanFactory.SCOPE_PROTOTYPE)
public Notepad notepad() {
    return new Notepad();
}
```

### 运行时注入

我们可以在装配 bean 时，为 bean 注入一些值：

```java
@Bean
public CompactDisc sgtPeppers() {
    return new BlankDisc("Sgt. Pepper's Lonely Hearts Club Band", "The Beatles");
}
```

问题在于，有时候我们希望这些值是在运行时才确定的，而不是硬编码在 JavaConfig 或 XML 中。Spring 提供了两种在运行时求值的方式：
- 属性占位符(Property placeholder);
- Spring 表达式语言(SpEL);

先看看简单的属性占位符：

```java
@Configuration
@PropertySource("classpath:/com/dada/learning/app.properties")
public class ExpressiveConfig {

    @Autowired
    Environment env;

    @Bean
    public BlankDisc disc() {
        return new BlankDisc(env.getProperty("disc.title"), env.getProperty("disc.artist"));
    }
}
```

我们在 `app.properties` 文件里配置了两个属性，内容大致如下：

```
disc.title = Sgt. Peppers Lonely Hearts Club Band
disc.artist = The Beatles
```

`@PropertySource` 注解引用了 `app.properties` 文件，它会把属性文件里的内容加载到 Spring 的 Environment 中，随后我们可以在配置代码里通过 `env.getProperty()` 方法来获取这些属性。

### 使用 Spring 表达式语言进行运行时注入

除了使用属性占位符，你还可以使用 SpEL(Spring Expression Language, Spring 表达式语言)来进行装配。相较于属性占位符的方式，它更加灵活和强大。

SpEL 拥有很多特性，包括：
- 使用 bean 的 ID 来引用 bean;
- 调用方法和访问对象的属性；
- 对值进行算数、关系和逻辑运算；
- 正则表达式匹配；
- 集合操作；

SpEL 表达式需要放在 `#{...}` 中，表达式体并不难理解：

```java
// 最简单的表达式
#{1}

// 获取 ID 为 sgtPeppers 的 bean 的 artist 属性
#{sgtPeppers.artist}

// 像占位符那样引用系统属性
#{systemProperties['dist.title']}

public BlankDisc(@Value("#{systemProperties['dist.title']}"),
                 @Value("#{systemProperties['dist.artist']}"))

// 调用 bean 的方法
#{artistSelector.selectArtist().toUpperCase()}

// 使用 ? 操作符，当 selectArtist() 返回 null 时，toUpperCase() 不会被调用
#{artistSelector.selectArtist()?.toUpperCase()}

// 访问类作用域的方法和常量
#{T(java.lang.Math).random()}

// 使用运算符
#{2 * T(Java.lang.Math).PI * circle.radius}
#{counter.total == 100}
#{scoreboard.score > 1000 ? "Winner!" : "Loser"}

// 如果 disc.title 为 null, 则为其赋一个值
#{disc.title ?: 'Rattle and Hum'}

// 正则表达式
#{admin.email matches '[a-ZA-Z0-9._%+-]+@[a-zA-Z0-9.-]+\\.com'}

// 获取集合元素, [] 运算符
#{jukebox.songs[4].title}

// 过滤集合元素, .?[] 运算符
#{jukebox.songs.?[artist eq 'Aerosmith']}
```

