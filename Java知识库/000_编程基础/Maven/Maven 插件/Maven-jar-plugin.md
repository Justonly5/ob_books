## maven-jar-plugin 系统介绍

### 核心作用

`maven-jar-plugin` 负责把编译后的 `.class` 文件和资源文件打包成 jar。它绑定在 Maven 的 `package` 阶段，`mvn package` 时自动触发，不需要手动调用。

### 默认行为

不做任何配置时，插件的默认行为：

```
输入：target/classes/（编译产物）
输出：target/${artifactId}-${version}.jar

包含：
  target/classes/ 下的所有文件（.class + resources 下的所有文件）

MANIFEST.MF 默认内容：
  Manifest-Version: 1.0
  Built-By: ${user.name}
  Build-Jdk: ${java.version}
  Created-By: Maven Jar Plugin
  （没有 Main-Class，不可直接执行）
```

> maven-jar 插件zhi
---

### 详细配置项

#### 输出控制