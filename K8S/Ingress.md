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

>  Ingress 不能跨 namespace 访问 Service。
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

> --controller-class 通过这个指定自己的唯一身份 id
> 推荐格式：`<domain>/<controller-name>`
## **工作原理**

Ingress Controller 会：
1. **监听 Kubernetes API（watch Ingress 资源）**
2. **读取规则**
3. **动态生成代理配置（如 Nginx / Envoy）**
4. **接管外部流量并转发**
通过以下两个方式声明自身身份：
- --ingress-class=nginx # 匹配 annotations 的
- --controller-class=k8s.io/ingress-nginx 匹配 ingressClass
>可以一起用，但**不建议这么做**。建立使用 --controller-class
## **常见 Ingress Controller**

### **1️⃣ NGINX Ingress（最常用）**

- 基于 NGINX    
- 社区版：kubernetes/ingress-nginx
特点：
- 成熟稳定
- 功能丰富（rewrite、限流、认证等）
- 最广泛使用


# IngressClass
> ✅ **IngressClass 是一个标准资源，用来标识“这条 Ingress 规则应该由哪一类 Controller 处理”**

- 属于 networking.k8s.io/v1
    
- 是 **集群级资源（cluster-scoped）**
    
- 解决：**多 Ingress Controller 并存时的归属问题**

## 资源结构
```YAML
apiVersion: networking.k8s.io/v1
kind: IngressClass
metadata:
  # 被 Ingress 引用
  name: nginx
  annotations:
	# 默认类：没有指定 ingressClassName 的 Ingress → 自动归我
    ingressclass.kubernetes.io/is-default-class: "true"  # 可选
spec:
  # 这个 class 属于哪种 controller 实现
  controller: k8s.io/ingress-nginx
  parameters:
    apiGroup: k8s.example.com
    kind: IngressParameters
    name: nginx-params
```
# Ingress 和 Controller 绑定关系

Ingress Controller 主动 watch Ingress → 根据规则筛选“我该处理哪些 Ingress”

👉 **Ingress 通过 ingressClassName → IngressClass → controller 标识，与 Ingress Controller 建立“归属关系”**。
👉 **本质是 Controller 侧筛选，而不是 Ingress 主动注册**。

## IngressClass

```TEXT
Ingress
  spec.ingressClassName = nginx
        ↓
IngressClass
  metadata.name = nginx
  spec.controller = k8s.io/ingress-nginx
        ↓
Ingress Controller
  --controller-class=k8s.io/ingress-nginx
```
- Ingress 中指定 
```YAML
spec:
  ingressClassName: nginx
```
- IngressClass 定义
> ingressClass 没有 namespace
```YAML
apiVersion: networking.k8s.io/v1
kind: IngressClass
metadata:
  name: nginx
spec:
  controller: k8s.io/ingress-nginx
```
- Controller 启动参数
Ingress Controller 启动时会声明自己是谁：
```YAML
args:
  - --controller-class=k8s.io/ingress-nginx # 匹配 ingressClass 
```

## Annotation

Ingress 设置 annotations:
```YAML
metadata:
  annotations:
    kubernetes.io/ingress.class: nginx
```

Controller 启动参数：
```YAML
args:
  - --ingress-class=nginx # 匹配 annotations 的
  # - --watch-ingress-without-class=true   # 👈 关键 是否匹配 annotation 没有指定 ingress.class 的 ingress。
```