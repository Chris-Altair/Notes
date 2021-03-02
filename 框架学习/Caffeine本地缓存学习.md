高性能本地缓存

## 1. 定义



## 2. 用法

### ①构建缓存

Caffeine提供了三种定时驱逐策略：

- expireAfterAccess(long, TimeUnit):在最后一次访问或者写入后开始计时，在指定的时间后过期。假如一直有请求访问该key，那么这个缓存将一直不会过期。

- expireAfterWrite(long, TimeUnit): 在最后一次写入缓存后开始计时，在指定的时间后过期。

- expireAfter(Expiry): 自定义策略，过期时间由Expiry实现独自计算。

  ### expireAfter 和 refreshAfter 之间的区别

  - expireAfter 条件触发后，新的值更新完成前，所有请求都会被阻塞，更新完成后其他请求才能访问这个值。这样能确保获取到的都是最新的值，但是有性能损失。
  - refreshAfter 条件触发后，新的值更新完成前也可以访问，不会被阻塞，只是获取的是旧的数据。更新结束后，获取的才是新的数据。有可能获取到脏数据。

```java
@Configuration
public class CacheConfig {
    @Bean
    public Cache<String, Object> caffeineCache(){
        return Caffeine.newBuilder()
                // 设置最后一次写入或访问后经过固定时间过期
                .expireAfterWrite(60, TimeUnit.SECONDS)
                // 初始的缓存空间大小
                .initialCapacity(100)
                // 缓存的最大条数
                .maximumSize(1000)
                .build();
    }
}
```

### ③使用缓存

```java
public class CacheTest extends BaseCase {
    @Autowired
    Cache<String, Object> caffeineCache;
    @Autowired
    private SensitiveWordRepository sensitiveWordRepository;

    @Test
    public void test() {
        String key = "sensitiveWordTree";
        Set<String> sensitiveWordSet = sensitiveWordRepository.findAllWord();
        SensitiveWordTree root = SensitiveWordTree.buildTree(sensitiveWordSet);
        caffeineCache.put(key, root);
        Assert.assertNotNull(caffeineCache.asMap().get(key));
        //下面两个都是删除缓存
        caffeineCache.invalidate(key);
        caffeineCache.asMap().remove(key);
        //如果没有就返回null
        Assert.assertNull(caffeineCache.getIfPresent(key));
        //如果没有key就调用buildTree构建，返回构建后的结果并将该结果存到缓存中;能查到就直接返回
        //这种形式是线程安全的，即使来多个请求也只会构建一次
        Assert.assertEquals(root, caffeineCache.get(key, k -> SensitiveWordTree.buildTree(sensitiveWordSet)));
    }
}
```



