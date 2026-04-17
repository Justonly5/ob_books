
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
  # 入口管理，主要是暴露、路径、访问范围、
  # 管“一批端点怎么暴露、挂在哪、走 web 还是 jmx”
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
      # 是否显示：never / when-authorized / always
      show-details: when-authorized
      show-components: when-authorized
      
      probes:
        # 开启 liveness / readiness。k8s 环境会默认开启
        enabled: true
        # 额外暴露 /livez 和 /readyz
        add-additional-paths: true
      group:
        # readiness 要包含哪些检查项
        readiness:
          include: readinessState,db,redis
        # liveness 包含的检查项。
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
## port

当 actuator 开启独立端口后，

不开启独立端口时：

- management.server.port 和 server.port 相同
- Actuator 和业务接口在**同一个 application context**
- 也在**同一个 Web 服务器实例**里处理请求。

业务代码里的 `Filter`、`Interceptor` 会对所有的接口起作用。

开启独立端口时：

- management.server.port 不同于 server.port
- 会创建**单独的 management context**
- 这个 management context 是主应用 context 的**child context**
- 对 servlet 应用来说，会有**单独的 Web server** 在管理端口监听

> 对于开启了独立端口，如果有需要使用 filter、interceptor 务必需要确认是否生效，不能默认是生效的！！！
## Health 

### 机制

Spring Boot 的 health 接口，本质上是 Actuator 暴露出来的一个 HealthEndpoint。它不是写死只返回 "UP"，而是会去收集一组 HealthContributor 的结果，再聚合成最终状态。

**它怎么实现**  
Spring Boot 启动时，会把容器里的健康检查贡献者放进 HealthContributorRegistry。这里的 contributor 分两类：

- HealthIndicator  
    直接做一次健康检查，返回 UP、DOWN、OUT_OF_SERVICE、UNKNOWN
- CompositeHealthContributor  
    下面再挂一组子检查，形成树状结构

当请求 /actuator/health 时，HealthEndpoint 会递归遍历这些 contributor，逐个执行检查，然后用 StatusAggregator 把结果合并成整体状态。

health 会**对依赖组件探活**，但不是“自动扫描所有依赖”，而是：

- 只有 Spring Boot 内置支持的组件，或者你自己注册的 HealthIndicator
- 并且通常要求相关 bean / classpath 存在，才会自动装配

官方文档列了很多内置检查项，比如：
- db
- redis
- mongo
- rabbit
- jms
- ldap
- mail
- diskspace
- ssl
自定义的：
```JAVA
@Component
public class XxxHealthIndicator implements HealthIndicator {
    @Override
    public Health health() {
        try {
            // 调你的依赖
            return Health.up().withDetail("xxx", "ok").build();
        } catch (Exception e) {
            return Health.down(e).build();
        }
    }
}
```

**这些“探活”具体怎么做**  
不是单纯看 bean 在不在，而是会真正调用对应客户端做一次轻量检查。几个典型例子：

1. DataSourceHealthIndicator
    - 先拿数据库连接
    - 读取数据库产品名
    - 如果你配置了校验 SQL，就执行那条 SQL
    - 否则调用 JDBC 的 Connection.isValid(0)

2. RedisHealthIndicator
    - 从 RedisConnectionFactory 拿连接
    - 集群模式下读 cluster info
    - 非集群模式下执行 INFO
总结：health 的探活是会调用相应的组件，去执行简单的命令的。

**有几个很容易混淆的点**
> ==不要把health当做 liveness、readiness ==
1. health 不等于 liveness/readiness  
    Spring Boot 官方明确建议：
    - liveness 不要依赖外部系统
    - readiness 是否检查外部依赖，要谨慎决定
### probes
Spring Boot Actuator 里专门给 K8s probe 用的配置。

`management.endpoint.health.probes.enabled=true`  意思是：启用探针相关的健康组。

启用后会有这两个健康组接口：

- /actuator/health/liveness
- /actuator/health/readiness

它们不是全局 /actuator/health，而是专门给 K8s 的：

- liveness：看应用自己是不是“还活着”
- readiness：看应用是不是“已经可以接流量”

补一句：

- 在一些 Spring Boot 版本/场景里，运行在 Kubernetes 环境时会自动开启
- 显式写成 true 的好处是：行为更明确，非 K8s 环境本地调试也能直接访问

`management.endpoint.health.probes.add-additional-paths=true`  
意思是：除了 actuator 路径外，再额外在主业务端口暴露更适合 K8s 的探针路径。

开启后通常会额外出现：

- /livez
- /readyz

也就是说同时能访问：

- /actuator/health/liveness
- /actuator/health/readiness
- /livez
- /readyz

这个配置特别适合下面这种情况：

- 你把 actuator 放到了独立管理端口
- 但 K8s 还是想探测主应用端口是否真的可用

因为官方也提醒过：如果 probe 只打管理端口，可能 actuator 正常，但主业务端口其实已经不行了。

## Tips

### 相同 uri

Actuator 的接口和 `@RequestMapping` 配置的 controller 接口是不一样的，因为 Actuator 用的是单独的 HandlerMapping。假设两个接口 uri 相同，那在**同一个端口**上就会有路径冲突。这种冲突和普通两个 @RequestMapping("/health") 不完全一样，**不一定是启动直接报错**，实际效果可能是：
- 某一个接口把另一个“遮住”
- 访问结果变成 404 / 405
- 某些版本下行为还会有差异 
> 测试结果为 自定义接口被 actuator 接口覆盖。

