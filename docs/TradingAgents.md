# TradingAgents 技术解析

> 本地快照说明：本地副本是 **v0.7.1**（`tradingagents/__init__.py:3 __version__`），CHANGELOG 顶部 `## [0.7.1] — 2026-10-27`。这已经**远离** arXiv 2412.20138 论文配套的原始代码：`llm_clients/`、`memory/`（结算 + 反思）、`portfolio.py`、`backtest.py`、`report_html.py`、多厂商数据路由全是后加的。本文只写本地代码里真实存在的东西，涉及论文原始实现的差异统一放在"别踩的坑"。

## 一句话定位

TradingAgents 是一个 **用 LangGraph 把"真实交易公司"的组织结构编码成状态图**的多智能体框架：分析师团队（技术/情绪/新闻/基本面）并行取证 → 多空研究员辩论 → 研究经理收敛成投资计划 → 交易员转成具体交易提案 → 激进/保守/中性三位风控辩论 → 组合经理拍板出 5 档评级（Buy/Overweight/Hold/Underweight/Sell）。

在 Agent 技术栈里它属于 **应用层（垂直 Agent 应用）+ 编排层的胶合部**：编排完全交给 LangGraph（`StateGraph` + 条件边 + 子图 + checkpoint），自己只贡献"角色定义、prompt、状态字段、数据工具"这四样。它的价值不在框架代码（很少），**在 prompt 工程和防幻觉的工程细节上**。

## 事实卡片

| 项 | 值 |
| --- | --- |
| 主语言 | Python（`pyproject.toml` `requires-python = ">=3.11"`；README 建议 3.13） |
| 代码行数 | Python 184 个文件 / 28791 行。其中 `tradingagents/` 83 个 / 10710 行，`cli/` 10 个 / 2861 行，`tests/` 90 个 / 15204 行 |
| 源文件数 | 192 |
| 主要依赖 | `langgraph>=1.2.12`、`langgraph-checkpoint-sqlite>=3.1.1`、`langchain-core>=1.6.5`、`langchain-openai>=1.6.6`、`langchain-anthropic>=1.7.4`、`langchain-google-genai>=4.4.0`、`yfinance>=1.7.0`、`stockstats>=0.6.5`、`pandas>=3.0.6`、`typer>=0.21.0`、`rich>=14.0.0`、`questionary>=2.1.0`、`python-dotenv`；可选 `bedrock`（`langchain-aws`） |
| 许可证 | Apache-2.0（LICENSE 文件头；pyproject 未声明 license 字段） |
| 本地快照 commit | `c454491 Merge pull request #1613 from TauricResearch/v0.7.1` |
| star 数 | 【待核】 |
| 版本 / 入口 | `0.7.1`；`pyproject [project.scripts] tradingagents = "cli.main:app"`；另有 `main.py` 做 7 行最小示例 |

## 目录地图

`tradingagents/` 下三个核心包的职责分工：

| 路径 | 作用 |
| --- | --- |
| `tradingagents/agents/` | **角色定义层**。`analysts/`（market / sentiment / news / fundamentals 四个分析师 + `turn.py` 的 `take_turn`）、`researchers/`（bull / bear）、`risk_mgmt/`（aggressive / conservative / neutral）、`managers/`（research_manager / portfolio_manager）、`trader/`；配套的 `state.py`（`AgentState`）、`context.py`（prompt 公共片段）、`tools.py`（LangChain `@tool` 定义）、`schemas.py`（Pydantic 结构化输出）、`structured.py`（结构化输出 + 降级）、`rating.py`（5 档评级解析）、`post_screen.py`（Jev 社交帖筛选） |
| `tradingagents/graph/` | **编排层**。`setup.py`（`GraphSetup` 搭 `StateGraph`）、`conditional_logic.py`（`ConditionalLogic` 辩论/风控轮次路由）、`propagation.py`（`Propagator` 初始状态 + `recursion_limit`）、`trading_graph.py`（`TradingAgentsGraph` 门面：建图、跑图、落报告、写记忆、checkpoint）、`analyst_execution.py`（分析师节点规格表）、`checkpointer.py`（SQLite per-ticker） |
| `tradingagents/dataflows/` | **数据层**。`router.py`（工具名 → 厂商实现的路由表 + `TOOLS_CATEGORIES`）、`config.py`（ContextVar 承载的 per-run 配置）、`date_window.py`（point-in-time 日期裁剪 `as_of` / `as_of_window`）、`symbols.py`（ticker 安全校验）、`files.py`（原子写 + 跨进程文件锁）、`net.py`（厂商可达性探测）、`vendors/`（yahoo / alpha_vantage / fred / sec_edgar / reddit / stocktwits / polymarket） |

其余：

| 路径 | 作用 |
| --- | --- |
| `tradingagents/llm_clients/` | 多厂商 LLM 客户端：`openai_client.py`（411 行，兼容 OpenAI / DeepSeek / Grok / Qwen / GLM / MiniMax / OpenRouter / Mistral / Kimi / NVIDIA / vLLM / LM Studio / Ollama）、`anthropic_client.py`、`google_client.py`、`azure_client.py`、`bedrock_client.py`、`chatgpt_auth.py`（ChatGPT Plus/Pro 订阅登录）、`factory.py`（`create_llm_client` / `create_tier_client` / `tier_provider`）、`model_catalog.py`、`capabilities.py`、`api_key_env.py` |
| `tradingagents/memory/` | `log.py`（`TradingMemoryLog`，Markdown 记忆文件 `~/.tradingagents/memory/trading_memory.md`）、`settlement.py`（到持仓期后结算盈亏与 alpha）、`reflection.py`（`Reflector` 生成一段反思） |
| `tradingagents/backtest.py` | `iter_grid` / `run_backtest` / `summarize`，按日期网格扫历史跑分 |
| `tradingagents/portfolio.py` | `PortfolioContext`，调用方持仓与现金，供交易员/风控/组合经理做仓位 sizing |
| `tradingagents/reporting.py` + `report_html.py` | 报告树落盘 + 单页 HTML |
| `cli/` | Typer + Rich + questionary 的交互式终端：`main.py`（`analyze` / `backtest` / `login` / `logout`）、`run.py`、`prompts.py`、`display.py`、`prefs.py`、`selections.py`、`models.py`、`stats_handler.py`、`updates.py` |
| `main.py` | 7 行最小示例：`TradingAgentsGraph(debug=True).propagate("NVDA", "2026-09-01")` |
| `tests/` | 90 个测试文件 / 15204 行，`pyproject` 配了 `-m "not integration"` 默认跳过集成测试 |

## 核心架构

```
                        cli/main.py (typer)   or   main.py   or   Python API
                                   |                                  |
                                   v                                  v
                    +--------------------------------------------------------------+
                    | tradingagents/graph/trading_graph.py                          |
                    | class TradingAgentsGraph                                       |
                    |  - create_tier_client(quick) -> analysts/researchers/trader    |
                    |  - create_tier_client(deep)  -> research/portfolio managers    |
                    |  - propagate() / stream_run() / save_reports() / record_decision() |
                    +--------+-------------------------------+-----------------------+
                             |                               |
                             v                               v
          +----------------------------------+   +--------------------------------+
          | graph/setup.py  GraphSetup       |   | graph/propagation.py Propagator|
          |  StateGraph(AgentState)          |   |  create_initial_state()        |
          +----------------------------------+   +--------------------------------+
                             |
       START ---> [analysts 并行, 各带自己的子图] ---> Memory Log(并行)
                             |  (全部报告就绪)
                             v
                     Bull Researcher <----> Bear Researcher      (ConditionalLogic
                             |                                    should_continue_debate)
                             v
                     Research Manager  (deep LLM, 结构化 ResearchPlan)
                             v
                     Trader            (结构化 TraderProposal)
                             v
          Aggressive <-> Conservative <-> Neutral                (should_continue_risk_analysis)
                             v
                     Portfolio Manager (deep LLM, 结构化 PortfolioDecision)
                             v
                            END

  横向能力：
   agents/tools.py (@tool)  ->  dataflows/router.py route_to_vendor()
                                     -> vendors/{yahoo,alpha_vantage,fred,sec_edgar,
                                                 reddit,stocktwits,polymarket}
                                     -> date_window.as_of() 做 point-in-time 裁剪
                                     -> data_cache_dir CSV/JSON 缓存

   memory:  run 结束 -> record_decision() -> TradingMemoryLog.store_decision()
            下次 run  -> Memory Log 节点 -> settle_all_pending() + get_past_context()
                                        -> past_context 注入 Portfolio Manager

   checkpoint: graph/checkpointer.py  SqliteSaver per ticker
               ~/.tradingagents/cache/checkpoints/<TICKER>.db
```

## 关键抽象与执行流

### 1. 状态：`AgentState`

`tradingagents/agents/state.py`。三个 TypedDict 叠加：

- `InvestDebateState`（第 8 行）：`bull_history` / `bear_history` / `history` / `current_response` / `count`。
- `RiskDebateState`（第 21 行）：`aggressive_history` / `conservative_history` / `neutral_history` / `history` / `latest_speaker` / 三个 `current_*_response` / `count`。
- `AgentState(MessagesState)`（第 45 行）：`company_of_interest` / `asset_type` / `instrument_context` / `trade_date` + 四份报告（`market_report` / `sentiment_report` / `news_report` / `fundamentals_report`）+ `investment_debate_state` / `investment_plan` / `trader_investment_plan` / `risk_debate_state` / `final_trade_decision` / `final_rating` / `past_context` / `memory_note` / `portfolio_context`。

**设计要点**：报告是**独立字段**而不是塞进 `messages`，所以四个分析师可以并行各写各的 key，不会互相覆盖。辩论的"对话"则是每个节点手写字符串拼接（`history + "\n" + argument`），不依赖 LangGraph 的 message reducer。

### 2. 建图：`GraphSetup.setup_graph`

`tradingagents/graph/setup.py:109`。

- **分析师各自编译成一张子图**（第 53-90 行 `_analyst_graph`）：`StateGraph(AgentState, output_schema=TypedDict(f"{key}Report", {report_key: str}))`，`agent → (tools | END)`，工具轮次到 `max_tool_rounds` 后走 `wrap_up` 节点强制出报告（第 76-83 行，解决 issue #1420 的递归爆炸）。子图只把 `report_key` 一个字段吐回父图——这就是"并行分析师互不干扰"的实现方式。
- 节点规格表在 `graph/analyst_execution.py:20 ANALYST_NODE_SPECS`：`market → Market Analyst / market_report / market_analyst.TOOLS`，`social → Sentiment Analyst`（注释说明它预先取数所以**没有 tools**，第 28-32 行），`news`、`fundamentals` 同理。
- 起始边（第 160-166 行）：所有分析师 + `Memory Log` 都从 `START` 出发并行，`workflow.add_edge(first_steps, "Bull Researcher")` —— **LangGraph 的 fan-in 语义保证最慢的分析师完成后才进入辩论**。
- 辩论条件边（第 169-183 行）用两份完整 path_map（`DEBATE_PATH_MAP` 第 35 行、`RISK_ANALYSIS_PATH_MAP` 第 40 行），注释写明是为了防止 prompt/i18n 漂移导致路由落到不存在的节点而崩图（issue #1088）。

### 3. 辩论与共识：靠"轮次计数 + 最新发言者"路由

`tradingagents/graph/conditional_logic.py`（34 行，极简）：

- `should_continue_debate`（第 12 行）：`count >= 2 * max_debate_rounds` → `"Research Manager"`；否则看 `current_response` 是否以 `"Bull"` 开头 → 交给 `"Bear Researcher"`，反之交给 `"Bull Researcher"`。
- `should_continue_risk_analysis`（第 23 行）：`count >= 3 * max_risk_discuss_rounds` → `"Portfolio Manager"`；否则按 `latest_speaker` 前缀在 Aggressive → Conservative → Neutral 之间循环。

**"共识"的实现方式值得注意**：没有投票、没有打分、没有一致性判定。**收敛完全靠两个 prompt 指令**：`research_manager.py:36` 和 `portfolio_manager.py:66` 都写了同一段——"The debate always contains conflicting arguments; deciding which side is stronger is the job, so conflict alone is not a reason to Hold. Commit to the side with the stronger case... Choose Hold only when the evidence is still balanced"。（这是针对模型一看到分歧就无脑输出 Hold 的补丁。）

辩论本身也不共享 `messages`，而是每个研究员把 `argument = f"Bull Analyst: {response.content}"` 拼进 `history`（`bull_researcher.py:53-61`），下一个人读全量 history 并针对 `current_response` 反驳。

**开场处理**：`tradingagents/agents/context.py:37 opponent_argument_or_opening()` —— 第一轮对方还没发言时，不插值空字符串（会让模型**编造**对方的论点），而是返回 `"(The {opponent} has not spoken yet — open the debate with your own case.)"`（issue #1176）。同理 `report_or_absent()`（第 176 行）让没选中的分析师报告变成 `"(No {source} report in this run: it is not available, not an empty finding.)"`，避免下游把"缺失"读成"空白结论"。

### 4. 各角色分工与 prompt

| 角色 | 文件 | 模型档 | 输入 | 输出 | prompt 要点 |
| --- | --- | --- | --- | --- | --- |
| Market Analyst | `agents/analysts/market_analyst.py:15` | quick | 3 个工具（OHLCV / 指标 / verified snapshot） | `market_report` | 第 22-50 行给了**完整指标清单**（SMA/EMA/MACD/RSI/Boll/ATR/VWMA）含每个指标的用途与坑；强制"最多选 8 个、先调 `get_stock_data` 再调 `get_indicators`、报告末尾附 Markdown 表"；第 48 行要求以 `get_verified_market_snapshot` 为唯一真值源 |
| Sentiment Analyst | `agents/analysts/sentiment_analyst.py:44` | quick | 预取 Yahoo 新闻 + StockTwits + Reddit（无 tools） | `sentiment_report` | 模块 docstring 第 1-19 行说明三源与"社交 feed 不归档，历史 run 的情绪输入不是 point-in-time"；可选经 `post_screen.jev_screen()` 用 TypeSafe Jev 筛帖 |
| News Analyst | `agents/analysts/news_analyst.py:20` | quick | `get_news` / `get_global_news` / `get_macro_indicators` / `get_prediction_markets` | `news_report` | 第 25 行 prompt 里直接教工具签名，并举例 FRED 指标名（`cpi`、`core_pce`、`unemployment`、`fed_funds_rate`、`10y_treasury`、`yield_curve`） |
| Fundamentals Analyst | `agents/analysts/fundamentals_analyst.py:21` | quick | 5 个财报/内部人工具 | `fundamentals_report` | 第 28 行要求"尽可能详细 + 末尾 Markdown 表" |
| Bull Researcher | `agents/researchers/bull_researcher.py:9` | quick | 四份报告 + debate history + 上一轮 bear 论点 | `investment_debate_state` | 第 31-48 行：Growth Potential / Competitive Advantages / Positive Indicators / **Bear Counterpoints** / Engagement（要求对话式反驳而非罗列数据） |
| Bear Researcher | `agents/researchers/bear_researcher.py:9` | quick | 同上（反驳 bull） | 同上 | 第 31 行起：Risks / Competitive Weaknesses / Negative Indicators / Bull Counterpoints |
| Research Manager | `agents/managers/research_manager.py:14` | **deep** | debate history | `investment_plan` | 第 23-51 行：给了 5 档评级定义 + "冲突不是 Hold 的理由"那一段 + 输出三段式（Recommendation / Rationale / Strategic Actions） |
| Trader | `agents/trader/trader.py:20` | quick | `investment_plan` + market_report + portfolio_context | `trader_investment_plan` | 第 36-59 行：把价格锚定到技术报告的 ATR/支撑阻力；**明确要求 entry/stop 用绝对价格（如 189.5）而非百分比**（issue #1288，因为结构化解析会失败）；Overweight→Buy、Underweight→Sell |
| Aggressive Analyst | `agents/risk_mgmt/aggressive_debator.py:10` | quick | trader 提案 + 四份报告 + 另外两方最新论点 | `risk_debate_state` | 第 32-46 行：主动唱多高风险高回报，逐条反驳保守与中性，"Output conversationally as if you are speaking without any special formatting" |
| Conservative Analyst | `agents/risk_mgmt/conservative_debator.py:10` | quick | 同上 | 同上 | 第 31 行起：保护资产、最小化波动，反驳激进与中性 |
| Neutral Analyst | `agents/risk_mgmt/neutral_debator.py:10` | quick | 同上 | 同上 | 第 34 行起：同时批评激进"过于乐观"和保守"过于谨慎" |
| Portfolio Manager | `agents/managers/portfolio_manager.py:23` | **deep** | 风控辩论 history + research_plan + trader_plan + `past_context` + portfolio_context | `final_trade_decision` + `final_rating` | 第 42-76 行：5 档评级 + 三段输出（Rating / Executive Summary / Investment Thesis）；第 36-40 行把记忆教训拼成 `lessons_line` |

**公共 prompt 片段**都在 `agents/context.py`：`get_instrument_context_from_state`（第 172 行）、`get_language_instruction`（第 14 行，非英语时要求正文用该语言但 `**Rating**:` 标签行保持英文，否则评级解析会误读 negated rating，issue #1435）、`get_portfolio_context_from_state`（第 204 行）。

### 5. 结构化输出与降级

`tradingagents/agents/structured.py`：

- `NO_EXTERNAL_TOOLS`（第 36 行）是一句固定禁令，因为 `with_structured_output` 只绑定一个工具（schema 本身），模型若去调搜索工具会导致整次结构化调用作废（issue #1130）。
- `bind_structured`（第 42 行）= `llm.with_structured_output(schema)`，失败返回 `None`。
- `invoke_structured`（第 59 行）捕获所有异常返回 `None`（含"思考模型直接吐纯文本导致解析为空"的情况，第 70-74 行）。
- `invoke_structured_or_freetext`（第 84 行）先结构化，失败降级成 `plain_llm.invoke(prompt).content`。

Schema 在 `agents/schemas.py`：`ResearchPlan`（第 95 行）/ `TraderProposal`（第 146 行）/ `PortfolioDecision`（第 221 行）/ `SentimentReport`（第 311 行），各配一个 `render_*` 函数渲染回 Markdown。

### 6. 信号聚合与最终决策

1. `TradingAgentsGraph._run_graph`（`graph/trading_graph.py:373`）拿到 `final_state`。
2. `record_decision`（第 359 行）→ `run_rating(final_state)` → 写记忆。
3. `run_rating`（`agents/rating.py:106`）= `final_state["final_rating"] or parse_rating(final_trade_decision)`。
4. `parse_rating`（第 96 行）用三段正则从自由文本里抠评级：`_RATING_LINE_RE`（第 46 行）只认"决策自己的评级行"（允许 final/our/overall/my 等限定词，**排除** "Consensus rating" / "Trader's rating"），`_RATING_SCALE_RE`（第 53 行）排除模型回显的评级量表说明。抠不到就返回 `"REVIEW"`（第 30 行 `RATING_REVIEW`）。
5. `propagate()` 返回 `(final_state, signal)`，docstring（第 177-181 行）明确要求调用方**先 `is_review(signal)` 再映射到 `PortfolioRating` 枚举**——`REVIEW` 不是一个可交易档位。
6. 落盘：`_log_state`（第 438 行）写 `results_dir/<ticker>/TradingAgentsStrategy_logs/full_states_log_<date>.json`（含 `run_settings()` 允许清单，第 255 行，刻意不记 endpoint/key/本地路径）。

### 7. 数据源接入与缓存

- **工具定义**：`agents/tools.py` 用 `@tool` + `Annotated[..., InjectedState("trade_date")]`。关键设计：`symbol` 也从 `InjectedState("company_of_interest")` 注入（第 20 行），**模型骗不了工具去查别的票**；`trade_date` 注入后在 `date_window.as_of()` / `as_of_window()` 里裁剪，模型传未来日期也会被夹回（docstring 第 1-5 行）。
- **厂商路由**：`dataflows/router.py:97 VENDOR_METHODS` 是 `{工具名: {厂商名: 实现函数}}` 的二级表；`TOOLS_CATEGORIES`（第 45 行）定义 6 个类别；`OPTIONAL_CATEGORIES = {"macro_data", "prediction_markets"}`（第 94 行）——这两类失败降级为哨兵字符串而不是抛异常中断 run。优先级：`tool_vendors` > `data_vendors[category]` > 默认。
- **默认厂商**（`default_config.py:158-167`）：`core_stock_apis: yfinance`、`technical_indicators: yfinance`、`fundamental_data: sec_edgar,yfinance`（有序回退）、`news_data: yfinance`、`macro_data: fred`（需 `FRED_API_KEY`）、`prediction_markets: polymarket`（免 key）。
- **缓存**：
  - `data_cache_dir`（默认 `~/.tradingagents/cache`）下的 CSV / JSON。
  - `vendors/yahoo/ohlcv.py`：每 symbol 一个缓存文件存 5 年数据，`_cache_is_fresh`（第 146 行）判断新鲜度；第 196-205 行明确"空缓存/无 Close 列的缓存视为 miss 重新拉"（防止上一次限流失败被永久缓存）；第 221 行"只缓存真实数据，从不落盘空帧"。
  - `vendors/sec_edgar.py:123 _cached_json`：CIK 映射表和 facts 落盘。
  - 并发安全：`dataflows/files.py:13 replace_file()` 用 uuid 临时文件 + `os.replace` 原子替换（Windows 上被占用则跳过并告警）；`locked()`（第 40 行）提供 POSIX `flock` / Windows `msvcrt.locking` 的跨进程锁。
  - 失败语义：`vendors/yahoo/common.py:32 raise_for_empty()` 先 `vendor_reachable(YAHOO_HOST)` 探测，区分"这个票没数据"和"Yahoo 挂了"（`NoMarketDataError` vs `VendorUnavailableError`）。
- **point-in-time**：`dataflows/date_window.py` 的 `as_of` / `as_of_window` / `is_historical` 贯穿工具层与记忆层（`get_past_context(as_of=...)`、`settlement` 的 `resolution_date`）。

### 8. 记忆与跨 run 学习

- `memory/log.py:11 TradingMemoryLog`：决策写进 `~/.tradingagents/memory/trading_memory.md`（Markdown，人可读）。
- `get_past_context(ticker, n_same=5, n_cross=3, as_of=None)`（第 93 行）：取同标的最近 5 条 + 跨标的最近 3 条；`as_of` 过滤掉"结果尚未发生"的条目（issue #1251，防回测偷看未来）。
- `memory/settlement.py:60 fetch_returns`：`holding_period_days` 个交易日后的 raw return 与 alpha（基准由 `resolve_benchmark` 第 16 行按 ticker 后缀从 `benchmark_map` 选，默认 SPY）；**持仓窗口没走完就返回全 None，保持 pending 下次再试**（issue #1169）；返回 `resolution_date` 作为 point-in-time 截止。
- `memory/reflection.py:6 Reflector`：用 quick LLM 就一段已结算决策生成反思。
- 挂载点：`TradingAgentsGraph._memory_step`（`trading_graph.py:309`）作为**与分析师并行的图节点** `Memory Log`，先 `settle_all_pending(wait=False)` 再 `get_past_context`，任何异常都只降级成 `memory_note` 而不中断 run（第 319-336 行）。

### 9. Checkpoint / 断点续跑

`graph/checkpointer.py`：`thread_id(ticker, date, signature)`（第 28 行）、`get_checkpointer`（第 42 行，`SqliteSaver`，路径 `data_cache_dir/checkpoints/<TICKER>.db`）、`checkpoint_step`（第 54 行）、`clear_checkpoint`（第 85 行）。

`TradingAgentsGraph._run_signature`（`trading_graph.py:140`）把"分析师选择 / 辩论轮数 / 风控轮数 / asset 类型 / portfolio 指纹 / 图布局标签 / 其余配置的 sha256"拼成签名——**换任何一个配置就自动开新线程而不是静默续跑旧状态**（issue #1089）。`checkpoint_input`（第 221 行）续跑时传 `None`（传初始 state 会被 message reducer 重复追加，issue #1249）。

### 10. 配置

`tradingagents/default_config.py`：`build_default_config()`（第 77 行）+ `_ENV_OVERRIDES`（第 10-34 行）一张表把 `TRADINGAGENTS_*` 环境变量映射到配置键，`_coerce`（第 41 行）按默认值类型强转且**拒绝静默回退**（拼错 `true` 会启动即报错）。可覆盖 `TRADINGAGENTS_RESULTS_DIR` / `CACHE_DIR` / `MEMORY_LOG_PATH`。

`dataflows/config.py` 用 `ContextVar` 实现 **per-run 配置隔离**（第 13 行）：`run_config()`（第 44 行）和 `run_config_context()`（第 54 行）让同一进程里多个 graph 各自读自己的厂商配置，LangGraph 会把 context 带进 tool call。

## 最小可运行入口

全部摘自 README.md（行号已标注）：

```bash
# 1. 安装（Python 3.11+，README 建议 3.13）            README.md:126-140
git clone https://github.com/TauricResearch/TradingAgents.git
cd TradingAgents
conda create -n tradingagents python=3.13 && conda activate tradingagents
# 或 uv venv --python 3.13 && source .venv/bin/activate
pip install .            # uv 用户用 uv pip install .

# 2. API key（择一）                                    README.md:164-183
export OPENAI_API_KEY=...          # OpenAI (GPT)
export GOOGLE_API_KEY=...          # Google (Gemini)
export ANTHROPIC_API_KEY=...       # Anthropic (Claude)
export FRED_API_KEY=...            # FRED 宏观数据（免费，可选）
export ALPHA_VANTAGE_API_KEY=...   # Alpha Vantage
# 另有 XAI / DEEPSEEK / DASHSCOPE / DASHSCOPE_CN / ZHIPU / ZHIPU_CN /
#      MINIMAX / MINIMAX_CN / OPENROUTER / MISTRAL / MOONSHOT / GROQ / NVIDIA
# 或 cp .env.example .env 后填（README.md:199-201）

# 3. 交互式 CLI                                          README.md:206-209
tradingagents          # 安装后的命令
python -m cli.main     # 从源码直接跑

# 4. 非交互（定时任务 / 脚本）                            README.md:212-216
export TRADINGAGENTS_LLM_PROVIDER=openai TRADINGAGENTS_QUICK_THINK_LLM=gpt-6-luna TRADINGAGENTS_DEEP_THINK_LLM=gpt-6-sol
export TRADINGAGENTS_OUTPUT_LANGUAGE=English TRADINGAGENTS_MAX_DEBATE_ROUNDS=1 TRADINGAGENTS_MAX_RISK_ROUNDS=1
tradingagents --ticker NVDA --date 2026-09-23 --analysts market,news,fundamentals --save --no-show
# 其他 flag：--checkpoint / --no-checkpoint、--clear-checkpoints、
#           --portfolio my_book.json、--html / --no-html
# 子命令：tradingagents backtest / login chatgpt / logout chatgpt

# 5. Docker                                              README.md:143-157
cp .env.example .env
docker compose run --rm tradingagents
mkdir -p data && TRADINGAGENTS_DATA_DIR=./data docker compose run --rm tradingagents
docker compose --profile ollama run --rm tradingagents-ollama      # 本地模型
```

Python API（`main.py` + README.md:258-296）：

```python
from tradingagents.graph.trading_graph import TradingAgentsGraph
from tradingagents.default_config import DEFAULT_CONFIG

ta = TradingAgentsGraph(debug=True, config=DEFAULT_CONFIG.copy())
state, decision = ta.propagate("NVDA", "2026-09-01")
print(decision)
ta.save_reports(state, "NVDA")            # 默认写到 results_dir/reports
```

ticker 规则（README.md:220-230）：US `AAPL`；香港 `0700.HK`；东京 `7203.T`；伦敦 `AZN.L`；印度 `RELIANCE.NS`；A 股 `600519.SS` / `.SZ`；加密 `BTC-USD`。

## 值得抄的工程手法

1. **给每个分析师套一层私有子图，用 `output_schema` 隔离写入**
   - 它怎么做：`_analyst_graph`（`graph/setup.py:53`）给每个分析师编译一张子图，`output_schema=TypedDict(f"{key}Report", {report_key: str})`，只把报告字段吐回父图；父图 `START → [全部分析师]` 并行，靠 LangGraph fan-in 汇合。
   - 为什么好：并发 LLM 节点最容易踩的坑就是"多个分支同时写同一个 state key"。用子图 + 窄 output_schema 从类型层面消灭了这个问题，同时每个分析师的工具调用历史不会污染别人的 messages。
   - 代码在：`tradingagents/graph/setup.py:53-90`、`:160-166`；规格表 `graph/analyst_execution.py:20`。

2. **工具签名里注入 state，而不是让模型传**
   - 它怎么做：`@tool def get_stock_data(symbol: Annotated[str, InjectedState("company_of_interest")], ..., trade_date: Annotated[str, InjectedState("trade_date")] = "")`，再用 `as_of_window(start, end, trade_date)` 把模型给的日期夹回 run 日期之前。
   - 为什么好：三件事一次性解决——模型没法查别的标的、没法偷看未来数据、省掉了两个参数。`get_verified_market_snapshot` 更进一步：给模型一个"确定性真值源"，并要求它和其他工具输出冲突时**上报差异而不是自己编一个调和值**（`market_analyst.py:48`）。
   - 代码在：`agents/tools.py:18-38`、`dataflows/date_window.py`、`vendors/yahoo/snapshot.py`。

3. **把"缺失"显式化，而不是留空**
   - 它怎么做：`context.py:37 opponent_argument_or_opening()` 在对手还没发言时返回 `(The X has not spoken yet — open the debate with your own case.)`；`context.py:190 report_or_absent()` 返回 `(No {source} report in this run: it is not available, not an empty finding.)`；`get_portfolio_context_from_state`（第 204 行）没给持仓时明确说"你不知道调用方的持仓，不要假设空仓"。
   - 为什么好：LLM 面对空字符串会**自动补全**。把空值换成一句"这里是空的，别脑补"的元信息，能消掉一大类幻觉。这几处注释都标了 issue 号（#1176、#1176、#814），是真实 bug 驱动出来的。
   - 代码在：`tradingagents/agents/context.py:37`、`:190`、`:204`。

4. **结构化输出 + 一次自由文本兜底，并把"读不出结果"设为 `REVIEW` 而不是 `Hold`**
   - 它怎么做：`structured.py` 的 `invoke_structured_or_freetext`；`rating.py` 的 `parse_rating(text, default=RATING_REVIEW)`，正则专门排除"别人家的 rating"和"评级量表回显"，并且要求所有找到的评级**必须一致**才采纳（第 96 行 `len(set(found)) == 1`）。
   - 为什么好：把"解析失败"降级成 Hold 是金融 Agent 最危险的默认行为——它会以"明确的持仓建议"的形式进入记忆、被下一次 run 引用。`REVIEW` 强迫调用方显式处理。docstring 第 13-16 行写得很直白："a Hold recorded in its place is quoted back to the next run as a call that was never made (#1170)."
   - 代码在：`agents/rating.py:22-116`、`agents/structured.py:59-95`。

5. **Checkpoint 的"运行签名"防串味**
   - 它怎么做：`_run_signature`（`trading_graph.py:140`）把图结构相关的配置（分析师集合、辩论轮数、asset 类型、portfolio 指纹、布局标签、其余配置 sha256）拼成签名，作为 `thread_id` 的一部分；`_NOT_IN_SIGNATURE`（第 46 行）只排除"文件存哪 / 是否 checkpoint / 重试次数"这三个不影响产物的键。
   - 为什么好：断点续跑最阴的 bug 是"换了配置却续跑了旧状态，拿到上一套参数产出的报告"。签名让这种续跑自动变成新跑。
   - 代码在：`trading_graph.py:140-165`、`graph/checkpointer.py:28`。

6. **per-run 配置用 ContextVar 而不是全局单例**
   - 它怎么做：`dataflows/config.py` 里 `_run_config: ContextVar[dict|None]`（第 13 行），`run_config()` 上下文管理器（第 44 行）与 `run_config_context()`（第 54 行，给 generator/stream 场景用），`get_config()` 优先读 scoped。
   - 为什么好：同一进程里跑多个 graph（比如 backtest 并发扫日期网格）时，各自用各自的厂商配置；LangGraph 会把 context 带进 tool call，不需要把 config 塞进 state。
   - 代码在：`tradingagents/dataflows/config.py`、`trading_graph.py:422`（`stream_run` 里 `context.run(...)`）。

7. **可选数据类别的"软失败"，核心类别的"硬失败"**
   - 它怎么做：`dataflows/router.py:94 OPTIONAL_CATEGORIES = {"macro_data", "prediction_markets"}`，这些类别厂商挂了返回哨兵字符串；价格/财报/新闻仍然抛异常。
   - 为什么好：明确区分"没有它这个决策就不成立"和"没有它只是少了点风味"，避免为了一条宏观数据把整轮分析搞崩。
   - 代码在：`dataflows/router.py:89-94`、`dataflows/errors.py`。

## 阅读路径

1. `main.py`（7 行）+ `README.md:206-296` —— 先跑通认知：一个 `TradingAgentsGraph`、一次 `propagate`、一个字符串 decision。
2. `tradingagents/agents/state.py`（74 行）—— 全系统的骨架。搞清"报告是独立字段、辩论是字符串拼接"这个核心取舍。
3. `tradingagents/graph/setup.py`（187 行）—— 图的形状全在这一个文件里：分析师子图、fan-in、两份 path_map。配合 `graph/analyst_execution.py:20` 看节点规格表。
4. `tradingagents/graph/conditional_logic.py`（34 行）+ `propagation.py`（81 行）—— 最短的两个文件，讲清"辩论/风控怎么轮转"和"初始状态怎么造"。
5. `tradingagents/agents/analysts/market_analyst.py`（85 行）—— 唯一一个带完整指标清单和"验证快照"要求的 prompt，是 prompt 工程密度最高的样本。再看 `analysts/turn.py`（43 行）的 `take_turn`，理解工具轮次与 `WRAP_UP`。
6. `tradingagents/agents/researchers/bull_researcher.py`（66 行）→ `risk_mgmt/aggressive_debator.py`（69 行）—— 两类辩论各看一个就够，其余三个是对称的。
7. `tradingagents/graph/trading_graph.py`（476 行）—— 门面层。重点读 `propagate`(167)、`_memory_step`(309)、`_run_graph`(373)、`stream_run`(407)、`_log_state`(438)。
8. `tradingagents/agents/rating.py`（117 行）—— 评级解析的正则与"为什么 REVIEW 不是 Hold"的推理，是这份代码里最值得单独摘出来的设计论证。
9. `tradingagents/agents/tools.py` + `dataflows/router.py` —— 工具层与厂商路由，看 `InjectedState` 与 `route_to_vendor` 怎么配合。
10. `tradingagents/memory/log.py:93 get_past_context` + `memory/settlement.py:60 fetch_returns` —— 跨 run 学习的两半：取教训、算结果。
11. `tradingagents/default_config.py`（209 行）—— 当字典查，把所有可调旋钮过一遍；配合 `dataflows/config.py` 看 per-run 隔离。
12. `tradingagents/backtest.py`（236 行）—— 想做历史回测时才读。

## 别踩的坑

**先说最要紧的一条：这不是论文那份代码**

- 论文（arXiv 2412.20138）配套的原始实现里，数据来自 **Finnhub / Simfin / Google News + 一个随仓库发布的离线数据集**，配置写在 `tradingagents/default_config.py` 之外的地方，也没有 `llm_clients/`、`memory/`、`portfolio.py`、`backtest.py`。本地 v0.7.1 已经把这些全换掉了：**数据全部走在线厂商**（yfinance / alpha_vantage / FRED / SEC EDGAR / Reddit / StockTwits / Polymarket），缓存落在 `~/.tradingagents/cache`。
- 因此**论文里的实验数字用这个分支复现不出来**：数据口径变了、默认模型变了、prompt 全改了（加了 5 档评级、结构化输出、instrument identity、point-in-time 裁剪等一堆补丁）。README.md:78 自己也写着 "Trading performance may vary based on many factors... It is not intended as financial, investment, or trading advice."
- 想对照论文读原始实现的具体文件路径与行号，本地副本无法提供，【待核】。

**版本 / 依赖**

- `pyproject.toml` 的 `requires-python = ">=3.11"`，但 README.md:128 建议用 **3.13**。用 3.11/3.12 理论上可行但不在官方测试主路径上。
- 依赖下限被拉得很高：`langchain-core>=1.6.5`、`langgraph>=1.2.12`、`pandas>=3.0.6`。**`langchain-core` 1.x 和 0.x 是不兼容的两代 API**（`with_structured_output`、`ChatPromptTemplate`、`ToolNode` 的行为都有变化），如果你机器上已有别的 langchain 项目，务必独立 venv，否则会遇到难以定位的 `bind_tools` / message 序列化问题。
- `pandas>=3.0.6` 是相当新的下限，很多老代码（`.append`、`inplace` 链）在 pandas 3.x 下会报错。
- `yfinance>=1.7.0` 是 Yahoo 的非官方抓取库，**Yahoo 一改接口就集体失效**。仓库为此写了相当多的防御代码（`vendors/yahoo/common.py` 的 `raise_for_empty`、`_answered_empty`、`vendor_reachable` 探测），说明这是常态而非意外。
- `docker compose` 镜像**不再打包任何 `.env` 文件**（CHANGELOG 0.7.1 "Upgrading from 0.7.0" 第一条），Azure OpenAI 用户必须把 key 挪到 `.env`；Docker 里跑 ChatGPT 订阅登录要用 `docker compose run --rm tradingagents login chatgpt` 并手动粘贴回调地址。

**默认模型名不要照抄**

- `default_config.py:93-94` 的默认值是 `deep_think_llm: "gpt-6-sol"` / `quick_think_llm: "gpt-6-luna"`。这是仓库当前假定的模型名，**你必须确认自己账号能访问这两个 ID**，否则第一次调用就 404。用 `TRADINGAGENTS_DEEP_THINK_LLM` / `TRADINGAGENTS_QUICK_THINK_LLM` 覆盖成你有的模型。

**实测会卡在哪**

1. **API key 缺失或厂商不支持结构化输出**：PM / Trader / Research Manager 三处都依赖 `with_structured_output`。`structured.py:50` 只在 `NotImplementedError / AttributeError` 时降级到自由文本，**其他失败（比如 key 无效、模型不存在）会在 `invoke_structured` 里被吞掉再走一次自由文本**，最终拿到一段没有评级行的话 → `run_rating` 返回 `REVIEW`。看到 `REVIEW` 先查 key 和模型名，别以为是逻辑 bug。
2. **数据厂商**：
   - `macro_data: fred` 需要 `FRED_API_KEY`，没配就是哨兵字符串（不报错，但宏观段是空的）。
   - `prediction_markets: polymarket` 免 key，但同样属于 optional 类别，拿不到就静默降级。
   - Reddit / StockTwits 是**实时 feed 不归档**（`sentiment_analyst.py` docstring 第 18-19 行明说），所以**历史回测的情绪输入不是 point-in-time**，回测结果偏乐观。
   - SEC EDGAR 的 company_tickers / facts 有本地缓存；CIK 映射表第一次拉会比较慢。
3. **Yahoo 限流**：`YFRateLimitError` 在 `vendors/yahoo/common.py` 有专门处理，但缓存目录里可能留下坏帧；代码已做"空缓存视为 miss"的兜底（第 196-205 行），真遇到可以删 `~/.tradingagents/cache` 重来。
4. **递归上限**：`max_tool_rounds` 默认 20、`max_recur_limit` 默认 100，`TradingAgentsGraph.__init__` 第 92-95 行会校验 `2 * max_tool_rounds + 2 < max_recur_limit`，你调大 tool rounds 但忘了调 recur limit 会**启动即抛 ValueError**。
5. **`trade_date` 校验**：`_validate_trade_date`（`trading_graph.py:30`）要求严格 `YYYY-MM-DD` 且不晚于今天。`main.py` 里写的是 `2026-09-01`，如果你机器上的"今天"早于这个日期会直接报错——照抄示例前先改日期。
6. **记忆日志默认开启且会触发结算**：每次 run 都会 `settle_all_pending(wait=False)`，这会**拉价格 + 调一次 LLM 生成反思**。跑 backtest 扫几十个日期时，这部分成本是叠加的，且它和你的 run 并发抢同一个 `trading_memory.md`（有文件锁，Windows 上锁不住时会跳过写入并告警，见 `dataflows/files.py:20-31`）。
7. **成本**：最小一次 run = 4 分析师（每个最多 20 轮工具调用）+ 2 研究员 × max_debate_rounds + 1 研究经理 + 1 交易员 + 3 风控 × max_risk_discuss_rounds + 1 组合经理 ≈ **12+ 次 LLM 调用，不含工具轮次**。默认 `max_debate_rounds=1`、`max_risk_discuss_rounds=1` 已经是最省配置；调成 2 就直接翻倍。用 `--analysts market` 只选一个分析师是最便宜的冒烟方式。
8. **非确定性**：`default_config.py:112-115` 的注释说得很清楚——"no setting makes LLM output bit-identical across runs"。同一天同一只票跑两次结果可能不同，别拿单次结果当信号。设 `TRADINGAGENTS_TEMPERATURE` 只能降低方差，推理模型基本忽略 temperature。
9. **Windows 控制台**：`cli/main.py` 里对 `_NO_CONSOLE_ERRORS` 有专门兜底（第 88-97 行），从管道/嵌入式终端跑交互式 CLI 会拿到 "no Windows console available" 的错误。用 Windows Terminal / PowerShell / cmd.exe，或者干脆用 `--ticker/--date/--analysts` 走非交互路径。
10. **Windows 文件锁**：`dataflows/files.py` 的 `replace_file` 在 Windows 上遇到 `PermissionError` 会**保留旧缓存并跳过写入**（只 warning）。并发跑 backtest 时会静默丢缓存，日志里有 "Kept the cached ... it is in use" 才是这个原因。

**其他**

- `tests/` 有 90 个文件，`pyproject` 默认 `addopts = "-ra --strict-markers -m \"not integration\""`，跑 `pytest` 会**跳过集成测试**；想全跑需要显式 `-m integration`，那会真调 API 花钱。
- `benchmark_map`（`default_config.py:182-204`）按 ticker 后缀自动选区域基准（`.HK → ^HSI`、`.T → ^N225` …），空后缀默认 `SPY`。如果你跑的是非美票却看到 "Alpha vs SPY"，说明 ticker 后缀没写对。
- `results_dir` / `data_cache_dir` / `memory_log_path` 默认都在 `~/.tradingagents/` 下，Docker 场景走 `TRADINGAGENTS_DATA_DIR`；换机器时这三样不会跟着仓库走。
