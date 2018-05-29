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


## Redis + EhCache 二级缓存

遇到一个业务情况，有一个配置表，查询次数非常多，所以想缓存起来，一开始想着缓存到 Redis 里就完了，后来觉得既然这么频繁，而且还是配置信息，不经常变动，缓存到内存里更好一点。

之前乐观地以为 Spring 中提供的 CompositeCacheManager 可以完成二级缓存的配置，后来试验了一下发现它只能完成在一级缓存里没有 Cache 时，去二级缓存里找 Cache 的功能。这里查找的只是 Cache, 不是 Cache 里的数据。

而我预期的情况是，如果二级缓存查到了，就更新一级缓存的内容；如果二级缓存也没有查到，但从数据库里查到了，就把数据都更新到一级和二级缓存里面。

要完成上述需求，得自己定义 CacheManager 和 Cache, 先来看看原始的接口定义是啥意思：

```java
/**
 * Cache 接口定义
 * 英语不好的董哒哒把注释翻译了一遍
 */
public interface Cache {

    /**
     * Return the cache name.
     */
    String getName();

    /**
     * 返回原始的缓存对象，比如 EhCache
     * Return the underlying native cache provider.
     */
    Object getNativeCache();

    /**
     * 根据 key 获取对应的值
     * 如果 key 没有对应的值，则返回 null,
     * 如果 key 有值，但这个值是 null, 那么 null 会被记录到 ValueWrapper 里返回
     */
    @Nullable
    ValueWrapper get(Object key);

    /**
     * 根据 key 获取对应的值
     * 增加了类型指定
     * 不区分值没有找到 和 找到的值就是 null 的情况，这两种情况下都会返回 null
     */
    @Nullable
    <T> T get(Object key, @Nullable Class<T> type);

    /**
     * 根据 key 获取对应的值
     * 如果获取不到，将通过 valueLoader.call() 调用 @Cacheable 注解的方法(也就是访问数据库)
     * 描述了 "如果缓存里有数据，直接取；如果没有，从数据库里取、保存到缓存里、然后返回" 这一规则。
     * 实现者需要保证 valueLoader.call() 调用时的线程安全问题，避免缓存失效下多个线程都调用 valueLoader.call() 访问数据库的问题。换句话说，缓存失效情况下，虽然有多个线程调用，但只应当发起一次 valueLoader.call()
     */
    @Nullable
    <T> T get(Object key, Callable<T> valueLoader);

    /**
     * 把值写入到缓存中
     * 当 @Cacheable 未命中, @CachePut 时被调用
     * 写入缓存完毕后，会调用注解修饰的方法(也就是数据库语句)把内容写入到数据库
     */
    void put(Object key, @Nullable Object value);

    /**
     * 当值不存在时，把值写入到缓存中；当值存在时，返回原有的值
     * 这一操作应该是原子的
     * 等价于
     * <code><pre>
     * Object existingValue = cache.get(key);
     * if (existingValue == null) {
     *     cache.put(key, value);
     *     return null;
     * } else {
     *     return existingValue;
     * }
     * </code></pre>
     */
    @Nullable
    ValueWrapper putIfAbsent(Object key, @Nullable Object value);

    /**
     * 删除 key-value 对
     */
    void evict(Object key);

    /**
     * Remove all mappings from the cache.
     */
    void clear();
}
```

看起来比较简单，我需要按照如下规则来实现 Cache 接口：
- `get(Object key)`, `get(Object key, Class<T> type)` 这两个方法比较简单，检查一级缓存有没有、再检查二级缓存有没有，没有的话返回 null 就可以了；
- `get(Object key, Callable<T> valueLoader)` 这个方法需要实现 "如果缓存里有数据，直接取；如果没有，从数据库里取、保存到缓存里、然后返回" 这一逻辑;
- `put(Object key, Object value)` 把值写入到缓存里，得注意线程安全问题;
- `putIfAbsent(Object key, Object value)` 值不存在的时候，写缓存，值存在的时候，返回原有的值;
- `evict(Object key)` 删除缓存里对应的值;
- `clear()` 清缓存;

Spring 提供了一个 AbstractValueAdaptingCache 类，它实现了 Cache 接口的一部分方法，主要是封装了 ValueWrapper 的那层转换。


再来看一下 CacheManager 的接口定义：

```java
public interface CacheManager {

    /**
     * Return the cache associated with the given name.
     * @param name the cache identifier (must not be {@code null})
     * @return the associated cache, or {@code null} if none found
     */
    @Nullable
    Cache getCache(String name);

    /**
     * Return a collection of the cache names known by this manager.
     * @return the names of all caches known by the cache manager
     */
    Collection<String> getCacheNames();
}
```

很简单，返回给定名称的 Cache 对象就可以了。不过 Spring 还提供了一个 AbstractCacheManager 类，实现了一些基本功能：

```java
public abstract class AbstractCacheManager implements CacheManager, InitializingBean {

    private final ConcurrentMap<String, Cache> cacheMap = new ConcurrentHashMap<>(16);

    private volatile Set<String> cacheNames = Collections.emptySet();


    // InitializingBean 接口中的方法，Spring 会在初始化 Bean 之后，自动调用这个方法
    // 在这个方法里对 Cache 进行初始化
    @Override
    public void afterPropertiesSet() {
        initializeCaches();
    }

    /**
     * 初始化 caches 的静态配置
     * 把一堆 cacheName 保存到了 cacheNames 里面
     */
    public void initializeCaches() {
        Collection<? extends Cache> caches = loadCaches();

        synchronized (this.cacheMap) {
            this.cacheNames = Collections.emptySet();
            this.cacheMap.clear();
            Set<String> cacheNames = new LinkedHashSet<>(caches.size());
            for (Cache cache : caches) {
                String name = cache.getName();
                this.cacheMap.put(name, decorateCache(cache));
                cacheNames.add(name);
            }
            this.cacheNames = Collections.unmodifiableSet(cacheNames);
        }
    }

    /**
     * 加载初始 caches
     * 需要子类实现
     */
    protected abstract Collection<? extends Cache> loadCaches();


    // 延迟初始化一个 Cache
    // 先看有没有，没有的话通过 getMissingCache 创建
    @Override
    @Nullable
    public Cache getCache(String name) {
        Cache cache = this.cacheMap.get(name);
        if (cache != null) {
            return cache;
        }
        else {
            // Fully synchronize now for missing cache creation...
            synchronized (this.cacheMap) {
                cache = this.cacheMap.get(name);
                if (cache == null) {
                    cache = getMissingCache(name);
                    if (cache != null) {
                        cache = decorateCache(cache);
                        this.cacheMap.put(name, cache);
                        updateCacheNames(name);
                    }
                }
                return cache;
            }
        }
    }

    @Override
    public Collection<String> getCacheNames() {
        return this.cacheNames;
    }


    // Common cache initialization delegates for subclasses

    /**
     * 检查 Cache 是不是已经有了，不会引起 Cache 初始化
     */
    @Nullable
    protected final Cache lookupCache(String name) {
        return this.cacheMap.get(name);
    }

    /**
     * 动态新增一个 Cache
     */
    @Deprecated
    protected final void addCache(Cache cache) {
        String name = cache.getName();
        synchronized (this.cacheMap) {
            if (this.cacheMap.put(name, decorateCache(cache)) == null) {
                updateCacheNames(name);
            }
        }
    }

    /**
     * 在 cacheNames 里新增一个 cacheName
     */
    private void updateCacheNames(String name) {
        Set<String> cacheNames = new LinkedHashSet<>(this.cacheNames.size() + 1);
        cacheNames.addAll(this.cacheNames);
        cacheNames.add(name);
        this.cacheNames = Collections.unmodifiableSet(cacheNames);
    }


    // 以下方法用于被复写

    /**
     * 装饰一个 Cache 对象，可以在 Cache 上做一些额外功能
     */
    protected Cache decorateCache(Cache cache) {
        return cache;
    }

    /**
     * cache 不存在的时候提供一个新的
     * 主要是为了应对在运行时根据条件动态添加新 cache 的情况
     * 相比于 addCache 方法来说不需要用户自己判断是否存在
     */
    @Nullable
    protected Cache getMissingCache(String name) {
        return null;
    }

}
```

看起来比较简单，实现 AbstractCacheManager 时只需要实现 `loadCaches()` 和 `getMissingCache` 方法即可。不过 Spring 还提供了一个实现 `AbstractTransactionSupportingCacheManager`, 这个类在 `AbstractCacheManager` 的基础上增加了事务的能力:

```java
public abstract class AbstractTransactionSupportingCacheManager extends AbstractCacheManager {
    private boolean transactionAware = false;

    public AbstractTransactionSupportingCacheManager() {
    }

    public void setTransactionAware(boolean transactionAware) {
        this.transactionAware = transactionAware;
    }

    public boolean isTransactionAware() {
        return this.transactionAware;
    }

    protected Cache decorateCache(Cache cache) {
        return (Cache)(this.isTransactionAware() ? new TransactionAwareCacheDecorator(cache) : cache);
    }
}
```

可以看到它复写了 AbstractCacheManager 的 decorateCache 方法，用一个 TransactionAwareCacheDecorator 包装了原有的 Cache 对象；


## 对象内部调用时，AOP 注解失效

在对象内部调用自己的另一个包含注解的方法，该方法的注解会失效，原因是没有走代理。

解决方法是获取 Bean 之后再调用：

```java
public class TicketService{
    //买火车票
    @Transactional
    public void buyTrainTicket(Ticket ticket){
        System.out.println("买到了火车票");
        try {
            //通过代理对象去调用sendMessage()方法          
            SpringUtil.getBean(this.getClass()).sendMessage();
        } catch (Exception  e) {
            logger.warn("发送消息异常");
        }
    }

    @Transactional
    public void sendMessage(){
        System.out.println("消息存入数据库");
        System.out.println("执行发送消息动作");
    }
}
```

参考自 [这篇文章](https://blog.csdn.net/u012373815/article/details/77345655)