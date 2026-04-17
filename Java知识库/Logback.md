## 完整 logback-spring.xml 最佳实践

```XML
<?xml version="1.0" encoding="UTF-8"?>
<configuration scan="true" scanPeriod="60 seconds">

    <!-- 从 application.yml 读取 -->
    <springProperty scope="context" name="APP_NAME" source="spring.application.name" defaultValue="app"/>
    <springProperty scope="context" name="LOG_HOME" source="logging.file.path" defaultValue="/var/log"/>
    <springProperty scope="context" name="LOG_LEVEL" source="logging.level.root" defaultValue="INFO"/>

    <!-- Pod 标识（K8s 场景） -->
    <property scope="context" name="POD_NAME" value="${POD_NAME:-local}"/>

    <!-- 日志格式 -->
    <property name="CONSOLE_PATTERN"
              value="%d{yyyy-MM-dd HH:mm:ss.SSS} [%thread] %-5level %logger{36} [%X{traceId}] - %msg%n"/>
    <property name="FILE_PATTERN"
              value="%d{yyyy-MM-dd HH:mm:ss.SSS} [%thread] %-5level %logger{36} [%X{traceId}] - %msg%n"/>

    <!-- ======================== Appender ======================== -->

    <!-- 控制台 -->
    <appender name="CONSOLE" class="ch.qos.logback.core.ConsoleAppender">
        <encoder>
            <pattern>${CONSOLE_PATTERN}</pattern>
            <charset>UTF-8</charset>
        </encoder>
    </appender>

    <!-- 文件（同步，作为异步的底层） -->
    <appender name="FILE" class="ch.qos.logback.core.rolling.RollingFileAppender">
        <file>${LOG_HOME}/${APP_NAME}-${POD_NAME}.log</file>
        <rollingPolicy class="ch.qos.logback.core.rolling.SizeAndTimeBasedRollingPolicy">
            <fileNamePattern>
                ${LOG_HOME}/${APP_NAME}-${POD_NAME}.%d{yyyy-MM-dd}.%i.log.gz
            </fileNamePattern>
            <maxFileSize>100MB</maxFileSize>
            <maxHistory>7</maxHistory>
            <totalSizeCap>10GB</totalSizeCap>
            <cleanHistoryOnStart>true</cleanHistoryOnStart>
        </rollingPolicy>
        <encoder>
            <pattern>${FILE_PATTERN}</pattern>
            <charset>UTF-8</charset>
        </encoder>
    </appender>

    <!-- 错误单独文件，方便排查 -->
    <appender name="ERROR_FILE" class="ch.qos.logback.core.rolling.RollingFileAppender">
        <file>${LOG_HOME}/${APP_NAME}-${POD_NAME}-error.log</file>
        <filter class="ch.qos.logback.classic.filter.LevelFilter">
            <level>ERROR</level>
            <onMatch>ACCEPT</onMatch>
            <onMismatch>DENY</onMismatch>
        </filter>
        <rollingPolicy class="ch.qos.logback.core.rolling.SizeAndTimeBasedRollingPolicy">
            <fileNamePattern>
                ${LOG_HOME}/${APP_NAME}-${POD_NAME}-error.%d{yyyy-MM-dd}.%i.log.gz
            </fileNamePattern>
            <maxFileSize>100MB</maxFileSize>
            <maxHistory>30</maxHistory>
            <totalSizeCap>5GB</totalSizeCap>
            <cleanHistoryOnStart>true</cleanHistoryOnStart>
        </rollingPolicy>
        <encoder>
            <pattern>${FILE_PATTERN}</pattern>
            <charset>UTF-8</charset>
        </encoder>
    </appender>

    <!-- ======================== 异步包装 ======================== -->

    <!-- 异步控制台（开发环境意义不大，生产可去掉） -->
    <appender name="ASYNC_CONSOLE" class="ch.qos.logback.classic.AsyncAppender">
        <appender-ref ref="CONSOLE"/>
        <queueSize>512</queueSize>
        <discardingThreshold>0</discardingThreshold>
        <includeCallerData>false</includeCallerData>
        <neverBlock>true</neverBlock>
    </appender>

    <!-- 异步文件 -->
    <appender name="ASYNC_FILE" class="ch.qos.logback.classic.AsyncAppender">
        <appender-ref ref="FILE"/>
        <queueSize>1024</queueSize>
        <!-- 0 表示队列满时不丢弃任何级别日志，默认是队列剩 20% 时丢弃 WARN 以下 -->
        <discardingThreshold>0</discardingThreshold>
        <!-- false 表示队列满时阻塞等待，不丢日志；true 表示直接丢弃，不阻塞业务线程 -->
        <neverBlock>false</neverBlock>
        <!-- 不收集调用者信息，避免性能损耗 -->
        <includeCallerData>false</includeCallerData>
        <!-- 应用关闭时最多等 2s 把队列里的日志刷完 -->
        <maxFlushTime>2000</maxFlushTime>
    </appender>

    <!-- 异步错误文件 -->
    <appender name="ASYNC_ERROR_FILE" class="ch.qos.logback.classic.AsyncAppender">
        <appender-ref ref="ERROR_FILE"/>
        <queueSize>256</queueSize>
        <discardingThreshold>0</discardingThreshold>
        <neverBlock>false</neverBlock>
        <maxFlushTime>2000</maxFlushTime>
    </appender>

    <!-- ======================== 按环境切换 ======================== -->

    <springProfile name="dev,local">
        <root level="DEBUG">
            <appender-ref ref="CONSOLE"/>
        </root>
        <!-- 开发时只看控制台，不写文件 -->
    </springProfile>

    <springProfile name="test">
        <root level="INFO">
            <appender-ref ref="ASYNC_CONSOLE"/>
            <appender-ref ref="ASYNC_FILE"/>
            <appender-ref ref="ASYNC_ERROR_FILE"/>
        </root>
    </springProfile>

    <springProfile name="prod">
        <root level="INFO">
            <appender-ref ref="ASYNC_FILE"/>
            <appender-ref ref="ASYNC_ERROR_FILE"/>
        </root>
    </springProfile>

    <!-- ======================== 降噪配置 ======================== -->
    <logger name="org.springframework" level="WARN"/>
    <logger name="org.hibernate" level="WARN"/>
    <logger name="com.zaxxer.hikari" level="WARN"/>
    <logger name="io.lettuce" level="WARN"/>
    <logger name="org.apache.http" level="WARN"/>

    <!-- 业务包保持 INFO，方便排查 -->
    <logger name="com.yourcompany" level="INFO"/>

</configuration>
```

## 异步核心参数说明

`AsyncAppender` 几个参数直接影响日志可靠性和性能，需要根据业务场景做取舍：

|参数|默认值|说明|
|---|---|---|
|`queueSize`|256|队列容量，建议 512~2048|
|`discardingThreshold`|80|队列剩余低于此百分比丢弃 WARN 以下，设 0 不丢弃|
|`neverBlock`|false|true=队列满直接丢，false=阻塞等待|
|`includeCallerData`|false|true=记录行号类名，有性能损耗|
|`maxFlushTime`|1000|应用关闭时等待队列清空的最长时间(ms)|

**`neverBlock` 和 `discardingThreshold` 的取舍：**

```
日志不能丢（金融/审计）：
  discardingThreshold=0，neverBlock=false
  → 队列满时业务线程阻塞等待，日志不丢但可能影响响应时间

日志可以丢（普通业务）：
  discardingThreshold=20，neverBlock=true
  → 队列满时直接丢弃，业务线程不受影响
```

---

## 异步日志的一个隐患

应用被 `kill -9` 强杀时，队列里还没写入文件的日志会直接丢失。K8s 滚动更新默认发 `SIGTERM`，Logback 能捕获并在 `maxFlushTime` 内刷完队列；但如果超过 `terminationGracePeriodSeconds` 被强杀就没办法了，确保 K8s 的优雅退出时间足够：


```yaml
# deployment.yaml
spec:
  template:
    spec:
      terminationGracePeriodSeconds: 30  # 要大于 maxFlushTime
```