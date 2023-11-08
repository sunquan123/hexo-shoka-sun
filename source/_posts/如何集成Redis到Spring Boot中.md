---
layout: shoka
title: 如何集成Redis到Spring Boot中
date: 2023-11-08 19:05:14
tags: ['Redis','Spring Boot','单元测试']
---

## 如何集成Redis到Spring Boot中

### 添加redis所需依赖：

```xml
<!-- redis 缓存操作 -->
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-data-redis</artifactId>
    <version>2.5.15</version>
</dependency>
<!-- 阿里JSON解析器 -->
<dependency>
    <groupId>com.alibaba.fastjson2</groupId>
    <artifactId>fastjson2</artifactId>
    <version>2.0.25</version>
</dependency>
<dependency>
    <groupId>org.apache.commons</groupId>
    <artifactId>commons-pool2</artifactId>
    <version>2.9.0</version>
</dependency>
```

### 添加配置项

常规配置如下： 

在application.yml配置文件中配置 redis的连接信息

```yaml
spring:
  redis:
    host: 127.0.0.1
    port: 6379
    password:
    database: 0
    lettuce:
      pool:
        max-idle: 16
        max-active: 32
        min-idle: 8
```

### 配置类

主要配置redis的string和hash两种数据结构的序列化器，需要注意这里使用了FastJson2作为hash的值序列化器。

```java
@Configuration
public class RedisConfig {
  @Bean
  public RedisTemplate<String, Object> redisTemplate(RedisConnectionFactory factory) {
    RedisTemplate<String, Object> redisTemplate = new RedisTemplate<String, Object>();
    redisTemplate.setConnectionFactory(factory);
    StringRedisSerializer stringRedisSerializer = new StringRedisSerializer();

    FastJson2JsonRedisSerializer fastJson2JsonRedisSerializer =
        new FastJson2JsonRedisSerializer(Object.class);

    // 设置key和value的序列化规则
    redisTemplate.setKeySerializer(stringRedisSerializer); // key的序列化类型
    redisTemplate.setValueSerializer(fastJson2JsonRedisSerializer); // value的序列化类型
    redisTemplate.setHashKeySerializer(stringRedisSerializer);
    redisTemplate.setHashValueSerializer(fastJson2JsonRedisSerializer);
    redisTemplate.afterPropertiesSet();

    return redisTemplate;
  }
}
```

```java
public class FastJson2JsonRedisSerializer<T> implements RedisSerializer<T>
{
    public static final Charset DEFAULT_CHARSET = Charset.forName("UTF-8");

    private Class<T> clazz;

    public FastJson2JsonRedisSerializer(Class<T> clazz)
    {
        super();
        this.clazz = clazz;
    }

    @Override
    public byte[] serialize(T t) throws SerializationException
    {
        if (t == null)
        {
            return new byte[0];
        }
        return JSON.toJSONString(t, JSONWriter.Feature.WriteClassName).getBytes(DEFAULT_CHARSET);
    }

    @Override
    public T deserialize(byte[] bytes) throws SerializationException
    {
        if (bytes == null || bytes.length <= 0)
        {
            return null;
        }
        String str = new String(bytes, DEFAULT_CHARSET);

        return JSON.parseObject(str, clazz, JSONReader.Feature.SupportAutoType);
    }
}
```

### 项目中使用

```java
@Component
public class CacheUtils {
  private static Logger logger = LoggerFactory.getLogger(CacheUtils.class);
  @Autowired public RedisTemplate redisTemplate;
  private static final String SYS_CACHE = "sys-cache";


  /**
   * 获取缓存
   *
   * @param cacheName
   * @param key
   * @return
   */
  public Object get(String cacheName, String key) {
    //    return getCache(cacheName).get(getKey(key));
    return redisTemplate.opsForHash().get(cacheName, getKey(key));
  }


  /**
   * 写入缓存
   *
   * @param cacheName
   * @param key
   * @param value
   */
  public void put(String cacheName, String key, Object value) {
    redisTemplate.opsForHash().put(cacheName, getKey(key), value);
  }

  /**
   * 从缓存中移除
   *
   * @param cacheName
   * @param key
   */
  public void remove(String cacheName, String key) {
    redisTemplate.opsForHash().delete(cacheName, getKey(key));
  }

  /**
   * 从缓存中移除所有
   *
   * @param cacheName
   */
  public void removeAll(String cacheName) {
    Set<String> keys = redisTemplate.opsForHash().keys(cacheName);
    for (String key : keys) {
      redisTemplate.opsForHash().delete(cacheName, key);
    }
    logger.info("清理缓存： {} => {}", cacheName, keys);
  }
}
```

### 如何在单元测试中使用Redis？

重点解决的问题是单元测试环境中似乎无法自动注入RedisTemplate，所以手动初始化RedisTemplate，中间需要初始化LettuceConnectionFactory工厂类，设置好redis服务器的地址和密码等参数。重点需要执行connectionFactory.afterPropertiesSet()方法，保证工厂类能正常初始化成功。

```java
@SpringBootTest(classes = CacheUtilsTest.class)
class CacheUtilsTest {
  private static Logger logger = LoggerFactory.getLogger(CacheUtilsTest.class);
  private static RedisTemplate<String, Object> redisTemplate;

  @BeforeAll
  static void startRedis() {
    redisTemplate = new RedisTemplate<String, Object>();
    RedisStandaloneConfiguration redisStandaloneConfiguration = new RedisStandaloneConfiguration("192.168.56.100", 6379);
    redisStandaloneConfiguration.setDatabase(0);
    redisStandaloneConfiguration.setPassword("123456");
    LettuceConnectionFactory connectionFactory = new LettuceConnectionFactory(redisStandaloneConfiguration);
    connectionFactory.afterPropertiesSet();
    redisTemplate.setConnectionFactory(connectionFactory);
    StringRedisSerializer stringRedisSerializer = new StringRedisSerializer();

    FastJson2JsonRedisSerializer fastJson2JsonRedisSerializer =
            new FastJson2JsonRedisSerializer(Object.class);

    // 设置key和value的序列化规则
    redisTemplate.setKeySerializer(stringRedisSerializer); // key的序列化类型
    redisTemplate.setValueSerializer(fastJson2JsonRedisSerializer); // value的序列化类型
    redisTemplate.setHashKeySerializer(stringRedisSerializer);
    redisTemplate.setHashValueSerializer(fastJson2JsonRedisSerializer);
    redisTemplate.afterPropertiesSet();
  }

  @Test
  void testGetCacheNames() {
    String pattern = "sys-config";
    redisTemplate.opsForHash().put("sys-config","sys_config:sys.index.skinName","skin-purple");
    Set<String> keys = new HashSet<>();
    // 获取redis全部key
    Set<String> hashKetSet = redisTemplate.keys("*");
    logger.info("getCacheNames:{}", hashKetSet);
    for (Object s : hashKetSet) {
      String ss = (String) s;
      if (ss.startsWith(pattern)) {
        keys.add(ss);
      }
    }
    logger.info("getCacheNames end:{}", keys);
  }
}
```

### 结尾

可以使用embedded-redis在单元测试环境下模拟redis服务器，进行redis相关的测试，后续有需求了再添加这个功能。
