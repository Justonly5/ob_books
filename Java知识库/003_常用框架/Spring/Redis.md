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

## ji'q
```YAML
spring:
  data:
    redis:
      #username: 
      password: ${REDIS_PASS:}
      timeout: 2000ms
      cluster:
        nodes:
          - 192.168.1.1:6379
          - 192.168.1.2:6379
          - 192.168.1.3:6379
          - 192.168.1.4:6379  # replica 节点也可以列进来，客户端自动发现全部
          - 192.168.1.5:6379
          - 192.168.1.6:6379
        max-redirects: 3       # MOVED/ASK 重定向最大次数
      lettuce:
        pool:
          enabled: true
          max-active: 16
          max-idle: 8
          min-idle: 2
          max-wait: 1000ms
        cluster:
          refresh:
            adaptive: true     # 感知到拓扑变化时自动刷新路由表
            period: 30s        # 定期主动刷新
```

# RedisConfig 配置类