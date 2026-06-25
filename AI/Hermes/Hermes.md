
# Hermes Agent

> Nous Research 开源的 AI Agent 框架，支持终端、IDE、多平台消息网关。

## 笔记索引

- [[快速入门]] — 安装 & K8s 基础部署
- [[对接 Hermes]] — 应用程序对接 Hermes 的 6 种方式 & 协议总览
- [[启动模式]] — Hermes 9 种启动/运行模式全景图
- [[功能模块与监控]] — 功能模块总览 & 4 种监控方案

## 核心概念

- **Skills**：可复用的过程性知识，Agent 从经验中学习并持久化
- **Memory**：跨会话持久记忆，记住用户偏好和环境信息
- **Gateway**：多平台消息中枢，支持 15+ IM 平台
- **MCP**：Model Context Protocol，双向工具能力扩展
- **Profiles**：多套独立配置、会话、技能、记忆

## 关键路径

| 路径 | 说明 |
|------|------|
| `~/.hermes/config.yaml` | 主配置 |
| `~/.hermes/.env` | API Keys & 环境变量 |
| `~/.hermes/skills/` | 已安装技能 |
| `~/.hermes/state.db` | 会话存储 (SQLite + FTS5) |
| `~/.hermes/logs/` | 日志 |
