# Agent 学习仓 — 技术解析文档

8 份文档，全部基于本地快照的**真实源码**编写：每个类名、函数、路径都能在对应 commit 上复核。标注【待核】的位置表示本地快照里查不到（例如 GitHub star 数），不猜。

## 怎么读这份东西

代码本体不在这份仓库里（主仓库只有引用指针）。这边是「地图 + 说明书」：先读每篇的 **核心架构** 与 **关键抽象** 两节建立结构感，再照 **阅读路径** 一节的顺序去代码库逐个精读。

路径格式是 `路径:行号`。行号会漂移，所以每条都**同时给了附近的函数或类名**——搜名字比搜行号稳。

## 推荐阅读顺序

| # | 项目 | 主要学什么 | 代码规模 | 难度 |
| --- | --- | --- | --- | --- |
| 1 | [openai-agents-python](./openai-agents-python.md) | Agent 最小循环长什么样：一个 `while True` 跑完一切 | 476k 行 | 低 |
| 2 | [langgraph](./langgraph.md) | 编排层范式：BSP 超步 + 通道版本号 + checkpointer | 204k 行 | 中 |
| 3 | [LightRAG](./LightRAG.md) | RAG 工程落地：级联队列、四种存储抽象、六种检索模式 | 443k 行 | 中 |
| 4 | [gpt-researcher](./gpt-researcher.md) | 端到端应用怎么组织：编排者、并发、上下文压缩 | 51k 行 | 中 |
| 5 | [storm](./storm.md) | 多阶段流水线 + DSPy 写法，前 LangChain 时代的代表作 | 14k 行 | 中 |
| 6 | [ChatDev](./ChatDev.md) | 多角色协作。注意本地是 2.0（DevAll），论文的 ChatChain 已不存在 | 32k 行 | 高 |
| 7 | [TradingAgents](./TradingAgents.md) | 防幻觉工程：私有子图隔离、point-in-time 裁剪、失败显式化 | 29k 行 | 中 |
| 8 | [generative_agents](./generative_agents.md) | 认知闭环原型：记忆流 + 反思 + 计划。2023 年 8 月后冻结，建议读不跑 | 12k 行 | 高 |

## 几条跨项目的结论

1. **编排模式只有三款**：单 while 循环（openai-agents）、显式图加超步（langgraph、TradingAgents）、级联队列（LightRAG）。其余框架都是这三种的变体。

2. **「多角色」不等于多 Agent**。ChatDev 2.0 里角色只是 YAML 中的 prompt 节点，真正的并行隔离要看 TradingAgents 的私有子图。

3. **持久化是最被低估的一环**。langgraph 把 checkpointer 做成可插拔契约，还配了一致性测试套件 `libs/checkpoint-conformance`——目前最值得抄的做法。

4. **论文和实际仓库经常不是一回事**。ChatDev、TradingAgents、storm 三个项目的代码都已大幅偏离原论文，读之前先看文档里记录的 commit 与版本。

5. **冻结的项目照样有读的价值**。generative_agents 的记忆三项加权（`reverie/backend_server/retrieve.py` 里的 `gw=[0.5,3,2]`）至今仍是记忆检索的参考实现。

## 代码在哪

八份文档各自记录了快照 commit。学习时按项目 README 的安装说明克隆，或直接在 GitHub 上按相对路径打开阅读——文档里的所有路径都是相对项目根目录的。

## 免责

文档针对**特定 commit 快照**写成，框架迭代很快。发现有对不上的地方，以代码为准。
