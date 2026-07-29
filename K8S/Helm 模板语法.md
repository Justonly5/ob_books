---
tags:
  - Helm
  - K8s
title: Helm 模板语法
created: 2026-07-29
---


Helm 的模板语法 是基于 Go Template + Sprig 函数库 + Helm 内置对象(Values、Chart、Release、Capabilities)/函数。

## 模版表达式
`{{ }}` 表示执行模板。

例如：
```YAML
image: {{ .Values.image.repository }}

--- values.yaml 
image:
  repository: nginx
```
渲染后的结果为：
```YAML
image: nginx
```

## 四大内置对象
### Values
核心来自于 `Values.yaml` 文件。

### Release
当前 Release 信息。
常用的：
```
.Release.Name
.Release.Namespace
.Release.Revision
.Release.Service
```

### Chart
Chart 自己的信息。来源 `Chart.yaml`。
```YAML
{{ .Chart.Name }}

{{ .Chart.Version }}

{{ .Chart.AppVersion }}
```
### Capabilities
获取 K8s 信息。

## 变量
定义：
```YAML
{{- $name := .Release.Name }}
```

使用：
```YAML
metadata:
  name: {{ $name }}
```

变量可以保存任何对象。

## 条件判断

