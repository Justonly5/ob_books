
## setting
settings 结构：
```XML
<settings>
  <!-- repository 不能直接放这里，settings.xml 不支持顶层 repositories -->

  <mirrors>...</mirrors>      <!-- ← 可以直接放顶层 -->
  <servers>...</servers>      <!-- ← 可以直接放顶层 -->
  <proxies>...</proxies>      <!-- ← 可以直接放顶层 -->

  <profiles>
    <profile>
      <repositories>
	      <repository></repository>
      </repositories>  <!-- ← repository 只能放这里 -->
      <pluginRepositories>
	      <pluginRepository></pluginRepository>
      </pluginRepositories>
    </profile>
  </profiles>

  <activeProfiles>...</activeProfiles>
</settings>
```

### 各节点说明速查

| 节点                   | 是否必须 | 能否放顶层               | 说明                           |
| -------------------- | ---- | ------------------- | ---------------------------- |
| `localRepository`    | 否    | 是                   | 默认 `~/.m2/repository`        |
| `proxies`            | 否    | 是                   | 网络代理，内网不需要                   |
| `servers`            | 否    | 是                   | 认证信息，id 对应 repository/mirror |
| `mirrors`            | 否    | 是                   | 仓库拦截代理                       |
| `profiles`           | 否    | 是                   | 配置集容器                        |
| `repositories`       | 否    | **否，只能在 profile 内** | 依赖仓库                         |
| `pluginRepositories` | 否    | **否，只能在 profile 内** | 插件仓库                         |
| `activeProfiles`     | 否    | 是                   | 激活指定 profile                 |
| `pluginGroups`       | 否    | 是                   | 简化插件命令                       |


![[Pasted image 20260610102802.png]]

### 完整示例
-----


```XML
<?xml version="1.0" encoding="UTF-8"?>
<settings xmlns="http://maven.apache.org/SETTINGS/1.2.0"
          xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
          xsi:schemaLocation="http://maven.apache.org/SETTINGS/1.2.0
                              https://maven.apache.org/xsd/settings-1.2.0.xsd">

    <!-- ══════════════════════════════════════════════════════
         基础路径配置
         ══════════════════════════════════════════════════════ -->

    <!-- 本地仓库路径，默认 ~/.m2/repository -->
    <localRepository>/data/maven/repository</localRepository>
    <!-- 是否使用交互模式，CI 环境设 false，默认 true -->
    <interactiveMode>true</interactiveMode>
    <!-- 是否离线模式，只使用本地仓库，默认 false -->
    <offline>false</offline>
    <!-- 插件自动更新检测，默认 true -->
    <usePluginRegistry>false</usePluginRegistry>


    <!-- ══════════════════════════════════════════════════════
         代理配置（访问外网需要代理时）
         ══════════════════════════════════════════════════════ -->

    <proxies>
        <proxy>
            <!-- 唯一 ID -->
            <id>http-proxy</id>
            <!-- 是否激活 -->
            <active>true</active>
            <!-- 协议：http 或 https -->
            <protocol>http</protocol>
            <!-- 代理服务器地址 -->
            <host>proxy.company.com</host>
            <port>8080</port>
            <!-- 代理认证（无认证可删除） -->
            <username>proxyuser</username>
            <password>proxypass</password>
            <!-- 不走代理的地址，| 分隔 -->
            <nonProxyHosts>localhost|127.0.0.1|*.company.com</nonProxyHosts>
        </proxy>
    </proxies>


    <!-- ══════════════════════════════════════════════════════
         认证配置
         id 必须和 repository / mirror / distributionManagement 的 id 一致
         ══════════════════════════════════════════════════════ -->

    <servers>

        <!-- 下载认证 -->
        <server>
            <id>nexus-public</id>
            <username>reader</username>
            <!-- 明文密码 -->
            <password>reader123</password>
            <!-- 或使用加密密码（mvn --encrypt-password 生成）-->
            <!-- <password>{COQLCE6DU6GtcS5P=}</password> -->
        </server>

        <!-- 发布 release 认证 -->
        <server>
            <id>nexus-releases</id>
            <username>deployer</username>
            <password>deploy123</password>
        </server>

        <!-- 发布 snapshot 认证 -->
        <server>
            <id>nexus-snapshots</id>
            <username>deployer</username>
            <password>deploy123</password>
        </server>

        <!-- 私钥认证（用于 SSH 场景） -->
        <server>
            <id>ssh-repo</id>
            <privateKey>${user.home}/.ssh/id_rsa</privateKey>
            <passphrase>keypassphrase</passphrase>
            <!-- 目录权限配置（scp 部署时生效） -->
            <directoryPermissions>775</directoryPermissions>
            <filePermissions>664</filePermissions>
        </server>

    </servers>


    <!-- ══════════════════════════════════════════════════════
         镜像配置
         拦截对特定仓库的请求，重定向到 mirror 地址
         ══════════════════════════════════════════════════════ -->

    <mirrors>

        <!-- 全局代理：所有仓库请求都走私服 -->
        <mirror>
            <id>nexus-all</id>
            <name>Nexus All Mirror</name>
            <url>http://nexus.company.com/repository/maven-public/</url>
            <!--
                mirrorOf 取值：
                *              所有仓库
                central        只拦截 central
                repo1,repo2    拦截多个指定仓库
                *,!repo1       除 repo1 外所有仓库
                external:*     除本地文件系统外所有外部仓库
            -->
            <mirrorOf>*</mirrorOf>
            <!-- 是否阻断（blocked=true 时该 mirror 请求直接失败，用于安全管控） -->
            <blocked>false</blocked>
        </mirror>

        <!-- 只代理 central（与上面二选一，不能同时生效） -->
        <!--
        <mirror>
            <id>aliyun-central</id>
            <name>Aliyun Central Mirror</name>
            <url>https://maven.aliyun.com/repository/central</url>
            <mirrorOf>central</mirrorOf>
        </mirror>
        -->

    </mirrors>


    <!-- ══════════════════════════════════════════════════════
         配置集
         repository 必须放在 profile 内才生效
         ══════════════════════════════════════════════════════ -->

    <profiles>

        <!-- ── 企业内网 profile ───────────────────────────── -->
        <profile>
            <id>nexus</id>
            <!-- profile 级别的属性，可在 pom.xml 里用 ${} 引用 -->
            <properties>
                <nexus.url>http://nexus.company.com</nexus.url>
                <env.name>company</env.name>
            </properties>
            <!-- 依赖仓库 -->
            <repositories>
                <repository>
                    <id>nexus-public</id>
                    <name>Nexus Public Group</name>
                    <url>${nexus.url}/repository/maven-public/</url>
                    <!-- release 版本策略 -->
                    <releases>
                        <enabled>true</enabled>
                        <!--
                            updatePolicy 取值：
                            always    每次构建都检查更新
                            daily     每天检查一次（默认）
                            interval:N 每 N 分钟检查一次
                            never     从不检查（release 包推荐）
                        -->
                        <updatePolicy>never</updatePolicy>
                        <!--
                            checksumPolicy 取值：
                            warn    校验失败打警告（默认）
                            fail    校验失败终止构建
                            ignore  忽略校验
                        -->
                        <checksumPolicy>warn</checksumPolicy>
                    </releases>
                    <!-- snapshot 版本策略 -->
                    <snapshots>
                        <enabled>true</enabled>
                        <updatePolicy>always</updatePolicy>
                        <checksumPolicy>warn</checksumPolicy>
                    </snapshots>
                    <!-- 仓库布局，默认 default，legacy 是 Maven 1.x 格式 -->
                    <layout>default</layout>
                </repository>
            </repositories>

            <!-- 插件仓库（和依赖仓库独立，两套都需要配） -->
            <pluginRepositories>
                <pluginRepository>
                    <id>nexus-plugins</id>
                    <name>Nexus Plugin Repository</name>
                    <url>${nexus.url}/repository/maven-public/</url>
                    <releases>
                        <enabled>true</enabled>
                        <updatePolicy>never</updatePolicy>
                    </releases>
                    <snapshots>
                        <enabled>false</enabled>
                    </snapshots>
                </pluginRepository>
            </pluginRepositories>

        </profile>


        <!-- ── 开发环境 profile ───────────────────────────── -->
        <profile>
            <id>dev</id>
            <properties>
                <env.name>dev</env.name>
                <skip.tests>false</skip.tests>
            </properties>
            <repositories>
                <repository>
                    <id>nexus-dev</id>
                    <url>http://nexus.company.com/repository/maven-public/</url>
                    <releases><enabled>true</enabled></releases>
                    <snapshots>
                        <enabled>true</enabled>
                        <!-- 开发环境 snapshot 实时更新 -->
                        <updatePolicy>always</updatePolicy>
                    </snapshots>
                </repository>
            </repositories>
        </profile>


        <!-- ── 生产环境 profile ───────────────────────────── -->
        <profile>
            <id>prod</id>
            <properties>
                <env.name>prod</env.name>
                <skip.tests>true</skip.tests>
            </properties>
            <repositories>
                <repository>
                    <id>nexus-prod</id>
                    <!-- 生产只用 release 仓库 -->
                    <url>http://nexus.company.com/repository/maven-releases/</url>
                    <releases>
                        <enabled>true</enabled>
                        <updatePolicy>never</updatePolicy>
                    </releases>
                    <snapshots>
                        <!-- 生产禁止 snapshot -->
                        <enabled>false</enabled>
                    </snapshots>
                </repository>
            </repositories>
        </profile>


        <!-- ── 按 JDK 版本自动激活的 profile ────────────── -->
        <profile>
            <id>jdk17</id>
            <activation>
                <!-- 满足条件时自动激活，无需 activeProfiles 声明 -->
                <jdk>17</jdk>
            </activation>
            <properties>
                <maven.compiler.source>17</maven.compiler.source>
                <maven.compiler.target>17</maven.compiler.target>
                <maven.compiler.release>17</maven.compiler.release>
            </properties>
        </profile>


        <!-- ── 按操作系统自动激活 ────────────────────────── -->
        <profile>
            <id>linux-profile</id>
            <activation>
                <os>
                    <name>Linux</name>
                    <family>unix</family>
                    <!-- <arch>amd64</arch> -->
                    <!-- <version>...</version> -->
                </os>
            </activation>
            <properties>
                <os.type>linux</os.type>
            </properties>
        </profile>


        <!-- ── 按系统属性激活：mvn package -Denv=special ── -->
        <profile>
            <id>special-env</id>
            <activation>
                <property>
                    <name>env</name>
                    <value>special</value>
                </property>
            </activation>
            <repositories>
                <repository>
                    <id>special-repo</id>
                    <url>http://special.repo.com/maven/</url>
                    <releases><enabled>true</enabled></releases>
                    <snapshots><enabled>false</enabled></snapshots>
                </repository>
            </repositories>
        </profile>


        <!-- ── 按文件是否存在激活 ────────────────────────── -->
        <profile>
            <id>file-activation</id>
            <activation>
                <file>
                    <!-- 文件存在时激活 -->
                    <exists>${basedir}/special-config.xml</exists>
                    <!-- 文件不存在时激活 -->
                    <!-- <missing>${basedir}/skip-flag</missing> -->
                </file>
            </activation>
        </profile>


        <!-- ── 默认激活的 profile ────────────────────────── -->
        <profile>
            <id>always-active</id>
            <activation>
                <!-- activeByDefault=true，除非命令行指定了其他 profile -->
                <activeByDefault>true</activeByDefault>
            </activation>
            <properties>
                <build.timestamp>${maven.build.timestamp}</build.timestamp>
            </properties>
        </profile>

    </profiles>


    <!-- ══════════════════════════════════════════════════════
         激活 profile
         这里列出的 profile 无论任何条件都会激活
         ══════════════════════════════════════════════════════ -->

    <activeProfiles>
        <!-- 必须激活的 profile，多个按声明顺序叠加，后声明的优先级更高 -->
        <activeProfile>nexus</activeProfile>
        <!-- 可以用命令行追加：mvn package -P dev -->
    </activeProfiles>


    <!-- ══════════════════════════════════════════════════════
         插件组
         简化插件调用，不用写完整 groupId
         ══════════════════════════════════════════════════════ -->

    <pluginGroups>
        <!-- 默认已包含 org.apache.maven.plugins 和 org.codehaus.mojo -->
        <!-- 配置后可以直接用 tomcat7:run 而不是 org.apache.tomcat.maven:tomcat7-maven-plugin:run -->
        <pluginGroup>org.apache.tomcat.maven</pluginGroup>
        <pluginGroup>com.spotify</pluginGroup>
    </pluginGroups>

</settings>
```

### Tips

#### repository

- repository 只能在 profile 里配置， setting 里的优先级低于 pom 文件里的 repository。profile 没有 active 的话 repository 不会生效。

生效机制和优先级

Maven 解析依赖时，仓库的查找顺序如下：

```
1. 本地仓库（~/.m2/repository）
        ↓ 未命中
2. settings.xml activeProfiles 里的 repository，多个 profile 激活时，后面的优先级更高
        ↓ 未命中
3. pom.xml 里的 repository
        ↓ 未命中
4. Super POM 里的 central（内置，始终存在）
        ↓ 未命中
5. 报错
```

#### mirrors 与私服

私服有几个作用：
- 代理外网仓库（Proxy Repository）
```
开发机 → Nexus Proxy → Maven Central / 阿里云 / ...
                ↓
         缓存到 Nexus 本地
                ↓
         下次相同请求直接从 Nexus 返回，不再访问外网
```

- 托管私有包（Hosted Repository）
- Group 聚合（Group Repository）
把多个仓库合并成一个虚拟入口，客户端只配一个地址：

```
maven-public (Group)
  ├── maven-releases  (Hosted)
  ├── maven-snapshots (Hosted)
  └── maven-central   (Proxy → Maven Central)

settings.xml 只需要配一个地址：
  http://nexus.company.com/repository/maven-public/
```

