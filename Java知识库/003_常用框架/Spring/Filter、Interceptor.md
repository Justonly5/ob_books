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

