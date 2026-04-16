
## 目录结构

```
myapp/
├── Chart.yaml
├── values.yaml
└── templates/
    ├── deployment.yaml
    ├── service.yaml
    ├── configmap.yaml
    └── _helpers.tpl
```

```YAML
--- chat.yaml
apiVersion: v2
name: myapp
description: A Helm chart for my app
type: application
version: 0.1.0
appVersion: "1.0.0"

--- values.yaml
replicaCount: 2

image:
  repository: nginx
  tag: "1.27.0"
  pullPolicy: IfNotPresent

service:
  type: ClusterIP
  port: 80
  targetPort: 80

config:
  appName: myapp
  logLevel: info

resources:
  requests:
    cpu: 100m
    memory: 128Mi
  limits:
    cpu: 500m
    memory: 512Mi

--- _helpers.tpl 定义一些公共名字的函数

{{- define "myapp.name" -}}
myapp
{{- end }}

{{- define "myapp.fullname" -}}
{{ .Release.Name }}-{{ include "myapp.name" . }}
{{- end }}

    
--- configmap.yaml

apiVersion: v1
kind: ConfigMap
metadata:
  name: {{ include "myapp.fullname" . }}-config
data:
  APP_NAME: {{ .Values.config.appName | quote }}
  LOG_LEVEL: {{ .Values.config.logLevel | quote }}
  
--- service.yaml

apiVersion: v1
kind: Service
metadata:
  name: {{ include "myapp.fullname" . }}
spec:
  type: {{ .Values.service.type }}
  selector:
    app: {{ include "myapp.fullname" . }}
  ports:
    - port: {{ .Values.service.port }}
      targetPort: {{ .Values.service.targetPort }}
      protocol: TCP
      
--- deployment.yaml

apiVersion: apps/v1
kind: Deployment
metadata:
  name: {{ include "myapp.fullname" . }}
spec:
  replicas: {{ .Values.replicaCount }}
  selector:
    matchLabels:
      app: {{ include "myapp.fullname" . }}
  template:
    metadata:
      labels:
        app: {{ include "myapp.fullname" . }}
    spec:
      containers:
        - name: app
          image: "{{ .Values.image.repository }}:{{ .Values.image.tag }}"
          imagePullPolicy: {{ .Values.image.pullPolicy }}
          ports:
            - containerPort: {{ .Values.service.targetPort }}
          envFrom:
            - configMapRef:
                name: {{ include "myapp.fullname" . }}-config
          resources:
            {{- toYaml .Values.resources | nindent 12 }}
```


## 部署步骤

假设你当前目录在 myapp/ 的上一层。

### 测试模板

```BASH
helm template test-myapp ./myapp
```

这一步不会真正部署，只会把最终 YAML 打印出来。
### 正式安装

```BASH
helm install test-myapp ./myapp -n namespace1
```

或者直接使用 
```BASH
helm update --install test-myapp ./myapp -n namespace1
```

这里：

- test-myapp 是 release 名称
    
- ./myapp 是 chart 路径
    
- -n namespace1 是命名空间

### 查看资源

```BASH
kubectl get deploy,svc,cm -n namespace1
```

### 更新部署

```BASH
helm upgrade test-myapp ./myapp -n namespace1
```

### 删除

```BASH
helm uninstall test-myapp -n namespace1
```

### 按环境部署

values 文件可以按环境区分，例如：

```
values-dev.yaml
values-prod.yaml
```

```BASH
helm install test-myapp ./myapp -n default -f values-dev.yaml

helm upgrade test-myapp ./myapp -n default -f values-dev.yaml
```

### 常用调试命令

#### 看 release 列表

```BASH
helm list -n default
```
#### 看渲染结果

```BASH
helm get manifest test-myapp -n default
```

#### 看values

```BASH
helm get values test-myapp -n default
```
#### 看状态

```BASH
helm status test-myapp -n default
```