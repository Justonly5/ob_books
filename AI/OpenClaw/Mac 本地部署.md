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