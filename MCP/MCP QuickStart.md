# 服务端开发

## 环境准备
- 安装Python 3.10或更高版本。
- 您必须使用Python MCP SDK 1.2.0或更高版本。

```BASH
curl -LsSf https://astral.sh/uv/install.sh | sh
```

## 创建项目
```BASH
# 为我们的项目创建一个新目录
uv init weather
cd weather

# 创建虚拟环境并激活它
uv venv
source .venv/bin/activate

# 安装依赖
uv add "mcp[cli]" httpx

# 创建我们的服务器文件
touch weather.py
```
## 构建服务器
### 导入依赖包
`weather.py`导入以下依赖：
```PYTHON
from typing import Any
import httpx
from mcp.server.fastmcp import FastMCP

# 初始化FastMCP服务器
mcp = FastMCP("weather")

# 常量
NWS_API_BASE = "https://api.weather.gov"
USER_AGENT = "weather-app/1.0"
```

