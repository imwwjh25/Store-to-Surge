### 一、先澄清：Redis 服务端的 “线程模型”≠ 线程池

首先要明确：**Redis 服务端本身是单线程模型（核心网络 IO / 命令执行），没有传统意义上的线程池**。

- Redis 用单线程处理所有客户端的命令请求（避免线程切换开销），保证了高性能；
- 只有 **异步任务（如持久化、过期键删除、集群同步）** 会使用后台线程池（由 `io-threads`、`bio-*` 等配置控制），但这不是开发者需要手动配置的，属于 Redis 服务端内部机制。

### 二、重点：Java 客户端的 Redis 线程池（开发中必用）

在 Spring Boot 中操作 Redis 时，客户端（Jedis/Lettuce）的线程池是优化性能的核心，目的是**复用连接、避免频繁创建 / 销毁连接的开销**。

#### 1. 两种主流客户端的线程池差异








| 客户端  |           线程模型           |           线程池核心作用           |
| :-----: | :--------------------------: | :--------------------------------: |
|  Jedis  |     直连型（线程不安全）     | 管理 Jedis 连接实例，保证线程安全  |
| Lettuce | Netty 异步非阻塞（线程安全） | 管理 IO 线程和连接池，优化异步请求 |

下面重点讲最常用的 **Jedis 线程池**（Lettuce 是默认客户端，连接池配置逻辑类似）。

#### 2. Jedis 线程池（JedisPool）配置实战

Jedis 本身是线程不安全的，必须通过 `JedisPool`（基于 Apache Commons Pool2）创建连接池，复用连接。

##### 步骤 1：引入依赖（Spring Boot）









```
<!-- Spring Data Redis + Jedis -->
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-data-redis</artifactId>
</dependency>
<dependency>
    <groupId>redis.clients</groupId>
    <artifactId>jedis</artifactId>
</dependency>
<!-- 连接池依赖（Commons Pool2） -->
<dependency>
    <groupId>org.apache.commons</groupId>
    <artifactId>commons-pool2</artifactId>
</dependency>
```

##### 步骤 2：配置 Jedis 连接池（Spring Boot 配置类）








```
import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;
import org.springframework.data.redis.connection.RedisConnectionFactory;
import org.springframework.data.redis.connection.jedis.JedisConnectionFactory;
import org.springframework.data.redis.core.RedisTemplate;
import org.springframework.data.redis.serializer.StringRedisSerializer;
import redis.clients.jedis.JedisPoolConfig;

@Configuration
public class RedisConfig {

    // 1. 配置 Jedis 连接池参数
    @Bean
    public JedisPoolConfig jedisPoolConfig() {
        JedisPoolConfig poolConfig = new JedisPoolConfig();
        // 核心池大小（默认5）：空闲时保持的最小连接数
        poolConfig.setMinIdle(5);
        // 最大池大小（默认8）：连接池最大可用连接数
        poolConfig.setMaxTotal(20);
        // 最大等待时间（毫秒）：获取连接时的最大阻塞时间，-1表示无限制
        poolConfig.setMaxWaitMillis(3000);
        // 空闲连接检查：获取连接时验证是否可用（避免拿到失效连接）
        poolConfig.setTestOnBorrow(true);
        return poolConfig;
    }

    // 2. 配置 Redis 连接工厂（绑定连接池）
    @Bean
    public RedisConnectionFactory redisConnectionFactory(JedisPoolConfig jedisPoolConfig) {
        JedisConnectionFactory factory = new JedisConnectionFactory();
        factory.setHostName("localhost"); // Redis 地址
        factory.setPort(6379); // Redis 端口
        factory.setPassword(""); // Redis 密码（无则空）
        factory.setDatabase(0); // 使用第0个数据库
        factory.setPoolConfig(jedisPoolConfig); // 绑定连接池
        return factory;
    }

    // 3. 配置 RedisTemplate（简化操作）
    @Bean
    public RedisTemplate<String, Object> redisTemplate(RedisConnectionFactory factory) {
        RedisTemplate<String, Object> template = new RedisTemplate<>();
        template.setConnectionFactory(factory);
        // 设置序列化器（避免乱码）
        template.setKeySerializer(new StringRedisSerializer());
        template.setValueSerializer(new StringRedisSerializer());
        return template;
    }
}
```

##### 步骤 3：使用连接池操作 Redis（业务代码）



```
import org.springframework.data.redis.core.RedisTemplate;
import org.springframework.stereotype.Service;
import javax.annotation.Resource;

@Service
public class RedisService {

    @Resource
    private RedisTemplate<String, Object> redisTemplate;

    public void setValue(String key, String value) {
        // 从连接池获取连接，用完自动归还，无需手动关闭
        redisTemplate.opsForValue().set(key, value);
    }

    public Object getValue(String key) {
        return redisTemplate.opsForValue().get(key);
    }
}
```

#### 3. Lettuce 连接池配置（Spring Boot 默认）

Lettuce 是异步非阻塞客户端，天生线程安全，默认不使用连接池，但高并发下仍建议配置：







```
import org.springframework.boot.autoconfigure.data.redis.LettuceClientConfigurationBuilderCustomizer;
import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;
import org.springframework.data.redis.connection.lettuce.LettucePoolingClientConfiguration;
import org.apache.commons.pool2.impl.GenericPoolConfig;

@Configuration
public class LettuceRedisConfig {

    @Bean
    public LettuceClientConfigurationBuilderCustomizer lettuceClientConfigurationBuilderCustomizer() {
        return builder -> {
            // 配置 Lettuce 连接池
            GenericPoolConfig<?> poolConfig = new GenericPoolConfig<>();
            poolConfig.setMaxTotal(20);
            poolConfig.setMinIdle(5);
            poolConfig.setMaxWaitMillis(3000);
            builder.poolConfig(poolConfig);
        };
    }
}
```

### 三、线程池核心参数说明（通用）

不管是 Jedis 还是 Lettuce，连接池的核心参数逻辑一致，重点关注：









|     参数      |                             作用                             |
| :-----------: | :----------------------------------------------------------: |
|   maxTotal    | 连接池最大连接数（核心）：根据业务 QPS 调整，过大易导致 Redis 服务端压力大 |
|    minIdle    | 最小空闲连接数：保证随时有可用连接，避免高并发时创建连接的开销 |
| maxWaitMillis |     获取连接的最大等待时间：超时抛异常，防止线程无限阻塞     |
| testOnBorrow  | 获取连接时验证是否有效：避免拿到断开的连接（如 Redis 重启后） |

### 四、使用线程池的核心注意事项

1. **避免手动创建 Jedis 实例**：直接 `new Jedis()` 会每次创建新连接，高并发下性能极差，必须用连接池；
2. **连接池大小要合理**：maxTotal 不是越大越好（Redis 服务端最大连接数默认 10000），建议根据压测调整（比如 20-50）；
3. **用完自动归还**：Spring Data Redis 会自动管理连接的获取和归还，无需手动调用 `close()`；
4. **异常处理**：获取连接超时（`PoolTimeoutException`）通常是连接池满或 Redis 服务端不可用，需监控并调整参数。

### 总结

1. **Redis 服务端**：核心逻辑单线程，仅异步任务用后台线程，无开发者可配置的线程池；
2. **Java 客户端**：Jedis 需通过 `JedisPool` 保证线程安全和连接复用，Lettuce 建议配置连接池提升高并发性能；
3. **核心参数**：maxTotal（最大连接数）、maxWaitMillis（最大等待时间）、testOnBorrow（连接有效性验证）是配置线程池的关键。