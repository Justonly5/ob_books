### JDK 17 vs JDK 21 核心差异

#### 语言特性

|特性|JDK 17|JDK 21|
|---|---|---|
|Sealed Classes|正式版|正式版|
|Pattern Matching for instanceof|正式版|正式版|
|Records|正式版|正式版|
|Switch 表达式|正式版|正式版|
|Pattern Matching for Switch|预览版|**正式版**|
|Record Patterns|不支持|**正式版**|
|字符串模板|不支持|预览版|

---

#### 最重要的新增：虚拟线程（JDK 21 正式版）

这是 JDK 21 最大的变化，对 Spring Boot 影响最直接。

**传统平台线程（JDK 17）**：

```
每个请求 → 占用一个 OS 线程 → 等待 IO 时线程阻塞
Tomcat 200 线程 → 最多处理 200 个并发请求
```

**虚拟线程（JDK 21）**：

```
每个请求 → 一个虚拟线程 → 等待 IO 时自动挂起，释放 OS 线程
少量 OS 线程 → 支撑数万个并发虚拟线程
```

Spring Boot 3.2+ 一行配置开启：


```yaml
spring:
  threads:
    virtual:
      enabled: true   # Tomcat/Jetty 自动切换到虚拟线程
```

或者代码方式：


```java
// 虚拟线程执行器
ExecutorService executor = Executors.newVirtualThreadPerTaskExecutor();

// 虚拟线程工厂
Thread.ofVirtual().name("vt-worker").factory();
```

虚拟线程适合 IO 密集型场景，CPU 密集型没有优势。

1. 极致的吞吐量提升（针对 I/O 密集型任务）

  * 传统的 Java 线程（Platform Threads）与操作系统线程是 1:1 映射的。每个线程需要消耗约 1MB 的栈内存，且内核切换开销很大。
   * 收益：虚拟线程是用户态线程，极其轻量（一个线程仅需几百字节到几 KB）。你可以轻松在一个普通的 JVM 进程中开启 几十万甚至上百万个线程。
   * 适用场景：像你的项目这种需要频繁调用 HTTP 接口（RestClient）、查询数据库的任务，以前受限于线程池大小（如 200 个线程），现在可以为每个请求分配一个独立线程，且几乎不占资源。

  2. 回归“同步编程模型”（告别响应式）
  
   * 过去为了追求高并发，开发者不得不学习复杂的响应式编程（如 WebFlux、Mono/Flux），这导致代码难以阅读、调试和维护。
   * 收益：有了虚拟线程，你可以继续写最简单的同步代码（如 restClient.get().execute()），但它在底层 阻塞时会自动释放载体线程。
   * 结果：你用“阻塞”的简单方式，写出了具有“非阻塞”性能的代码。

  3. 极低的内存与上下文切换开销
   * 内存：虚拟线程不预分配大块内存，而是按需动态增加。
   * 切换：虚拟线程的调度由 JVM（在用户态）管理，不涉及操作系统的系统调用（System Call）。
   * 收益：在高并发场景下，CPU 能花费更多时间在真正的业务逻辑上，而不是在调度线程上。

  4. 完美的开发体验（调试与监控）
  响应式编程最大的痛点是异常堆栈（Stack Trace）支离破碎，很难定位问题。
   * 收益：虚拟线程拥有完整的调用栈。
   * 优势：现有的调试器（Debugger）、性能分析工具（JFR）、分布式追踪（如你的 TraceId 注入）在虚拟线程下依然有效，无需修改任何代码就能像以前一样调试。

---

#### Sequenced Collections（JDK 21 新增）

新增三个接口，解决集合「有序访问」语义不统一的问题：


```java
// JDK 17 获取第一个/最后一个元素，各集合写法不同
list.get(0);             // List
deque.peekFirst();       // Deque
sortedSet.first();       // SortedSet

// JDK 21 统一接口 SequencedCollection
collection.getFirst();   // 统一
collection.getLast();    // 统一
collection.reversed();   // 反转视图，不复制数据
```

---

#### Pattern Matching for Switch 正式版



```java
// JDK 17：只能预览，不能用于生产
// JDK 21：正式可用

Object obj = ...;
String result = switch (obj) {
    case Integer i -> "整数: " + i;
    case String s when s.length() > 5 -> "长字符串: " + s;  // 带 when 守卫
    case String s -> "短字符串: " + s;
    case null -> "空值";
    default -> "其他类型";
};
```

---

#### Record Patterns（JDK 21 新增）

解构 Record，配合 switch 使用更简洁：



```java
record Point(int x, int y) {}
record Line(Point start, Point end) {}

// JDK 17：需要手动解构
if (obj instanceof Point p) {
    int x = p.x();
    int y = p.y();
}

// JDK 21：直接解构
if (obj instanceof Point(int x, int y)) {
    // 直接用 x、y
}

// 嵌套解构
if (obj instanceof Line(Point(int x1, int y1), Point(int x2, int y2))) {
    // 直接用 x1 y1 x2 y2
}
```

---

#### 其他值得关注的变化

**Generational ZGC（JDK 21）**

ZGC 在 JDK 17 已经是正式版，JDK 21 新增分代模式，GC 停顿更短，内存利用率更高：



```bash
# JDK 21 开启分代 ZGC
-XX:+UseZGC -XX:+ZGenerational
```

**String 新增方法（JDK 21）**：



```java
// 重复字符直到指定长度
"ab".repeat(3);  // "ababab" (JDK 11 已有)

// JDK 21 新增
String s = "Hello World";
s.indexOf("World", 0, s.length());  // 带边界的查找
```

**Math 新增方法（JDK 21）**：



```java
Math.clamp(value, min, max);  // 限制值在范围内，JDK 17 没有
```

---

#### 总结

|维度|JDK 17|JDK 21|
|---|---|---|
|LTS|是（2029 年 EOL）|是（2031 年 EOL）|
|虚拟线程|无|正式支持，IO 密集场景性能大幅提升|
|语言特性完整度|较完整|switch 模式匹配、Record 解构正式版|
|GC|ZGC/G1|分代 ZGC，停顿更短|
|Spring Boot 支持|3.x / 4.x|3.x / 4.x|
|迁移成本|基准|从 17 升 21 成本很低，基本无破坏性变更|

从 JDK 17 升到 JDK 21 的迁移成本很低，如果新项目没有特殊约束，**直接选 JDK 21**，虚拟线程对 IO 密集的 Web 应用收益明显。