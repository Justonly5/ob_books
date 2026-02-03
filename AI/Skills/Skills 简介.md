# Skills 是什么？
Skill 就是一个可复用的专家知识包：把一套判断逻辑、执行 SOP 以及必要工具打包成一个模块，这样 AI 就可以按照预设的方式来执行任务。
# Skills 和 Prompt、MCP 有什么区别？
Prompt 是一次性指令，MCP 是工具箱，Skills 是操作手册。
Prompt 通常是告诉 AI 干什么。
Skills 做的事情不一样。它告诉 AI 怎么判断、怎么决策、整个流程应该怎么走。
# Skills 长什么样？
接下来讲讲 Skills 具体长什么样，以及怎么用。

## **先说结构。**

Skill 其实很简单，极简的 Skill 就是一个文件夹加一个 SKILL.md 文件。文件夹的名字就是这个 Skill 的名字，SKILL.md 里面写具体内容。

再复杂一点的 Skill，会包含附加的资源目录。比如有的 Skill 会用到 Python 脚本，会有参考的文档，这些都可以放进来。

所以更准确的说 Skill 的完整目录结构应该是这样：

```markdown
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

```markdown
---  
name: api-mocker  
description: |  
  根据 API 定义生成 Mock 测试数据。  
  当用户说"生成mock数据"、"造测试数据"时使用。  
---
```

这个 description 特别重要，它是 AI 判断什么时候该用这个 Skill 的依据。写得越清晰，AI 匹配得越准。

第二部分是正文，写具体的规则和流程。还是拿 api-mocker 举例：

```markdown
# API Mock 数据生成器  
  
## 规则  
1. 分析用户提供的接口定义  
2. 根据字段名推断数据语义（name→姓名，email→邮箱）  
3. 生成 5 条真实感数据  
  
## 字段映射  
- name/姓名 → 中文姓名（王晓明、刘思远）  
- email → user_xxx@example.com  
- phone → 138-XXXX-XXXX  
- avatar → https://api.dicebear.com/7.x/avataaars/svg?seed=随机  
  
## 输出  
JSON 数组，用代码块包裹
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
~/.qoder/skills/  
├── pdf-processing/  
│   └── SKILL.md  
├── code-reviewer/  
│   └── SKILL.md  
└── ...
```

另一个是项目目录，在项目根目录下的 .qoder/skills/。放在这里的 Skill 只有当前项目能用，适合那些跟特定项目强相关的能力。

比如这个项目的微服务架构知识、这个项目特有的排查流程。目录结构像这样：

```
your-project/  
└── .qoder/  
    └── skills/  
        └── microservice-troubleshooter/  
            └── SKILL.md
```

项目级的 Skills 有个好处：可以跟代码一起提交到仓库。团队成员 clone 下来就能用，不用再单独传文件、发文档。

新人入职第一天，项目的 Skills 就已经在他的工作环境里了。

## 怎么触发。


两种方式。

第一种是自动触发。你正常描述需求，AI 会根据 description 自动匹配。

比如你说“帮我查看这个 PDF 的内容”，它会自动找到 pdf-processing 这个 Skill；你说“帮我造点测试数据”，它会匹配到 api-mocker。

这就是为什么 description 要写清楚，它决定了 AI 能不能在对的时机调用对的技能。

第二种是手动调用。用斜杠加技能名，比如 /pdf-processing，强制使用某个 Skill。

或者在对话里明确说“用 pdf-processing skill 帮我处理这个文件”。当你明确知道要用哪个技能时，手动调用更直接。

想看当前有哪些 Skill 可用，输入 /skills 就能看到完整列表。

它会分开展示用户级和项目级的 Skills，还能看到每个 Skill 的描述。选中某个 Skill 回车，还能预览里面的具体内容。

# 在 Qoder 中实操 Skills

如前面所说，Skill 可以装到两个地方。一个是自己的项目目录下，这时候只对当前项目生效。一个是用户目录下，这会对所有的项目生效。

现在网上有很多别人做好的 Skills。比如下面这个链接，是 Anthropic 分享的一些好用的 Skill。

https://github.com/anthropics/skills/blob/main/skills/skill-creator/SKILL.md

我们可以把这些 Skill 克隆下来，放到自己的文件夹下。比如下面的截图，我它放到了当前项目下。

![[Pasted image 20260128101108.png]]

咱们随便进入一个 Skill 的文件夹来看看，这和我们前面提到的结构一模一样。
![[Pasted image 20260128101124.png]]

紧接着，我们再在 IDE 里验证下这些 Skill 是不是已经装好了。

复制完文件之后，大家记得要重启下 IDE，这样才能检测到。如下图，右侧的列表，代表已经装上了。

![[Pasted image 20260128101137.png]]

现在这些 Skills 装好了之后，就可以像前面那样运行了。

接下来我想说说 Skills 怎么创建。

Skill 的核心是刚刚反复提到的 md 文件。这个 md 文件，如果纯手写，多少还是有些复杂的。所以官方也提供了一个 Skill 来帮我们创建 Skill。

我现在使用的流程是，第一，先自己梳理一份自己的流程。用自然语言写就行。

第二，基于这个流程，让 skill-creator 来生成一个最终的 SKILL。

第三，在这个初稿的基础上，基于自己的经验手动修改。

下面我先根据自己的理解梳理了一个阿颖文风的 md 文档，然后告诉它，基于这个文档，用 skill-creator 创建一个 Skill。

可以看到，它已经调用了对应的 Skill。

![[Pasted image 20260128101151.png]]

等大概不到一分钟，这个 Skill 就能创建好：

![[Pasted image 20260128101202.png]]

其实很多场景下，第一步都不用自己手动梳理。可以先让 AI 去 Research，拿到结果后再让它生成 Skill 的初稿。

以上，就是整体的流程。


