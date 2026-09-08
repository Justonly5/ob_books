Skills 是 agent 在需要时可以加载的按需知识文档。

所有 skills 存放在 **`~/.hermes/skills/`** 中——这是主目录和唯一可信来源。全新安装时，捆绑的 skills 会从仓库复制过来。通过 Hub 安装和 agent 创建的 skills 也存放在此处。agent 可以修改或删除任何 skill。

Hermes 技能分为 3 类：
1. 内置技能（bundled）：镜像自带，安装时同步到 `~/.hermes/skills/`
2. 可选技能（Optional）： 镜像自带但默认不激活。
3. 用户技能（User/Custom）：用户自建或从外部安装，也是安装到 `~/.hermes/skills/`
