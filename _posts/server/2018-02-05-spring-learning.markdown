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


## 高级装配

### 环境与 profile

在开发软件的时候，一个很麻烦的事情就是把程序从一个环境迁移到另一个环境。比如数据库的配置，我们可能希望在开发环境中使用一个嵌入式的数据库、在生产环境中又期望通过 JDBC 从容器中获取一个 DataSource、在测试环境中则使用测试用的数据库。

如此一来，根据使用的数据库的不同，我们需要用不同的策略来生成一个 bean，Spring 提供了 profile 功能来支持这一点。你可以通过 profile 告知 Spring 当前的 bean 适用于哪种环境。之后再按照不同需求来激活对应的 profile。

例如你可以编写如下两个 bean 来适应开发环境和生产环境：

```java
@Configuration
@Profile("dev")
public class DevelopmentProfileConfig {

    @Bean(destroyMethod="shutdown")
    public DataSource dataSource() {
        return new EmbeddedDatabaseBuilder()
            .setType(EmbeddedDatabaseType.H2)
            .addScript("classpath:schema.sql")
            .addScript("classpath:test-dada.sql")
            .build();
    }
}
```

```java
@Configuration
@Profile("prod")
public class ProductionProfileConfig {

    @Bean
    public DataSource dataSource() {
        JndiObjectFactoryBean jndiObjectFactoryBean = 
            new JndiObjectFactoryBean();
        jndiObjectFactoryBean.setJndiName("jdbc/myDS");
        jndiObjectFactoryBean.setResourceRef(true);
        jndiObjectFactoryBean.setProxyInterface(
            javax.sql.DataSource.class);
        return (DataSource)jndiObjectFactoryBean.getObject();
    }
}
```

上述代码中使用了 `@Profile` 注解来修饰不同的配置类，你也可以用 `@Profile` 注解来修饰方法：

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

对应的，在 XML 配置中，你也可以使用 `<beans>` 元素的 profile 属性来设置。这一点就不介绍了，用到的时候再去看吧。

接下来，你需要告诉 Spring 应该激活哪个 profile. 这一点依赖于两个属性：`spring.profiles.active` 和 `spring.profiles.default`, Spring 会首先查找 `spring.profiles.active` 的设置，如果没有，就采用 `spring.profiles.default` 的设置。

要设置这两个属性，有一下几种不同的方式：
- 作为 DispatcherServlet 的初始化参数
- 作为 Web 应用的上下文参数；
- 作为 JNDI 条目；
- 作为环境变量；
- 作为 JVM 的系统属性；
- 在集成测试类上，使用 `@ActiveProfiles` 注解来设置； 

现在还不清楚 Servlet 相关的内容，因此先不介绍，以后用到再去查找。

### 条件化的 bean

有时候我们会希望在满足了某种条件的情况下才创建 bean, 比如只会在另外某个特定的 bean 也声明了之后才创建，或者只有在设置了某个环境变量的情况下才创建。

这种情况下可以通过 `@Conditional` 注解设置一个条件类，例如：

```java
@Bean
@Conditional(MagicExistCondition.class)
public MagicBean magicBean() {
    // ...
}
```

MagicExistCondition 类必须实现 Condition 接口：

```java
public interface Condition {
    boolean matches(ConditionContext ctxt, AnnotatedTypeMetadata metadata);
}
```

例如：

```java
public class MagicExistCondition implements Condition {
    public boolean matches(ConditionContext ctxt, AnnotatedTypedMetadata metadata) {
        // 检查环境变量中是否包含 magic 字段，从而决定是否创建 bean
        Environment env = ctxt.getEnvironment();
        return env.containsProperty("magic");
    }
}
```

除了获取环境变量，你还可以利用 ConditionContext 的其他方法，检查某个 bean 是否存在，检查资源，检查某些类是否存在，从而为 bean 是否创建提供依据。

AnnotatedTypedMetadata 则能够让你检查方法的注解。

### 处理自动装配的歧义性

之前我们已经讨论过，在自动装配时，如果有不止一个 bean 能够匹配结果，Spring 会无法装配，精确地说，此时 Spring 会抛出 NoUniqueBeanDefinitionException 异常。

解决上述问题的一个方法是设置首选(primary)bean：

```java
// 将优先使用 IceCream 作为甜点
@Component
@Primary
public class IceCream implements Dessert {
    // ...
}

@Component
public class Cookies implements Dessert {
    // ...
}
```

Primary bean 的问题在于，当有多个 Primary Bean 的时候，Spring 又会面临不知道该选谁的问题，这种情况下你可以使用 `@Qualifier` 注解来明确告知 Spring 此次要绑定的依赖是谁：

```java
@Autowired
@Qualifier("iceCream")
public void setDessert(Dessert dessert) {
    this.dessert = dessert;
}
```

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

XML 里也可以配置，这里不啰嗦了。

会话和请求作用域暂不描述，以后进行 Web 开发后再回来看这部分内容。

### 运行时值注入

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

类似的，在 XML 配置中也可以使用属性占位符：

```xml
<bean id="sgtPeppers"
      class="soundsystem.BlankDisc"
      c:_title="${disc.title}"
      c:_artist="${disc.artist" />
```

对于组件扫描和自动装配的机制，则可以使用 `@Value` 注解：

```java
public BlankDisc(@Value("${disc.title}") String title,
                 @Value("${disc.artist}") String artist) {
    //...
}
```

### 使用 Spring 表达式语言进行装配

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


## 面向切面的 Spring

就像简介中所展现的，切面能帮助我们模块化横切关注点。简而言之，横切关注点可以被描述为影响应用多处的功能。例如，安全就是一个横切关注点，应用中的许多方法都护涉及到安全规则。

过去，我们重用通用功能的常见技术是通过继承或委托。但是，如果在整个应用中都使用相同的基类，继承往往会导致一个脆弱的对象体系；而使用委托可能需要对委托对象进行复杂的调用。

![]( {{site.url}}/asset/spring-aspect.png )

### AOP 术语

![]( {{site.url}}/asset/spring-aspect-terminology.png )

- 通知(Advice): 切面所需要完成的工作被称为通知，通知定义了切面是什么以及何时使用。具体到代码中，通知可能是一些函数的集合，这些函数描述了切面所做的各种工作，以及该通知何时被调用，是在目标方法前、后、还是在执行成功后、抛出异常后这些时机。
- 连接点(Join point): 连接点指的是发出通知的时机。比如登陆成功后、下载失败后，满足这些条件的连接点可能有很多。
- 切点(Poincut): 并不是所有的连接点都值得发出通知，我们通过定义切点来告诉通知要匹配哪些连接点。
- 切面(Aspect): 切面是通知和切点的结合，它描述了在什么时候(登陆成功后)、在什么地方(在登陆函数执行结束的地方)、需要干什么(更新用户头像)。
- 引入(Introduction): 引入允许我们向现有的类添加新方法或属性。比如我们可以在无需修改现有类的情况下，引入一个 Auditable 通知类，使现有类具有新的行为和状态。
- 织入(Weaving): 织入也就是把切面应用到目标对象并创建新的代理对象的过程。切面在指定的连接点把通知织入到目标对象中。

Spring 并不是唯一的 AOP 框架，还有另一个著名的框架叫做 AspectJ。两者在实现上有一个重要的区别，Spring 是在运行期把切面织入到 bean 中；而 AspectJ 则是在编译期把切面织入到目标中的，这导致 Spring 不支持字段连接点，无法创建细粒度的通知，比如拦截对对象字段的修改，而且 Spring 不支持构造器连接点，这意味着我们无法在 bean 创建时应用通知。

Spring 使用代理类来包裹切面，代理类封装了目标类，并拦截了被通知方法的调用，然后执行切面逻辑，再把调用转发给真正的目标 bean。

![]( {{site.url}}/asset/spring-aspect-proxy-class.png )

### 通过切点来选择连接点

