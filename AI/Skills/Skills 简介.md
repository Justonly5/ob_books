# Skills 是什么？
Skill 就是一个可复用的专家知识包：把一套判断逻辑、执行 SOP 以及必要工具打包成一个模块，这样 AI 就可以按照预设的方式来执行任务。
# Skills 和 Prompt、MCP 有什么区别？
Prompt 是一次性指令，MCP 是工具箱，Skills 是操作手册。
Prompt 通常是告诉 AI 干什么。
Skills 做的事情不一样。它告诉 AI 怎么判断、怎么决策、整个流程应该怎么走。
# Skills 长什么样？
接下来讲讲 Skills 具体长什么样，以及怎么用。

**先说结构。**

Skill 其实很简单，极简的 Skill 就是一个文件夹加一个 SKILL.md 文件。文件夹的名字就是这个 Skill 的名字，SKILL.md 里面写具体内容。

再复杂一点的 Skill，会包含附加的资源目录。比如有的 Skill 会用到 Python 脚本，会有参考的文档，这些都可以放进来。

所以更准确的说 Skill 的完整目录结构应该是这样：

```
~/.qoder/skills/  
├── pdf-processing/  
│   └── SKILL.md  
├── code-reviewer/  
│   └── SKILL.md  
└── ...
```

SKILL.md 分两部分。

第一部分是元数据，放在文件开头，用三个短横线包起来。里面有两个字段：name 是技能名称，description 是描述。

比如一个生成 mock 数据的 Skill，元数据可能长这样：

```
your-project/  
└── .qoder/  
    └── skills/  
        └── microservice-troubleshooter/  
            └── SKILL.md
```

这个 description 特别重要，它是 AI 判断什么时候该用这个 Skill 的依据。写得越清晰，AI 匹配得越准。

第二部分是正文，写具体的规则和流程。还是拿 api-mocker 举例：

```
# API Mock 数据生成器## 规则1. 分析用户提供的接口定义2. 根据字段名推断数据语义（name→姓名，email→邮箱）3. 生成 5 条真实感数据## 字段映射- name/姓名 → 中文姓名（王晓明、刘思远）- email → user_xxx@example.com- phone → 138-XXXX-XXXX- avatar → https://api.dicebear.com/7.x/avataaars/svg?seed=随机## 输出JSON 数组，用代码块包裹
```

看到没，这就是一套完整的 SOP。AI 拿到这个，就知道该怎么一步步执行，而不是每次都靠它自己发挥。

除了 SKILL.md，文件夹里还可以放附加资源。比如 scripts 目录放可执行的脚本，references 目录放 API 文档或规范说明，assets 目录放模板和图片。

正文里可以引用这些资源，比如写“执行到这一步时，先调用 scripts/check_health.sh 检查服务状态”。

为什么要分成三层？这里有个设计思想叫渐进式披露。

AI 的上下文窗口是有限的公共资源。像 MCP 工具、自定义命令这些，通常要常驻在上下文里，每次对话都会占用空间。

如果你配了很多工具，上下文很快就被挤满，token 消耗也跟着上去。

Skills 的分层设计不一样。

第一层，元数据，始终在上下文里。但它很轻量，就几行字，让 AI 知道有哪些技能可用、大概什么时候该用。

第二层，正文，只有触发时才加载。AI 判断当前任务需要用某个 Skill，才会去读 SKILL.md 的正文内容。

第三层，附加资源，按需加载。正文里如果写了"执行到这一步调用某个脚本"，AI 执行到那一步才会去读。

这就像一本书。目录始终在手边，让你知道有哪些章节。但你不会一开始就把整本书背下来，而是翻到哪章读哪章。这样你可以配很多 Skills，而不用担心上下文爆炸。

再说存储位置。

Skills 可以放两个地方。拿阿里的 Qoder 举例子。

一个是全局目录，在 ～/. qoder/skills/。放在这里的 Skill 所有项目都能用，适合那些通用能力，比如 PDF 处理、代码审查。目录结构像这样：

```
~/.qoder/skills/├── pdf-processing/│   └── SKILL.md├── code-reviewer/│   └── SKILL.md└── ...
```

另一个是项目目录，在项目根目录下的 .qoder/skills/。放在这里的 Skill 只有当前项目能用，适合那些跟特定项目强相关的能力。

比如这个项目的微服务架构知识、这个项目特有的排查流程。目录结构像这样：

```
your-project/└── .qoder/    └── skills/        └── microservice-troubleshooter/            └── SKILL.md
```

项目级的 Skills 有个好处：可以跟代码一起提交到仓库。团队成员 clone 下来就能用，不用再单独传文件、发文档。

新人入职第一天，项目的 Skills 就已经在他的工作环境里了。