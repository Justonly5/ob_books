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
