# 什么是ACP
ACP（Agent Client Protocol） 是一种基于 stdio + JSON-RPC 的开放通信协议，允许外部宿主应用（主要是代码编辑器）以子进程方式启动并驱动 AI Agent。

> 编辑器（客户端） ← ACP JSON-RPC over stdio → Hermes Agent（服务端）

