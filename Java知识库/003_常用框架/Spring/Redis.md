# 依赖
```XML
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-data-redis</artifactId>
</dependency>
<dependency>
    <groupId>org.apache.commons</groupId>
    <artifactId>commons-pool2</artifactId>
</dependency>
```

# 配置

```YAML
spring:
  data:
    redis:
      #username: 
      password: ${REDIS_PASS:}
      timeout: 2000ms
      lettuce:
        pool:
          enabled: true
          max-active: 16
          max-idle: 8
          min-idle: 2
          max-wait: 1000ms
          # 这两个参数是关键，默认没有开启
          # 每 30s 检测一次空闲连接
          time-between-eviction-runs: 30s
          # 空闲超过 60s 的连接驱逐
          min-evictable-idle-time: 60s

      # ── 单点（三选一）──────────────────────────
      #host: 127.0.0.1
      #port: 6379
      #database: 0
      # ── 哨兵（三选一）──────────────────────────
      # password: ${REDIS_PASS:}
      # sentinel:
      #   master: mymaster
      #   nodes:
      #     - sentinel1:26379
      #     - sentinel2:26379
      #     - sentinel3:26379
      #   password: ${SENTINEL_PASS:}   # 哨兵自身密码
      # ── 集群（三选一）──────────────────────────
      cluster:
        nodes:
          - 192.168.1.1:6379
          - 192.168.1.2:6379
          - 192.168.1.3:6379
          - 192.168.1.4:6379  # replica 节点也可以列进来，客户端自动发现全部
          - 192.168.1.5:6379
          - 192.168.1.6:6379
        max-redirects: 3       # MOVED/ASK 重定向最大次数
          
        cluster:
          refresh:
            adaptive: true     # 感知到拓扑变化时自动刷新路由表
            period: 30s        # 定期主动刷新
```

# RedisConfig 配置类

```JAVA
@Configuration
@EnableConfigurationProperties(RedisProperties.class)
public class RedisConfig {

    private final RedisProperties redisProperties;

    public RedisConfig(RedisProperties redisProperties) {
        this.redisProperties = redisProperties;
    }

    // ══════════════════════════════════════════════
    // 节点宕机期间主库选举时，读请求可以降级到 Replica
    // ══════════════════════════════════════════════
	@Bean 
	public LettuceClientConfigurationBuilderCustomizer customizer() { 
		return builder -> builder.readFrom(ReadFrom.REPLICA_PREFERRED); 
		// MASTER 只读主（默认） 
		// REPLICA_PREFERRED 优先读从，从不可用时读主 
		// NEAREST 读延迟最低的节点 
	}
    // ══════════════════════════════════════════════
    // ConnectionFactory：根据配置自动选择模式
    // ══════════════════════════════════════════════

    @Bean
    public RedisConnectionFactory redisConnectionFactory() {
        if (redisProperties.getCluster() != null) {
            return createClusterConnectionFactory();
        }
        if (redisProperties.getSentinel() != null) {
            return createSentinelConnectionFactory();
        }
        return createStandaloneConnectionFactory();
    }

    // ── 单点 ──────────────────────────────────────

    private RedisConnectionFactory createStandaloneConnectionFactory() {
        RedisStandaloneConfiguration config = new RedisStandaloneConfiguration();
        config.setHostName(redisProperties.getHost());
        config.setPort(redisProperties.getPort());
        config.setDatabase(redisProperties.getDatabase());
        if (redisProperties.getPassword() != null) {
            config.setPassword(RedisPassword.of(redisProperties.getPassword()));
        }
        return new LettuceConnectionFactory(config, buildLettuceClientConfig());
    }

    // ── 哨兵 ──────────────────────────────────────

    private RedisConnectionFactory createSentinelConnectionFactory() {
        RedisProperties.Sentinel sentinel = redisProperties.getSentinel();
        RedisSentinelConfiguration config =
            new RedisSentinelConfiguration(sentinel.getMaster(), 
                new HashSet<>(sentinel.getNodes()));
        config.setDatabase(redisProperties.getDatabase());
        if (redisProperties.getPassword() != null) {
            config.setPassword(RedisPassword.of(redisProperties.getPassword()));
        }
        // 哨兵节点自身的密码（sentinel auth）
        if (sentinel.getPassword() != null) {
            config.setSentinelPassword(RedisPassword.of(sentinel.getPassword()));
        }
        LettuceConnectionFactory factory =
            new LettuceConnectionFactory(config, buildLettuceClientConfig());
        // 哨兵模式配置读从节点，提升读可用性
        factory.setReadFrom(ReadFrom.REPLICA_PREFERRED);
        return factory;
    }

    // ── 集群 ──────────────────────────────────────

    private RedisConnectionFactory createClusterConnectionFactory() {
        RedisProperties.Cluster cluster = redisProperties.getCluster();
        RedisClusterConfiguration config =
            new RedisClusterConfiguration(cluster.getNodes());
        if (cluster.getMaxRedirects() != null) {
            config.setMaxRedirects(cluster.getMaxRedirects());
        }
        if (redisProperties.getPassword() != null) {
            config.setPassword(RedisPassword.of(redisProperties.getPassword()));
        }
        LettuceConnectionFactory factory =
            new LettuceConnectionFactory(config, buildClusterLettuceClientConfig());
        factory.setReadFrom(ReadFrom.REPLICA_PREFERRED);
        return factory;
    }

    // ══════════════════════════════════════════════
    // Lettuce 客户端配置（单点 / 哨兵共用）
    // ══════════════════════════════════════════════

    private LettuceClientConfiguration buildLettuceClientConfig() {
        return LettucePoolingClientConfiguration.builder()
            .poolConfig(buildPoolConfig())
            .commandTimeout(redisProperties.getTimeout() != null
                ? redisProperties.getTimeout()
                : Duration.ofSeconds(2))
            .shutdownTimeout(Duration.ofMillis(200))
            .build();
    }

    // ══════════════════════════════════════════════
    // Lettuce 客户端配置（集群专用，多一个拓扑刷新）
    // ══════════════════════════════════════════════

    private LettuceClientConfiguration buildClusterLettuceClientConfig() {
        // 集群拓扑自适应刷新
        ClusterTopologyRefreshOptions topologyRefreshOptions =
            ClusterTopologyRefreshOptions.builder()
                .enableAdaptiveRefreshTrigger(
                    // 感知到这些事件时立即刷新路由表
                    ClusterTopologyRefreshOptions.RefreshTrigger.MOVED_REDIRECT,
                    ClusterTopologyRefreshOptions.RefreshTrigger.PERSISTENT_RECONNECTS,
                    ClusterTopologyRefreshOptions.RefreshTrigger.ASK_REDIRECT
                )
                .enablePeriodicRefresh(Duration.ofSeconds(30)) // 兜底定期刷新
                .build();

        ClusterClientOptions clusterClientOptions = ClusterClientOptions.builder()
            .topologyRefreshOptions(topologyRefreshOptions)
            .autoReconnect(true)
            .build();

        return LettucePoolingClientConfiguration.builder()
            .poolConfig(buildPoolConfig())
            .clientOptions(clusterClientOptions)
            .commandTimeout(redisProperties.getTimeout() != null
                ? redisProperties.getTimeout()
                : Duration.ofSeconds(2))
            .shutdownTimeout(Duration.ofMillis(200))
            .build();
    }

    // ══════════════════════════════════════════════
    // 连接池配置（三种模式共用）
    // ══════════════════════════════════════════════

    private GenericObjectPoolConfig<Object> buildPoolConfig() {
        RedisProperties.Lettuce lettuce = redisProperties.getLettuce();
        RedisProperties.Pool pool = (lettuce != null) ? lettuce.getPool() : null;

        GenericObjectPoolConfig<Object> config = new GenericObjectPoolConfig<>();
        if (pool != null) {
            config.setMaxTotal(pool.getMaxActive());
            config.setMaxIdle(pool.getMaxIdle());
            config.setMinIdle(pool.getMinIdle());
            if (pool.getMaxWait() != null) {
                config.setMaxWait(pool.getMaxWait());
            }
            if (pool.getTimeBetweenEvictionRuns() != null) {
                config.setTimeBetweenEvictionRuns(pool.getTimeBetweenEvictionRuns());
            }
            if (pool.getMinEvictableIdleTime() != null) {
                config.setMinEvictableIdleTime(pool.getMinEvictableIdleTime());
            }
        } else {
            // 没有配置连接池时的默认值
            config.setMaxTotal(16);
            config.setMaxIdle(8);
            config.setMinIdle(2);
            config.setMaxWait(Duration.ofMillis(1000));
            config.setTimeBetweenEvictionRuns(Duration.ofSeconds(30));
            config.setMinEvictableIdleTime(Duration.ofSeconds(60));
        }
        // 取出连接时检测是否有效，避免使用死连接
        config.setTestOnBorrow(true);
        config.setTestWhileIdle(true);
        return config;
    }

    // ══════════════════════════════════════════════
    // RedisTemplate：JSON 序列化
    // ══════════════════════════════════════════════

    @Bean
    public RedisTemplate<String, Object> redisTemplate(
            RedisConnectionFactory redisConnectionFactory) {
        RedisTemplate<String, Object> template = new RedisTemplate<>();
        template.setConnectionFactory(redisConnectionFactory);

        ObjectMapper objectMapper = new ObjectMapper();
        // 序列化时带上类型信息，反序列化时能还原为原始类型
        objectMapper.activateDefaultTyping(
            objectMapper.getPolymorphicTypeValidator(),
            ObjectMapper.DefaultTyping.NON_FINAL
        );
        objectMapper.configure(
            DeserializationFeature.FAIL_ON_UNKNOWN_PROPERTIES, false);

        Jackson2JsonRedisSerializer<Object> jsonSerializer =
            new Jackson2JsonRedisSerializer<>(objectMapper, Object.class);

        template.setKeySerializer(new StringRedisSerializer());
        template.setValueSerializer(jsonSerializer);
        template.setHashKeySerializer(new StringRedisSerializer());
        template.setHashValueSerializer(jsonSerializer);
        template.afterPropertiesSet();
        return template;
    }

    // StringRedisTemplate 直接用 Spring Boot 自动装配的即可，无需手动声明
}
```

# 使用示例
```JAVA
@Service
@RequiredArgsConstructor
public class RedisClusterService {

    private final RedisTemplate<String, Object> redisTemplate;
    private final StringRedisTemplate stringRedisTemplate;

    // ── String 操作 ──────────────────────────────

    public void set(String key, Object value, Duration ttl) {
        redisTemplate.opsForValue().set(key, value, ttl);
    }

    public Object get(String key) {
        return redisTemplate.opsForValue().get(key);
    }

    // ── Hash 操作 ────────────────────────────────

    public void hset(String key, String field, Object value) {
        redisTemplate.opsForHash().put(key, field, value);
    }

    public Object hget(String key, String field) {
        return redisTemplate.opsForHash().get(key, field);
    }

    // ── 原子计数（集群下 incr 只支持单 key）────────

    public Long increment(String key) {
        return stringRedisTemplate.opsForValue().increment(key);
    }

    // ── 分布式锁（简单版）────────────────────────

    public boolean tryLock(String lockKey, String value, Duration ttl) {
        Boolean result = stringRedisTemplate.opsForValue()
            .setIfAbsent(lockKey, value, ttl);
        return Boolean.TRUE.equals(result);
    }

    public void unlock(String lockKey, String value) {
        // 用 Lua 脚本保证原子性，避免误删其他线程的锁
        String script =
            "if redis.call('get', KEYS[1]) == ARGV[1] then " +
            "   return redis.call('del', KEYS[1]) " +
            "else return 0 end";
        stringRedisTemplate.execute(
            new DefaultRedisScript<>(script, Long.class),
            List.of(lockKey),
            value
        );
    }
}
```

# 注意点
## 连接池连接清理策略

### 连接池在节点宕机时的行为

```
Redis Cluster
  ├── Master 1 (正常)
  ├── Master 2 (宕机) ← 连接池里有指向它的连接
  └── Master 3 (正常)

连接池里已有的连接：
  ├── 指向 Master 1 的连接  → 正常可用
  ├── 指向 Master 2 的连接  → 已死，但连接池不知道
  └── 指向 Master 3 的连接  → 正常可用
```

连接池里指向宕机节点的连接**不会立即失效**，只有在下次使用这条连接时才会发现它已经断了，然后抛出异常。
### 三个关键机制

#### ① 拓扑刷新（路由层）

这是最重要的，`adaptive: true` 开启后：

```
节点宕机
  → 集群内部选举新 Master（通常 10-30s）
  → Lettuce 收到 MOVED 错误
  → 触发拓扑刷新，更新路由表
  → 新请求自动路由到新 Master
```

但注意：**拓扑刷新只影响新请求的路由，不会主动清理连接池里的旧连接。**

#### ② 连接池的旧连接处理

旧连接被取出使用时，会出现两种情况：

```
取出指向宕机节点的连接
  ├── 执行命令 → 抛出 RedisConnectionException
  │     → Lettuce 重试（如果开启重试）
  │     → 连接标记为失效，从池中移除
  │     → 重建新连接指向新 Master
  └── 连接在池中空闲 → 依赖 validationQuery 或 maxIdleTime 来淘汰
```

#### ③ 空闲连接的主动淘汰
需要配置 `testWhileIdle` 来定期检测空闲连接是否还活着：
```YAML
# 这两个参数是关键，默认没有开启 
time-between-eviction-runs: 30s 
# 每 30s 检测一次空闲连接 
min-evictable-idle-time: 60s # 空闲超过 60s 的连接驱逐
```

### 节点宕机后的完整时序

```
t=0s    Master 2 宕机
t=0~30s 集群内部哨兵选举新 Master（期间该分片不可用）
t=30s   新 Master 选举完成
t=30s   业务请求打到旧连接 → MOVED 错误
t=30s   adaptive refresh 触发 → 路由表更新
t=30s   旧连接标记失效移除 → 重建新连接
t=31s   后续请求正常路由到新 Master

期间影响：
  - 选举过程中（0~30s）该分片的写请求会失败
  - 读请求如果配置了 readFrom=REPLICA 则部分可用
```