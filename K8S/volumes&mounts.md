
## 总结
**volumes 定义“存储从哪里来”，volumeMounts 定义“挂到容器哪里去”。volumes 解决“数据从哪来”，volumeMounts 解决“数据到哪去”，两者通过 name 绑定。


```
Pod
 ├── volumes（存储定义）
 └── containers
      └── volumeMounts（挂载到容器）
```

👉 **volume 是 Pod 级别的，volumeMount 是容器(containers)级别的**

## volumes

```YAML
volumes:
  - name: config-vol
    configMap:
      name: my-config
```

👉 表达的是：“我这个 Pod 需要一个叫 config-vol 的存储，它的数据来源是 ConfigMap: my-config”

**常见类型：**

| **类型**                | **用途**         |
| --------------------- | -------------- |
| emptyDir              | 临时存储（Pod 生命周期） |
| configMap             | 配置文件           |
| secret                | 敏感数据           |
| persistentVolumeClaim | 持久化存储          |
| hostPath              | 宿主机目录（慎用）      |
## volumeMounts（挂载行为）

```yml
volumeMounts:
  - name: config-vol
    mountPath: /etc/config
```

👉 表达的是：“把 config-vol 挂载到容器的 /etc/config 目录”

**关键点**
- 必须引用 volumes 里的 name
- 可以挂多个位置
- 每个容器可以不同挂载方式

## **关键特性**

**✅ 1. 一个 volume 可以被多个容器挂载**
```yaml
containers:
  - name: app1
    volumeMounts:
      - name: shared
        mountPath: /data

  - name: app2
    volumeMounts:
      - name: shared
        mountPath: /cache
```
**✅ 2. 挂载可以只读**
```yaml
readOnly: true
```
**✅ 3. 可以只挂载部分文件（subPath）**
```YAML
volumeMounts:
  - name: config-vol
    mountPath: /app/config.yaml
    subPath: config.yaml
```
**⚠️ 4. 挂载会“覆盖”目录（非常容易踩坑）**
