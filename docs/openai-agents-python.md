# OpenAI Agents SDK (Python) 技术解析（本地快照 cd26bbb）

## 一句话定位

OpenAI Agents SDK 是「多 Agent 工作流的轻量编排 SDK」，解决的是：用一个 `Agent`（模型 + 指令 + 工具 + 护栏 + handoff）作为最小抽象单元，把「LLM 循环 + 工具调用 + 代理间移交 + 历史管理 + 可观测」这套样板代码收敛掉，让开发者只写业务。

在 Agent 技术栈里它处在**编排层偏上（framework 层）**，比 LangGraph 高一层：LangGraph 给你图与状态机让你自己搭，Agents SDK 直接给你一个内置的 `while True` 循环和一套约定。代价是控制流自由度低——没有显式的「边」，下一个动作只能由模型输出或 `NextStep*` 决定。README 的自我定位是 `a lightweight yet powerful framework for building multi-agent workflows`，并强调 provider-agnostic（支持 OpenAI Responses / Chat Completions 以及 100+ 其他 LLM）。

## 事实卡片

| 项 | 值 | 来源 |
| --- | --- | --- |
| 主语言 | Python（另有 `.js` 4 文件 / 1,062 行，属文档站点资源） | 扫描统计：`.py` 1,008 文件 / 474,784 行 |
| 代码行数 | 全仓 474,784 行 Python；其中 `src/agents/` 317 文件 / 134,504 行 | `find`/`wc -l` 实测 |
| 源文件数 | 扫描命中 1,578 个（含 `.md` 474、`.json` 60、`tests/` 250 py、`examples/` 227 py） | `_scan_facts.json` |
| 主要依赖 | `openai>=3.0.0,<4`、`pydantic>=2.12.2,<3`、`griffelib>=2,<3`、`typing-extensions>=4.12.2,<5`、`requests>=2.0,<3`、`websockets>=15.0,<17`、`mcp>=1.30.0,<3`、`httpx2>=2.12.0`；可选组有 `voice`/`viz`/`litellm`/`realtime`/`redis`/`encrypt`/`dapr` 等 | 根目录 `pyproject.toml` |
| 许可证 | MIT（Copyright 2025 OpenAI） | 根目录 `LICENSE` |
| 本地快照 commit | `cd26bbb`（`git rev-parse --short HEAD`），对应 `fix: stop native tool queues during function failure cleanup (#5446)` | 实测 |
| 包版本 / Python 要求 | `openai-agents 0.24.0`，`requires-python >=3.10` | 根目录 `pyproject.toml` |
| star 数 | 【待核】（本地快照不含 GitHub 元数据） | — |

## 目录地图

顶层：`AGENTS.md`、`CHANGELOG.md`、`CLAUDE.md`、`CONTRIBUTING.md`、`LICENSE`、`Makefile`、`PLANS.md`、`README.md`、`SECURITY.md`、`docs/`、`examples/`、`integration_tests/`、`mkdocs.yml`、`pyproject.toml`、`pyrightconfig.json`、`release-please-config.json`、`src/`、`tests/`、`uv.lock`。

- `docs/` — mkdocs 站点源码，`quickstart.md`/`agents.md`/`tools.md`/`handoffs.md`/`sessions.md`/`tracing.md`/`running_agents.md` 等，另有 `zh/`、`ja/`、`ko/` 三套翻译和 `llms.txt`/`llms-full.txt`。**学概念优先看这里**，比读源码快得多。
- `examples/` — 227 个 py，21 个子目录：`basic/`（hello world、lifecycle、stream、retry）、`agent_patterns/`、`handoffs/`、`tools/`、`memory/`（15 个 session 示例）、`mcp/`、`realtime/`、`voice/`、`sandbox/`、`customer_service/`、`research_bot/`、`financial_research_agent/`、`model_providers/`。
- `tests/` — 250 个 py 文件，测试密度很高，是可以当规格书读的地方。

`src/agents/` 内部（按行数排序，行数是「重要程度」的粗略代理）：

| 文件 / 目录 | 行数 | 装什么 |
| --- | --- | --- |
| `run_state.py` | 5,682 | `RunState`：可序列化、可恢复的完整运行状态（审批、中断、续跑）。本快照里最大的单文件，新版能力集中在这里 |
| `tool.py` | 3,032 | `Tool` 全家桶：`FunctionTool`、`FileSearchTool`、`WebSearchTool`、`ComputerTool`、`HostedMCPTool`、`CodeInterpreterTool`、`ImageGenerationTool` |
| `run.py` | 2,935 | `Runner`（门面）+ `AgentRunner`（真身）+ `_run_impl`（主循环本体） |
| `run_context.py` | 1,717 | `RunContextWrapper`，贯穿全链路的上下文与用量统计 |
| `result.py` | 1,337 | `RunResult` / `RunResultStreaming` / `RunResultBase` |
| `agent.py` | 1,194 | `AgentBase` / `Agent` 数据类定义 |
| `items.py` | 1,034 | `RunItem` 家族（MessageOutputItem / ToolCallItem / HandoffCallItem / ToolApprovalItem …）+ `ModelResponse` + `ItemHelpers` |
| `run_internal/` | 26 文件 / 20,789 行 | **执行引擎的真正所在**。主循环被拆成 `run_loop.py`(3,081)、`turn_resolution.py`(4,101)、`tool_execution.py`(2,875)、`session_persistence.py`(2,065)、`tool_planning.py`(1,233)、`items.py`(1,139)、`tool_actions.py`(1,122)、`model_retry.py`(894) 等子模块 |
| `models/` | 21 文件 | `Model` 抽象（`interface.py`）+ OpenAI Responses（2,371 行）/ Chat Completions + `MultiProvider` + `chatcmpl_*` 转换器 + 重试与 WebSocket |
| `memory/` | 7 文件 | `Session` 协议与 `SessionABC`、`SQLiteSession`、`openai_conversations_session`、`openai_responses_compaction_session`（自动压缩） |
| `tracing/` | 15 文件 | `trace()` / `Span` / `Trace` / `TracingProcessor` / `BackendSpanExporter` / span_data（17 种 span 类型） |
| `handoffs/` | 2 文件 | `Handoff`、`handoff()`、`history.py`（移交时的历史裁剪策略） |
| `guardrail.py` | 343 | `@input_guardrail` / `@output_guardrail` 装饰器与 tripwire 结果类型 |
| `mcp/` | — | MCP server 客户端（stdio / SSE / streamable HTTP） |
| `realtime/` `voice/` | — | 语音：实时 WebSocket Agent 与 STT-Agent-TTS 流水线 |
| `sandbox/` | — | 容器工作区（`UnixLocalSandboxClient` / `DockerSandboxClient` / 托管） |
| `extensions/` | — | 实验特性：`handoff_filters`、`handoff_prompt`、`visualization`、`memory/`、`models/`（litellm 等） |
| `_config.py` `_tool_identity.py` `strict_schema.py` `function_schema.py` `_tool_invocation.py` | — | 下划线前缀的内部基础设施：配置、工具身份去重、strict JSON schema 生成、函数签名到 schema 的转换 |

## 核心架构

```
调用方
  Runner.run / run_sync / run_streamed     src/agents/run.py:281 / :386 / :489
        |
        v
+---------------------------------------------------------------------------+
| 门面层                                                                     |
|   Runner (class)          run.py:279   只是一层包装，转发给 DEFAULT_AGENT_RUNNER|
|   AgentRunner             run.py:568                                       |
|     run()      :574                                                        |
|     _run_impl():629   <-- 主循环在这里（约 1880 行，直到 :2511）             |
|     run_sync() :2511 / run_streamed() :2632                                |
+---------------------------------------------------------------------------+
        |
        v
+---------------------------------------------------------------------------+
| 编排层：单轮执行 + 下一步判定                                                |
|                                                                           |
|   while True:                          run.py:1032                        |
|     run_single_turn(...)               run.py:1909 调用                    |
|                    定义                 run_internal/run_loop.py:2789      |
|       +-- get_all_tools()              run_internal/turn_preparation.py:119|
|       +-- get_handoffs()               run_internal/turn_preparation.py:96 |
|       +-- get_new_response()           run_internal/run_loop.py:2928       |
|       +-- process_model_response()     run_internal/turn_resolution.py:3158|
|       +-- get_single_step_result_..()  run_internal/turn_resolution.py:4016|
|       +-- execute_tools_and_side_effects()  turn_resolution.py:966        |
|                                                                           |
|   返回 SingleStepResult  run_internal/run_steps.py:193                     |
|     -> ProcessedResponse      run_steps.py:119                             |
|     -> NextStepFinalOutput    run_steps.py:169   退出循环                  |
|     -> NextStepHandoff        run_steps.py:164   换 agent，continue        |
|     -> NextStepRunAgain       run_steps.py:174   再来一轮                  |
|     -> NextStepInterruption   run_steps.py:179   等人审批                  |
+---------------------------------------------------------------------------+
        |                                        |
        v                                        v
+-------------------------------+   +-------------------------------------+
| 执行单元                       |   | LLM 客户端                           |
|   Agent      agent.py:323     |   |   Model (ABC) models/interface.py:37 |
|     instructions / prompt     |   |     get_response()          :80      |
|     tools / handoffs          |   |     stream_response()       :115     |
|     guardrails / output_type  |   |   OpenAIResponsesModel               |
|     tool_use_behavior         |   |     models/openai_responses.py:513   |
|   Tool       tool.py:455 ...  |   |   OpenAIChatCompletionsModel         |
|   Handoff    handoffs/__init__|   |     models/openai_chatcompletions.py |
|              .py:127          |   |            :55                       |
|   Guardrail  guardrail.py:72  |   |   MultiProvider                      |
|              / :134           |   |     models/multi_provider.py:62      |
|   MCP        mcp/             |   |   get_default_model()                |
|                               |   |     models/default_models.py:99      |
|                               |   |     默认 "gpt-5.6-luna"              |
+-------------------------------+   +-------------------------------------+
        |                                        |
        v                                        v
+-------------------------------+   +-------------------------------------+
| 状态存储                       |   | 观测                                 |
|   Session (Protocol)          |   |   trace()   tracing/create.py:31    |
|     memory/session.py:70      |   |   Trace     tracing/traces.py:51    |
|   SessionABC            :113  |   |   Span      tracing/spans.py:59     |
|   SQLiteSession                |   |   17 种 SpanData                    |
|     memory/sqlite_session.py:18|   |     tracing/span_data.py:11        |
|   RunState  run_state.py:842  |   |   BackendSpanExporter                |
|     to_json()          :1909  |   |     tracing/processors.py:45        |
|     to_string()        :2281  |   |   ConsoleSpanExporter          :28  |
|     approve()/reject() :1380  |   |                                     |
|                        :1398  |   |                                     |
|   save_result_to_session()     |   |                                     |
|     run_internal/session_      |   |                                     |
|     persistence.py:778         |   |                                     |
+-------------------------------+   +-------------------------------------+
```

## 关键抽象与执行流

### 1. Runner 循环

`src/agents/run.py:279` 的 `Runner` **只是个门面**：`Runner.run()`（:281）把参数原样转发给模块级常量 `DEFAULT_AGENT_RUNNER`（:349），真正的实现在 `AgentRunner`（:568）。

`Runner.run()` 的 docstring（:296-347）是官方对循环语义的权威描述，原文四步：

1. The agent is invoked with the given input.
2. If there is a final output (i.e. the agent produces something of type `agent.output_type`), the loop terminates.
3. If there's a handoff, we run the loop again, with the new agent.
4. Else, we run tool calls (if any), and re-run the loop.

两个异常出口：`MaxTurnsExceeded`（`exceptions.py:354`）、guardrail tripwire（`InputGuardrailTripwireTriggered` :429 / `OutputGuardrailTripwireTriggered` :442）。注意 `run.py:314` 的一句注释：**只有第一个 agent 的 input guardrails 会跑**。

主循环实体是 `AgentRunner._run_impl()`（`run.py:629`），`while True:` 在 `run.py:1032`。每轮做的事：

- 输入护栏（仅第 0 轮，`run.py:1047-1055`），串行/并行两组
- `bind_public_agent(current_agent)`（:1082）得到 `AgentBindings`（public agent 与 execution agent 分离，见 `run_internal/agent_bindings.py`）
- `await run_single_turn(...)`（:1909）
- 按 `turn_result.next_step` 的类型分派（:2103 FinalOutput / :1543 Handoff / :1331 RunAgain / :1284 Interruption）
- `current_turn += 1`，超过 `max_turns` 抛 `MaxTurnsExceeded`（:1597-1606）

Handoff 分支（:1543-1555）尤其值得看：它把 `current_agent` 换成 `next_step.new_agent`、重置 `starting_input`、关掉当前 span、置 `should_run_agent_start_hooks = True`，然后 `continue`——**代理间的切换只是一次变量重绑定，没有任何图结构**。这是 Agents SDK 与 LangGraph 最本质的设计差异。

`max_turns` 默认值来自 `DEFAULT_MAX_TURNS`（`run.py:46` 引入），传 `None` 可关闭上限。

### 2. 单轮：run_single_turn

定义在 `src/agents/run_internal/run_loop.py:2789`，签名长达 20 个关键字参数（:2790-2811）。执行顺序：

1. `:2821` `guard_agent_tool_configuration(public_agent)` 上下文里跑 `on_agent_start` hooks（agent 级 + run 级，用 `gather_with_cancel` 并发）
2. `:2838` `get_all_tools()` —— 见 `turn_preparation.py:119`
3. `:2839` `initialize_computer_tools()` —— ComputerTool 需要惰性初始化
4. `:2841-2844` 并发取 `get_system_prompt()`（`agent.py:1153`）和 `get_prompt()`（`agent.py:1184`）
5. `:2846` `get_handoffs()` —— `turn_preparation.py:96`
6. `:2847` `resolve_tool_name_collisions()` —— 工具名与 handoff 名冲突时按 `tool_name_collision_policy` 处理
7. `:2860` `get_output_schema()` 决定本轮是否要结构化输出
8. `:2866` `get_new_response(...)` —— 打模型，定义在 `run_loop.py:2928`
9. 之后交给 `process_model_response()`（`turn_resolution.py:3158`）-> `get_single_step_result_from_response()`（`turn_resolution.py:4016`）产出 `SingleStepResult`

流式版本是 `start_streaming()`（`run_loop.py:987`）与 `run_single_turn_streamed()`（:2423），`Runner.run_streamed()`（`run.py:489`）走这条路径，返回 `RunResultStreaming`（`result.py:660`），用 `async for event in result.stream_events()`（`result.py:1005`）消费。

### 3. Agent / Tool / Handoff

**Agent**（`agent.py:323`）是一个 dataclass，字段即能力清单：

- `instructions`（:336）—— system prompt，可以是字符串或 `(context, agent) -> str` 的函数
- `prompt`（:351）—— 从 OpenAI 平台侧动态拉取的 prompt，仅 Responses API 可用
- `handoffs`（:357）—— 子 agent 列表
- `model`（:365）—— 字符串或 `Model` 实例；不设就用 `get_default_model()`
- `model_settings`（:372）—— `ModelSettings` 或 dict
- `input_guardrails` / `output_guardrails`（:378 / :384）
- `output_type`（:390）—— 结构化输出类型，不设就是 `str`
- `hooks`（:402）、`tool_use_behavior`（:407）、`reset_tool_choice`（:429）

`AgentBase`（`agent.py:191`）是与 `RealtimeAgent` 共享的基类，`get_all_tools()`（:290）/ `get_mcp_tools()`（:266）在这里。`as_tool()`（:630）把 agent 当工具用（"Agents as tools" 模式）。

**Tool**：`tool.py:455` 的 `FunctionTool` 是最核心的一个，字段除了 `name`/`description`/`params_json_schema`/`on_invoke_tool` 外，还有几个体现工程成熟度的：

- `strict_json_schema`（:482）—— 默认 True，注释明确「we **strongly** recommend setting this to True」
- `is_enabled`（:486）—— 可以是 `(run_context, agent) -> bool`，动态开关工具
- `needs_approval`（:500）—— HITL 的挂钩点，可以是 `(run_context, params, call_id) -> bool`。docstring 里详细说明了一堆「必须人工审批」的边界情况（默认值、变换参数、无法内省的参数等）
- `tool_input_guardrails` / `tool_output_guardrails`（:494 / :497）—— 工具级护栏
- `timeout_seconds` / `timeout_behavior` / `timeout_error_function`（:515-526）

其余工具类型：`FileSearchTool`(:815)、`WebSearchTool`(:853)、`ComputerTool`(:908)、`HostedMCPTool`(:1158)、`CodeInterpreterTool`(:1189)、`ImageGenerationTool`(:1254)。工具调用的身份去重逻辑在 `_tool_identity.py`（683 行），这批文件是本快照新增的。

**Handoff**：`handoffs/__init__.py:127` 的 `Handoff` 是 `Generic[TContext, TAgent]`，三个关键方法 `get_transfer_message()`(:224)、`default_tool_name()`(:228)、`default_tool_description()`(:235) 说明它在模型侧就是一个普通 function tool。`handoff()` 工厂函数有 4 个重载（:243/:256/:271/:284）。`handoffs/history.py`（675 行）处理移交时历史怎么裁剪——这是多 Agent 系统里最容易被忽略、上线后最容易出问题的地方。

### 4. Session 与 RunState

这是两个不同层次的东西，很容易混淆：

**Session** 是「跨 run 的对话历史」，抽象在 `memory/session.py:70` 的 `Session` **Protocol**（不是 ABC），只有四个方法：`get_items()`(:80)、`add_items()`(:92)、`pop_item()`(:100)、`clear_session()`(:108)。`SessionABC`（:113）给出可选基类。`SQLiteSession`（`memory/sqlite_session.py:18`）是最常用的实现。`memory/session.py:209` 的 `_session_method_accepts_wrapper()` 说明它在兼容「老式不接受 wrapper 的第三方 session」——Protocol + 运行时内省，是它保持可插拔的代价。

写入路径在 `run_internal/session_persistence.py`：`prepare_input_with_session()`(:424) 读历史、`save_result_to_session()`(:778) 写回。`_run_impl` 里 `session_persistence_enabled` 的判断见 `run.py:739`。

**RunState** 是「单次 run 的可序列化快照」，定义在 `run_state.py:842`。它存在的意义是 HITL 与恢复：`get_interruptions()`(:1110)、`approve()`(:1380)、`reject()`(:1398)、`to_json()`(:1909)、`to_string()`(:2281)、`from_string()`(:2339)、`from_json()`(:2433)。`RunResult.to_state()`（`result.py:606`）和 `RunResultStreaming.to_state()`（`result.py:1272`）是入口。

一句话区分：**Session 管「跨轮次的对话内容」，RunState 管「这一轮跑到哪、卡在哪、能不能续」。**

### 5. Tracing

`tracing/` 是一套自研的 span/trace 模型，不依赖 OpenTelemetry，但形状很像：

- `trace()` 上下文管理器（`tracing/create.py:31`），`get_current_trace()`(:79) / `get_current_span()`(:84)
- 17 种 span 构造函数：`agent_span`(:89)、`task_span`(:124)、`turn_span`(:139)、`function_span`(:155)、`generation_span`(:188)、`response_span`(:236)、`handoff_span`(:262)、`custom_span`(:293)、`guardrail_span`(:325)、`transcription_span`(:353)、`speech_span`(:397)、`mcp_tools_span`(:466) 等
- 对应的数据类全在 `tracing/span_data.py`（`AgentSpanData`:28、`GenerationSpanData`:169、`HandoffSpanData`:245 …）
- `Span` ABC（`tracing/spans.py:59`），`SpanImpl`(:289) 是真身，`NoOpSpan`(:216) 是关闭 tracing 时的空实现——**这个空对象模式保证了调用点不用写 if**
- 导出端 `ConsoleSpanExporter`（`processors.py:28`）与 `BackendSpanExporter`(:45)；后者有完整的重试、超时、deadline、payload 截断（`_truncate_string_for_json_limit` :452）和脱敏（`_sanitize_for_openai_tracing_api` :337）逻辑

`Trace` ABC 在 `tracing/traces.py:51`，`TraceImpl`(:486)、`NoOpTrace`(:407)、`ReattachedTrace`(:305)。`attach_trace` 相关能力在 `run_state.py:2305` 的 `set_trace()`。

## 最小可运行入口

安装（出自 `README.md` 的 "Get started" 段与 `docs/quickstart.md`）：

```bash
# venv
python -m venv .venv
source .venv/bin/activate          # Windows: .venv\Scripts\activate
pip install openai-agents

# 或 uv
uv init
uv add openai-agents

# 可选组
pip install 'openai-agents[voice]'    # 语音
pip install 'openai-agents[redis]'    # Redis session
```

环境变量：`docs/quickstart.md` 的 "Set an OpenAI API key" 段要求设置 `OPENAI_API_KEY`（**所有示例运行前都要设**）。另有一个非必填的 `OPENAI_DEFAULT_MODEL`（见 `models/default_models.py:99`，默认模型为 `gpt-5.6-luna`）。

最小 demo A —— 文本 Agent（出自 `examples/basic/hello_world.py`，原样摘录）：

```python
import asyncio

from agents import Agent, Runner


async def main():
    agent = Agent(
        name="Assistant",
        instructions="You only respond in haikus.",
    )

    result = await Runner.run(agent, "Tell me about recursion in programming.")
    print(result.final_output)
    # Function calls itself,
    # Looping in smaller pieces,
    # Endless by design.


if __name__ == "__main__":
    asyncio.run(main())
```

同步版（出自 `README.md` "Run a text agent" 段）：

```python
from agents import Agent, Runner

agent = Agent(name="Assistant", instructions="You are a helpful assistant")
result = Runner.run_sync(agent, "Write a haiku about recursion in programming.")
print(result.final_output)
```

最小 demo B —— 带持久会话（出自 `examples/memory/sqlite_session_example.py`，节选）：

```python
import asyncio

from agents import Agent, Runner, SQLiteSession


async def main():
    agent = Agent(name="Assistant", instructions="Reply very concisely.")
    session = SQLiteSession("conversation_123")   # 跨 run 复用同一个 session_id

    result = await Runner.run(
        agent,
        "What city is the Golden Gate Bridge in?",
        session=session,
    )
    print(f"Assistant: {result.final_output}")

    # 第二轮不需要手动传历史，Session 自动带上
    result = await Runner.run(agent, "What state is it in?", session=session)
    print(f"Assistant: {result.final_output}")


if __name__ == "__main__":
    asyncio.run(main())
```

## 值得抄的工程手法

1. **「下一步做什么」用 tagged union 表达，而不是布尔标志**。`run_steps.py` 定义 `NextStepFinalOutput`(:169) / `NextStepHandoff`(:164) / `NextStepRunAgain`(:174) / `NextStepInterruption`(:179) 四个轻量类型，主循环用 `isinstance` 分派（`run.py:2103`/`:1543`/`:1331`/`:1284`）。好处是新增一种控制流转态时，所有分支点都会被类型检查器逼着补齐。这是你在任何状态机代码里都能立刻用上的手法。

2. **门面与实现分离，且门面负责错误脱敏**。`Runner`（`run.py:279`）不含逻辑，只做转发；`except BaseException` 后调用 `_is_error_data_redacted()` / `_detach_data_redacted_error_traceback()` / `_raise_data_redacted_error()`（:365-383），并在返回前把 `starting_agent`、`input`、`context`、`hooks` 全部置 None——**主动切断引用，避免在 traceback 里长期持有用户对象造成泄漏**。对一个会捕获用户任意对象的框架，这个细节很关键。

3. **工具契约把「非功能性需求」直接做进数据结构**。`FunctionTool` 把 `needs_approval`（HITL）、`is_enabled`（动态开关）、`timeout_seconds`/`timeout_behavior`/`timeout_error_function`（超时）、`tool_input_guardrails`/`tool_output_guardrails`（工具级护栏）、`strict_json_schema`（schema 严格性）全部做成一等字段（`tool.py:482-526`），并且给 `needs_approval` 写了长 docstring 穷举「哪些情况必须人工审批」。好处是工具作者不需要自己实现这些横切关注点，框架统一兜底。

4. **Tracing 用 NoOp 对象而不是 if 判断**。`NoOpSpan`（`tracing/spans.py:216`）和 `NoOpTrace`（`tracing/traces.py:407`）实现与真身完全相同的接口，tracing 关闭时替换实现即可。好处是所有埋点调用点零分支，可读性接近无埋点代码，也不怕漏判。

5. **对外契约用 Protocol 而非 ABC**。`Session` 是 `Protocol`（`memory/session.py:70`），同时提供可选的 `SessionABC`(:113)。好处是第三方可以不继承你的基类就接入；代价是 `_session_method_accepts_wrapper()`(:209) 这类运行时内省。这是「结构化子类型 vs 名义子类型」的一个活标本，值得对照你自己的插件接口设计。

6. **把巨型模块拆成 `run_internal/` 并保持前缀命名**。`run_internal/` 有 26 个文件、20,789 行，按职责切：`run_loop`（单轮）、`turn_resolution`（响应解析与下一步）、`tool_execution`（工具并发与失败优先级）、`session_persistence`（历史读写）、`tool_planning`（工具调度）、`guardrails`、`error_handlers`、`approvals`。好处是主文件的行数下降、职责可定位。缺点是 `_` 前缀目录意味着**这些不是公开 API，升级随时会动**。

## 阅读路径

按这个顺序读，从用户 API 一路下沉到引擎：

1. `docs/quickstart.md` + `docs/agents.md` + `docs/running_agents.md` —— **为什么现在读**：先拿到官方的概念词汇表（Agent / Handoff / Guardrail / Session / Tracing），否则直接看源码会被 300 个名字淹没。
2. `src/agents/run.py` 的 `Runner.run()` docstring（:296-347）—— **为什么现在读**：官方对循环四步语义的权威表述，读完就知道整个 SDK 的心智模型，成本 50 行。
3. `examples/basic/hello_world.py` + `examples/memory/sqlite_session_example.py` —— **为什么现在读**：先跑通两个 demo，建立「我写的代码 -> 输出」的直接因果，后面读引擎才有对照物。
4. `src/agents/agent.py`（`AgentBase`:191 与 `Agent`:323）—— **为什么现在读**：Agent 是唯一的顶层抽象，它的字段清单就是整个 SDK 的能力清单，逐个字段看一遍等于过一遍功能列表。
5. `src/agents/tool.py` 的 `FunctionTool`（:455-560）—— **为什么现在读**：工具是 Agent 与外界交互的唯一出口，`needs_approval` / `is_enabled` / timeout 三处设计决定了你以后怎么做 HITL 和稳定性。
6. `src/agents/run_internal/run_steps.py` —— **为什么现在读**：`ProcessedResponse`(:119) 与四个 `NextStep*`(:164-179) 是循环的「返回值协议」，看懂它才能看懂主循环的分派。
7. `src/agents/run.py` 的 `AgentRunner._run_impl()`（:629 起，重点看 :1032 的 `while True`、:1543 的 handoff 分支、:1909 的 `run_single_turn` 调用、:2103 的终态判定）—— **为什么现在读**：这是主循环的本体，其他所有模块都是它调用出去的。建议只读骨架，跳过 try/except 细节。
8. `src/agents/run_internal/run_loop.py` 的 `run_single_turn()`（:2789-2890）—— **为什么现在读**：单轮的完整时序（hooks -> 取工具 -> 取 handoffs -> 冲突处理 -> 打模型）在这里一目了然。
9. `src/agents/models/interface.py`（:37-150）—— **为什么现在读**：`Model` ABC 两个抽象方法 `get_response`/`stream_response` 就是「provider-agnostic」的全部实现成本，看完就知道接一个新 LLM 要写什么。
10. `src/agents/run_state.py` 的 `RunState`（:842，重点看 `approve` :1380 / `reject` :1398 / `to_json` :1909）—— **为什么现在读**：HITL 与断点续跑的落点，也是本快照最大的新增面，做生产级 Agent 绕不开。
11. `src/agents/tracing/create.py` + `tracing/spans.py` —— **为什么现在读**：选读。理解 17 种 span 的分类维度，对设计你自己的 Agent 可观测方案有直接参考价值。

## 别踩的坑

- **`_run_impl` 是一个近 1900 行的方法**（`run.py:629` 到 `run.py:2511`）。这是本仓库最大的历史包袱，读它时**只跟骨架**（`while True`、四个 `NextStep*` 分派、`current_turn` 递增），不要试图理解每个 try/except。它已经处于被拆到 `run_internal/` 的中途状态，未来版本很可能继续变。
- **`run_internal/`、`_config.py`、`_tool_identity.py` 等下划线模块不是公开 API**。目录名已经写明 internal，但网上不少教程会直接 import 里面的东西。升级到下一个 minor 就可能炸。只用 `agents` 顶层导出的名字（`src/agents/__init__.py` 的 `__all__` 有完整清单）。
- **版本号是 0.24.0，仍在 0.x**。虽然 README 说 provider-agnostic、支持 100+ LLM，但默认模型 `gpt-5.6-luna`、默认走 Responses API，非 OpenAI 模型的能力覆盖（尤其是 hosted tool、server-side conversation）是不对等的。`run.py:338-340` 自己承认：`conversation_id` 建议只在纯 OpenAI 模型下用，否则历史会不完整。
- **`mcp` 依赖被显式排除了 2.0.* 和 2.1.***（`pyproject.toml` 注释指向 GHSA-rwrf-2pqf-9j8j）。这是安全下限，别为了装新版本手动放宽，也别以为它是笔误。
- **行号会漂移，用类名/函数名定位**。本文档行号基于 commit `cd26bbb`。`run_state.py`（5,682 行）、`tool.py`（3,032 行）、`run.py`（2,935 行）都在快速膨胀中，尤其是 `run_internal/` 是本快照新拆出来的结构。**搜索时优先搜 `async def run_single_turn`、`class RunState`、`class FunctionTool` 这类锚点。**
- **不要从 `realtime/`、`voice/`、`sandbox/` 入门**。这三个模块各自有独立的会话模型和传输层，与核心 `Runner` 循环差异很大，且需要额外的音频/容器依赖。先把 `run.py` + `agent.py` + `tool.py` 吃透。
- **`docs/` 里有 `zh/` 翻译目录，但不一定跟得上英文版**。发现中文文档与源码行为冲突时，**以源码为准**。
- **`tests/` 有 250 个 py 文件**，是被低估的资源。当你怀疑某个行为（尤其是 handoff 历史裁剪、session 写入时机、guardrail 触发顺序）时，搜对应测试文件往往比读实现快。
