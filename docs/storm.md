# STORM 技术解析文档

> 面向有 5 年前端 + Node/PHP 经验、正在转 AI Agent 工程方向的工程师。
> 本文所有路径、类名、行号均来自本地快照 `github-refs/storm`（commit `fb951af`）的真实读取，未使用外部记忆。
> 行号会漂移，因此每条都同时标注了附近的函数名作为锚点。

---

## 一句话定位

**它解决什么问题**：STORM 解决「给一个主题，自动写一篇带引用的类维基百科长文」的问题。它的核心洞察是——自动化的难点不在「写」，而在「知道该问什么问题」。LLM 直接被提示去提问，问出来的东西又浅又窄；STORM 用两种策略补上：**多视角地问（Perspective-Guided Question Asking）** 和 **模拟一场维基作者与领域专家的对话（Simulated Conversation）**，把检索到的互联网信息沉淀成素材，再走提纲 → 正文 → 润色的流水线产出文章。

**在 Agent 技术栈的哪一层**：处在 **知识策展层（knowledge curation layer）**，位于检索和写作之间。它不是一个通用 Agent 运行时，没有工具调用的自主循环、没有动态规划链路、也没有持久会话——它是一条**显式编排的四阶段流水线**。LLM 只是流水线里被调用的确定性组件。README 的原话是："Both STORM and Co-STORM are working in the information curation layer, you need to set up the information retrieval module and language model module to create their `Runner` classes respectively."

**和现代 Agent 框架的关系（历史位置）**：STORM 是 **前 LangChain/LangGraph 时代**的产物，它的编排层是自己手写的而不是用图框架。具体表现：

- **它的 DSL 是 DSPy，不是 LangChain**。全部 LLM 调用都写成 `dspy.Signature` + `dspy.Module` + `dspy.Predict`/`dspy.ChainOfThought`（见 `knowledge_storm/storm_wiki/modules/*.py`）。`dspy_ai==2.4.9` 被硬钉在 `requirements.txt:1`。这意味着你看到的是「用声明式签名 + Prompt 编译器」的范式，而不是 LangChain 的 `Chain`/`Runnable` 范式。
- **编排是手写的顺序函数**。`STORMWikiRunner.run()`（`knowledge_storm/storm_wiki/engine.py:341`，附近函数名 `run`）就是一个 `if do_research / if do_generate_outline / ...` 的直线流程。换成 LangGraph，这里会是一个四节点的有向图。
- **2024/04 做过一次重构**，把接口抽象出来了。README 的 release note 写：*"We release refactored version of STORM codebase! We define interface for STORM pipeline and reimplement STORM-wiki ... to demonstrate how to instantiate the pipeline."* 所以 `knowledge_storm/interface.py` 里那批 ABC（`Engine` / `KnowledgeCurationModule` / `OutlineGenerationModule` / `ArticleGenerationModule` / `ArticlePolishingModule`）是后加的"面向扩展"的壳，不是一开始就有的设计。
- **2025/01 才接 litellm**。README: *"We add litellm integration for language models and embedding models in `knowledge-storm` v1.1.0."* 在此之前每个厂商一个 wrapper 类，`lm.py:272` 附近的注释原样写着 `# The following language model classes were deprecated after v1.1.0.`——这条注释本身就是这个仓库演进史的化石。

**一句话结论**：把它读成「**RAG 写作流水线**」而不是「Agent」。它的价值在多阶段拆解与素材去重/引用对齐的工程细节，不在自主决策。

---

## 事实卡片

| 项 | 值 |
|---|---|
| 主语言 | Python（`>=3.10`，`setup.py:29` 的 `python_requires`） |
| 代码行数 | **14,390** 行 Python（`git ls-files "*.py"` 全部统计） |
| 源文件数 | 被 git 跟踪的文件 **74** 个，其中 `.py` **52** 个 |
| 主要依赖 | `dspy_ai==2.4.9`（硬钉）、`wikipedia==1.4.0`、`sentence-transformers`、`litellm`、`qdrant-client`、`langchain-qdrant`、`langchain-text-splitters`、`trafilatura`、`diskcache`、`toml` —— 见 `requirements.txt` |
| 许可证 | **MIT**（`LICENSE` 第 1 行；`setup.py` 写 `"MIT License"`，版权属 Stanford Open Virtual Assistant Lab） |
| 本地快照 commit | **`fb951af`**（完整 `fb951af7744dab086e34962e9bc6fe878e145f83`），提交者 Yijia Shao，`2025-09-30 11:07:17 -0700`，commit message `Merge pull request #400 from stanford-oval/patch/requirement-change` |
| star 数 | 【待核】（本次为离线快照，未联网查询） |

**快照完整性警告**：本地 `.git` 是 **shallow clone**（存在 `.git/shallow`，`git rev-list --count HEAD` 返回 `1`）。所以"最后一次提交距今多久"只能到 commit 本身，无法推断仓库真实活跃度。下一 FAQ 段会进一步说明。

---

## 目录地图

顶层：

| 路径 | 一句话 |
|---|---|
| `knowledge_storm/` | **唯一真正的研究代码包**，全部要读的东西在这 |
| `examples/` | 可直接跑的入口脚本，按 LLM/检索后端分类，是最快的上手材料 |
| `frontend/` | `demo_light/`，一个用 Streamlit 写的本地演示界面（不是生产 UI） |
| `assets/` | 论文插图、NAACL 幻灯片 PDF、`overview.svg` / `two_stages.jpg`（建议先看图，对理解两阶段很有用） |
| `requirements.txt` / `setup.py` / `MANIFEST.in` | 打包配置。`setup.py` 里 `version="1.1.1"` |

`knowledge_storm/` 下各子模块：

| 路径 | 一句话 |
|---|---|
| `interface.py` | **先读它**。定义全流水线的抽象基类与数据契约：`Information` / `Article` / `ArticleSectionNode` / `Retriever` / `Engine` / `LMConfigs`，以及四个 XXModule 的 ABC |
| `lm.py` | LLM 适配层。`LitellmModel`（v1.1.0 起的主力）+ 一排 deprecated 的厂商 wrapper |
| `rm.py` | Retrieval Module 适配层。搜索引擎 RM + 本地向量 RM，全部继承 `dspy.Retrieve` |
| `dataclass.py` | **Co-STORM 专用**的数据结构：`ConversationTurn` / `KnowledgeNode` / `KnowledgeBase`（思维导图） |
| `encoder.py` | 薄封装 `Encoder` 类（`encoder.py:36`），给 VectorRM 生成 embedding |
| `utils.py` | 文本处理工具箱：`ArticleTextProcessing`（引用编号重排/清理/截断）、`FileIOHelper`、`WebPageHelper`、`QdrantVectorStoreManager`、以及提示词合规检查 |
| `logging_wrapper.py` | Co-STORM 的日志/DSL 包装，初读可跳过 |
| `storm_wiki/` | **经典 STORM 四阶段流水线实现**，主线在这 |
| `collaborative_storm/` | **Co-STORM**：人机协作版，多 Agent 话语轮换 + 思维导图。第二优先级 |

`knowledge_storm/storm_wiki/` 内部：

| 路径 | 一句话 |
|---|---|
| `engine.py` | `STORMWikiLMConfigs` + `STORMWikiRunnerArguments` + `STORMWikiRunner`，编排全在这里 |
| `modules/knowledge_curation.py` | 阶段 1。`ConvSimulator` / `WikiWriter` / `TopicExpert` |
| `modules/outline_generation.py` | 阶段 2。`WriteOutline` 两遍走：先草拟提纲，再用对话重构提纲 |
| `modules/article_generation.py` | 阶段 3。`ConvToSection`，按一级章节并发写正文 |
| `modules/article_polish.py` | 阶段 4。`PolishPageModule`，补 lead section + 去重 |
| `modules/storm_dataclass.py` | `StormInformationTable` + `StormArticle` + `DialogueTurn` |
| `modules/persona_generator.py` | 抠人品指的是 multi-perspective 的**视角生成器**（不是小说人设） |
| `modules/retriever.py` | **注意这是个误导性的文件名**。里面只有 `GENERALLY_UNRELIABLE` / `DEPRECATED` / `BLACKLISTED` 三个域名黑名单集合和 `is_valid_wikipedia_source()`（`:225`），没有任何 Retriever 类 |
| `modules/callback.py` | `BaseCallbackHandler`，9 个空钩子，用于 UI 打点 |

---

## 核心架构

```
                 ┌──────────────────────────────────────────────┐
                 │  CLI / 脚本入口                                │
                 │  examples/storm_examples/run_storm_wiki_*.py   │
                 │  (argparse + load_api_key(secrets.toml))       │
                 └───────────────────────┬──────────────────────┘
                                         │ 组装
                                         v
    ┌────────────────────────────────────────────────────────────────────┐
    │  STORMWikiRunner  (interface.Engine 子类)                            │
    │  engine.py:171   编排 = 直线 if 序列, 非图调度                        │
    │                                                                     │
    │   run() ──► run_knowledge_curation_module()                          │
    │         ──► run_outline_generation_module()                          │
    │         ──► run_article_generation_module()                          │
    │         ──► run_article_polishing_module()                           │
    └───────┬──────────────────────────────────────────────┬─────────────┘
            │ 依赖注入                                       │ 装饰器套壳
            v                                              v
 ┌──────────────────────────┐              ┌──────────────────────────────┐
 │  外部能力适配层            │              │  apply_decorators()           │
 │                          │              │  interface.py:512              │
 │  LMConfigs (5 个 *_lm)    │              │  扫描所有 run_* 方法, 自动包上 │
 │    └─ LitellmModel        │              │  log_execution_time_and_       │
 │       (lm.py:192)         │              │  lm_rm_usage 计时/计 token     │
 │                          │              └──────────────────────────────┘
 │  Retriever (interface.py:260)                                            │
 │    └─ rm: YouRM / BingSearch / SerperRM / BraveRM /                      │
 │       DuckDuckGoSearchRM / TavilySearchRM / SearXNG /                    │
 │       GoogleSearch / AzureAISearch / VectorRM / StanfordOvalArxivRM      │
 │    └─ 多线程并发Thread-pool retrieve (interface.py:311)                  │
 └──────────────────────────┘                                              │
            │                                                              │
            v                                                              │
 ┌─────────────────────────────────────────────────────────────────────┐   │
 │  DSPy 层：所有 LLM 认知单元都是 dspy.Signature + dspy.Module           │   │
 │                                                                      │   │
 │   阶段1  Conversation Simulator                                       │   │
 │     WikiWriter  AskQuestionWithPersona ──┐                            │   │
 │                                          ├─► 多轮循环 max_turn         │   │
 │     TopicExpert QuestionToQuery ──► rm ──┘                            │   │
 │                 AnswerQuestion                                        │   │
 │                                                                      │   │
 │   阶段2  WriteOutline  WritePageOutline → WritePageOutlineFromConv     │   │
 │   阶段3  ConvToSection WriteSection   (按一级章节并发)                  │   │
 │   阶段4  PolishPageModule WriteLeadSection + PolishPage                │   │
 └──────────────────────────────────────────────────────────────────────┘   │
            │                                                               │
            v                                                               │
 ┌─────────────────────────────────────────────────────────────────────┐   │
 │  数据层  Article = 树                                                 │   │
 │    Article (root=ArticleSectionNode)                                 │   │
 │      └─ children: [ArticleSectionNode{section_name, content, pref}]   │   │
 │    StormArticle.reference = {url_to_unified_index, url_to_info}       │   │
 └─────────────────────────────────────────────────────────────────────┘   │
            │                                                               │
            v                                                               │
 ┌─────────────────────────────────────────────────────────────────────┐   │
 │  落盘  article_output_dir/                                            │   │
 │    conversation_log.json / raw_search_results.json                    │   │
 │    direct_gen_outline.txt / storm_gen_outline.txt                     │   │
 │    storm_gen_article.txt / storm_gen_article_polished.txt             │   │
 │    url_to_info.json / run_config.json / llm_call_history.jsonl        │   │
 └─────────────────────────────────────────────────────────────────────┘   │
```

---

## 关键抽象与执行流

### 1. 编排层：`STORMWikiRunner`

**文件：`knowledge_storm/storm_wiki/engine.py`**

- `STORMWikiLMConfigs`（`engine.py:21`）在 `__init__` 里声明 **5 个槽位**，字段名必须带 `_lm` 后缀——这是约定，不是巧合：
  `conv_simulator_lm` / `question_asker_lm` / `outline_gen_lm` / `article_gen_lm` / `article_polish_lm`。
  带 `_lm` 后缀这个约束被 `interface.py` 的 `LMConfigs.init_check()`（`:436`）、`collect_and_reset_lm_history()`（`:443`）、`collect_and_reset_lm_usage()`（`:452`）用 `if "_lm" in attr_name` 反射遍历利用。**改字段名会让成本统计静默失效。**
- 五个模型槽位存在的原因写在 `examples/storm_examples/run_storm_wiki_gpt.py:66-70` 的注释里： conversational stage 的 query 拆分/回答合成用便宜快模型省钱，outline 和 article 生成用强模型保质量。
- `STORMWikiRunnerArguments`（`engine.py:127`）是关键超参集中地，默认值：`max_conv_turn=3`、`max_perspective=3`、`max_search_queries_per_turn=3`、`search_top_k=3`、`retrieve_top_k=3`、`max_thread_num=10`。
- `STORMWikiRunner.__init__`（`engine.py:174`）做三件事：建 `Retriever(rm, max_thread)` → 依次 new 四个 module → **调 `apply_decorators()`**。
- `apply_decorators()`（`interface.py:512`）用 `dir(self)` 反射找出**所有以 `run_` 开头的方法**，逐个套上 `log_execution_time_and_lm_rm_usage` 装饰器（包方法形参是 `wrapper(*args, **kwargs)`，注意这里**丢掉了 `self`**，靠闭包捕获）。这就是为什么 `run_knowledge_curation_module` 等方法定义里一行计时代码都没有，却能自动统计。
- `run()`（`engine.py:341`）是**断点续跑友好**的直线流程：每个阶段用 `do_research` / `do_generate_outline` / `do_generate_article` / `do_polish_article` 四个 bool 开关，关掉时会尝试从本地文件回填（`_load_information_table_from_local_fs` `:312`、`_load_outline_from_local_fs` `:320`、`_load_draft_article_from_local_fs` `:326`）。这四个开关直接对应 examples 里的 `--do-*` 命令行参数。
- `post_run()`（`engine.py:290`）写 `run_config.json` 与 `llm_call_history.jsonl`——**调试时最有价值的产物**，`llm_call_history.jsonl` 里有每一完整的 prompt/response。注意它会把每条记录的 `kwargs` pop 掉（`:306`），因为 kwargs 已经汇总进 `run_config.json`。
- `summary()`（`interface.py:544`）打印耗时 / token / 检索次数三类成本。

### 2. 阶段 1：Knowledge Curation（知识策展）

**文件：`knowledge_storm/storm_wiki/modules/knowledge_curation.py`**

调用链：`StormKnowledgeCurationModule.research()`（`:347`） → `_get_considered_personas()`（`:281`） → `_run_conversation()`（`:286`） → `ConvSimulator.forward()`（`:47`）。

- **视角生成**：`research()` 先判断 `disable_perspective`。注意一个容易踩的细节——尽管 args 里有 `disable_perspective` 字段，`STORMWikiRunner.run_knowledge_curation_module()`（`engine.py:215`）在调用时**硬编码传了 `disable_perspective=False`**（`:224`），所以 CLI 上的开关对 stage 1 实际无效，视角永远启用。这是代码与参数名不一致的地方，值得留意。
- **多视角并发**：`_run_conversation()`（`:286`）对每个 persona 起一条对话，`ThreadPoolExecutor(max_workers=min(max_thread_num, len(considered_personas)))`。它还有一段 Streamlit 特判（`:333`）：如果能 import `streamlit.runtime.scriptrunner`，会遍历 `executor._threads` 调 `add_script_run_ctx(t)`，把 script context 挂到线程池线程上，否则 Streamlit 前端拿不到日志。**这是很实用的一个技巧，但依赖私有属性 `_threads`，脆弱。**
- **对话循环**：`ConvSimulator.forward()`（`:47`）跑 `max_turn` 轮：
  1. `WikiWriter`（`:84`）生成提问。有两个 Signature：`AskQuestionWithPersona`（`:139`，带 persona）和 `AskQuestion`（`:128`，不带）。**上下文窗口控制**：只保留最近 4 轮的 Expert 回答全文，更早的轮次替换成 `"Expert: Omit the answer here due to space limit."`（`:103-110`），再 `limit_word_count_preserve_newline(conv, 2500)` 截断（`:113`）。
  2. 终止条件是两个字符串判断（`:64-68`）：空 utterance，或以 `"Thank you so much for your help!"` 开头。这个终止信号是写死在 Signature docstring 里的（`knowledge_curation.py:130`）。
  3. `TopicExpert`（`:181`）四步：生成 query（截断到 `max_search_queries`）→ `retriever.retrieve()` → **只用每条结果的第 1 个 snippet**（`:221`，注释说明简化了 multi-hop citation）→ `AnswerQuestion` 生成带引用回答。
  4. 检索无结果时**不幻觉**，直接返回固定句（`:240`）；回答生成异常时同样落到固定句（`:235-237`）。
  5. 每轮结束回调 `callback_handler.on_dialogue_turn_end()`（`:79`）。
- 收尾时对整个 conversation 调 `ArticleTextProcessing.clean_up_citation(conv)`（`:342`）清理引用编号。

### 3. 阶段 2：Outline Generation（提纲生成）

**文件：`knowledge_storm/storm_wiki/modules/outline_generation.py`**

- `StormOutlineGenerationModule.generate_outline()`（`:22`）先把 `information_table.conversations` 用 `sum([...], [])` **拍平成一维 dlg_history**（`:56`）交给 `WriteOutline`。
- `WriteOutline.forward()`（`:84`）是**两遍式**设计，这是这个阶段的核心：
  1. **第一遍** `self.draft_page_outline`（Signature `WritePageOutline`，`:128`）——**不看任何检索素材**，纯靠 LLM 参数知识草拟提纲。产出落盘为 `direct_gen_outline.txt`，用途是对照实验（证明"检索真的有用"）。
  2. **第二遍** `self.write_page_outline`（Signature `WritePageOutlineFromConv`，`:153`）——把第一遍的提纲 + 对话历史喂进去做 **refinement**。产出 `storm_gen_outline.txt`，这才是正文生成要用的。
  3. 中间有一层噪点过滤（`:92-98`）：跳过任何 `agent_utterance` 或 `user_utterance` 里含 `"topic you"` 的 turn。上下文同样截断，`limit_word_count_preserve_newline(conv, 5000)`（`:106`）。
  4. 两次都过 `ArticleTextProcessing.clean_up_outline()`（`:110` / `:117`）。
- `NaiveOutlineGen`（`:140`）是只做第一遍的基线类，实际流水线里**没有被调用**，但对做消融实验有用。

### 4. 阶段 3：Article Generation（正文生成）

**文件：`knowledge_storm/storm_wiki/modules/article_generation.py`**

- `generate_article()`（`:53`）先调 `information_table.prepare_table_for_retrieval()`（`storm_dataclass.py:109`）——这一步会加载 `SentenceTransformer("paraphrase-MiniLM-L6-v2")` 并 **把所有 snippet 一次性编码进内存**（`:117`）。这是本地 CPU embedding，不是 OpenAI embedding，注意区分：STORM 的 section 检索用的是这一段（本地 MiniLM），而 Co-STORM 的 `encoder.py` 才走 litellm/OpenAI。
- 取 `get_first_level_section_names()`（`storm_dataclass.py:423`）得到一级章节列表。
- **并发 + 两段式查询构造**（`:104-110`）：对每个一级章节，先取该章节子树的 `get_outline_as_list(add_hashtags=False)` 作为**检索 query**，再取 `add_hashtags=True` 的版本拼成 `section_outline` **喂给 LLM**。同一个子树被渲染成两种格式，一个用于向量检索、一个用于 prompt，很省事。
- **跳过 introduction / conclusion / summary**（`:96-103`）：这三个会被跳过，因为 summary 在阶段 4 会重新生成，避免重复写。
- `generate_section()`（`:33`）负责单章节：`information_table.retrieve_information(section_query, retrieve_top_k)` → `ConvToSection`（`:136`）。
- `ConvToSection.forward()`（`:144`）把 `collected_info` 拼成 `[1]\nsnippet...\n\n` 格式（`:148-150`），**截断到 1500 词**（`:152`），然后 `write_section`，再 `clean_up_section`。
- 所有章节写完后：`article.update_section()` 逐章节回填（`:127`）→ `article.post_processing()`（`:132`）。

### 5. 阶段 4：Article Polishing（润色）

**文件：`knowledge_storm/storm_wiki/modules/article_polish.py`**

- `StormArticlePolishingModule.polish_article()`（`:29`）: `to_string()` → `PolishPageModule` → 拼 `"# summary\n" + lead_section`（`:45-46`）→ 重新 `parse_article_into_dict` + `insert_or_create_section` → `post_processing()`。
- `PolishPageModule.forward()`（`:87`）里两个 linellm 调用各自包在独立的 `with dspy.settings.context(...)` 里，**显式传了 `show_guidelines=False`**，代码里两处注释解释得很清楚：`:88` 和 `:96` 都写着 `# NOTE: Change show_guidelines to false to make the generation more robust to different LM families.` —— 这是为兼容非 OpenAI 模型族做的妥协。
- `polish_whole_page` 参数由 `remove_duplicate` 控制（`:95`）；关掉时 `page = draft_page`，只加 lead 不去重。
- `WriteLeadSection` Signature（`:56`）要求"不超过四段"。
- 健壮性细节：`:93-94` 处理了模型把前缀 `"The lead section:"` 也吐出来的情况，手动 split 掉。

### 6. Wikipedia / Search 检索适配层

这里要说清楚一件事，因为容易和 `requirements.txt` 里的 `wikipedia==1.4.0` 混淆：**全仓库没有任何一处 `import wikipedia`（已 grep 全仓确认）**。所谓 "Wikipedia 适配" 实际是这两处：

1. **`persona_generator.py:10` 的 `get_wiki_page_title_and_toc(url)`** —— 用 `requests` + `BeautifulSoup` 直接抓一个维基页面的 HTML，从 `<h1>` 取标题，遍历 `h2..h6` 构建目录树；硬编码排除了 `Contents` / `See also` / `Notes` / `References` / `External links`（`:21-27`）。产出的 TOC 作为 `examples` 喂给 `GenPersona` Signature（`:56`），让 LLM 参考同类条目的章节结构。**这就是 STORM 借维基的方式——抓竞争者页面当 few-shot 素材。**
2. **`modules/retriever.py`** —— 三份按维基可靠来源标准整理的域名黑名单：`GENERALLY_UNRELIABLE`（约 140 项）/ `DEPRECATED`（约 45 项）/ `BLACKLISTED`（约 19 项），由 `is_valid_wikipedia_source(url)`（`:225`）消费。这套名单会作为 `is_valid_source` 回调注入各个搜索 RM 的构造函数。

搜索 RM 全部在 `knowledge_storm/rm.py`，统一继承 `dspy.Retrieve` 并实现 `forward(query_or_queries, exclude_urls)`，返回 `[{url, title, description, snippets}]`：
`YouRM:13` / `BingSearch:77` / `VectorRM:179`（Qdrant 本地或在线库，接私有语料）/ `StanfordOvalArxivRM:340` / `SerperRM:406` / `BraveRM:570` / `SearXNG:644` / `DuckDuckGoSearchRM:728` / `TavilySearchRM:858` / `GoogleSearch:984` / `AzureAISearch:1108`。

`Retriever` 基类本身在 **`interface.py:260`**（不在 `storm_wiki/modules/retriever.py`！）。它的 `retrieve()`（`:288`）用线程池并发处理多个 query，并在入库前对每个 snippet 调 `ArticleTextProcessing.remove_citations()`（`:303`）——注释写明是为了避免源文本自带的引用编号和 STORM 自己生成的编号混淆。

### 7. LLM 抽象层与支持的 backend

**文件：`knowledge_storm/lm.py`**

- 基类 `LM`（`:57`），核心是 `\`LM.__call__`（`:78`）统一分派：`model_type == "chat"` 走 `litellm_completion`，否则走 `litellm_text_completion`。
- **两级缓存**（值得抄）：`cached_litellm_completion`（`:116`）用 `functools.lru_cache(maxsize=3000)`（`LM_LRU_CACHE_MAX_SIZE`，`:54`）做进程内缓存，同时 `litellm.cache = Cache(disk_cache_dir=~/.storm_local_cache, type="disk")`（`:42-43`）做磁盘缓存。request 被 `ujson.dumps` 序列化成字符串当 cache key。这意味着**重跑同一主题的调试成本极低**。
- `LitellmModel`（`:192`）是 v1.1.0 起的主力实现。它在 `__call__`（`:233`）里额外做了 `log_usage()`（`:210`，**带 `threading.Lock`** 累加 prompt/completion token），以及 `get_usage_and_reset()`（`:218`）供 Engine 统计。注意单个 `LM.__call__` 基类版本**没有** `log_usage`，并发下可安全累加正是靠这把锁。
- Deprecated 但仍在的 backend（`lm.py:272` 的注释明确标注 `# deprecated after v1.1.0`）：
  `OpenAIModel:276` / `DeepSeekModel:366` / `AzureOpenAIModel:461` / `GroqModel:595` / `ClaudeModel:709` / `VLLMClient:845` / `OllamaClient:935` / `TGIClient:948` / `TogetherClient:1010` / `GoogleModel:1159`。
- **注意**：即使切到 litellm 主线，`examples/storm_examples/run_storm_wiki_gpt.py:30` 仍然 `from knowledge_storm.lm import OpenAIModel, AzureOpenAIModel`——**默认 example 走的还是 deprecated 路径**。想用 litellm 要按 README 顶部 API 示例自己组装 `LitellmModel`。
- Embedding 侧：新接的 litellm embedding 与 Co-STORM 相关，见 `README.md:96` 声明支持 "all embedding models supported by litellm"。经典 STORM 路径仍用 `SentenceTransformer`。

### 8. Article 数据结构

**文件：`interface.py` 定义抽象，`storm_wiki/modules/storm_dataclass.py` 实现**

- `ArticleSectionNode`（`interface.py:136`）：`section_name` / `content` / `children: List` / `preference`。`add_child(new, insert_to_front=False)`（`:152`）支持头插——被 StormArticle 用来把 summary 插到最前面。
- `Article`（`interface.py:162`，ABC）：`root = ArticleSectionNode(topic_name)`。核心方法 `find_section(node, name)`（`:166`，递归前序查找）、`get_outline_tree()`（`:193`）、`get_first_level_section_names()`（`:232`）、`prune_empty_nodes()`（`:246`，递归剪枝返回 None 表示可删）。**抽象方法**：`to_string()` 和类方法 `from_string()`——这两个是扩展点，换输出格式（比如改 Markdown 为 HTML）只需实现它们。
- `StormArticle`（`storm_dataclass.py:148`）在 Article 之上加了 `reference = {"url_to_unified_index": {}, "url_to_info": {}}`。
- **引用编号管理机制**（这是 STORM 最值得看的一段工程细节）：
  - `_merge_new_info_to_references()`（`:174`）把章节级的局部引用编号映射到全文级统一编号，返回 `{局部 idx: 全局 idx}` 映射，编码从 **1 开始**（`:196`、`:206` 两处注释都强调）。
  - `update_section()`（`:249`）先用 `re.findall(r"\[(\d+)\]", content)` 扫出实际用到的编号，把超出实际素材数量的幻觉编号直接**删 掉**（`:272-280`），然后按 `index_to_keep` 收敛 `current_section_info_list`（`:282`）——**只保留真正被引用到的素材**，最后用 `ArticleTextProcessing.update_citation_index()` 重写编号（`:286`）。
  - `reorder_reference_index()`（`:374`）做全文级重整：前序遍历收集引用出现顺序 → 构建 `{旧编号: 新编号}` 映射 → 重写所有节点 content → 同步重写 `url_to_unified_index`，并从 reference 里删掉最终未被使用的 url（`:406-412`）。
  - `post_processing()`（`:502`）= `prune_empty_nodes()` + `reorder_reference_index()`，在 article generation 和 polish 两阶段末尾都被调用。
- `insert_or_create_section()`（`:209`）支持 `trim_children`——覆盖式更新时删掉未出现在新 dict 中的旧子节点。
- `StormInformationTable`（`:48`）：`construct_url_to_info()`（`:65`）把所有对话 turns 里的 `search_results` 按 url 去重聚合 snippets；`prepare_table_for_retrieval()`（`:109`）用 MiniLM 编码；`retrieve_information(queries, search_top_k)`（`:119`）用 `cosine_similarity` + `np.argsort` 取 top-k（注意 `sorted_indices[-search_top_k:][::-1]`，逆序排列）。

---

## 最小可运行入口

全部命令摘自仓库真实文件。

**安装**（`README.md:75-89`，Installation 段）：

```bash
pip install knowledge-storm          # 用库
# 或从源码：
git clone https://github.com/stanford-oval/storm.git
cd storm
conda create -n storm python=3.11
conda activate storm
pip install -r requirements.txt
```

**配置 API key**（`README.md:224-238`，Quick Start with Example Scripts 段）——`secrets.toml` 放仓库根目录：

```toml
# ============ language model configurations ============
OPENAI_API_KEY="your_openai_api_key"
OPENAI_API_TYPE="openai"          # 或 "azure" 时配 AZURE_API_BASE / AZURE_API_VERSION
# ============ retriever configurations ============
BING_SEARCH_API_KEY="your_bing_search_api_key"
# ============ encoder configurations ============
ENCODER_API_TYPE="openai"
```

加载由 `knowledge_storm.utils.load_api_key(toml_file_path)`（`utils.py:41`）完成，example 脚本在 `main()` 第一行调用 `load_api_key(toml_file_path="secrets.toml")`（`run_storm_wiki_gpt.py:45`）。

**跑 GPT + Bing（最短路径）**（`README.md:243-250`）：

```bash
python examples/storm_examples/run_storm_wiki_gpt.py \
    --output-dir $OUTPUT_DIR \
    --retriever bing \
    --do-research \
    --do-generate-outline \
    --do-generate-article \
    --do-polish-article
```

这个脚本 **交互式读取 topic**：`run_storm_wiki_gpt.py:144` 有 `topic = input("Topic: ")`。

**可用参数**（`run_storm_wiki_gpt.py:156-241` 的 argparse）：`--output-dir`、`--max-thread-num`（默认 3）、`--retriever`（`bing`/`you`/`brave`/`serper`/`duckduckgo`/`tavily`/`searxng`/`azure_ai_search` 八选一）、四个 `--do-*`、`--max-conv-turn`（默认 3）、`--max-perspective`（默认 3）、`--search-top-k`（默认 3）、`--retrieve-top-k`（默认 3）、`--remove-duplicate`。

**跑本地开源模型（VLLM + Mistral）**（`examples/storm_examples/README.md:11-24`）：

```bash
python examples/storm_examples/run_storm_wiki_mistral.py \
    --url $URL --port $PORT --output-dir $OUTPUT_DIR \
    --retriever you --do-research --do-generate-outline \
    --do-generate-article --do-polish-article
```

**用自己的语料（VectorRM）**（`examples/storm_examples/README.md:33-58`）——语料需为 CSV，列必须含 `content`，可选 `title`/`url`/`description`；`url` 作为唯一标识：

```bash
python examples/storm_examples/run_storm_wiki_gpt_with_VectorRM.py \
    --output-dir $OUTPUT_DIR \
    --vector-db-mode offline \
    --offline-vector-db-dir $OFFLINE_VECTOR_DB_DIR \
    --csv-file-path $CSV_FILE_PATH \
    --device cuda \
    --do-research --do-generate-outline --do-generate-article --do-polish-article
```

**跑 Co-STORM**（`README.md:257-264`）：需要额外配 `BING_SEARCH_API_KEY` 和 `ENCODER_API_TYPE` 到 `secrets.toml`，然后 `python examples/costorm_examples/run_costorm_gpt.py --output-dir $OUTPUT_DIR --retriever bing`。

**输出目录约定**（文档字符串在 `run_storm_wiki_gpt.py:10-19`，最完整的一处说明）：

```
args.output_dir/<topic_name>/
    conversation_log.json           # 信息寻求对话日志
    raw_search_results.json         # 搜索引擎原始结果
    direct_gen_outline.txt          # 纯参数知识生成的提纲（阶段2第一遍）
    storm_gen_outline.txt           # 用检索素材重构后的提纲（阶段2第二遍）
    url_to_info.json                # 最终用到的来源
    storm_gen_article.txt           # 正文
    storm_gen_article_polished.txt  # 润色后正文
    run_config.json                 # post_run() 产物
    llm_call_history.jsonl          # post_run() 产物
```

---

## 值得抄的工程手法

### 1. 「先不看资料写一遍，再带着资料改一遍」的双遍提纲

- **它怎么做**：`WriteOutline.forward()`（`outline_generation.py:84`）先用不含任何检索素材的 `WritePageOutline`（`:128`）产出 `old_outline`，再把它连同对话历史喂给 `WritePageOutlineFromConv`（`:153`）产出终版。两份都落盘，`engine.py:248-253` 分别写成 `storm_gen_outline.txt` 和 `direct_gen_outline.txt`。
- **为什么好**：便宜提纲作为「先验锚点」，把 LLM 从「面对一大堆杂乱素材不知从何下手」里解放出来，变成「在现有结构上补漏」这个更容易的任务。同时两份产物天然构成消融对照基线，能量化"检索到底贡献了多少"。
- **代码在哪**：`knowledge_storm/storm_wiki/modules/outline_generation.py:84-125`

### 2. 树形 Article + 两阶段引用编号重整

- **它怎么做**：文章表达成 `ArticleSectionNode` 树（`interface.py:136`），引用编号分两步收敛——章节级 `_merge_new_info_to_references()`（`storm_dataclass.py:174`）先做局部→全局映射并剔除未被引用的素材，全文级 `reorder_reference_index()`（`:374`）再按前序出现顺序重排。幻觉出的越界编号由 `update_section()`（`:249`）里的正则扫描直接删除。
- **为什么好**：并发生成各章节时，每章的 `[1][2]` 都从 1 开始，天然冲突；集中重整比在 prompt 里要求模型"记住全局编号"可靠得多。这是用**后处理确定性**换取**生成灵活性**的典型取舍。
- **代码在哪**：`knowledge_storm/storm_wiki/modules/storm_dataclass.py:174-412`

### 3. 用「命名约定 + 反射」自动套 AOP 埋点

- **它怎么做**：`LMConfigs` 规定所有 LLM 字段必须以 `_lm` 结尾，所有 Retriever 字段以 `_rm` 结尾（`interface.py:266` 注释）。`LMConfigs.init_check()`（`:436`）/`collect_and_reset_lm_usage()`（`:452`）用 `if "_lm" in attr_name` 反射遍历 self.\__dict\_\_ 完成检查与汇总；`Engine.apply_decorators()`（`:512`）用 `dir(self)` 找出所有 `run_*` 方法自动套上计时装饰器。
- **为什么好**：新增一个 `xxx_lm` 字段或 `run_yyy_module()` 方法时，不需要改任何埋点代码。代价是约定不写在类型系统里、改错名字会静默失效——所以配了 `init_check()` 在构造时打 warning。
- **代码在哪**：`knowledge_storm/interface.py:427-473`（LMConfigs 反射）、`512-522`（apply_decorators）、`knowledge_storm/storm_wiki/engine.py:208`（调用点）

### 4. 两级 LLM 缓存（LRU + 磁盘）

- **它怎么做**：`functools.lru_cache(maxsize=3000)` 包在函数级（`:116`、`:126`），同时 `litellm.cache = Cache(disk_cache_dir=~/.storm_local_cache, type="disk")`（`:42`）。cache key 是 `ujson.dumps(dict(model=..., messages=..., **kwargs))` 的序列化串。
- **为什么好**：调试流水线时 90% 的时间在反复重跑同一批 call。这两层缓存让「改了阶段 3 重跑」的成本接近零。而且因为是依据完整 request 序列化做 key，改个 temperature 都会自动失效，语义正确。
- **代码在哪**：`knowledge_storm/lm.py:42-43`（磁盘缓存初始化）、`115-129`（LRU 包装）、`233`（调用点）

### 5. 全流程中间产物落盘 + 断点续跑

- **它怎么做**：每个阶段跑完立即 dump（`engine.py:228-273`），四个 `do_*` 开关关闭时能对应的 `_load_*_from_local_fs`（`:312`/`:320`/`:326`）从这些文件回填上下文。
- **为什么好**：几十分钟的流水线，第 4 步挂了不用从第 1 步重烧钱；同时 artifact 本身就是最好的调试现场——`conversation_log.json` 能直接看出"问了什么问题"，这比看最终文章有用得多。
- **代码在哪**：`knowledge_storm/storm_wiki/engine.py:211-339`

---

## 阅读路径

按这个顺序读，每一步都为下一步铺路：

| # | 文件 | 为什么现在读它 |
|---|---|---|
| 1 | `README.md`（Overview + How STORM works + Installation 段，约 L28-L90） | 先把「两阶段 / 两种提问策略」的心智模型建起来，再看代码才不会迷失。别跳过那段 `Overview` 的图说明 |
| 2 | `knowledge_storm/interface.py`（**全文件，608 行**） | 这是整个仓库的地基。`Information` / `Article` / `ArticleSectionNode` / `Retriever` / 四个 Module ABC / `Engine` + `apply_decorators` 全在这。读完它，后面每个 module 都只是在填同一个槽位 |
| 3 | `knowledge_storm/storm_wiki/engine.py`（**全文件，441 行**） | 唯一的主控编排。看 `run()`（`:341`）如何串起四个阶段，以及 `__init__`（`:174`）如何注入 lm/rm。读到这里你就拿到了完整骨架 |
| 4 | `knowledge_storm/storm_wiki/modules/knowledge_curation.py`（393 行） | 阶段 1，**STORM 的核心创新点**。重点看 `ConvSimulator.forward()`（`:47`）的多轮循环和 `WikiWriter.forward()`（`:95`）的上下文截断策略 |
| 5 | `knowledge_storm/storm_wiki/modules/outline_generation.py`（167 行） | 阶段 2，最短的一个模块但设计最巧（双遍提纲）。配合第 4 步理解"对话历史怎么变成提纲" |
| 6 | `knowledge_storm/storm_wiki/modules/article_generation.py`（176 行） | 阶段 3，看 `generate_article()`（`:53`）如何用 `add_hashtags` 的两种取值分别喂检索和 LLM |
| 7 | `knowledge_storm/storm_wiki/modules/article_polish.py`（102 行） | 阶段 4，收尾。注意 `show_guidelines=False` 的兼容性注释 |
| 8 | `knowledge_storm/storm_wiki/modules/storm_dataclass.py`（504 行） | 现在回头看数据层，因为你已经见过它被怎么用。重点按 「引用编号重整」这条线读：`_merge_new_info_to_references`（`:174`）→ `update_section`（`:249`）→ `reorder_reference_index`（`:374`）→ `post_processing`（`:502`） |
| 9 | `knowledge_storm/lm.py` 只读 **L1-L270**（`LM` 基类 + `LitellmModel`）+ `knowledge_storm/rm.py` 只读 **L1-L80**（`YouRM`） | 只看这两段就够理解适配层的契约。rm.py 剩下 1100 行是各家 API 的差异实现，用到哪家再读哪家 |
| 10 | `examples/storm_examples/run_storm_wiki_gpt.py`（242 行） | 回头看组装方式 + 文档字符串里的输出目录约定。**然后实际跑一次**，这是最有价值的一步 |

**可以跳过的**：`knowledge_storm/collaborative_storm/`（Co-STORM 是另一条产品线，经典 STORM 主线读完后如果想深入多 Agent 话语轮换再回来）；`knowledge_storm/logging_wrapper.py`；`frontend/demo_light/`（Streamlit 演示，不是核心）；`defunct_*` 之类的历史遗留文件在这个仓库没有对应物（那是 generative_agents 那边的问题）。

---

## 别踩的坑

### 维护状态

- **本地快照最后一次提交：`2025-09-30 11:07:17 -0700`**，commit `fb951af`（Merge PR #400 `patch/requirement-change`）。截至本文写作日期，距今约 **13 个月**。这段时间里 LLM 生态（尤其 litellm 与各家模型代号）变动极大。
- **快照完整性**：本地 `.git` 是 **shallow clone**（`.git/shallow` 存在，`git rev-list --count HEAD == 1`）。所以无法从这里判断上游是否还有更新——**这个"13 个月"是快照时间，不一定是上游最后一次提交时间**。入直接使用请自行核对上游。【待核：上游 HEAD 与真实活跃度】
- 从 README 的 Latest News 看，最后一次实质性功能更新是 **2025/01 的 litellm 集成**；此后主要是在修依赖。

### 跑起来要做什么改造

1. **`dspy_ai==2.4.9` 是硬钉版**（`requirements.txt:1`）。DSPy 本身 API 演进很快，`runs_on` 的 `2.4.9` 距今已一年多；`dsp.modules.hf`、`dsp.modules.hf_client`（`lm.py:15-16` 的 import）这类**旧命名空间在 DSPy 新版本里已经改地方了**。想升级 dspy，必须先确认 `lm.py` 顶部那批 `from dsp...` import 还能解析。保守做法是按 requirements 原样装。
2. **`scikit-learn` 是隐式依赖**（最容易踩的一条）。`storm_dataclass.py:8` 明确写着 `from sklearn.metrics.pairwise import cosine_similarity`，同时 `:7` 是 `from sentence_transformers import SentenceTransformer`；但 `requirements.txt` 里**只有 `sentence-transformers`，没有显式列出 `scikit-learn`**。它靠 sentence-transformers 间接带入。一旦你精简依赖或换安装方式，会直接 ImportError。
3. **`wikipedia==1.4.0` 声明了但从未使用**（已 grep 全仓确认无 `import wikipedia`）。别为它费心，也别指望它能提供什么能力。
4. **默认 example 走的是 deprecated 的 LM 路径**：`run_storm_wiki_gpt.py:30` import 的是 `OpenAIModel, AzureOpenAIModel`，而 `lm.py:272` 明确标注这些类在 v1.1.0 后 deprecated。想用 litellm（README 推荐的现代路径），得自己按 README 的 Python snippet 组装 `LitellmModel`。
5. **版本号不一致**：`setup.py` 写 `version="1.1.1"`，而 `knowledge_storm/__init__.py` 末尾写 `__version__ = "1.1.0"`。别拿错 version 判断功能集。
6. **`Engine.init_check()` 是软性警告不抛异常**（`interface.py:436-441`）。5 个 `_lm` 槽位没配全时只会打 warning，流水线会继续跑到某个环节才炸。调试时留意那行 warning。
7. **`max_thread_num` 与限流**。`STORMWikiRunnerArguments` 默认 `max_thread_num=10`（`engine.py:163`），但 `run_storm_wiki_gpt.py` 的 CLI 默认值是 **3**（`:166`）。默认脚本保守是有原因的——argparse help 原文写着 `Consider reducing it if keep getting "Exceed rate limit" error when calling LM API.`。两处默认值不一致，别以为改了类的默认值就会生效。
8. **Python 版本上限**。`setup.py:29` 声明 `python_requires=">=3.10"`，classifier 只列到 3.11，README 示例明确 `conda create -n storm python=3.11`。3.12 / 3.13 没有测试凭据，不建议冒进。
9. **CPU/GPU 预期**：阶段 3 的 `prepare_table_for_retrieval()`（`storm_dataclass.py:109`）会在本地加载 `paraphrase-MiniLM-L6-v2` 并一次性编码全部 snippet。纯 CPU 环境首次调用会明显卡顿，语料大时内存占用不低。VectorRM 路径还需要通过 `--device` 指定 `mps / cuda / cpu`（见 `examples/storm_examples/README.md`）。

### 结论

**不要指望 `pip install -r requirements.txt` 之后直接无痛跑通**。最可能卡住的三个点：

- (a) `dspy_ai==2.4.9` 硬钉版与现代 Python 组合下，能否找到匹配的 wheel；
- (b) 隐式依赖 `scikit-learn`（见上面第 2 条）；
- (c) `from dsp.modules.hf import ...` 这批旧命名空间 import（`lm.py:15-16`）。

建议先用虚拟环境隔离，`pip install knowledge-storm` 装官方发布的 wheel（`setup.py` 标 1.1.1，依赖组合是作者验过的），跑通之后再决定是否切到源码编辑。一旦跑通，`llm_call_history.jsonl` 和 `conversation_log.json` 会是最有用的两份调试材料——后者直接告诉你「它对着搜来的东西问了什么问题」，这是判断输出质量唯一可靠的依据。

---
