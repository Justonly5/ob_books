
## DeepSeek 官方 API 配置流程

完成 OpenClaw 基础部署后，配置 DeepSeek API 作为智能交互的模型提供商，支持 DeepSeek Chat V3、DeepSeek Reasoner R1 等模型。

### 3.1 获取 DeepSeek API Key

1. 访问 [DeepSeek 官方平台](https://cloud.tencent.com/developer/tools/blog-entry?target=https%3A%2F%2Fplatform.deepseek.com%2F&objectId=2638806&objectType=1&contentType=markdown)，完成注册与账号验证；  
    
2. 进入平台「API 密钥」模块，创建并获取 API Key，格式为 `sk-xxxxxxxxxxxxxxxxxxxxx`；  
    
3. 妥善保存 API Key，避免泄露（后续配置需直接使用）。
    

### 3.2 配置 DeepSeek 模型提供商

执行以下命令，将`你的API_KEY`替换为实际获取的 DeepSeek API Key，完成提供商基础配置：


```bash
openclaw config set models.providers.deepseek '{
  "baseUrl": "https://api.deepseek.com/v1",
  "apiKey": "你的API_KEY",
  "api": "openai-completions",
  "models": [
    {
      "id": "deepseek-chat",
      "name": "DeepSeek Chat (V3)"
    },
    {
      "id": "deepseek-reasoner",
      "name": "DeepSeek Reasoner (R1)"
    }
  ]
}'
```

### 3.3 设置默认交互模型

指定 `deepseek-chat` 为 OpenClaw 默认的智能交互模型，后续可根据需求切换：

```bash
openclaw config set agents.defaults.model.primary "deepseek/deepseek-chat"
```

### 3.4 创建模型别名（可选）

为模型创建简短别名，简化后续模型切换命令，提升使用效率：

```bash
openclaw models aliases add deepseek-v3 "deepseek/deepseek-chat"
openclaw models aliases add deepseek-r1 "deepseek/deepseek-reasoner"
```

### 3.5 重启 Gateway 服务使配置生效


```bash
openclaw gateway restart
```

等待 3-5 秒让服务完全重启，确保配置加载成功。

### 3.6 DeepSeek 配置测试与验证

#### 3.6.1 命令行快速测试

发送测试消息，若收到 DeepSeek 中文回复，即表示 API 配置成功：


```bash
openclaw agent --session-id test --message "你好，请介绍一下你自己"
```

#### 3.6.2 查看模型配置状态

确认默认模型与已配置模型列表是否正确：

```bash
openclaw models status
```

**预期结果**：显示 `Default: deepseekdeepseek-chat`，且配置模型列表包含 deepseek 相关模型。

#### 3.6.3 打开 Web 控制面板

通过可视化面板管理 OpenClaw，命令执行后将自动在浏览器打开面板：

```bash
openclaw dashboard
```

面板访问 URL 格式：`http://127.0.0.1:18789/#token=你的gateway_token`