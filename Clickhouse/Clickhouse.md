### ClickHouse 集群架构

```
ClickHouse 集群 = 多个分片（Shard）+ 每个分片多个副本（Replica）

典型 3分片2副本 集群：
  Shard1: Node1(主) + Node2(副)
  Shard2: Node3(主) + Node4(副)
  Shard3: Node5(主) + Node6(副)

数据按分片规则分散存储：
  订单1 → Shard1
  订单2 → Shard2
  订单3 → Shard3
```

---

### 本地表（Local Table）

![[Pasted image 20260513183035.png]]


### 本地表（Local Table）

**每个节点上真实存储数据的表**，是 ClickHouse 中实际存在的物理表。


```sql
-- 在每个节点上分别创建，表名通常加 _local 后缀
CREATE TABLE events_local ON CLUSTER my_cluster
(
    event_date  Date,
    user_id     UInt64,
    event_type  LowCardinality(String),
    amount      Decimal(10,2)
)
ENGINE = ReplicatedMergeTree(
    '/clickhouse/tables/{shard}/events',  -- ZooKeeper 路径，{shard} 自动替换
    '{replica}'                           -- 副本标识，自动替换
)
PARTITION BY toYYYYMM(event_date)
ORDER BY (event_date, user_id);
```

关键点：

- 数据只存在这个节点的这张表里
- `ON CLUSTER` 让建表语句在集群所有节点自动执行
- `ReplicatedMergeTree` 负责分片内主副本之间的数据同步

---

### 分布式表 / 集群视图（Distributed Table）

**本身不存数据，是一个路由层**，把读写请求分发到各个分片的本地表。



```sql
-- 建在任意一个节点上即可（通常每个节点都建，方便任意节点接入）
CREATE TABLE events_distributed ON CLUSTER my_cluster
AS events_local  -- 结构和本地表一致
ENGINE = Distributed(
    my_cluster,     -- 集群名，对应 config.xml 里的配置
    default,        -- 数据库名
    events_local,   -- 本地表名
    rand()          -- sharding key，决定数据写入哪个分片
                    -- rand() = 随机，cityHash64(user_id) = 按用户分片
);
```

写入流程：

```
INSERT INTO events_distributed
    → Distributed 引擎计算 sharding key
    → 路由到对应分片的 events_local
    → 分片内 ReplicatedMergeTree 同步到副本
```

查询流程：

```
SELECT FROM events_distributed
    → 向所有分片的 events_local 并行发送子查询
    → 收集结果合并返回
```

---

### 分区（Partition）

**本地表内部按某个维度切分的物理数据块**，是 ClickHouse 管理数据的基本单位。


```sql
-- PARTITION BY 定义分区规则
PARTITION BY toYYYYMM(event_date)  -- 按月分区
-- 实际存储：
--   /data/events_local/202401/   ← 一个分区一个目录
--   /data/events_local/202402/
--   /data/events_local/202403/
```

分区的核心价值是**分区裁剪（Partition Pruning）**：


```sql
-- 查询时带分区键条件，ClickHouse 直接跳过不相关分区
SELECT * FROM events_local
WHERE event_date >= '2024-01-01'
AND   event_date <  '2024-02-01';
-- 只读取 202401 目录，其他月份的数据完全不访问
```

分区操作：


```sql
-- 按分区删除（效率极高，推荐替代 DELETE）
ALTER TABLE events_local ON CLUSTER my_cluster
DROP PARTITION '202401';

-- 查看所有分区
SELECT partition, count(), sum(rows), formatReadableSize(sum(bytes_on_disk))
FROM system.parts
WHERE table = 'events_local' AND active
GROUP BY partition
ORDER BY partition;
```

---

### 三者关系总结

```
写入：
  App → Distributed表（路由） → 各分片 本地表 → 分区目录

查询：
  App → Distributed表 → 并行查各分片 本地表 → 利用分区裁剪跳过无关分区

删除：
  直接操作本地表的分区 → DROP PARTITION（最快）
  或通过 Distributed 表 → ALTER DELETE（异步慢）
```


### 建表最佳实践

#### 标准建表流程：先本地表，再分布式表



```sql
-- 第一步：建本地表（集群所有节点同时执行）
CREATE TABLE events_local ON CLUSTER my_cluster
(
    -- 分区键放第一位，查询时触发分区裁剪
    event_date      Date,
    event_time      DateTime,
    user_id         UInt64,
    -- 低基数字段用 LowCardinality，节省存储和加速查询
    event_type      LowCardinality(String),
    platform        LowCardinality(String),
    amount          Decimal(18, 4)          DEFAULT 0,
    -- 避免 Nullable，用默认值替代
    remark          String                  DEFAULT '',
    -- 数据版本，ReplacingMergeTree 去重用
    version         UInt64                  DEFAULT toUnixTimestamp(now()),
    -- 软删除标记
    is_deleted      UInt8                   DEFAULT 0
)
-- 集群环境用 ReplicatedMergeTree，自动处理副本同步
ENGINE = ReplicatedMergeTree(
    '/clickhouse/tables/{shard}/events',
    '{replica}'
)
-- 按月分区，平衡分区数量和查询效率
-- 分区太细（按天）→ 分区数爆炸；分区太粗（按年）→ 裁剪效果差
PARTITION BY toYYYYMM(event_date)
-- ORDER BY 是数据物理排序，最常用的查询维度放前面
-- 基数低的字段放前面（event_type），基数高的放后面（user_id）
ORDER BY (event_date, event_type, user_id)
-- 主键可以是 ORDER BY 的前缀子集，用于索引
PRIMARY KEY (event_date, event_type)
-- 稀疏索引粒度，默认 8192，OLAP 场景可适当调大
SETTINGS index_granularity = 8192;
```


```sql
-- 第二步：建分布式表
CREATE TABLE events ON CLUSTER my_cluster
AS events_local
ENGINE = Distributed(
    my_cluster,
    currentDatabase(),
    events_local,
    -- sharding key 选择原则：
    -- rand()              → 数据均匀分布，但同一用户数据分散
    -- cityHash64(user_id) → 同一用户数据在同一分片，聚合查询更快
    cityHash64(user_id)
);
```

---

#### IF NOT EXISTS 防止重复创建



```sql
-- 幂等建表，多次执行不报错
CREATE TABLE IF NOT EXISTS events_local ON CLUSTER my_cluster
(
    event_date  Date,
    user_id     UInt64,
    event_type  LowCardinality(String)
)
ENGINE = ReplicatedMergeTree('/clickhouse/tables/{shard}/events', '{replica}')
PARTITION BY toYYYYMM(event_date)
ORDER BY (event_date, user_id);
```

---

#### 需要去重时用 ReplacingMergeTree



```sql
CREATE TABLE orders_local ON CLUSTER my_cluster
(
    order_id        UInt64,
    order_date      Date,
    status          LowCardinality(String),
    amount          Decimal(18,4),
    updated_at      UInt64,        -- 版本号，值越大越新
    is_deleted      UInt8 DEFAULT 0
)
ENGINE = ReplicatedReplacingMergeTree(
    '/clickhouse/tables/{shard}/orders',
    '{replica}',
    updated_at         -- 指定版本列，保留最大值的那条
)
PARTITION BY toYYYYMM(order_date)
ORDER BY order_id;     -- 去重 key，相同 order_id 只保留最新版本
```

查询时加 `FINAL` 触发去重：



```sql
-- FINAL 强制合并去重，数据量大时有性能损耗
SELECT * FROM orders FINAL WHERE is_deleted = 0;

-- 或者用 argMax 替代 FINAL，性能更好
SELECT
    order_id,
    argMax(status,     updated_at) AS status,
    argMax(amount,     updated_at) AS amount,
    argMax(is_deleted, updated_at) AS is_deleted
FROM orders
GROUP BY order_id
HAVING is_deleted = 0;
```

---

#### 常用辅助索引



```sql
CREATE TABLE events_local ON CLUSTER my_cluster
(
    event_date  Date,
    user_id     UInt64,
    event_type  LowCardinality(String),
    trace_id    String,

    -- 跳数索引：加速非 ORDER BY 列的查询
    -- bloom_filter 适合字符串等值查询
    INDEX idx_trace_id trace_id TYPE bloom_filter(0.01) GRANULARITY 4,
    -- minmax 适合数值范围查询
    INDEX idx_user_id  user_id  TYPE minmax GRANULARITY 4
)
ENGINE = ReplicatedMergeTree('/clickhouse/tables/{shard}/events', '{replica}')
PARTITION BY toYYYYMM(event_date)
ORDER BY (event_date, event_type);
```

---

### 删表最佳实践

#### 正确的删表顺序：先删分布式表，再删本地表



```sql
-- 第一步：先删分布式表（路由层）
-- 必须先删，否则本地表删了之后分布式表还在，写入会报错
DROP TABLE IF EXISTS events ON CLUSTER my_cluster;

-- 第二步：再删本地表
DROP TABLE IF EXISTS events_local ON CLUSTER my_cluster;
```

顺序反了的后果：

```
错误顺序：先删本地表 → 后删分布式表
  → 分布式表路由到已删除的本地表
  → 写入报错：Table default.events_local doesn't exist
  → 查询报错：所有分片均无法响应
```

---

#### SYNC 关键字等待确认



```sql
-- 默认 DROP 是异步的，加 SYNC 等待集群所有节点完成
DROP TABLE IF EXISTS events ON CLUSTER my_cluster SYNC;
DROP TABLE IF EXISTS events_local ON CLUSTER my_cluster SYNC;
```

不加 `SYNC` 的风险：

```
DROP 返回成功，但部分节点还没执行完
  → 立即重建同名表可能在部分节点失败
  → 加 SYNC 确保所有节点都删完再继续
```

---

#### 只清数据不删表用 TRUNCATE



```sql
-- 清空所有数据，保留表结构
TRUNCATE TABLE events_local ON CLUSTER my_cluster;

-- 只清特定分区（更常用，只删历史数据）
ALTER TABLE events_local ON CLUSTER my_cluster
DROP PARTITION '202401';
```

---

#### ZooKeeper 路径清理（ReplicatedMergeTree 必须注意）

`ReplicatedMergeTree` 在 ZooKeeper 里存了元数据，删表后 ZooKeeper 里的路径不会自动清理：



```sql
-- ClickHouse 21.8+ 支持删表时自动清理 ZooKeeper 路径
-- 建议开启，否则 ZooKeeper 会堆积垃圾路径
DROP TABLE IF EXISTS events_local ON CLUSTER my_cluster SYNC;
-- 新版本自动处理，旧版本需要手动清理 ZooKeeper 对应路径
```

验证 ZooKeeper 路径是否残留：

sql

```sql
-- 查看当前存活的 ZooKeeper 路径
SELECT * FROM system.zookeeper
WHERE path = '/clickhouse/tables';
```

---

### 完整建删表流程汇总



```sql
-- ══ 建表（顺序：本地 → 分布式）══════════════════

CREATE TABLE IF NOT EXISTS events_local ON CLUSTER my_cluster
( ... )
ENGINE = ReplicatedMergeTree(...)
PARTITION BY toYYYYMM(event_date)
ORDER BY (event_date, user_id);

CREATE TABLE IF NOT EXISTS events ON CLUSTER my_cluster
AS events_local
ENGINE = Distributed(my_cluster, currentDatabase(), events_local, cityHash64(user_id));


-- ══ 删表（顺序：分布式 → 本地）══════════════════

DROP TABLE IF EXISTS events       ON CLUSTER my_cluster SYNC;
DROP TABLE IF EXISTS events_local ON CLUSTER my_cluster SYNC;


-- ══ 只删数据保留结构 ══════════════════════════

-- 删指定分区（推荐，精确）
ALTER TABLE events_local ON CLUSTER my_cluster DROP PARTITION '202401';

-- 清空整表
TRUNCATE TABLE events_local ON CLUSTER my_cluster;
```

---

### 关键原则速查

|操作|原则|
|---|---|
|建表|先本地表再分布式表，加 `IF NOT EXISTS`|
|删表|先分布式表再本地表，加 `SYNC`|
|清数据|优先 `DROP PARTITION`，避免 `ALTER TABLE DELETE`|
|引擎选择|单机用 `MergeTree`，集群用 `ReplicatedMergeTree`|
|需要去重|`ReplicatedReplacingMergeTree` + `argMax` 查询|
|避免 Nullable|用 `DEFAULT ''` 或 `DEFAULT 0` 替代|