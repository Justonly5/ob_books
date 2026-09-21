https://datawhalechina.github.io/deepagents-in-action/

[Hello-Agents]([Hello-Agents](https://hello-agents.datawhale.cc/))

# 什么是智能体

AGENT = LLM + MEMORY + 规划 + 工具

Agent

						    Prompt
	external knowledge <-->   LLM    <-->  memory
                            Tools

reAct
planAndExecute

## Agent Loop
智能体并非一次性完成任务，而是通过一个持续的循环与环境进行交互，这个核心机制被称为 **智能体循环 (Agent Loop)**。

Harness Engineering
Harness 驱动和分析智能体的“执行脚手架”，它决定模型合适被调用、调用什么工具、如何评估结果、何时停下。

提示词工程 --> 解决模型听不懂                 模型不是万能的，
上下文工程 --> 解决大模型看到什么东西   提示词爆炸💥
    skill 渐进式披露
Harness     -->  
     工具治理、编排、验证&评估、治理&安全

Agent 的一些问题：
总是想一步到位，在一个会话里解决所有问题
过早的宣布胜利
过分依赖自己已有的经验