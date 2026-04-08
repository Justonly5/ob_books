在 Kubernetes 里，**Ingress** 和 **Ingress Controller** 是一组配套机制，用来解决「**集群外流量如何优雅进入集群内部服务**」的问题。你可以把它理解为：**七层（HTTP/HTTPS）网关 + 控制平面驱动的数据平面实现**。

⚠️ **重点：Ingress 本身不会生效！**

  Ingress 只是一个“规则配置”，**真正执行这些规则的是 Ingress Controller**

**类比理解

|**组件**|**类比**|
|---|---|
|Ingress|配置文件|
|Ingress Controller|Nginx / 网关程序|
👉 **Ingress = 路由规则定义**

👉 **Ingress Controller = 实际执行流量转发的网关**

# **一、Ingress 是什么（定义规则）**
**Ingress** 是一个 **Kubernetes API 对象（资源）**，本质上是一个**声明式的路由规则集合**，描述：
👉 “外部请求进来后，应该如何转发到集群内的 Service”。
## 示例
```YAML
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: demo-ingress
spec:
  rules:
  - host: example.com
    http:
      paths:
      - path: /api
        pathType: Prefix
        backend:
          service:
            name: api-service
            port:
              number: 80
```

## 能做啥
Ingress 主要解决：
### **1️⃣ 域名路由（Host-based routing）**
```TEXT
api.example.com  -> api-service
www.example.com  -> web-service
```
### **2️⃣ 路径路由（Path-based routing）**
```
/example.com/api  -> api-service
/example.com/web  -> web-service
```
### **3️⃣ TLS / HTTPS**
```YAML
tls:
- hosts:
  - example.com
  secretName: tls-secret
```
### **4️⃣ 统一入口**

👉 替代多个 NodePort / LoadBalancer
👉 提供一个统一的入口网关

# IngressController
## **工作原理**

Ingress Controller 会：
1. **监听 Kubernetes API（watch Ingress 资源）**
2. **读取规则**
3. **动态生成代理配置（如 Nginx / Envoy）**
4. **接管外部流量并转发**
5. 多个 controller 用 IngressClass 区分
## **常见 Ingress Controller**

### **1️⃣ NGINX Ingress（最常用）**

- 基于 NGINX    
- 社区版：kubernetes/ingress-nginx
特点：
- 成熟稳定
- 功能丰富（rewrite、限流、认证等）
- 最广泛使用