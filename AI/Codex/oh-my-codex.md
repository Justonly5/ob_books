## 简介

  oh-my-codex（OMX）是一个构建在 OpenAI Codex CLI 之上的工作流编排层（Workflow Orchestration Layer）。
  
  **Codex** 负责实际的 Agent 执行；**OMX** 负责提供更好的任务路由、工作流和运行时管理。

## 解决什么

|痛点|OMX 的解法|
|:--|:--|
|每次使用 Codex 都要手动写大量 prompt|提供标准化的技能库（40个技能），通过 `$skill-name` 一键触发|
|复杂任务没有标准化流程|提供从需求澄清 → 计划 → 执行 → 验证的完整工作流|
|多智能体并行协作复杂|`$team`<br><br> 模式一键启动 tmux 多 Agent 并行工作流|
|会话间上下文丢失|`.omx/`<br><br> 目录持久化计划、日志、内存、状态|
|Codex 角色不明确，执行质量不稳定|提供 30 个专业角色 prompt（architect、debugger、executor 等）|
|大任务执行到一半就停止|`$ralph`<br><br> 持久化完成循环，强制推进到真正完成|
## 安装

### 全局安装
```BASH
# 安装 OMX  
npm install -g oh-my-codex  
  
# 初始化 OMX（安装 prompts、skills、配置 Codex CLI）  
omx setup  
  
# 验证安装是否成功  
omx doctor
```

### 验证

```BASH
oh-my-codex doctor  
==================  
  
  [OK] Codex CLI: installed  
  [OK] Node.js: v20+  
  [OK] Codex home: ~/.codex  
  [OK] Config: config.toml has OMX entries  
  [OK] Prompts: 30 agent prompts installed  
  [OK] Skills: 40 skills installed  
  [OK] AGENTS.md: found in project root  
  [OK] State dir: .omx/state  
  [OK] MCP Servers: 4 servers configured (OMX present)  
  
Results: 9 passed, 0 warnings, 0 failed
```
## 使用



## 使用场景

| 场景                      | 推荐做法                                      |
| :---------------------- | :---------------------------------------- |
| 功能需求不清晰，想先澄清边界          | `$deep-interview "..."`                   |
| 需求清晰，想在实现前制定计划和权衡       | `$ralplan "..."`                          |
| 大型功能开发，多个子任务可并行         | `$team N:executor "..."`                  |
| 一个任务需要持续推进直到完成/验证通过     | `$ralph "..."`                            |
| 从零构建一个新功能/应用            | `$autopilot "build me ..."`               |
| 代码审查                    | `$code-review`                            |
| 安全审计                    | `$security-review`                        |
| 修复 TypeScript 类型错误或构建失败 | `$build-fix`                              |
| 自动化研究/优化任务              | `omx autoresearch missions/<mission-dir>` |
| 只需查看仓库结构，不需要编辑          | `omx explore --prompt "..."`              |

## 案例

我看了下 你现在整理的那个 deploy 的文件 有几个问题确认下：
1. service/cm、ingress yaml 文件里是没有配置 namespace