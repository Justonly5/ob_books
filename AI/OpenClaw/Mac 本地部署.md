> https://docs.openclaw.ai/zh-CN/install

## 前置软件
Node.js 22.0.0+(会自动安装)
## 安装步骤

```BASH
curl -fsSL https://openclaw.ai/install.sh | bash
```

安装过程会自动：
- 检测系统环境
- 安装 Node.js（如果未安装）
- 下载 OpenClaw
- 配置环境变量
## 验证安装

```BASH
openclaw --version
```



配置模型
```BASH
openclaw configure --section model
```

重启网关
```bash
openclaw gateway restart
```

## 百炼模型

``````json
{
  "meta": {
    "lastTouchedVersion": "2026.2.1",
    "lastTouchedAt": "2026-02-03T08:20:00.000Z"
  },
  "models": {
    "mode": "merge",
    "providers": {
      "bailian": {
        "baseUrl": "https://dashscope.aliyuncs.com/compatible-mode/v1",
        "apiKey": "DASHSCOPE_API_KEY",
        "api": "openai-completions",
        "models": [
          {
            "id": "qwen3.5-plus",
            "name": "qwen3.5-plus",
            "reasoning": false,
            "input": ["text", "image"],
            "contextWindow": 1000000,
            "maxTokens": 65536
          },
          {
            "id": "qwen3-coder-next",
            "name": "qwen3-coder-next",
            "reasoning": false,
            "input": ["text"],
            "contextWindow": 262144,
            "maxTokens": 65536
          }
        ]
      }
    }
  },
  "agents": {
    "defaults": {
      "model": {
        "primary": "bailian/qwen3.5-plus"
      },
      "models": {
        "bailian/qwen3.5-plus": {},
        "bailian/qwen3-coder-next": {}
      }
    }
  },
  "gateway": {
    "mode": "local",
    "auth": {
      "mode": "token",
      "token": "test123"
    }
  }
}
```
```