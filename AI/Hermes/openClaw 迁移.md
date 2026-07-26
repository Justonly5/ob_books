---
title: openClaw 迁移
tags:
  - Hermes
  - OpenClaw
  - Migration
  - CLI
created: 2026-07-25
---

# `hermes claw migrate` 命令详解

`hermes claw migrate` 是 Hermes 内置的 **OpenClaw → Hermes 迁移工具**。它从 OpenClaw 安装目录（默认为 `~/.openclaw`）读取配置、记忆、技能、API 密钥等数据，并将其导入到 Hermes 的 `~/.hermes/` 下。

---

## 命令定位

`hermes claw` 下有两条子命令：

| 子命令 | 用途 |
|--------|------|
| `hermes claw migrate` | 执行 OpenClaw → Hermes 的完整迁移 |
| `hermes claw cleanup` | 迁移完成后归档残留的 OpenClaw 目录（重命名为 `.pre-migration`） |

---

## 命令行参数

```bash
hermes claw migrate [options]
```

| 参数 | 默认值 | 说明 |
|------|--------|------|
| `--source` | `~/.openclaw` | 指定 OpenClaw 安装目录（自动回退检测 `.clawdbot`、`.moltbot` 等旧名称） |
| `--dry-run` | `false` | **预览模式** — 只展示将被迁移的内容，不执行任何写入 |
| `--preset` | `full` | 迁移预设：`user-data` 或 `full`（见下文预设对比） |
| `--overwrite` | `false` | 覆盖已存在的文件（默认遇到冲突会拒绝执行） |
| `--migrate-secrets` | `false` | **必须显式指定**才会迁移密钥（即使 `--preset full` 也不会自动包含） |
| `--no-backup` | `false` | 跳过迁移前的自动备份快照（默认会生成 `~/.hermes/backups/pre-migration-<timestamp>.zip`） |
| `--workspace-target` | 无 | 将 OpenClaw workspace 指令文件复制到指定绝对路径 |
| `--skill-conflict` | `skip` | 技能名称冲突策略：`skip` / `overwrite` / `rename` |
| `--yes`, `-y` | `false` | 跳过所有确认提示 |

---

## 两阶段执行流程

源码设计为清晰的 **Phase 1（预览）→ Phase 2（执行）** 两阶段：

### Phase 1 — 预览（always）

1. 检测 OpenClaw 源目录是否存在
2. 动态加载 `openclaw_to_hermes.py` 迁移脚本（从 `optional-skills` 或已安装的技能目录）
3. 创建 `Migrator(execute=False)` 实例，执行预览扫描
4. 输出详细的迁移报告（哪些会迁移、哪些冲突、哪些跳过）
5. 如果 `--dry-run`，在此终止

### Phase 1b — 冲突检查

- 如果预览报告有冲突且未指定 `--overwrite`，**拒绝执行**
- 提示用户用 `--overwrite` 重新运行

### Phase 2 — 执行（确认后）

1. 检查 OpenClaw 是否仍在运行（防止 token 冲突，如 Telegram 409 错误）
2. 检查 Hermes gateway 是否在运行且有活跃连接
3. **自动创建预迁移备份** — 调用 `hermes_cli.backup.create_pre_migration_backup()`，生成 `~/.hermes/backups/pre-migration-<timestamp>.zip`，自动保留最近 5 个，可通过 `hermes import` 恢复
4. 创建 `Migrator(execute=True)` 实例执行实际迁移
5. 输出迁移结果报告

---

## 迁移预设对比

### `user-data` 预设包括

| 类别 | 说明 |
|------|------|
| `soul` | 导入 `SOUL.md` 到 Hermes home |
| `workspace-agents` | 复制 workspace 指令文件（需指定 `--workspace-target`） |
| `memory` | 将 OpenClaw `MEMORY.md` 转换为 Hermes 记忆条目 |
| `user-profile` | 将 OpenClaw `USER.md` 转换为 Hermes 用户档案 |
| `messaging-settings` | 迁移 `TELEGRAM_ALLOWED_USERS` 等兼容设置 |
| `command-allowlist` | 合并 OpenClaw 命令审批模式 |
| `skills` | 复制技能到 `~/.hermes/skills/openclaw-imports/` |
| `tts-assets` | 镜像 `workspace/tts/` 等资产到 `~/.hermes/tts/` |
| `archive` | 归档没有直接 Hermes 目标的非秘密文档 |

### `full` 预设

`full` = `user-data` + `secret-settings`

> **注意：** 即使 `--preset full`，密钥也不会自动迁移。必须显式加 `--migrate-secrets` 才会导入 allowlisted 密钥（目前仅 `TELEGRAM_BOT_TOKEN`）。

---

## 安全守卫机制

源码中实现了多层安全保护：

1. **运行时检测** — 检测仍在运行的 OpenClaw 进程，以及 Hermes gateway 的活跃连接，提醒用户 Token 冲突风险
2. **预览先行** — 任何时候都要先预览再执行，除非指定 `-y`
3. **冲突阻断** — 有冲突未配 `--overwrite` 时拒绝执行，避免静默跳过
4. **自动备份** — 执行前自动创建完整的 Hermes home zip 快照，可恢复
5. **密钥安全** — 密钥必须**显式**通过 `--migrate-secrets` 指定，不会因 `--preset full` 意外泄露

---

## 典型用法

```bash
# 1. 先预览
hermes claw migrate --dry-run

# 2. 确认后执行（交互式确认）
hermes claw migrate

# 3. 静默执行（跳过确认）
hermes claw migrate --yes

# 4. 完整迁移（含密钥覆盖）
hermes claw migrate --preset full --overwrite --migrate-secrets --yes

# 5. 自定义源目录
hermes claw migrate --source /path/to/.openclaw --dry-run

# 6. 用户数据迁移 + 技能重命名 + 指定 workspace 目标
hermes claw migrate --preset user-data --skill-conflict rename --workspace-target /path/to/workspace

# 7. 迁移完成后清理旧目录
hermes claw cleanup
```

---

## 架构设计要点

- **CLI 层** (`hermes_cli/claw.py`) — 负责参数解析、运行检测、备份、冲突检查、用户交互
- **迁移引擎** (`optional-skills/migration/openclaw-migration/scripts/openclaw_to_hermes.py`) — 动态加载，负责实际的数据读取、转换、写入
- **备份层** (`hermes_cli/backup.py`) — 共享 `create_pre_migration_backup()` 实现，与 update 备份共用同一套 SQLite 安全拷贝和 zip 打包逻辑
- 迁移完成后**源目录保留不动**，用户可自行 `hermes claw cleanup` 来归档

---

## 相关命令

- `hermes claw cleanup` — 归档残留的 OpenClaw 目录
- `hermes import <archive>` — 恢复预迁移备份
- `hermes setup` — 首次设置时自动检测 `~/.openclaw` 并引导迁移