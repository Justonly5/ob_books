# OpenClaw vs Hermes Agent — 详细功能对比（2026年6月）

> 整理时间：2026-06-26

## 一、项目背景与哲学

| 维度               | OpenClaw                                   | Hermes Agent                         |
| ---------------- | ------------------------------------------ | ------------------------------------ |
| **开发方**          | 原 Moltbot（Peter Steinberger），后转入基金会管理      | Nous Research（Hermes 模型系列的团队）        |
| **首次公开**         | 2025年11月（v3.0），2026年2月（v4.0 架构重写）          | 2026年3月12日（v0.2.0）                   |
| **许可证**          | MIT                                        | MIT                                  |
| **核心理念**         | **多通道网关 + Agent OS** — 一个守护进程连接所有聊天平台，统一路由 | **自学习 Agent** — 从经验中创建技能，跨会话持续改进     |
| **商业模式**         | 开源 + ClawHub 技能市场 + openclawai.io SaaS     | 纯开源，无官方托管/SaaS，通过 Nous Portal 提供免费模型 |
| **GitHub Stars** | 成熟项目，社区活跃                                  | ~61,000+（发布后两个月快速增长）                 |

---

## 二、架构对比

### OpenClaw — Hub-and-Spoke 网关架构

```
WhatsApp / Telegram / Slack / Discord / Signal / iMessage / ...
         │
         ▼
┌───────────────────────────────┐
│     Gateway（常驻守护进程）      │
│     控制面 + 所有通道适配器      │
│     ws://127.0.0.1:18789      │
└──────────────┬────────────────┘
               │
    ├── Agent Runtime（RPC）
    ├── CLI（openclaw ...）
    ├── WebChat UI
    ├── macOS App
    └── iOS / Android Nodes
```

- **一个 Gateway 进程**加载所有通道适配器（WhatsApp via Baileys、Telegram via grammY 等）
- 所有通信通过 **类型化 WebSocket API**（3 种帧类型：req/res/event）
- 每个主机运行 **一个 Gateway**

### Hermes Agent — CLI 优先 + 可选网关

- **主入口是 CLI**，不是守护进程
- **Gateway 模式是可选扩展**，连接 Telegram、Discord、WhatsApp、Slack 等
- 容器后端是**一等公民**：Docker、Singularity、Modal、Daytona、Vercel Sandbox
- 支持 **6 种终端后端**：本地、Docker、SSH、Daytona、Singularity、Modal
- 可**作为 MCP Server** 运行，为其他 Agent 提供服务

### 架构差异总结

- OpenClaw = 网关中心化，所有通道内嵌到守护进程
- Hermes = CLI 中心化，网关是可选附加，容器/远程执行是原生能力

---

## 三、通道 / 消息平台支持

| 平台 | OpenClaw | Hermes Agent |
|------|----------|-------------|
| Telegram | ✅ 内置 | ✅ 网关模式 |
| WhatsApp | ✅ 内置（Baileys） | ✅ 网关模式 |
| Discord | ✅ 内置 | ✅ 网关模式 |
| Signal | ✅ 内置 | ✅ 网关模式 |
| Slack | ✅ 内置 | ✅ 网关模式 |
| iMessage | ✅ 内置 | ❌（通过 BlueBubbles 插件） |
| Google Chat | ✅ 内置 | ✅ 网关模式 |
| Matrix | ✅ 捆绑插件 | ✅ 网关模式 |
| Mattermost | ✅ 捆绑插件 | ✅ 网关模式 |
| Microsoft Teams | ✅ 捆绑插件 | ✅ 网关模式 |
| Feishu/Lark | ✅ 捆绑插件 | ✅ 网关模式 |
| Zalo | ✅ 捆绑插件 | ❌ |
| Nostr | ✅ 捆绑插件 | ❌ |
| Twitch | ✅ 捆绑插件 | ❌ |
| IRC | ✅ 内置 | ❌ |
| QQ Bot | ✅ 捆绑插件 | ✅ 网关模式 |
| WeChat | ✅ 第三方插件 | ❌ |
| WeCom | ❌ | ✅ 网关模式 |
| DingTalk | ❌ | ✅ 网关模式 |
| Email/SMS | ❌ | ✅ 网关模式 |
| Home Assistant | ❌ | ✅ 网关模式 |
| WebChat | ✅ 内置 | ❌（有 Web Dashboard） |
| **总平台数** | **20+** | **20+** |

**差异：** OpenClaw 通道内置于 Gateway 进程，开箱即用更多；Hermes 的网关是可选模式，但覆盖的广度相当。

---

## 四、Agent 运行时与会话

| 功能 | OpenClaw | Hermes Agent |
|------|----------|-------------|
| 运行时类型 | 内嵌 Agent Runtime，一个 Gateway 一个进程 | CLI 驱动，支持多种终端后端 |
| 会话隔离 | 每 Agent 独立 workspace + session store | 配置文件隔离，v0.6 引入 Profiles |
| 会话持久化 | JSONL 文件 | 文件系统 + 可插拔内存后端 |
| 流式输出 | ✅ Block streaming，支持 Steering Queue | ✅ 流式响应 |
| 子 Agent / 委托 | ✅ sessions_spawn（隔离子会话） | ✅ delegate_task（并发子 Agent，默认 3 个） |
| 多 Agent 路由 | ✅ 完整的多 Agent 绑定系统 | ✅ 多 Agent 配置 |
| ACP 协议 | ✅ Agent Communication Protocol | ✅ ACP 兼容（VS Code、Zed、JetBrains） |
| API Server | ✅ WebSocket API | ✅ OpenAI 兼容 HTTP 端点 |

---

## 五、记忆与学习系统 ⭐ 核心差异

这是两者**最本质的区别**。

### OpenClaw 的记忆系统

- **MEMORY.md / USER.md** — 用户编辑的静态文件，注入到系统提示
- **memory_search** — 语义搜索索引化的会话记录
- **QMD 后端** — 可跨 Agent 搜索会话转录
- **无原生学习循环** — 每次会话从相同基线开始，不自动从经验中提取可复用模式
- 技能需要**手动编写**或从 ClawHub 安装

### Hermes Agent 的学习循环

这是 Hermes 的标志性特性：

1. **任务执行** → 分解目标、选择工具、执行
2. **结果评估** → 判断成功/失败，分析用户反馈（显式+隐式）
3. **技能提取** → 从成功任务中提取推理模式，保存为命名技能模板
4. **技能优化** → 遇到类似场景时对比新旧方法，自动更新技能
5. **技能检索** → 新任务到来时自动搜索技能库，应用最佳模式

**附加特性：**
- **自动策展（v0.12 "The Curator"）** — 后台进程独立评估技能库，删除不再使用的技能，合并冗余技能
- **用户建模** — 跨会话跟踪：任务偏好、决策历史、常见模式、反馈信号
- **三层次记忆** — 超越简单的向量数据库+草稿板模式
- **Honcho 辩证用户建模** — 可插拔记忆提供商（Honcho、OpenViking、Mem0 等）

### 学习能力对比

| 能力 | OpenClaw | Hermes Agent |
|------|----------|-------------|
| 从经验创建技能 | ❌（需手动编写） | ✅ 自动 |
| 技能随时间优化 | ❌ | ✅ 自动 |
| 跨会话用户建模 | 有限（MEMORY.md 静态文件） | ✅ 动态学习 |
| 技能库自动清理 | ❌ | ✅ Curator 后台进程 |
| 技能可移植性 | ✅ ClawHub 市场 | ✅ agentskills.io 开放标准 |

---

## 六、技能系统

| 特性 | OpenClaw | Hermes Agent |
|------|----------|-------------|
| 技能格式 | SKILL.md + 支持文件 | agentskills.io 开放标准 |
| 技能来源 | 工作区、ClawHub | 内置 + Skills Hub 社区贡献 |
| 技能市场 | ✅ ClawHub（v4.1） | ✅ agentskills.io |
| 热加载 | ✅ 技能热重载 | ✅ |
| 自动创建 | ❌ | ✅ 从成功任务提取 |
| 自动优化 | ❌ | ✅ 持续改进 |
| 自动清理 | ❌ | ✅ Curator 后台 |

---

## 七、工具与集成

| 工具类别 | OpenClaw | Hermes Agent |
|----------|----------|-------------|
| 文件操作 | read / write / edit / apply_patch / exec | 文件编辑、终端执行 |
| Web 搜索 | 12+ 引擎（Brave、Tavily、Perplexity 等） | ✅ 内置搜索（通过 Tool Gateway） |
| 浏览器自动化 | ✅ Playwright | ✅ 多后端 + Camofox 反检测浏览器 |
| MCP 集成 | ✅ | ✅（支持 MCP OAuth 2.1，可作为 MCP Server） |
| 图像生成 | ✅ 共享能力面 | ✅ FAL.ai（9 种模型） |
| 视频生成 | ✅ | ❌ |
| TTS / 语音 | ✅ 多提供商 | ✅ 10+ 提供商 |
| 语音模式 | ✅ | ✅ 实时语音（CLI、Telegram、Discord VC） |
| 代码执行 | ✅ exec | ✅ execute_code（Python 脚本调用工具） |
| 沙箱 | ✅ fs-safe + 可配置沙箱 | ✅ Docker 硬隔离 + Singularity + Modal |
| 批量处理 | ❌ | ✅ 批量提示处理，生成训练数据 |
| 事件钩子 | ✅ | ✅ Gateway hooks + Plugin hooks |
| 凭证池 | ❌ | ✅ 多 API Key 自动轮换 |
| 回滚/检查点 | ❌ | ✅ 自动文件快照 + /rollback |

---

## 八、安全模型 ⭐ 重要差异

### OpenClaw

- **个人助理信任模型** — 假设单用户/单信任边界
- 安全审计命令：`openclaw security audit`
- 文件操作通过 `@openclaw/fs-safe`（根边界限制、原子写入）
- 通道级别的 allowlist、配对码
- 早期 CVEs：CVE-2026-25253、CVE-2026-25891、CVE-2026-26102（已修复）
- Microsoft 曾评价其默认权限模型"对企业环境过于宽松"
- 非多租户安全边界

### Hermes Agent

**七层安全架构**（从设计之初就文档化）：

1. **用户授权** — 每平台 allowlist、DM 配对（8 字符无歧义字母表、1 小时 TTL、5 次失败锁定）
2. **危险命令审批** — 三层模式：手动（默认）/ 智能（LLM 评估）/ 关闭 + **硬线黑名单**（不可绕过）
3. **容器隔离** — Docker 硬安全标志、无特权模式、无敏感挂载
4. **MCP 凭证过滤** — 仅暴露批准的 env vars、SSRF 保护、Tirith 预执行扫描
5. **上下文文件扫描** — 检查 prompt injection 模式
6. **审批按钮** — 消息平台上的交互式审批 UI
7. **YOLO 模式** — `--yolo` 或 `/yolo` 显式激活

**安全对比总结：** Hermes 的安全模型更系统化、分层化、从设计之初就内置；OpenClaw 的安全是演进式的，早期有 CVE 记录，v4.0 后大幅改进。

---

## 九、部署选项

| 部署方式 | OpenClaw | Hermes Agent |
|----------|----------|-------------|
| 本地安装 | ✅ npm install -g openclaw | ✅ curl 安装脚本 / Desktop 安装器 |
| Docker | ✅ | ✅（硬安全标志） |
| 桌面应用 | ✅ macOS 菜单栏应用 | ✅ Hermes Desktop（Windows/macOS） |
| 移动节点 | ✅ iOS + Android（配对、Canvas、相机、屏幕） | ❌ 无官方移动应用 |
| 服务器部署 | ✅ VPS / 自托管 | ✅ VPS / GPU 集群 |
| Serverless | ❌ | ✅ Daytona、Modal（空闲近乎零成本） |
| SSH 远程 | ✅ | ✅（作为终端后端） |
| HPC | ❌ | ✅ Singularity 支持 |

---

## 十、自动化

| 特性 | OpenClaw | Hermes Agent |
|------|----------|-------------|
| Cron 调度 | ✅ 系统事件 + Agent Turn | ✅ 自然语言或 cron 表达式 |
| Heartbeat | ✅ 定期检查 | ❌（用 cron 替代） |
| 工作流管道 | ✅ Lobster 管道 | ✅ execute_code 多步折叠 |
| 事件钩子 | ✅ | ✅ 网关 + 插件钩子 |
| 批量处理 | ❌ | ✅ 批量提示/任务并行 |

---

## 十一、UI 与界面

| 界面 | OpenClaw | Hermes Agent |
|------|----------|-------------|
| Web 控制台 | ✅ Control UI（http://127.0.0.1:18789） | ✅ Web Dashboard |
| CLI | ✅ openclaw 命令 | ✅ hermes 命令 |
| 桌面应用 | ✅ macOS 菜单栏 | ✅ Hermes Desktop（Win/Mac） |
| 移动端 | ✅ iOS + Android 节点 | ❌ |
| Canvas 系统 | ✅ 嵌入式 HTML 渲染 | ❌ |
| 主题/皮肤 | ✅ | ✅ Skins & Themes |
| IDE 集成 | ✅ ACP | ✅ ACP（VS Code、Zed、JetBrains） |

---

## 十二、模型提供商

| 提供商 | OpenClaw | Hermes Agent |
|--------|----------|-------------|
| OpenAI | ✅ | ✅ |
| Anthropic (Claude) | ✅ | ✅ |
| Google (Gemini) | ✅ | ✅ |
| Ollama (本地) | ✅ | ✅ |
| OpenRouter | ✅ | ✅ |
| Nous Portal | ❌ | ✅（MiMo v2 Pro 免费） |
| xAI (Grok) | ✅（2026.5.x-beta） | ❌ |
| Tencent Cloud | ✅（2026.5.x-beta） | ❌ |
| vLLM / SGLang | ✅ | ✅ |
| **总计** | **35+** | **大量**（含自定义端点） |
| 提供商路由 | ✅ 基本 | ✅ 高级（排序、白名单、优先级、回退链） |
| 回退提供商 | ❌ | ✅ 自动故障转移 |
| 凭证池 | ❌ | ✅ 多 Key 自动轮换 |
| 提示缓存 | ✅ | ✅ Claude 前缀缓存 |

---

## 十三、各自特色与优势

### OpenClaw 的特色与优势

1. **🦞 通道之王** — 20+ 消息平台内置于一个 Gateway 进程，开箱即用，配置简单
2. **📱 移动节点** — 唯一支持 iOS/Android 配对节点的框架，可远程使用相机、屏幕录制、位置
3. **🎨 Canvas 系统** — 嵌入式 HTML 渲染，可在聊天中呈现富交互内容
4. **🏗️ 成熟架构** — v4.0 "The Agent OS" 架构重写，Gateway 设计成熟
5. **🔌 插件生态** — ClawHub 市场 + 捆绑插件 + 第三方插件
6. **🔧 工具丰富度** — 12+ 搜索引擎、视频生成、Lobster 工作流管道
7. **⚡ 部署简单** — `npm install -g openclaw && openclaw onboard` 几分钟内启动
8. **🔐 fs-safe** — 根边界限制的文件操作安全层
9. **多 Agent 绑定** — 一个 Gateway 运行多个完全隔离的 Agent，各有独立 workspace
10. **商业支持** — openclawai.io SaaS 选项

### Hermes Agent 的特色与优势

1. **🧠 自学习循环** — 最大差异化特性：从经验自动创建和优化技能，跨会话持续改进
2. **👤 用户建模** — 学习你的偏好、决策模式，不再重复问已知问题
3. **🔒 七层安全** — 从设计之初就内置的系统化安全架构，危险命令黑名单不可绕过
4. **🖥️ 多终端后端** — 本地、Docker、SSH、Daytona、Modal、Singularity，灵活部署
5. **🕵️ Camofox 反检测浏览器** — 专为有机器人检测的网站优化
6. **🔌 MCP Server 模式** — 自身可作为 MCP Server 为其他 Agent 提供服务
7. **🧹 Curator 自动策展** — 后台自动清理和合并技能库
8. **🔄 回退提供商链** — 主模型失败时自动切换到备用模型
9. **🔑 凭证池** — 多个 API Key 自动轮换，应对限速
10. **📊 批量处理** — 支持批量提示处理，可用于训练数据生成
11. **🔄 检查点/回滚** — 自动文件快照，`/rollback` 一键恢复
12. **🎭 深度个性化** — SOUL.md + 可换 personality 预设

---

## 十四、各自劣势与不足

### OpenClaw 的劣势

1. **❌ 无原生学习循环** — 每次会话从零开始，不会自动从经验中提取可复用技能
2. **🔒 安全模型演进式** — 早期有 CVE 记录，v4.0 后大幅改进，但非设计之初就内置
3. **⚖️ 非多租户** — 官方明确声明不是敌对多租户安全边界
4. **📦 单体 Gateway** — 所有通道适配器加载到同一个守护进程，出问题可能影响全局
5. **🧩 技能需手动编写** — 没有自动技能创建机制
6. **💼 项目历史包袱** — 从 Moltbot 更名而来，早期安全问题留下 CVE 记录
7. **🖥️ 无 Serverless 部署** — 不支持 Daytona/Modal 等 Serverless 环境
8. **🔁 无回退提供商** — 主模型失败时没有自动故障转移机制
9. **🔑 无凭证池** — 不支持多个 API Key 自动轮换
10. **📊 无批量处理** — 不支持批量提示/任务处理

### Hermes Agent 的劣势

1. **🆕 项目年轻** — 2026年3月才公开发布，生态成熟度不如 OpenClaw
2. **📱 无移动节点** — 没有官方 iOS/Android 应用
3. **🌐 无 SaaS 选项** — 纯开源，没有官方托管服务
4. **🏗️ 架构复杂度** — 学习循环增加了设置和理解的复杂度
5. **📡 通道非内置** — Gateway 是可选模式，不像 OpenClaw 那样开箱即用
6. **🎨 无 Canvas** — 没有嵌入式 HTML 渲染能力
7. **🔌 插件生态较小** — 没有类似 ClawHub 的成熟市场
8. **⚙️ 配置复杂度** — 安全层多、配置选项多，上手门槛高于 OpenClaw
9. **🔧 部分工具缺失** — 没有视频生成、没有 Zalo/Twitch/Nostr 等通道
10. **📈 企业级验证不足** — 在受监管工作负载中的验证尚不充分

---

## 十五、总结对比表

| 维度 | OpenClaw | Hermes Agent |
|------|----------|-------------|
| 核心理念 | 多通道 Agent OS / 网关中心 | 自学习 Agent / 技能循环中心 |
| 学习能力 | ❌ 无原生学习 | ✅ 自动创建+优化技能 |
| 用户建模 | 有限（静态文件） | ✅ 动态跨会话学习 |
| 通道覆盖 | ✅ 20+，内置多 | ✅ 20+，网关模式 |
| 安全模型 | ⚠️ 演进式，v4.0 后改善 | ✅ 七层系统化设计 |
| 移动端 | ✅ iOS + Android 节点 | ❌ 无 |
| Serverless | ❌ | ✅ Daytona / Modal |
| 容器后端 | ✅ Docker | ✅ Docker + Singularity + Modal + Daytona |
| Canvas/富交互 | ✅ | ❌ |
| 技能市场 | ✅ ClawHub | ✅ agentskills.io |
| 自动技能策展 | ❌ | ✅ Curator |
| 回退提供商 | ❌ | ✅ |
| 凭证池 | ❌ | ✅ |
| 批量处理 | ❌ | ✅ |
| 检查点/回滚 | ❌ | ✅ |
| 部署简易度 | ⭐⭐⭐⭐⭐（npm install） | ⭐⭐⭐⭐（curl 安装） |
| 项目成熟度 | ⭐⭐⭐⭐⭐（v4.x） | ⭐⭐⭐（v0.12，快速迭代中） |
| 企业就绪度 | ⚠️ 需额外加固 | ✅ 安全设计更完善 |
| 学习曲线 | 低 | 中 |

---

## 十六、选型建议

### 选 OpenClaw 如果你：

- **需要从手机/平板控制 Agent** — 移动节点是独有优势
- **需要连接大量消息平台** — 一个 Gateway 搞定所有通道
- **想要快速上手、低门槛部署** — npm install 几分钟搞定
- **需要 Canvas 富交互** — 嵌入式 HTML 渲染
- **偏好成熟稳定的项目** — v4.x 架构已经过多次迭代
- **需要 ClawHub 生态** — 现成的技能市场

### 选 Hermes Agent 如果你：

- **需要 Agent 持续进步** — 学习循环是最大价值，重复性任务越多越划算
- **安全是首要考量** — 七层安全架构，从设计之初就内置
- **需要灵活部署** — 本地、Docker、SSH、Serverless、HPC 都支持
- **需要模型回退保障** — 主模型挂了自动切备用
- **处理大量 API 调用** — 凭证池自动轮换 Key
- **需要批量处理/训练数据** — 批量提示处理能力
- **想要个性化体验** — 用户建模让 Agent 越来越懂你

### 也可以两者都用

社区中已经有人将 Hermes 作为 OpenClaw 的"看门狗"（watchdog）使用，两者互补：
- OpenClaw 做**通道层和日常交互**
- Hermes 做**深度学习和技能积累**

---

## 参考来源

- [OpenClaw 官方文档](https://docs.openclaw.ai)
- [Hermes Agent 官方文档](https://hermes-agent.nousresearch.com/docs)
- [PetronellaTech: OpenClaw vs Hermes Agent 2026](https://petronellatech.com/blog/openclaw-vs-hermes-agent-2026)
- [innFactory: OpenClaw vs Hermes Agent Comparison](https://innfactory.ai/en/blog/openclaw-vs-hermes-agent-comparison)
- [MindStudio: What Is Hermes Agent?](https://www.mindstudio.ai/blog/what-is-hermes-agent-openclaw-alternative)
- [NVIDIA Blog: Hermes Unlocks Self-Improving AI Agents](https://blogs.nvidia.com/blog/rtx-ai-garage-hermes-agent-dgx-spark)

---

## 附录：小墨审核记录（2026-06-26）

> 以下为基于 Docker 容器内 Hermes Agent v0.17.0 实际运行状态的审核意见。

### 整体评价

文档结构清晰、内容详实，394 行覆盖 16 个维度，作为技术选型参考文档质量很高。以下列出 3 处事实偏差和 2 处可优化点，供后续修订参考。

### 事实偏差

**1. xAI (Grok) — Hermes 实际支持**

第十二章表格标注 Hermes 不支持 xAI (Grok)，但实际 Hermes 通过 `XAI_API_KEY` 环境变量支持 xAI 模型调用，且内置了 `x_search` 搜索工具和 `xai` 视频生成插件。建议改为 ✅。

**2. 视频生成 — Hermes 实际支持**

第七章表格标注 Hermes 不支持视频生成，但实际 Hermes 通过 `fal` 插件（Veo 3.1、Kling、Pixverse）和 `xai` 插件（Grok Imagine）支持视频生成，只是默认未启用插件。建议改为 ✅。

**3. 模型提供商数量**

第十二章对比 OpenClaw "35+" 与 Hermes "大量" 不够精确。Hermes 实际支持的提供商包括：OpenAI、Anthropic、Google Gemini、DeepSeek、xAI/Grok、OpenRouter、Nous Portal、GitHub Copilot、Ollama、vLLM/SGLang、HuggingFace、Z.AI/GLM、MiniMax、Kimi/Moonshot、Alibaba/DashScope、Xiaomi MiMo、Kilo Code、OpenCode 等，加上自定义端点。建议标出具体数量或写 "20+"。

### 可优化点

**4. Hermes 版本号过时**

文档多处引用 v0.12，但当前 Docker 容器实际运行版本为 v0.17.0（经 `hermes doctor` 确认）。建议更新为最新版本号。

**5. 安全模型可补充细节**

> "Hermes 七层安全架构"

描述准确，但可补充：Hermes 的危险命令黑名单是**硬编码不可绕过的**，即使 `--yolo` 模式也不能绕过黑名单命令——这是与 OpenClaw 安全模型的一个关键区别。

### 已确认准确的内容

| 条目 | 结论 |
|------|------|
| 项目哲学/核心理念 | ✅ 准确 |
| 架构对比 | ✅ 准确 |
| 通道/平台支持 | ✅ 准确 |
| 记忆与学习系统 | ✅ 准确，核心差异 |
| 技能系统 | ✅ 准确 |
| 工具与集成 | ✅ 除视频生成外均准确 |
| 部署选项 | ✅ 准确 |
| 选型建议 | ✅ 合理 |

---

## 附录二：Hermes 七层安全架构详解（2026-06-26）

> 基于 Hermes Agent 官方文档 [Security](https://hermes-agent.nousresearch.com/docs/user-guide/security) 及 Docker 容器内 v0.17.0 实际运行状态整理。

### 总览

Hermes 的七层安全架构是**从设计之初就内置的防御纵深（defense-in-depth）**，而非事后修补。每一层解决一个不同的攻击面，且层与层之间独立运作——即使某一层被绕过，下一层仍然生效。

---

### 第一层：用户授权（User Authorization）

**谁可以跟 Agent 说话。**

**Hermes 实现：**
- **每平台 allowlist** — `TELEGRAM_ALLOWED_USERS=12345,67890` 等
- **全局 allowlist** — `GATEWAY_ALLOWED_USERS=12345,67890`
- **DM 配对系统** — 8 字符无歧义字母表（去掉了 0/O/1/I）、加密随机生成、1 小时 TTL、每用户 10 分钟限频、最多 3 个待审批码、5 次失败锁定 1 小时、文件权限 `chmod 0600`
- **默认拒绝** — 没有任何 allowlist 时，所有用户被拒绝

**vs OpenClaw：** OpenClaw 也有 allowlist 和配对码，但配对码是 6 位数字（10^6 = 100 万组合），Hermes 是 8 位 32 字符字母表（32^8 ≈ 10^12 组合），暴力破解难度高 1000 万倍。且 Hermes 有显式的失败锁定机制，OpenClaw 没有。

---

### 第二层：危险命令审批（Dangerous Command Approval）

**执行破坏性操作前必须人工确认。**

**Hermes 实现：**
- **三种模式：**
  - `manual`（默认）— 每次触发都弹窗
  - `smart` — 辅助 LLM 评估风险，低风险自动放行，高风险自动拒绝，不确定的升级到人工
  - `off` — 完全关闭（仅限可信环境）
- **审批超时** — 默认 60 秒无响应则拒绝（fail-closed）
- **审批流：** CLI 弹 4 选项（一次/本次会话/永久允许/拒绝），网关弹交互按钮
- **永久 allowlist** — 写入 `config.yaml`，后续会话自动放行

**🔥 关键差异：硬线黑名单（Hardline Blocklist）**

这是 Hermes 最硬的一层，也是跟 OpenClaw 最本质的区别：

```bash
rm -rf /                    # 擦除根文件系统
:(){ :|:& };:               # fork 炸弹
mkfs.* 挂载中的根设备          # 格式化系统盘
dd if=/dev/zero of=/dev/sd* # 清零物理磁盘
管道未信任 URL 到 sh          # RCE 攻击向量
```

**这些命令在任何模式下都不可执行：**
- `--yolo` 模式 → ❌ 不行
- `approvals.mode: off` → ❌ 不行
- Cron 任务的 `approve` 模式 → ❌ 不行
- 用户点了"始终允许" → ❌ 不行

这个黑名单在审批层**之前**拦截，没有任何覆盖标志。

**vs OpenClaw：** OpenClaw 有 `security audit` 命令和 fs-safe 层，但**没有硬线黑名单**。它的审批是软性的，理论上可以被绕过。早期 CVE（CVE-2026-25253 等）也说明其安全模型是演进式的，不是设计之初就内置的。

---

### 第三层：容器隔离（Container Isolation）

**Agent 在沙箱里搞破坏也伤不到宿主机。**

**Hermes 实现：**
Docker 容器启动时强制应用的安全标志：

```python
--cap-drop ALL                # 丢弃所有 Linux 内核能力
--cap-add DAC_OVERRIDE        # 仅保留写入挂载目录的能力
--cap-add CHOWN               # 包管理器需要
--cap-add FOWNER              # 同上
--security-opt no-new-privileges  # 阻止提权
--pids-limit 256              # 限制进程数，防 fork 炸弹
--tmpfs /tmp:rw,nosuid,size=512m   # 限制 /tmp 大小
--tmpfs /var/tmp:rw,noexec,nosuid   # /var/tmp 不可执行
```

资源限制可配置：

```yaml
container_cpu: 1
container_memory: 5120   # MB
container_disk: 51200    # MB
```

终端后端安全对比：

| 后端 | 隔离性 | 危险命令检查 | 适用场景 |
|------|--------|------------|---------|
| local | ❌ 无隔离 | ✅ 需要 | 开发环境 |
| ssh | ✅ 远程机器 | ✅ 需要 | 远程服务器 |
| docker | ✅ 容器隔离 | ❌ 跳过（容器即边界） | **生产网关** |
| modal | ✅ 云沙箱 | ❌ 跳过 | 可扩展云隔离 |
| singularity | ✅ 容器 | ❌ 跳过 | HPC 环境 |

**vs OpenClaw：** OpenClaw 也有 Docker 部署，但**没有 Hermes 这么严格的默认安全标志**（cap-drop ALL、no-new-privileges、pids-limit 等）。OpenClaw 的 fs-safe 层是文件操作级别的安全，不是进程/容器级别的隔离。

---

### 第四层：MCP 凭证过滤（MCP Credential Filtering）

**MCP 子进程拿不到宿主机的敏感环境变量。**

**Hermes 实现：**
MCP stdio 子进程只接收白名单环境变量：

```
PATH, HOME, USER, LANG, LC_ALL, TERM, SHELL, TMPDIR
```

加上 `XDG_*` 变量。**其他所有环境变量（API Key、Token、密钥）全部剥离。**

只有在 MCP 配置中显式声明的 env 才会传入：

```yaml
mcp_servers:
  github:
    command: "npx"
    args: ["-y", "@modelcontextprotocol/server-github"]
    env:
      GITHUB_PERSONAL_ACCESS_TOKEN: "ghp_..."  # 只有这个传进去
```

错误信息也会被清理，匹配到 `sk-...`、`ghp_...`、`Bearer`、`key=` 等模式自动替换为 `[REDACTED]`。

**vs OpenClaw：** OpenClaw 也支持 MCP，但**没有文档化的凭证过滤机制**。MCP 子进程默认继承父进程的全部环境变量，存在凭证泄露风险。

---

### 第五层：上下文文件扫描（Context File Scanning）

**防止项目文件中的 prompt injection 攻击 Agent。**

**Hermes 实现：**
`AGENTS.md`、`CLAUDE.md`、`.cursorrules`、`SOUL.md` 等上下文文件在注入系统提示前会经过扫描器检查：

- 指示忽略/无视先前指令的模式
- 隐藏的 HTML 注释中的可疑关键词
- 尝试读取密钥文件（`.env`、credentials、`.netrc`）
- 通过 curl 外泄凭证
- 不可见 Unicode 字符（零宽空格、双向覆盖符）

命中扫描器的内容被替换为：

```
[BLOCKED: AGENTS.md contained potential prompt injection. Content not loaded.]
```

**vs OpenClaw：** OpenClaw 没有这个能力。它的 `MEMORY.md` / `USER.md` 直接注入到系统提示，没有经过 prompt injection 扫描。这是 Hermes 独有的安全层。

---

### 第六层：跨会话隔离（Cross-Session Isolation）

**一个会话不能访问另一个会话的数据或状态。**

**Hermes 实现：**
- 每个会话有独立的 session store
- Cron 任务的存储路径经过路径遍历攻击防护
- 会话文件权限严格控制
- 不同 profile 的数据完全隔离

**vs OpenClaw：** OpenClaw 也有会话隔离（每个 Agent 独立 workspace），但**没有文档化的路径遍历防护**。Hermes 在这层多了一个 Cron 存储路径的硬化处理。

---

### 第七层：输入清理（Input Sanitization）

**防止 shell 注入和 SSRF 攻击。**

**Hermes 实现：**
Terminal 工具后端的**工作目录参数**会经过白名单验证，防止恶意路径注入。所有 URL 工具（web search、web extract、vision、browser）在请求前验证 URL，防止 SSRF：

- 私有网络（RFC 1918）：`10.0.0.0/8`、`172.16.0.0/12`、`192.168.0.0/16`
- 回环地址：`127.0.0.0/8`、`::1`
- 链路本地：`169.254.0.0/16`（含云元数据 `169.254.169.254`）
- CGNAT：`100.64.0.0/10`（Tailscale、WireGuard VPN）
- 云元数据 hostname：`metadata.google.internal`
- **重定向链每跳重新验证**，防止重定向绕过

另外还有 **Tirith 预执行扫描** — 在命令执行前检测同形字 URL 欺骗、管道到解释器模式（`curl | bash`）、终端注入攻击。Tirith 自带 SHA-256 校验和验证 + cosign 来源验证。

**vs OpenClaw：** OpenClaw 有 fs-safe 层（根边界限制），但**没有 SSRF 保护**、**没有 Tirith 扫描**、**没有重定向链验证**。OpenClaw 的 URL 处理相对简单。

---

### 总结对比表

| 安全层 | Hermes | OpenClaw |
|--------|--------|----------|
| **① 用户授权** | 平台 allowlist + 全局 allowlist + 加密配对码（8 字符 32 字母表）+ 5 次失败锁定 | allowlist + 6 位数字配对码，无失败锁定 |
| **② 危险命令审批** | 三层模式（manual/smart/off）+ **硬线黑名单不可绕过** | 有审批，但无硬线黑名单 |
| **③ 容器隔离** | cap-drop ALL + no-new-privileges + pids-limit + tmpfs 限制 | 有 Docker 部署，但无默认硬化标志 |
| **④ MCP 凭证过滤** | 环境变量白名单 + 错误信息自动脱敏 | 无文档化凭证过滤 |
| **⑤ 上下文文件扫描** | Prompt injection 检测 + 不可见 Unicode 检测 | ❌ 无 |
| **⑥ 跨会话隔离** | 独立 session store + 路径遍历防护 | 有 workspace 隔离，无路径遍历防护 |
| **⑦ 输入清理** | SSRF 保护 + 重定向链验证 + Tirith 预执行扫描 | fs-safe 根边界限制，无 SSRF/Tirith |

**额外 Hermes 有但 OpenClaw 没有的：**
- 🔄 **回退提供商链** — 主模型挂了自动切备用，避免单点故障
- 🔑 **凭证池** — 多个 API Key 自动轮换，应对限速
- 📋 **供应链安全扫描** — 启动时检查已知被投毒的 Python 包版本
- 📦 **懒加载安全** — 可选依赖按需安装，不引入不必要的攻击面

**OpenClaw 有但 Hermes 没有的：**
- 📱 移动节点（iOS/Android）— 但这是功能不是安全
- 🎨 Canvas 系统 — 同上


hermes 

自建技能目录、默认提供的技能目录、

镜像

openAI 协议接口

目录挂载