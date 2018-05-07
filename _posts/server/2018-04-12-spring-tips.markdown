---
layout: post
title:  "Spring 小知识"
date:   2018-04-12 13:29:30 +0800
categories: server
---

* TOC
{:toc}


## 应用上下文 (ApplicationContext) 之间的隔离

Spring 中的 Bean 需要存在于容器中，由容器去管理 Bean 的生命周期。

我没太区分出来容器和应用上下文的区别, 这里就把它们当做是同一个概念，即 Bean 存在于 ApplicationContext 中，由 ApplicationContext 管理它的生命周期。

要使用 Bean, 首先需要创建 ApplicationContext 实例。向 ApplicationContext 的构造函数传入配置类或 xml 之后，ApplicationContext 就会根据配置去创建出对应的 Bean, 典型地，在程序启动时会有如下代码：

```java
// 通过配置类构造 ApplicationContext
ApplicationContext applicationContext = new AnnotationConfigApplicationContext(SpringConfig.class);
```

这就带来一个疑惑，如果创建了两个 ApplicationContext 实例，那它们中的 Bean 能够互通吗？

```java
// 通过配置类加载上下文
ApplicationContext cdContext = new AnnotationConfigApplicationContext(CDConfig.class);
ApplicationContext playerContext = new AnnotationConfigApplicationContext(PlayerConfig.class);

// 上下文加载完毕后，就可以获取容器中的 bean
MusicPlayer player = playerContext.getBean(MusicPlayer.class);
player.play();
```

试了一下是不能的，会报错提示 `NoSuchBeanDefinitionException`, 因为 cd 和 player 在不同的 Application 中，导致 player 找不到 cd.

然而 ApplicationContext 之间是可以有继承关系的，典型的例子是 Spring MVC, 它的 ServletInitializer 将创建两个 ApplicationContext:

```java
public class ServletInitializer extends AbstractAnnotationConfigDispatcherServletInitializer {
    @Override
    protected Class<?>[] getRootConfigClasses() {
        return new Class<?>[] { RepositoryConfig.class, SecurityConfig.class };
    }

    @Override
    protected Class<?>[] getServletConfigClasses() {
        return new Class<?>[] { WebConfig.class };
    }

    @Override
    protected String[] getServletMappings() {
        return new String[] { "/" };
    }
}
```

`AbstractAnnotationConfigDispatcherServletInitializer` 能够帮助你创建出 **一个** DispatcherServlet. 你需要提供两个配置类，分别用来构建 Root ApplicationContext 和 Servlet ApplicationContext, 其中 Servlet ApplicationContext 继承于 Root ApplicationContext.

这个有啥用呢？主要用在一个 Web App 中包含多个 DispatcherServlet 的情况，你可以把一些公共的 Bean 放在 Root ApplicationContext 里面，Servlet ApplicationContext 里只包含哪些与当前 DispatcherServlet 有关的 Bean. Root 中的 Bean 将被多个 Servlet 共享，比如 Repository, Security 这些基础设施，中间件之类的东西。

不过我猜大部分场景下一个 Web App 里只要有一个 DispatcherServlet 就可以了。如果需要多个 DispatcherServlet, 就不能用 `AbstractAnnotationConfigDispatcherServletInitializer` 了，得用别的办法。

参考自：
- [getServletConfigClasses() vs getRootConfigClasses()](https://stackoverflow.com/questions/35258758/getservletconfigclasses-vs-getrootconfigclasses-when-extending-abstractannot)
- [Why use Spring ApplicationContext hierarchies?](https://stackoverflow.com/questions/5132604/why-use-spring-applicationcontext-hierarchies/5132637#5132637)


## RedisTemplate 中使用 GenericJackson2JsonRedisSerializer 的一些细节

Spring 中使用缓存，需要进行如下配置:

```java
@Configuration
@ComponentScan
@EnableCaching
public class CacheConfiguration {

    /**
     * RedisConnectionFactory 这个 Bean 提供了连接到 Redis 数据库的能力
     */
    @Bean
    public RedisConnectionFactory redisConnectionFactory() {
        JedisConnectionFactory jedisConnectionFactory = new JedisConnectionFactory();
        jedisConnectionFactory.setHostName("localhost");
        jedisConnectionFactory.setPort(6379);
        return jedisConnectionFactory;
    }

    @Bean(name = "tempCacheRedisTemplate")
    public RedisTemplate<String, String> tempCacheRedisTemplate(RedisConnectionFactory redisConnectionFactory) {
        RedisTemplate<String, String> template = new RedisTemplate<String, String>();
        template.setConnectionFactory(redisConnectionFactory);

        template.setKeySerializer(new StringRedisSerializer());

        ObjectMapper objectMapper = new ObjectMapper();
        objectMapper.enableDefaultTyping(ObjectMapper.DefaultTyping.NON_FINAL, JsonTypeInfo.As.PROPERTY);
        objectMapper.setDateFormat(new SimpleDateFormat("yyyy-MM-dd HH:mm:ss"));
        objectMapper.registerModule(new JavaTimeModule());
        template.setValueSerializer(new GenericJackson2JsonRedisSerializer(objectMapper));

        template.afterPropertiesSet();
        return template;
    }

    @Bean(name = "tempCacheManager")
    @Primary
    public CacheManager tempCacheManager(@Qualifier("tempCacheRedisTemplate") RedisTemplate redisTemplate) {
        RedisCacheManager redisCacheManager = new RedisCacheManager(redisTemplate);
        redisCacheManager.setDefaultExpiration(60*60);
        redisCacheManager.setTransactionAware(true);
        return redisCacheManager;
    }
}
```

配置完之后才可以使用 `@Cacheable`, `@CachePut`, `@CacheEvict` 等注解。

值得注意的是配置 RedisTemplate 的一段代码：

```java
ObjectMapper objectMapper = new ObjectMapper();

// 把 Java 类型信息保存到生成的 Json 里面，以便反序列化时 Jackson 能够得到类型
objectMapper.enableDefaultTyping(ObjectMapper.DefaultTyping.NON_FINAL, JsonTypeInfo.As.PROPERTY);

// 设置日期格式
objectMapper.setDateFormat(new SimpleDateFormat("yyyy-MM-dd HH:mm:ss"));

// JavaTimeModule 这个模块可以处理 ZonedDateTime 这种 Java8 里的时间类型
objectMapper.registerModule(new JavaTimeModule());

// 用 ObjectMapper 来初始化 GenericJackson2JsonRedisSerializer
template.setValueSerializer(new GenericJackson2JsonRedisSerializer(objectMapper));
```

之前由于没有配置 `enableDefaultTyping`, 导致对象可以序列化成 json 后保存到 redis 里，但取不出来，因为 json 里没有包含类型信息，jackson 无法完成反序列化；

另外由于没有配置 `JavaTimeModule`, 导致 ZonedDateTime 序列化后的 json 很大(包含了详细的成员信息)。

