# gpt-researcher 技术解析

> 面向有前端 + Node/PHP 背景、转 AI Agent 工程的工程师。所有结论均取自本地快照源码，行号取自快照 commit `3582a68`。

## 一句话定位

gpt-researcher 是一个 **自主深度研究 Agent**：给定一句话主题，它自动完成「生成子问题 → 并发检索 → 抓取正文 → 上下文筛选压缩 → 分段撰写 → 事实核查 → 汇总成带引用的报告」。它位于技术栈的 **Agent 编排层**——上游调 LLM 做规划与写作，下游通过可插拔的 **retriever** 接各种搜索源（Tavily / Google / DuckDuckGo / arXiv / MCP 等）。与 LightRAG 的分工是：LightRAG 是「知识底座」，gpt-researcher 是「会用知识的编排者」，两者可组合（把 LightRAG 当 retriever 用）。

## 事实卡片

| 项目 | 值 |
|---|---|
| 主语言 | Python（要求 `>=3.12`，见 `pyproject.toml`）；仓库另含 `frontend/`（Next.js，TSX） |
| 代码行数 | 核心三目录共 **19,758 行**：`gpt_researcher/` 15,934、`multi_agents/` 1,894、`deep_agents/` 1,930 |
| 源文件数 | 核心三目录共 **169** 个 `.py`；仓库总文件数 716（其中 `.py` 366） |
| 主要依赖 | `langchain>=1.0.0`、`langchain-core`、`langchain-community`、`langchain-openai`、`langchain-ollama`、`langgraph>=0.2.76`、`openai`、`ollama`、`litellm`、`google-genai`、`tavily-python`、`ddgs`、`arxiv`、`beautifulsoup4`、`pymupdf`、`unstructured`、`pandas`、`tiktoken`、`numpy`、`loguru`、`md2pdf`、`mistune`、`mcp`、`langchain-mcp-adapters`、`sqlalchemy`、`json-repair`、`json5` |
| 许可证 | Apache-2.0（`LICENSE` 首行 "Apache License Version 2.0, January 2004"） |
| 本地快照 commit | `3582a68`（`main`，2026-10-10）—— "Merge pull request #2206 from assafelovic/fix/config-isolate-mutable-defaults" |
| 包版本 | `0.16.0`（`pyproject.toml` / `setup.py`） |
| star 数 | 【待核】 |

## 目录地图

顶层：

| 路径 | 装什么 |
|---|---|
| `gpt_researcher/` | 核心包：单 Agent 研究流水线（`GPTResearcher` 类） |
| `multi_agents/` | 基于 LangGraph 的 8 角色多智能体团队（另有 `ag2/` 变体） |
| `deep_agents/` | 基于 LangChain Deep Agents 的 2 角色实现 + 基准测试套件 |
| `backend/` | FastAPI 服务：`server/`（API）、`report_type/`（basic/detailed/deep）、`chat/`、`memory/` |
| `frontend/` | Next.js 前端 |
| `skills/` | 对外暴露的 skill 封装层（仅含 `gpt-researcher/` 子目录） |
| `mcp-server/` | 仅 README，MCP 集成说明 |
| `docs/` | Docusaurus 站点 |
| `evals/` | 评测集（含 `context_filter` 基准） |
| `main.py` / `cli.py` | uvicorn 服务入口 / 命令行入口 |
| `terraform/` | IaC 部署清单 |

`gpt_researcher/` 关键二级：

| 路径 | 装什么 |
|---|---|
| `agent.py` | `GPTResearcher` 主类（:37）；`conduct_research`(:343)、`write_report`(:463) |
| `actions/` | 原子动作：`query_processing.py`（子问题生成）、`retriever.py`（retriever 工厂）、`report_generation.py`、`web_scraping.py`、`agent_creator.py` |
| `skills/` | 组合层：7 个模块，见架构图第三节 |
| `context/` | 上下文处理：`compression.py`、`jev_filter.py`、`lexical.py`、`retriever.py`、`select.py` |
| `retrievers/` | **23 个检索源**实现 + `base.py` + `utils.py` |
| `scraper/` | 网页抓取：`scraper.py`、`browser/`、`utils.py` |
| `llm_provider/` | LLM 抽象：`generic/base.py`、`image/` |
| `config/` | 配置：`config.py`、`variables/{base,default}.py` |
| `memory/`、`vector_store/`、`mcp/`、`document/`、`utils/` | 记忆、向量库、MCP 客户端、本地文档、通用工具 |

## 核心架构

```
  ┌────────────────────────────────────────────────────────────────────────┐
  │ 入口                                                                 │
  │   main.py  -> uvicorn.run(backend.server.app:app, :8000)            │
  │   cli.py   -> python cli.py "<query>" --report_type ...             │
  │   pip      -> from gpt_researcher import GPTResearcher               │
  └───────────────────────────────┬────────────────────────────────────────┘
                                  v
  ════════════════════════════════════════════════════════════════════════
  一、研究流水线   gpt_researcher/skills/researcher.py : ResearchConductor
  ════════════════════════════════════════════════════════════════════════

   conduct_research (:97)
        │ 按 report_source 分派 (:145-211)
        │   source_urls / web / local / hybrid / azure /
        │   langchain_documents / langchain_vectorstore
        v
   _get_context_by_web_search (:298)
        ├─[1] MCP 预热 (:319-362)   _mcp_cache_lock 保证只跑一次
        │      mcp_strategy: disabled / fast(缓存) / deep(每子查询各跑)
        ├─[2] _get_initial_search_results (:50) -> initial_results
        ├─[3] plan_research (:68) -> generate_sub_queries
        │        actions/query_processing.py:83（三级 LLM 降级）
        │        => sub_queries [+ 原始 query，非 subtopic_report 时]
        ├─[4] asyncio.gather(*[_process_sub_query(q)...]) (:392-397)
        │        └─ _process_sub_query (:520)
        │             ├─ MCP 检索 (:563-603)
        │             ├─ _scrape_data_by_urls (:607) -> scraper_manager
        │             ├─ context_manager.get_similar_content_by_query (:612)
        │             └─ _combine_mcp_and_web_context (:739)
        v
   researcher.context = research_data
   [可选] source_curator.curate_sources (:215)

  报告生成链路（另一条）：
   write_report (:463) -> ReportGenerator.write_report (writer.py:49)
        -> generate_report (actions/report_generation.py:209)
   write_introduction (:520) -> ReportGenerator.write_introduction (writer.py:175)
        -> write_report_introduction (report_generation.py:12)
   write_report_conclusion (:506) -> ReportGenerator... (:136)
        -> write_conclusion (report_generation.py:63)

  ════════════════════════════════════════════════════════════════════════
  二、Agent 集合  —— 两套并行实现
  ════════════════════════════════════════════════════════════════════════

   (A) multi_agents/   LangGraph StateGraph，图是硬编码的，8 个角色
       ChiefEditorAgent (multi_agents/agents/orchestrator.py:26)
         browser (ResearchAgent.run_initial_research)
           -> planner (EditorAgent.plan_research)
             -> human (HumanAgent.review_plan)
                  ├─ accept -> researcher (EditorAgent.run_parallel_research)
                  │               └─ 每个 section 一个 DraftState 子图
                  │                    researcher -> reviewer -> reviser
                  │                    reviewer: accept->END / revise->reviser
                  └─ revise -> planner      (上限 max_plan_revisions)
         researcher -> writer -> fact_checker
                         ├─ accept -> visualizer -> publisher -> END
                         └─ revise -> writer    (上限 max_fact_check_revisions)

   (B) deep_agents/    LangChain Deep Agents harness，2 个角色
       build_agent (deep_agents/agent.py:60)
         create_deep_agent(
             model        = task["model"] 或 STRATEGIC_LLM,
             tools        = [quick_search],
             system_prompt= CHIEF_EDITOR_PROMPT + guidelines,
             subagents    = [researcher],
             backend      = FilesystemBackend(root_dir, virtual_mode=True))
         子 agent researcher 持有 [quick_search, deep_research]

  ════════════════════════════════════════════════════════════════════════
  三、skill 与检索器插拔
  ════════════════════════════════════════════════════════════════════════

   gpt_researcher/skills/__init__.py 导出 7 个：
     ResearchConductor    编排（子问题 + 并发检索 + 上下文）
     ReportGenerator      写作（正文 / 引言 / 结论）
     ContextManager       上下文检索与压缩
     BrowserManager       网页抓取调度
     SourceCurator        源可信度排序过滤
     ImageGenerator       配图生成与嵌入
     DeepResearchSkill    递归式深度研究

   gpt_researcher/actions/retriever.py:
     get_retriever(name) (:16)
        17 个内置名走 match/case 惰性 import
        未命中 -> _load_plugin_retriever (:102)，
                   查 entry_points(group="gpt_researcher.retrievers") (:14)
     get_retrievers(headers, cfg) (:113)
        优先级 headers.retrievers > headers.retriever
              > cfg.retrievers  > cfg.retriever
              > TavilySearch（:146 定义的默认）
```

## 关键抽象与执行流

### 1. `GPTResearcher` —— 门面（`gpt_researcher/agent.py:37`）

整个包对外只导出它一个（`gpt_researcher/__init__.py`）。它的 `__init__`（:53）接收约 20 个参数，但真正重要的是三件事：

- **检索器在构造时就被解析并固化**：`self.retrievers = get_retrievers(self.headers, self.cfg)`（:176）。这意味着运行期改 env 不生效。
- **七个子组件在 :186-196 一次性装配**：`ResearchConductor`、`ReportGenerator`、`ContextManager`、`BrowserManager`、`SourceCurator`、`ImageGenerator`；`DeepResearchSkill` 只在 `report_type == "deep_research"` 时创建（:192-193）。
- **`memory` 是懒加载的**（:283-289 的 property）。`agent.py:178` 的注释解释了原因：默认的 Jev 或 no-op 过滤器下，一次标准研究**根本不需要 embedding 模型**，因此不需要它的凭据。

### 2. `ResearchConductor` —— 流程编排（`gpt_researcher/skills/researcher.py:21`）

`conduct_research`（:97）是整个系统的中枢，逻辑只有两步：**选路 + 汇聚**。

**选路**：按 `report_source` 分派（:145-211）

| 分支 | 行为 | 行号 |
|---|---|---|
| `source_urls` | 直接用给定 URL，可选补一次网络检索 | :145 |
| `web` | 完整网络检索 | :163 |
| `local` | 先 `DocumentLoader(cfg.doc_path).load()`，再灌进 `vector_store` | :166 |
| `hybrid` | 本地 + 网络**两条并行** `asyncio.gather` | :175 |
| `azure` | Azure Blob 文档加载 | :189 |
| `langchain_documents` | LangChain Document 对象 | :199 |
| `langchain_vectorstore` | 直接查向量库 | :208 |

`hybrid` 那条值得单独看（:184-188）：两条 `_get_context_by_web_search` 并发跑，靠共享的 `visited_urls` 跨两条去重，最后用 `prompt_family.join_local_web_documents(docs_context, web_context)` 合并。

**子问题生成**：`plan_research`（:68）→ `generate_sub_queries`（`actions/query_processing.py:83`）。这一段有个实用的**三级降级链**：

1. 用 `strategic_llm` 且 `max_tokens=None`（:122）
2. 失败则重试并显式传 `max_tokens=cfg.strategic_token_limit`（:132-140）
3. 再失败则退回 `smart_llm` 并带上 `temperature`（:145-154）

返回前统一过 `_normalize_sub_queries`（`query_processing.py:12`），把 LLM 可能返回的 `dict` / 裸 `str` / `None` 强制归一化成 `list[str]`，全空则退回原始 query。这是很值得抄的防御式解析。

**并发检索**（`researcher.py:392-397`）：

```python
context = await asyncio.gather(
    *[self._process_sub_query(sq, scraped_data, query_domains)
      for sq in sub_queries]
)
```

注意这里是**无上限的 gather**，并发度等于子问题数量（默认 3-4 个）。真正限制并发的地方在更外层：单篇研究内部的 `_process_sub_query`（:520）串行做 MCP 检索 → 抓取（`_scrape_data_by_urls`，:607）→ 上下文筛选（`context_manager.get_similar_content_by_query`，:612）→ 合并（`_combine_mcp_and_web_context`，:739）。

`_get_context_by_web_search` 在 :388 之前还多做一步：`_get_context_from_initial_results`（:836）先把规划阶段那批结果固化成上下文，因为「后面的检索会把它们冲掉」。这个顺序依赖在源码注释里写得很清楚。

**结果汇聚**：`self.researcher.context = research_data`（:212）；若 `cfg.curate_sources` 为真，再过 `source_curator.curate_sources`（:215），并把 `List[dict]` 归一化成字符串，避免下游 `"\n".join()` 撞上 dict 崩溃（:219-227）。

### 3. 报告生成链路

两条独立路径，取决于 `report_type`：

**普通路径**（默认 `research_report`）：

```
GPTResearcher.write_report (agent.py:463)
  └─ ReportGenerator.write_report (skills/writer.py:49)
       └─ generate_report (actions/report_generation.py:209)
            ├─ get_prompt_by_report_type(report_type, prompt_family) (:250)
            ├─ 分支：subtopic_report / custom_prompt / 默认 (:253-258)
            ├─ 追加 AVAILABLE IMAGES 指令（若有预生成图）(:261-297)
            └─ create_chat_completion(stream=True, websocket=websocket) (:298)
                 失败则降级为无 system 角色的单轮调用 (:315-330)
```

`ReportGenerator.write_report` 开头有一个值得记住的守卫（`writer.py:82-88`）：**若上下文为空，返回一段"拒绝生成"的文案**，而不是硬编一份看起来有引用的报告。RAG 系统里这类"宁可 abstain 也不幻觉"的兜底很关键。

**深度研究路径**：`conduct_research` 在 :363-365 直接短路到 `_handle_deep_research`（:415），后者调 `DeepResearchSkill.run`。

### 4. skills 与 retrievers 插拔机制

**retriever 侧两级解析**（`gpt_researcher/actions/retriever.py`）：

- `get_retriever(name)`（:16）：17 个内置名用 `match/case` 惰性 import；**未命中则查 `entry_points(group="gpt_researcher.retrievers", name=name)`**（常量在 :14，加载函数在 :102）。第三方包只要在 `pyproject.toml` 声明这个 entry-point 组即可被识别，无需改本仓库。
- `get_retrievers(headers, cfg)`（:113）：四级优先级 `headers.retrievers` → `headers.retriever` → `cfg.retrievers` → `cfg.retriever` → 默认 `TavilySearch`（:146）。支持逗号分隔字符串与 list 两种输入，并 `.strip()` 每一项——源码注释（:151-154）指出这是为了修 "tavily, exa" 这类带空格的配置静默回退到默认的问题。
- 当前内置 23 个检索源：`retrievers/__init__.py` 的 `__all__`。

**retriever 契约**（`gpt_researcher/retrievers/base.py`）：`BaseRetriever` 是**可选**基类，不强制继承；`_search_relevant_source_urls` 只找 `requires_scraping` 属性。这段 module docstring 讲清了一个重要的历史坑：旧代码靠 `len(raw_content) > 100` 猜「这个检索器是否已抓过正文」，猜错就永不抓取、报告缺引用；`requires_scraping` 把猜测换成声明，默认 `True`，不声明则保持旧行为以兼容外部实现。

**skill 侧**（`gpt_researcher/skills/__init__.py`）：导出 7 个组合层组件，全部由 `GPTResearcher.__init__` 装配。分层是 `agent.py`（门面）→ `skills/`（组合）→ `actions/`（原子）。

**检索调用为何要 `to_thread`**：`get_search_results`（`actions/query_processing.py:44`）最后一行是 `await asyncio.to_thread(search_retriever.search, **search_kwargs)`（:81），注释说明「检索器是阻塞 HTTP 调用，要让事件循环空出来」。这是把同步 SDK 包进 async 编排的标准做法。

**MCP 的成本防护**：`_tavily_mcp_redundant_with_direct`（`researcher.py:492`）会检测「直接配了 Tavily 检索器 + 又配了 Tavily MCP」这种重复路径并跳过 MCP，源码注释指向 issue #1875。

### 5. `multi_agents` vs `deep_agents` —— 两套并行实现的区别

`deep_agents/README.md` 里有一张对照表，是最权威的说明：

| 能力 | `multi_agents/`（LangGraph） | `deep_agents/`（Deep Agents） |
|---|---|---|
| 规划 | `EditorAgent` 节点 + `ResearchState` | 内置 `write_todos` 工具 |
| 并行分段研究 | 每个 section 一个嵌套子图 | `task` 工具并行派生 `researcher` 子 agent |
| 上下文管理 | 状态 key 在节点间传递 | 草稿落盘到文件系统，子 agent 上下文隔离 |
| 评审 / 修订 | `Reviewer` + `Revisor` 节点组成循环 | chief editor 读草稿后委派修订 |
| 发布 | `PublisherAgent` | 磁盘上组装最终 `report.md` |

具体差别：

**`multi_agents/`** —— 图是显式硬编码的。`ChiefEditorAgent`（`orchestrator.py:26`）用 `StateGraph(ResearchState)`（:66）建图，8 个节点（browser / planner / researcher / writer / fact_checker / visualizer / publisher / human）在 `_create_workflow`（:66）注册，边在 `_add_workflow_edges`（:84）连。两个条件边：

- `human` → `accept: researcher` / `revise: planner`（:97-102）
- `fact_checker` → `accept: visualizer` / `revise: writer`（:105-110）

循环有界是通过「抛异常 → 边上捕获 → 强制 accept」实现的（`_route_human_feedback` :112、`_route_fact_check` :131），注释明确说这是为了不撞 LangGraph 的 `recursion_limit`。并行发生在 `EditorAgent.run_parallel_research`（`editor.py:52`）：对每个 section `asyncio.gather` 一个子图调用，并通过 `sibling_sections`（:137）告知兄弟章节标题，避免各段重复覆盖。

**`deep_agents/`** —— 没有图。`build_agent`（`deep_agents/agent.py:60`）只调一次 `create_deep_agent`，声明 model / tools / system_prompt / subagents / backend。规划、委派、上下文管理都由 harness 涌现。子 agent `researcher` 持有 `quick_search` 与 `deep_research` 两个工具（`deep_agents/tools.py:9`）。上下文隔离靠 `FilesystemBackend(root_dir=run_dir, virtual_mode=True)`：草稿写到 `sections/<两位序号>-<slug>.md`，最终 `report.md` 在磁盘上组装。

**选择建议**：要精确控制流程、插入自定义节点、可复现调试 → `multi_agents/`；要少写代码、让模型自己决定怎么拆和什么时候停 → `deep_agents/`。README 给出的基准（DeepResearch Bench）显示 deep agent + GPT Researcher 相比 deep agent + 原始搜索工具，有效引用数 18.6 → 35.2（+89%）。另外 `multi_agents/ag2/` 是同一流程的 AG2 实现，配置与 `multi_agents/` 独立。

## 最小可运行入口

**(1) pip 包（最快）** —— `README.md:138-155`

```bash
pip install gpt-researcher
```

```python
from gpt_researcher import GPTResearcher

query = "why is Nvidia stock going up?"
researcher = GPTResearcher(query=query)
research_result = await researcher.conduct_research()
report = await researcher.write_report()
```

**(2) 源码 + Web 服务** —— `README.md:97-134`

```bash
# 1. Python 3.12+
git clone https://github.com/assafelovic/gpt-researcher.git
cd gpt-researcher

# 2. API keys（export 或写进 .env）
export OPENAI_API_KEY={Your OpenAI API Key here}
export TAVILY_API_KEY={Your Tavily API Key here}
# 自定义 OpenAI 兼容端点（本地模型等）：
# export OPENAI_BASE_URL={Your custom API base URL here}

# 3. 装依赖并起服务
pip install -r requirements.txt
python -m uvicorn main:app --reload
# 浏览器打开 http://localhost:8000
```

可选追踪（`README.md:114-119`）：`export LANGCHAIN_TRACING_V2=true` + `LANGCHAIN_API_KEY`。

**(3) Docker Compose** —— `README.md:250-270`

```bash
cp .env.example .env      # 填入 API Keys
docker-compose up --build
# 或 docker compose up --build
```

默认起两个进程：Python 服务 `localhost:8000` 与 React 应用 `localhost:3000`。

**(4) 命令行** —— `cli.py:1-9`

```shell
python cli.py "<query>" --report_type <report_type> --tone <tone> --query_domains <foo.com,bar.com>
```

`--report_type` 必填，取值来自 `ReportType`（`gpt_researcher/utils/enum.py:6`）；`--tone` 默认 `objective`；另有 `--encoding`（默认 utf-8）、`--no-pdf`、`--no-docx`。

**(5) MCP 混合检索** —— `README.md:159-164`

```bash
export RETRIEVER=tavily,mcp    # 网络检索 + MCP 数据源
```

```python
researcher = GPTResearcher(
    query="What are the top open source web research agents?",
    mcp_configs=[{"name": "github", "command": "npx",
                  "args": ["-y", "@modelcontextprotocol/server-github"],
                  "env": {"GITHUB_TOKEN": os.getenv("GITHUB_TOKEN")}}])
```

**(6) 本地文档研究** —— `README.md:275-285`

```bash
export DOC_PATH="./my-docs"
```

支持 PDF / 纯文本 / CSV / Excel / Markdown / PowerPoint / Word。

**(7) 内联配图** —— `README.md:198-203`

```bash
IMAGE_GENERATION_ENABLED=true
GOOGLE_API_KEY=your_google_api_key
IMAGE_GENERATION_MODEL=models/gemini-2.5-flash-image
```

**(8) 上下文过滤器** —— `README.md:240-245`

```bash
export CONTEXT_FILTER=auto    # auto | jev | keyword | embeddings | none
```

## 值得抄的工程手法

**1. 用 entry-point 做插件发现，内置名优先**
`gpt_researcher/actions/retriever.py:14` 定义 `RETRIEVER_ENTRY_POINT_GROUP = "gpt_researcher.retrievers"`，`get_retriever`（:16）先走内置 `match/case`，未命中才 `_load_plugin_retriever`（:102）遍历 `entry_points`；加载失败只 warning 并返回 `None`，由调用方 `or get_default_retriever()` 兜底。生态扩展与核心解耦，且不会因为一个坏插件炸掉整条流水线。

**2. 三级 LLM 降级链，而不是一崩就完**
`generate_sub_queries`（`actions/query_processing.py:83`）在 :117-154 做了战略模型（`max_tokens=None`）→ 战略模型（显式 token 上限）→ 智能模型（带 temperature）三层回退，每层都 `logger.warning` 记录原因并指向 issue #1022。对「不同 provider 对 `max_tokens=None` 支持不一致」这类现实问题，这比 try/except 包一层有效得多。

**3. 防御式解析 LLM 输出**
`_normalize_sub_queries`（`query_processing.py:12`）显式处理 `json_repair.loads` 可能返回的 `list` / `dict`（含 `queries`/`sub_queries`/`subQueries`/`items` 与单键 `query` 两种形态）/ 裸 `str` / `None`，最终一定返回 `list[str]`，且全空时退回原始 query。配合 `json_repair` 与 `extract_json_with_regex`（`actions/agent_creator.py`），把「LLM 不按格式输出」从崩溃变成降级。

**4. 有界循环：抛异常 + 边上捕获，而不是图内计数**
`multi_agents/agents/orchestrator.py` 的 `_route_human_feedback`（:112）与 `_route_fact_check`（:131）都捕获 `MaxPlanRevisionsExceededError` / `MaxFactCheckRevisionsExceededError` 并返回 `"accept"`。语义是「撞到上限就放行，别死在 `recursion_limit` 上」。把「超限策略」放在边上而不是节点里，节点只管抛，职责清楚。

**5. 并行章节之间靠 `sibling_sections` 显式避让**
`multi_agents/agents/editor.py:137` 的 `_create_task_input` 写入 `sibling_sections`，注释说明「章节并行研究，谁也看不到别人写了什么；把名字告诉它，让每段作者别越界」。这是并行生成长文时避免重复覆盖的最小成本做法。

**6. 用 `asyncio.to_thread` 包住阻塞 SDK**
`get_search_results`（`actions/query_processing.py:81`）把同步 `search()` 丢进线程池，注释写明理由。编排层全是 async，但底层搜索 SDK 多为同步——这个桥是必须的，别写成直接调用。

**7. 空上下文时主动 abstain**
`ReportGenerator.write_report`（`skills/writer.py:82-88`）在上下文为空时返回拒答文案。RAG 里「检索失败要可见」这条原则，这里给了可直接复制的实现。

## 阅读路径

按 9 步走，每步都为下一步铺路：

1. `gpt_researcher/utils/enum.py:6-50` — 先认清 `ReportType` / `ReportSource` / `Tone` 三个枚举，后面所有分支都由它们驱动。
2. `gpt_researcher/agent.py:37-201` — `GPTResearcher` 的字段与组件装配。重点看 :176 的 `get_retrievers` 与 :186-196 的七个子组件。只读构造，跳过方法体。
3. `gpt_researcher/actions/retriever.py` — 整篇 155 行，短而关键。看懂 `get_retriever` / `get_retrievers` 的解析优先级与 entry-point 插件机制，就理解了"插拔"的落点。
4. `gpt_researcher/skills/researcher.py:97-243` `conduct_research` — 中枢选路逻辑，先只看分派不看细节。
5. `gpt_researcher/actions/query_processing.py` — 整篇 216 行。子问题生成 + 三级降级 + 防御式归一化，是"LLM 输出不可信"这一课的最佳样例。
6. `gpt_researcher/skills/researcher.py:298-408` `_get_context_by_web_search` — 回到编排层看 MCP 预热、初始检索、规划、gather 的完整顺序，注意 :388 那条"先固化初始结果"的顺序依赖。
7. `gpt_researcher/skills/researcher.py:520-663` `_process_sub_query` — 单个子问题的完整生命周期：MCP → 抓取 → 上下文筛选 → 合并。
8. `gpt_researcher/skills/writer.py:49-134` + `gpt_researcher/actions/report_generation.py:209` — 报告生成链路，顺带看 `writer.py:82` 的空上下文守卫。
9. 二选一深入编排实现：
   - 要显式图 → `multi_agents/main.py` → `multi_agents/agents/orchestrator.py:26` → `multi_agents/agents/editor.py:52`
   - 要 harness → `deep_agents/main.py` → `deep_agents/agent.py:60` → `deep_agents/tools.py:9`

补充入口：想加新检索源照抄 `gpt_researcher/retrievers/tavily/tavily_search.py`（156 行，最短）；想理解上下文压缩看 `gpt_researcher/context/compression.py` 与 `context/select.py`。

## 别踩的坑

**`GPTResearcher` 构造时就固化了 retriever 列表**
`self.retrievers = get_retrievers(self.headers, self.cfg)`（`agent.py:176`）。运行期改 `os.environ` **不会**生效。这在 `multi_agents` / `deep_agents` 里尤其容易出错——嵌套创建的 `GPTResearcher` 会各自解析一遍。

**默认依赖 Tavily，没有 key 会退化**
`get_default_retriever()`（`actions/retriever.py:146`）返回 `TavilySearch`。不配 `TAVILY_API_KEY` 又没显式设 `RETRIEVER`，检索会失败；配合 `write_report` 的空上下文守卫（`writer.py:82`），最终表现为「报告说检索不到东西」而不是报错——排查时先看 retriever 解析结果。

**`mcp_configs` 走 cfg 不走 env，这是刻意的**
`_process_mcp_configs`（`agent.py:295`）的 docstring 写明：通过修改 `self.cfg.retrievers` 而非 `os.environ` 来启用 MCP，就是为了避免进程级环境变量污染影响并发或后续请求（修 issue #1676）。改代码时别"优化"回 env 写法。

**Tavily 双路径会自动去重，可能让你以为 MCP 没生效**
`_tavily_mcp_redundant_with_direct`（`skills/researcher.py:492`）在检测到「直接 Tavily 检索器 + Tavily MCP」时跳过 MCP 并 warning。想要 MCP 生效，就得从 `RETRIEVER` 里去掉 `tavily`，或换非 Tavily 的 MCP server。

**`asyncio.gather` 没有并发上限**
子问题在 `skills/researcher.py:392` 一次性 gather，并发数 = 子问题数（`generate_sub_queries` 默认 3-4 个，`deep_research` 的 breadth 默认 4）。`deep_agents` 分支里有 `Semaphore(concurrency_limit)`（`deep_research.py:432`，默认 2），但普通路径没有。量大的场景要自己加闸门，否则容易触发搜索 API 限流。

**`deep_research` 是递归的，成本随 breadth × depth 增长**
`DeepResearchSkill.__init__`（`skills/deep_research.py:251`）默认 `breadth=4`、`depth=2`、`concurrency=2`；递归时 `new_breadth = max(2, breadth // 2)`、`new_depth = depth - 1`。README 自己也提示单次 deep research 要几分钟。别把它放在循环里。

**`deep_agents/` 依赖外部包 `deepagents`**
`deep_agents/agent.py:6-7` 从 `deepagents` 与 `deepagents.backends` 导入，requirements 见 `deep_agents/requirements.txt`——与仓库根 `requirements.txt` 分离。装了根依赖不等于能跑 `deep_agents`。

**`multi_agents/` 有独立的 requirements 与 langgraph.json**
`multi_agents/requirements.txt`、`multi_agents/langgraph.json`（指定 `./multi_agents` 为依赖、图入口为 `./multi_agents/agent.py:graph`，python 3.12）。`multi_agents/ag2/` 又有一套自己的 requirements。三个目录的依赖是**互不包含**的。

**`multi_agents/agent.py` 在 import 时就构造了 ChiefEditorAgent**
它是模块级脚本（含硬编码 query "Is AI in a hype cycle?"），不是可导入的库。LangGraph Server 通过 `langgraph.json` 引用它的 `graph` 对象——所以 import 副作用是设计的一部分，别随手改成函数。

**版本要求偏新**
根 `requirements.txt` 首行注释写明 "Python 3.10+ required for LangChain v1"，但 `pyproject.toml` 要求 `>=3.12`。**以 `pyproject.toml` 的 3.12 为准**，Python 3.10/3.11 装不上官方包。

**日志与产物落盘到固定目录**
`main.py:6-21` 在启动时创建 `./logs/` 并写 `logs/app.log`；`multi_agents` 输出到 `./outputs/run_{task_id}_{query前40字符}/`（`orchestrator.py:44-50`）。容器化部署时记得挂载这两处。
