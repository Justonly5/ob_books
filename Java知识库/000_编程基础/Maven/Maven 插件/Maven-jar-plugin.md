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

> maven-jar 插件只是搜集 .class 文件和 resources 下的资源文件，默认情况下，打出来的包是没法运行的，因为没有 Main-class、没有依赖的 jar。
---

### 详细配置项

#### 输出控制
```XML
<plugin>
    <groupId>org.apache.maven.plugins</groupId>
    <artifactId>maven-jar-plugin</artifactId>
    <configuration>
        <!-- 输出目录，默认 target/ -->
        <outputDirectory>${project.build.directory}/deploy/lib</outputDirectory>
        <!-- 输出文件名，默认 ${artifactId}-${version} -->
        <finalName>app</finalName>
        <!-- 是否跳过打包，默认 false -->
        <skip>false</skip>
        <!-- 是否强制覆盖已有的 jar，默认 false -->
        <forceCreation>false</forceCreation>
        <!-- 文件包含与排除 -->
        <!-- 排除特定文件，支持 Ant 风格通配符 -->
	    <excludes>
	        <exclude>application*.yml</exclude>
	        <exclude>application*.yaml</exclude>
	        <exclude>logback-spring.xml</exclude>
	        <exclude>**/*-dev.properties</exclude>
	        <!-- 排除整个目录 -->
	        <exclude>static/**</exclude>
	    </excludes>

	    <!-- 只包含特定文件（includes 和 excludes 同时配置时，excludes 优先）-->
	    <includes>
	        <include>**/*.class</include>
	        <include>mapper/**</include>
	        <include>db/migration/**</include>
	    </includes>
        
    </configuration>
    

</plugin>
```

#### MANIFEST.MF 配置

```XML
<configuration>
    <archive>
        <!-- manifest 节点：常用快捷配置 -->
        <manifest>
            <!-- 主类，配置后 jar 可直接 java -jar 执行 -->
            <mainClass>com.example.Application</mainClass>
            <!-- 是否在 MANIFEST 里生成 Class-Path 条目 -->
            <addClasspath>true</addClasspath>
            <!-- Class-Path 条目的前缀，对应依赖 jar 的存放目录 -->
            <classpathPrefix>extlib/</classpathPrefix>
            <!-- Class-Path 中使用文件名而不是 groupId/artifactId/version 路径 -->
            <classpathLayoutType>simple</classpathLayoutType>
            <!-- 是否添加 Build-Jdk 等构建信息，默认 true -->
            <addBuildEnvironmentEntries>true</addBuildEnvironmentEntries>
            <!-- 是否添加 Implementation-* 和 Specification-* 条目 -->
<addDefaultImplementationEntries>true</addDefaultImplementationEntries>
<addDefaultSpecificationEntries>false</addDefaultSpecificationEntries>
        </manifest>

        <!-- manifestEntries：手动追加任意 MANIFEST 条目 -->
        <manifestEntries>
            <!-- 追加额外的 Class-Path 条目 
	            相对于启动的 jar 的目录。
            -->
            <Class-Path>../conf</Class-Path>
            <!-- 自定义版本信息 -->
            <App-Version>${project.version}</App-Version>
            <Build-Time>${maven.build.timestamp}</Build-Time>
            <Git-Commit>${git.commit.id.abbrev}</Git-Commit>
        </manifestEntries>

        <!-- 使用外部 MANIFEST 文件完全替代自动生成 -->
        <!-- 
	        <manifestFile>
		        src/main/resources/META-INF/MANIFEST.MF
             </manifestFile> 
        -->
        <!-- 是否压缩 jar，默认 true -->
        <compress>true</compress>
        <!-- 是否添加 Maven 描述文件（pom.xml / pom.properties）到 jar 内 -->
        <addMavenDescriptor>true</addMavenDescriptor>
    </archive>
</configuration>
```