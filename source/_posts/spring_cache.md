---
title: SpringCache
date: 2021-07-10 20:50:00
tags: Spring
categories: 缓存
comments: true
typora-root-url: ../
---

Spring3.1开始定义了 `org.springframework.cache.Cache` 和 `org.springframework.cache.CacheManager` 两个接口用以统一Spring缓存使用规范，我们可以通过 spring 的 cache 和 cacheManager 接口去扩展自定义的缓存框架，用以简化业务中的缓存开发。接下来我们讲讲怎么在spring boot中集成spring cache和扩展caffeine cache。

#### 概述

首先，让我们了解下spring cache的使用特点：

- 通过少量的 annotation 就可以使既有代码支持缓存
- 支持开箱即用 Out-Of-The-Box，即不用安装和部署额外第三方组件就可以使用缓存
- 支持 Spring Express Language，使用表达式描述我们想要缓存的 key 和 condition
- 支持自己定义 key 和自己定义缓存管理者，具有相当的灵活性和扩展性
- 支持多种本地缓存方案

然后我们了解下他的重要概念和缓存注解

| 概念/注解      | 作用                                                         |
| :------------- | :----------------------------------------------------------- |
| @Cacheable     | 主要针对方法配置，能够根据方法的请求参数对其结果进行缓存     |
| @CacheEvict    | 清空缓存                                                     |
| @CachePut      | 保证方法被调用，又希望结果被缓存。                           |
| @EnableCaching | 开启基于注解的缓存                                           |
| Cache          | 缓存接口，定义缓存操作。实现有：RedisCache、EhCacheCache、ConcurrentMapCache等 |
| CacheManager   | 缓存管理器，管理各种缓存(Cache)组件                          |
| keyGenerator   | 缓存数据时key生成策略                                        |
| serialize      | 缓存数据时value序列化策略                                    |

了解了上面的特点，接下来我们具体进行配置和开发~

#### 配置

假设我们已经有了具体的springboot项目，现在进行maven配置

```xml
<dependency>
  <groupId>com.github.ben-manes.caffeine</groupId>
  <artifactId>caffeine</artifactId>
  <version>2.8.6</version>
</dependency>
<dependency>
  <groupId>org.springframework.boot</groupId>
  <artifactId>spring-boot-starter-cache</artifactId>
</dependency>
```

接下来就是创建application启动类、controller、service、domain等类，这些这里省略，直接讲讲需要注意的地方。

1、application 启动类需要增加支持缓存的注解`@EnableCaching`

```java
@SpringBootApplication
public class CacheApplication {
   public static void main(String[] args) {
       SpringApplication.run(CacheApplication.class, args);
  }
}
```

2、JDK8可以支持caffeine的2.x及以下版本,如果使用使用caffeine3.x则需要JDK11，本例中使用JDK8所以使用的 caffeine-2.8.6版本

3、本例中可以同时实现：springCache缓存，caffeine缓存，spring-caffeine组合缓存。使用的时候使用Resource("xx")区分具体实现

#### 示例代码

这里省略domain、controller，具体说说其中涉及的具体实现。首先，是service接口：

```java
public interface CacheService {

   User save(User user);

   void remove(Long userId);

   void removeAll();

   User findOne(User user);

   User findOne(Long userId);

   boolean update(User user);
}
```

然后是具体实现，这里有三套实现，分别是：springCache缓存，caffeine缓存，spring-caffeine组合缓存1.springCache缓存，使用的springCache原生缓存，只需依赖`spring-boot-starter-cache`即可开箱即用。

```java
/**
* springCache
*
* @author Tim
* @date 2021-09-14
*/
@Slf4j
@Primary
@Service("springCache")
public class SpringCacheImpl implements CacheService {

   @Override
   @Cacheable(value = "cache:user" /**key = "#user.userID" 不填key缺省为全属性拼接*/)
   public User save(User user) {
       log.info("Service保存用户:{}", JSONUtil.toJsonStr(user));
       user.setAge(27);
       return user;
  }

   @Override
   @CacheEvict(value = "cache:user", key = "#userId")
   public void remove(Long userId) {
       log.info("Service删除用户ID:{}", userId);
       // 此处省略走DB删除
  }

   @Override
   @CacheEvict(value = "cache:user", allEntries = true)
   public void removeAll() {
       log.info("Service删除所有用户缓存");
  }

   @Override
   @Cacheable(value = "cache:user", key = "#user.username")
   public User findOne(User user) {
       log.info("Service查询指定用户{}缓存!", user.getUsername());
       return new User(user.getUserID(), user.getUsername(), user.getPassword(), user.getAge()+1);
  }

   @Override
   @Cacheable(value = "cache:user", key = "#userId")
   public User findOne(Long userId) {
       log.info("Service查询指定用户{}缓存!", userId);
       return queryDB(userId);
  }

   @Override
   @CachePut(value = "cache:user", key = "#user.userID")
   public boolean update(User user) {
       log.info("Service更新指定用户{}缓存!", user.getUserID());
       user.setAge(user.getAge()+1);
       return true;
  }

   public User queryDB(Long userId) {
       log.info("Service查询指定用户{}DB数据!", userId);
       return new User(userId, "test", "test", 0);
  }
}
```

2.caffeine缓存，spring-context支持的、使用caffeine实现的本地缓存方案，官方将其与市面上的其他缓存做了对比，性能极高。

```java
/**
* caffeineCache
*
* @author Tim
* @date 2021-09-14
*/
@Slf4j
@Service("caffeineCache")
public class CaffeineCacheImpl implements CacheService {

   public static final String USER_CACHE_NAME = "USER_INFO";
   public static final String USER_CACHE_KEY = "USER_KEY";

   Cache cache;

   @Autowired
   private CacheManager cacheManager;

   @PostConstruct
   public void postConstruct() {
       cache = cacheManager.getCache(USER_CACHE_NAME);
  }

   @Override
   public User save(User user) {
       cache.put(USER_CACHE_KEY + user.getUserID(), user);
       return user;
  }

   @Override
   public void remove(Long userId) {
       log.info("caffeine删除用户:{}", userId);
       cache.evict(USER_CACHE_KEY + userId);
  }

   @Override
   public void removeAll() {
       cache.clear();
  }

   @Override
   public User findOne(User user) {
       User u = cache.get(USER_CACHE_KEY + user.getUserID(), User.class);
       log.info("caffeine1查询用户{}:{}", user.getUserID(), JSONUtil.toJsonStr(u));
       return u;
  }

   /**
    * 请求获取User，首先请求缓存(只有一层Caffeine本地缓存)，如果没有找到则调用localValueLoader获取DB的数据
    *
    * @param userId
    * @return
    */
   @Override
   public User findOne(Long userId) {
       log.info("caffeine2查询用户:{}", userId);
       User user = cache.get(USER_CACHE_KEY + userId, new localValueLoader<>(userId));
       return user;
  }

   @Override
   public boolean update(User user) {
       return true;
  }
}
```

3.spring-caffeine组合缓存，是基于springCache注解配置，集成自定义的cacheManager(caffeineCacheManager)的一种组合实现。

```java
/**
* compositeCache
*
* @author Tim
* @date 2021-09-14
*/
@Slf4j
@Service("compositeCache")
public class CompositeCacheImpl implements CacheService {

   public static final String USER_CACHE_NAME = "USER_INFO";

   @Override
   @Cacheable(value = USER_CACHE_NAME, key = "#user.userID", cacheManager = "caffeineCacheManager")
   public User save(User user) {
       log.info("Service保存用户:{}", JSONUtil.toJsonStr(user));
       return user;
  }

   @Override
   @CacheEvict(value = USER_CACHE_NAME, key = "#userId", cacheManager = "caffeineCacheManager")
   public void remove(Long userId) {
       log.info("Service删除用户ID:{}", userId);
  }

   @Override
   @CacheEvict(value = USER_CACHE_NAME, allEntries = true, cacheManager = "caffeineCacheManager")
   public void removeAll() {
       log.info("Service删除所有用户缓存");
  }

   @Override
   public User findOne(User user) {
       return null;
  }

   @Override
   @Cacheable(value = USER_CACHE_NAME, key = "#userId", cacheManager = "caffeineCacheManager")
   public User findOne(Long userId) {
       log.info("Service查询指定用户{}缓存!", userId);
       return null;
  }

   @Override
   public boolean update(User user) {
       return false;
  }
}
```

#### 测试

具体接口CacheService，我们注意到在SpringCacheImpl实现中有一个@Primary注解，这是因为接口在装载bean的时候发现有多个实现，这会导致spring容器不知道具体使用哪个实现类去实现接口，且报错`org.springframework.beans.factory.NoUniqueBeanDefinitionException: No qualifying bean of type 'com.mimi.spring.cache.service.CacheService' available: expected single matching bean but found 3: caffeineCache,compositeCache,springCache`。接下里，我们具体针对这三种方案进行测试：

```java
@Slf4j
@RunWith(SpringRunner.class)
@SpringBootTest(classes = CacheApplication.class)
public class CacheTest {

   @Resource(name = "springCache")
   private CacheService springCache;

   @Resource(name = "caffeineCache")
   private CacheService caffeineCache;

   @Resource(name = "compositeCache")
   private CacheService compositeCache;

   /**
    * 本地缓存 - SpringCache缓存
    */
   @Test
   public void testSpringCache() {
       log.info(">>>>>>>>>>>>>>>>>>>> 测试 SpringCache缓存");
       springCache.findOne(1L);
       springCache.findOne(1L);
       System.out.println();

       springCache.remove(1L);
       System.out.println();

       springCache.findOne(1L);
       springCache.findOne(1L);
       System.out.println();

       springCache.removeAll();
       System.out.println();

       springCache.findOne(1L);
       springCache.findOne(1L);
       System.out.println();
  }

   /**
    * 本地缓存 - Caffeine缓存
    */
   @Test
   public void testCaffeineCache() {
       log.info(">>>>>>>>>>>>>>>>>>>> 测试 Caffeine缓存");
       caffeineCache.findOne(1L);
       User one = caffeineCache.findOne(1L);

       User two = caffeineCache.findOne(one);
       one.setUserID(2L);
       User three = caffeineCache.findOne(one);

       caffeineCache.remove(1L);
       caffeineCache.findOne(1L);
  }

   /**
    * 组合缓存 - 用SpringCache的方式，操作Caffeine缓存（也是一层本地缓存）
    */
   @Test
   public void testCompositeCache() {
       log.info(">>>>>>>>>>>>>>>>>>>> 测试 CompositeCache缓存");
       compositeCache.findOne(1L);
       compositeCache.findOne(1L);
       System.out.println();

       compositeCache.remove(1L);
       System.out.println();

       compositeCache.findOne(1L);
       compositeCache.findOne(1L);
       System.out.println();

       compositeCache.removeAll();
       System.out.println();

       compositeCache.findOne(1L);
       compositeCache.findOne(1L);
       System.out.println();
  }
}
```

这里针对上面的caffeineCache做了下测试，结果如下：

```shell
  [2021-09-17 17:18:18][] [main] INFO com.mimi.spring.cache.CacheTest - >>>>>>>>>>>>>>>>>>>> 测试 Caffeine缓存
  [2021-09-17 17:18:18][] [main] INFO com.mimi.spring.cache.service.impl.CaffeineCacheImpl - caffeine2查询用户:1
  [2021-09-17 17:18:18][] [main] INFO com.mimi.spring.cache.config.localValueLoader - caffeine查询DB:1
  [2021-09-17 17:18:18][] [main] INFO com.mimi.spring.cache.service.impl.CaffeineCacheImpl - caffeine2查询用户:1
  [2021-09-17 17:18:18][] [main] INFO com.mimi.spring.cache.service.impl.CaffeineCacheImpl - caffeine1查询用户1:{"userID":1,"password":"test","age":0,"username":"test"}
  [2021-09-17 17:18:18][] [main] INFO com.mimi.spring.cache.service.impl.CaffeineCacheImpl - caffeine1查询用户2:null
  [2021-09-17 17:18:18][] [main] INFO com.mimi.spring.cache.service.impl.CaffeineCacheImpl - caffeine删除用户:1
  [2021-09-17 17:18:18][] [main] INFO com.mimi.spring.cache.service.impl.CaffeineCacheImpl - caffeine2查询用户:1
  [2021-09-17 17:18:18][] [main] INFO com.mimi.spring.cache.config.localValueLoader - caffeine查询DB:1
```

经过分析caffeineCache测试实现，如下：

- 先走(1)，发现查询时走了DB，打印"caffeine查询DB:1"
- 再走(2)，发现只打印了查询用户，没有查询DB，所以缓存生效了
- 再走(3)，仍然还是查询到了相同UserID=1的数据
- 此时拿到(3)的结果，设置UserID=2，执行(4)再去查询时发现结果为null
- 再走(5)，删除UserID=1的数据，此时会删除缓存
- 最后走(6)，发现查询时走了DB，说明缓存删除了