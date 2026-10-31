# LightRAG 技术解析

> 面向有前端 + Node/PHP 背景、转 AI Agent 工程的工程师。所有结论均取自本地快照源码，行号取自快照 commit `aaebfe1`。

## 一句话定位

LightRAG 是一个 **图谱增强的 RAG 引擎（GraphRAG）**，位于技术栈的「知识底座 / 检索中间件层」——不是 Agent 编排框架，没有规划循环与工具调用；它解决的问题是：把原始文档离线抽取成**实体-关系知识图 + 向量索引**，在线阶段用**双层级（实体层 / 关系层）检索**把结构化与语义证据一起喂给 LLM，从而回答「跨段落、需要全局关联」的问题。Agent 层（如 gpt-researcher）可以把它当作可插拔的外部记忆来用。

## 事实卡片

| 项目 | 值 |
|---|---|
| 主语言 | Python（要求 `>=3.10`，见 `pyproject.toml`）；仓库另含 `lightrag_webui/` 的 TS/TSX 前端 |
| 代码行数 | `lightrag/` 核心包共 **164,901 行**（`.py`，排除 `evaluation/`）；其中顶层模块 46,365 行，`kg/` 子包 51,027 行 |
| 源文件数 | `lightrag/` 下 **177** 个 `.py`；仓库总文件数 1,281 |
| 主要依赖 | `nano-vectordb`、`networkx`、`numpy`、`pandas`、`pydantic`、`tiktoken`、`tenacity`、`pipmaster`、`json_repair`、`google-genai`、`aiohttp`、`pypinyin`、`PyYAML`、`python-dotenv`、`xlsxwriter` |
| 许可证 | MIT（`LICENSE` 首行 "MIT License"，`pyproject.toml` 声明 `license = {text = "MIT"}`） |
| 本地快照 commit | `aaebfe1`（`main`，2026-10-14）—— "Merge pull request #4210 from ModestXin/fix/pgvector-filtered-ann" |
| 包版本 | `1.5.8`（`lightrag/_version.py`），API 版本 `0347` |
| star 数 | 【待核】 |

单文件体量提示（决定是否值得通读）：`lightrag.py` 8431 行、`pipeline.py` 8228 行、`utils.py` 8293 行、`operate.py` 7126 行、`utils_graph.py` 3735 行、`kg/postgres_impl.py` 10177 行。

## 目录地图

顶层：

| 路径 | 装什么 |
|---|---|
| `lightrag/` | 核心 Python 包，全部逻辑都在这里 |
| `lightrag_webui/` | React 前端（Bun 构建），非核心，读核心时跳过 |
| `examples/` | 20 个可运行脚本，官方只支持 `lightrag_openai_demo.py` 与 `lightrag_openai_compatible_demo.py` |
| `docs/` | 部署、API、分块策略、Parser 插件、设计文档（`docs/design/`） |
| `prompts/` | 提示词模板的外部化目录 |
| `k8s-deploy/` | Kubernetes 部署清单 |
| `reproduce/` | 论文实验复现脚本 |
| `Dockerfile` / `Dockerfile.lite` / `Dockerfile.postgres` | 三种镜像变体 |
| `docker-compose*.yml` | 一体化 / 分体部署编排 |
| `config.ini.example`、`env.example` | 配置样板 |

`lightrag/` 关键二级：

| 路径 | 装什么 |
|---|---|
| `lightrag.py` | `LightRAG` 主类（dataclass），对外暴露 `insert` / `query` 门面 |
| `pipeline.py` | `_PipelineMixin`：文档入队、三级级联队列批处理编排 |
| `operate.py` | 实体关系抽取、关键词抽取、`kg_query` / `naive_query`、上下文四阶段构建 |
| `base.py` | 四种存储抽象基类 + `QueryParam` / `QueryResult` 等数据契约 |
| `llm_roles.py` | `_RoleLLMMixin`：EXTRACT / KEYWORD / QUERY / VLM 四个角色可配独立 LLM |
| `kg/` | 存储后端实现 + 跨进程共享状态（锁、pipeline status） |
| `llm/` | 15 个 Provider 适配（openai / ollama / gemini / anthropic / azure / bedrock / hf / jina / lmdeploy / lollms / nvidia / voyageai / zhipu / llama_index） |
| `api/` | FastAPI 服务端：`lightrag_server.py` + `routers/`（document/graph/query/ollama_api） |
| `parser/` | 解析引擎（native / MinerU / docling）+ 插件注册表 + `cli.py` 离线调试 |
| `chunker/` | 分块策略实现 |
| `tools/` | 运维工具：`rebuild_vdb.py`、`migrate_graph_storage.py`、`kg_integrity_repair.py` 等 |
| `sidecar/` | 多模态 sidecar（`*.parsed/`）读写 |
| `evaluation/` | RAGAS 等评测集成 |

## 核心架构

```
                       ┌─────────────────────────────────────────────┐
  接入层                │  API Server (lightrag/api/)                │
                       │  lightrag_server.py + routers/              │
                       │  document_routes / query_routes / graph...  │
                       └───────────────┬─────────────────────────────┘
                                       │  或直接用 SDK: lightrag.LightRAG
                                       v
  ═════════════════════════════════════════════════════════════════════════
  一、离线索引流水线  (pipeline.py : _PipelineMixin)
  ═════════════════════════════════════════════════════════════════════════
   apipeline_enqueue_documents (pipeline.py:740)
        │ 1 校验/生成 doc_id (MD5)  2 去重  3 建初始状态  4 入队
        v
   ┌────────────────────────────────────────────────────────────────┐
   │  _run_pipeline_batch (pipeline.py:2619)                      │
   │  三层级联队列，每层一组 worker + 有界 Queue                   │
   │                                                              │
   │  Layer 1  PARSE    _parse_worker   (pipeline.py:4409)        │
   │           native / mineru / docling -> 正文 + sidecar        │
   │                    │                                        │
   │                    v  q_analyze (maxsize=queue_size_analyze)   │
   │  Layer 2  ANALYZE  _analyze_worker (pipeline.py:4805)        │
   │           analyze_multimodal (VLM / EXTRACT 角色)            │
   │                    │                                        │
   │                    v  q_process (maxsize=queue_size_insert)  │
   │  Layer 3  PROCESS  _process_worker (pipeline.py:4950)        │
   │           process_single_document (pipeline.py:4991)          │
   │             分块 -> extract_entities -> merge_nodes_and_edges │
   │             状态机 PROCESSING -> PROCESSED / FAILED            │
   └──────────────────────────────────────────────────────────────┘
        │                                        ▲
        │ 并发闸门：asyncio.Semaphore(max_parallel_insert)
        │ 入队闸门：_reserve_ingress_slot (pipeline.py:638) / admission_middleware
        v
  ═════════════════════════════════════════════════════════════════════════
  三、存储抽象层  (base.py)               实现在 kg/
  ═════════════════════════════════════════════════════════════════════════
   ┌────────────────────┬──────────────────────┬──────────────────────┐
   │ BaseKVStorage      │ BaseVectorStorage   │ BaseGraphStorage    │
   │ (base.py:535)      │ (base.py:334)       │ (base.py:671)       │
   │ get_by_id(s)       │ query / upsert      │ has_node/edge       │
   │ filter_keys        │ get_by_id(s)        │ node_degree         │
   │ upsert / delete    │ delete_entity       │ get_node/get_edge   │
   │ is_empty           │ get_vectors_by_ids  │ upsert_node/edge    │
   │                    │                     │ get_knowledge_graph │
   ├────────────────────┴──────────────────────┴──────────────────────┤
   │ DocStatusStorage (base.py:1558, 继承 BaseKVStorage)            │
   │ get_status_counts / get_docs_by_statuses / 分页游标            │
   └────────────────────────────────────────────────────────────────┘
     默认实现:  JsonKVStorage / JsonDocStatusStorage /
                NanoVectorDBStorage / NetworkXStorage
     一体化:    Postgres(KV+Vector+Graph+DocStatus) / Mongo / OpenSearch
     向量专用:  Milvus / Qdrant / Faiss / Noop
     图专用:    Neo4J / Memgraph / PGTable

  ═════════════════════════════════════════════════════════════════════════
  二、在线检索模式  (operate.py)
  ═════════════════════════════════════════════════════════════════════════
   query(4819) -> aquery(4843) -> aquery_llm(5114)
                                -> aquery_data(4901)  只取上下文

   get_keywords_from_query (operate.py:4975)
        ├── hl_keywords  高层级（主题/全局）
        └── ll_keywords  低层级（实体/细节）

   mode 分派 (aquery_data:5042 / _perform_kg_search:5325)
     naive  -> naive_query (6827)          只检索 chunks_vdb
     local  -> _get_node_data (6200)       ll -> entities_vdb -> 节点 + 邻边
     global -> _get_edge_data  (6529)      hl -> relations_vdb -> 边 + 端实体
     hybrid -> 上两者都跑 + round-robin 合并 (5280:5420-5490)
     mix    -> hybrid + _get_vector_context (5223) 原始块
     bypass -> 空数据，直连 LLM (lightrag.py:5067)

   _build_query_context (6075) 四阶段：
     Search -> Truncate -> Merge chunks -> Build LLM context
```

三层之间唯一的耦合点是 `base.py` 的四个抽象基类：流水线只依赖抽象，检索层也只依赖抽象，换后端不用改任何上层代码。

## 关键抽象与执行流

### 1. `QueryParam` — 检索契约入口（`lightrag/base.py:90`）

```python
@dataclass
class QueryParam:
    mode: Literal["local", "global", "hybrid", "naive", "mix", "bypass"] = "mix"
```

注意三点与老资料不同：

- **默认 mode 是 `mix`，不是 `hybrid`**（`base.py:93`，默认值字符串 `"mix"`）。
- 实际有 **6 种**模式，不是 4 种；`mix` 与 `bypass` 是后加的。
- Token 预算分三级：`max_entity_tokens`(6000) / `max_relation_tokens`(8000) / `max_total_tokens`(30000)；`top_k`=40（实体或关系条数）、`chunk_top_k`=20、`related_chunk_number`=5、`kg_chunk_pick_method`=`VECTOR`。默认值集中在 `lightrag/constants.py:57-64`。

`QueryParam` 是纯 dataclass，字段顺序即位置参数顺序。源码在 `base.py:177-181` 有显式注释：新字段只能追加到末尾（因为 `disable_user_prompt_prefix` 已被声明在最后），否则会静默重绑所有位置参数。照抄这个模式时同样要小心。

### 2. 检索模式的差异与实现位置

分派点有两处：`aquery_data`（`lightrag.py:5042-5079`，决定调用哪个顶层函数）与 `_perform_kg_search`（`operate.py:5325` 起，决定跑哪条分支）。

| mode | 用什么关键词 | 检索路径 | 实现位置 |
|---|---|---|---|
| `naive` | 不用（原始 query） | 只查 `chunks_vdb`，返回纯文本块；entities/relations 为空 | `operate.py:6827 naive_query` |
| `local` | `ll_keywords` | `entities_vdb` 查实体 → 批量取节点属性与度数 → 取邻边 → 反查文本块 | `operate.py:6200 _get_node_data` |
| `global` | `hl_keywords` | `relationships_vdb` 查关系 → 批量取边属性 → 取两端实体 → 反查文本块 | `operate.py:6529 _get_edge_data` |
| `hybrid` | 两者都用 | local + global 两条都跑，round-robin 交错合并去重 | `operate.py:5280 _perform_kg_search`（:5420-5490） |
| `mix` | 两者都用 | hybrid + 额外用原始 query 查 `chunks_vdb` 补原始块 | `operate.py:5223 _get_vector_context` |
| `bypass` | 无 | 不检索，数据数组全空，直连 LLM | `lightrag.py:5067-5077` |

几个实现细节值得记住：

- **local 与 global 的排序语义不同**：`_get_node_data` 返回的实体按余弦相似度排序、关系按 rank+weight 排序；`_get_edge_data` 保留向量检索顺序。源码分别在 `operate.py:6253-6256` 与 `operate.py:6585` 附近有注释点明。
- **round-robin 去重键**：实体用 `entity_name`，关系用 `tuple(sorted([src_id, tgt_id]))` 或 `src_tgt` 字段（`operate.py:5434-5483`）。关系去重是**无序对**，所以 A→B 与 B→A 会被视作同一条。
- **关键词可以外部注入**：`get_keywords_from_query`（`operate.py:4975`）在 `query_param.hl_keywords` 或 `ll_keywords` 非空时直接返回，跳过 LLM 抽取。想在 Agent 里复用自己的规划结果时，这是最省钱的入口。
- **空关键词的处理**：`kg_query`（`operate.py:4693`）在 :4755-4762 附近会先按 mode 分别告警，再在两者都空且 query 过短时返回失败响应。

### 3. 实体关系抽取与图合并

**抽取阶段** —— `extract_entities`（`operate.py:3942`）：

1. 入口先检查 `pipeline_status["cancellation_requested"]`，支持批量取消（:3961-3968）。
2. 建立 operation 级 `PipelineStatusLogger` 与 stage 级 `TokenLimitTruncationTally`（:3971-3975），后者最终写回 `doc_status.metadata`，因为 `pipeline_status` 是有界环形缓冲会过期。
3. 从 `global_config` 取 `role_llm_funcs["extract"]` 与 `entity_extract_max_gleaning`（:3986）。
4. **gleaning（补充抽取）**在 :4253 的 `run_gleaning = entity_extract_max_gleaning > 0` 决定；提示词用 `PROMPTS["entity_continue_extraction_user_prompt"]`（:4183、:4211）。默认 `DEFAULT_MAX_GLEANING = 1`（`constants.py:17`），即最多补抽一轮。
5. 结果解析分两路：传统分隔符格式走 `_handle_single_entity_extraction`（:710）/ `_handle_single_relationship_extraction`（:774），结构化 JSON 走 `_process_json_extraction_result`（:907）；缓存复用见 `_get_cached_extraction_results`（:1415）。

**合并阶段** —— `merge_nodes_and_edges`（`operate.py:3514`），顺序是刻意设计的「先可发现、后可变更」：

- **Phase 0**：把候选全集写入 `full_entities` / `full_relations` 并**先 flush**，作为崩溃恢复锚点（docstring 原文 "discoverability before mutation"）。空也写；这里失败则中止，不碰图。
- **Phase 1**：并发处理所有实体 → `_merge_nodes_then_upsert`（:2429）。
- **Phase 2**：并发处理所有关系 → `_merge_edges_then_upsert`（:2782），可能顺带补建缺失实体（`added_entities` 参数）。
- 描述累积过长时走 `_handle_entity_relation_summary`（:372）→ `_summarize_descriptions`（:532）做 LLM 摘要，而不是无限拼接。
- `on_anchors_durable` 回调让调用方在「锚点已持久化、图变更尚未开始」这个唯一窗口里写入 `kg_write_state`，供 fail-closed purge 区分「从未碰过图」与「锚点丢失」。

`_merge_nodes_then_upsert` 里还能看到对脏数据的防守：从 DB 读回的 `entity_type` 先强制转 str、空则置 `UNKNOWN`、含逗号只取第一个非空 token（:2465-2483）。

### 4. 四种存储抽象与各自实现

公共基类 `StorageNameSpace`（`base.py:193`）提供 `namespace` / `workspace` / `global_config` 三元组与 `initialize` / `finalize` / `index_done_callback` 生命周期钩子，以及两个「放弃写入」钩子：`drop_pending_index_ops`（:229）与 `drop_pending_upserts`（:243）。

**(a) KV —— `BaseKVStorage`（`base.py:535`）**

抽象方法：`get_by_id`(553) / `get_by_ids`(581) / `filter_keys`(585) / `upsert`(589) / `delete`(644) / `is_empty`(662)。另有非抽象的 `get_by_id_strict`(556)。

代表实现：`JsonKVStorage`（`kg/json_kv_impl.py`）、`PGKVStorage`（`kg/postgres_impl.py:3251`）、`MongoKVStorage`、`RedisKVStorage`、`OpenSearchKVStorage`。用途：原文 `full_docs`、文本块 `text_chunks`、LLM 响应缓存 `llm_response_cache`、实体/关系块索引。

**(b) 向量 —— `BaseVectorStorage`（`base.py:334`）**

抽象方法：`query`(385) / `upsert`(398) / `delete_entity`(418) / `delete_entity_relation`(431) / `get_by_id`(446) / `get_by_ids`(458) / `delete`(470) / `get_vectors_by_ids`(486)。实例化时会 `_validate_embedding_func`(341)，并生成 collection 后缀 `_generate_collection_suffix`(357)。

代表实现：`NanoVectorDBStorage`（`kg/nano_vector_db_impl.py:62`，默认）、`PGVectorStorage`（`kg/postgres_impl.py:3997`）、`MilvusVectorDBStorage`、`QdrantVectorDBStorage`、`FaissVectorDBStorage`、`MongoVectorDBStorage`、`OpenSearchVectorDBStorage`、`NoopVectorDBStorage`（`kg/noop_vector_db_impl.py`，纯图入库场景，query 直接抛 `StorageCapabilityError`）。三个向量库分工：entities / relationships / chunks。

**(c) 图 —— `BaseGraphStorage`（`base.py:671`）**

抽象方法：`has_node`(717) / `has_edge`(728) / `node_degree`(740) / `edge_degree`(782) / `get_node`(794) / `get_edge`(805) / `get_node_edges`(819) / `upsert_node`(959) / `upsert_edge`(1055) / `delete_node`(1072) / `remove_nodes`(1085) / `remove_edges`(1098) / `get_all_labels`(1114) / `get_knowledge_graph`(1134) / `get_all_nodes`(1228) / `get_all_edges`(1237) / `get_popular_labels`(1257) / `search_labels`(1301)。

批量化是这个类的主要设计重点——它额外提供了 `get_nodes_batch`(868) / `node_degrees_batch`(882) / `edge_degrees_batch`(901) / `get_edges_batch`(925) / `get_nodes_edges_batch`(943) / `upsert_nodes_batch`(1007) / `has_nodes_batch`(1020) / `upsert_edges_batch`(1039)。local 检索正是靠 `asyncio.gather(get_nodes_batch, node_degrees_batch)` 把 N 次往返压成 1 次（`operate.py:6222-6226`）。

代表实现：`NetworkXStorage`（`kg/networkx_impl.py:38`，默认，GraphML 落盘）、`PGGraphStorage`（`kg/postgres_impl.py:7414`）、`PGTableGraphStorage`（`kg/pgtable_impl.py`）、`Neo4JStorage`、`MemgraphStorage`、`MongoGraphStorage`、`OpenSearchGraphStorage`。另有 `discard_uncommitted_mutations`(692) 用于回滚未提交的图变更。

**(d) 文档状态 —— `DocStatusStorage`（`base.py:1558`，继承 `BaseKVStorage`）**

抽象方法：`get_status_counts`、`get_docs_by_statuses`（含 `strict` 参数区分「尽力读」与「调度控制面必须完整否则抛错」两种契约）。状态枚举 `DocStatus`（`base.py:1317`）、记录结构 `DocProcessingStatus`（:1333）、分页游标 `CursorPosition`(1412) / `CursorAfter`(1443)、分页结果 `DocStatusPage`(1473)。

代表实现：`JsonDocStatusStorage`（`kg/json_doc_status_impl.py`）、`PGDocStatusStorage`（`kg/postgres_impl.py:5662`）、Mongo / Redis / OpenSearch。

**注册表机制**：`lightrag/kg/__init__.py` 用三张表把「名字 → 实现」解耦：`STORAGE_IMPLEMENTATIONS`（每类存储允许的 implementation 列表 + 必需方法）、`STORAGE_ENV_REQUIREMENTS`（每个实现所需的环境变量，缺则拒绝启动）、`STORAGES`（实现名 → 模块路径，用于惰性导入）。`verify_storage_implementation()` 在实例化前校验兼容性。

### 5. `insert` 主链路

```
insert (lightrag.py:2272)              # 同步外壳，内部 _run_sync 转异步
  └─ ainsert (2310)
       ├─ generate_track_id("insert")   # 生成 track_id 供 /documents 轮询
       ├─ resolve_chunk_options(...)     # 把 split_by_character 固化成 per-doc 快照
       ├─ apipeline_enqueue_documents (pipeline.py:740)
       │    1 校验 ids / 无则按内容或 file_path 生成 MD5
       │    2 生成初始 DocProcessingStatus
       │    3 过滤已处理文档
       │    4 写入 doc_status 并入队（含 process_options / chunk_options 原样落库）
       └─ apipeline_process_enqueue_documents (pipeline.py:1724)
            └─ _run_pipeline_batch (pipeline.py:2619)
                 _pipeline_feeder (2507) -> parse_queues[group]
                    Layer1 _parse_worker (4409)
                    Layer2 _analyze_worker (4805)
                    Layer3 _process_worker (4950)
                            └─ process_single_document (4991)
                                 分块 -> extract_entities -> merge_nodes_and_edges
                                 PROCESSING -> PROCESSED / FAILED
```

`apipeline_enqueue_documents`（`pipeline.py:740`）的 docstring 里有一个关键陷阱说明：**只要传了 `ids`，就被标记为 SDK 直插路径，文档永远按 RAW 入队**，不会走 parse worker 延迟解析；`docs_format="pending_parse"` 只在服务端上传路径生效。

`process_single_document`（`pipeline.py:4991`）内部还有几个阶段：`_purge_stale_extraction_if_resuming`（:5941，换 `process_options` 重跑时先清旧块与旧图）、`_mark_graph_mutation_started`（:6043）、`_upsert_doc_status_transition`（:6099）。取消检查点散布在 `_raise_if_cancelled`（:6082 附近）等位置。

### 6. `query` 主链路

```
query (lightrag.py:4819)
  └─ aquery (4843)  ── 兼容外壳，只取 llm_response
       └─ aquery_llm (5114)
            └─ (内部走 aquery_data 的检索部分)
  └─ aquery_data (4901)    # 只检索不生成，返回结构化 dict
       ├─ validate_query_not_empty / validate_rag_query
       ├─ 复制一份 QueryParam 并强制 only_need_context=True
       ├─ mode in {local,global,hybrid,mix} -> kg_query   (operate.py:4693)
       ├─ mode == naive                     -> naive_query(6827)
       ├─ mode == bypass                    -> convert_to_user_format([]...,"bypass")
       └─ _query_done (5300)   # flush 缓存类存储
```

`kg_query`（`operate.py:4693`）内部：抽取关键词 → `_build_query_context`（:6075）→ 拼 prompt → 调 `role_llm_funcs["query"]`。它返回的 `QueryResult`（`base.py:2085`）同时带 `content`（给 LLM 的文本）、`raw_data`（结构化实体/关系/块/引用）与 `response_iterator`（流式）。

`_build_query_context` 的四阶段（`operate.py:6075` 的 docstring 明写）：

1. **Search** — `_perform_kg_search`（:5280）
2. **Truncate** — `_apply_token_truncation`（:5501），按 entity/relation/total 三级预算裁剪
3. **Merge chunks** — `_merge_all_chunks`（:5765），含 `_attach_content_headings`（:5719）把父标题路径挂到块上
4. **Build context** — `_build_context_str`（:5873）

`_perform_kg_search` 有一个明确的优化：**按 mode 只嵌入需要的文本，并合并成一次批量调用**（:5317-5364）。`need_ll` / `need_hl` 两个布尔量决定哪些文本进 `texts_to_embed`，省掉 2-3 次串行 API 往返。

### 7. 并发控制

LightRAG 的并发是**多进程 + 多协程**双层，锁的实现集中在 `kg/shared_storage.py`。

**跨进程层**：

- `UnifiedLock`（`shared_storage.py:247`）：通用锁门面，内部在「多进程 multiprocessing.Lock」与「单进程 asyncio.Lock」之间切换（`get_data_init_lock` :1442 是典型用法）。
- `KeyedUnifiedLock`（:936）/ `get_storage_keyed_lock`（:1430）：**按 key 加锁**，用于实体级并发控制。
- `initialize_share_data`（:1524）：一次性初始化共享字典与锁表，未初始化就取锁会抛 `RuntimeError("Shared-Data is not initialized")`。

一个容易误解的点，源码在 `shared_storage.py:1424-1427` 有直接注释：

```
# Workspace based storage_lock is implemented by get_storage_keyed_lock instead.
# Workspace based pipeline_status_lock is implemented by get_storage_keyed_lock instead.
# No need to implement graph_db_lock:
#    data integrity is ensured by entity level keyed-lock and allowing only one process to hold pipeline at a time.
```

即：**没有全局图锁**。数据安全靠两件事保证——实体级 keyed lock，加上「同一时刻只有一个进程能持有 pipeline」。读代码时别去找 `graph_db_lock`，它已被移除。

**进程内层**：

- 插入并发上限：`asyncio.Semaphore(self.max_parallel_insert)`，在 `_run_pipeline_batch` 构造 `_BatchRunContext` 时创建（`pipeline.py:2670` 附近），`process_single_document` 用 `async with ctx.semaphore` 包住整篇处理（:5048）。
- 三级有界队列：`parse_queues[group]`（`maxsize=self.queue_size_parse`）、`q_analyze`（`queue_size_analyze`）、`q_process`（`queue_size_insert`）。有界意味着 feeder 会被背压，而不是无脑吃内存。
- 入队准入：`_reserve_ingress_slot`（`pipeline.py:638`）/ `_release_admission_reservation`（:726），配合 `api/admission.py` 与 `admission_middleware.py`。
- 解析分组并发：`_group_concurrency(group)`（`pipeline.py:2709` 附近），内置 group 用 `max_parallel_parse_{group}` 字段，第三方 group 用注册时声明的 `concurrency`。

**写入提交屏障**：

- `_index_storages`（`lightrag.py:3793`）列出会被一起 flush 的 12 类存储实例。
- `_flush_storages`（:4075）是可指定子集的 flush 屏障：先跑完所有 flush 再对第一个失败抛 `IndexFlushError`；返回显式 `False`（DECLINED，后端拒绝提交并丢弃内存变更）也计为失败；`llm_response_cache` 排在 `text_chunks` 之后，前者失败则跳过。
- `_discard_pending_index_ops`（:3814）在批次因内部错误中止时清空各后端的 `_pending_*` 缓冲，避免脏记录被后续文档带出。

**LLM 侧限流**：`priority_limit_async_func_call`（`lightrag.py:188` 导入），用于包装 rerank 与各角色 LLM（:1762、:1793）。角色定义见 `llm_roles.py`：

```python
ROLES: tuple[RoleSpec, ...] = (
    RoleSpec("extract", "EXTRACT", "extract LLM func"),
    RoleSpec("keyword", "KEYWORD", "keyword LLM func"),
    RoleSpec("query",   "QUERY",   "query LLM func"),
    RoleSpec("vlm",     "VLM",     "vlm LLM func"),
)   # lightrag/llm_roles.py:59-64
```

## 最小可运行入口

均摘自仓库 `README.md`（行号见标注）。

**(1) PyPI 装服务端（推荐）** —— `README.md:104-128`

```bash
uv tool install "lightrag-hku[api]"
# 或：python -m venv .venv && pip install "lightrag-hku[api]"

cp env.example .env      # 填 LLM / Embedding 配置
lightrag-server          # 默认绑定 0.0.0.0
```

安全提示（`README.md:122-127` 原文语义）：暴露到网络前必须在 `.env` 配 `LIGHTRAG_API_KEY`，或 `AUTH_ACCOUNTS` + `TOKEN_SECRET`，或只绑 `127.0.0.1`；否则所有端点公开。Ollama 兼容的 `/api/*` 路由默认放行。

**(2) 源码安装 + 前端构建** —— `README.md:130-166`

```bash
git clone https://github.com/HKUDS/LightRAG.git
cd LightRAG
make dev
source .venv/bin/activate          # Windows: .venv\Scripts\activate

cd lightrag_webui && bun install --frozen-lockfile && bun run build && cd ..
make env-base                      # 或 cp env.example .env
lightrag-server
```

**(3) Docker Compose** —— `README.md:168-176`

```bash
git clone https://github.com/HKUDS/LightRAG.git
cd LightRAG
cp env.example .env                # 修改 LLM 与 Embedding 设置
docker compose up
```

**(4) SDK 最小示例** —— `README.md:412-425` + `examples/lightrag_openai_demo.py`

```bash
cd LightRAG
export OPENAI_API_KEY="sk-...your_opeai_key..."
curl https://raw.githubusercontent.com/gusye1234/nano-graphrag/main/tests/mock_data.txt > ./book.txt
python examples/lightrag_openai_demo.py
```

对应代码骨架（`examples/lightrag_openai_demo.py:65-75`）：

```python
from lightrag import LightRAG, QueryParam
from lightrag.llm.openai import gpt_4o_mini_complete, openai_embed

rag = LightRAG(working_dir=WORKING_DIR,
               embedding_func=openai_embed,
               llm_model_func=gpt_4o_mini_complete)
await rag.initialize_storages()      # 会自动初始化 pipeline_status
...
await rag.ainsert(f.read())
await rag.aquery("What are the top themes in this story?", param=QueryParam(mode="naive"))
```

demo 依次演示 `naive` / `local` / `global` / `hybrid` 四种模式。流式响应示例见 `examples/lightrag_openai_compatible_demo.py`。

## 值得抄的工程手法

**1. 抽象基类 + 声明式注册表，把「换后端」变成改字符串**
`lightrag/kg/__init__.py` 的 `STORAGE_IMPLEMENTATIONS` 声明每类存储允许的后端与必需方法，`STORAGE_ENV_REQUIREMENTS` 声明每个后端需要的环境变量（缺则在启动期失败而不是运行期炸），`STORAGES` 提供惰性导入路径，`verify_storage_implementation()` 在实例化前校验。四个存储层（KV/向量/图/文档状态）全部走这套。好处：新增后端不用碰 pipeline 与 operate 任何一行；配置错误前移。

**2. 写前日志（write-ahead anchor）：先落「可发现性」，再做变更**
`merge_nodes_and_edges`（`operate.py:3514`）的 Phase 0 先把候选实体/关系全集写进 `full_entities` / `full_relations` 并 flush，之后才动图。崩溃时 purge / retry 总能从锚点发现该文档贡献过什么。配套 `on_anchors_durable` 回调把「已可能碰图」这一状态变更收窄到唯一窗口。这是分布式写入里标准的 WAL 思路，值得直接搬。

**3. 有界队列 + 信号量做背压，而不是无脑并发**
`_run_pipeline_batch`（`pipeline.py:2619`）为每个 parser queue_group 建一条 `asyncio.Queue(maxsize=...)`，分析层与入库层同理；并发度由 `asyncio.Semaphore(max_parallel_insert)` 统一收口。长文本入库不会把内存打爆，也不会把 LLM 提供商打挂。

**4. 实体级键控锁替代全局锁**
`get_storage_keyed_lock`（`kg/shared_storage.py:1430`）按实体 key 加锁，源码注释（:1424-1427）明确写了「不需要 graph_db_lock」。同一篇文档里的不同实体可以并行合并，只有同名实体串行。对「图写入」这种天然热点的数据结构，这是比全局锁高一个量级的做法。

**5. 按分支预聚合 embedding，消除串行往返**
`_perform_kg_search`（`operate.py:5326-5347`）先用 `need_ll` / `need_hl` 两个布尔量算出「当前 mode 真正需要哪些文本」，一次性 `await actual_embedding_func(texts_to_embed)` 拿到全部向量后再分发。省掉 2-3 次串行 API 往返，且失败只降级为 warning 不中断。

补充一条等价思路：角色化 LLM（`llm_roles.py` 的 `ROLES` 元组声明 `extract`/`keyword`/`query`/`vlm`，`RoleSpec` 携带 `env_prefix` 与 `queue_name`）同样用「注册表驱动」消除硬编码，让抽取用便宜模型、问答用强模型。

## 阅读路径

按顺序读这 9 个文件，每一步都为下一步提供上下文：

1. `lightrag/base.py:90` `QueryParam` — 先看清检索模式的契约，后面所有分支都由 `mode` 驱动。从这开始，不要从 `lightrag.py` 开始。
2. `lightrag/base.py:193-671` `StorageNameSpace` → `BaseKVStorage` / `BaseVectorStorage` / `BaseGraphStorage` — 三个抽象的方法签名就是整个数据流的骨架。
3. `lightrag/base.py:1317-1600` `DocStatus` / `DocProcessingStatus` / `DocStatusStorage` — 理解文档状态机，才知道流水线在追踪什么。
4. `lightrag/lightrag.py:684-1363` `LightRAG` 类字段 — 一次性看到所有可配置项（四个 storage 名、workspace、token 预算、抽取参数）。只读字段定义，跳过方法。
5. `lightrag/lightrag.py:2272-2400` `insert` / `ainsert` — 确认 SDK 入口只是入队 + 处理两步的薄封装。
6. `lightrag/operate.py:3942` `extract_entities` — 离线抽取的正主：gleaning、双解析路径、缓存。配合 `:710` / `:774` / `:907` 看单条记录怎么落库。
7. `lightrag/operate.py:3514` `merge_nodes_and_edges` — 接着看 Phase 0/1/2，再跳到 `:2429` `_merge_nodes_then_upsert` 与 `:2782` `_merge_edges_then_upsert` 看实际合并逻辑。
8. `lightrag/operate.py:4693` `kg_query` → `:6075` `_build_query_context` → `:5280` `_perform_kg_search` → `:6200` `_get_node_data` / `:6529` `_get_edge_data` — 在线链路一口气读完，四种模式的差异在这一步全部闭合。
9. `lightrag/pipeline.py:2619` `_run_pipeline_batch` → `:4991` `process_single_document` — 最后补编排层，看三级队列怎么把前两步串起来。

想深入某一层时的补充入口：存储后端看 `kg/nano_vector_db_impl.py` 与 `kg/networkx_impl.py`（最短，适合当模板）；LLM 适配看 `llm/openai.py`；服务端看 `api/lightrag_server.py` + `api/routers/query_routes.py`。

## 别踩的坑

**体积坑（最重要）**
`lightrag.py` 8431 行、`pipeline.py` 8228 行、`utils.py` 8293 行、`operate.py` 7126 行、`kg/postgres_impl.py` 10177 行。这些是「模块级巨型文件」，**不要顺序通读**。一律先 `grep -n "^class \|^    async def \|^def "` 拿目录再跳读。`utils.py`（366 KB）里混杂了分词、日志、文件锁、导出 Excel 等大量与主线无关的工具，别把它当入口。

**SDK `ainsert` 只能走固定 token 分块**
`ainsert`（`lightrag.py:2310`）的 docstring 明写：它**总是**用固定 token（F）策略分块，`split_by_character` 只是 F 策略的运行时参数。F / R（递归字符）/ V（语义向量）/ P（段落语义）/ C 只能通过 `apipeline_enqueue_documents(process_options=...)` + `apipeline_process_enqueue_documents()` 选择——服务端 REST 就走这条路。想用 R/V/P/C 就别调 `ainsert`。

**默认检索模式是 `mix` 不是 `hybrid`**
`QueryParam.mode` 默认 `"mix"`（`base.py:93`）。很多博客与旧文档写的是 `hybrid` 或 `global`，直接按默认值推演行为会错。

**切换 embedding 模型 ≠ 改配置就完事**
向量由模型写入，换模型后旧向量不可用。README Note 1（`README.md:429`）的提醒：demo 数据可以直接删 `./dickens` 重跑；**真实部署绝对不要删工作目录**（里面是知识图和文本块，删了等于全量重灌）。正确做法是跑 `lightrag-rebuild-vdb`（见 `docs/ProgramingWithCore.md#switching-embedding-models`，实现在 `lightrag/tools/rebuild_vdb.py`）。

**storage 实现之间存在能力差异，不是「换个类名」就等价**
- `NoopVectorDBStorage`（`kg/noop_vector_db_impl.py`）的 `query` 直接抛 `StorageCapabilityError`，只适用于「只要图、不要检索」的入库流程。
- `DocStatusStorage.get_docs_by_statuses(strict=True)` 是**调度控制面契约**：结果必须完整否则抛错；`strict=False` 才是 UI 列表用的尽力读。自研后端必须区分这两种语义，否则会出现「文档悄悄漏处理」。
- `BaseKVStorage.get_by_id_strict`（`base.py:556`）说明「严格点读」是**可选**能力，不是所有后端都实现。

**`kg/__init__.py` 里存在指向已移动文件的映射**
`STORAGES` 中仍有 `"ChromaVectorDBStorage": ".kg.chroma_impl"` 与 `"AGEStorage": ".kg.age_impl"`，但 `lightrag/kg/` 下已无这两个文件——`chroma_impl.py` 被移到了 `kg/deprecated/chroma_impl.py`，`age_impl.py` 在快照中不存在。同时 `STORAGE_IMPLEMENTATIONS` 的 VECTOR 列表里 `"ChromaVectorDBStorage"` 处于注释状态，GRAPH 列表里也没有 `AGEStorage`。即：**Chroma 与 AGE 后端在当前快照下不应被视为可用**。选后端时用 `STORAGE_IMPLEMENTATIONS` 里的「未注释」列表作为准绳，而不是 `STORAGES`。

**`insert_custom_chunks` / `ainsert_custom_chunks` 已标记 deprecated**
源码注释（`lightrag.py:2378`、`2388` 附近）为 `# TODO: deprecated, use insert instead`。新代码不要走这条路径。

**可选依赖、成本与前端产物**
- 核心 `dependencies` 很轻（无 openai/fastapi），服务端需要 `[api]` extra；向量库 / 图数据库 / 离线栈另有 `requirements-offline-*.txt` 与 `[test]` / `[offline]` extras。`make dev` 会装测试 + 完整离线栈并构建前端。
- `entity_extract_max_gleaning` 默认 1，意味着每个 chunk 至少 2 次 LLM 调用。批量入库前先算账，必要时调成 0。
- 前端产物需单独构建：`lightrag_webui/` 用 Bun（`bun install --frozen-lockfile` + `bun run build`），PyPI 安装的服务端不含它。参见 `docs/FrontendBuildGuide.md`。
