[Log4j – Log4j 2 Lock-free Asynchronous Loggers for Low-Latency Logging](https://logging.apache.org/log4j/2.x/manual/async.html)

# 异步日志

AsyncLogger 是专为高吞吐量和低延迟日志记录而设计的记录器。它不会在调用（应用程序）线程中执行任何 I/ O，而是尽快将工作移交给另一个线程。实际日志记录是在后台线程中执行的。它使用 LMAX Disruptor 进行线程间通信。

若要使用 AsyncLogger，请在获取 Logger 之前指定 System 属性 -DLog4jContextSelector=org. apache. logging. log4j. core. async. AsyncLoggerContextSelector ，LogManager. getLogger 返回的所有 Logger 都将是 AsyncLoggers。

**请注意，出于性能原因，默认情况下，此记录器不包括源位置。您需要在配置或任何 %class、log4j. xml 配置中的 %location 或 %line 转换模式中指定 includeLocation="true" 将产生 “？” 字符或根本不输出。**

**为了获得最佳性能，请将 AsyncLogger 与 RandomAccessFileAppender 或 RollingRandomAccessFileAppender 一起使用**，并使用 immediateFlush=false。这些 appender 内置了对 Disruptor 库使用的批处理机制的支持，并且它们将在每个批处理结束时刷新到磁盘。这意味着，即使使用 immediateFlush=false，缓冲区中也永远不会留下任何项目;所有日志事件都将以非常有效的方式写入磁盘。

配置异步日志是 Log4j2 的一个重要功能，它可以显著提高应用程序的性能，特别是在高吞吐量的日志记录场景下。异步日志通过将日志事件从主线程卸载到一个单独的线程中进行处理，从而减少了对主线程的影响。

下面是一个基本的 Log4j2 异步日志配置示例：

## 步骤1：添加 Log4j2 依赖

确保你的项目中包含了 Log4j2 和异步日志所需的依赖。对于 Maven 项目，可以在 pom.xml 中添加以下依赖：

依赖 disruptor 库。

```xml
<dependency>
    <groupId>org.apache.logging.log4j</groupId>
    <artifactId>log4j-core</artifactId>
    <version>2.17.1</version>
</dependency>
<dependency>
    <groupId>org.apache.logging.log4j</groupId>
    <artifactId>log4j-api</artifactId>
    <version>2.17.1</version>
</dependency>
<dependency>
    <groupId>org.apache.logging.log4j</groupId>
    <artifactId>log4j-slf4j-impl</artifactId>
    <version>2.17.1</version>
</dependency>
<dependency>
    <groupId>com.lmax</groupId>
    <artifactId>disruptor</artifactId>
    <version>3.4.2</version>
</dependency>
```

## 步骤2：配置 log4j2.xml

创建或更新你的 log4j2.xml 配置文件来使用异步日志。以下是一个示例配置：

```xml
<?xml version="1.0" encoding="UTF-8"?>
<Configuration status="WARN">
    <Appenders>
        <Console name="Console" target="SYSTEM_OUT">
            <PatternLayout pattern="%d{yyyy-MM-dd HH:mm:ss} %-5level %logger{36} - %msg%n"/>
        </Console>
    </Appenders>
    
    <!-- 异步RootLogger -->
    <Loggers>
        <AsyncRoot level="info">
            <AppenderRef ref="Console"/>
        </AsyncRoot>
    </Loggers>
</Configuration>
```

**说明**

1. **Appenders**：定义了一个 Console 输出的 Appender，它会将日志输出到控制台，并使用指定的日志格式。

2. **AsyncRoot**：定义了一个异步根日志器，指定了 info 级别，并引用了上面定义的 Console Appender。

## 步骤3：启用全局异步日志

为了启用全局的异步日志，你可以在 log4j2.xml 的根元素 <Configuration> 中添加 async="true" 属性：

```xml
<Configuration status="WARN" async="true">
    <Appenders>
        <Console name="Console" target="SYSTEM_OUT">
            <PatternLayout pattern="%d{yyyy-MM-dd HH:mm:ss} %-5level %logger{36} - %msg%n"/>
        </Console>
    </Appenders>
    
    <Loggers>
        <Root level="info">
            <AppenderRef ref="Console"/>
        </Root>
    </Loggers>
</Configuration>
```

## 高级配置：混合同步和异步日志

在一些场景中，你可能希望某些日志器使用异步日志，而其他日志器使用同步日志。以下是一个混合配置的示例：

```xml
<?xml version="1.0" encoding="UTF-8"?>
<Configuration status="WARN">
    <Appenders>
        <Console name="Console" target="SYSTEM_OUT">
            <PatternLayout pattern="%d{yyyy-MM-dd HH:mm:ss} %-5level %logger{36} - %msg%n"/>
        </Console>
        
        <File name="File" fileName="logs/app.log">
            <PatternLayout pattern="%d{yyyy-MM-dd HH:mm:ss} %-5level %logger{36} - %msg%n"/>
        </File>
    </Appenders>
    
    <Loggers>
        <!-- 异步日志器 -->
        <AsyncLogger name="com.example.async" level="info">
            <AppenderRef ref="Console"/>
        </AsyncLogger>
        
        <!-- 同步日志器 -->
        <Logger name="com.example.sync" level="info" additivity="false">
            <AppenderRef ref="File"/>
        </Logger>
        
        <!-- 异步RootLogger -->
        <AsyncRoot level="info">
            <AppenderRef ref="Console"/>
        </AsyncRoot>
    </Loggers>
</Configuration>
```

**说明**

1. **AsyncLogger**：定义了一个异步日志器 com.example.async，指定了 info 级别，并引用了 Console Appender。

2. **Logger**：定义了一个同步日志器 com.example.sync，指定了 info 级别，并引用了 File Appender。

3. **AsyncRoot**：定义了一个异步根日志器，指定了 info 级别，并引用了 Console Appender。

通过这种方式，你可以灵活地配置哪些日志器使用异步日志，哪些日志器使用同步日志。

# 异步日志问题

## 问题点

Log4j2 的异步日志功能可以显著提高日志记录的性能，因为它避免了应用程序在写入日志时阻塞等待 I/O 操作完成。然而，异步日志也会带来一些潜在的问题，这些问题包括但不限于：

1. **日志顺序问题**：

- **日志顺序错乱**：异步日志可能会导致日志消息的顺序与它们在代码中出现的顺序不一致。这是因为异步队列中的日志消息可能会以不同的顺序被处理。
- **日志丢失**：在极端情况下，如果应用程序崩溃或 JVM 关闭，未被处理的日志消息可能会丢失。

2. **资源管理问题**：

- **内存使用增加**：异步日志使用内存队列来缓存日志消息，如果队列过大或日志消息产生速度过快，可能会导致内存使用量增加。
- **线程池配置不当**：如果不正确配置异步日志处理器的线程池，可能会导致性能下降或资源争用。

3. **配置复杂性**：

- **配置难度增加**：使用异步日志需要更复杂的配置，包括设置队列大小、溢出策略、线程池大小等。
- **调试困难**：当出现问题时，调试异步日志配置可能比同步日志配置更加困难。

4. **调试和故障定位问题**：

- **日志顺序错乱**：如前所述，日志顺序的错乱可能会影响调试和故障定位的过程，特别是当依赖于日志顺序来追踪问题时。
- **日志丢失**：日志丢失可能会导致重要信息的缺失，从而影响问题的诊断。

5. **性能问题**：

- **过度配置**：如果队列和线程池配置得过大，可能会导致资源浪费。
- **队列溢出**：如果队列满了而没有适当的溢出策略，新的日志消息可能会被丢弃，导致日志丢失。

6. **安全性问题**：

- **敏感数据泄露**：在某些情况下，如果敏感数据被记录在日志中，可能会导致安全问题。异步日志处理可能需要额外的安全措施来保护这些数据。

7. **稳定性问题**：

- **死锁风险**：在某些配置下，不当的线程池配置或队列管理可能会导致死锁。
- **资源耗尽**：如果队列或线程池配置不当，可能会导致资源耗尽，从而影响整个应用程序的稳定性。

## 解决方案和最佳实践

为了缓解这些问题，可以采取以下一些解决方案和最佳实践：

- **配置适当的队列大小**：确保队列大小足以容纳高峰期的日志消息数量，同时避免占用过多内存。
- **设置溢出策略**：配置适当的溢出策略，例如 `DISCARD`（丢弃多余的日志消息）或 `BLOCK`（阻止新的日志消息进入队列直到有足够的空间）。
- **合理配置线程池**：根据预期的日志生成速率和系统资源，合理配置线程池的大小。
- **启用日志序列化**：在 Log4j2 中，可以启用序列化来保证日志消息的顺序。
- **使用日志持久化**：考虑使用日志持久化技术，如 `MemoryAppender` 和 `RollingFileAppender` 组合，以减少日志丢失的风险。
- **监控和报警**：实施监控和报警机制来检测日志队列的状态和性能指标，以便及时发现并解决问题。

通过这些措施，可以最大限度地发挥异步日志的优势，同时减轻潜在的问题。

# 队列、线程池配置

在 Log4j2 中配置异步日志记录涉及到配置一个异步日志 Appender 以及相关的线程池和队列。以下是一个基本的配置示例，说明如何配置异步日志 Appender 的线程池和队列。

## 配置文件结构

Log4j2 的配置文件通常是 XML 或 JSON 格式。这里以 XML 格式为例：

```xml
<?xml version="1.0" encoding="UTF-8"?>
<Configuration status="WARN">
    <Appenders>
        <!-- 定义异步日志 Appender -->
        <Async name="ASYNC" queueSize="1000">
            <appender-ref ref="RollingFile"/>
        </Async>

        <!-- 定义同步的 RollingFile Appender -->
        <RollingFile name="RollingFile" fileName="logs/app.log"
                     filePattern="logs/$${date:yyyy-MM}/app-%d{MM-dd-yyyy}-%i.log.gz">
            <PatternLayout>
                <pattern>%d{HH:mm:ss.SSS} [%t] %-5level %logger{36} - %msg%n</pattern>
            </PatternLayout>
            <Policies>
                <TimeBasedTriggeringPolicy/>
                <SizeBasedTriggeringPolicy size="10 MB"/>
            </Policies>
            <DefaultRolloverStrategy max="20"/>
        </RollingFile>
    </Appenders>

    <Loggers>
        <Root level="info">
            <AppenderRef ref="ASYNC"/>
        </Root>
    </Loggers>
</Configuration>
```

## 配置解释

1. **异步 Appender (**`**Async**`**)**:

- `<Async name="ASYNC" queueSize="1000">`: 这定义了一个名为 `ASYNC` 的异步 Appender。`queueSize` 属性指定了内部队列的最大容量，这里设置为 1000 条消息。

2. **同步 Appender (**`**RollingFile**`**)**:

- `<RollingFile name="RollingFile" ...>`: 这定义了一个滚动文件 Appender，用于将日志写入到磁盘上。

3. **引用同步 Appender**:

- `<appender-ref ref="RollingFile"/>`: 在异步 Appender 中引用了 `RollingFile` Appender。

4. **配置日志级别**:

- `<Root level="info">`: 设置根日志器的日志级别为 `info`。

5. **使用异步 Appender**:

- `<AppenderRef ref="ASYNC"/>`: 在根日志器中引用了 `ASYNC` Appender。

## 高级配置选项

### 配置线程池

你可以进一步定制线程池的行为，例如指定线程池的大小、拒绝策略等。这通常通过使用 `ThreadPools` 元素实现：

```xml
<Configuration status="WARN">
    <ThreadPools>
        <ThreadPool name="AsyncThreadPool" size="10" maxQueueSize="1000" blocking="false" 
          reject="Abort">
            <!-- Other ThreadPool configurations here -->
        </ThreadPool>
    </ThreadPools>

    <Appenders>
        <Async name="ASYNC" threadPool="AsyncThreadPool" queueSize="1000">
            <appender-ref ref="RollingFile"/>
        </Async>

        <!-- RollingFile Appender configuration here -->
    </Appenders>

    <Loggers>
        <Root level="info">
            <AppenderRef ref="ASYNC"/>
        </Root>
    </Loggers>
</Configuration>
```

- `name="AsyncThreadPool"`: 线程池的名字。
- `size="10"`: 线程池中的线程数量。
- `maxQueueSize="1000"`: 队列的最大大小。
- `blocking="false"`: 当队列满时是否阻塞等待。
- `reject="Abort"`: 当队列满并且不允许阻塞时，如何处理新来的日志事件。

#### 配置队列

在上面的例子中，我们已经设置了 `queueSize` 属性来限制队列的大小。如果你需要更高级的队列配置，可以使用 `BlockingQueue` 元素：

```xml
<Configuration status="WARN">
    <ThreadPools>
        <ThreadPool name="AsyncThreadPool" size="10" maxQueueSize="1000" blocking="false" reject="Abort">
            <BlockingQueue type="LinkedBlockingQueue" capacity="1000"/>
        </ThreadPool>
    </ThreadPools>

    <Appenders>
        <Async name="ASYNC" threadPool="AsyncThreadPool">
            <appender-ref ref="RollingFile"/>
        </Async>

        <!-- RollingFile Appender configuration here -->
    </Appenders>

    <Loggers>
        <Root level="info">
            <AppenderRef ref="ASYNC"/>
        </Root>
    </Loggers>
</Configuration>
```

- `type="LinkedBlockingQueue"`: 使用 LinkedBlockingQueue 类型的队列。
- `capacity="1000"`: 队列的最大容量。

以上就是配置 Log4j2 异步日志的基本方法。你可以根据你的需求调整这些配置选项来优化日志记录的性能和可靠性。