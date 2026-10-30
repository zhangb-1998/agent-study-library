# Agent Study Library

个人 AI Agent 学习代码库。用 **git submodule** 引用外部优秀开源项目——主仓只存引用（URL + commit），代码本体从原项目拉取，不复制、不改写、不占主仓体积。

## 已引用项目

| 目录 | 项目 | star | 看什么 |
|---|---|---|---|
| `openai-agents-python` | openai/openai-agents-python | 30k | Agent / Handoff / Guardrail / Session 四个核心抽象，最轻量 |
| `langgraph` | langchain-ai/langgraph | 44k | 状态机编排、checkpoint、中断与恢复 |
| `generative_agents` | joonspk-research/generative_agents | 22k | 记忆流 / 反思 / 规划，Agent 记忆设计的源头 |
| `TradingAgents` | TauricResearch/TradingAgents | - | 多角色辩论式决策架构 |
| `storm` | stanford-oval/storm | 31k | 多视角提问 + 大纲驱动的知识策展 |
| `gpt-researcher` | assafelovic/gpt-researcher | 30k | 研究 pipeline 的阶段拆分 |
| `LightRAG` | HKUDS/LightRAG | 40k | 图谱 RAG 轻量实现 |
| `ChatDev` | OpenBMB/ChatDev | 35k | SOP 驱动的多智能体协作 |

## 使用

```bash
# 首次克隆（含子模块）
git clone --recurse-submodules <你的仓库地址>

# 已克隆但子模块为空时
git submodule update --init --recursive

# 更新某个子模块到上游最新
cd langgraph && git fetch && git checkout main && git pull
cd .. && git add langgraph && git commit -m "update langgraph"
```

子模块默认处于 detached HEAD，不要直接在里面改代码；要改就 fork 后换 URL。

## 分支约定

- AI 侧（WorkBuddy）的所有改动一律推 `ai/YYYYMMDD-<内容>` 分支，**不直接推主分支**
- 仓库 owner 审核后自行合并到主分支；审核不通过直接删分支即可
- 主分支保持 owner 亲手提交的内容

## 配套资料

- 飞书「AI大模型资料库」离线副本：Skill `llm-study-kb`（78 篇 / 89 万字）
- 案例 ↔ 项目对照表：`outputs/GitHub项目对照清单_V1.md`

## 建议阅读顺序

1. openai-agents-python（先把抽象搞清）
2. langgraph（编排怎么落地）
3. generative_agents（记忆与规划的经典设计）
4. TradingAgents（多角色协作）
5. storm / gpt-researcher（研究类 pipeline）
6. LightRAG → ChatDev（RAG 进阶与 SOP 协作）
