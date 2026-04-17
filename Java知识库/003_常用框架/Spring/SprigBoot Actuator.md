
## 依赖引入

```XML
<dependency>  
    <groupId>org.springframework.boot</groupId>  
    <artifactId>spring-boot-starter-actuator</artifactId>  
</dependency>

```

## 基础配置

```YAML
management:
  # 配置 actuator 独立端口
  server:
    port: 8081
  ## 入口管理，主要是暴露、路径、访问范围、按 web/jmx 统一控制
  endpoints:
    web:
      # 配置 actuator 独立的 context-path
      base-path: /actuator
      # 
      exposure:
        # 暴露的 endpoint
        include: health,info,metrics,prometheus
        exclude: env,beans
      # 某个 endpoint 改路径例如把 /actuator/health 改成 /actuator/healthcheck
      path-mapping:
        health: healthcheck
  # 端点本身的行为，主要是某个端点自己的功能参数
  endpoint:
    # 配置 health 端点的具体情况
    health:
      show-details: when-authorized
      show-components: when-authorized
      # k8s 环境会默认开启。为 readiness、liveness
      probes:
        enabled: true
        add-additional-paths: true
      group:
        readiness:
          include: readinessState,db,redis
        liveness:
          include: livenessState

    shutdown:
      access: none
  # 配置 “某个健康检查项开不开”
  health:
    defaults:
      enabled: true
    redis:
      enabled: true
    db:
      enabled: true

```
## Health 
