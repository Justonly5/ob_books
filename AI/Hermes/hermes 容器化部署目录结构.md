---
title: "Hermes 容器化部署目录结构"
tags: [Hermes, Docker, K8s, Deployment, Directory]
created: 2026-07-25
---

# Hermes 容器化部署目录结构

Hermes 容器化部署采用 **两层架构**：不可变的安装树与可变的运行时数据卷分离。

---

## 架构概览

```
容器内部
├── /opt/hermes/          ← 不可变安装树（镜像自带，root 只读）
└── /opt/data/            ← 运行时数据卷（PVC/挂载卷，持久化）
     ├── <标准 Hermes home 布局>
     └── profiles/        ← 多 profile 支持（每个 profile 一个子目录）
```

- **`HERMES_HOME=/opt/data`** — 容器内环境变量 `ENV HERMES_HOME=/opt/data`
- 宿主机挂载 `-v ~/.hermes:/opt/data` 即可持久化所有状态
- 镜像升级只需拉新版本，数据卷完全保留

---

## 一、不可变安装树 `/opt/hermes/`

由 Dockerfile 构建，容器运行期间**不可修改**。Hermes 的 `hermes` 用户无写入权限。

| 路径 | 内容 |
|------|------|
| `/opt/hermes/.venv/` | Python 虚拟环境（所有依赖） |
| `/opt/hermes/.playwright/` | Playwright 浏览器二进制（Chromium） |
| `/opt/hermes/.install_method` | 安装标记（内容为 `docker`） |
| `/opt/hermes/.hermes_build_sha` | 构建时的 Git commit SHA |
| `/opt/hermes/docker/` | 容器启动脚本、Hook 脚本 |
| `/opt/hermes/hermes_cli/` | Hermes 核心 CLI 代码 |
| `/opt/hermes/plugins/` | 内置插件 |
| `/opt/hermes/optional-skills/` | 可选技能模板 |
| `/opt/hermes/ui-tui/` | TUI 终端界面 |
| `/opt/hermes/scripts/` | 辅助脚本 |
| `/opt/hermes/bin/hermes` | `hermes` 命令执行 shim（特权降级包装） |
| `/opt/hermes/node_modules/` | Node.js 依赖（TUI 构建、WhatsApp 桥接） |
| `/opt/hermes/.venv/bin/hermes` | 真正的 Hermes 可执行文件路径 |

> 运行时 lazy install 被禁用（`HERMES_DISABLE_LAZY_INSTALLS=1`），可选依赖安装目标被重定向到 `/opt/data/lazy-packages`。

---

## 二、运行时数据卷 `/opt/data/`

由 `docker/stage2-hook.sh` 中的 `as_hermes mkdir -p` 块在首次启动时创建。

### 2.1 核心配置文件

| 路径 | 说明 | 持久化策略 |
|------|------|-----------|
| `config.yaml` | Hermes 主配置（模型、平台、网关、插件等） | 首次启动从 `cli-config.yaml.example` 种子生成 |
| `.env` | API 密钥、Token 等敏感信息（权限 600） | 首次启动从 `.env.example` 种子生成 |
| `SOUL.md` | Agent 人格/身份设定 | 首次启动从 `docker/SOUL.md` 种子生成 |
| `gateway_state.json` | Gateway 运行状态（`running`/`stopped`） | 生命周期管理自动写入 |

### 2.2 会话与记忆

| 路径 | 说明 | 持久化需求 |
|------|------|-----------|
| `state.db` | **SQLite 会话存储** — 所有对话历史、记忆索引 | **必须持久化** |
| `sessions/` | 会话转录文件（纯文本日志） | 建议持久化 |
| `memories/` | 持久化记忆存储（Hermes 跨会话记忆） | **必须持久化** |
| `auth.json` | OAuth Token 等认证信息 | 建议持久化 |

### 2.3 技能与扩展

| 路径 | 说明 |
|------|------|
| `skills/` | 安装的技能（Hub 下载、agent 创建） |
| `skins/` | 自定义 CLI 主题 |
| `plugins/` | 用户安装的插件 |
| `hooks/` | 事件钩子（event hooks） |
| `cron/` | 定时任务定义 |
| `plans/` | 执行计划（agent 编写的计划文件） |
| `lazy-packages/` | 运行时 lazy install 重定向目标 |

### 2.4 日志

| 路径 | 说明 |
|------|------|
| `logs/` | 通用日志目录 |
| `logs/gateways/` | 每个 profile 的 gateway 独立日志 |
| `logs/gateways/<profile>/current` | 当前 profile 的 gateway 日志（s6-log 轮转，10 个 × 1 MB） |
| `logs/container-boot.log` | 容器启动审计日志（记录 profile 恢复情况） |

### 2.5 备份与恢复

| 路径 | 说明 |
|------|------|
| `backups/` | 升级前自动备份、迁移前备份（`pre-migration-*.zip`），可通过 `hermes import` 恢复 |

### 2.6 工作区与工具

| 路径 | 说明 |
|------|------|
| `workspace/` | 工作区目录（agent 可操作的文件） |
| `home/` | **每个 profile 的 HOME** — 用于 agent 子进程（`git`、`ssh`、`gh`、`npm` 等 CLI 工具），避免污染 `/root` |
| `pairing/` | 设备配对信息 |
| `platforms/pairing/` | 平台级配对信息 |

### 2.7 多 Profile 支持

| 路径 | 说明 |
|------|------|
| `profiles/<name>/` | 每个 profile 的独立数据目录 |
| `profiles/<name>/SOUL.md` | 该 profile 的 Agent 人格 |
| `profiles/<name>/config.yaml` | 该 profile 的独立配置 |
| `profiles/<name>/.env` | 该 profile 的独立密钥 |
| `profiles/<name>/gateway_state.json` | 该 profile 的 gateway 运行状态 |
| `profiles/<name>/memories/` | 该 profile 的独立记忆 |
| `profiles/<name>/skills/` | 该 profile 的独立技能 |

每个 profile 在容器内对应一个独立的 s6 受监督服务 `gateway-<name>`，启动/停止/重启与 root profile 完全隔离。

---

## 三、容器启动时目录结构创建流程

来自 `docker/stage2-hook.sh` 的 `as_hermes mkdir -p` 块：

```bash
as_hermes mkdir -p \
    "$HERMES_HOME/backups" \
    "$HERMES_HOME/cron" \
    "$HERMES_HOME/sessions" \
    "$HERMES_HOME/logs" \
    "$HERMES_HOME/logs/gateways" \
    "$HERMES_HOME/hooks" \
    "$HERMES_HOME/memories" \
    "$HERMES_HOME/skills" \
    "$HERMES_HOME/skins" \
    "$HERMES_HOME/plans" \
    "$HERMES_HOME/workspace" \
    "$HERMES_HOME/home" \
    "$HERMES_HOME/pairing" \
    "$HERMES_HOME/platforms/pairing" \
    "$HERMES_HOME/lazy-packages"
```

这些目录在 `stage2-hook.sh` 的 **cont-init.d** 阶段以 `hermes` 用户身份创建，确保所有者和权限正确。

---

## 四、K8s 挂载建议

### 需要持久化的目录（PVC）

| 优先级 | 路径 | 原因 |
|--------|------|------|
| **必须** | `state.db` | 所有会话状态、记忆索引 |
| **必须** | `memories/` | 跨会话记忆 |
| **必须** | `skills/` | 安装的技能 |
| **必须** | `auth.json` | OAuth 凭证 |
| 建议 | `sessions/` | 会话转录 |
| 建议 | `cron/` | 定时任务 |
| 建议 | `home/` | 子进程 HOME 状态 |
| 建议 | `backups/` | 备份存档 |

### 不需要持久化的目录

| 路径 | 原因 |
|------|------|
| `logs/` | 建议重定向到 stdout（`docker logs`）或集中式日志系统 |
| `cache/` | 临时缓存 |
| `lazy-packages/` | 可重建 |

### ConfigMap 挂载

| 文件 | 挂载方式 | 注意 |
|------|---------|------|
| `config.yaml` | ConfigMap subPath | **只读** — 运行时配置变更（如启用插件）会失败 |
| `.env` | Secret 环境变量注入 | 不要挂载到 PVC 上 |

> **建议：** 对于需要运行时动态配置的场景，把 `config.yaml` 放在 PVC 上，仅首次启动时从 ConfigMap 种子生成。

---

## 五、与其他路径的关系

| 路径 | 用途 |
|------|------|
| `/run/service/` | s6 服务目录（tmpfs，容器重启后重建） |
| `/run/service/gateway-default/` | 默认 profile 的 gateway s6 服务槽 |
| `/run/service/gateway-<name>/` | 各 profile 的 gateway s6 服务槽 |
| `/run/service/dashboard/` | Dashboard 的 s6 服务槽（`HERMES_DASHBOARD=1` 时启用） |
| `/opt/data/.local/bin/` | 在 PATH 中，用于用户级 CLI 工具安装 |

---

## 参考

- 目录种子代码：`docker/stage2-hook.sh`（`as_hermes mkdir -p` 块，约第 357 行）
- Profile 恢复逻辑：`hermes_cli/container_boot.py`（`reconcile_profile_gateways()`）
- 容器入口包装：`docker/main-wrapper.sh`
- 构建定义：`Dockerfile`（`ENV HERMES_HOME=/opt/data` 约第 296 行）
- 官方文档：`website/docs/user-guide/docker.md`（Persistent volumes 章节）