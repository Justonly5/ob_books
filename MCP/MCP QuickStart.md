https://docs.spring.io/spring-ai/reference/api/mcp/mcp-server-boot-starter-docs.html
https://modelcontextprotocol.io/docs/develop/build-server#java


# 服务端开发
## Python

### 环境准备
- 安装Python 3.10或更高版本。
- 您必须使用Python MCP SDK 1.2.0或更高版本。

```BASH
curl -LsSf https://astral.sh/uv/install.sh | sh
```

### 创建项目
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
### 构建服务器
#### 导入依赖包
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

#### 实现工具执行
```python
@mcp.tool()
async def get_alerts(state: str) -> str:
    """获取指定州的天气警报（使用两字母州代码如CA/NY）"""
    url = f"{NWS_API_BASE}/alerts/active/area/{state}"
    data = await make_nws_request(url)

    if not data or "features" not in data:
        return "无法获取警报或未找到警报。"

    if not data["features"]:
        return "该州没有活动警报。"

    alerts = [format_alert(feature) for feature in data["features"]]
    return "\n---\n".join(alerts)
```

#### 运行服务器
```PYTHON
if __name__ == "__main__":
    # 初始化并运行服务器
    mcp.run(transport='stdio')
```

```bash
# 本地测试 debug 运行
uv run mcp dev weather.py
```
![[Pasted image 20251128145449.png]]


## Java


> [!NOTE]
> [spring-ai-weather](https://github.com/spring-projects/spring-ai-examples/tree/main/model-context-protocol/weather)

### 环境准备

- Java 17 or higher installed.
- [Spring Boot 3.3.x](https://docs.spring.io/spring-boot/installing.html) or higher

### 创建项目
通过 [Spring Initializer](https://start.spring.io/)
```XML
<dependencies>
      <dependency>
          <groupId>org.springframework.ai</groupId>
          <artifactId>spring-ai-starter-mcp-server</artifactId>
      </dependency>

      <dependency>
          <groupId>org.springframework</groupId>
          <artifactId>spring-web</artifactId>
      </dependency>
</dependencies>
```


```JSON
{
  "mcpServers": {
    "spring-ai-mcp-weather": {
      "command": "java",
      "args": [
        "-Dspring.ai.mcp.server.stdio=true",
        "-jar",
        "/ABSOLUTE/PATH/TO/PARENT/FOLDER/mcp-weather-stdio-server-0.0.1-SNAPSHOT.jar"
      ]
    }
  }
}
```