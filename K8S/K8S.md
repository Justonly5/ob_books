## volumes&volumesMount

**volumes 定义“存储从哪里来”，volumeMounts 定义“挂到容器哪里去”**

```
Pod
 ├── volumes（存储定义）
 └── containers
      └── volumeMounts（挂载到容器）
```

👉 **volume 是 Pod 级别的，volumeMount 是容器(containers)级别的**

### volumes

```YAML
volumes:
  - name: config-vol
    configMap:
      name: my-config
```

👉 表达的是：“我这个 Pod 需要一个叫 config-vol 的存储，它的数据来源是 ConfigMap: my-config”

常见类型：

| **类型**                | **用途**         |
| --------------------- | -------------- |
| emptyDir              | 临时存储（Pod 生命周期） |
| configMap             | 配置文件           |
| secret                | 敏感数据           |
| persistentVolumeClaim | 持久化存储          |
| hostPath              | 宿主机目录（慎用）      |
###  volumeMounts（挂载行为）

```YAML
volumeMounts:
  - name: config-vol
    mountPath: /etc/config
```