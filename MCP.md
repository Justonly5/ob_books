# MCP 概念

## MCP 是什么
	MCP (Model Context Protocol) 2024年11月底，由 Anthropic 推出的一种开放协议，它为应用程序向 LLM 提供上下文的方式进行了标准化。MCP 作为一种标准化协议，极大地简化了大语言模型与外部世界的交互方式，使开发者能够以统一的方式为 AI 应用添加各种能力。


![[Pasted image 20251123213358.png]]

![[Pasted image 20251123213543.png]]

## 通用架构
![[Pasted image 20251123213901.png]]
MCP 核心采用客户端-服务器架构，主机应用可以连接多个服务器：

- **MCP Hosts**: 发起请求的 LLM 应用程序，如 Claude Desktop、IDE 或 AI 工具。
- **MCP Clients**: 在 host 内部，维护与 MCP Server 保持一对一连接的协议客户端。
- **MCP Servers**: 轻量级程序，通过标准的 Model Context Protocol 为 MCP Client 提供上下文、工具和 prompt 信息。
- **本地资源**: MCP 服务器可安全访问的计算机文件、数据库和服务。
- **远程资源**: MCP 服务器可连接远程外部系统（如通过 APIs）。

![[Pasted image 20251123214337.png]]