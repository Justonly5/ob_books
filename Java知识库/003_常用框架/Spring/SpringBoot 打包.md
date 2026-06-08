
## classpath、load.path、spring.config.location
![[Pasted image 20260608094858.png]]

### 一、classpath

classpath 是 JVM 的概念，定义了 JVM 去哪里找 `.class` 文件和资源文件。Spring Boot 项目打包后，classpath 就是 fat jar 内部的 `BOOT-INF/classes/` 和 `BOOT-INF/lib/` 下所有 jar 里的内容。

```
src/main/resources/application.yml
  → 编译后进入 BOOT-INF/classes/
  → 可以用 classpath:/application.yml 访问
```

在 Spring 里有两种写法，含义不同：


```yaml
# classpath:/ — 只搜索第一个匹配的位置（通常是项目自身的 resources）
spring:
  config:
    location: classpath:/config/

# classpath*:/ — 搜索所有 jar 包里的匹配路径（包括依赖 jar 内的资源）
# 常用于多模块项目，合并各模块的配置
```

代码里访问 classpath 资源：


```java
// 读取 classpath 下的文件
@Value("classpath:templates/email.html")
private Resource emailTemplate;

// 或者用 ResourceLoader
Resource resource = resourceLoader.getResource("classpath:/data/init.sql");
```

---

### 二、loader.path

`loader.path` 是 Spring Boot 的概念，不是标准 JVM 参数。它只在用 ==`PropertiesLauncher`== 启动时生效，作用是把  jar 外部的目录或 jar 追加到 classpath，实现运行时动态扩展。

默认的 `JarLauncher` 不支持 `loader.path`，必须切换启动器：


```xml
<!-- pom.xml 配置使用 PropertiesLauncher -->
<plugin>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-maven-plugin</artifactId>
    <configuration>
        <layout>ZIP</layout>  <!-- 使用 PropertiesLauncher -->
    </configuration>
</plugin>
```

启动时指定外部目录：


```bash
# 把 /app/ext 目录下的 jar 和资源追加到 classpath
java -Dloader.path=/app/ext,/app/config \
     -jar app.jar

# 也可以写在 loader.properties 文件里（放在 jar 同级目录）
# loader.path=/app/ext,/app/config
```

典型使用场景：

```
/app/
  app.jar                    # 主程序
  ext/
    mysql-connector.jar      # 外置数据库驱动
    custom-plugin.jar        # 动态插件
  config/
    application-prod.yml     # 外置生产配置（也会被加入 classpath）
```

---

### 三、spring.config.location

`spring.config.location` 是 Spring Boot 专属配置，只影响 `application.yml` / `application.properties` 这类配置文件的搜索位置，和 classpath、类加载完全无关。

Spring Boot 默认的配置搜索顺序（优先级从低到高）：

```
classpath:/
classpath:/config/
file:./
file:./config/
file:./config/*/
```

`spring.config.location` 会完全替换这个默认列表：


```bash
# 完全替换：只从这个目录找配置文件
java -jar app.jar \
  --spring.config.location=file:/app/config/

# 如果只想追加，不替换默认路径，用 additional-location
java -jar app.jar \
  --spring.config.additional-location=file:/app/config/
```

注意路径写法的差异：


```bash
# 以 / 结尾 = 目录，Spring Boot 会在此目录下找 application.yml
--spring.config.location=file:/app/config/

# 不以 / 结尾 = 具体文件
--spring.config.location=file:/app/config/application-prod.yml

# 多个路径用逗号分隔，后面的优先级更高
--spring.config.location=classpath:/,file:/app/config/
```

---

### 三者对比总结

|         | classpath         | loader.path            | spring.config.location\| |
| ------- | ----------------- | ---------------------- | ------------------------ |
| 属于哪层    | JVM 类加载层          | Spring Boot Launcher 层 | Spring Environment 层     |
| 影响范围    | 所有类和资源的加载         | 运行时追加到 classpath       | 仅 application 配置文件       |
| 生效时机    | JVM 启动时确定         | PropertiesLauncher 启动时 | Spring 上下文初始化时           |
| 需要特殊启动器 | 否                 | 是（PropertiesLauncher）  | 否                        |
| 典型用途    | 访问 resources 下的资源 | 外置驱动、插件                | K8s ConfigMap 外挂配置       |


K8s 场景下最常见的组合用法：


```bash
# 配置文件挂载到 /app/config（ConfigMap）
# 同时外置了数据库驱动
java -Dloader.path=/app/ext \
     -jar app.jar \
     --spring.config.additional-location=file:/app/config/
```

`spring.config.location` 指向的路径下的配置文件会自动进入 classpath 吗？不会。这是最常见的误解——它只是告诉 Spring 去哪里读配置文件，读完之后配置内容进入 `Environment`，但那个目录本身不会被加入 classpath，你无法用 `classpath:` 前缀访问那里的其他资源。

## 打包示例分析

```XML
<plugins>  
    <!-- 编译配置 -->  
    <plugin>  
        <groupId>org.apache.maven.plugins</groupId>  
        <artifactId>maven-compiler-plugin</artifactId>  
        <configuration>            
	        <source>17</source>  
            <target>17</target>  
        </configuration>  
    </plugin>  
  
    <!--  
        Thin jar：          
        - 输出到 target/deploy/lib/          
        - MANIFEST 设置 Main-Class 和 Class-Path（extlib/ 前缀，与依赖目录对应）              - 环境相关配置文件不打入 jar，由外部 conf/ 目录提供    
    -->    
    <plugin>  
        <groupId>org.apache.maven.plugins</groupId>  
        <artifactId>maven-jar-plugin</artifactId>  
		<configuration>
			<outputDirectory>${project.build.directory}/deploy/lib</outputDirectory>
			<archive>
				<manifest><mainClass>com.dataacc.management.OrchestrationApplication</mainClass>  
                    <addClasspath>true</addClasspath>  
                    <classpathPrefix>extlib/</classpathPrefix>  
            </manifest>  
            <manifestEntries>                    
                <Class-Path>../conf</Class-Path>  
            </manifestEntries>  
            </archive>  
            <excludes>                
	            <exclude>application*.yml</exclude>  
                <exclude>application*.yaml</exclude>  
                <exclude>logback-spring.xml</exclude>  
            </excludes>  
        </configuration>  
    </plugin>  
  
    <!--  
        拷贝运行时依赖到 target/deploy/lib/extlib/        
        includeScope=runtime 涵盖 compile + runtime，排除 test    
    -->    
        <plugin>  
        <groupId>org.apache.maven.plugins</groupId>  
        <artifactId>maven-dependency-plugin</artifactId>  
        <executions>            
	        <execution>  
                <id>copy-dependencies</id>  
                <phase>package</phase>  
                <goals>
                <goal>copy-dependencies</goal>  
                </goals>  
                <configuration>                    <outputDirectory>${project.build.directory}/deploy/lib/extlib</outputDirectory>  
                    <includeScope>runtime</includeScope>  
                    <overWriteIfNewer>true</overWriteIfNewer>  
                </configuration>  
            </execution>  
        </executions>  
    </plugin>  
  
    <!--  
        拷贝配置文件到 target/deploy/conf/        
        db/migration SQL 保留在 jar 内供 Flyway 加载，不在此处复制    
    -->    
        <plugin>  
        <groupId>org.apache.maven.plugins</groupId>  
        <artifactId>maven-resources-plugin</artifactId>  
        <executions>            
	        <execution>  
                <id>copy-config</id>  
                <phase>package</phase>  
                <goals>                    
	                <goal>copy-resources</goal>  
                </goals>  
                <configuration>                    <outputDirectory>${project.build.directory}/deploy/conf</outputDirectory>  
                    <resources>                        
	                    <resource>  
                            <directory>src/main/resources</directory>  
                            <filtering>false</filtering>  
                            <includes>
	                            <include>application*.yml</include>  
                                <include>application*.yaml</include>  
                                <include>logback-spring.xml</include>  
                            </includes>  
                        </resource>  
                    </resources>  
                </configuration>  
            </execution>  
        </executions>  
    </plugin>  
  
    <!--  
        跳过 spring-boot 的 fat jar 重打包；        
        可执行入口由 maven-jar-plugin 的 MANIFEST Main-Class 承担    
    -->    
    <plugin>  
        <groupId>org.springframework.boot</groupId>  
        <artifactId>spring-boot-maven-plugin</artifactId>  
        <configuration>            
        <skip>true</skip>  
        </configuration>  
    </plugin>  
</plugins>
```




```
target/deploy/
  lib/
    app.jar              ← thin jar，MANIFEST 里 Class-Path=extlib/*.jar
    extlib/
      spring-boot.jar
      mysql-connector.jar
      ...（所有依赖）
  conf/
    application.yml
    application-prod.yml
    logback-spring.xml
```

### 问题一：这个配置用 loader.path 是无效的

`loader.path` 只对 `PropertiesLauncher` 生效，但这个项目：


```xml
<!-- 跳过了 spring-boot-maven-plugin 的 repackage -->
<plugin>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-maven-plugin</artifactId>
    <configuration>
        <skip>true</skip>   ← fat jar 没有打，PropertiesLauncher 不存在
    </configuration>
</plugin>
```

这是标准的 thin jar 打包方式，启动用的是普通 `java -jar`，入口是 MANIFEST 里的 `Main-Class`。依赖通过 MANIFEST 的 `Class-Path` 条目加载：

```
# MANIFEST.MF 内容
Main-Class: com.dataacc.management.OrchestrationApplication
Class-Path: extlib/spring-boot.jar extlib/mysql.jar ...
```

所以正确的启动方式是：

bash

```bash
# 不需要 loader.path，Class-Path 已经在 MANIFEST 里写死了 extlib/
java -jar lib/app.jar --spring.config.location=file:conf/
```

---

### 问题二：conf 下的 logback-spring.xml 和 mapper 会被使用吗

**`spring.config.location` 只影响 Spring 配置文件**（`application*.yml`），不影响其他文件的加载。所以要分开分析：

#### logback-spring.xml — 不会自动使用

```
conf/logback-spring.xml ← spring.config.location 管不到这里

Logback 的加载机制：
  1. 在 classpath 下按顺序找：
     logback-test.xml → logback-spring.xml → logback.xml
  2. conf/ 目录不在 classpath 里
  3. 所以 conf/logback-spring.xml 不会被加载
```

要让外部的 `logback-spring.xml` 生效，需要单独指定：

```bash
java -jar lib/app.jar \
     --spring.config.location=file:conf/ \
     --logging.config=file:conf/logback-spring.xml
```

或者在 `application.yml` 里配置：



```yaml
logging:
  config: file:conf/logback-spring.xml
```

#### mapper xml — 取决于配置怎么写

mapper xml 文件的加载由 MyBatis 的 `mapper-locations` 配置决定：

yaml

```yaml
# 如果配置的是 classpath 路径
mybatis:
  mapper-locations: classpath:/mapper/*.xml
  # conf/ 不在 classpath，所以 conf/mapper/ 下的文件不会被加载
```

如果 mapper xml 已经打进了 jar 内（没有被 `excludes` 排除），那就没问题。看这个打包配置：


```xml
<excludes>
    <exclude>application*.yml</exclude>
    <exclude>application*.yaml</exclude>
    <exclude>logback-spring.xml</exclude>
    <!-- mapper xml 没有被排除 -->
</excludes>
```

mapper xml 没有被排除，所以它还在 jar 内的 classpath 里，`classpath:/mapper/*.xml` 正常生效，不需要从 conf/ 加载。

---

### 完整正确的启动命令


```bash
java -jar lib/app.jar \
     --spring.config.location=file:conf/ \
     --logging.config=file:conf/logback-spring.xml
```

---

### 各文件加载方式汇总

|文件|在哪里|怎么加载|是否需要额外配置|
|---|---|---|---|
|`application*.yml`|`conf/`|`spring.config.location=file:conf/`|是，启动参数指定|
|`logback-spring.xml`|`conf/`|`logging.config=file:conf/logback-spring.xml`|是，单独指定|
|`mapper/*.xml`|jar 内 classpath|`classpath:/mapper/*.xml`|否，已在 classpath|
|依赖 jar|`extlib/`|MANIFEST Class-Path 自动加载|否，MANIFEST 已写死|



```xml
<manifestEntries>
    <Class-Path>../conf</Class-Path>
</manifestEntries>
```

如果增加这个配置呢 classPath 会包含了 conf 目录吗？

是的，加了这个配置之后 `conf/` 目录会被追加到 classpath。

追加后的 MANIFEST.MF 内容

```
Main-Class: com.dataacc.management.OrchestrationApplication
Class-Path: extlib/spring-boot.jar extlib/mysql.jar ... ../conf
```

`../conf` 是相对于 jar 文件位置的路径。当前 jar 在 `lib/app.jar`，所以 `../conf` 指向的就是 `conf/` 目录。

---

### 重新回答之前的问题

conf 目录进了 classpath 之后，各文件的加载情况发生变化：

**logback-spring.xml — 会被自动加载**

```
Logback 在 classpath 下找 logback-spring.xml
conf/ 已在 classpath
→ conf/logback-spring.xml 自动被找到并加载
→ 不再需要 --logging.config 参数
```

**mapper xml — 取决于 mapper-locations 配置**


```yaml
# 如果配置的是 classpath:/mapper/*.xml
# conf/ 已在 classpath
# 那么 conf/mapper/*.xml 也会被扫描到

# 但此时 jar 内和 conf/ 下同时有 mapper xml
# 会产生重复加载，需要确认两个地方的文件是否一致
```

---

### 但有一个重要问题要注意

`Class-Path` 里的路径是相对于 **jar 文件所在目录**解析的，也就是 `lib/` 目录。

```
target/deploy/
  lib/
    app.jar          ← jar 在这里
    extlib/
  conf/              ← ../conf 从 lib/ 往上一级再进 conf/，路径正确
```


```bash
# 正确：在 deploy/ 目录下执行
cd target/deploy
java -jar lib/app.jar --spring.config.location=file:conf/

# 错误：在其他目录执行，../conf 解析位置不对
cd target/deploy/lib
java -jar app.jar  ← 此时 ../conf 指向 deploy/conf，反而是对的

# 最稳妥：用绝对路径
java -jar /app/lib/app.jar \
     --spring.config.location=file:/app/conf/
```

---

### 加了 manifestEntries 后的完整启动命令



```bash
java -jar lib/app.jar \
     --spring.config.location=file:conf/
```

`--logging.config` 不再需要，`logback-spring.xml` 会通过 classpath 自动加载。

## You said: 最终目录结构为 bin/start.

最终目录结构为 bin/start.sh ，lib/ ，conf/ 三个目录。 start.sh 会在当前目录下执行 java -jar lib/app.jar -Dload.path=.,conf,lib/extlib 这样有问题吗



## Claude responded: 有几个问题，逐一分析。

有几个问题，逐一分析。

### 问题一：`-D` 参数位置错误


```bash
# 错误写法：-D 参数放在 jar 后面，会被当成程序参数而不是 JVM 参数
java -jar lib/app.jar -Dloader.path=.,conf,lib/extlib

# 正确写法：-D 参数必须在 -jar 之前
java -Dloader.path=.,conf,lib/extlib -jar lib/app.jar
```

### 问题二：loader.path 对这个项目无效

如前面分析，这个项目跳过了 `spring-boot-maven-plugin` 的 repackage，打出来的是普通 thin jar，用标准 JVM 启动，不存在 `PropertiesLauncher`，所以 `loader.path` 完全不起作用。

依赖加载已经由 MANIFEST 的 `Class-Path` 条目负责，不需要 `loader.path`。

### 问题三：工作目录与相对路径

`start.sh` 在 `bin/` 目录下，执行 `java -jar lib/app.jar` 时，相对路径基准是 `bin/`，但 jar 和 conf 都在上一级：

```
deploy/
  bin/
    start.sh         ← 从这里执行
  lib/
    app.jar          ← MANIFEST 里 ../conf 是相对 lib/ 的，指向 deploy/conf ✓
    extlib/
  conf/
    application.yml
    logback-spring.xml
```

`start.sh` 从 `bin/` 执行时：


```bash
# lib/app.jar 相对 bin/ 找不到
java -jar lib/app.jar   # 报错：找不到 lib/app.jar
```

### 正确的 start.sh 写法


```bash
#!/bin/bash

# 获取 start.sh 所在目录，再往上一级得到部署根目录
BASE_DIR=$(cd "$(dirname "$0")/.." && pwd)

echo "BASE_DIR: $BASE_DIR"

# 所有路径都用绝对路径，不依赖工作目录
java -jar "$BASE_DIR/lib/app.jar" \
     --spring.config.location=file:"$BASE_DIR/conf/"
```

这样无论从哪个目录执行 `start.sh`，路径都能正确解析。

---

### MANIFEST Class-Path 的路径问题再确认

你加了 `<Class-Path>../conf</Class-Path>`，这个路径是相对于 **jar 文件所在目录**（`lib/`）解析的：

```
lib/app.jar 位置 → lib/
../conf           → deploy/conf/  ✓ 正确
```

所以 `conf/` 进了 classpath，`logback-spring.xml` 会自动加载，没问题。

---

### 最终建议的 start.sh


```bash
#!/bin/bash

BASE_DIR=$(cd "$(dirname "$0")/.." && pwd)

# JVM 参数
JVM_OPTS="-Xms512m -Xmx1g \
          -XX:+UseContainerSupport \
          -XX:MaxRAMPercentage=75.0 \
          -XX:+ExitOnOutOfMemoryError"

exec java $JVM_OPTS \
     -jar "$BASE_DIR/lib/app.jar" \
     --spring.config.location=file:"$BASE_DIR/conf/"
```

用 `exec` 替换当前 shell 进程，让 java 进程直接成为 PID 1（或 tini 的子进程），保证 K8s 优雅停机信号能正确传递，这也是之前讨论过的最佳实践。


## 需要让 spring-boot-maven-plugin 的 repackage 生效，并指定使用 PropertiesLauncher。

-Dload.path 参数要生效需要让 `spring-boot-maven-plugin` 的 repackage 生效，并指定使用 `PropertiesLauncher`。

### 调整打包配置


```xml
<build>
    <plugins>
        <!-- 编译配置不变 -->
        <plugin>
            <groupId>org.apache.maven.plugins</groupId>
            <artifactId>maven-compiler-plugin</artifactId>
            <configuration>
                <source>17</source>
                <target>17</target>
            </configuration>
        </plugin>

        <!-- thin jar 配置：去掉 manifestEntries 的 Class-Path -->
        <plugin>
            <groupId>org.apache.maven.plugins</groupId>
            <artifactId>maven-jar-plugin</artifactId>
            <configuration>
                <outputDirectory>${project.build.directory}/deploy/lib</outputDirectory>
                <archive>
                    <manifest>
                        <mainClass>com.dataacc.management.OrchestrationApplication</mainClass>
                        <addClasspath>true</addClasspath>
                        <classpathPrefix>extlib/</classpathPrefix>
                    </manifest>
                    <!-- 去掉之前手动加的 ../conf，改由 loader.path 负责 -->
                </archive>
                <excludes>
                    <exclude>application*.yml</exclude>
                    <exclude>application*.yaml</exclude>
                    <exclude>logback-spring.xml</exclude>
                </excludes>
            </configuration>
        </plugin>

        <!-- 依赖拷贝不变 -->
        <plugin>
            <groupId>org.apache.maven.plugins</groupId>
            <artifactId>maven-dependency-plugin</artifactId>
            <executions>
                <execution>
                    <id>copy-dependencies</id>
                    <phase>package</phase>
                    <goals>
                        <goal>copy-dependencies</goal>
                    </goals>
                    <configuration>
                        <outputDirectory>${project.build.directory}/deploy/lib/extlib</outputDirectory>
                        <includeScope>runtime</includeScope>
                        <overWriteIfNewer>true</overWriteIfNewer>
                    </configuration>
                </execution>
            </executions>
        </plugin>

        <!-- 配置文件拷贝不变 -->
        <plugin>
            <groupId>org.apache.maven.plugins</groupId>
            <artifactId>maven-resources-plugin</artifactId>
            <executions>
                <execution>
                    <id>copy-config</id>
                    <phase>package</phase>
                    <goals>
                        <goal>copy-resources</goal>
                    </goals>
                    <configuration>
                        <outputDirectory>${project.build.directory}/deploy/conf</outputDirectory>
                        <resources>
                            <resource>
                                <directory>src/main/resources</directory>
                                <filtering>false</filtering>
                                <includes>
                                    <include>application*.yml</include>
                                    <include>application*.yaml</include>
                                    <include>logback-spring.xml</include>
                                </includes>
                            </resource>
                        </resources>
                    </configuration>
                </execution>
            </executions>
        </plugin>

        <!--
            关键改动：
            1. 去掉 skip=true，让 repackage 执行
            2. layout=ZIP 指定使用 PropertiesLauncher
        -->
        <plugin>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-maven-plugin</artifactId>
            <configuration>
                <!-- 不再 skip -->
                <layout>ZIP</layout>

                <!-- repackage 输出到 deploy/lib，覆盖 maven-jar-plugin 的输出 -->
                <outputDirectory>${project.build.directory}/deploy/lib</outputDirectory>

                <!--
                    排除依赖不打进 fat jar
                    因为依赖已经单独拷贝到 extlib/，由 loader.path 加载
                    避免 fat jar 体积膨胀
                -->
                <excludes>
                    <!-- 排除所有依赖，只保留项目自身的类 -->
                </excludes>
            </configuration>
            <executions>
                <execution>
                    <goals>
                        <goal>repackage</goal>
                    </goals>
                </execution>
            </executions>
        </plugin>
    </plugins>
</build>
```

---

### 打包后的结构变化

```
target/deploy/
  lib/
    app.jar              ← repackage 后的 jar，Main-Class = PropertiesLauncher
    extlib/
      spring-boot.jar
      mysql.jar
      ...
  conf/
    application.yml
    logback-spring.xml
  bin/
    start.sh
```

打包后 jar 内的 MANIFEST 变成：

```
Main-Class: org.springframework.boot.loader.PropertiesLauncher
Start-Class: com.dataacc.management.OrchestrationApplication
```

---

### 对应的 start.sh


```bash
#!/bin/bash

BASE_DIR=$(cd "$(dirname "$0")/.." && pwd)

JVM_OPTS="-Xms512m -Xmx1g \
          -XX:+UseContainerSupport \
          -XX:MaxRAMPercentage=75.0"

exec java $JVM_OPTS \
     -Dloader.path="$BASE_DIR/conf,$BASE_DIR/lib/extlib" \
     -jar "$BASE_DIR/lib/app.jar" \
     --spring.config.location=file:"$BASE_DIR/conf/"
```

---

### 两种方案对比

||MANIFEST Class-Path + 普通 JarLauncher|loader.path + PropertiesLauncher|
|---|---|---|
|依赖加载|MANIFEST 写死 Class-Path|运行时动态指定|
|配置目录进 classpath|需要手动加 `../conf` 到 MANIFEST|`loader.path` 直接指定|
|灵活性|低，路径打包时确定|高，启动时可任意指定|
|logback 自动加载|需要 MANIFEST 加 conf 路径|loader.path 包含 conf 即可|
|打包复杂度|低（skip repackage）|略高（需要 repackage）|
|适合场景|路径固定的传统部署|需要灵活切换配置目录|

`loader.path` 的主要优势是运行时灵活指定路径。



##  repackage 

`repackage` 是 `spring-boot-maven-plugin` 的一个 goal，它绑定在 `package` 阶段自动执行，不需要单独调用。

### 执行流程

```
mvn package
  → maven-compiler-plugin    编译 .java → .class
  → maven-jar-plugin         打普通 jar（原始 jar）
  → maven-dependency-plugin  拷贝依赖到 extlib/
  → maven-resources-plugin   拷贝配置文件到 conf/
  → spring-boot-maven-plugin repackage 对原始 jar 进行二次处理
      → 注入 PropertiesLauncher
      → 修改 MANIFEST.MF
      → 输出最终可执行 jar（覆盖原始 jar）
```

---

之前 skip=true 的作用


```xml
<plugin>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-maven-plugin</artifactId>
    <configuration>
        <skip>true</skip>   ← 跳过 repackage 这个步骤
    </configuration>
</plugin>
```

加 `skip=true` 是主动跳过 repackage，所以 `mvn package` 时 spring-boot-maven-plugin 什么都不做，最终产物就是 maven-jar-plugin 打出的普通 thin jar。

去掉 `skip=true` 之后，`mvn package` 就会自动执行 repackage，不需要任何额外操作。

---

### 验证 repackage 是否生效

打包完成后检查 jar 内的 MANIFEST：


```bash
# 查看 MANIFEST 内容
unzip -p target/deploy/lib/app.jar BOOT-INF/classes/ 2>/dev/null \
  || jar xf target/deploy/lib/app.jar META-INF/MANIFEST.MF -C /tmp && cat /tmp/META-INF/MANIFEST.MF

# 或者更简单
unzip -p target/deploy/lib/app.jar META-INF/MANIFEST.MF
```

repackage 生效时看到：

```
Main-Class: org.springframework.boot.loader.PropertiesLauncher
Start-Class: com.dataacc.management.OrchestrationApplication
Spring-Boot-Version: 3.x.x
```

repackage 未生效（普通 jar）时看到：

```
Main-Class: com.dataacc.management.OrchestrationApplication
Class-Path: extlib/spring-boot.jar extlib/...
```



## 怎么验证下当前的 classPath 以及下的文件

有几种方式，从简单到详细排列。

### 方式一：启动时打印（最直接）


```bash
# 打印完整 classpath 后退出，不真正启动应用
java -cp $(java -jar lib/app.jar --print-classpath 2>/dev/null) \
     -XshowSettings:all -version 2>&1 | grep classpath
```

更简单的方式，加 JVM 参数打印：


```bash
java -verbose:class \
     -Dloader.path=../conf,lib/extlib \
     -jar lib/app.jar 2>&1 | head -50
```

启动日志里会看到每个类从哪个 classpath 位置加载：

```
[Loaded java.lang.Object from /usr/lib/jvm/...]
[Loaded com.dataacc.OrchestrationApplication from file:/app/lib/app.jar]
```

---

### 方式二：代码里打印 classpath（推荐）

写一个启动监听器，应用启动后自动打印：


```java
@Component
@Slf4j
public class ClasspathPrinter implements ApplicationListener<ApplicationReadyEvent> {

    @Override
    public void onApplicationEvent(ApplicationReadyEvent event) {
        // 打印所有 classpath URL
        ClassLoader classLoader = Thread.currentThread().getContextClassLoader();
        printClasspath(classLoader);
    }

    private void printClasspath(ClassLoader classLoader) {
        if (classLoader == null) return;

        if (classLoader instanceof URLClassLoader urlClassLoader) {
            log.info("=== Classpath URLs ===");
            for (URL url : urlClassLoader.getURLs()) {
                log.info("  {}", url);
            }
        } else {
            // JDK 9+ 无法直接转 URLClassLoader，用系统属性
            log.info("=== java.class.path ===");
            String cp = System.getProperty("java.class.path");
            Arrays.stream(cp.split(File.pathSeparator))
                  .forEach(p -> log.info("  {}", p));
        }

        // 递归打印父类加载器
        log.info("=== Parent ClassLoader: {} ===", classLoader.getParent());
        printClasspath(classLoader.getParent());
    }
}
```

---

### 方式三：验证特定文件是否在 classpath 下可访问


```java
@Component
@Slf4j
public class ClasspathResourceChecker implements ApplicationListener<ApplicationReadyEvent> {

    @Autowired
    private ResourceLoader resourceLoader;

    @Override
    public void onApplicationEvent(ApplicationReadyEvent event) {
        // 检查特定文件是否能从 classpath 找到
        checkResource("classpath:application.yml");
        checkResource("classpath:logback-spring.xml");
        checkResource("classpath:mapper/UserMapper.xml");
    }

    private void checkResource(String location) {
        Resource resource = resourceLoader.getResource(location);
        if (resource.exists()) {
            try {
                log.info("FOUND   {} → {}", location, resource.getURL());
            } catch (IOException e) {
                log.info("FOUND   {} (URL不可读)", location);
            }
        } else {
            log.warn("MISSING {}", location);
        }
    }
}
```

输出示例：

```
FOUND   classpath:application.yml → file:/app/conf/application.yml
FOUND   classpath:logback-spring.xml → file:/app/conf/logback-spring.xml
MISSING classpath:mapper/UserMapper.xml
```

---

### 方式四：命令行直接验证 jar 内容


```bash
# 查看 jar 内的文件列表
jar tf lib/app.jar | grep -E "\.yml|\.xml|\.properties"

# 查看 MANIFEST 确认 Class-Path
unzip -p lib/app.jar META-INF/MANIFEST.MF

# 查看 conf 目录下的文件
ls -la conf/

# 查看 extlib 下有哪些 jar
ls lib/extlib/ | head -20
```

---

### 方式五：Actuator 端点验证（生产环境推荐）


```yaml
management:
  endpoints:
    web:
      exposure:
        include: env,configprops
```



```bash
# 查看所有配置来源，能看到配置文件从哪里加载的
curl http://localhost:8080/actuator/env | python3 -m json.tool | grep -A2 "applicationConfig"
```

输出里能看到：



```json
"applicationConfig: [file:/app/conf/application.yml]": {
    ...
}
```

---

### 最快的验证方式

如果只是想快速确认 `conf/` 和 `extlib/` 有没有进 classpath，在 `application.yml` 里加一行：



```yaml
logging:
  level:
    org.springframework.core.io: DEBUG
```

启动日志里会打印每个资源的加载位置：

```
DEBUG o.s.core.io.support.PathMatchingResourcePatternResolver - 
  Resolved location pattern [classpath:logback-spring.xml] 
  to resources [file:/app/conf/logback-spring.xml]
```

从路径就能直接判断文件从哪里加载的。



## 加了 maven 的repack 后 lib 下会有两个 jar


对，这是 `spring-boot-maven-plugin` repackage 的标准行为。

### 两个 jar 的来源

```
mvn package 执行顺序：

1. maven-jar-plugin 打出普通 jar
   → target/deploy/lib/app.jar     ← 普通 thin jar

2. spring-boot-maven-plugin repackage
   → 把原来的 app.jar 重命名为 app.jar.original  ← 备份原始 jar
   → 生成新的 app.jar                             ← 可执行 fat/thin jar，含 PropertiesLauncher
```

---

### 两个 jar 的区别


```bash
# 查看原始 jar 的 MANIFEST
unzip -p lib/app.jar.original META-INF/MANIFEST.MF

Main-Class: com.dataacc.management.OrchestrationApplication
Class-Path: extlib/spring-boot.jar extlib/mysql.jar ...

# 查看 repackage 后的 jar 的 MANIFEST
unzip -p lib/app.jar META-INF/MANIFEST.MF

Main-Class: org.springframework.boot.loader.PropertiesLauncher
Start-Class: com.dataacc.management.OrchestrationApplication
Spring-Boot-Version: 3.x.x
```

---

`.original` 只是 repackage 前的备份，生产部署不需要它。可以在打包脚本里清理：


```bash
mvn package
rm -f target/deploy/lib/*.original
```

或者在 maven 里配置打包完自动删除：


```xml
<plugin>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-maven-plugin</artifactId>
    <configuration>
        <layout>ZIP</layout>
        <outputDirectory>${project.build.directory}/deploy/lib</outputDirectory>
    </configuration>
    <executions>
        <execution>
            <goals>
                <goal>repackage</goal>
            </goals>
        </execution>
    </executions>
</plugin>

<!-- 用 maven-clean-plugin 在 package 后清理 .original -->
<plugin>
    <groupId>org.apache.maven.plugins</groupId>
    <artifactId>maven-antrun-plugin</artifactId>
    <executions>
        <execution>
            <id>remove-original-jar</id>
            <phase>package</phase>
            <goals>
                <goal>run</goal>
            </goals>
            <configuration>
                <target>
                    <delete>
                        <fileset dir="${project.build.directory}/deploy/lib"
                                 includes="*.original"/>
                    </delete>
                </target>
            </configuration>
        </execution>
    </executions>
</plugin>
```



`spring-boot-maven-plugin` 的 `repackage` 默认会把所有依赖打进去变成 fat jar。要保持 thin jar 效果，需要配置 `excludeGroupIds` 或者用 `excludes` 明确排除，但这两种方式要一个个列举依赖，很麻烦。

更简洁的做法是用 `includeSystemScope` 结合 `excludeArtifactIds`，但最直接的方案是利用 repackage 的 `excludeDevtools` 配合另一个思路：

### 正确做法：用 classifier 区分两个 jar


```xml
<plugin>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-maven-plugin</artifactId>
    <configuration>
        <layout>ZIP</layout>
        <outputDirectory>${project.build.directory}/deploy/lib</outputDirectory>
        <!-- 给 repackage 的 jar 加后缀，不覆盖原始 thin jar -->
        <classifier>exec</classifier>
    </configuration>
    <executions>
        <execution>
            <goals>
                <goal>repackage</goal>
            </goals>
        </execution>
    </executions>
</plugin>
```

这样打出来两个 jar：

```
target/deploy/lib/
  app.jar          ← maven-jar-plugin 打的原始 thin jar（Class-Path 指向 extlib/）
  app-exec.jar     ← repackage 后的可执行 jar，含 PropertiesLauncher
```

启动脚本用 `app-exec.jar`：



```bash
exec java $JVM_OPTS \
     -Dloader.path="$BASE_DIR/conf,$BASE_DIR/lib/extlib" \
     -jar "$BASE_DIR/lib/app-exec.jar" \
     --spring.config.location=file:"$BASE_DIR/conf/"
```

---

### 验证是否是 thin jar


```bash
# 查看 jar 大小，thin jar 通常只有几百 KB
ls -lh target/deploy/lib/

# 查看 jar 内容，thin jar 里不应该有 BOOT-INF/lib/
jar tf target/deploy/lib/app-exec.jar | grep "BOOT-INF/lib" | head -5

# 有输出 = fat jar，无输出 = thin jar（依赖在外部 extlib/）
```

---

### 完整打包结构确认

```
target/deploy/
  lib/
    app.jar          ← 不用这个启动
    app-exec.jar     ← 用这个启动，含 PropertiesLauncher，无依赖
    extlib/
      spring-boot.jar
      mysql.jar
      ...
  conf/
    application.yml
    logback-spring.xml
  bin/
    start.sh
```

`app-exec.jar` 里只有项目自身的类，依赖通过 `loader.path` 从 `extlib/` 加载，这才是真正的 thin jar + `PropertiesLauncher` 的正确用法。


## You said: 这个问题到底是和启动的时候执行路径有关，还是和相对jar 文件的路径有关


## Claude responded: 和启动时的执行路径无关，只和 jar 文件本身的位置有关。

和启动时的执行路径**无关**，只和 **jar 文件本身的位置**有关。

### 验证一下

```
deploy/
  lib/
    app.jar     ← jar 在这里
    extlib/
  conf/
```

MANIFEST 里写的是 `../conf`，这个路径的基准永远是 **jar 文件所在目录**（`lib/`），和你在哪里执行 `java -jar` 完全没关系。


```bash
# 从 deploy/ 执行
cd deploy
java -jar lib/app.jar
# ../conf 基准是 lib/，解析为 deploy/conf ✓

# 从 bin/ 执行
cd deploy/bin
java -jar ../lib/app.jar
# ../conf 基准还是 lib/，解析还是 deploy/conf ✓

# 从任意目录执行
java -jar /opt/app/deploy/lib/app.jar
# ../conf 基准还是 /opt/app/deploy/lib/，解析为 /opt/app/deploy/conf ✓
```

结论：**只要 jar 文件和 conf/ 目录的相对位置不变，从任何地方执行都没问题。**

---

### 我之前说的有误

之前说「启动时必须在 lib/ 目录下执行」是错的，`Class-Path` 的相对路径解析是基于 jar 文件位置，不是基于工作目录（`pwd`）。工作目录只影响 `--spring.config.location=file:conf/` 这类没有写绝对路径的参数，和 MANIFEST `Class-Path` 无关。

