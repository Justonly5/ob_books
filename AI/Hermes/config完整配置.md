---
title: Hermes config.yaml 完整配置参考
tags: [hermes, config, reference]
created: 2026-06-30
---

# Hermes config.yaml 完整配置参考

> 来源：Hermes Agent 源码 `hermes_cli/config.py` 中的 `DEFAULT_CONFIG`
> 版本：_config_version 31

---

## 一、模型配置

```yaml
model: ""                          # 默认模型名，如 "anthropic/claude-sonnet-4.6"
providers: {}                      # 旧版 provider 配置（已废弃，用 custom_providers）
fallback_providers: []             # 备用 provider 链
credential_pool_strategies: {}     # 凭据池轮转策略

custom_providers:
  - name: my-aliyun                # 自定义 provider 名称
    base_url: https://.../v1       # API 端点
    api_key: ${MY_KEY}             # API Key（支持 ${VAR} 引用）
    api_mode: chat_completions     # 协议：chat_completions | codex_responses | anthropic_messages
```

---

## 二、Agent 行为

```yaml
agent:
  max_turns: 90                          # 每轮最大工具调用次数
  gateway_timeout: 1800                  # Gateway 执行超时（秒）
  restart_drain_timeout: 0               # Gateway 重启优雅关闭等待（秒）
  api_max_retries: 3                     # API 调用最大重试次数
  service_tier: ""                       # 服务等级
  tool_use_enforcement: "auto"           # 工具使用强制：auto | true | false | [模型名列表]
  intent_ack_continuation: "auto"        # 意图确认续推
  task_completion_guidance: True         # 完成任务引导
  parallel_tool_call_guidance: True      # 并行工具调用引导
  environment_probe: True                # 本地环境探测
  environment_hint: ""                   # 环境提示文本
  coding_context: "auto"                 # 编码上下文：auto | focus | on | off
  verify_on_stop: False                  # 停止前验证
  gateway_timeout_warning: 900           # 超时前警告（秒）
  clarify_timeout: 3600                  # 澄清等待超时（秒）
  gateway_notify_interval: 180           # 工作中通知间隔（秒）
  gateway_auto_continue_freshness: 3600  # 自动继续新鲜度窗口（秒）
  image_input_mode: "auto"               # 图片输入模式：auto | native | text
  disabled_toolsets: []                  # 禁用的工具集
  save_trajectories: false               # 是否保存轨迹（默认关闭）
```

---

## 三、终端

```yaml
terminal:
  backend: "local"                       # 终端后端：local | docker | ssh | modal | daytona
  modal_mode: "auto"
  cwd: "."                               # 工作目录
  timeout: 180                           # 命令超时（秒）
  daemon_term_grace_seconds: 2.0         # 守护进程终止宽限期
  env_passthrough: []                    # 透传的环境变量
  home_mode: "auto"                      # HOME 处理模式
  shell_init_files: []                   # Shell 初始化文件
  auto_source_bashrc: True               # 自动 source bashrc
  docker_image: "nikolaik/python-nodejs:python3.11-nodejs20"
  docker_forward_env: []
  docker_env: {}
  docker_volumes: []
  docker_mount_cwd_to_workspace: False
  docker_extra_args: []
  docker_run_as_host_user: False
  container_cpu: 1
  container_memory: 5120                 # MB
  container_disk: 51200                  # MB
  container_persistent: True
  persistent_shell: True                 # 持久化 shell
```

---

## 四、Web 搜索

```yaml
web:
  backend: ""                            # 共享回退后端
  search_backend: ""                     # 搜索后端（如 searxng）
  extract_backend: ""                    # 提取后端（如 native）
```

---

## 五、浏览器

```yaml
browser:
  inactivity_timeout: 120                # 不活动超时（秒）
  command_timeout: 30                    # 命令超时（秒）
  record_sessions: False                 # 录制浏览器会话
  allow_private_urls: False              # 允许访问内网 URL
  engine: "auto"                         # 引擎：auto | lightpanda | chrome
  auto_local_for_private_urls: True      # 内网 URL 自动使用本地浏览器
  cdp_url: ""                            # CDP 端点 URL
  dialog_policy: "must_respond"          # 对话框策略
  dialog_timeout_s: 300
  camofox:
    managed_persistence: False
    user_id: ""
    session_key: ""
    adopt_existing_tab: False
    rewrite_loopback_urls: False
    loopback_host_alias: "host.docker.internal"
```

---

## 六、检查点（文件快照）

```yaml
checkpoints:
  enabled: False
  max_snapshots: 20
  max_total_size_mb: 500
  max_file_size_mb: 10
  auto_prune: True
  retention_days: 7
  delete_orphans: True
  min_interval_hours: 24
```

---

## 七、上下文压缩

```yaml
compression:
  enabled: True
  threshold: 0.50                       # 触发压缩的上下文占比
  target_ratio: 0.20                    # 压缩目标比例
  protect_last_n: 20                    # 保留最近 N 条消息
  hygiene_hard_message_limit: 5000      # 强制压缩的消息数阈值
  protect_first_n: 3                    # 保留前 N 条消息
  abort_on_summary_failure: False
  codex_gpt55_autoraise: True
  in_place: False                       # 原地压缩（不轮转 session id）
```

---

## 八、辅助模型（Auxiliary）

```yaml
auxiliary:
  vision:
    provider: "auto"
    model: ""                           # 如 google/gemini-2.5-flash
    base_url: ""
    api_key: ""
    timeout: 120
    extra_body: {}
    download_timeout: 30
  web_extract:
    provider: "auto"
    model: ""
    timeout: 360
  compression:
    provider: "auto"
    model: ""
    timeout: 120
  skills_hub:
    provider: "auto"
    timeout: 30
  approval:
    provider: "auto"
    model: ""
    timeout: 30
  mcp:
    provider: "auto"
    timeout: 30
  title_generation:
    provider: "auto"
    timeout: 30
    language: ""
  tts_audio_tags:
    provider: "auto"
    timeout: 30
  triage_specifier:
    provider: "auto"
    timeout: 120
  kanban_decomposer:
    provider: "auto"
    timeout: 180
  profile_describer:
    provider: "auto"
    timeout: 60
  curator:
    provider: "auto"
    timeout: 600
  monitor:
    provider: "auto"
    timeout: 60
  background_review:
    provider: "auto"
    timeout: 120
  moa_reference:
    provider: "auto"
    timeout: 600
  moa_aggregator:
    provider: "auto"
    timeout: 600
```

---

## 九、显示

```yaml
display:
  compact: False
  personality: ""
  interface: "cli"                       # cli | tui
  tui_auto_resume_recent: False
  tui_agents_nudge: True
  streaming: False
  show_reasoning: False
  reasoning_full: False
  reasoning_style: "code"                # code | blockquote | subtext
  memory_notifications: "on"             # off | on | verbose
  timestamps: False
  final_response_markdown: "strip"       # render | strip | raw
  persistent_output: True
  persistent_output_max_lines: 200
  inline_diffs: True
  show_cost: False
  skin: "default"
  language: "en"                         # en | zh | ja | de | es | fr | tr | uk
  bell_on_complete: False
  busy_input_mode: "interrupt"           # interrupt | queue | steer
  cli_refresh_interval: 1.0
  copy_shortcut: "auto"
  platforms:
    telegram:
      streaming: True
    discord:
      streaming: False
  runtime_footer:
    enabled: False
    fields: ["model", "context_pct", "cwd"]
  pet:
    enabled: False
    slug: ""
    render_mode: "auto"
    scale: 0.33
```

---

## 十、Dashboard

```yaml
dashboard:
  theme: "default"                       # default | midnight | ember | mono | cyberpunk | rose
  show_token_analytics: False
  oauth:
    client_id: ""
    portal_url: ""
  basic_auth:
    username: ""
    password_hash: ""
    password: ""
    secret: ""
    session_ttl_seconds: 0
  drain_auth:
    scope: "drain"
    min_secret_chars: 43
  public_url: ""
```

---

## 十一、隐私

```yaml
privacy:
  redact_pii: False                      # 脱敏个人身份信息
```

---

## 十二、TTS（文字转语音）

```yaml
tts:
  provider: "edge"                       # edge | elevenlabs | openai | xai | minimax | mistral | gemini | neutts | kittentts | piper
  edge:
    voice: "en-US-AriaNeural"
  elevenlabs:
    voice_id: "pNInz6obpgDQGcFmaJgB"
    model_id: "eleven_multilingual_v2"
  openai:
    model: "gpt-4o-mini-tts"
    voice: "alloy"
  gemini:
    model: "gemini-2.5-flash-preview-tts"
    voice: "Kore"
    audio_tags: False
    persona_prompt_file: ""
  xai:
    voice_id: "eve"
  mistral:
    model: "voxtral-mini-tts-2603"
  neutts:
    ref_audio: ""
    model: "neuphonic/neutts-air-q4-gguf"
    device: "cpu"
  piper:
    voice: "en_US-lessac-medium"
```

---

## 十三、STT（语音转文字）

```yaml
stt:
  enabled: True
  provider: "local"                      # local | groq | openai | mistral | elevenlabs
  local:
    model: "base"
    language: ""
  openai:
    model: "whisper-1"
  mistral:
    model: "voxtral-mini-latest"
  elevenlabs:
    model_id: "scribe_v2"
```

---

## 十四、语音

```yaml
voice:
  record_key: "ctrl+b"
  max_recording_seconds: 120
  auto_tts: False
  beep_enabled: True
  silence_threshold: 200
  silence_duration: 3.0
```

---

## 十五、上下文引擎

```yaml
context:
  engine: "compressor"                   # compressor | 插件名
```

---

## 十六、记忆

```yaml
memory:
  memory_enabled: True
  user_profile_enabled: True
  write_approval: False                  # 写入审批
  memory_char_limit: 2200
  user_char_limit: 1375
  provider: ""                           # 外部记忆提供者：openviking | mem0 | hindsight | holographic | retaindb | byterover
```

---

## 十七、子代理（Delegation）

```yaml
delegation:
  model: ""
  provider: ""
  base_url: ""
  api_key: ""
  api_mode: ""
  inherit_mcp_toolsets: True
  max_iterations: 50
  child_timeout_seconds: 0
  reasoning_effort: ""
  max_concurrent_children: 3
  max_async_children: 3
  max_spawn_depth: 1
  orchestrator_enabled: True
  subagent_auto_approve: False
```

---

## 十八、技能

```yaml
skills:
  external_dirs: []                      # 外部技能目录
  template_vars: True
  inline_shell: False                    # 允许 SKILL.md 中内联 shell 命令
  inline_shell_timeout: 10
  guard_agent_created: False             # 安全扫描 agent 创建的技能
  write_approval: False                  # 技能写入审批
```

---

## 十九、Curator（技能自动管理）

```yaml
curator:
  enabled: True
  interval_hours: 168                    # 7天
  min_idle_hours: 2
  stale_after_days: 30
  archive_after_days: 90
  consolidate: False                     # LLM 合并技能
  prune_builtins: True                   # 可归档内置技能
  backup:
    enabled: True
    keep: 5
```

---

## 二十、审批

```yaml
approvals:
  mode: "manual"                         # manual | smart | off
  timeout: 60
  cron_mode: "deny"                      # deny | approve
  mcp_reload_confirm: True
  destructive_slash_confirm: True

command_allowlist: []                    # 永久允许的危险命令
quick_commands: {}                       # 用户快速命令
```

---

## 二十一、平台设置

### Slack

```yaml
slack:
  require_mention: True
  free_response_channels: ""
  allowed_channels: ""
  channel_prompts: {}
```

### Discord

```yaml
discord:
  require_mention: True
  free_response_channels: ""
  allowed_channels: ""
  auto_thread: True
  thread_require_mention: False
  history_backfill: True
  history_backfill_limit: 50
  reactions: True
  channel_prompts: {}
  dm_role_auth_guild: ""
  server_actions: ""
  max_attachment_bytes: 33554432
  voice_fx:
    enabled: False
    ambient_enabled: True
    ambient_gain: 0.18
    speech_gain: 1.0
    ack_enabled: True
```

### Telegram

```yaml
telegram:
  reactions: False
  channel_prompts: {}
  allowed_chats: ""
  extra:
    rich_messages: False
    rich_drafts: False
```

### WhatsApp / Mattermost / Matrix

```yaml
whatsapp: {}

mattermost:
  require_mention: True
  free_response_channels: ""
  allowed_channels: ""
  channel_prompts: {}

matrix:
  require_mention: True
  free_response_rooms: ""
  allowed_rooms: ""
```

---

## 二十二、Gateway

```yaml
gateway:
  scale_to_zero:
    idle_timeout_minutes: 5
  message_timestamps:
    enabled: False
  max_inbound_media_bytes: 134217728     # 128 MiB
  strict: False                          # 媒体投递严格模式
  media_delivery_allow_dirs: []
  trust_recent_files: True
  trust_recent_files_seconds: 600
  api_server:
    max_concurrent_runs: 10
```

---

## 二十三、流式输出

```yaml
streaming:
  enabled: False
  transport: "auto"                      # auto | draft | edit | off
  edit_interval: 0.8
  buffer_threshold: 24
  cursor: " ▉"
  fresh_final_after_seconds: 0.0
```

---

## 二十四、会话管理

```yaml
sessions:
  auto_prune: False                      # 自动清理旧会话
  retention_days: 90
  vacuum_after_prune: True
  min_interval_hours: 24
  write_json_snapshots: False            # 写入 JSON 快照文件
```

---

## 二十五、日志

```yaml
logging:
  level: "INFO"                          # DEBUG | INFO | WARNING
  max_size_mb: 5                         # 单文件上限
  backup_count: 3                        # 备份数
```

---

## 二十六、Cron

```yaml
cron:
  provider: ""                           # 调度器提供者
  wrap_response: True
  mirror_delivery: False
  max_parallel_jobs: null
  output_retention: 50
```

---

## 二十七、Kanban

```yaml
kanban:
  dispatch_in_gateway: True
  dispatch_interval_seconds: 60
  failure_limit: 2
  worker_log_rotate_bytes: 2097152
  worker_log_backup_count: 1
  orchestrator_profile: ""
  default_assignee: ""
  max_in_progress_per_profile: null
  auto_decompose: True
  auto_decompose_per_tick: 3
  dispatch_stale_timeout_seconds: 14400
```

---

## 二十八、安全

```yaml
security:
  allow_private_urls: False
  redact_secrets: True
  tirith_enabled: True
  tirith_path: "tirith"
  tirith_timeout: 5
  tirith_fail_open: True
  website_blocklist:
    enabled: False
    domains: []
    shared_files: []
  acked_advisories: []
  allow_lazy_installs: True
```

---

## 二十九、外部 Secret 源

```yaml
secrets:
  bitwarden:
    enabled: False
    access_token_env: "BWS_ACCESS_TOKEN"
    project_id: ""
    cache_ttl_seconds: 300
    override_existing: True
    auto_install: True
    server_url: ""
```

---

## 三十、其他配置

```yaml
# 网络
network:
  force_ipv4: False

# 工具输出截断
tool_output:
  max_bytes: 50000
  max_lines: 2000
  max_line_length: 2000

# 工具循环护栏
tool_loop_guardrails:
  warnings_enabled: True
  hard_stop_enabled: False
  warn_after:
    exact_failure: 2
    same_tool_failure: 3
    idempotent_no_progress: 2
  hard_stop_after:
    exact_failure: 5
    same_tool_failure: 8
    idempotent_no_progress: 5

# 工具搜索（渐进式工具暴露）
tools:
  tool_search:
    enabled: "auto"                      # auto | on | off
    threshold_pct: 10
    search_default_limit: 5
    max_search_limit: 20

# 代码执行
code_execution:
  mode: "project"                        # project | strict

# LSP 语言服务器
lsp:
  enabled: True
  wait_mode: "document"
  wait_timeout: 5.0
  install_strategy: "auto"
  servers: {}

# OpenRouter
openrouter:
  response_cache: True
  response_cache_ttl: 300
  min_coding_score: 0.65

# Bedrock
bedrock:
  region: ""
  discovery:
    enabled: True
    provider_filter: []
    refresh_interval: 3600
  guardrail:
    guardrail_identifier: ""
    guardrail_version: ""
    stream_processing_mode: "async"
    trace: "disabled"

# 提示缓存
prompt_caching:
  cache_ttl: "5m"

# 模型目录
model_catalog:
  enabled: True
  url: "https://hermes-agent.nousresearch.com/docs/api/model-catalog.json"
  ttl_hours: 1
  providers: {}

# 目标系统
goals:
  max_turns: 20

# MOA（模型混合）
moa:
  default_preset: "default"
  active_preset: ""
  presets:
    default:
      reference_models:
        - provider: openai-codex
          model: gpt-5.5
        - provider: openrouter
          model: deepseek/deepseek-v4-pro
      aggregator:
        provider: openrouter
        model: anthropic/claude-opus-4.8
      reference_temperature: 0.6
      aggregator_temperature: 0.4
      max_tokens: 4096
      enabled: True

# 其他
timezone: ""
max_concurrent_sessions: null
max_live_sessions: 16
context_file_max_chars: null
file_read_max_chars: 100000
mcp_discovery_timeout: 1.5
prefill_messages_file: ""
platform_hints: {}
hooks: {}
hooks_auto_accept: False
personalities: {}
human_delay:
  mode: "off"
  min_ms: 800
  max_ms: 2500
onboarding:
  seen: {}
  profile_build: "ask"
updates:
  pre_update_backup: False
  backup_keep: 5
  non_interactive_local_changes: "stash"
x_search:
  model: "grok-4.20-reasoning"
  timeout_seconds: 180
  retries: 2
computer_use:
  cua_telemetry: False
desktop:
  electron_flags: []
  disable_gpu: "auto"
paste_collapse_threshold: 5
paste_collapse_threshold_fallback: 5
paste_collapse_char_threshold: 2000
_config_version: 31
```

---

## 快速参考：常用配置模板

```yaml
# 最简配置
model:
  provider: custom
  default: qwen3.5-plus
  base_url: https://dashscope.aliyuncs.com/compatible-mode/v1
  api_key: ${DASHSCOPE_API_KEY}

# 日志配置
logging:
  level: INFO
  max_size_mb: 10
  backup_count: 7

# 终端后端（Docker）
terminal:
  backend: docker
  docker_image: python:3.11-slim
  container_memory: 8192

# 辅助模型（省钱）
auxiliary:
  vision:
    provider: openrouter
    model: google/gemini-2.5-flash

# 子代理
delegation:
  model: google/gemini-3-flash-preview
  provider: openrouter
  max_concurrent_children: 5
```
