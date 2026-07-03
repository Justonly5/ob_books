# Hermes 与 OpenClaw WebSocket 协议对比

> 文档目的：梳理 OpenClaw sidecar（kimi-claw connector）与 Hermes Dashboard 的 WebSocket 协议差异，为 sidecar 适配 Hermes 提供参考。

---

## 一、架构对比

```
┌─────────────────────────────────────────────────────────────────┐
│                     OpenClaw 架构                                │
│                                                                 │
│  sidecar 应用 ──WS──▶ kimi-claw connector ──▶ OpenClaw Gateway   │
│  (ACP 协议)          (port 18789)             (内部调度)          │
└─────────────────────────────────────────────────────────────────┘

┌─────────────────────────────────────────────────────────────────┐
│                     Hermes 架构                                  │
│                                                                 │
│  sidecar 应用 ──WS──▶ Hermes Dashboard ──▶ tui_gateway.dispatch  │
│  (JSON-RPC 2.0)      (port 9119)           ──▶ AIAgent          │
└─────────────────────────────────────────────────────────────────┘
```

---

## 二、传输层对比

| 维度 | OpenClaw (kimi-claw) | Hermes |
|------|---------------------|--------|
| **传输协议** | WebSocket | WebSocket |
| **消息格式** | JSON-RPC 2.0 风格（换行分隔） | JSON-RPC 2.0（换行分隔） |
| **认证方式** | HTTP Header `X-Kimi...` token | Query param `?token=` 或 `?internal=` |
| **心跳机制** | 文本 `ping`/`pong` + WebSocket ping/pong | 无应用层心跳（依赖 WS 层） |
| **连接端点** | 由 kimi-claw 配置决定 | `ws://127.0.0.1:9119/api/ws?token=xxx` |
| **启动方式** | OpenClaw Gateway 自动拉起 connector | `hermes web` 启动 Dashboard |

---

## 三、RPC 方法对比

### 3.1 会话管理

| 功能 | OpenClaw (ACP) | Hermes |
|------|---------------|--------|
| 创建会话 | `session/new` | `session.create` |
| 列出会话 | `session/list` | `session.list` |
| 加载/恢复会话 | `session/load` | `session.resume` |
| 删除会话 | ❌ 无 | `session.delete` |
| 关闭会话 | ❌ 无 | `session.close` |
| 分支会话 | ❌ 无 | `session.branch` |
| 设置工作目录 | `session/new` 时传 `cwd` | `session.cwd.set` |
| 获取会话状态 | ❌ 无 | `session.status` |
| 获取会话历史 | ❌ 无 | `session.history` |
| 撤销上轮对话 | ❌ 无 | `session.undo` |
| 压缩上下文 | ❌ 无 | `session.compress` |
| 设置标题 | ❌ 无 | `session.title` |
| Token 用量 | ❌ 无 | `session.usage` |
| 中断执行 | `session/cancel` | `session.interrupt` |
| 中段注入消息 | ❌ 无 | `session.steer` |
| 激活会话 | ❌ 无 | `session.activate` |

### 3.2 消息发送

| 功能 | OpenClaw (ACP) | Hermes |
|------|---------------|--------|
| 发送提示词 | `session/prompt` | `prompt.submit` |
| 后台发送 | ❌ 无 | `prompt.background` |
| 协议初始化 | `initialize` | ❌ 无（连接即就绪，推送 `gateway.ready`） |
| 设置模型 | `session/set_model` | ❌ 无（通过 `/model` 斜杠命令或 config） |

### 3.3 交互响应

| 功能 | OpenClaw (ACP) | Hermes |
|------|---------------|--------|
| 审批响应 | ❌ 无 | `approval.respond` |
| 澄清响应 | ❌ 无 | `clarify.respond` |
| sudo 密码响应 | ❌ 无 | `sudo.respond` |
| 密钥响应 | ❌ 无 | `secret.respond` |
| 终端输入响应 | ❌ 无 | `terminal.read.respond` |

### 3.4 附件

| 功能 | OpenClaw (ACP) | Hermes |
|------|---------------|--------|
| 粘贴图片 | `session/prompt` 内嵌 | `image.attach` / `image.attach_bytes` |
| 粘贴剪贴板 | ❌ 无 | `clipboard.paste` |
| 附加文件 | `session/prompt` 内嵌 | `file.attach` |
| 附加 PDF | ❌ 无 | `pdf.attach` |
| 移除图片 | ❌ 无 | `image.detach` |

### 3.5 配置与工具

| 功能 | OpenClaw (ACP) | Hermes |
|------|---------------|--------|
| 读取配置 | ❌ 无 | `config.get` |
| 修改配置 | ❌ 无 | `config.set` |
| 浏览器管理 | ❌ 无 | `browser.manage` |
| 插件列表 | ❌ 无 | `plugins.list` |
| 插件管理 | ❌ 无 | `plugins.manage` |
| 技能管理 | ❌ 无 | `skills.manage` |
| 重载技能 | ❌ 无 | `skills.reload` |

### 3.6 执行与进程

| 功能 | OpenClaw (ACP) | Hermes |
|------|---------------|--------|
| 执行 Shell | ❌ 无（agent 内部） | `shell.exec` |
| 执行 CLI 命令 | ❌ 无 | `cli.exec` |
| 执行斜杠命令 | ❌ 无 | `slash.exec` |
| 停止进程 | ❌ 无 | `process.stop` |
| 调整终端大小 | ❌ 无 | `terminal.resize` |

### 3.7 委托与子代理

| 功能 | OpenClaw (ACP) | Hermes |
|------|---------------|--------|
| 委托状态 | ❌ 无 | `delegation.status` |
| 暂停委托 | ❌ 无 | `delegation.pause` |
| 中断子代理 | ❌ 无 | `subagent.interrupt` |

### 3.8 其他

| 功能 | OpenClaw (ACP) | Hermes |
|------|---------------|--------|
| 交接请求 | ❌ 无 | `handoff.request` |
| 交接状态 | ❌ 无 | `handoff.state` |
| 交接失败 | ❌ 无 | `handoff.fail` |
| 保存/加载生成树 | ❌ 无 | `spawn_tree.save/list/load` |
| 安装状态 | ❌ 无 | `setup.status` |
| 运行时检查 | ❌ 无 | `setup.runtime_check` |
| 积分查看 | ❌ 无 | `credits.view` |
| 预览重启 | ❌ 无 | `preview.restart` |
| 拖放检测 | ❌ 无 | `input.detect_drop` |

---

## 四、服务端推送事件对比

### 4.1 OpenClaw (ACP) — session/update 通知

OpenClaw 通过 `session/update` 通知推送流式内容：

```json
{
  "jsonrpc": "2.0",
  "method": "session/update",
  "params": {
    "sessionId": "main",
    "sessionUpdate": "agent_message_chunk",
    "content": { "type": "text", "text": "你好" }
  }
}
```

**sessionUpdate 类型：**

| 类型 | 说明 |
|------|------|
| `user_message_chunk` | 用户消息内容 |
| `agent_message_chunk` | Agent 回复文本（流式） |
| `agent_thought_chunk` | 推理/思考过程 |
| `tool_call` | 工具调用开始 |
| `tool_call_update` | 工具调用结果 |

### 4.2 Hermes — event 通知

Hermes 通过 `event` 通知推送：

```json
{
  "jsonrpc": "2.0",
  "method": "event",
  "params": {
    "type": "token",
    "text": "你"
  }
}
```

**event type 类型：**

| 类型 | 说明 |
|------|------|
| `token` | 流式文本 token |
| `tool.start` | 工具调用开始（含工具名、参数） |
| `tool.end` | 工具调用结束（含返回值） |
| `turn.end` | 本轮对话结束 |
| `session.info` | 会话信息（模型、状态等） |
| `error` | 错误信息 |
| `approval.request` | 审批请求 |
| `clarify.request` | 澄清请求 |
| `gateway.ready` | 连接就绪（首次推送） |
| `background.notification` | 后台进程通知 |

---

## 五、典型交互流程对比

### 5.1 OpenClaw (ACP) 流程

```
Client                          Server
  │                                │
  │── initialize ────────────────▶│  协议握手
  │◀─────────────── result ───────│  返回 capabilities
  │                                │
  │── session/new ───────────────▶│  创建会话
  │◀─────────────── result ───────│  返回 sessionId
  │                                │
  │── session/prompt ────────────▶│  发送提示词
  │◀── session/update ────────────│  user_message_chunk
  │◀── session/update ────────────│  agent_message_chunk (流式)
  │◀── session/update ────────────│  agent_thought_chunk (思考)
  │◀── session/update ────────────│  tool_call
  │◀── session/update ────────────│  tool_call_update
  │◀── session/update ────────────│  agent_message_chunk (继续)
  │◀─────────────── result ───────│  stopReason: "end_turn"
```

### 5.2 Hermes 流程

```
Client                          Server
  │                                │
  │── session.create ────────────▶│  创建会话
  │◀─────────────── result ───────│  返回 session_id
  │◀── event(gateway.ready) ──────│  就绪通知
  │                                │
  │── prompt.submit ─────────────▶│  发送提示词
  │◀─────────────── result ───────│  {"status":"streaming"}
  │◀── event(token) ──────────────│  流式 token
  │◀── event(token) ──────────────│  流式 token
  │◀── event(tool.start) ─────────│  工具调用开始
  │◀── event(tool.end) ───────────│  工具调用结束
  │◀── event(token) ──────────────│  流式 token
  │◀── event(turn.end) ───────────│  本轮结束
```

---

## 六、关键差异总结

| 维度 | OpenClaw (ACP) | Hermes |
|------|---------------|--------|
| **协议风格** | ACP（Agent Communication Protocol），方法名用 `/` 分隔 | 自定义 JSON-RPC，方法名用 `.` 分隔 |
| **初始化** | 需要显式 `initialize` 握手 | 连接即就绪，推送 `gateway.ready` |
| **会话模型** | 单会话模型（`sessionId: "main"`） | 多会话模型（每个连接可管理多个 session） |
| **流式推送** | `session/update` 通知，类型用 `sessionUpdate` 字段区分 | `event` 通知，类型用 `params.type` 字段区分 |
| **工具调用** | `tool_call` + `tool_call_update`（含 toolCallId、status） | `tool.start` + `tool.end`（独立事件） |
| **思考过程** | `agent_thought_chunk` | 包含在 `event` 流中 |
| **审批/澄清** | ❌ 不支持 | ✅ 完整支持（`approval.respond` 等） |
| **附件上传** | 内嵌在 `session/prompt` 中 | 独立方法（`image.attach` 等） |
| **会话管理** | 基础（new/list/load/cancel） | 丰富（20+ 方法） |
| **配置管理** | ❌ 无 | ✅ `config.get/set` |
| **功能丰富度** | 精简，聚焦核心对话 | 完整，覆盖 TUI 全部交互 |

---

## 七、适配建议

### 7.1 最小适配路径

如果 sidecar 只需要**发送消息 + 接收流式回复**，只需适配以下方法：

| OpenClaw (ACP) | → | Hermes |
|---------------|----|--------|
| `initialize` | → | 忽略（连接即就绪） |
| `session/new` | → | `session.create` |
| `session/prompt` | → | `prompt.submit` |
| `session/cancel` | → | `session.interrupt` |

### 7.2 事件映射

| OpenClaw `sessionUpdate` | → | Hermes `event.type` |
|--------------------------|----|---------------------|
| `agent_message_chunk` | → | `token` |
| `agent_thought_chunk` | → | `token`（需根据上下文区分） |
| `tool_call` | → | `tool.start` |
| `tool_call_update` | → | `tool.end` |

### 7.3 请求/响应格式差异

**OpenClaw `session/prompt`：**
```json
{
  "jsonrpc": "2.0",
  "method": "session/prompt",
  "id": 1,
  "params": {
    "sessionId": "main",
    "prompt": [{ "type": "text", "text": "你好" }]
  }
}
```

**Hermes `prompt.submit`：**
```json
{
  "jsonrpc": "2.0",
  "method": "prompt.submit",
  "id": 1,
  "params": {
    "session_id": "abc123",
    "text": "你好"
  }
}
```

### 7.4 注意事项

1. **认证方式不同**：OpenClaw 用 HTTP Header token，Hermes 用 query param `?token=`。需要在 sidecar 连接时适配。
2. **方法名风格不同**：OpenClaw 用 `/`（`session/prompt`），Hermes 用 `.`（`prompt.submit`）。建议在 sidecar 中加一层映射。
3. **参数命名不同**：OpenClaw 用 camelCase（`sessionId`），Hermes 用 snake_case（`session_id`）。
4. **Hermes 功能更丰富**：审批、澄清、附件上传等 OpenClaw 不支持的功能，Hermes 有独立 RPC 方法。如果 sidecar 不需要这些，可以忽略。
5. **多会话支持**：Hermes 支持同时管理多个会话，sidecar 需要维护 `session_id` 映射。
