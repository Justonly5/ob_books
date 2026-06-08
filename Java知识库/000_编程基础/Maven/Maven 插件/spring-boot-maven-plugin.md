## spring-boot-maven-plugin 系统介绍

### 核心作用

Spring Boot 官方提供的 Maven 插件，主要做三件事：

```
1. repackage  把普通 jar 重新打包成可执行 fat jar 或 thin jar
2. run        直接用 Maven 启动 Spring Boot 应用（开发用）
3. build-info 生成构建信息，供 Actuator /info 端点展示
```

### 默认行为

不做任何配置时：


```xml
<!-- Spring Boot 项目默认继承这个插件 -->
<plugin>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-maven-plugin</artifactId>
</plugin>
```

```
mvn package 时自动执行 repackage：
  输入：maven-jar-plugin 打出的普通 jar
  输出：
    app.jar          ← fat jar，含所有依赖，Main-Class=JarLauncher
    app.jar.original ← 原始普通 jar 的备份
  
  MANIFEST.MF：
    Main-Class: org.springframework.boot.loader.JarLauncher
    Start-Class: com.example.Application
    Spring-Boot-Version: 3.x.x
    Spring-Boot-Classes: BOOT-INF/classes/
    Spring-Boot-Lib: BOOT-INF/lib/
```

---

### 三种 layout 模式

layout 决定了用哪个启动器，是最核心的配置项：


```xml
<configuration>
    <layout>JAR</layout>   <!-- 默认值 -->
</configuration>
```

|layout|启动器|依赖位置|支持 loader.path|
|---|---|---|---|
|`JAR`|`JarLauncher`|`BOOT-INF/lib/`|否|
|`WAR`|`WarLauncher`|`WEB-INF/lib/`|否|
|`ZIP`|`PropertiesLauncher`|外部目录|是|
|`NONE`|无|无|否|

```
JAR  → 标准 fat jar，最常用，开箱即用
ZIP  → thin jar 场景，依赖外置，loader.path 动态指定
NONE → 只重新打包结构，不注入启动器，通常不用
```

---

### 详细配置项

#### 基础配置


```xml
<plugin>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-maven-plugin</artifactId>
    <configuration>
        <!-- 是否跳过所有操作，默认 false -->
        <skip>false</skip>
        <!-- 输出目录，默认和 maven-jar-plugin 一致（target/）-->
        <outputDirectory>${project.build.directory}/deploy/lib</outputDirectory>
        <!-- 输出文件名，默认继承 maven-jar-plugin 的 finalName -->
        <finalName>app</finalName>
        <!-- 启动器类型，默认 JAR -->
        <layout>ZIP</layout>
        <!-- 给 repackage 产物加后缀，避免覆盖原始 jar -->
        <!-- 加了 classifier 后：app.jar（原始）+ app-exec.jar（repackage）-->
        <classifier>exec</classifier>
        <!-- 主类，不配置时插件自动扫描 @SpringBootApplication -->
        <mainClass>com.example.Application</mainClass>
    </configuration>
</plugin>
```

#### 依赖排除配置


```xml
<configuration>
    <!-- 排除特定依赖不打进 fat jar -->
    <excludes>
        <exclude>
            <groupId>org.projectlombok</groupId>
            <artifactId>lombok</artifactId>
        </exclude>
        <exclude>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-devtools</artifactId>
        </exclude>
    </excludes>

    <!-- 排除某个 groupId 下所有依赖 -->
    <excludeGroupIds>
        org.projectlombok,
        com.google.code.findbugs
    </excludeGroupIds>

    <!-- 排除某些 artifactId -->
    <excludeArtifactIds>
        lombok,
        findbugs
    </excludeArtifactIds>

    <!-- 排除哪些 scope 的依赖，默认排除 test 和 provided -->
    <!-- runtime,compile 都会打进去 -->
</configuration>
```

#### ZIP 模式（thin jar）专属配置


```xml
<configuration>
    <layout>ZIP</layout>

    <!--
        layout=ZIP 时，哪些依赖打进 jar，哪些依赖走 loader.path 外置
        不配置则所有依赖都外置（完全 thin）
        配置了则指定的依赖打进 jar，其余外置
    -->
    <requiresUnpack>
        <!-- 某些 jar 需要解压才能运行（如 JNI 库）-->
        <dependency>
            <groupId>org.xerial</groupId>
            <artifactId>sqlite-jdbc</artifactId>
        </dependency>
    </requiresUnpack>
</configuration>
```

#### build-info 配置


```xml
<executions>
    <!-- 生成 build-info.properties，供 Actuator /info 展示 -->
    <execution>
        <id>build-info</id>
        <goals>
            <goal>build-info</goal>
        </goals>
        <configuration>
            <additionalProperties>
                <encoding>${project.build.sourceEncoding}</encoding>
                <java.version>${java.version}</java.version>
                <git.branch>${git.branch}</git.branch>
            </additionalProperties>
        </configuration>
    </execution>
</executions>
```

访问 `/actuator/info` 返回：


```json
{
  "build": {
    "version": "1.0.0",
    "artifact": "app",
    "name": "my-app",
    "time": "2024-01-01T10:00:00.000Z",
    "group": "com.example",
    "java": { "version": "17" },
    "git": { "branch": "main" }
  }
}
```

---

### repackage 的内部结构

repackage 执行后，jar 内部结构：

```
app.jar（fat jar，layout=JAR）
  ├── META-INF/
  │     └── MANIFEST.MF
  ├── BOOT-INF/
  │     ├── classes/           ← 项目自身的 .class 和 resources
  │     │     ├── com/example/Application.class
  │     │     └── application.yml
  │     ├── lib/               ← 所有依赖 jar
  │     │     ├── spring-boot-3.x.jar
  │     │     └── mysql-connector.jar
  │     └── classpath.idx      ← 依赖加载顺序索引（3.x 新增）
  └── org/springframework/boot/loader/
        ├── JarLauncher.class  ← 启动器本身
        └── ...
```

```
app.jar（thin jar，layout=ZIP）
  ├── META-INF/
  │     └── MANIFEST.MF        ← Main-Class=PropertiesLauncher
  ├── BOOT-INF/
  │     └── classes/           ← 只有项目自身的类，没有 lib/
  └── org/springframework/boot/loader/
        ├── PropertiesLauncher.class
        └── ...
```

---

### 与 maven-jar-plugin 的执行关系

```
mvn package 执行顺序：

phase: compile
  → maven-compiler-plugin:compile
      编译 .java → .class

phase: package
  → maven-jar-plugin:jar
      打出普通 jar（target/app.jar）

  → maven-dependency-plugin:copy-dependencies
      拷贝依赖到 extlib/

  → spring-boot-maven-plugin:repackage
      读取 maven-jar-plugin 的产物
      重新打包注入启动器
      原始 jar 重命名为 app.jar.original
      输出新的 app.jar
```

这就是为什么加了 `classifier` 之后能同时保留两个 jar：


```xml
<classifier>exec</classifier>
```

```
maven-jar-plugin  → app.jar        ← 普通 thin jar，不被覆盖
repackage         → app-exec.jar   ← 可执行 jar，含启动器
```

---

### 各场景最佳配置

#### 场景一：标准 fat jar（最常见）


```xml
<plugin>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-maven-plugin</artifactId>
    <executions>
        <execution>
            <goals>
                <goal>repackage</goal>
                <goal>build-info</goal>
            </goals>
        </execution>
    </executions>
    <configuration>
        <!-- 排除编译期工具，不需要打进运行时 -->
        <excludes>
            <exclude>
                <groupId>org.projectlombok</groupId>
                <artifactId>lombok</artifactId>
            </exclude>
        </excludes>
    </configuration>
</plugin>
```

#### 场景二：thin jar + 外置依赖


```xml
<plugin>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-maven-plugin</artifactId>
    <executions>
        <execution>
            <goals>
                <goal>repackage</goal>
            </goals>
        </execution>
    </executions>
    <configuration>
        <layout>ZIP</layout>
        <outputDirectory>${project.build.directory}/deploy/lib</outputDirectory>
        <finalName>app</finalName>
        <!-- 加 classifier 避免覆盖 maven-jar-plugin 的产物 -->
        <classifier>exec</classifier>
    </configuration>
</plugin>
```

启动：


```bash
java -Dloader.path=../conf,extlib \
     -jar lib/app-exec.jar \
     --spring.config.location=file:../conf/
```

#### 场景三：跳过 repackage（只用 maven-jar-plugin 打 thin jar）


```xml
<plugin>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-maven-plugin</artifactId>
    <configuration>
        <skip>true</skip>
    </configuration>
</plugin>
```

依赖通过 MANIFEST `Class-Path` 加载，不需要 `PropertiesLauncher`。

---

### 常见问题

**问：为什么 java -jar 报 ClassNotFoundException，但 IDE 里能跑？**

```
IDE 跑：classpath 包含所有依赖 jar（IDE 自动管理）
java -jar：只读 MANIFEST 的 Class-Path，没配置就没有依赖
解决：用 spring-boot-maven-plugin repackage 打 fat jar
```

**问：repackage 后 jar 很大，CI 上传很慢怎么办？**

```
换 layout=ZIP + 依赖外置（thin jar 方案）
业务代码 jar 只有几十 KB
依赖层单独管理，不频繁变动不需要重新上传
```

**问：多模块项目，子模块不需要打可执行 jar 怎么办？**


```xml
<!-- 子模块 pom.xml 里跳过 repackage -->
<plugin>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-maven-plugin</artifactId>
    <configuration>
        <skip>true</skip>
    </configuration>
</plugin>
```
