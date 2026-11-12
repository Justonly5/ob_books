
## 一、先用一张表理解它们

|命令|所属生态|核心作用|类比|
|---|---|---|---|
|`pip`|Python|安装、管理 Python 包|Python 传统包安装器|
|`uv`|Python|管理依赖、虚拟环境、Python 版本及项目|更现代的 Python 项目管理工具|
|`uvx`|Python|临时运行 Python 命令行工具|类似 `npx`|
|`npm`|Node.js / TypeScript|安装依赖、管理项目、运行脚本|Node.js 默认包管理器|
|`pnpm`|Node.js / TypeScript|管理依赖和项目，节省磁盘空间|npm 的替代方案|

这里有一个关键点：`uv` 和 `pip` 并不是完全平级的工具。`uv` 能覆盖很多传统 `pip` 的使用场景，但还额外管理虚拟环境、项目依赖等。

同样，`pnpm` 和 `npm` 都属于 Node.js 生态，可以完成大部分相同的项目依赖管理任务。

## 二、Python 生态：pip、uv、uvx

### 1. pip：传统的 Python 包管理器
`pip` 的核心职责是从 Python 包仓库（通常是 PyPI）安装、升级和卸载 Python 包。
例如，你要写一个调用 HTTP 接口的 Python 程序，需要安装 `requests`：
```bash
pip install requests
pip show requests
pip list
pip uninstall requests
```

常见项目通常会用 `requirements.txt` 记录依赖：
```
requests==2.32.0
flask==3.1.0
```

然后执行：
```bash
pip install -r requirements.txt
```

注意：`pip` 通常与某个 Python 解释器或虚拟环境绑定。为了避免 Mac 上存在多个 Python 时装错环境，更推荐：

```bash
python -m pip install requests
```

### 2. uv：现代化的 Python 项目管理工具
`uv` 由 Astral 团队开发，使用 Rust 编写。它不仅能安装 Python 包，还能管理 Python 版本、虚拟环境、项目依赖和锁文件等。可以从三个层次理解它：
第一层：包管理。相当于 pip 的常见功能

```
uv pip install requests
uv pip list
```

第二层：项目管理。管理项目依赖、虚拟环境和锁文件

```bash
uv init my-project
cd my-project
uv add requests
uv run python main.py
```
第三层：Python 版本管理。可以下载和管理多个 Python 版本

```
uv python list
uv python install 3.12
```

这里最值得注意的是 `uv add` 与 `uv pip install` 的区别：
- `uv add requests`：将依赖加入项目配置，并更新项目锁文件和环境。
- `uv pip install requests`：直接向当前选定的 Python 环境安装包，更接近传统 `pip` 的工作方式。
如果是新建 Python 项目，通常优先使用 `uv init`、`uv add`、`uv run` 这一套项目管理命令。
### 3. uvx：临时运行 Python 工具
`uvx` 是理解起来最容易混淆的一个。
它不是另一个独立的包管理器，而是 `uv tool run` 的快捷命令。它可以在隔离的工具环境中运行 Python 命令行程序，不需要先把工具安装到当前项目中。
例如，你想使用 Python 代码格式化工具 `ruff`：
```
uvx ruff check .
```
或者使用代码格式化功能：
```
uvx ruff format .
```
通常不需要提前执行：
```
pip install ruff
```

如果经常使用这个工具，也可以选择安装到用户级工具环境：
```
uv tool install ruff
```
核心区别：

| 命令                     | 主要用途                   |
| ---------------------- | ---------------------- |
| `uv add ruff`          | 把 ruff 添加为当前项目的依赖      |
| `uv pip install ruff`  | 把 ruff 安装到当前 Python 环境 |
| `uvx ruff check .`     | 直接运行 ruff，不要求它成为当前项目依赖 |
| `uv tool install ruff` | 安装 ruff，供日常反复使用        |
这里的“临时”不代表每次都重新下载所有文件。`uv` 会复用缓存；运行时使用的工具环境也与项目自己的依赖环境相分离。

## 三、Node.js / TypeScript 生态：npm、pnpm
### 1. npm：Node.js 默认的包管理器
`npm` 是 Node.js 生态中最基础、最常见的包管理工具。通常安装 Node.js 时也会一并获得 npm。它既能安装依赖，也能运行 `package.json` 中定义的脚本。
例如，你正在开发一个 TypeScript 项目：
```
# 创建项目目录
mkdir my-project
cd my-project

# 初始化 package.json
npm init -y

# 安装运行时依赖
npm install express

# 安装开发依赖
npm install -D typescript

# 安装项目内声明的所有依赖
npm install

# 运行 package.json 中定义的脚本
npm run build
```

几个重要概念：
- `package.json`：记录项目元信息、依赖和脚本，类似项目的依赖声明文件。
- `package-lock.json`：锁定依赖解析结果，帮助不同机器安装一致的依赖版本。
- `node_modules/`：实际安装的包所在的目录。
`npm` 还有一个很重要的功能：`npx`。
```
npx tsc --version
```
`npx` 可以运行 npm 包提供的命令行工具；必要时可以获取对应的包。因此，`npx` 在 Node.js 生态中的定位与 Python 的 `uvx` 比较接近。

### 2. pnpm：npm 的替代方案
`pnpm` 也是 Node.js / TypeScript 生态中的包管理器，和 npm 一样，能够管理项目依赖、执行脚本以及安装命令行工具。
例如，使用 pnpm 实现上面的操作：
```
mkdir my-project
cd my-project

pnpm init

pnpm add express
pnpm add -D typescript

pnpm install
pnpm run build
```

它和 npm 的主要区别在于依赖存储方式。
npm：传统的依赖安装方式。不同项目可能各自存储相同依赖的文件。
pnpm：集中存储，链接到项目。相同内容可以复用，减少磁盘占用；pnpm 还采用更严格的依赖访问布局。
pnpm 的核心优势是通过内容寻址存储和链接机制复用依赖文件，并且通常能更高效地管理多项目或 Monorepo。
### 3. npm 与 pnpm 的命令对照

|操作|npm|pnpm|
|---|---|---|
|初始化项目|`npm init -y`|`pnpm init`|
|安装全部依赖|`npm install`|`pnpm install`|
|安装运行时依赖|`npm install axios`|`pnpm add axios`|
|安装开发依赖|`npm install -D typescript`|`pnpm add -D typescript`|
|卸载依赖|`npm uninstall axios`|`pnpm remove axios`|
|运行项目脚本|`npm run build`|`pnpm run build`|
|全局安装工具|`npm install -g xxx`|`pnpm add -g xxx`|
|临时运行工具|`npx xxx`|`pnpm dlx xxx`|

例如，想临时执行一个脚手架工具：
```
# npm
npx create-vite my-app

# pnpm
pnpm dlx create-vite my-app
```
`pnpm dlx` 和 `npx` 的定位，与 `uvx` 都比较接近：获取并运行命令行工具，而不是把它作为当前项目的常规依赖。
注意，`npm` 和 `pnpm` 的命令参数并非所有情况下都完全兼容，切换包管理器时应以各自文档为准。
## 四、把五个命令放在一起比较
最重要的是分清它们属于哪套生态，以及它们分别管理什么。

| 对比项          | `pip`      | `uv`      | `uvx`        | `npm`               | `pnpm`           |
| ------------ | ---------- | --------- | ------------ | ------------------- | ---------------- |
| 生态           | Python     | Python    | Python       | Node.js             | Node.js          |
| 安装项目依赖       | 是          | 是         | 不是主要用途       | 是                   | 是                |
| 管理项目配置与锁文件   | 有限         | 是         | 否            | 是                   | 是                |
| 管理 Python 版本 | 否          | 是         | 通过 uv 能力间接支持 | 不适用                 | 不适用              |
| 临时运行 CLI 工具  | 不方便        | 支持        | 核心用途         | 通过 npx              | 通过 dlx           |
| 常见锁文件        | 无固定项目锁文件机制 | `uv.lock` | 不负责项目锁文件     | `package-lock.json` | `pnpm-lock.yaml` |
