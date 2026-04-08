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
> @Component + @Order 注解是没法控制 Filter 的执行顺序，建议手动注册。
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

## Interceptor

### 实现
```JAVA
public class AuthInterceptor implements HandlerInterceptor {
    @Override
    public boolean preHandle(HttpServletRequest request,
                             HttpServletResponse response,
                             Object handler) throws Exception {
        String token = request.getHeader("Authorization");
        if (token == null || !isValid(token)) {
            response.setStatus(401);
            return false;
        }
        return true;
    }
    private boolean isValid(String token) {
        return true;
    }
}
```

### 注册

>Filter：@Component 会自动注册（Spring Boot 帮你挂到 Servlet 容器）
>Interceptor：@Component 不会生效，必须手动注册到 WebMvcConfigurer
>此处的 pattern 不需要考虑 context-path，Spring 会自动处理。
>**Interceptor 的路径匹配基于“去掉 context-path 后的路径”，所以 addPathPatterns 永远写相对路径即可。**

```JAVA
@Configuration
public class WebConfig implements WebMvcConfigurer {
    @Override
    public void addInterceptors(InterceptorRegistry registry) {
        registry.addInterceptor(new AuthInterceptor())
                .addPathPatterns("/api/**").excludePatterns("/health");
        registry.addInterceptor(new PermissionInterceptor())
                .addPathPatterns("/api/**");
    }
}
```

