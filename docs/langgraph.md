# LangGraph 技术解析（本地快照 6f2df05）

## 一句话定位

LangGraph 是「有状态 Agent 的底层编排框架」，解决的是 ReAct 这类简单循环没法解决的三个工程问题：**长时间运行中途崩了要从断点继续**、**跑到一半要人插手改状态**、**多步推理过程要可回放可追溯**。

在 Agent 技术栈里它处在**编排层（orchestration）**，位置在 LLM 客户端（langchain-core / 各家 SDK）之上、业务 Agent 应用之下。它自己不提供任何模型调用能力，只提供「图 + 状态 + 持久化 + 流」的运行时。`libs/langgraph/README.md` 的定位原话：

> LangGraph is a low-level orchestration framework for building, managing, and deploying long-running, stateful agents.

README 末尾也明确了它的血统：灵感来自 Google 的 Pregel 论文与 Apache Beam，公开接口借鉴 NetworkX。

## 事实卡片

| 项 | 值 | 来源 |
| --- | --- | --- |
| 主语言 | Python（另有少量 TS/JS 在 `libs/sdk-js`） | 扫描统计：`.py` 479 文件 / 204,122 行；`.ts` 7 文件 257 行，`.js` 1 文件 18 行 |
| 代码行数 | 全仓 204,122 行 Python；`libs/langgraph` 本体 30,376 行 | `find`/`wc -l` 实测 |
| 源文件数 | 扫描命中 642 个文件（其中 `.ipynb` 35、`.py` 479） | `_scan_facts.json` |
| 主要依赖 | `langchain-core>=1.4.7,<2`、`langgraph-checkpoint>=4.3.0,<5.0.0`、`langgraph-sdk>=0.4.6,<0.5.0`、`langgraph-prebuilt>=1.1.0,<1.2.0`、`xxhash>=3.5.0`、`pydantic>=2.7.4` | `libs/langgraph/pyproject.toml` |
| 许可证 | MIT（Copyright 2024 LangChain, Inc.） | 根目录 `LICENSE` |
| 本地快照 commit | `6f2df05`（`git rev-parse --short HEAD`），对应 `test(sdk-py): check that a reconnect loop resumes after the newest event (#9483)` | 实测 |
| 包版本 / Python 要求 | `langgraph 1.2.18`，`requires-python >=3.10` | `libs/langgraph/pyproject.toml` |
| star 数 | 【待核】（本地快照不含 GitHub 元数据） | — |

## 目录地图

顶层：`AGENTS.md`、`LICENSE`、`Makefile`、`README.md`、`docs/`、`examples/`、`libs/`。

- `docs/` — 只有 `generate_redirects.py`、`llms.txt`、`redirects.json` 三个文件，是给 LLM 抓取用的索引，不是教程。真正的文档在站外（docs.langchain.com）。
- `examples/` — 39 个文件，其中 35 个 `.ipynb`、2 个 `.py`，20 个子目录（`rag/`、`multi_agent/`、`plan-and-execute/`、`reflection/`、`rewoo/`、`human_in_the_loop/` 等）。它们是**用法参考**，不是架构参考，且部分基于旧 API。

`libs/` 是 monorepo，8 个子包职责完全分离，这是理解 LangGraph 最关键的一层：

| 子包 | 规模 | 装什么 |
| --- | --- | --- |
| `libs/langgraph` | 88 py / 30,376 行 | **核心运行时**。`pregel/` 是执行引擎，`graph/` 是 StateGraph 构建器，`channels/` 是状态通道，`func/` 是 Functional API，`types.py` 定义 `Send`/`Command`/`interrupt` |
| `libs/checkpoint` | 17 py / 6,049 行 | **抽象层**。`checkpoint/base/` 定义 `Checkpoint`/`BaseCheckpointSaver`；`checkpoint/memory/` 是 `InMemorySaver` 参考实现；`checkpoint/serde/` 是 jsonplus / msgpack / 加密序列化；另有 `store/`（长期记忆）和 `cache/`（节点级缓存） |
| `libs/checkpoint-sqlite` | 8 py / 4,017 行 | SQLite 版 saver/store 与 cache，含 `_delta.py`（增量通道落盘逻辑） |
| `libs/checkpoint-postgres` | 9 py / 5,436 行 | Postgres 版 saver（含 `shallow.py` 浅层检查点）+ 异步版 |
| `libs/prebuilt` | 7 py / 3,710 行 | 高层组件：`create_react_agent`、`ToolNode`、`interrupt.py` 辅助。**在本快照中 `create_react_agent` 已被标记废弃** |
| `libs/sdk-py` | 66 py / 21,447 行 | `langgraph_sdk`，访问托管 LangGraph 服务器的 Python 客户端（HTTP/SSE/stream），与核心执行模型无关 |
| `libs/cli` | 47 py / 11,296 行 | `langgraph_cli`，`new`/`dev`/`up`/`build`/`dockerfile` 命令，本地开发服务器与镜像构建 |
| `libs/checkpoint-conformance` | 17 py / 3,451 行 | 给第三方 checkpointer 实现跑的一致性测试套件，说明「checkpointer 是一个可插拔契约」 |
| `libs/sdk-js` | 0 py（ts/mts） | JS/TS SDK，Python 侧不用看 |

## 核心架构

LangGraph 的执行模型是 **BSP（Bulk Synchronous Parallel，批量同步并行）**，即 Pregel 论文的那一套。一个「超步（superstep）」分三段：Plan（选出本步要跑的 actor）-> Execution（并行跑，写入彼此不可见）-> Update（统一提交写入到通道）。

```
调用方
  app.invoke / stream / astream / get_state / update_state
      pregel/main.py:3873 invoke | :2709 stream | :3115 astream | :1464 get_state | :2617 update_state
        |
        v
+---------------------------------------------------------------------------+
| 编排层：Pregel（BSP 超步调度器）                                            |
|                                                                           |
|   Pregel                             pregel/main.py:460                   |
|     +-- PregelLoop                   pregel/_loop.py:170                  |
|     |     tick()        :650   计划本步任务 + 落 checkpoint                 |
|     |     after_tick()  :700   应用写入 + 推进 step                        |
|     |     _first()      :921   首步：装载 checkpoint / 输入写入             |
|     |     _put_checkpoint() :1237  异步持久化                              |
|     +-- apply_writes()              pregel/_algo.py:233                   |
|     +-- prepare_next_tasks()        pregel/_algo.py:391                   |
|     +-- prepare_single_task()       pregel/_algo.py:523                   |
|     +-- PregelRunner                pregel/_runner.py:132                 |
|           tick() :173 / atick() :359 / commit() :575                      |
+---------------------------------------------------------------------------+
        |
        v
+---------------------------------------------------------------------------+
| 执行单元：PregelNode（actor）        pregel/_read.py:97                    |
|   读通道 ChannelRead                pregel/_read.py:25                    |
|   写通道 ChannelWrite               pregel/_write.py                      |
|                                                                           |
|   三种构造来源（都编译成同一个 IR）：                                        |
|     StateGraph -> compile()         graph/state.py:131 / :1177            |
|     @entrypoint / @task             func/__init__.py:262 / :110           |
|     NodeBuilder（直接用 Pregel）     pregel/main.py:216                    |
+---------------------------------------------------------------------------+
        |                                        |
        v                                        v
+-----------------------------+   +----------------------------------------+
| 状态存储（本轮）             |   | 状态存储（跨轮持久化）                  |
|  channels/                  |   |  checkpoint/                           |
|   BaseChannel   base.py:19  |   |   Checkpoint        base/__init__.py:93|
|   LastValue     last_value  |   |   BaseCheckpointSaver           :199   |
|   BinaryOperatorAggregate   |   |     +-- InMemorySaver                  |
|              binop.py:66    |   |         checkpoint/memory/__init__.py:34|
|   Topic / EphemeralValue /  |   |     +-- SqliteSaver   (checkpoint-     |
|   NamedBarrierValue / Delta |   |          sqlite/langgraph/checkpoint/  |
|                             |   |          sqlite/__init__.py)           |
|  store/base/   长期记忆      |   |     +-- PostgresSaver (checkpoint-     |
|  cache/base/   节点缓存      |   |          postgres/.../postgres/)      |
+-----------------------------+   +----------------------------------------+
        |
        v
+---------------------------------------------------------------------------+
| LLM 客户端：不在本仓库内                                                    |
|  langchain-core 的 BaseChatModel / BaseTool（依赖 langchain-core>=1.4.7）  |
|  通过 libs/prebuilt 的 ToolNode、create_react_agent 接进图里                |
+---------------------------------------------------------------------------+
```

## 关键抽象与执行流

### 1. Pregel —— 执行模型的全部秘密

`libs/langgraph/langgraph/pregel/main.py:460` 的 `Pregel` 类 docstring（460-713 行）是**整个项目最值得先读的 250 行文本**，它把 BSP 三段式讲得比任何外部文档都清楚。核心字段在 `main.py:715-766`：

- `nodes: dict[str, PregelNode]` —— actor 表
- `channels: dict[str, BaseChannel | ManagedValueSpec]` —— 状态通道表
- `input_channels` / `output_channels` / `stream_channels`
- `checkpointer`、`store`、`cache`
- `interrupt_before_nodes` / `interrupt_after_nodes`（`All | Sequence[str]`，`All` 表示每个节点都断）
- `step_timeout`、`retry_policy`、`cache_policy`

超步循环的实际推进在 `pregel/_loop.py` 的 `PregelLoop`（:170）。它不是一个函数，而是一个**显式状态机**：`status` 字段有 `input / pending / done / draining / interrupt_before / interrupt_after / out_of_steps` 七种取值（`_loop.py:295-303`）。同步版 `SyncPregelLoop`（:1652）与异步版 `AsyncPregelLoop`（:1908）各自实现 `__enter__`/`__aenter__` 装载 checkpoint，`_checkpointer_put_after_previous`（:1712 / :1968）把落盘串行化。

`_loop.py:197-237` 的注释值得精读：它解释了为什么 checkpoint 的保存必须**等**产生它的 delta 写入 future 先完成——否则重启后从一个残缺的 checkpoint 恢复会永久读不到数据。这是「durable execution」最容易写错的地方，LangGraph 在这里给出了标准答案。

### 2. apply_writes —— 并行写入如何做到确定性

`pregel/_algo.py:233` 的 `apply_writes()` 是 BSP 的 Update 阶段。逻辑顺序：

1. `:255` 先按 `task.path` 排序（`writes_sort_key`），**保证同一超步内多个并行任务的写入顺序确定**，这是可重放的前提。
2. `:262-268` 更新 `versions_seen`，记录每个任务读到了哪个通道版本。
3. `:284-291` 对「被读过的通道」调用 `channels[chan].consume()`，并 bump `channel_versions`。
4. `:294-312` 按通道聚合所有写入，写到未知通道只记 warning 不报错。
5. `:316-322` 调用 `channels[chan].update(vals)`，只有 `is_available()` 的通道才算 `updated_channels`（才能触发下游）。
6. `:325-341` 若本步没有任何通道能触发任务，通知所有通道 `finish()`。

返回 `updated_channels`，交给 `prepare_next_tasks`（`_algo.py:391`）算出下一步该跑谁。**「谁被触发」不是靠边（edge）直接决定的，而是靠「通道版本变没变」决定的**——这是 LangGraph 和一般 DAG 引擎最本质的区别，也是它能做循环、Send、子图重入的原因。

### 3. StateGraph / 节点 / 边 —— 编译到 Pregel 的那一层

`graph/state.py:131` 的 `StateGraph` 是**纯构建器**，docstring 明确警告「不能直接执行，必须先 `.compile()`」。它内部维护的是：

- `edges: set[tuple[str, str]]`（:202）
- `nodes: dict[str, StateNodeSpec]`（:203）
- `branches: defaultdict[str, dict[str, BranchSpec]]`（:204）
- `channels: dict[str, BaseChannel]`（:205）
- `schemas: dict[type, dict[str, BaseChannel | ManagedValueSpec]]`（:207）

关键方法：
- `add_node()` —— **有 5 个重载**（:376 / :446 / :520 / :590 / :667），因为节点签名可以是 `state -> ...`、`(state, config)`、`(state, *, writer)`、`(state, *, store)`、`(state, runtime)` 各种组合。这些签名协议定义在 `graph/_node.py:22-90` 的一串 `Protocol` 里。
- `add_edge(start_key, end_key)` :928
- `add_conditional_edges(...)` :982 -> 内部生成 `BranchSpec`（`graph/_branch.py:83`），`BranchSpec.run()`/`_route()`（:122 / :146）在运行时决定走哪条分支
- `compile(checkpointer=..., interrupt_before=..., ...)` :1177 -> 返回 `CompiledStateGraph`（:1404），而 `CompiledStateGraph` 就是 `Pregel` 的子类

`compile()` 里有两个容易被忽略但很重要的动作：
- `:1234-1254` 在 `STRICT_MSGPACK_ENABLED` 时，遍历 `state_schema`/`input_schema`/`output_schema`/所有节点与分支的 `input_schema`，用 `_serde.build_serde_allowlist()` 构造**反序列化类型白名单**，并包到 checkpointer 上。这是防反序列化 RCE 的做法。
- `:1270-1286` 决定 `output_channels` 和 `stream_channels`：单字段 schema 直接用 `__root__`。

### 4. Checkpointer —— 时间机器

`libs/checkpoint/langgraph/checkpoint/base/__init__.py`：

- `Checkpoint`（:93）是一个 `TypedDict`，含 `channel_versions`、`versions_seen`、`pending_writes` 等
- `BaseCheckpointSaver`（:199）定义 `get` / `get_tuple` / `list` / `put` / `put_writes` / `delete_thread` / `prune` 以及各自的 async 版本（:441-585）
- `empty_checkpoint()`（:843）与 `create_checkpoint()`（:858）是构造入口
- `uuid6`（`base/id.py:79`）是自研的、可按时间排序的 checkpoint id 生成，保证 checkpoint 历史天然有序

最短的参考实现是 `checkpoint/memory/__init__.py:34` 的 `InMemorySaver`，几百行，把「线程 -> checkpoint 列表 -> pending writes」这套模型讲完了。

线程模型：`config = {"configurable": {"thread_id": "..."}}`（见 `graph/state.py:1204-1214` 的注释）。`thread_id` 是同一份状态的标识，复用它就累加状态（对话记忆），换一个就独立开跑。

### 5. Send / Command / interrupt —— 三个控制流原语

都在 `libs/langgraph/langgraph/types.py`：

- `Send`（:738）—— 动态扇出。节点返回 `Send("node_name", payload)` 可以在运行时凭空造出一批并行任务。对应实现在 `_algo.py:937` 的 `prepare_push_task_send()`（还有函数式版 `:799` `prepare_push_task_functional`）。这是 map-reduce 风格 Agent 的底座。
- `Command`（:833）—— 一个返回值里同时携带「状态更新 + 跳转目标」，替代传统的「边 + 状态更新分离」写法。
- `interrupt(value, response_schema=...)`（:903，重载在 :896/:900）—— Human-in-the-loop 的入口。它在节点内抛出中断，`_algo.py:156` 的 `should_interrupt()` 在调度侧判断是否要停。恢复时用 `Command(resume=...)`。
- `Interrupt`（:584）、`StateSnapshot`（:711）、`PregelTask`（:665）是配套的数据结构。
- `RetryPolicy`（:427）、`CachePolicy`（:530）、`TimeoutPolicy`（:461）是标注在节点上的三个横切策略。

### 6. Functional API —— 另一条编译到 Pregel 的路

`func/__init__.py` 导出 `task`（:110）和 `entrypoint`（:262）。被 `@entrypoint` 装饰的函数最终由 `entrypoint.__call__`（:516）返回一个 `Pregel`。它的特殊之处在 docstring（:262-300）里：函数可以声明注入参数 `config`、`previous`（同一 thread 上一次的返回值）、`runtime`，并可以用 `entrypoint.final` 让「返回值」和「存进 checkpoint 的值」不同。

Functional API 和 Graph API 不是两套引擎，是**两套语法糖 + 同一个 Pregel 后端**。想验证这一点，直接对比 `graph/state.py:1177` 的 `compile()` 和 `func/__init__.py:516` 的 `__call__`。

## 最小可运行入口

安装（来自 `README.md:26` 与 `libs/langgraph/README.md` 的 "Quick Install" 段）：

```bash
pip install -U langgraph
# 或
uv add langgraph
```

带开发服务器（来自 `libs/cli/README.md` 的 "Quick Install" 与 "Commands" 段）：

```bash
uv add "langgraph-cli[inmem]"
langgraph dev          # --host 127.0.0.1 --port 2024 --no-reload 可选
```

环境变量：核心包 `langgraph` 自己**不需要**任何 LLM key，它只依赖 langchain-core。要接真模型需要另装 provider 包。

最小 demo A —— 纯 StateGraph，不需要任何 LLM（出自 `libs/langgraph/langgraph/graph/state.py:159-199` 的 `StateGraph` docstring，原样摘录）：

```python
from typing_extensions import Annotated, TypedDict
from langgraph.graph import StateGraph
from langgraph.runtime import Runtime


def reducer(a: list, b: int | None) -> list:
    if b is not None:
        return a + [b]
    return a


class State(TypedDict):
    x: Annotated[list, reducer]


class Context(TypedDict):
    r: float


graph = StateGraph(state_schema=State, context_schema=Context)


def node(state: State, runtime: Runtime[Context]) -> dict:
    r = runtime.context.get("r", 1.0)
    x = state["x"][-1]
    return {"x": x * r * (1 - x)}


graph.add_node("A", node)
graph.set_entry_point("A")
graph.set_finish_point("A")
compiled = graph.compile()
print(compiled.invoke({"x": 0.5}, context={"r": 3.0}))   # {'x': [0.5, 0.75]}
```

最小 demo B —— 接 LLM（出自 `libs/prebuilt/README.md` 的 "## Agents" 段）：

```bash
uv add langchain-anthropic
```

```python
from langchain_anthropic import ChatAnthropic
from langgraph.prebuilt import create_react_agent


def search(query: str):
    """Call to surf the web."""
    if "sf" in query.lower() or "san francisco" in query.lower():
        return "It's 60 degrees and foggy."
    return "It's 90 degrees and sunny."


model = ChatAnthropic(model="claude-3-7-sonnet-latest")
app = create_react_agent(model, [search])
app.invoke({"messages": [{"role": "user", "content": "what is the weather in sf"}]})
```

注意：demo B 在本快照里**会打废弃警告**，见下一节。

## 值得抄的工程手法

1. **并行写入靠排序 + 版本号做到确定性重放**。`apply_writes()` 先把任务按 `task.path` 排序再统一提交（`pregel/_algo.py:255`），每个通道维护 `channel_versions`、每个任务维护 `versions_seen`（`_algo.py:262-268`）。好处是「同一份输入重跑一定得到同一份状态」，这是 durable execution、缓存、断点续跑三件事的共同前提。代码在 `pregel/_algo.py:233-344`。

2. **把「调度」拆成可执行的状态机，而不是一个 while 循环**。`PregelLoop` 用显式的 `status` 七态机（:295-303）+ `tick()`/`after_tick()` 两段方法（:650 / :700）表达一个超步。好处是 interrupt、resume、replay、draining 这些语义都变成了状态转移，而不是散落在循环里的 if。代码在 `pregel/_loop.py:170-1650`。

3. **多套用户 API 编译到同一个中间表示**。Graph API（`graph/state.py:1177`）和 Functional API（`func/__init__.py:516`）都产出 `Pregel`。好处是所有横切能力（checkpoint、stream、cache、retry、interrupt）只需在 IR 上实现一次，不会在两套 API 间漂移。这也是你以后做框架设计时最该抄的一条。

4. **Checkpoint 的持久化顺序用 future 显式约束**。`_loop.py:197-237` 的注释写明：`_put_checkpoint` 会先等待产生 delta 写入的 future，写失败则存盘也失败，绝不允许 checkpoint 先于产生它的写入变持久。好处是避免「恢复后数据永久缺一块」这类最难查的 bug。代码在 `pregel/_loop.py:1237` 与 `SyncPregelLoop._checkpointer_put_after_previous`（:1712）。

5. **编译期收集 schema 构造反序列化白名单**。`compile()` 遍历所有 schema 与节点签名类型，生成 `serde_allowlist` 并包到 checkpointer 上（`graph/state.py:1234-1254` + `checkpoint/serde/`）。好处是 msgpack 反序列化不再是一个任意类加载面，同时运行时零反射开销。

6. **废弃改造用统一的类型化装饰器承载**。`LangGraphDeprecatedSinceV10` / `LangGraphDeprecatedSinceV05` + `DeprecatedKwargs` 让老参数（如 `config_schema`）在 `__init__` 里优雅降级并给出迁移提示（`graph/state.py:225-226`、`func/__init__.py:449-465`、`prebuilt/chat_agent_executor.py:274-277`）。对一个每周发版的框架，这决定了用户敢不敢升级。

## 阅读路径

按这个顺序读，每一步都建立在上一步的概念之上：

1. `libs/langgraph/langgraph/graph/state.py` —— 先看 `StateGraph` 的 docstring（:131-199）和 `compile()`（:1177）。**为什么现在读**：这是你唯一日常会写的代码面，先建立「我写的那几行到底编译成了什么」的直觉。
2. `libs/langgraph/langgraph/pregel/main.py` —— 只读 `Pregel` 类的 docstring（:460-713）。**为什么现在读**：它是全项目的概念规范，BSP 三段式、actor、channel 都在这里讲清，比外部文档准。
3. `libs/langgraph/langgraph/channels/base.py` + `channels/last_value.py` —— `BaseChannel`（:19）的 `update`/`consume`/`finish`/`is_available` 四个方法，配 `LastValue`（last_value.py:20）。**为什么现在读**：通道是 LangGraph 状态的原子单位，不懂这四个方法后面 `_algo.py` 完全看不懂。
4. `libs/langgraph/langgraph/pregel/_read.py` —— `ChannelRead`（:25）和 `PregelNode`（:97）。**为什么现在读**：这是「actor 怎么读通道」，是 StateGraph 节点与 Pregel 节点之间的接缝。
5. `libs/langgraph/langgraph/pregel/_algo.py` —— `apply_writes()`（:233）与 `prepare_next_tasks()`（:391）。**为什么现在读**：调度的核心算法，看完你就明白「边」在运行时其实不存在，只有通道版本。
6. `libs/langgraph/langgraph/pregel/_loop.py` —— `PregelLoop.tick()`（:650）、`after_tick()`（:700）、`_first()`（:921）、`_put_checkpoint()`（:1237）。**为什么现在读**：把算法和持久化缝成状态机，中断与恢复的全部逻辑都在这。
7. `libs/langgraph/langgraph/pregel/_runner.py` —— `PregelRunner.tick()`（:173）、`atick()`（:359）、`commit()`（:575）。**为什么现在读**：并发执行与错误传播（`_panic_or_proceed` :657）的落点。
8. `libs/checkpoint/langgraph/checkpoint/base/__init__.py` —— `Checkpoint`（:93）、`BaseCheckpointSaver`（:199）。**为什么现在读**：把「时间机器」的契约看清楚，之后所有 saver 都是它的实现。
9. `libs/checkpoint/langgraph/checkpoint/memory/__init__.py` —— `InMemorySaver`（:34）。**为什么现在读**：最短的完整实现，读完就知道自己接公司数据库要写哪几个方法。
10. `libs/langgraph/langgraph/types.py` —— `Send`（:738）、`Command`（:833）、`interrupt`（:903）。**为什么现在读**：三个动态控制流原语，做复杂 Agent 才用得上，放最后避免一开始被绕晕。
11. `libs/langgraph/langgraph/func/__init__.py` —— `entrypoint`（:262）。**为什么现在读**：选读。验证「两套 API 一个 IR」这个判断，同时学它的参数注入设计。

## 别踩的坑

- **`create_react_agent` 已经废弃**。`libs/prebuilt/langgraph/prebuilt/chat_agent_executor.py:274-277` 上挂着 `@deprecated("create_react_agent has been moved to `langchain.agents`. Please update your import to `from langchain.agents import create_agent`.")`。网上大量教程和 35 个 notebook 里的写法都基于它，照抄会一路吃警告。把它当「理解 ReAct 图怎么搭」的样例读，新代码按提示走 `langchain.agents`。
- **`config_schema` 已废弃，改用 `context_schema`**。`graph/state.py:155-157` 写明 v0.6.0 起废弃、v2.0.0 移除。你搜到的老代码里凡是 `config_schema=` 的都要改。
- **版本已到 1.x，跨版本资料会互相矛盾**。本快照 `langgraph 1.2.18`，大量中文教程停留在 0.0.x/0.1.x 时代，那时 `checkpointer` 参数、`Command`、`Send` 的语义都与现在不同。以本地代码为准。
- **行号会漂移，用类名/函数名定位**。本文档引用的行号基于 commit `6f2df05`。`pregel/main.py` 4,556 行、`pregel/_loop.py` 2,172 行、`pregel/_algo.py` 1,459 行，这几个大文件几乎每个 release 都动。搜索时优先搜 `def apply_writes`、`class PregelLoop`、`class StateGraph` 这类锚点。
- **不要从 `libs/sdk-py`、`libs/sdk-js` 入手**。那是访问 LangSmith 托管服务的客户端，21,447 行 Python，跟核心执行模型没有任何关系，进去会迷路。
- **不要从 `examples/*.ipynb` 学架构**。35 个 notebook 是用法演示，很多基于旧 API，且 notebook 天然不利于理解分层。想看架构就按上面的阅读路径走源码。
- **初学不要碰 `libs/checkpoint-sqlite` 的 `_delta.py`**。增量通道（DeltaChannel）的落盘是 1.x 新引入的性能优化，逻辑复杂且与核心模型无关。先只看 `libs/checkpoint` 的 `memory/` 实现。
- **模型调用不在这个仓库里**。想看 LLM 怎么被调、工具怎么绑定，要去 `langchain-core`（本仓只声明依赖）。误以为 LangGraph 封装了模型层，是初学者最常见的认知错位。
