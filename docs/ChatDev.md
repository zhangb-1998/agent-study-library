# ChatDev 2.0 (DevAll) 技术解析

> 本地快照说明：本地副本是 **ChatDev 2.0 / 代号 DevAll**，不是论文《Communicative Agents for Software Development》里那个 1.0 的"虚拟软件公司"。1.0 已迁到 `chatdev1.0` 分支（README.md 第 3-11 行明确说明）。本文所有结论来自本地代码实测，涉及 1.0 的部分一律标注【待核】。

## 一句话定位

ChatDev 2.0 是一个 **零代码多智能体编排平台**：把"多角色协作"从硬编码的 Python 类改写成 **YAML 描述的有向图**，节点是角色（agent / human / literal / python / loop_counter / subgraph 等），边是带条件与上下文控制的数据流，用户不写一行代码就能拼出软件开发、数据可视化、3D 生成、深度调研等流水线。

在 Agent 技术栈里它属于 **编排层（Orchestration / Workflow Runtime）**，位于 LLM API 之上、具体 Agent 应用之下：它不管 prompt 怎么写得好，只负责"谁在什么时候、看到什么上下文、调用哪些工具、把结果交给谁"。对比同流派的其他项目，它的特点是 **图是一等公民、带环、可持久化、可视化编辑**。

## 事实卡片

| 项 | 值 |
| --- | --- |
| 主语言 | Python 3.12（pyproject.toml `requires-python = ">=3.12,<3.13"`）+ Vue 3 / Vite 前端 |
| 代码行数 | Python 197 个文件 / 31149 行（不含前端）；另有 `.vue` 18 个、`.js` 12 个、`.yaml` 47 个、`.md` 48 个 |
| 源文件数 | 342（含前端、配置、文档） |
| 主要依赖 | `openai`、`fastapi==0.124.0`、`pydantic==2.12.5`、`fastmcp` + `mcp`、`faiss-cpu`、`mem0ai>=1.0.9`、`google-genai`、`uvicorn`、`pyyaml`、`tenacity`、`networkx`、`cartopy`、`pandas/numpy/seaborn/matplotlib`、`pygame` |
| 许可证 | Apache-2.0（LICENSE 文件头 + pyproject `license = { text = "Apache-2.0" }`） |
| 本地快照 commit | `a2ea4a2 Merge pull request #687 from NA-Wen/new_pr` |
| star 数 | 【待核】 |
| 包名 / 版本 | pyproject `name = "DevAll"`，`version = "0.1.0"` |

## 目录地图

顶层与关键二级目录（每个一句话）：

| 路径 | 作用 |
| --- | --- |
| `run.py` | CLI 入口：解析 `--path/--name/--attachment/--inspect-schema`，读一个 YAML，交互式收任务 prompt，跑图 |
| `server_main.py` | Web 服务启动器（uvicorn + `--reload` 只监听源码目录） |
| `server/` | FastAPI 后端：`app.py` 建 app，`bootstrap.py` 挂路由，`routes/`（execute / workflows / artifacts / websocket / sessions / batch / uploads / vuegraphs / tools），`services/`（session_execution、websocket_*、workflow_run_service、prompt_channel 等） |
| `runtime/` | **核心运行时**：`node/`（agent 执行器 + memory/providers/skills/thinking/tool 子模块）、`edge/`（conditions 条件 + processors 载荷变换）、`bootstrap/schema.py`、`sdk.py`（Python SDK `run_workflow`） |
| `entity/` | 配置与消息的数据模型：`configs/`（node / edge 的 pydantic-ish dataclass 配置）、`graph_config.py`（不可变配置）、`messages.py`（`Message` / `MessageBlock` / `AttachmentRef`）、`tool_spec.py`、`enums.py` |
| `workflow/` | 编排逻辑：`graph.py`（`GraphExecutor`）、`graph_context.py`（可变运行时状态）、`graph_manager.py` + `topology_builder.py`（建图/分层）、`cycle_manager.py`（环检测与迭代计数）、`executor/`（dag / cycle / dynamic_edge / parallel）、`runtime/`（execution_strategy、runtime_builder、result_archiver）、`hooks/workspace_artifact.py` |
| `functions/` | 可被 YAML 引用并注入给模型的函数：`function_calling/`（file、web、code_executor、deep_research、user、video、weather、uv_related …）、`edge/`（边条件用到）、`edge_processor/`（边载荷变换用到） |
| `tools/` | 三个离线脚本：`validate_all_yamls.py`（跑 `make validate-yamls`）、`sync_vuegraphs.py`（把 yaml 同步给前端）、`export_design_template.py` |
| `mcp_example/` | 只有一个 `mcp_server.py`，用 FastMCP 起了一个暴露 `rand_num(a,b)` 的 demo 服务，配合 `yaml_instance/demo_mcp.yaml` 演示 `mcp_remote` 工具接入 |
| `frontend/` | Vue 3 + Vite 的可视化控制台（画布编排、实时日志、产物查看、human-in-the-loop），`src/pages`、`src/components`、`src/router` |
| `yaml_instance/` | 42 个现成工作流样例，是**最好的学习材料**：`ChatDev_v1.yaml`（经典瀑布流复刻）、`MACNet_v1.yaml`、`deep_research_v1.yaml`、`demo_*.yaml`（逐个演示一个特性） |
| `yaml_template/` | 只有 `design.yaml`，新工作流的起始模板 |
| `check/` | `check.py:load_config` 加载 + 校验 YAML，`check_yaml.py` / `check_workflow.py` |
| `schema_registry/` | 所有节点/边/配置的 schema 注册表，前端表单和 `--inspect-schema` 都从这里取 |
| `utils/` | 横切能力：`function_manager.py`（动态加载函数）、`function_catalog.py`（自省函数签名生成 tool schema）、`vars_resolver.py`（`${VAR}` 替换）、`registry.py`（通用注册表）、`token_tracker.py`、`workspace_paths.py`（路径越界防护）、`code_execution.py`（危险代码执行门禁） |
| `docs/user_guide/` | `en/` 与 `zh/` 双语用户手册 |

## 核心架构

```
                         +--------------------------------+
                         |  frontend/ (Vue3 + Vite)       |
                         |  画布编排 / 实时日志 / HITL     |
                         +---------------+----------------+
                                         | REST + WebSocket
                         +---------------v----------------+
                         | server/  (FastAPI)             |
                         | routes -> services             |
                         +---------------+----------------+
                                         |
   yaml_instance/*.yaml                  |  run.py / runtime/sdk.py
            |                            |
            v                            v
   +----------------+          +---------------------------+
   | check/check.py |  load    | entity/graph_config.py    |
   | 校验 + ${VAR}  |--------->| GraphConfig (不可变配置)   |
   +----------------+          +------------+--------------+
                                            |
                                            v
                               +---------------------------+
                               | workflow/graph_context.py |
                               | GraphContext (可变状态)    |
                               | directory / outputs / ... |
                               +------------+--------------+
                                            |
              +-----------------------------+----------------------------+
              | workflow/graph_manager.py + topology_builder.py          |
              |  分层 layers、start_nodes、cycle_detection               |
              +-----------------------------+----------------------------+
                                            |
                               +------------v--------------+
                               | workflow/graph.py         |
                               | GraphExecutor.run()       |
                               +------+-------------+------+
                                      |             |
                   执行策略            |             |  边处理
        +-----------------------------+             +------------------+
        |                                                              |
   dag_executor / cycle_executor / dynamic_edge_executor          runtime/edge/
   (parallel_executor 做层内并发)                          conditions/ + processors/
        |                                                              |
        v                                                              v
 +-------------------------------------+                   +----------------------+
 | runtime/node/executor/              |                   | carry_data           |
 |  agent_executor   (LLM + tools)     |                   | keep_message         |
 |  human_executor   (HITL)            |                   | clear_context        |
 |  literal_executor (静态 prompt)      |                   | clear_kept_context   |
 |  python_executor  (跑代码)           |                   | condition: keyword / |
 |  subgraph_executor                  |                   |  function / regex    |
 |  loop_counter / loop_timer / passth.|                   +----------------------+
 +-----------------+-------------------+
                   |
      +------------+------------+----------------+
      v            v            v                v
  providers/    memory/      thinking/        tool/
  openai,       simple,      self_reflection  tool_manager
  gemini        file,        (reflection)     + functions/
                 blackboard,                    function_calling/*
                 mem0
                   |
                   v
        WareHouse/<name>_<ts>/  (node_outputs.yaml, workflow_summary.yaml,
                                 code_workspace/, attachments/, logs)
```

## 关键抽象与执行流

### 1. 配置层：`GraphConfig` vs `GraphContext`

- `entity/graph_config.py` 的 `GraphConfig.from_definition(design.graph, name=..., output_root=..., source_path=..., vars=...)`：不可变的配置数据（graph id、is_majority_voting、log_level、metadata、vars）。
- `workflow/graph_context.py:37` `GraphContext.__init__`：可变运行时状态，`nodes / edges / layers / topology / outputs / subgraphs / has_cycles / cycle_execution_order / directory`。第 64-72 行用 `validate_path_component` + 时间戳生成输出目录 `WareHouse/<name>_<YYYYmmddHHMMSS>`。
- `graph_context.py:96 record()` 把 `node_outputs.yaml` + `workflow_summary.yaml` 落盘；`final_message()` 在第 122 行汇总"N 个节点输出 / M 个终止节点"。

### 2. 建图：`GraphManager` + `topology_builder` + `CycleManager`

- `workflow/graph.py:263-265`：`GraphManager(self.graph).build_graph()`，内部用 `topology_builder.py` 分层，用 `cycle_manager.py` 做环检测。
- `workflow/cycle_manager.py`：`CycleDetector.detect_cycles`（第 74 行起，Tarjan 强连通分量，第 90 行 `_has_self_loop`）、`CycleManager.initialize_cycles`（第 141 行）、`activate_cycle`（第 207 行）、`deactivate_cycle`（第 213 行）。环的迭代次数由节点 `loop_counter` 的 `max_iterations` 控制。

### 3. 执行：`GraphExecutor`

`workflow/graph.py:53` `class GraphExecutor`。

- 入口：`GraphExecutor.execute_graph(graph, task_prompt)`（第 126 行，classmethod）→ `_execute` → `run(task_prompt)`。
- `run()` 第 260-342 行是主干：
  1. `GraphManager.build_graph()`；
  2. `_prepare_edge_conditions()`（第 344 行）把 YAML 里 `condition: {type: keyword, config: {any: [...]}}` 编译成 `EdgeConditionManager` 实例，`process:` 编译成 payload processor；
  3. `_build_memories_and_thinking()`（第 143 行）→ 全局 memory store、thinking manager、每个 agent 的 `MemoryManager`、节点执行器；
  4. 起始节点注入任务消息（第 291-298 行，`node.start_triggered = True` + `append_input`）；
  5. **三种执行策略**三选一（第 301-326 行）：`MajorityVoteStrategy`（`is_majority_voting`，`workflow/runtime/execution_strategy.py:67`）、`CycleExecutionStrategy`（有环，第 39 行）、`DagExecutionStrategy`（无环，第 14 行，内部是 `workflow/executor/dag_executor.py` 按层 + `parallel_executor.py` 层内并发）；
  6. `_collect_all_outputs()`（第 692 行）→ `get_final_output()`（第 738 行）→ `_save_memories()` → `ResultArchiver.export()`（第 339 行）。

### 4. 单节点执行与消息流

- `workflow/graph.py:549 _execute_node(node)`：取 `node.input` → `reset_triggers()` → `_process_result`（第 659 行，委派给 `NodeExecutorFactory` 选出的执行器）→ 输出挂 source → **按节点的 `context_window` 清理输入上下文**（第 596-605 行：`context_window == -1` 表示无限保留）→ 遍历出边调 `_process_edge_output`（第 396 行）→ 若 `context_window != 0` 再补一条指向自己的 "pseudo edge"（第 644-657 行），这就是一份输出能回流成自己下一轮上下文的机制。
- 边语义（`yaml_instance/ChatDev_v1.yaml` 的 edges 段就是活教材）：
  - `trigger: true/false` —— 是否触发目标节点执行（`false` 只把消息排队塞进目标 input）；
  - `condition` —— 字符串 `'true'` 或 `{type: keyword, config: {any: ["<INFO> Finished"], case_sensitive: true}}`（`runtime/edge/conditions/keyword_manager.py`）；
  - `carry_data` —— 是否把源消息内容带过去；
  - `keep_message` —— 消息是否标记为 keep（配合 `clear_kept_context` 使用）；
  - `clear_context` / `clear_kept_context` —— 到达目标前清空其历史输入（实现在 `runtime/edge/conditions/base.py:207 _clear_target_context`）。
- `runtime/edge/conditions/base.py:29 EdgeConditionManager`，`process()`（第 38 行，子类实现）、`transform_payload()`（第 47 行，走 processor）、`_process_with_condition()`（第 94 行，先 `record_edge_process` 再求值和投递）。

### 5. Agent 节点：`AgentNodeExecutor`

`runtime/node/executor/agent_executor.py:43`（1361 行，是全仓最核心的文件）。

`execute()`（第 46 行）流程：

1. `ProviderRegistry.get_provider(agent_config.provider)`（第 66 行）—— 内置只注册了 `openai`；`gemini` 需要 `google-genai` 装了才注册（见 `runtime/node/agent/providers/builtin_providers.py:7-29`，装不上会 print 一行警告而非报错）。
2. `_inputs_to_text(inputs)`（继承来的，见 `runtime/node/executor/base.py:110`）—— **上游消息被拼成 `=== INPUT FROM <source> (<role>) ===` 分隔的文本**，这是多角色"对话"的物理形态。
3. `_prepare_prompt_messages`（第 184 行）/ `_prepare_message_conversation`（第 209 行）：拼 `[system_prompt, user_input]`。`_build_system_prompt`（第 261 行）把 `node.role` 直接当 system prompt，再追加工能技能 XML。
4. 可选 `thinking`（第 98 行 `_apply_pre_generation_thinking`）。
5. `_apply_memory_retrieval`（第 504 行）：命中 memory 时，MESSAGES 模式插入一条 memory 消息，PROMPT 模式则把检索结果 **拼到最后一条 user 消息前面**。
6. `_invoke_provider` → 若有 tool_calls 走 `_handle_tool_calls`（第 563 行，含 `_execute_tool_batch` 第 619 行）。
7. 可选 `_apply_post_generation_thinking`（第 1197 行），再 `_update_memory`（第 1278 行，把 input/output 快照写回）。
8. 异常兜底：第 172-180 行捕获任何异常，**把错误文本当作 assistant 消息继续往下传**，不让整图崩掉。

`_prepare_call_options`（第 229 行）只把 YAML 的 `params` 原样透传，**没有默认 temperature / max_tokens**（第 240-241 行被注释掉了）。

### 6. 节点类型（注册表驱动）

`runtime/node/registry.py:46 register_node_type(name, *, config_cls, executor_cls, capabilities)`，实际注册点在 `runtime/node/builtin_nodes.py`：

| type | 注册行 | 执行器 |
| --- | --- | --- |
| `agent` | 27 | `AgentNodeExecutor` |
| `human` | 38 | `HumanExecutor`（human-in-the-loop，`utils/human_prompt.py`） |
| `subgraph` | 49 | `SubgraphExecutor` |
| `python` | 60 | `PythonExecutor`（受 `MAC_ALLOW_UNSAFE_CODE_EXECUTION` 门禁） |
| `passthrough` | 71 | `PassthroughExecutor` |
| `literal` | 79 | `LiteralExecutor`（吐静态 prompt，角色是 `user` 或 `assistant`） |
| `loop_counter` | 87 | `LoopCounterExecutor` |
| `loop_timer` | 95 | `LoopTimerExecutor` |

`NodeExecutorFactory.create_executors`（`runtime/node/executor/factory.py:16`）按 type 批量实例化。

### 7. 角色扮演：prompt 写在 YAML 里

**这是 2.0 和 1.0 最大的差别：没有 `chat_chain.py`，没有 `camel/` 角色目录，role prompt 就是 YAML 节点上的 `role:` 字段。**

- 角色 system prompt = 节点的 `role`（`agent_executor.py:263`）。
- 每个阶段的"用户指令" = 一个 `type: literal` 节点，`config.role` 是 `user` 或 `assistant`，`config.content` 是那段指令文本。
- 共享前缀用变量注入：`yaml_instance/ChatDev_v1.yaml` 第 1017-1021 行定义 `vars.COMMON_PROMPT`（"ChatDev is a software company powered by multiple intelligent agents, such as chief executive officer..."），每个角色 prompt 开头写 `${COMMON_PROMPT}`，由 `utils/vars_resolver.py` 的 `PlaceholderResolver` 递归展开（第 56 行 `_resolve_string`，支持 `${VAR}` 单独占位和文本内嵌替换，先查 `vars` 再查环境变量，循环引用会抛 `ConfigError`）。
- `${BASE_URL}` / `${API_KEY}` 也是同一套机制，从 `.env` 读（`utils/env_loader.py:13 load_dotenv_file`）。

### 8. 经典瀑布流怎么表达（"聊天链"的 2.0 形态）

`yaml_instance/ChatDev_v1.yaml`（1022 行）就是论文流水线的图化复刻。它的做法：

- **阶段 = literal 节点**：`Coding Phase Prompt for Assistant`（第 429 行）、`Code Complete Phase for Assistant`（第 199 行）、`Code Review Comment Phase Prompt for Assistant`（第 137 行）、`Test Error Summary Phase Prompt for Assistant`（第 247 行）、`Test Modification Phase Prompt for Assistant`（第 401 行）、`Manual Phase Prompt for Assistant/User`（第 297 / 310 行）。每个 literal 的 content 就是原论文里那个 phase prompt（含 `Modality: Code.`、`Programming Language: Python.` 等原句）。
- **角色 = agent 节点**：`Chief Executive Officer`（第 104 行）、`Chief Product Officer`（第 265 行）、`Programmer Coding`（第 214 行）、`Programmer Code Complete`（第 166 行）、`Code Reviewer`（第 360 行）、`Programmer Code Review`（第 38 行）、`Software Test Engineer`（第 458 行）、`Programmer Test Error Summary`（第 323 行）、`Programmer Test Modification`（第 71 行）。角色 prompt 就是原论文角色描述（"You are Chief Executive Officer. ... Your decision-making role involves high-level decisions about policy and strategy..."）。
- **"多轮对话" = loop_counter + 带条件回边**：`Code Review Phase Loop Counter`（`max_iterations: 10`，第 497 行）、`Test Modification Phase Loop Counter`（5，第 158 行）、`Code Complete All Phase Loop Counter`（5，第 7 行）、`Test Phase Loop Counter`（3，第 15 行）。退出靠关键词条件：`Code Reviewer -> Test Error Summary Phase Prompt for Assistant` 的条件是 `any: ["<INFO> Finished"]`（第 924-939 行），`Programmer Code Complete -> Code Review Comment ...` 的条件是 `<INFO> FINISHED`（第 940-955 行）。**这就是"聊天链终止条件"在 2.0 的等价物。**
- **上下文裁剪**：`context_window` 字段。`0` 表示用完即清，`-1` 表示全保留（如 `Programmer Coding` 是 0，`Chief Product Officer` 是 -1）。边上 `clear_context: true` 进一步控制。

想看论文原始 ChatChain 实现（`chat_chain.py`、`camel/` 下的 CEO/CTO/Programmer 角色 `.py` 文件、`codes/test/` 阶段脚本），必须切到 `chatdev1.0` 分支——**本地副本不含这些文件，路径与行号【待核】**。

### 9. Memory 与产物传递

- 抽象：`runtime/node/agent/memory/memory_base.py:193 MemoryBase`（`load/save/retrieve/update`）+ `MemoryItem`（第 118 行，含 `input_snapshot` / `output_snapshot`）+ `MemoryWritePayload`（第 167 行）+ `MemoryRetrievalResult`（第 175 行）。
- `MemoryManager`（第 233 行）管一个 agent 挂多个 store，`retrieve`（第 243 行）里 `_score_memory`（第 290 行）做打分。
- 四种 store 注册在 `runtime/node/agent/memory/builtin_stores.py`：`simple`（进程内，重启即失）、`file`（落盘 + embedding 检索，`file_memory.py` 485 行，有文件 hash 增量索引 `_validate_and_update_index` 第 221 行）、`blackboard`（多节点共享黑板）、`mem0`（Mem0 托管，lazy import）。
- 全局 store 在图级 `graph.memory` 声明（如 `ChatDev_v1.yaml` 第 1010 行 `memory: []`），agent 节点用 `memories: [名字]` 引用（第 67 行等，样例里全是空）。
- 写回发生在 `agent_executor.py:1278 _update_memory`，输入输出都存成 `MemoryContentSnapshot`（含多模态 blocks）。
- **产物（artifacts）不走 memory，走文件系统**：`workflow/hooks/workspace_artifact.py:52 WorkspaceArtifactHook` 在 `_process_result`（graph.py:674-689）的 `before_node` / `after_node` 里对 `code_workspace` 做文件签名比对，自动发现 agent 新建/修改的文件；路径由 `utils/workspace_paths.py:29 resolve_workspace_path` 强制约束在工作区内（防 `../` 穿越，第 32-36 行）。

### 10. 工具：动态加载的 Python 函数

- `utils/function_manager.py:34 FunctionManager.load_functions()`：`rglob("*.py")` 扫 `functions/function_calling/`，`importlib.util.spec_from_file_location` 动态导入，收集**本文件内定义的非下划线开头函数**（第 66-77 行）。
- `utils/function_catalog.py:31 FunctionCatalog` 用 `inspect` 自省签名 + `Annotated[..., ParamMeta(...)]` 生成 OpenAI tool schema（`ParamMeta` 第 15 行）。
- YAML 里引用：`tooling: [{type: function, config: {tools: [{name: "uv_related:All"}, {name: save_file}, ...]}]`（见 `ChatDev_v1.yaml` 第 52-65 行）。
- 三个目录分别对应三种用途，环境变量可覆盖：`MAC_FUNCTIONS_DIR` / `MAC_EDGE_FUNCTIONS_DIR` / `MAC_EDGE_PROCESSOR_FUNCTIONS_DIR`（function_manager.py:11-31）。

### 11. 反思（thinking）

`runtime/node/agent/thinking/builtin_thinking.py:11` 只注册了一种：`reflection` → `SelfReflectionThinkingManager`。配置在 agent 节点的 `thinking:` 字段，`ChatDev_v1.yaml` 里全是空。pre-gen / post-gen 两处挂载点见 `agent_executor.py:99` 与 `:149`。

### 12. MCP 示例的用途

`mcp_example/mcp_server.py` 只有 25 行：用 `FastMCP("Company Simple MCP Server")` 注册了一个 `rand_num(a:int,b:int)->int` 工具，`mcp.run()` 默认 stdio，注释里给了 `uv run fastmcp run simple_server.py --transport 'streamable-http' --port 8001`。

配套 `yaml_instance/demo_mcp.yaml`：节点 A 的 `tooling: [{type: mcp_remote, config: {server: http://127.0.0.1:8001/mcp}}]`，prompt 要求先调工具取随机数再写诗，节点 B 是文学评论家。**用途只有一个：演示 `mcp_remote` 工具类型怎么填。** 它和 TradingAgents 无关，也不是 ChatDev 的必选组件。

### 13. `run.py` 的参数入口

`run.py:38 parse_arguments()`：

| 参数 | 默认 | 说明 |
| --- | --- | --- |
| `--path` | `yaml_instance/net_loop_test_included.yaml` | 工作流 YAML 路径（**该文件本地不存在，见"别踩的坑"**） |
| `--name` | `test_project` | 项目名，决定 `WareHouse/<name>_<ts>/` |
| `--fn-module` | None | 额外提供边条件/处理函数的模块 |
| `--inspect-schema` | False | 打印配置 schema 后退出 |
| `--schema-breadcrumbs` | None | JSON 数组，缩小 schema 范围 |
| `--attachment` | `[]` | 可重复，文件作为附件注入首条消息 |

`main()`（第 77 行）里的顺序：先 `ensure_schema_registry_populated()`（第 21 行，模块级）→ `load_config(args.path, fn_module=...)` → `input("Please enter the task prompt: ")`（第 99 行，**交互式，不可非交互运行**）→ 建 `GraphConfig` / `GraphContext` → `build_task_input_payload` → `GraphExecutor.execute_graph`。

Python SDK 版本：`runtime/sdk.py` 的 `run_workflow(yaml_file, task_prompt, attachments, variables)`，README.md 第 247-260 行给了示例。

## 最小可运行入口

全部摘自 README.md（行号已标注）：

```bash
# 前置：macOS/Linux/WSL/Windows，Python 3.12+，Node.js 18+，包管理器 uv   (README.md:104-109)

# 1. 安装后端依赖（uv 管理 Python）                    README.md:113-116
uv sync

# 2. 安装前端依赖                                      README.md:118-121
cd frontend && npm install

# 3. 配置密钥                                          README.md:123-130
cp .env.example .env
#   在 .env 里设 API_KEY 和 BASE_URL；YAML 里用 ${API_KEY} 引用
#   .env.example 给的示例值：BASE_URL=https://api.openai.com/v1  API_KEY=sk-your-openai-api-key-here

# 4. 启动（推荐 Makefile）                             README.md:134-141
make dev          # 前端 http://localhost:5173

# 4'. 手动启动                                         README.md:145-156
uv run python server_main.py --port 6400 --reload
cd frontend && VITE_API_BASE_URL=http://localhost:6400 npm run dev

# 4''. 命令行跑单个工作流
uv run python run.py --path yaml_instance/demo_code.yaml --name my_proj

# 5. 工具命令                                          README.md:168-183
make help
make sync              # 把 yaml_instance/ 同步到前端库
make validate-yamls    # 校验全部 YAML

# 6. Docker                                            README.md:208-223
docker compose up --build     # 后端 6400 / 前端 5173
```

Python SDK（README.md:247-260）：

```python
from runtime.sdk import run_workflow
result = run_workflow(
    yaml_file="yaml_instance/demo.yaml",      # 注意：该文件本地不存在
    task_prompt="Summarize the attached document in one sentence.",
    attachments=["/path/to/document.pdf"],
    variables={"API_KEY": "sk-xxxx"},
)
print(result.final_message.text_content())
```

## 值得抄的工程手法

1. **"配置即产品"：YAML schema 自举 + 前端表单自动生成**
   - 它怎么做：`schema_registry/registry.py` 收集所有节点/边配置的 `FIELD_SPECS`（如 `entity/configs/node/agent.py` 里的 `ConfigFieldSpec`），`utils/schema_exporter.py:build_schema_response` 输出 JSON schema，前端据此渲染表单，`run.py --inspect-schema` 也能打印。
   - 为什么好：新增一种节点类型只要 `register_node_type(...)` 一处，UI、校验、文档三处自动跟上，不需要前端改代码。
   - 代码在：`runtime/node/registry.py:46`（注册时顺便 `register_node_schema`，第 64 行）、`utils/schema_exporter.py`、`run.py:80-92`。

2. **注册表驱动的三层可插拔：节点类型 / 边条件 / 边处理器 / memory store / provider**
   - 它怎么做：`utils/registry.py:Registry` 是通用注册表，`runtime/node/registry.py`（节点）、`runtime/edge/conditions/registry.py`（条件）、`runtime/edge/processors/registry.py`（处理器）、`runtime/node/agent/memory/registry.py`（memory）、`runtime/node/agent/providers/base.py:81 ProviderRegistry`（provider）各建一个实例。
   - 为什么好：加一个新能力 = 写一个文件 + 一行 `register_*`，执行器里零 `if type == "..."` 分支。
   - 代码在：`runtime/node/executor/factory.py:16`（一句 `iter_node_registrations()` 造出全部执行器）。

3. **把"上下文窗口"做成节点级 + 边级的两级旋钮**
   - 它怎么做：节点有 `context_window`（0=用完即清，-1=全留，N=留 N 条），边有 `carry_data / keep_message / clear_context / clear_kept_context` 四个布尔，外加 `trigger` 决定"排队还是触发"。
   - 为什么好：多 Agent 系统最贵的不是调用次数而是 **上下文膨胀**。ChatDev 让你能精确控制"Programmer 只能看到当前这一轮 review 意见，不能看到前 9 轮的"，成本和行为同时可控。`ChatDev_v1.yaml` 里 `Programmer Code Review` 是 `context_window: -1` 而 `User -> 各角色` 的边是 `keep_message: true`，说明"原始需求"被永久保留、中间过程被裁剪。
   - 代码在：`workflow/graph.py:596-605`、`runtime/edge/conditions/base.py:207`、`entity/configs/edge/edge.py`。

4. **工具 = 目录里的普通 Python 函数，靠自省生成 schema**
   - 它怎么做：`FunctionManager` 扫目录动态导入，`FunctionCatalog` 用 `inspect` + `Annotated[..., ParamMeta(description=..., enum=...)]` 生成 OpenAI function schema，YAML 里只写函数名的字符串。
   - 为什么好：写工具的门槛降到"在 `functions/function_calling/` 下加一个带类型标注的函数"，零装饰器、零注册代码。
   - 代码在：`utils/function_manager.py:34`、`utils/function_catalog.py:31`、`functions/function_calling/file.py:29 FileToolContext`（用 `_context` 注入运行时依赖，是工具拿全局状态的干净做法）。

5. **危险能力用"服务端环境变量门禁"，且明确不从工作流变量读取**
   - 它怎么做：`utils/code_execution.py:12 require_unsafe_code_execution(operation)` 只读 `os.environ["MAC_ALLOW_UNSAFE_CODE_EXECUTION"]`，注释明写 "Never consult workflow variables, tool arguments, or child-process env overrides"，不通过就抛 `SecurityError`。`.env.example` 里注释："Disabled by default ... this is not a sandbox."
   - 为什么好：把"能不能跑代码"的权限边界放在进程启动那一刻，杜绝了"YAML 里写个 env 覆盖就把沙箱开了"这类提权路径。
   - 代码在：`utils/code_execution.py`、`functions/function_calling/uv_related.py:9`、`utils/workspace_paths.py:29`（配套的路径穿越防护）。

6. **Agent 出错不炸图，把错误转成一条 assistant 消息**
   - 它怎么做：`agent_executor.py:172-180` 捕获所有异常（除 `SecurityError` 继续抛），返回一条 `Error calling model ...` 的 assistant 消息。
   - 为什么好：长流程里单节点限流/超时是常态，宁可降级也不让整条流水线白跑。
   - 代价：错误会顺着边污染下游上下文，调试时要看日志而不是看最终输出。

## 阅读路径

按这个顺序读，每个都标注"为什么现在读它"：

1. `README.md:100-262` —— 先建立"这是个平台不是个软件公司脚本"的认知，抄下安装与启动命令。
2. `yaml_instance/ChatDev_v1.yaml` —— **第二优先**。42 个样例里唯一复刻经典瀑布流的，读完就懂"阶段=字面量节点、角色=agent 节点、循环=loop_counter+关键词条件"这套编码方式，后面看 Python 全是它的解释。
3. `run.py`（全 123 行）—— 最短的端到端路径：参数 → load_config → GraphConfig/GraphContext → execute_graph。
4. `entity/graph_config.py` + `workflow/graph_context.py`（共 252 行）—— 搞清"不可变配置 vs 可变状态"的切分，以及输出目录怎么定。
5. `workflow/graph.py:260-342`（`run` 方法）+ `:549-657`（`_execute_node`）—— 编排主干，三种执行策略的分派点和单节点的消息流全在这里。
6. `runtime/node/executor/agent_executor.py:46-208` —— 单个 Agent 的完整生命周期：provider 选择 → prompt 拼接 → thinking → memory 检索 → 调模型 → tool calls。
7. `runtime/node/executor/agent_executor.py:504-561` 与 `:1278-1311` —— memory 的读与写两端，配合 `runtime/node/agent/memory/memory_base.py:193` 看抽象。
8. `runtime/edge/conditions/base.py:38-207` —— 边的条件求值与上下文清理，是"多角色对话如何收敛"的答案。
9. `utils/vars_resolver.py` + `utils/function_manager.py` —— 两个小而通用的机制：`${VAR}` 注入和工具自动发现。
10. `runtime/node/builtin_nodes.py`（102 行）—— 一口气看完 8 种节点类型的注册，作为扩展自己节点类型的模板。
11. `server/routes/execute.py` + `server/services/session_execution.py`（可选）—— 想搞懂 Web 端怎么驱动同一个 `GraphExecutor`、怎么接 human-in-the-loop 时读。

## 别踩的坑

**版本与环境**

- Python 版本被**锁死在 3.12**：pyproject.toml `requires-python = ">=3.12,<3.13"`。用 3.13 会直接解析失败，用 3.11 也不行。
- `fastapi==0.124.0`、`pydantic==2.12.5` 是 **精确 pin**，和别的库装在同一环境里容易冲突，务必用 `uv sync` 的独立 venv，别 `pip install -r requirements.txt` 到系统 Python。
- `cartopy` 是传递依赖里的硬骨头，需要 GEOS/PROJ 系统库，**Windows 上 `uv sync` 大概率卡在这里**。如果只是学编排，可以先注释掉它（它只服务于地图类可视化样例）。
- `faiss-cpu` 在 Python 3.12 上有 SWIG 的 `DeprecationWarning`，pyproject 的 `filterwarnings` 已经显式忽略（注释写明"Awaiting SWIG 4.4 fix"），看到这堆警告不用慌。
- Node.js 18+ 与 `uv` 是硬性前置（README.md:106-109），没有 `uv` 就跑不了 `make dev`。

**文档与代码不一致（实测确认）**

- `run.py:43` 的默认 `--path` 是 `yaml_instance/net_loop_test_included.yaml`，**该文件在本地 yaml_instance/ 的 42 个文件里不存在**。直接 `python run.py` 会报文件不存在，必须显式传 `--path`。
- README.md:252 的 SDK 示例写 `yaml_file="yaml_instance/demo.yaml"`，**该文件也不存在**（目录里是 `demo_code.yaml` / `demo_human.yaml` / `demo_mcp.yaml` 等具体样例，没有裸 `demo.yaml`）。照抄会失败。
- README.md 里"Run with OpenClaw"（第 185-205 行）依赖外部 CLI 工具 `clawdhub`，本地无法验证【待核】。

**论文复现的缺口**

- 论文里的 **ChatChain 类、camel 角色目录、多轮 instructor/assistant 对等对话机制**在这个分支里**全部不存在**，只有 `ChatDev_v1.yaml` 一个行为等价的图化复刻。要复现论文本体必须切 `chatdev1.0` 分支（具体文件路径与行号本地无副本，【待核】）。
- `ChatDev_v1.yaml` 里对应论文的"需求分析与设计（CEO/CPO/CTO 参与）"阶段被大幅简化：图里只有 CEO 和 CPO 两个 agent、且只出现在最后的 Manual 阶段（写使用手册），**没有独立的 Design / 需求确认 / 语言选择子阶段**。想要完整瀑布流得自己照 `yaml_template/design.yaml` 往下补。
- 样例里的模型写死 `name: gpt-4o`（`ChatDev_v1.yaml:41` 等）和 `provider: openai`，`base_url: ${BASE_URL}`。换模型/换厂商必须改 YAML 或改 `.env`。

**成本与实测阻塞点**

- 一次 `ChatDev_v1.yaml` 跑满 = 5 次 Code Complete + 10 次 Code Review + 5 次 Test Modification + 3 次 Test Phase，每个 agent 还挂着 9 个文件操作工具（`ChatDev_v1.yaml:56-65`），**工具轮次没有显式上限**（不像 TradingAgents 有 `max_tool_rounds`）。默认配置下烧掉的 token 量级会远超预期，建议先用 `demo_code.yaml` 这类小样例试。
- `run.py:99` 是 `input("Please enter the task prompt: ")`，**没有非交互入口**；要脚本化必须走 `runtime/sdk.py:run_workflow` 或 HTTP 接口。
- Agent 生成的代码默认**不允许执行**：`.env.example` 里 `MAC_ALLOW_UNSAFE_CODE_EXECUTION=0`，`uv_related` / `python` 节点 / `code_executor` 都会抛 `SecurityError`。开了之后注释明说"this is not a sandbox"，务必在容器里跑。
- 代码执行有超时设计（`LIB_INSTALL_TIMEOUT`，默认 120 秒，见 `functions/function_calling/uv_related.py:12`），装大包会超时。
- 长跑任务会撞上 OpenAI 限流。`entity/configs/node/agent.py` 的 `AgentRetryConfig` 默认 `max_attempts: 5`、`min_wait_seconds: 1.0`、`max_wait_seconds: 6.0`，覆盖 429/5xx 与常见异常名（第 30-53 行），但 YAML 样例里 `retry:` 全是空，走默认。
- 前端默认占 5173、后端默认 6400，README.md:158-164 提示端口占用时需同步改 `--port` 与 `VITE_API_BASE_URL`。
- `mcp_example/mcp_server.py` 只是 demo，主流程不依赖它；但它注册的 `rand_num` 会 `print` 到 stdout——如果你用 stdio 传输，这个 print 会污染协议流。

**其他**

- 仓库里有 `WareHouse/`、`logs/` 等运行时目录，且 `server_main.py:26-44` 专门为 `--reload` 写了排除规则（issue #569），说明"agent 生成的文件触发热重载"是真实踩过的坑。本地调试时别把 `WareHouse` 放在源码目录监听范围内。
- 目录里同时存在 `.env.example` 与 `.env.docker`，Docker 用的是后者（`docker compose up --build`），别改错文件。
