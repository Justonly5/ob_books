## 前端适配
- 前端创建实例的时候选择创建的实例类型，是 openclaw 还是 hermes。
- 会话

## claw-admin 适配
admin-server 适配创建 hermes 实例。


## chatbot 适配

### chat
聊天接口。

### session
会话控制，涉及以下两个接口：
- list：查询会话列表
- get：

### model
模型控制相关，涉及以下两个接口：
- list：查询模型列表
- set：设置模型

## charts 模版编写
涉及 deployment、service、configMap、secret（如果需要）、networkPolicy、ingress 等资源模版文件编写。


## Anget 元数据

### SOUL.MD

### AGENT.MD

>一个实用的判断规则：
>  - 如果它应该随你到处适用，属于 `SOUL.md`
   - 如果它属于某个项目，属于 `AGENTS.md`
  

### config.yaml



## 技能



## workspace


## 持久化目录

/opt/data 


## 文件上传目录
HERMES_DASHBOARD_FILES_ROOT 用于配置 Dashboard 的文件上传/浏览就锁定在那个目录下。


## 敏感数据加密解密



## openAI 协议接口

https://hermes-agent.nousresearch.com/docs/zh-Hans/user-guide/features/api-server#post-apijobsjob_idrun

## Hooks
https://hermes-agent.nousresearch.com/docs/zh-Hans/user-guide/features/hooks#plugin-hooks

## langfuse
### 日志问题
`HERMES_LANGFUSE_DEBUG=true`（环境变量开关） 开启 langfuse 上报打印日志，会打印一条 INFO 的日志到  agent.log 文件里。

## 监控

| Metric           | 用途                    |
| ---------------- | --------------------- |
| Conversation 数   | 每天 Agent 使用量          |
| Skill 调用次数       | 哪些 Skill 最热门          |
| Skill 成功率        | 哪些 Skill 最容易失败        |
| Tool 耗时 P95      | 哪些外部依赖是瓶颈             |
| Token 消耗         | 成本分析                  |
| Cost             | 每用户 / 每 Agent / 每模型成本 |
| Memory Hit Rate  | 长期记忆是否真正发挥作用          |
| Planner Retry    | Planner 是否经常走弯路       |
| Tool Retry       | 外部工具稳定性               |
| Agent Loop Count | 一个任务平均循环多少轮           |



