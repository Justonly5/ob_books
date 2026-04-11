
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
            .setConnectionRequestTimeout(Timeout.ofSeconds(2)) // 从池中获取连接超时
            .build();

        return HttpClients.custom()
            .setConnectionManager(cm)
            .setDefaultRequestConfig(requestConfig)
            // 空闲连接清理（30s 一次，驱逐超过 60s 的空闲连接）
            .evictExpiredConnections()
            .evictIdleConnections(TimeValue.ofSeconds(60))
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