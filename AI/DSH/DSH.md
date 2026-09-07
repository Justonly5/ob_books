> https://deepseek-harness.github.io/deepseek-harness/guide/quickstart

https://github.com/deepseek-ai/deepseek-harness

Agent Sphere 

云端多用户，共享资源池


AGENT LOOP 

SASS



插件固化

技能

记忆
    短期记忆、长期记忆


存储  cfs  权限管理，预防越权


openSandbox

演示：
多人对话
同一个 LOOP


大脑引擎： 无状态、可扩展。



POC 可行性
组件清单、功能、接口。



## 安全沙箱
智能体需要执行代码、读取文件、安装依赖、访问网络等。

OpenSandBox-底层基于 Docker 
https://github.com/opensandbox-group/OpenSandbox
https://github.com/TencentCloud/CubeSandbox


E2B

Sandbox 的目标不是阻止 Agent 修改工作目录，而是把 Agent 的"能力边界"限制在工作目录及允许的资源范围内。
Agent 仍然可以修改挂载目录中的文件；Sandbox 主要限制的是 Agent “怎么执行”和“还能碰哪些东西”，而不是自动阻止它修改被授权挂载的目录。