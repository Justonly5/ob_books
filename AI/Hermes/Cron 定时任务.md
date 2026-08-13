
## 创建定时任务

### 聊天中使用 /cron

```BASH
/cron add 30m "Remind me to check the build"
```

>>>>>  /cron is only available in the terminal interface.


### 独立的 cli

```BASH
hermes cron create "every 2h" "Check server status"
```

### 通过自然对话

直接向 Hermes 描述：
```
Every morning at 9am, check Hacker News for AI news and send me a summary on Telegram.
```

Hermes 会在内部使用统一的 `cronjob` 工具。

`hermes tools disable cronjob` 可以禁用 `cronjob` 工具。