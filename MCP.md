# MCP 概念

## MCP 是什么
	MCP (Model Context Protocol) 2024年11月底，由 Anthropic 推出的一种开放协议，它为应用程序向 LLM 提供上下文的方式进行了标准化。MCP 作为一种标准化协议，极大地简化了大语言模型与外部世界的交互方式，使开发者能够以统一的方式为 AI 应用添加各种能力。


![[Pasted image 20251123213358.png]]

![[Pasted image 20251123213543.png]]

## MCP 通用架构
MCP 遵循一个 client-server 架构，其中：
- **Hosts** 是 LLM 应用（如 Claude Desktop 或 IDEs），它们发起连接。
- **Clients** 在 host 应用中，与 servers 保持 1:1 的连接。
- **Servers** 为 clients 提供上下文、tools 和 prompts。
![[Pasted image 20251123215440.png]]

![[Pasted image 20251123213901.png]]


- **本地资源**: MCP 服务器可安全访问的计算机文件、数据库和服务。
- **远程资源**: MCP 服务器可连接远程外部系统（如通过 APIs）。

![[Pasted image 20251123214337.png]]

## MCP 数据格式
在 MCP 中所有传输都使用 [JSON-RPC](https://www.jsonrpc.org/) 2.0 进行消息交换。

## MCP 传输层

传输层处理 clients 和 servers 之间的实际通信。MCP 支持多种传输机制：

### **Stdio 传输**
    - 使用标准输入/输出进行通信
    - 适用于本地进程
    MCP Client 通过启动一个子进程（MCP Server）并通过 stdin、stdout 的本地通信方式来交换 JSON 消息来实现通信。
    详细描述如下:
    1. 启动子进程(MCP Server)
    MCP Client 以子进程形式启动 MCP Server，通过命令行指定 Server 的可执行文件及其参数
    2. 消息交换
	MCP Client 通过 stdin 向 MCP Server 写入 JSON-RPC 消息
	MCP Server 处理请求后，通过 stdout 返回 JSON-RPC 消息，也可通过 stderr 输出日志
	3. 生命周期管理
	MCP Client 控制子进程 (MCP Server)的启动和关闭。通信结束后，MCP Client 关闭 stdin，终止 MCP Server
###  SSE 传输
	 20250326版本前。SSE(服务器发送事件)是基于 HTTP 协议的一种单向通信技术。
    - 使用服务器发送事件进行服务器到客户端的消息传递
    - 使用 HTTP POST 进行客户端到服务器的消息传递
### 流式传输