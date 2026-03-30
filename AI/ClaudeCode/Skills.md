Skills 是 Claude Code 最轻量化的扩展方式，通过 Skills CLI 即可一键安装，类似前端常用的 npm 包管理器，开箱即用。

# 前置必看

技能安装与管理全命令

## 核心安装命令
```BASH
# 1. 搜索社区技能（关键词匹配）  
npx skills find <关键词>  
  
# 2. 安装技能（-y 跳过确认，-g 全局安装，必加！）  
npx skills add <owner/repo@skill> -y -g  
  
# 3. 查看已安装的全部技能  
npx skills list -g  
  
# 4. 检查技能更新  
npx skills check  
  
# 5. 更新所有已安装技能  
npx skills update
```

关键提醒：安装技能必须加 `-g` 全局安装参数，否则 Claude Code 无法识别；安装完成后**必须重启 Claude Code** 才能生效。