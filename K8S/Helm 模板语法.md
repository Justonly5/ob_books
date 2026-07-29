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
### IF

```YAML
{{ if .Values.debug }} 
DEBUG 
{{ else }} 
INFO 
{{ end }}
```

| 函数    | 含义                    | 示例                       |
| ----- | --------------------- | ------------------------ |
| `eq`  | 等于                    | `eq .Values.env "prod"`  |
| `ne`  | 不等于                   | `ne .Values.env "test"`  |
| `gt`  | 大于                    | `gt .Values.port 80`     |
| `lt`  | 小于                    | `lt .Values.port 1000`   |
| `ge`  | 大于等于                  | `ge .Values.replicas 3`  |
| `le`  | 小于等于                  | `le .Values.replicas 10` |
| `and` | 与                     | `and (eq ...) (...)`     |
| `or`  | 或                     | `or (eq ...) (...)`      |
| `not` | 非，适用于 null "" 未定义 三种。 | `not .Values.enabled`    |


写法都是 
```YAML
{{ if 表达式 .Values.debug "比较的值" }}

{{ end }} 

---
{{- if or
      (eq .Values.profile "dev")
      (eq .Values.profile "test")
}}
```

### with
减少层级。

```YAML
{{ with .Values.image }}

{{ .repository }}

{{ .tag }}

{{ end }}
```

这里：`.` 已经变成 `.Values.image`

### include
配合 `_helpers.tpl`。`include` 返回的是字符串，如果要返回对象使用 

```YAML
{{ include "demo.labels" . | fromYaml }}
```

**`_helpers.tpl`（或者 `_xxx.tpl`）里面定义的不是变量，而是"命名模板（Named Template）"。**

**模板内部定义的变量（`$var := ...`）只能在该模板内部使用，外部不能直接引用。**

定义：
```YAML
## 定义模板，在 deployment 里可以通过 {{ include "demo.fullname" . }} 引用
{{ define "demo.fullname" }}

{{ .Release.Name }}

{{ end }}

---
## 定义变量 只能在demo.labels里使用
{{- define "demo.labels" -}} 
{{- $app := .Chart.Name }} 
app: {{ $app }} 
{{- end -}}
```

调用：
```YAML
metadata:
  name: {{ include "demo.fullname" . }}
```

