---
layout: post
title:  "Spring Cache 框架分析"
date:   2018-05-08 14:07:30 +0800
categories: server
---

* TOC
{:toc}

Spring 的 Cache 框架不包含具体实现，而是提供一个框架，可以把一些常见的 Cache 实现(例如 Redis, EhCache, Guava)接入到 Spring 当中。

因为允许接入的实现众多，因此 Cache 框架抽象出来的能力是比较通用的，在调用者看来，它就像一个 key-value 形式的 map, 只提供了 get/put/evict/clear 这些通用方法。

## Cache 和 CacheManager 接口

Spring 提供了上述两个接口来管理缓存。其中：
- Cache : 代表缓存实例，一个 Cache 对象维护一组 key-value;
- CacheManager : 缓存管理器，管理一组 Cache;

以下是直接使用 CacheManager 和 Cache 的例子：

```java
@Repository
public class UserRepo {

    private static final String USER_CACHE_NAME = "user";

    @Autowired
    UserDAO userDAO;

    @Autowired
    CacheManager cacheManager;

    Cache userCache = null;

    @PostConstruct
    private void init() {
        userCache = cacheManager.getCache(USER_CACHE_NAME);
    }

    public void insert(UserDTO userDTO) {
        int affectedRows =userDAO.insert(userDTO);
        if (affectedRows <= 0) {
            return;
        }

        userCache.put(userDTO.getId(), userDTO);
    }

    private UserDTO selectById(Long id) {
        UserDTO userDTO = userCache.get(id, UserDTO.class);
        if (userDTO != null) {
            return userDTO;
        }

        return userDAO.selectById(id);
    }
}
```


## 注解支持

直接操作 Cache 接口有点麻烦，Spring 还提供了注解形式来调用 Cache 接口的方法：
- Cacheable: 指定一个方法或者类中的所有方法，其返回值可以被缓存起来；
- CachePut: 指定一个方法或者类中的所有方法，其返回值可以被缓存起来；
- CacheEvict: 指定一个方法或者类中的所有方法，该方法调用时将清除缓存中的一部分或者全部条目；
- Caching: 集成一组注解，可以包含多个 `Cacheable`, `CachePut`, `CacheEvict` 注解，这些注解将同时生效；

Cacheable 的语义为：
- 检查 Cache 里有没有，有的话直接返回；
- Cache 里没有，则调用方法，如果成功，将结果放到缓存；

CachePut 的语义为：
- 调用方法，如果成功，将结果放到缓存;

除 Caching 外，上述注解都包含如下字段：
- cacheManager: 定位 CacheManager 实例；
- cacheNames, cacheResolver: 定位 Cache 实例；
- key, keyGenerator: 计算 key;
- condition: 在方法调用 **前** 判断是否访问缓存；
- unless: 在方法调用 **后** 判断是否访问缓存；(CacheEvict 不包含此字段)

不同注解的 condition/unless 字段含义不太一样：
- Cacheable:
    - condition: 为 true 表示方法调用前先读缓存，方法调用后写入缓存; 为 false 表示忽略缓存，直接调用方法，调用后不写缓存；
    - unless: 为 true 表示方法调用后写缓存; false 表示不写;
- CachePut:
    - condition: 为 true 表示方法调用后写缓存; false 表示不写；
    - unless: 为 true 表示方法调用后写缓存; false 表示不写；
- CacheEvict:
    - contition: 为 true 表示方法调用后清除该缓存; false 表示不清;

除了使用 SpEl 表达式获取参数值，Spring Cache 还提供了一个 root 对象，可以访问 对象、参数、函数 等内容：
- root.methodName: 方法名;
- root.method: 方法;
- root.targetClass: 目标对象类;
- root.target: 目标对象;
- root.args: 方法的参数列表;
- root.caches: 当前方法调用所使用的缓存实例(比如 `@Cacheable(cacheNames = {"cache1", "cache2"})` 中使用了两个缓存实例);

除了 root 对象，你还可以直接访问方法参数，比如：

```java
@Cacheable(cacheNames = "user", key = "#user.id", condition = "#root.target.canCache()")
UserDTO select(UserDTO user);
```

对于 unless 而言，还可以通过 result 对象来访问方法调用的结果：

```java
@CachePut(cacheNames = "user", key = "#user.id", unless = "#result.isVip()")
UserDTO insert(UserDTO user);
```

除了上述注解外，Spring 还提供了另外几个与缓存有关的注解：
- CacheConfig : 指定一个类中的所有方法，在未指定 cacheManager, cacheNames, cacheResolver, keyGenerator 时使用默认配置；
- EnableCaching : 用在配置类上，表示开启 Spring Cache 机制；


## 一些关于注解的 Tips

### 一些复杂的 condition 可以放到类方法中，然后通过 #root.target 来调用：

之前已经提到过的技巧：

```java
@Cacheable(cacheNames = "user", key = "#user.id", condition = "#root.target.canCache()")
UserDTO select(UserDTO user);
```

如果写 SpEl 太麻烦，可以考虑单独搞一个方法来做判断；

### 把 Cacheable 变为 CacheGet

有时候不需要 Cacheable 在缓存未命中的情况下保存方法返回值。只需要检查缓存中有没有，有就返回，没有就调方法。

这种场景应该比较少见，一个可能不恰当的例子是，我想要提前缓存好最新的前 100 条数据，这时可以在 insert 的时候来缓存数据，在 select 的时候读取数据；

```java
// 在 insert 的时候提前缓存好发给用户的前 100 条消息
public int insert(MessageDTO message) {
    List<MessageDTO> userList = userCache.get(message.getUserId());
    userList.add(0, message);
    if (userList.size() > 100) {
        userList.remove(userList.size() - 1);
    }

    return userMessageDAO.insert(message);
}

// list 接口只从缓存里取消息，不要覆盖，避免 limit 等条件导致查出来的结果不足 100 条，破坏了我原先缓存好的 100 条消息
@Cacheable(cacheName = "user_messages", key = "#userId", unless="false")
public List<MessageDTO> list(Long userId, Long limit);
```

上述例子展现了一种 "预缓存" 的思路，而不是临时缓存。这种场景下可能要求 `@Cacheable` 不要破坏预缓存。解决办法就是设置 unless 为 false.
