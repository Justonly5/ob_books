## 介绍
```
HTTP 请求
   ↓
[Filter]
   - TraceIdFilter
   - LoggingFilter
   ↓
DispatcherServlet
   ↓
[Interceptor]
   - AuthInterceptor（登录）
   - PermissionInterceptor（注解权限）
   ↓
Controller
```

Filter 是 Servlet 规范，Interceptor 是 SpringMVC 支持的，在 Filter 里抛出的异常不会被 Spring 的 GlobalExceptionHandler 处理，Interceptor 抛出的异常会被处理。

## Filter
### 实现
```JAVA
public class TraceFilter extends OncePerRequestFilter {
    @Override
    protected void doFilterInternal(HttpServletRequest request,
                                    HttpServletResponse response,
                                    FilterChain filterChain)
            throws ServletException, IOException {
        String traceId = UUID.randomUUID().toString();
        // 放入 MDC（日志框架使用）
        MDC.put("traceId", traceId);
        long start = System.currentTimeMillis();
        try {
            // 需要调用后续的 filter
            filterChain.doFilter(request, response);
        } finally {
            long cost = System.currentTimeMillis() - start;
            System.out.println("traceId=" + traceId +
                    " uri=" + request.getRequestURI() +
                    " cost=" + cost);
            MDC.remove("traceId");
        }
    }
}
```

> OncePerRequestFilter Spring 提供的保证只执行一次的 Filter。

### 注册

一般手动控制注册，通过 order 来控制执行顺序，order 值越小执行越靠前。
> @Component 注解是没法直接注册 Filter 的。
#### FilterRegistrationBean

```JAVA
@Bean
public FilterRegistrationBean<TraceFilter> myFilter() {
    FilterRegistrationBean<TraceFilter> bean = new FilterRegistrationBean<>();
    bean.setFilter(new TraceFilter());
    bean.UrlPattern("/*");
    bean.setOrder(1);
    return bean;
}
```

#### @WebFilter
```JAVA
@WebFilter(filterName = "myFilter", urlPatterns = "/*", order = 1)
public class TraceFilter extends OncePerRequestFilter {}
```