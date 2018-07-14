
springboot注解方式使用redis缓存

-------------

## 引入依赖库

在pom中引入依赖库，如下
```
<dependency>
    <groupId>org.springframework.boot</groupId>
	<artifactId>spring-boot-starter-data-redis</artifactId>
</dependency>
<dependency>
    <groupId>redis.clients</groupId>
    <artifactId>jedis</artifactId>
</dependency>
```

## 注解使用

```
@Cacheable

@Cacheable("product")
@Cacheable(value = {"product","order"}, key = "#root.targetClass+'-'+#id")
@Cacheable(value = "product", key = "#root.targetClass+'-'+#id")
自定义cacheManager
@Cacheable(value = "product", key = "#root.targetClass+'-'+#id” cacheManager="cacheManager")
@CachePut
应用到写数据的方法上，如新增/修改方法
@CachePut(value = "product", key = "#root.targetClass+'-'+#product.id")
@CacheEvict 
即应用到移除数据的方法上，如删除方法
@CacheEvict(value = "product", key = "#root.targetClass+'-'+#id")
提供的SpEL上下文数据
```
Spring Cache提供了一些供我们使用的SpEL上下文数据，下表直接摘自Spring官方文档：

|名字|位置|描述|示例|
|---|----|---|---|
|methodName|root对象|当前被调用的方法名|#root.methodName|
|method|root对象|当前被调用的方法|#root.method.name|
|target|root对象|当前被调用的目标对象|#root.target|
|targetClass|root对象|当前被调用的目标对象类|#root.targetClass|
|args|root对象|当前被调用的方法的参数列表|#root.args[0]|
|caches|root对象|当前方法调用使用的缓存列表（如@Cacheable(value={"cache1", "cache2"})），则有两个cache|#root.caches[0].name|
|argument name|执行上下文|当前被调用的方法的参数，如findById(Long id)，我们可以通过#id拿到参数|#user.id|
|result|执行上下文|方法执行后的返回值（仅当方法执行之后的判断有效，如‘unless’，'cache evict'的beforeInvocation=false）|#result|

## 自定义Cache配置
```
@Configuration
@EnableCaching
public class RedisConfig extends CachingConfigurerSupport {
    /**
    * 自定义redis key值生成策略
    */
    @Bean
    @Override
    public KeyGenerator keyGenerator() {
        return (target, method, params) -> {
            StringBuilder sb = new StringBuilder();
            sb.append(target.getClass().getName());
            sb.append(method.getName());
            for (Object obj : params) {
                sb.append(obj.toString());
            }
            return sb.toString();
        };
    }
    
    @Bean
    public RedisTemplate<String, String> redisTemplate(RedisConnectionFactory factory) {
      ObjectMapper om = new ObjectMapper();
      om.setVisibility(PropertyAccessor.ALL, JsonAutoDetect.Visibility.ANY);
      om.enableDefaultTyping(ObjectMapper.DefaultTyping.NON_FINAL);
      //redis序列化
      Jackson2JsonRedisSerializer jackson2JsonRedisSerializer = new Jackson2JsonRedisSerializer(Object.class);
      jackson2JsonRedisSerializer.setObjectMapper(om);

      StringRedisTemplate template = new StringRedisTemplate(factory);
      template.setValueSerializer(jackson2JsonRedisSerializer);
      template.afterPropertiesSet();
      return template;
    }

    /**
    * 自定义CacheManager
    */
    @Bean
     public CacheManager cacheManager(RedisTemplate redisTemplate) {
         //全局redis缓存过期时间
         RedisCacheConfiguration redisCacheConfiguration = RedisCacheConfiguration.defaultCacheConfig().entryTtl(Duration.ofDays(1));  
         RedisCacheWriter redisCacheWriter = RedisCacheWriter.nonLockingRedisCacheWriter(redisTemplate.getConnectionFactory());
        return new RedisCacheManager(redisCacheWriter, redisCacheConfiguration);
    }
}
```

