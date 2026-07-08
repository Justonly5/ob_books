## 通过 exec 命令获取 API Key 的配置方法

OpenClaw 使用 **SecretRef** 机制，让 API Key 不直接写在配置文件里，而是通过 `exec` 命令动态获取。配置分两步：

### 1. 定义 exec provider

在 `secrets.providers` 下定义一个 exec 类型的 provider：

```json5
{
  secrets: {
    providers: {
      // 你的 exec provider 名称
      my_key_fetcher: {
        source: "exec",
        command: "/usr/local/bin/my-fetch-key",  // 绝对路径，不能用 shell
        args: ["--project", "openclaw"],
        passEnv: ["PATH", "HOME"],    // 需要传递的环境变量
        jsonOnly: false,              // false=命令 stdout 直接作为值
      },
    },
  },
}
```

### 2. 在 apiKey 上引用

```json5
{
  models: {
    providers: {
      openai: {
        baseUrl: "https://api.openai.com/v1",
        models: [{ id: "gpt-5", name: "gpt-5" }],
        apiKey: {
          source: "exec",
          provider: "my_key_fetcher",
          id: "value",        // jsonOnly: false 时固定用 "value"
        },
      },
    },
  },
}
```

### 两种模式

**`jsonOnly: false`（简单模式）**：命令 stdout 直接当 key 用，`id` 固定为 `"value"`。

**`jsonOnly: true`（批量模式）**：OpenClaw 通过 stdin 传入 JSON 请求，命令返回 JSON 响应：

```jsonc
// stdin 输入
{ "protocolVersion": 1, "provider": "my_fetcher", "ids": ["providers/openai/apiKey"] }

// stdout 返回
{ "protocolVersion": 1, "values": { "providers/openai/apiKey": "sk-xxx..." } }
```

此时 `id` 可以是任意符合规范的字符串路径。

### 实用示例

**1Password CLI：**
```json5
secrets: {
  providers: {
    onepassword: {
      source: "exec",
      command: "/opt/homebrew/bin/op",
      allowSymlinkCommand: true,   // Homebrew 的 symlink 需要
      trustedDirs: ["/opt/homebrew"],
      args: ["read", "op://Personal/OpenAI API Key/password"],
      passEnv: ["HOME"],
      jsonOnly: false,
    },
  },
},
models: {
  providers: {
    openai: {
      apiKey: { source: "exec", provider: "onepassword", id: "value" },
    },
  },
},
```

**自定义 Shell 脚本获取 key：**
```json5
secrets: {
  providers: {
    fetch_key: {
      source: "exec",
      command: "/usr/local/bin/fetch-api-key.sh",
      passEnv: ["PATH", "SECRET_TOKEN"],
      jsonOnly: false,
    },
  },
},
models: {
  providers: {
    volcengine: {
      apiKey: { source: "exec", provider: "fetch_key", id: "value" },
    },
  },
},
```

### 配置后验证

```bash
openclaw secrets audit --check           # 静态审计
openclaw secrets audit --allow-exec      # 含 exec 执行验证
```

### 注意事项

- `command` 必须是**绝对路径**，不加 shell 解释
- Homebrew 安装的二进制是 symlink，需要 `allowSymlinkCommand: true` + `trustedDirs`
- 用 `openclaw secrets configure` 可以交互式配置，更省心

