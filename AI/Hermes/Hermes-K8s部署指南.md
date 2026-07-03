# Hermes Agent K8s 部署指南

> 基于 Hermes Agent 官方 Docker 镜像，单 Pod 同时运行 Gateway + Dashboard。

---

## 一、架构概览

```
┌─────────────────────────────────────────────────────────────────────┐
│  K8s Cluster                                                        │
│                                                                      │
│  ┌──────────────────────────────────────────────────────────────┐   │
│  │  Hermes Pod (replicas: 1)                                     │   │
│  │                                                                │   │
│  │  /init (s6-svscan, PID 1)                                     │   │
│  │  ├── main-hermes (s6 占位服务)                                 │   │
│  │  ├── dashboard  (s6 服务) ── hermes dashboard :9119           │   │
│  │  └── 主程序 (CMD) ── hermes gateway run                       │   │
│  │       ├── 消息平台适配器 (飞书/企微/Telegram...)               │   │
│  │       ├── API Server :8642 (OpenAI 兼容)                       │   │
│  │       ├── Webhook :8644                                        │   │
│  │       └── Cron 调度器                                          │   │
│  │                                                                │   │
│  │  本地存储: /opt/data                                           │   │
│  │  ├── config.yaml / .env / auth.json                            │   │
│  │  ├── state.db (SQLite 会话)                                    │   │
│  │  ├── skills/ / sessions/ / logs/ / cron/                       │   │
│  └──────────────────────────────────────────────────────────────┘   │
│                                                                      │
│  sidecar 应用 ──WS──▶ hermes-dashboard:9119/api/ws                  │
│  外部客户端 ──HTTP──▶ hermes-gateway:8642/v1/chat/completions       │
│  消息平台   ──回调──▶ hermes-gateway:8644/webhooks/...              │
└─────────────────────────────────────────────────────────────────────┘
```

### 端口一览

| 端口 | 协议 | 提供方 | 用途 |
|------|------|--------|------|
| 9119 | TCP + WS | Dashboard (s6 伴随服务) | Web UI + WebSocket `/api/ws` |
| 8642 | TCP | Gateway (API Server) | OpenAI 兼容 HTTP API |
| 8644 | TCP | Gateway (Webhook) | Webhook 接收（可选） |

---

## 二、存储规划

### 2.1 单 Pod 无需共享存储

Gateway 和 Dashboard 在同一个容器内运行，共享同一个文件系统，**不需要 CFS / ReadWriteMany**。使用普通 PVC 即可：

```yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: hermes-data-pvc
  namespace: hermes
spec:
  accessModes:
    - ReadWriteOnce       # ← 单 Pod，RWO 即可
  resources:
    requests:
      storage: 50Gi
```

> 如果使用云厂商的 CBS（腾讯云）/ ESSD（阿里云）块存储，直接创建 PVC 即可，无需手动创建 PV。

### 2.2 目录结构

```
/opt/data/                          ← HERMES_HOME
├── config.yaml                     # 主配置
├── .env                            # API Keys / 环境变量
├── auth.json                       # OAuth token + credential pools
├── state.db                        # 会话存储 (SQLite + FTS5)
├── gateway.pid                     # Gateway PID 锁
├── gateway_state.json              # Gateway 运行时状态
├── skills/                         # 技能
├── sessions/                       # 会话转录
├── logs/                           # 日志
│   ├── gateway.log
│   ├── agent.log
│   └── errors.log
├── profiles/                       # 多 Profile（可选）
├── cron/                           # Cron 作业
├── webhook_subscriptions.json      # Webhook 订阅
├── cache/                          # 缓存
└── scripts/                        # Cron 脚本
```

### 2.3 容量建议

| 规模 | 建议容量 |
|------|---------|
| 开发/测试 | 20Gi |
| 生产（少量用户） | 50Gi |
| 生产（多用户/高频） | 100Gi+ |

---

## 三、配置管理

### 3.1 ConfigMap

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: hermes-config
  namespace: hermes
data:
  config.yaml: |
    model:
      default: "deepseek/deepseek-chat"
      provider: "deepseek"

    agent:
      max_turns: 90

    terminal:
      backend: "local"
      timeout: 180

    compression:
      enabled: true
      threshold: 0.50
      target_ratio: 0.20

    security:
      redact_secrets: true

    approvals:
      mode: "smart"

    memory:
      memory_enabled: true
      user_profile_enabled: true

    delegation:
      max_concurrent_children: 3
      max_iterations: 50
```

### 3.2 Secret

```yaml
apiVersion: v1
kind: Secret
metadata:
  name: hermes-secrets
  namespace: hermes
type: Opaque
stringData:
  DEEPSEEK_API_KEY: "sk-xxxxxxxx"
  DASHSCOPE_API_KEY: "sk-xxxxxxxx"

  # API Server（必填，否则 API Server 不启动）
  API_SERVER_KEY: "your-bearer-token-here"

  # 飞书（可选）
  # FEISHU_APP_ID: "cli_xxxxxxxx"
  # FEISHU_APP_SECRET: "xxxxxxxx"

  # 企业微信（可选）
  # WECOM_CORP_ID: "xxxxxxxx"
  # WECOM_SECRET: "xxxxxxxx"
```

---

## 四、安全组配置

### 4.1 端口清单

| 端口 | 协议 | 用途 | 建议访问范围 |
|------|------|------|-------------|
| 9119 | TCP + WS | Dashboard Web UI + WebSocket | 内网 + 反向代理 |
| 8642 | TCP | OpenAI 兼容 API | 内网 + sidecar 应用 |
| 8644 | TCP | Webhook 接收 | 外部回调源（GitHub/GitLab 等） |

### 4.2 K8s NetworkPolicy

```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: hermes-network-policy
  namespace: hermes
spec:
  podSelector:
    matchLabels:
      app: hermes
  policyTypes:
    - Ingress
    - Egress
  ingress:
    # Dashboard — 内网 + Ingress Controller
    - from:
        - namespaceSelector:
            matchLabels:
              name: ingress-nginx
        - ipBlock:
            cidr: 10.0.0.0/8
      ports:
        - protocol: TCP
          port: 9119
    # API Server — sidecar 应用
    - from:
        - podSelector:
            matchLabels:
              app: your-sidecar
        - ipBlock:
            cidr: 10.0.0.0/8
      ports:
        - protocol: TCP
          port: 8642
    # Webhook — 外部回调（可选）
    - from:
        - ipBlock:
            cidr: 0.0.0.0/0
      ports:
        - protocol: TCP
          port: 8644
  egress:
    # LLM API 调用
    - to:
        - ipBlock:
            cidr: 0.0.0.0/0
      ports:
        - protocol: TCP
          port: 443
    # DNS
    - to:
        - namespaceSelector: {}
          podSelector:
            matchLabels:
              k8s-app: kube-dns
      ports:
        - protocol: UDP
          port: 53
```

### 4.3 云安全组（腾讯云为例）

| 方向 | 来源 | 端口 | 协议 | 说明 |
|------|------|------|------|------|
| 入站 | CLB/Ingress 安全组 | 9119 | TCP | Dashboard |
| 入站 | Sidecar 安全组 | 8642 | TCP | API Server |
| 入站 | 0.0.0.0/0（或 webhook 源 IP 段） | 8644 | TCP | Webhook（可选） |
| 出站 | 0.0.0.0/0 | 443 | TCP | LLM API |
| 出站 | 0.0.0.0/0 | 53 | UDP | DNS |

---

## 五、K8s 部署清单

### 5.1 Deployment

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: hermes
  namespace: hermes
  labels:
    app: hermes
spec:
  replicas: 1                    # ← 必须是 1（PID 锁 + 内存状态）
  strategy:
    type: Recreate               # ← 必须 Recreate
  selector:
    matchLabels:
      app: hermes
  template:
    metadata:
      labels:
        app: hermes
    spec:
      securityContext:
        runAsUser: 10000
        runAsGroup: 10000
        fsGroup: 10000

      containers:
        - name: hermes
          image: ghcr.io/nousresearch/hermes-agent:latest
          command: ["/init", "/opt/hermes/docker/main-wrapper.sh"]
          args: ["gateway", "run"]

          env:
            # ── UID/GID ──
            - name: HERMES_UID
              value: "10000"
            - name: HERMES_GID
              value: "10000"

            # ── Dashboard（s6 伴随服务）──
            - name: HERMES_DASHBOARD
              value: "true"
            - name: HERMES_DASHBOARD_HOST
              value: "0.0.0.0"
            - name: HERMES_DASHBOARD_PORT
              value: "9119"

            # ── API Server ──
            - name: API_SERVER_ENABLED
              value: "true"
            - name: API_SERVER_HOST
              value: "0.0.0.0"
            - name: API_SERVER_PORT
              value: "8642"
            - name: API_SERVER_KEY
              valueFrom:
                secretKeyRef:
                  name: hermes-secrets
                  key: API_SERVER_KEY

            # ── 模型 API Keys ──
            - name: DEEPSEEK_API_KEY
              valueFrom:
                secretKeyRef:
                  name: hermes-secrets
                  key: DEEPSEEK_API_KEY
            - name: DASHSCOPE_API_KEY
              valueFrom:
                secretKeyRef:
                  name: hermes-secrets
                  key: DASHSCOPE_API_KEY
              optional: true

            # ── 飞书（可选）──
            # - name: FEISHU_APP_ID
            #   valueFrom:
            #     secretKeyRef:
            #       name: hermes-secrets
            #       key: FEISHU_APP_ID
            # - name: FEISHU_APP_SECRET
            #   valueFrom:
            #     secretKeyRef:
            #       name: hermes-secrets
            #       key: FEISHU_APP_SECRET

            # ── 企业微信（可选）──
            # - name: WECOM_CORP_ID
            #   valueFrom:
            #     secretKeyRef:
            #       name: hermes-secrets
            #       key: WECOM_CORP_ID
            # - name: WECOM_SECRET
            #   valueFrom:
            #     secretKeyRef:
            #       name: hermes-secrets
            #       key: WECOM_SECRET

          ports:
            - name: dashboard
              containerPort: 9119
              protocol: TCP
            - name: api
              containerPort: 8642
              protocol: TCP
            - name: webhook
              containerPort: 8644
              protocol: TCP

          volumeMounts:
            - name: hermes-data
              mountPath: /opt/data
            - name: hermes-config
              mountPath: /opt/data/config.yaml
              subPath: config.yaml

          resources:
            requests:
              memory: "512Mi"
              cpu: "500m"
            limits:
              memory: "2Gi"
              cpu: "2000m"

          livenessProbe:
            httpGet:
              path: /health
              port: 8642
            initialDelaySeconds: 30
            periodSeconds: 30
            timeoutSeconds: 10
            failureThreshold: 3

          readinessProbe:
            httpGet:
              path: /health
              port: 8642
            initialDelaySeconds: 10
            periodSeconds: 10
            timeoutSeconds: 5
            failureThreshold: 3

      volumes:
        - name: hermes-data
          persistentVolumeClaim:
            claimName: hermes-data-pvc
        - name: hermes-config
          configMap:
            name: hermes-config
```

### 5.2 Service

```yaml
apiVersion: v1
kind: Service
metadata:
  name: hermes
  namespace: hermes
spec:
  selector:
    app: hermes
  ports:
    - name: dashboard
      port: 9119
      targetPort: 9119
    - name: api
      port: 8642
      targetPort: 8642
    - name: webhook
      port: 8644
      targetPort: 8644
```

### 5.3 Ingress（可选）

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: hermes-ingress
  namespace: hermes
  annotations:
    nginx.ingress.kubernetes.io/proxy-read-timeout: "3600"
    nginx.ingress.kubernetes.io/proxy-send-timeout: "3600"
    # WebSocket 支持
    nginx.ingress.kubernetes.io/proxy-http-version: "1.1"
    nginx.ingress.kubernetes.io/upstream-hash-by: "$remote_addr"
spec:
  rules:
    - host: hermes.your-domain.com
      http:
        paths:
          - path: /
            pathType: Prefix
            backend:
              service:
                name: hermes
                port:
                  number: 9119
```

---

## 六、部署步骤

```bash
# 1. 创建命名空间
kubectl create namespace hermes

# 2. 创建 PVC（根据存储类调整）
kubectl apply -f - <<EOF
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: hermes-data-pvc
  namespace: hermes
spec:
  accessModes:
    - ReadWriteOnce
  resources:
    requests:
      storage: 50Gi
EOF

# 3. 创建 Secret（先替换实际值）
kubectl apply -f secret.yaml

# 4. 创建 ConfigMap
kubectl apply -f configmap.yaml

# 5. 部署
kubectl apply -f deployment.yaml
kubectl apply -f service.yaml

# 6. 等待就绪
kubectl wait --for=condition=ready pod -l app=hermes -n hermes --timeout=120s

# 7. 验证
kubectl exec -n hermes deploy/hermes -- curl -s http://localhost:8642/health
kubectl exec -n hermes deploy/hermes -- curl -s http://localhost:9119/health
```

---

## 七、关键注意事项

### 7.1 单副本限制

**replicas 必须为 1**。Gateway 使用 PID 文件锁（`gateway.pid`），Dashboard 的 WebSocket session 状态在内存中，均不支持多副本。

### 7.2 更新策略

```yaml
strategy:
  type: Recreate    # 必须！不能 RollingUpdate
```

### 7.3 存储

- 使用普通 `ReadWriteOnce` PVC 即可，**不需要 CFS**
- Gateway 和 Dashboard 在同一容器内，共享本地文件系统
- 如需持久化，使用云厂商块存储（CBS/ESSD）

### 7.4 高可用

当前架构不支持无状态多副本。可用措施：

```yaml
# Pod 反亲和 — 分散到不同节点
affinity:
  podAntiAffinity:
    requiredDuringSchedulingIgnoredDuringExecution:
      - labelSelector:
          matchLabels:
            app: hermes
        topologyKey: kubernetes.io/hostname

# 自动重启
restartPolicy: Always
```

### 7.5 Sidecar 连接 WebSocket

```
ws://hermes.hermes.svc.cluster.local:9119/api/ws?token=<SESSION_TOKEN>
```

Dashboard 启动日志中输出 `_SESSION_TOKEN`，或通过 Ingress 暴露后从浏览器 DevTools 获取。

### 7.6 资源建议

| 环境 | CPU Request | CPU Limit | Memory Request | Memory Limit |
|------|------------|-----------|----------------|--------------|
| 开发/测试 | 250m | 1000m | 256Mi | 1Gi |
| 生产 | 500m | 2000m | 512Mi | 2Gi |

---

## 八、环境变量速查

| 变量 | 默认值 | 说明 |
|------|--------|------|
| `HERMES_HOME` | `/opt/data` | 数据根目录 |
| `HERMES_UID` | `10000` | 运行用户 UID |
| `HERMES_GID` | `10000` | 运行用户 GID |
| `HERMES_DASHBOARD` | - | 设为 `true` 启用 Dashboard（s6 伴随服务） |
| `HERMES_DASHBOARD_HOST` | `0.0.0.0` | Dashboard 绑定地址 |
| `HERMES_DASHBOARD_PORT` | `9119` | Dashboard 端口 |
| `API_SERVER_ENABLED` | `false` | 启用 API Server |
| `API_SERVER_KEY` | - | API Server Bearer Token（必填） |
| `API_SERVER_HOST` | `127.0.0.1` | API Server 绑定地址 |
| `API_SERVER_PORT` | `8642` | API Server 端口 |
| `API_SERVER_CORS_ORIGINS` | - | CORS 域名（逗号分隔） |
| `WEBHOOK_ENABLED` | `false` | 启用 Webhook |
| `WEBHOOK_PORT` | `8644` | Webhook 端口 |
| `WEBHOOK_SECRET` | - | Webhook HMAC Secret |
