---
title: Hermes Agent 核心概念与架构
tags: [hermes, architecture, tools, skills, database, insights]
created: 2026-07-22
---

# Hermes Agent 核心概念与架构

## 一、Tools vs Skills

### 核心区别

| 维度 | **Tools** | **Skills** |
|------|-----------|------------|
| **本质** | 结构化函数（LLM 可调用的 API） | 指令文档（agent 阅读的指南） |
| **谁来调用** | LLM 直接调用（`tool_calls`） | agent 阅读后，用已有 tools 执行 |
| **注册方式** | `registry.register()` 代码注册 | 放 SKILL.md 到 `~/.hermes/skills/` 目录 |
| **Schema** | JSON Schema（name, description, parameters） | YAML 前置元数据（name, description, tags, platforms） |
| **加载时机** | 每个会话启动时自动加载 | 按需加载（`skill_view()` 调用时才进上下文） |
| **Token 成本** | 固定开销（每个会话都发送给 LLM） | 零开销直到被加载 |
| **可编写者** | 开发者（Python 代码） | 任何人（Markdown 文件） |
| **需要什么** | Python 代码 + 注册 | 纯 Markdown 文件 |
| **典型场景** | 终端执行、文件操作、浏览器、搜索 | 代码审查流程、K8s 部署指南、Git 工作流 |
| **架构优先级** | 最后手段（Footprint Ladder 第 6 级） | 首选方案（Footprint Ladder 第 2 级） |

### 一句话

> **Tool 让 LLM 能做一件事，Skill 教 LLM 怎么做一件事。**

### Footprint Ladder（能力决策阶梯）

```
1. 扩展已有代码          ← 0 新增表面
2. CLI 命令 + 技能       ← ← ← 新能力首选
3. 条件门控工具          ← 有条件时才出现
4. 插件                  ← 第三方/用户自定义
5. MCP 服务器            ← 跨平台可复用
6. 新增核心工具          ← ← ← 最后手段
```

---

## 二、默认 Tools 完整列表（`_HERMES_CORE_TOOLS`）

共约 **45 个工具**，按功能分类：

### 🌐 Web 搜索与内容提取
| 工具名 | 功能 |
|--------|------|
| `web_search` | 搜索互联网，返回标题/URL/摘要，支持 `site:`、`filetype:` 等高级语法 |
| `web_extract` | 提取网页内容为 Markdown，支持 PDF URL |

### 💻 终端与进程管理
| 工具名 | 功能 |
|--------|------|
| `terminal` | 执行 Shell 命令（支持后台、超时、PTY 模式） |
| `process` | 管理后台进程：list/poll/log/wait/kill 等 |
| `read_terminal` | 读取 Desktop 内嵌终端面板内容（仅 Desktop GUI） |
| `close_terminal` | 关闭后台进程只读标签页（仅 Desktop GUI） |

### 📁 文件操作
| 工具名 | 功能 |
|--------|------|
| `read_file` | 读取文本文件（带行号、分页），支持 .ipynb/.docx/.xlsx 提取 |
| `write_file` | 写入/覆盖文件，自动语法检查 |
| `patch` | 精确查找替换编辑（模糊匹配 9 种策略） |
| `search_files` | rg 引擎搜索文件内容或按文件名查找 |

### 🖼️ 视觉与图像
| 工具名 | 功能 |
|--------|------|
| `vision_analyze` | 分析图片（主模型视觉或辅助模型回退） |
| `image_generate` | 文生图/图生图（FAL.ai / OpenAI / xAI / Krea） |

### 🧠 技能管理
| 工具名 | 功能 |
|--------|------|
| `skills_list` | 列出所有已安装技能 |
| `skill_view` | 加载单个技能详情 |
| `skill_manage` | 创建/更新/删除技能 |

### 🌍 浏览器自动化（12 个）
| 工具名 | 功能 |
|--------|------|
| `browser_navigate` | 加载 URL |
| `browser_snapshot` | 获取页面无障碍树快照 |
| `browser_click` | 点击元素（ref ID） |
| `browser_type` | 输入框输入文本 |
| `browser_scroll` | 上下滚动 |
| `browser_back` | 后退 |
| `browser_press` | 按键（Enter、Tab 等） |
| `browser_get_images` | 获取页面图片列表 |
| `browser_vision` | 截图 + 视觉分析 |
| `browser_console` | 获取控制台输出 |
| `browser_cdp` | 原始 CDP 命令（需 CDP 端点） |
| `browser_dialog` | 响应 JS 对话框 |

### 🎤 语音合成
| 工具名 | 功能 |
|--------|------|
| `text_to_speech` | 文本转语音（Edge / ElevenLabs / OpenAI / MiniMax / Mistral） |

### 📋 任务规划与记忆
| 工具名 | 功能 |
|--------|------|
| `todo` | 管理会话任务列表 |
| `memory` | 持久化跨会话记忆 |

### 🔍 其他
| 工具名 | 功能 |
|--------|------|
| `session_search` | FTS5 全文检索历史会话 |
| `clarify` | 向用户发起澄清提问 |
| `execute_code` | 执行 Python 脚本（可调用 Hermes 工具 API） |
| `delegate_task` | 派生子 agent 隔离工作 |
| `cronjob` | 管理定时任务 |

### 条件门控工具（需配置才显示）
- Home Assistant 4 个（`ha_list_entities`, `ha_get_state`, `ha_list_services`, `ha_call_service`）
- Kanban 11 个（多 agent 协作看板）
- Computer Use 1 个（`computer_use`，需 `cua-driver`）

> 实际每次 API 调用发送约 **35~40 个工具**（受 `check_fn` 条件门控影响）。

---

## 三、Insight 数据存储

### 数据来源
`hermes insights` 没有独立存储，直接查询 `~/.hermes/state.db` 中的 `sessions`、`messages`、`session_model_usage` 表实时计算。

### 存储多久
- **默认永久存储**，没有自动删除
- 提供两种清理机制：

### 1. 手动清理
```bash
hermes sessions prune                    # 默认 90 天前已结束的会话
hermes sessions prune --older-than 30    # 自定义保留天数
hermes sessions prune --source telegram --older-than 7   # 按来源过滤
hermes sessions prune --older-than 60 --min-tokens 10000 # 多条件过滤
```

### 2. 自动清理（默认关闭）
```yaml
# config.yaml
sessions:
  auto_prune: false          # 开启自动清理
  retention_days: 90         # 保留天数
  vacuum_after_prune: true   # 清理后 VACUUM
  min_interval_hours: 24     # 最小间隔
```
```bash
hermes config set sessions.auto_prune true
hermes config set sessions.retention_days 30
```

### 注意
`hermes insights --days N` 只是一个**查询窗口**参数，不删除数据。真正的数据保留控制靠 `prune` 机制。

---

## 四、数据库表结构

**数据库文件**：`~/.hermes/state.db`（SQLite，WAL 模式）

### 完整表清单

| 表名 | 用途 | 备注 |
|------|------|------|
| `schema_version` | 版本号（当前 v21） | 单行 |
| `sessions` | 会话元数据（核心） | ~40 列 |
| `messages` | 消息历史（核心） | ~20 列 |
| `session_model_usage` | 按模型用量明细 | 复合主键 |
| `state_meta` | KV 元数据存储 | key/value |
| `gateway_routing` | 网关路由索引 | 会话↔平台映射 |
| `compression_locks` | 上下文压缩锁 | 防死锁 |
| `async_delegations` | 异步子任务追踪 | 生命周期管理 |
| `telegram_dm_topic_mode` | Telegram 话题模式 | |
| `telegram_dm_topic_bindings` | Telegram 话题绑定 | |
| `messages_fts` | FTS5 全文索引（英文） | 虚拟表，触发器同步 |
| `messages_fts_trigram` | FTS5 trigram 索引（CJK） | 虚拟表，触发器同步 |

### 核心关系图

```
state_meta (KV)                     gateway_routing (路由)
     │                                     │
     │                                     │
     ▼                                     ▼
sessions ────1:N──── messages ────FTS5──── messages_fts
     │                                      │
     │                              messages_fts_trigram
     │
     └───1:N──── session_model_usage
     │
     └───0:1──── compression_locks
     │
     └───0:N──── async_delegations
```

### sessions 表关键字段

| 字段 | 类型 | 说明 |
|------|------|------|
| `id` | TEXT PK | 会话 ID |
| `source` | TEXT NOT NULL | 来源平台（cli/telegram/discord/cron） |
| `model` | TEXT | 使用的模型 |
| `parent_session_id` | TEXT FK | 父会话（lineage 链） |
| `started_at` / `ended_at` | REAL | 时间戳 |
| `message_count` / `tool_call_count` | INTEGER | 计数 |
| `input_tokens` / `output_tokens` / `cache_*_tokens` / `reasoning_tokens` | INTEGER | Token 用量 |
| `estimated_cost_usd` / `actual_cost_usd` | REAL | 费用 |
| `title` | TEXT UNIQUE | 会话标题（可空，非空唯一） |
| `cwd` / `git_branch` / `git_repo_root` | TEXT | 工作环境 |
| `archived` | INTEGER DEFAULT 0 | 归档标记 |
| `profile_name` | TEXT | 所属 profile |

### messages 表关键字段

| 字段 | 类型 | 说明 |
|------|------|------|
| `id` | INTEGER PK AUTOINCREMENT | 自增 ID |
| `session_id` | TEXT FK | 所属会话 |
| `role` | TEXT NOT NULL | user/assistant/tool |
| `content` | TEXT | 消息内容 |
| `tool_calls` | TEXT | 工具调用 JSON 数组 |
| `tool_name` | TEXT | 工具名 |
| `timestamp` | REAL | 时间戳 |
| `token_count` | INTEGER | 该消息的 token 数 |
| `active` | INTEGER DEFAULT 1 | 是否活跃（压缩后标记 0） |
| `compacted` | INTEGER DEFAULT 0 | 是否已被压缩 |

### session_model_usage 表

复合主键：`(session_id, model, billing_provider, billing_base_url, billing_mode, task)`

`task` 字段标注任务类型：`main` / `vision` / `compression` / `titles` 等。这是 insights 按模型拆分的统计来源。

### 数据库文件位置

- 主数据库：`~/.hermes/state.db`
- WAL 日志：`~/.hermes/state.db-wal`
- 共享内存：`~/.hermes/state.db-shm`
- 若设 `HERMES_HOME`，则为 `$HERMES_HOME/state.db`