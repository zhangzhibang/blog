---
title: Redeis操作
---

## 1.springboot集成redis

- 启动redis服务

- 导入spring boot和redis的依赖包

  ```xml
  <dependency>
              <groupId>org.springframework.boot</groupId>
              <artifactId>spring-boot-starter-redis</artifactId>
              <version>1.4.1.RELEASE</version>
   </dependency>
  ```

- 配置文件配置redis

  ```yaml
  spring: 
  		redis:
      			host: 127.0.0.1 #配置redis主机
      			port: 6379
      	jedis:
       			pool:
          			  max-active: 8
          			  max-wait: -1
  
  ```

- 注入RedisTemplate和StringRedisTemplate

  ```Java
    @Autowired
      StringRedisTemplate stringRedisTemplate;
     @Autowired
      RedisTemplate redisTemplate;
  ```



> StringRedisTemplate的键和值都是字符串，在传入和获取的时候都是对字符串进行操作；
>
> RedisTemplate的键和值都是对象，在进行数据的传入和获取时会对键和值进行序列化和反序列化，传入的对象必须实现序列化接口。

```java
@Data
@AllArgsConstructor
@NoArgsConstructor
@Accessors(chain = true)
public class User implements Serializable {  //实体类必须序列化后才可以传入到redis
    private int id;
    private String name;
    private String password;
    private String per;

}
```





> 由于 RedisTemplate 会对键和值都进行序列化操作，有着较大的局限性，所以会对其键的序列化进行操作使键直接展示，只对值进行序列化操作。

```java
// redisTemplate的键和值默认的序列化方式是jdk序列化，而 StringRedisTemplate默认对键和值都进行String序列化，因此需要把键的序列化改为String序列化来保证使得键直接展示传入的值
redisTemplate.setKeySerializer(new StringRedisSerializer());

//针对特殊类型的哈希值，我们希望其map的键也能够直接展示，故使用string序列化
        redisTemplate.setHashKeySerializer(new StringRedisSerializer());
```



> 当频繁的对同一个key进行操作时，可以把模板与指定key进行绑定，然后直接对绑定的值进行操作就是对指定key进行操作。

```java
 BoundValueOperations zzb = redisTemplate.boundValueOps("zzb");
//返回的值就是对zzb的绑定的值
        zzb.append("666");
        zzb.set("zzb1");
        zzb.get();
```







## 2.mybatis缓存集成redis

- 编写工具类获取被spring管理的redisTemplate，==该获取bean的工具类必须交给spring容器进行管理==

  ```Java
  
  import org.springframework.beans.BeansException;
  import org.springframework.context.ApplicationContext;
  import org.springframework.context.ApplicationContextAware;
  
  /*
  author:zzb
  此工具类用来在获取被spring容器管理的Bean，不被spring管理的类可通过此工具类得到被管理的bean，
   */
  @Component
  public class ApplicationContextUtils implements ApplicationContextAware {
  
      private static ApplicationContext applicationContext;
      //实现了此接口之后，spring容器在启动过后会自动调用这个方法，并且将创好的工厂以参数的形式传递进来
      @Override
      public void setApplicationContext(ApplicationContext applicationContext) throws BeansException {
          this.applicationContext=applicationContext;
      }
  
  //调用该静态方法传入需要获取的Bean的名称，Bean在spring中管理的时候默认的名字是类名首字母小写，返回对应的Bean
      public static Object  getBean(String beanName){
          return applicationContext.getBean(beanName);
      }
  }
  ```

- mybatis开启缓存只需要在mapper文件中添加cache标签即可，更换为redis缓存则需要实现Cache接口并重写对应方法在cache标签中指定对应的缓存类即可

```Java
package com.zzb.Cache;

import com.zzb.Utils.ApplicationContextUtils;
import org.apache.ibatis.cache.Cache;
import org.springframework.data.redis.core.RedisTemplate;
import org.springframework.data.redis.serializer.StringRedisSerializer;

public class RedisCache implements Cache {

    private RedisTemplate getTemplate(){
        //通过工具类获取redisTemplate，然后对键的序列化进行操作
        RedisTemplate redisTemplate= (RedisTemplate) ApplicationContextUtils.getBean("redisTemplate");
        redisTemplate.setKeySerializer(new StringRedisSerializer());
        redisTemplate.setHashKeySerializer(new StringRedisSerializer());
        return redisTemplate;
    }

//实现Cache接口需要提供一个包含id的构造函数，这个id对应的就是mapper文件里面的namespace
    private final String id;

    public  RedisCache(String id){
        this.id=id;
    }

    //此get方法的返回值就是传入的id,必须实现该方法
    @Override
    public String getId() {
        return id;
    }

    @Override
    public void putObject(Object key, Object value) {
        //程序从数据库查到数据的时候会调用此方法向缓存中写入数据
        System.out.println("努力     放入缓存中");
        getTemplate().opsForHash().put(id.toString(),key.toString(),value);
    }

    @Override
    public Object getObject(Object key) {
        //程序会在查询数据库之前向缓存中查询数据
        System.out.println("努力     获取=============");
        return getTemplate().opsForHash().get(id.toString(),key.toString());
        //return null;
    }


    @Override
    public Object removeObject(Object key) {
        //mybatis默认没有实现该方法，为保留方法
        return null;
    }

    @Override
    public void clear() {
    //当对数据库内的数据进行修改操作的时候，会调用此方法，需要在方法内进行缓存清空操作
        getTemplate().delete(id.toString());
    }

    @Override
    public int getSize() {
        return getTemplate().opsForHash().size(id.toString()).intValue();
    }
}
```



### 2.1共享缓存

默认的缓存策略仅仅适用于单表查询，当对数据进行修改的时候仅清空当前表的对应缓存，而当表之间具有关联关系的时候这种缓存策略就会造成数据的不一致，这时候可以通过共享缓存实现数据同步。

```xml
<!--可以在mapper文件中使用cache-ref标签进行缓存共享，当实体类中有另一个实体类的属性关系的时候可直接在实体类的mapper文件中添加该标签，namespace就是相关联的实体类对应的mapper的namespace-->
<cache-ref namespace="com.zzb.mapper.usermapper"/>
```

### 2.2缓存优化

使用redis进行缓存的使用的是哈希类型的数据，mybatis默认生成的第二个键的随机ID过长不便于操作，于是可以通过md5对随机ID进行加密进行ID的简化。

```java
  private String getStringtoMd5(String s){
        //使用spring自带的工具类中的md5加密方法对key进行加密
       return DigestUtils.md5DigestAsHex(s.getBytes());
    }
```





## 3.缓存穿透

应用程序大量向数据库查询一个不存在的数据记录导致缓存在这种情况下无法得到利用的情况。

> 解决方案：
>
> -  布隆过滤器是将所有可能存在的数据哈希到一个足够大的bitmap中，一个一定不存在的数据会被 这个bitmap拦截掉，从而避免了对底层存储系统的查询压力。 
> -  如果一个查询结果空（不管是数 据不存在，还是查询异常），我们仍然把这个空结果进行缓存，但它的过期时间会很短，几分钟即可。

## 4.缓存雪崩

缓存中的大量数据在某一时刻同时失效，大量访问涌向数据库造成数据库雪崩

> 解决方案：
>
> -  在原有的失效时间基础上增加一个随机值，让缓存的失效时间错开，就可以有效的避免缓存雪崩 。
> - 加锁保证单线程读取

## 5.缓存击穿

一个热点的缓存数据突然失效，短时间内大量对该热点的访问涌入数据库造成崩溃。

> 解决方案：
>
> - 加锁
> - 提前更新缓存， 当我们读取缓存的时候，先判断它是否快到过期时间，如果是则在返回缓存数据的同时，后台异步请求DB重设缓存，更新它的过期时间。 

## 6.redis实现session管理

- 导入依赖

  ```xml
        <dependency>
              <groupId>org.springframework.session</groupId>
              <artifactId>spring-session-data-redis</artifactId>
          </dependency>
  ```

- 创建配置类

  ```java
  import org.springframework.context.annotation.Configuration;
  import org.springframework.session.data.redis.config.annotation.web.http.EnableRedisHttpSession;
  
  @Configuration
  @EnableRedisHttpSession
  public class SessionConfig {
  }
  ```

- 运行即可



