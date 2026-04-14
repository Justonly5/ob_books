

OpenAI 出品的由 Rust 编写的 CLI 工具，和 Claude Code 一样。

- • **Codex Cloud**（云端托管版）
    
- • **Codex CLI**（本地终端工具）
    
- • **Codex App**（桌面应用）
    
- • **VSCode 扩展**（IDE 集成版）。
    

官网：https://developers.openai.com/codex

Github：https://github.com/openai/codex
```BASH
# 安装  
npm install -g @openai/codex 
# 交互模式运行  
codex                       
# 非交互  
codex exec
# 升级  
npm i -g @openai/codex@latest
```

## best-practice
>https://developers.openai.com/codex/learn/best-practices


### Context a nd prompts --> AGENTS.md

- **Goal:** What are you trying to change or build?
- **Context:** Which files, folders, docs, examples, or errors matter for this task? You can @ mention certain files as context.
- **Constraints:** What standards, architecture, safety requirements, or conventions should Codex follow?
- **Done when:** What should be true before the task is complete, such as tests passing, behavior changing, or a bug no longer reproducing?

### Plan first for difficult tasks --> PLANS.md

/plan PLANS.md

