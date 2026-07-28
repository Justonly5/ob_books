---
title: "hermes insights 命令详解"
tags: [Hermes, CLI, Insights, Analytics, Observability]
created: 2026-07-25
---

# `hermes insights` 命令详解

`hermes insights` 是 Hermes 内置的**用量分析**命令，基于 SQLite 会话数据库（`state.db`）生成全面的使用洞察报告，包括 Token 消耗、成本估算、工具调用模式、活动趋势、模型/平台分布等。

---

## 基本信息

| 项目   | 内容                                                                        |
| ---- | ------------------------------------------------------------------------- |
| 命令   | `hermes insights`                                                         |
| 源码位置 | `hermes_cli/subcommands/insights.py`（CLI 解析）<br>`agent/insights.py`（引擎实现） |
| 数据源  | `~/.hermes/state.db`（SQLite, WAL 模式）                                      |
| 输出格式 | 终端（CLI）+ 网关（精简版）                                                          |
| 来源   | 受 Claude Code 的 `/insights` 命令启发                                          |

---

## 命令行参数

```
hermes insights [options]
```

| 参数 | 默认值 | 说明 |
|------|--------|------|
| `--days` | `30` | 分析的时间窗口（天），仅查询窗口，**不删除数据** |
| `--source` | 无 | 按平台来源过滤（`cli`、`telegram`、`discord` 等） |

---

## 输出内容

### 终端输出（CLI）格式

```
╔══════════════════════════════════════════════════════════╗
║                    📊 Hermes Insights                    ║
║                   Last 30 days                           ║
╚══════════════════════════════════════════════════════════╝

  Period: Jun 25, 2026 — Jul 25, 2026

  📋 Overview
  ────────────────────────────────────────────────────────
  Sessions:          127           Messages:        3,452
  Tool calls:        891           User messages:     512
  Input tokens:      2,345,678     Output tokens:     89,012
  Total tokens:      2,434,690
  Active time:       ~2h 34m       Avg session:       ~1.2m
  Avg msgs/session:  27.2

  🤖 Models Used
  ────────────────────────────────────────────────────────
  Model                                   Sessions     Tokens
  deepseek-v4-flash-260425                      89   1,823,456
  deepseek-v4-pro-260425                        38     611,234

  📱 Platforms
  ────────────────────────────────────────────────────────
  Platform       Sessions   Messages         Tokens
  cli                 89      1,234        1,456,789
  telegram            38        456          977,901

  🔧 Top Tools
  ────────────────────────────────────────────────────────
  Tool                              Calls        %
  terminal                            345    38.7%
  read_file                           234    26.3%
  web_search                           89    10.0%
  ...

  🧠 Top Skills
  ────────────────────────────────────────────────────────
  Skill                           Loads   Edits  Last used
  openclaw-migration                 12       0    Jul 20
  github-code-review                  8       2    Jul 18
  Distinct skills: 5  Loads: 34  Edits: 4

  📅 Activity Patterns
  ────────────────────────────────────────────────────────
  Mon  ████████████        45
  Tue  ████████████        42
  ...
  Peak hours: 9AM (30), 2PM (28), 10AM (25)
  Active days: 22
  Best streak: 7 consecutive days

  🏆 Notable Sessions
  ────────────────────────────────────────────────────────
  Most messages       89 (Jul 20, sess_xxx)
  Most tokens         234,567 (Jul 18, sess_yyy)
  Most tool calls     45 (Jul 15, sess_zzz)
  Longest duration    34m (Jul 12, sess_aaa)
```

### 6 大输出模块

| 模块 | 内容 |
|------|------|
| **📋 Overview** | 会话数、消息数、工具调用数、用户消息数、I/O Token、总 Token、活跃时间、平均会话时长、平均消息数/会话 |
| **🤖 Models Used** | 每个模型的使用情况：会话数、总 Token 数 |
| **📱 Platforms** | 按平台来源的分布：会话数、消息数、Token 数 |
| **🔧 Top Tools** | 工具调用排名（Top 15）：调用次数、百分比 |
| **🧠 Top Skills** | 技能使用排名（Top 10）：加载次数、编辑次数、最后使用日期 |
| **📅 Activity Patterns** | 周活跃度柱状图、高峰时段、活跃天数、最长连续活跃天数 |
| **🏆 Notable Sessions** | 最突出的会话：消息最多、Token 最多、工具调用最多、时长最长 |

---

## 数据来源

`hermes insights` **没有独立存储**，直接查询 `state.db` 中的表：

### sessions 表

```sql
CREATE TABLE IF NOT EXISTS sessions (
    id TEXT PRIMARY KEY,
    source TEXT NOT NULL,           -- 'cli', 'telegram', 'discord', etc.
    model TEXT,                     -- 使用的模型名称
    started_at REAL NOT NULL,       -- Unix 时间戳
    ended_at REAL,                  -- Unix 时间戳（NULL = 活跃会话）
    message_count INTEGER DEFAULT 0,
    tool_call_count INTEGER DEFAULT 0,
    input_tokens INTEGER DEFAULT 0,
    output_tokens INTEGER DEFAULT 0,
    cache_read_tokens INTEGER DEFAULT 0,
    cache_write_tokens INTEGER DEFAULT 0,
    billing_provider TEXT,
    billing_base_url TEXT,
    estimated_cost_usd REAL,
    actual_cost_usd REAL,
    parent_session_id TEXT,         -- 压缩触发的会话链
    title TEXT,                     -- 标题（非 NULL 值唯一）
    api_call_count INTEGER DEFAULT 0
);
```

### messages 表

```sql
CREATE TABLE IF NOT EXISTS messages (
    id INTEGER PRIMARY KEY AUTOINCREMENT,
    session_id TEXT NOT NULL REFERENCES sessions(id),
    role TEXT NOT NULL,             -- 'user', 'assistant', 'tool'
    content TEXT,                   -- 消息文本
    tool_call_id TEXT,              -- 工具响应关联
    tool_calls TEXT,                -- JSON 数组（assistant 消息）
    tool_name TEXT,                 -- tool 角色消息的工具名
    timestamp REAL NOT NULL,
    token_count INTEGER,
    finish_reason TEXT,
    reasoning TEXT,
    reasoning_content TEXT,
    reasoning_details TEXT,         -- JSON
    codex_reasoning_items TEXT,     -- JSON
    codex_message_items TEXT        -- JSON
);
```

### 辅助表

- `session_model_usage` — 按模型维度的 Token 消耗明细（包括辅助模型如 vision/compression）

---

## 成本估算

- 通过 `agent/usage_pricing.py` 中的 `estimate_usage_cost()` 计算
- 使用 `canonical_usage()` 统一不同 Provider 的 Token 计数
- 报告 `estimated_cost_usd`（基于定价表）和 `actual_cost_usd`（来自 Provider，如有）
- 未知定价的模型单独列出（`models_without_pricing`）

---

## 查询窗口 vs 数据保留

| 操作 | 作用 | 是否删除数据 |
|------|------|------------|
| `hermes insights --days N` | 仅控制查询时间窗口 | ❌ 不删除 |
| `hermes sessions prune --older-than N` | 删除历史会话 | ✅ 永久删除 |

---

## 数据保留策略

### 默认：永久保留

会话数据在 `state.db` 中无限累积，无自动清理。大量使用（gateway + cron）可能积累到 384MB+ 数据库、68K+ 消息。

### 手动清理：`hermes sessions prune`

```bash
# 默认：删除 90 天前已结束的会话
hermes sessions prune

# 自定义保留期
hermes sessions prune --older-than 30

# 按平台过滤
hermes sessions prune --source telegram --older-than 7

# 丰富过滤：标题、模型、Token 范围、成本范围等
hermes sessions prune --older-than 60 --min-tokens 10000 --max-cost 5.0

# 预览模式（dry-run）
hermes sessions prune --older-than 90 --dry-run
```

**关键行为：**
- 仅删除**已结束**的会话，活跃会话始终保留
- 子会话的 `parent_session_id` 设为 NULL（非级联删除）
- 磁盘上的转录文件（`session_{id}.json`、`request_dump_*`）也一并删除
- 实现在 `hermes_state.py`（SessionDB 类）

### 自动清理（可选）

在 `config.yaml` 中配置，**默认关闭**：

```yaml
sessions:
  auto_prune: false             # 启用自动清理
  retention_days: 90            # 保留已结束会话的天数
  vacuum_after_prune: true      # 清理后执行 VACUUM 回收空间
  min_interval_hours: 24        # 维护操作的最小间隔
```

CLI 配置：
```bash
hermes config set sessions.auto_prune true
hermes config set sessions.retention_days 30
```

**触发时机：** CLI/gateway/cron **启动时**，非持续运行
**实现路径：** `cli.py` → `_run_state_db_auto_maintenance()` → `SessionDB.maybe_auto_prune_and_vacuum()`
**安全机制：** 使用 `state_meta` 表记录上次运行时间，异常不会阻塞启动

---

## 实现架构

```
CLI 入口
└── hermes_cli/main.py
    └── cmd_insights()                    ← 路由到 insights 子命令
        └── agent/insights.py
            └── InsightsEngine            ← 核心引擎
                ├── generate(days, source) ← 生成报告
                │   ├── _get_sessions()    ← 查询 sessions 表
                │   ├── _get_tool_usage()  ← 查询工具调用（tool_name + tool_calls JSON）
                │   ├── _get_skill_usage() ← 查询技能使用
                │   └── _get_message_stats()← 消息统计
                │
                ├── _compute_overview()    ← 概览统计
                ├── _compute_model_breakdown()  ← 模型分布
                ├── _compute_platform_breakdown() ← 平台分布
                ├── _compute_tool_breakdown()    ← 工具排名
                ├── _compute_skill_breakdown()   ← 技能排名
                ├── _compute_activity_patterns() ← 活动模式
                └── _compute_top_sessions()      ← 最突出会话
                │
                ├── format_terminal(report)  ← CLI 终端输出
                └── format_gateway(report)   ← 网关精简输出
```

### 工具调用统计的数据来源

`hermes insights` 从两个来源统计工具调用次数：

1. **`tool_name` 列** — `messages` 表中 `role='tool'` 的消息（gateway 平台填充）
2. **`tool_calls` JSON 列** — `messages` 表中 `role='assistant'` 的消息（CLI 会话使用）

合并策略：取两个来源中每个工具的最大值，避免重复计数。

---

## 典型用法

```bash
# 最近 7 天分析
hermes insights --days 7

# 最近 30 天分析（默认）
hermes insights

# 按平台过滤
hermes insights --days 30 --source telegram

# 仅查看 CLI 使用
hermes insights --days 7 --source cli

# 结合清理：先查看再清理
hermes insights --days 90
hermes sessions prune --older-than 90
```

---

## 注意事项

- **仅查看当前 profile 的会话 DB** — 不是全局 Hermes 使用
- **`--days` 是查询窗口，不是保留策略** — 不删除数据
- **自动清理默认关闭** — 避免意外丢失会话历史
- **VACUUM 阻塞写入** — 仅在启动时执行，不在活跃会话期间
- **清理仅影响已结束会话** — 活跃会话始终保留
- 成本估算基于内置定价表，可能不精确

---

## 参考

- 源码：`agent/insights.py`
- CLI 解析：`hermes_cli/subcommands/insights.py`
- 成本估算：`agent/usage_pricing.py`
- 会话清理：`hermes_state.py`（SessionDB 类）
- 官方文档：`website/docs/reference/cli-commands.md`