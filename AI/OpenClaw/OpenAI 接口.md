
OpenClaw **支持 OpenAI 协议的 HTTP 接口**。根据官方文档 `gateway/openai-http-api.md`，Gateway 可以暴露以下 OpenAI 兼容端点：

## 支持的端点

| 端点 | 说明 |
|------|------|
| `POST /v1/chat/completions` | Chat Completions（支持流式 SSE） |
| `GET /v1/models` | 模型列表 |
| `GET /v1/models/{id}` | 单个模型详情 |
| `POST /v1/embeddings` | 向量嵌入 |
| `POST /v1/responses` | Responses API |

所有端点复用 Gateway 的 HTTP 端口（默认 18789），与 WebSocket 多路复用。

## 关键特性

- **默认关闭**，需要在配置中显式启用
- 支持 `stream: true` 的 SSE 流式输出
- 支持 Function Calling（`tools` / `tool_choice`）
- 支持 `temperature`、`top_p`、`max_tokens` 等标准参数
- 支持 `x-openclaw-model` 头覆盖后端模型

## 启用方式

```json5
{
  gateway: {
    http: {
      endpoints: {
        chatCompletions: { enabled: true }
      }
    }
  }
}
```

## 认证

使用 Gateway 的认证体系，通常是 `Authorization: Bearer <token>`。

## 模型路由

OpenAI 的 `model` 字段在 OpenClaw 中映射为 **agent 目标**：
- `openclaw` 或 `openclaw/default` → 默认 agent
- `openclaw/<agentId>` → 指定 agent

## 适用场景

适合与 Open WebUI、LobeChat、LibreChat 等前端工具集成，或需要标准 OpenAI API 格式的自定义工具。

---

⚠️ **安全提醒**：该端点属于 operator 级别访问，拥有完整权限，建议仅暴露在 loopback/tailnet/私有网络，不要直接暴露到公网。