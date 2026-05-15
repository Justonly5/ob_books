## HttpClient VS OkHttpClient
### 选型核心差异


|           | Apache HttpClient 5                      | OkHttp                            |
| --------- | ---------------------------------------- | --------------------------------- |
| 出身        | Apache 基金会，Java 老牌 HTTP 库                | Square 出品，Android/移动端起家           |
| 协议支持      | HTTP/1.1、HTTP/2（5.x）                     | HTTP/1.1、HTTP/2、WebSocket连接池      |
| 连接池       | PoolingHttpClientConnectionManager       | 内置 `ConnectionPool`，开箱即用          |
| 异步支持      | 支持（`CloseableHttpAsyncClient`）           | 支持（`Call.enqueue`）                |
| 拦截器       | `HttpRequestInterceptor`                 | 拦截器链，更简洁                          |
| 重试机制      | 需要手动配置                                   | 内置自动重试                            |
| WebSocket | 不支持                                      | 原生支持                              |
| 包大小       | 较大                                       | 较小                                |
| Spring 集成 | `HttpComponentsClientHttpRequestFactory` | `OkHttp3ClientHttpRequestFactory` |
| 社区活跃度     | 稳定维护                                     | 非常活跃                              |

---

### 选型决策树

```
需要 WebSocket？
  └── 是 → OkHttp（唯一选择）

Android / 移动端项目？
  └── 是 → OkHttp（官方推荐）

Spring Boot 企业项目？
  ├── 已有 Apache 生态（Hadoop/ES/Solr）→ HttpClient（统一依赖）
  └── 新项目 / 无历史包袱 → OkHttp（配置更简单）

需要精细控制 TLS / 代理 / 连接策略？
  └── HttpClient（配置项更丰富）

团队熟悉度优先？
  └── 选团队更熟的那个，两者差距不大
```

## 依赖
```XML
<!-- HTTP 客户端 --> 
<dependency>
	<groupId>org.apache.httpcomponents.client5</groupId>
	<artifactId>httpclient5</artifactId> 
</dependency> 
<!-- 监控 --> 
<dependency> 
	<groupId>org.springframework.boot</groupId> 
	<artifactId>spring-boot-starter-actuator</artifactId> 
</dependency> 
<dependency> 
	<groupId>io.micrometer</groupId> 
	<artifactId>micrometer-registry-prometheus</artifactId> 
</dependency>
```
## 配置

### HttpClient

```JAVA
@Configuration
public class HttpClientConfig {

    @Bean
    public CloseableHttpClient httpClient() {
        // 连接池管理器
        PoolingHttpClientConnectionManager cm =
            new PoolingHttpClientConnectionManager();
        cm.setMaxTotal(200);          // 总连接上限
        cm.setDefaultMaxPerRoute(50); // 每个 host 最大连接

        // 请求配置（超时）
        RequestConfig requestConfig = RequestConfig.custom()
            .setConnectTimeout(Timeout.ofSeconds(3))    // 建立连接超时
            .setResponseTimeout(Timeout.ofSeconds(5))   // 等待响应超时
            // 从池中获取连接超时
            .setConnectionRequestTimeout(Timeout.ofSeconds(2)) 
            .build();

        return HttpClients.custom()
            .setConnectionManager(cm)
            .setDefaultRequestConfig(requestConfig)
            // 空闲连接清理（30s 一次，驱逐超过 60s 的空闲连接）
            .evictExpiredConnections()
            .evictIdleConnections(TimeValue.ofSeconds(60))
            // 请求拦截器（加日志、traceId 等）
            .addRequestInterceptorFirst((request, entity, context) -> {
	            
	            request.addHeader("X-Trace-Id", MDC.get("traceId")); 
            })
            .build();
    }

    // 注入到 RestTemplate
    @Bean
    public RestTemplate restTemplate(CloseableHttpClient httpClient) {
        HttpComponentsClientHttpRequestFactory factory =
            new HttpComponentsClientHttpRequestFactory(httpClient);
        return new RestTemplate(factory);
    }
}
```

### OkHttpClient

```JAVA
@Configuration
public class OkHttpConfig {

    @Bean
    public OkHttpClient okHttpClient() {
        return new OkHttpClient.Builder()
            // 连接池
            .connectionPool(new ConnectionPool(
                50,              // 最大空闲连接数
                5,               // 空闲连接存活时间
                TimeUnit.MINUTES
            ))
            // 超时
            .connectTimeout(3, TimeUnit.SECONDS)
            .readTimeout(5, TimeUnit.SECONDS)
            .writeTimeout(5, TimeUnit.SECONDS)
            // 自动重试（默认开启，可关闭）
            .retryOnConnectionFailure(true)
            // 拦截器：日志 + traceId
            .addInterceptor(chain -> {
                Request request = chain.request().newBuilder()
                    .addHeader("X-Trace-Id",
                        MDC.get("traceId") != null
                            ? MDC.get("traceId") : "")
                    .build();
                // 打印请求耗时
                long start = System.currentTimeMillis();
                Response response = chain.proceed(request);
                long duration = System.currentTimeMillis() - start;
                log.info("HTTP {} {} → {} ({}ms)",
                    request.method(), request.url(),
                    response.code(), duration);
                return response;
            })
            .build();
    }

    @Bean
    public RestTemplate restTemplate(OkHttpClient okHttpClient) {
        OkHttp3ClientHttpRequestFactory factory =
            new OkHttp3ClientHttpRequestFactory(okHttpClient);
        return new RestTemplate(factory);
    }
}
```