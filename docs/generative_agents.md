# Generative Agents（Smallville）技术解析文档

> 面向有 5 年前端 + Node/PHP 经验、正在转 AI Agent 工程方向的工程师。
> 本文所有路径、类名、行号、公式、默认值均来自本地快照 `github-refs/generative_agents`（commit `fe05a71`）的真实读取，未使用外部记忆。
> 行号会漂移，因此每条结论都同时标注了附近的函数名作为锚点。
> 两个流传较广的事实错误写在「事实卡片」和「目录地图」里，请务必看。

---

## 一句话定位

**它解决什么问题**：论文标题 "Interactive Simulacra of Human Behavior" 的关键词是 **believable**——不是正确、不是高效，而是**可信**。为此它提出一套完整的认知架构：**观察 → 记忆（三项加权检索）→ 反思（更高层抽象）→ 计划（长程到短程分解）→ 行动 → 再观察**，所有状态落盘、可回放、可干预。

注意它**不是**「用 LLM 做多 Agent 协作完成任务」的框架（那是 ChatDev / MetaGPT 那条线），也不是「带工具的自主循环」（那是 ReAct 那条线）。它的目标是**模拟**：25 个 agent 在 Smallville 小镇里起床、做饭、上班、偶遇、聊天、传播情人节派对的消息。评估指标是「行为是否像一个可信的人」，不是「任务是否做对」。

**在 Agent 技术栈的哪一层**：处在 **认知架构层（cognitive architecture）**，三大 contribution：

1. **记忆流（Memory Stream）** —— 长期记忆的完整记录 + 合成检索。
2. **反思（Reflection）** —— 把底层观察定期合成为更高层的 thought，这是「从经历里提炼出对一个人的看法」的机制。
3. **计划与反应（Planning & Reacting）** —— 递归分解的日计划，以及对突发感知事件的应对。

外加一层把它们接起来的循环执行顺序。现代 Agent 框架里的 memory / reflection / planning 三个概念，很大一部分是从这个仓库扩散出去的。

**历史位置**：这是 **2023 年上半年的作品，完全站在 LangChain / LangGraph / OpenAI Agents SDK 之前**。几个非常具体的时代痕迹：**没有用任何 LLM 框架**，全部调用是裸 `openai` SDK 的手写 wrapper，集中在 `persona/prompt_template/gpt_structure.py`；**Prompt 全是磁盘上的 .txt 文件**，110 个模板躺在 `prompt_template/v1/`、`v2/`、`v3_ChatGPT/`，运行时由 `generate_prompt()`（`gpt_structure.py:227`）读文件、用 `!<INPUT 0>!` 占位符做字符串替换（今天我们会写成 f-string 或 `ChatPromptTemplate`）；**术语是旧名**，作者在 `reverie.py:12-19` 的 release note 里自己交代了 *"I use the term 'personas' to refer to generative agents, 'associative memory' to refer to the memory stream..."*，所以代码里 **`Persona` = 论文的 GenerativeAgent**、**`AssociativeMemory` = 论文的 Memory Stream**，这不是歧义而是历史地层；**架构和你熟的东西相反**——没有 async/await、没有向量数据库、没有 tool schema、没有 structured output（JSON 是用 `rfind('}')` 手工切出来的，见 `gpt_structure.py:105-107`），有的是一条非常清晰的**认知循环**加一套非常朴素的**文件系统状态存储**。

**一句话结论**：把它读成「**LLM 认知架构的参考实现**」，而不是拿来用的框架。价值全在 `persona/cognitive_modules/` 那 5 个文件的算法细节里，尤其「记忆检索三项打分」那 20 行——那是整个领域被引用最多、但真正读到实现的人最少的一段。

---

## 事实卡片

| 项 | 值 |
|---|---|
| 主语言 | Python（README 声明**测试于 Python 3.9.12**） |
| 研究代码行数 | **11,174 行** Python（`find reverie -name "*.py"`，21 个文件） |
| 全仓行数 | **12,292 行** Python（多出的 1,118 行是 `environment/` 的 Django 服务） |
| 研究代码文件数 | `reverie/` 共 **131 个**：**21 个 `.py` + 110 个 `.txt` prompt 模板** |
| 仓库总文件数 | **210,029**（`git ls-files \| wc -l`）——见下方真相对比 |
| 主要依赖 | `openai==0.27.0`（**致命**）、`gensim==3.8.0`、`numpy==1.25.2`、`Django==2.2`、`matplotlib==3.7.2`、`scikit-learn==1.3.0`、`selenium==4.8.2`、`Pillow==8.4.0` |
| 许可证 | **Apache License 2.0** |
| 快照 commit | **`fe05a71`**（`fe05a71d3e4ed7d10bf68aa4eda6dd995ec070f4`），作者 Joon Sung Park，**`2023-08-11 00:07:21 -0700`**，message `Merge pull request #24 from eltociear/main-1` |
| star 数 | 【待核】（离线快照，未联网查询） |

### 仓库文件数真相对比（纠正一个流传的说法）

常见说法是「21 万个文件里 20.9 万是 `static_dirs/` 下的网页素材」。**这个说法是错的**，我逐目录实测了：

| 区域 | 文件数 | 占比 | 是什么 |
|---|---|---|---|
| `environment/frontend_server/storage/` | **208,432** | **99.24%** | **真正的 bulk**：预先跑好的回放存档（每步一个 `environment/<step>.json` + `movement/<step>.json`） |
| `environment/frontend_server/static_dirs/` | **1,403** | 0.67% | Smallville 瓦片素材（碰撞/图层 CSV、精灵图）——**不到 bulk 的 1%** |
| `reverie/` | **131** | 0.06% | **真正的研究代码，全在这** |
| `compressed_storage/` + `templates/` + `translator/` | 40 | — | 演示压缩包、Django HTML、Django app |
| 其余（根目录 + Django 工程散件） | 63 | — | `README.md`、`LICENSE`、`manage.py`、`requirements.txt` 等 |

**结论**：该读的是 **131 个**，该跳过的是 **约 209,898 个**。跳过时**不能只排除 `static_dirs/`**，必须排除 `storage/`。用 Glob 时用 `reverie/**`，不要对仓库根做递归。

---

## 目录地图

顶层：

| 路径 | 一句话 |
|---|---|
| `reverie/` | **唯一的研究代码所在**：模拟核心 + 交互式 CLI 入口 |
| `environment/` | **可视化 / 回放端**：Django 服务 + 瓦片素材 + 20 万回放存档。除 `manage.py` 和 `templates/` 外基本可跳过 |
| `requirements.txt` | Python 3.9.12 时代的全套依赖，含致命的 `openai==0.27.0` |
| `README.md` | 12,659 字节，写得意外地完整。**跑起来的步骤全在这，而且是唯一权威来源** |

### `reverie/` 内部

| 路径 | 一句话 |
|---|---|
| `backend_server/reverie.py`（664 行） | **模拟侧主入口**。`ReverieServer`：`start_server()` 是主循环，`open_server()` 是交互式 CLI |
| `backend_server/maze.py`（416 行） | **空间环境**。`Maze`：把 Tiled 导出的 CSV 矩阵 + 色块表组装成 `tiles[row][col]` 字典矩阵 |
| `backend_server/path_finder.py`（269 行） | A*/DFS 寻路。文件头注释写着 *"Some of the functions are defunct."*，只有 `path_finder()` 真被调用 |
| `backend_server/persona/persona.py`（272 行） | **一个 agent 大脑的入口**。`Persona` 把 5 个认知模块串成 `move()` 一条链 |
| `backend_server/persona/memory_structures/` | 三类记忆：`associative_memory.py`（长期记忆流）、`scratch.py`（短期工作记忆 + 全部超参）、`spatial_memory.py`（空间记忆树，最短，123 行） |
| `backend_server/persona/cognitive_modules/` | **五个认知模块 = 论文核心**：`perceive.py` → `retrieve.py` → `plan.py` → `reflect.py` → `execute.py`，外加 `converse.py` |
| `backend_server/persona/prompt_template/run_gpt_prompt.py`（2,930 行） | **Prompt 分发层**。32 个 `run_gpt_prompt_*` 函数，各自读模板、填占位符、调 GPT、清洗结果 |
| `backend_server/persona/prompt_template/gpt_structure.py`（331 行） | **唯一的 LLM 调用出口**。4 个 request 函数 + `get_embedding()` |
| `backend_server/persona/prompt_template/v1/ v2/ v3_ChatGPT/` | 110 个 prompt .txt。**生效的是 `v3_ChatGPT/`（48 个）**，`v1`/`v2` 是 GPT-3 时代遗物 |
| `backend_server/persona/prompt_template/defunct_run_gpt_prompt.py`（2,179 行） | 文件名写着 defunct，**不要读** |
| `backend_server/global_methods.py`、`reverie/global_methods.py` | 两份几乎相同的工具库，同名两份源于 `sys.path.append('../../')` 这类 hack |

### 另一个必须纠正的事实

任务描述里提到的 `reverie/frontend_server/` **在这个快照里根本不存在**（已 `ls` 确认）。可视化服务实际在 **`environment/frontend_server/`**（Django 工程，`manage.py` 在那）。分工仍是「模拟 vs 可视化」两端，但目录名和常见描述不一致。

### 分工：后端 `reverie/` 与前端 `environment/`

这两端**通过文件系统通信**，没有任何 socket/RPC。这是全仓库最反直觉也最值得理解的一点：

`reverie/` 与 `environment/` 之间没有任何 socket/RPC，只有三个约定文件：

```
写 temp_storage/curr_sim_code.json + curr_step.json   (对齐模拟代号与 step)        ---->
读 storage/<sim>/environment/<step>.json  (前端挪完角色后写入的位置)               <----
写 storage/<sim>/movement/<step>.json  ({persona:{name:{movement,pronunciatio,description,chat}}, meta:{curr_time}})  ---->
```

核心含义：

- **前端既是消费者也是生产者**。每次渲染把 agent 挪到新位置就写一个 `environment/<step>.json`；后端**轮询**它（`reverie.py:316` 的 `check_if_file_exists`）获知世界状态。同步机制本质是一个 **polled file queue**。
- **后端会被前端阻塞**。文件不出现就 `time.sleep(self.server_sleep)`（默认 0.1 秒，`reverie.py:138`）。这就是 README 强调「**必须保持浏览器 tab 打开**」的原因。
- Django 侧只有 `translator` 一个有实质逻辑的 app，其余是静态页面。

---

## 核心架构

```
┌──────────────────────────────────────────────────────────────────────────┐
│ 交互层  ReverieServer.open_server()                reverie.py:415         │
│         "run <n>" / "save" / "fin" / "print persona associative ..."      │
└──────────────────────────────┬───────────────────────────────────────────┘
                               │ run <n>
                               v
┌──────────────────────────────────────────────────────────────────────────┐
│ 驱动层  ReverieServer.start_server(int_counter)    reverie.py:279         │
│   while int_counter > 0:                                                  │
│     1. poll <sim>/environment/<step>.json，不存在 -> sleep(0.1)           │
│     2. 更新 maze tile 上的 events (remove_subject_events / add_event)     │
│     3. for each persona: persona.move(maze, personas, tile, curr_time)    │
│     4. 写 <sim>/movement/<step>.json                                      │
│     5. step += 1;  curr_time += sec_per_step (默认 10 秒)                 │
└──────────────────────────────┬───────────────────────────────────────────┘
                               │ persona.move()        persona.py:185
                               v
┌──────────────────────────────────────────────────────────────────────────┐
│ Agent 层  Persona（= 论文中的 GenerativeAgent）                            │
│   1. new_day 判定 (False / "First day" / "New day")            :211       │
│   2. perceived = self.perceive(maze)        -> perceive.py:25            │
│   3. retrieved = self.retrieve(perceived)   -> retrieve.py:16            │
│   4. plan      = self.plan(...)             -> plan.py:931               │
│   5. self.reflect()                         -> reflect.py:172            │
│   6. return self.execute(...)               -> execute.py:15             │
└───┬────────────┬────────────┬─────────────┬─────────────┬────────────────┘
    v            v            v             v             v
 PERCEIVE     RETRIEVE      PLAN         REFLECT       EXECUTE   (细节见下节)
 (197 行)    (284 行)     (1053 行)     (271 行)      (174 行)
    │            │            │             │             │
    v            v            v             v             v
    v            v            v             v             v
┌──────────────────────────────────────────────────────────────────────────┐
│ 记忆层  persona/memory_structures/                                        │
│  s_mem  MemoryTree        spatial_memory.py:15  {world:{sector:{arena:    │
│                                                  [game_object]}}}        │
│  a_mem  AssociativeMemory associative_memory.py:50   = 论文 Memory Stream │
│         id_to_node {node_N: ConceptNode}                                  │
│         seq_event/seq_thought/seq_chat      (新 -> 旧，头插)              │
│         kw_to_event/kw_to_thought/kw_to_chat  倒排索引                     │
│         kw_strength_event/kw_strength_thought 词频                         │
│         embeddings {embedding_key: vector}                                │
│  scratch Scratch         scratch.py:14  当前动作 / 今日 schedule /         │
│                          反思计数器 / 对话状态 / 路径 + 全部超参          │
└──────────────────────────────────────────────────────────────────────────┘
                               │ 所有 LLM 调用
                               v
┌──────────────────────────────────────────────────────────────────────────┐
│ Prompt / LLM 层                                                           │
│  run_gpt_prompt.py (2930 行, 32 个 run_gpt_prompt_* 函数)                 │
│    读 v3_ChatGPT/*.txt -> 填 !<INPUT n>! -> 调用 -> validate -> clean_up   │
│  gpt_structure.py (331 行)  ChatGPT_request / GPT4_request /              │
│    ChatGPT_safe_generate_response / get_embedding(ada-002)                │
├──────────────────────────────────────────────────────────────────────────┤
│ 世界模型  Maze  maze.py:19                                                │
│  tiles[row][col] = {world, sector, arena, game_object, spawning_location, │
│                     collision, events: set()}     访问约定 tiles[y][x]     │
│  address_tiles {地址串: {(x,y), ...}}     collision_maze -> path_finder    │
├──────────────────────────────────────────────────────────────────────────┤
│ 持久化  storage/<sim_code>/                                               │
│  reverie/meta.json    environment/<step>.json    movement/<step>.json     │
│  personas/<Name>/bootstrap_memory/{scratch, spatial_memory}.json           │
│                       associative_memory/{nodes, embeddings, kw_strength} │
└──────────────────────────────────────────────────────────────────────────┘
```

---

## 关键抽象与执行流

### 1. 记忆流 = `AssociativeMemory`

论文里的 Memory Stream 在这里叫 `AssociativeMemory`（`associative_memory.py:50`）。它本质是**一个全量节点字典 + 一组倒排索引**，没有任何 ANN / 向量数据库。

**存储结构**（`__init__`，`:51`）：`id_to_node`（主存储）；`seq_event` / `seq_thought` / `seq_chat`（三条按时序、按类型分开的列表）；`kw_to_event` / `kw_to_thought` / `kw_to_chat`（倒排表）；`kw_strength_event` / `kw_strength_thought`（词频）；`embeddings`（向量池）。

**插入方向很关键**：`add_event` 用 `self.seq_event[0:0] = [node]`（`:177`）在**头部插入**，即 **`seq_event[0]` 最新、`seq_event[-1]` 最老**。这解释了后面 `retrieve.py` 里那些下标的方向。

**`ConceptNode`**（`:19`）：每条记忆 = **(s, p, o) 主谓宾 + `description` + `embedding_key` + `poignancy`（重要度）+ `keywords: set` + `filling`**。三种 `type`：`"event"`（观察到的）、`"thought"`（反思生成的）、`"chat"`（对话）。

**`depth` 字段**：event 恒为 0（`:161`）；thought 初始 1，若带 `filling` 则 `depth += max(filling 各节点的 depth)`（`:209-210`）。**这就是论文的反思树——每层反思比原料深一层。** 但 `generate_insights_and_evidence()`（`reflect.py:38`）只调用一次，且 `filling` 指向的 event/chat 都是 depth 0，所以**实际跑起来反思是单层的，depth 恒为 1**。

`save()`（`:112`）把 `id_to_node` 逆序（最新优先）写成 `nodes.json`，另写 `kw_strength.json` 和 `embeddings.json`；`__init__` 对应三处 `json.load`（`:65`、`:67`、`:105`）。

**两种检索路径**：① **关键词精确匹配**（plan 阶段用）——`retrieve_relevant_thoughts`（`:305`）和 `retrieve_relevant_events`（`:317`），把 s/p/o 各查一次倒排表后合并。**注意一个真实瑕疵**：thought 版用 `i.lower()`（`:311`），event 版用原始 `i`（`:323`），而建表时统一 lower 了（`:178`、`:222`），所以 **event 检索命中率偏低**。② **加权合成检索**（reflection 用）——下一节。

### 2. 三项打分（recency / importance / relevance）—— 本文最重要的一段

**位置：`reverie/backend_server/persona/cognitive_modules/retrieve.py:244-249`，函数名 `new_retrieve`。**

```python
gw = [0.5, 3, 2]
master_out = dict()
for key in recency_out.keys():
  master_out[key] = (persona.scratch.recency_w*recency_out[key]*gw[0]
                 + persona.scratch.relevance_w*relevance_out[key]*gw[1]
                 + persona.scratch.importance_w*importance_out[key]*gw[2])
```

**这里有两层权重，必须区分清楚**：

- **代码写死的经验增益 `gw = [0.5, 3, 2]`**（`:244`），顺序是 **[recency, relevance, importance]**。
- **每个 persona 可调的 `scratch.recency_w` / `relevance_w` / `importance_w`**（`scratch.py:57-59`，**默认全为 1**）。

所以实际生效比例是 `0.5 : 3 : 2` —— **relevance 最高（3），importance 次之（2），recency 被压到 0.5**。只看三个 `*_w` 默认值会得出「三者等权」的错误直觉，这是最容易看错的地方。

代码上方留有作者的注释（`:239-243`）：

```
# Note to self: test out different weights. [1, 1, 1] tends to work
# decently, but in the future, these weights should likely be learned,
# perhaps through an RL-like process.
# gw = [1, 1, 1]
# gw = [1, 2, 1]
gw = [0.5, 3, 2]
```

**这是全仓库最有价值的一条注释**——它直接告诉你这组数字是试出来的，作者自己也认为应该可学习。

**三项分怎么算**：

| 项 | 位置 | 实现 |
|---|---|---|
| **recency** | `extract_recency()` `:132` | `recency_vals = [decay ** i for i in range(1, len(nodes)+1)]`（`:145`）。`i` 从 1 开始，最新节点得 `decay**1` 而非 `decay**0`。**默认 `recency_decay = 0.99`**（`scratch.py:60`），即 100 条前的记忆权重约 0.37 |
| **importance** | `extract_importance()` `:155` | 直接取 `node.poignancy`（`:170`）。该分数由 LLM 在 perceive 时打。**特判**：`if "is idle" in description: return 1`（`perceive.py:16-17`、`reflect.py:76-77`） |
| **relevance** | `extract_relevance()` `:175` | `get_embedding(focal_pt)`（`:189`）后对每个节点算 `cos_sim`（`:194`，实现在 `:49`） |

**归一化**：三项各自过 `normalize_dict_floats(d, 0, 1)`（`:70`）做 min-max 到 [0,1]，再加权求和（`:232`、`:234`、`:236`）。**注意特殊分支**：`range_val == 0`（所有值相同）时每个值被设为 `(target_max-target_min)/2`（`:99`），即全等于 0.5 而非 0——避免「只有一个候选时 relevance 归一化成 0」的退化。

**取 top-N**：`top_highest_x_values(d, x)`（`:107`）就是 `sorted(reverse=True)[:x]`。`:251` 有一次取全部的调用（纯打印调试）；真正截断在 `:262`，**默认 `n_count=30`**（签名 `:199`）。

**副作用**：被选中的节点会更新 `last_accessed = curr_time`（`:266-267`）。而 `new_retrieve` 内排序（`:224-228`）和 `generate_focal_points`（`reflect.py:24-29`）都以 `last_accessed` 升序——**刚被检索过的记忆下次排到时间轴末尾，检索历史会反馈影响下一次检索**。这是论文没强调的隐含行为。

**另一条路径**：`retrieve(persona, perceived)`（`retrieve.py:16`）是给 CONVERSATION 用的**非加权**版，只做关键词倒排（`:38-44`）。**一句话区分：`retrieve` 管「要不要搭话」（关键词精确），`new_retrieve` 管「回忆与反思」（三项加权）。**

### 3. 反思（Reflection）

**触发条件 —— `reflection_trigger(persona)`，`reflect.py:135`，判断式在 `:152-153`**：

```python
if (persona.scratch.importance_trigger_curr <= 0
    and [] != persona.a_mem.seq_event + persona.a_mem.seq_thought):
  return True
return False
```

- 计数器初值 `= importance_trigger_max`（`scratch.py:62`），**该值为 150**（`:61`）。
- **每次 perceive 到新事件就扣**：`perceive.py:178` 的 `importance_trigger_curr -= event_poignancy`，同时 `importance_ele_n += 1`（`:179`）。poignancy 通常 1—10，所以约 **15—150 个新事件后才反思一次**。
- **重置**：`reset_reflection_counter()`（`reflect.py:158`）恢复 max 并把 `importance_ele_n` 归零（`:167-169`）。

**流程 —— `run_reflect(persona)`，`reflect.py:99`**，四步：

1. `generate_focal_points(persona, 3)`（`:21`）—— 取最近 `importance_ele_n` 条记忆（`:32`，`nodes[-1*importance_ele_n:]`）拼成 statements，让 LLM 提 **3 个焦点问题**。注意这不是闲聊话题，而是「这批记忆里什么是重要的」。
2. `new_retrieve(persona, focal_points)`（`:113`）—— 每个焦点跑三项加权检索，默认 top-30。
3. `generate_insights_and_evidence(persona, nodes, 5)`（`:38`）—— **每个焦点最多 5 条 insight**，每条附带 LLM 指认的证据下标（`:50-53`）。**异常处理很粗糙**（`:54-55`）：解析失败直接 `return {"this is blank": "node_1"}`。
4. 每条 insight 写入前（`:122-132`）：设 `expiration = curr_time + timedelta(days=30)`（`:124`，**反思 thought 有 30 天过期，而 event 的 `expiration=None` 永不过期**）→ `generate_action_event_triple` 取 (s,p,o)（`:125`）→ `keywords = set([s,p,o])`（`:126`）→ 再次 `generate_poig_score` 给反思自身打分（`:127`）→ `a_mem.add_thought(...)`（`:130`）把 evidence 当 `filling` 传入。

**对话后的反思** —— `reflect()` 尾部（`:190-244`）在 `curr_time + timedelta(0,10) == chatting_end_time`（`:192`）时额外生成两条 thought：`planning_thought`（前缀 `"For {name}'s planning: "`，`:216-217`）和 `memo_thought`（前缀 `"{name} "`，`:232-233`），都以同一条 dialogue 为 filling。**这个用 `==` 做的精确时间差判断非常脆弱**，步长或时序稍偏就不会执行。

**实际入口 `reflect()`（`:172`）只有三行**：`if reflection_trigger(persona): run_reflect(persona); reset_reflection_counter(persona)`。

### 4. 计划（Planning）

`plan.py`（1053 行，全仓最大的认知模块）。**入口 `plan(persona, maze, personas, new_day, retrieved)`，`plan.py:931`**，三部分：

- **PART 1**（`:953-955`）—— `if new_day: _long_term_planning(persona, new_day)`
- **PART 2**（`:958-959`）—— `if persona.scratch.act_check_finished(): _determine_action(persona, maze)`
- **PART 3**（`:970-997`）—— 若 retrieve 到东西：`_choose_retrieved()` → `_should_react()` → `_chat_react()` / `_wait_react()`；最后对 `chatting_with_buffer` 递减（`:1002-1005`，**防止两个 agent 无限对话**）
- 返回值 `persona.scratch.act_address`（`:1007`），形如 `{world}:{sector}:{arena}:{game_object}`。

**长程计划 `_long_term_planning()`，`:461`** —— 三级递归：`generate_wake_up_hour()`（`:23`）得到一个整数小时 → 按 new_day 分岔（`"First day"` 走 `generate_first_daily_plan()` `:41` 产出粗粒度 daily_req；`"New day"` 走 `revise_identity()` `:408`，**先 `new_retrieve` 两个 focal point**（`:411-413`）再连发三个 prompt 更新 `scratch.currently`（`:426-448`），**这就是「每天醒来先复盘昨天」的实现**）→ `generate_hourly_schedule()`（`:71`）**逐小时问 LLM**。后者有个有意思的 `diversity_repeat_count = 3` 循环（`:98-109`）：重复采样直到产出 ≥5 种不同活动，避免 LLM 把一天填成同一件事；最后压缩成 `[['sleeping', 6], ...]`（`:119-129`）再乘 60 转分钟（`:134-136`）。**未完成点**：`:489` 保留着 `# We need to create a new daily_req here... TODO`。

**短程分解 `_determine_action()`，`:521`**：

- 内层 `determine_decomp()`（`:533`）决定值不值得分解：含 "sleeping" / "asleep" / "in bed" 的**不分解**（`:547-548`）；含 sleep/bed 但时长 > 60 分钟也不分解（`:549-551`）。
- 分解本身是 `generate_task_decomp()`（`:141`），把 60 分钟拆成 `[['going to the bathroom', 5], ['getting dressed', 5], ...]`。
- **然后决定地址**，一串 6 级 LLM 调用（`:167-276`）：`generate_action_sector`（`:167`）→ `generate_action_arena`（`:185`）→ `generate_action_game_object`（`:203`）→ `generate_action_pronunciatio`（`:226`，**生成 emoji**）→ `generate_action_event_triple`（`:252`）→ `generate_act_obj_desc`（`:267`）+ `generate_act_obj_event_triple`（`:272`）。**每层输入是上层输出。**

**反应决策**：`_choose_retrieved()`（`:655`）三级优先级 —— 删掉主语是自己的（`:674-677`）、优先选主语是另一个真人的（`:681-685`，用 `":" not in subject` 判断，因为物体地址带冒号）、平局用 `random.choice`、跳过 "is idle"（`:690-695`）。`_should_react()`（`:699`）内层 `lets_talk()`（`:715`）/ `lets_react()`（`:746`）各有一串**硬编码否决条件**：任一方 sleeping、当前 23 点、对方 waiting、任一方已在 chatting、冷却计数 > 0（`:736-738`）；**大量社会常识被写成 if 而不是 prompt** —— 这是很值得注意的取舍。三种反应模式在 `:980-989` 分派：`"chat with {name}"` → `_chat_react()`（`:860`），`"wait"` → `_wait_react()`（`:907`），第三种 `"do other things"` 被注释掉了。

### 5. `reverie.py` 主循环

- `ReverieServer.__init__(fork_sim_code, sim_code)`，`:43`。**所有模拟都必须 fork 自已有模拟**——`copyanything(fork_folder, sim_folder)`（`:58`）再改 meta.json（`:63-65`）。没有「从零创建」，只有「复制」。
- 读 `meta.json` 拿 `start_date` / `curr_time` / `sec_per_step` / `maze_name` / `persona_names` / `step`（`:60-94`）。以 `base_the_ville_isabella_maria_klaus` 为例：`sec_per_step: 10`、`maze_name: "the_ville"`、`start_date: "February 13, 2023"` —— **一步 = 游戏内 10 秒**，与 README 的 "One game step represents 10 seconds" 吻合。
- `Maze(maze_name)`（`:89`）在此实例化；随后对每个 persona（`:124-133`）从 `environment/<step>.json` 读初始 x/y，`new Persona(name, folder)` 并把当前事件写进对应 tile 的 `events`。握手靠写 `temp_storage/curr_sim_code.json` + `curr_step.json`（`:146-154`）。

**`start_server(int_counter)`，`:279` —— 真正的主循环**：① `while True:` + `if int_counter == 0: break`（`:306-309`）；② **轮询** `sim_folder/environment/{step}.json`（`:315-316`），不存在就 `sleep(server_sleep)`（`:412`）；③ `game_obj_cleanup`（`:303`）先把上轮物体事件复位为 `(name, None, None, None)`（见 `maze.turn_event_from_tile_idle()`，`maze.py:360`）；④ 把每个 persona 从旧 tile 挪到新 tile（`:348-350`）；⑤ **若 `planned_path` 为空（`:354`，已到达）**，把「对物体做动作」的事件加到 tile（`:359-360`）并移掉对应 blank 动作（`:363-365`）；⑥ **调认知链** `persona.move(maze, personas, personas_tile[name], curr_time)`（`:379`）返回 `(next_tile, pronunciatio, description)`；⑦ 写 `movement/<step>.json`（`:400-402`）；⑧ `step += 1`、`curr_time += timedelta(seconds=sec_per_step)`（`:406-407`）、`int_counter -= 1`。

**`open_server()`，`:415` —— 交互式 CLI**。开头打印免责声明（`:425-428`）：*"The agents in this simulation package are computational constructs ... lack human-like agency, consciousness, and independent decision-making."* 常用命令：

| 命令 | 作用 | 行 |
|---|---|---|
| `run <n>` / `fin` / `save` / `exit` | 跑 n 步 / 保存退出 / 保存 / **不保存并删库** | `:464` / `:439` / `:459` / `:452` |
| `print persona schedule <name>` | 已分解日程 | `:470` |
| `print hourly org persona schedule <name>` | 未分解的原始小时日程 | `:487` |
| `print persona associative memory (event\|thought\|chat) <name>` | **打印记忆流** | `:513` / `:522` / `:531` |
| `print current time` + `print persona current tile <name>` | 世界时间/坐标（查是否卡在轮询） | `:546` / `:496` |
| `call -- load history <file.csv>` / `call -- analysis <name>` | 加载初始记忆 / 临时对话 | `:577` / `:569` |

**调试最有用的三个**：`print persona associative memory (thought) <name>`（看反思产出了什么）、`print persona schedule <name>`（看计划是否被正确分解）、`print current time`（确认有没有卡在轮询）。**两个坑**：整个 REPL 被包在 `try/except` 里（`:438` / `:595-598`），异常只打 traceback 后继续，**不会崩但你可能看错边界**；`run` 分支里还有一处写死的全局引用 `rs.start_server(int_count)`（`:468`），用模块级变量 `rs` 而非 `self`，直接 import 这个类会 NameError。`ReverieServer.save()`（`:157`）写 `meta.json` 并对每个 persona 调 `persona.save()`（`persona.py:51`）落盘三个记忆文件。

### 6. `maze.py` 如何驱动世界

`Maze.__init__(maze_name)`（`:19`）读两类东西：**元数据** `maze_meta_info.json`（`:24`，含 `maze_width` / `maze_height` / `sq_tile_size`）；**`special_blocks/` 5 个 CSV**（`:48-72`）是「色块值 → 层级名」映射（如 `"25335, Double Studio, Studio, Common Room"`）；**`maze/` 5 个 CSV**（`:77-88`）是同名的 5 张矩阵，**每个是「宽×高 个元素的单行 CSV」**，`:104-110` 按 `maze_width` 切片重排成二维。

组装出的 `self.tiles[row][col]`（`:127-157`）每格是一个 dict：`{world, sector, arena, game_object, spawning_location, collision, events: set()}`；访问约定是 `self.tiles[y][x]`（`:246`）——**先 y 后 x**。**反查索引 `address_tiles`**（`:178-205`）是 `{地址字符串: {(x,y), ...}}`，execute 阶段靠它把「去 Hobbs Cafe 的柜台」翻译成候选坐标；`spawning_location` 地址带 `<spawn_loc>` 前缀（`:198`），execute 会 split 掉（`execute.py:50`）。**每个 game_object 自带一个初始 event**（`:160-168`）`(object_address, None, None, None)`，这就是 `reverie.py:363` 那个 `blank` 的来历——到达时用真实动作事件替换它。

关键方法：`access_tile`（`:226`，(x,y) → dict）、`get_tile_path`（`:249`，tile → `"world:sector:arena:game_object"`，level 控制截断层级）、`get_nearby_tiles`（`:286`，**方形而非圆形**邻域，注释 `:291-296` 明写 *"this implementation looks at a square boundary"*）、`add_event_from_tile` / `remove_event_from_tile`（`:327` / `:342`）、`turn_event_from_tile_idle`（`:360`，把 `("x:is:unmade", ...)` 复位成 `("x", None, None, None)`）、`remove_subject_events_from_tile`（`:369`，agent 离开时清空）、`turn_coordinate_to_tile`（`:208`，`math.ceil`）。

**驱动关系的本质**：`Maze` 是**被共享的可变世界模型，它自己不推进时间**。真正的推进者只有 `start_server()` 的循环；`Maze` 提供「地址 ↔ 坐标」双向映射和「这个 tile 上正在发生什么」。**agent 之间的间接影响完全通过 `tiles[][]["events"]` 这个共享 set 发生** —— perceive 读视野半径内 tile 的 events，这就是「偶遇」和「看到别人在做什么」的物理基础。没有消息队列，没有黑板以外的东西。

### 7. personas 与空间环境数据怎么组织

数据全在文件系统里，**没有数据库**（Django 侧的 `db.sqlite3` 只给 `translator` app 用）。一个 base simulation 的目录：

```
reverie/meta.json                 <- 世界级配置 (fork源/时间/步长/人名单)
environment/0.json                <- 初始: 每个 persona 的 x,y
personas/Isabella Rodriguez/bootstrap_memory/
    scratch.json                       <- 身份 + 超参 + 当前动作 + 日程 + 对话状态
    spatial_memory.json                <- MemoryTree: {world:{sector:{arena:[obj]}}}
    associative_memory/nodes.json      <- {node_1: {...}, node_2: {...}}
    associative_memory/embeddings.json <- {embedding_key: [1536 维向量]}
    associative_memory/kw_strength.json<- {"kw_strength_event":{...}, ...}
personas/Klaus Mueller/...   personas/Maria Lopez/...    (同构)
```

两个 base simulation（README Customization 段）：`base_the_ville_isabella_maria_klaus`（3 人）、`base_the_ville_n25`（25 人）。
**persona 的三层身份**（`scratch.py:33-46` 注释 + Isabella 真实数据）：`innate` = L0 永久核心特质（`"friendly, outgoing, hospitable"`），`learned` = L1 稳定特质（`"Isabella Rodriguez is a cafe owner of Hobbs Cafe..."`），`currently` = L2 当前状态（`"Isabella Rodriguez is planning on having a Valentine's Day party at Hobbs Cafe..."`）。**目前只有 `currently` 会随时间被 reflection 改写** —— `plan.revise_identity()` 只更新 `currently` 和 `daily_plan_req`，从不碰 `innate` / `learned`。

**超参因人而异**（这点很容易看错，必须核对磁盘值）：

| 超参 | `scratch.py` 代码默认值 | Isabella 的 `scratch.json` 实际值 |
|---|---|---|
| `vision_r` | 4（`:19`） | **8** |
| `att_bandwidth` | 3（`:21`） | **8** |
| `retention` | 5（`:23`） | **8** |
| `recency_decay` | 0.99（`:60`） | **0.995** |
| `importance_trigger_max` | 150（`:61`） | 150 |
| `recency_w` / `relevance_w` / `importance_w` | 1 / 1 / 1（`:57-59`） | 1 / 1 / 1 |

**只看 `Scratch.__init__` 的默认值会低估真实视野** —— 载入已有 persona 时（`:165-167`、`:194-200`）磁盘值会覆盖代码默认值，`scratch.json` 一共 47 个 key。

**空间记忆 `MemoryTree`**（`spatial_memory.py:15`）就是嵌套 dict `{world: {sector: {arena: [game_object, ...]}}}`，在 `perceive.py:51-68` 里**被现场扩充**——视野扫过的 tile 若含未知层级就插进树。读取侧三个方法（`:44` / `:63` / `:85`）产出 prompt 用的逗号分隔串。**注意 `MemoryTree` 是「已经看到的」，`Maze` 是「真实存在的」**，两者不同步时 agent 会不知道它没去过的地方。

**初始记忆怎么塞进去** —— `call -- load history <file>.csv`（`reverie.py:577`）→ `load_history_via_whisper()`（`converse.py:239`）。CSV 是**分号分隔**的 `<agent_name>; <whisper>; <whisper>; ...`，现成两份：`agent_history_init_n3.csv` 和 `agent_history_init_n25.csv`。

---

## 最小可运行入口

全部摘自 `README.md`，这是唯一权威来源。

**Step 1. 手工创建 `reverie/backend_server/utils.py`（仓库里没有，必须自己建）**，README 给出的完整内容：

```python
openai_api_key = "<Your OpenAI API>"   # key_owner = "<Name>"
maze_assets_loc = "../../environment/frontend_server/static_dirs/assets"
env_matrix = f"{maze_assets_loc}/the_ville/matrix"
env_visuals = f"{maze_assets_loc}/the_ville/visuals"
fs_storage = "../../environment/frontend_server/storage"
fs_temp_storage = "../../environment/frontend_server/temp_storage"
collision_block_id = "32125"           # Tiled 里那堵墙的色块编号
debug = True
```

它被 `.gitignore` 排除（防 API key 泄露），但代码里到处是 `from utils import *`（`reverie.py:34`、`maze.py:16`、`gpt_structure.py:12`），**不建这个文件任何东西都跑不起来**。`collision_block_id = "32125"` 是 Tiled 导出里那堵墙的色块编号，`execute.py` 全程用它（`:55`、`:66`、`:135`）。

**Step 2. 装依赖**（README 原文：*"Install everything listed in the `requirements.txt` file ... we tested our environment on Python 3.9.12."*）：先建 venv，`pip install -r requirements.txt`。

**Step 3. 起两个服务（必须同时跑）**。终端 A：`cd environment/frontend_server && python manage.py runserver`，浏览器打开 `http://localhost:8000/` 看到 *"Your environment server is up and running"* 才算好，**这个 tab 不能关**。终端 B：`cd reverie/backend_server && python reverie.py`，按提示依此输入 `Enter the name of the forked simulation: base_the_ville_isabella_maria_klaus` 与 `Enter the name of the new simulation: test-simulation`。

**Step 4. 推进**。浏览器打开 `http://localhost:8000/simulator_home`（tab 保持打开），回终端 B 输入 `run 100`。跑完回到 `Enter option:`，可继续 `run`、可 `exit`（不保存并删库）、可 `fin`（保存并退出）。

**Step 5. 回放与演示**。回放：`http://localhost:8000/replay/<simulation-name>/<starting-time-step>`（现成例子 `/replay/July1_the_ville_isabella_maria_klaus-step-3-20/1/`）。若要角色精灵正常显示的演示版，需**先改 `reverie/compress_sim_storage.py` 的 `compress(...)` 填上目标模拟名执行一次**，再访问 `http://localhost:8000/demo/<simulation-name>/<starting-time-step>/<simulation-speed>`，`<simulation-speed>` 取 1（最慢）到 5（最快）。

**Step 6. 加载初始记忆（可选）**：`call -- load history the_ville/agent_history_init_n3.csv`。

---

## 值得抄的工程手法

**1. 三项加权 + min-max 归一化 + 「值全同」特判**（`retrieve.py:107-271`）。`new_retrieve()` 把 recency / importance / relevance 各自 `normalize_dict_floats(..., 0, 1)` 后再加权求和。**三种信号量纲完全不同**（0.99 的幂乘 / 1—10 的 LLM 评分 / -1~1 的 cosine），归一化不可省略，否则 relevance 会因值域小被完全淹没；而 `range_val == 0` 分支（`:97-99`）把「所有候选同分」映射为 0.5 而非 0，退化情况下不会选出荒谬结果。三个打分器解耦，各自可替换。

**2. 「多次小调用」而非「一次大调用」的层级约束满足**（`plan.py:167-276`）。决定一个动作地址时连发 6 次 LLM：sector → arena → game_object → pronunciatio → event_triple → obj_desc，上一层输出是下一层输入。**每步候选空间被前一步大幅缩小**：一次性问「在哪、干嘛、emoji 是什么」，LLM 很可能给出 `maze.address_tiles`（`maze.py:178`）里不存在的地址，execute 阶段直接炸。分层后错误被限制在单层，且每层都能用「候选集合里有没有这个值」做硬校验——今天叫它 constrained decoding 的穷人版。

**3. 用文件系统当 IPC + 轮询队列**（`reverie.py:279-412`、`146-154`）。Python 后端与 Django 前端是互不知情的独立进程，可各自重启、崩溃、升级。更实际的好处是**每一步的完整输入输出都在磁盘上**——回放（`replay/`）、演示压缩（`compress_sim_storage.py`）、fork（`:58` 的 `copyanything`）三大功能几乎零成本地建立在此之上。今天多数 Agent 框架反而做不到：状态在内存里，重放就等于重跑（还花钱）。

**4. 「感知预算」三件套 vision_r / att_bandwidth / retention**（`perceive.py:25-181`）。三重收敛：限定半径 → 同 arena 内按距离只取最近 N 个 → 查最近 N 条里是否已有同一个 (s,p,o)，有则跳过。**LLM token 有价格，而「客厅里的床每帧都看到」是无意义信息**；这三条把「每秒看到什么」压缩成「有哪些新变化值得进记忆」，本质是事件去重 + 变化检测的最小实现。

**5. 「同一批记忆，两种检索」**（`retrieve.py:16-46` vs `199-271`，配合 `associative_memory.py:305-333`）。关键词倒排的 `retrieve()` 管「要不要搭话」（要确定性和低延迟），三项加权的 `new_retrieve()` 管「回忆与反思」（要召回率）。用一套检索服务两种需求是常见的架构错误。同样分工今天依然成立——BM25 + dense embedding 的混合检索，只是那时叫倒排和 cosine。

---

## 阅读路径

| # | 文件 | 为什么现在读它 |
|---|---|---|
| 1 | `README.md`（尤其 Setting Up / Running a Simulation / Customization） | 唯一权威 setup 来源，也是唯一写明「必须同时起两个服务、必须保持浏览器 tab 打开」的地方 |
| 2 | `persona/memory_structures/scratch.py`（先只读 `:1-80`） | 超参全集在这，`:33-46` 的 L0/L1/L2 身份注释尤其重要。先建立旋钮清单 |
| 3 | `persona/persona.py`（全文 272 行，很短） | **最该先读**。`move()`（`:185`）逻辑约 45 行，完整定义 `perceive → retrieve → plan → reflect → execute`。读完就有骨架 |
| 4 | `persona/cognitive_modules/perceive.py`（197 行） | 链的第一环：三重感知预算、poignancy 怎么产生、怎么扣 `importance_trigger_curr`（`:178`） |
| 5 | `persona/memory_structures/associative_memory.py`（重点 `:50-196`） | 记忆流物理实现：`ConceptNode` 字段、`seq_*` 头插方向、倒排表、`depth` 怎么算（`:209-210`） |
| 6 | `persona/cognitive_modules/retrieve.py`（284 行，**重中之重**） | 三个 `extract_*` + `new_retrieve` 加权公式。**领域最常被引用、最少被真正读到的一段**，务必亲手理清 `gw=[0.5,3,2]` 与三个 `*_w=1` 的关系 |
| 7 | `persona/cognitive_modules/reflect.py`（271 行） | 触发机制（`reflection_trigger:135`）+ `run_reflect:99` 四步；`generate_focal_points`（`:21`）用 `importance_ele_n` 做窗口别漏 |
| 8 | `persona/cognitive_modules/plan.py`（`:1-175`、`:461-520`、`:931-1007`） | 分三次读：生成层、长程计划、主入口。`:566-930` 的具体反应分支第二轮再看 |
| 9 | `backend_server/reverie.py`（重点 `:279-412` 主循环 + `:415-598` CLI） | 「轮询 → 挪 tile → 调 move → 写 movement → step+1」是整个系统的心跳 |
| 10 | `backend_server/maze.py`（重点 `:1-207` + `:286-324`）+ `cognitive_modules/execute.py` | 世界模型与执行：地址 → 候选 tile → A*/DFS → **只返回下一格** |

**明确可以直接跳过的**：`environment/frontend_server/storage/`（**208,432 个回放存档，占仓库 99.24%，一行都不要看**）、`environment/frontend_server/static_dirs/`（1,403 个美术资源，但运行时会被读，别删）、`prompt_template/defunct_run_gpt_prompt.py`（2,179 行）、`prompt_template/v1/` 与 `v2/`、`prompt_template/run_gpt_prompt.py` 全文（要哪个 grep 哪个）、`environment/frontend_server/translator/`、`path_finder.py` 里标注 defunct 的函数。

---

## 别踩的坑

### 维护状态：2023 年 8 月就冻结了

本地快照**最后一次提交 `2023-08-11 00:07:21 -0700`**（commit `fe05a71`，距今约 3 年 2 个月）。最后一个 PR 由 `eltociear` 提交（这位在 GitHub 上以修 typo 和死链著称）——**这不是持续演进，是「已经没人管了，只接受 trivial fix」**。注意本地 `.git` 是 **shallow clone**（存在 `.git/shallow`，`git rev-list --count HEAD == 1`），只能确认快照状态，无法推断上游后来的更新；据一般认识该仓库此后无实质更新，但上游 HEAD 与真实活跃度【待核】。**它是论文配套的研究产物，不是生产代码**：`execute.py:45` 那行 `print('aldhfoaf/????')`、`reflect.py:55` 的 `return {"this is blank": "node_1"}`、`plan.py:488` 的 `# TODO We need to create a daily_req here...`、几十处 `except: pass` 都是明证。

### 跑起来要做什么改造

1. **`openai==0.27.0` 是致命的**（`requirements.txt:31`）。`gpt_structure.py` 用的 `openai.ChatCompletion.create`（`:22`、`:48`、`:73`）与 `openai.Embedding.create`（`:280`）**在 openai >= 1.0 已被移除**。照 requirements 装能装上但基本跑不通；**必须重写那 4 个函数**（`ChatGPT_single_request` / `GPT4_request` / `ChatGPT_request` / `get_embedding`），好在调用点全集中在这一个 331 行文件里。
2. `gensim==3.8.0` + `numpy==1.25.2` —— gensim 3.x 的 C 扩展对 numpy ABI 敏感，现代环境大概率 import 就炸；**全仓 grep 不到 `import gensim`**，疑似无用依赖【待核】，确认后可移除。
3. `sklearn==0.0` 是历史遗留空壳包（与真的 `scikit-learn==1.3.0` 并列），会尝试装一个已不可用的 PyPI 占位包，**建议直接删这一行**。
4. `Django==2.2` 已 EOL 多年且有已知 CVE，`Pillow==8.4.0` / `requests==2.26.0` / `urllib3==1.26.7` 同为 2021 年产物。**前端服务只在 localhost 跑，不要暴露公网。**
5. **两个同名 `global_methods.py`**（`reverie/` 与 `reverie/backend_server/` 各一份），靠 `sys.path` hack 决定加载哪个。改了一处不生效时先确认改的是哪个。
6. **所有模块顶部都写 `sys.path.append('../../')`**，意味着**必须在 `reverie/backend_server/` 目录下执行 `python reverie.py`**；从仓库根跑会 ModuleNotFoundError，IDE 点运行基本必炸。
7. **selenium 是硬依赖**（`reverie.py:31` `from selenium import webdriver`），但全文件并未真正使用【待核：遗留 import】，白白带来一整套浏览器驱动依赖。
8. **存储开销**：那 20 次预跑模拟在 `storage/` 生成了 208,432 个文件，克隆本身就是负担；跑新模拟每步同样生成 2 个 JSON，`run 1000` 就是 2000 个文件。
9. **钱**：README 原文 *"could be somewhat costly, especially when there are many agents"*。每步每 agent 都可能触发多次调用（poignancy 打分、6 级地址生成、偶发反思），且 `get_embedding()` **没有缓存**（`gpt_structure.py:276-281`）。**先跑 3 人的 `base_the_ville_isabella_maria_klaus`，别一上来跑 25 人。**
10. **API 限流会挂死**：README 用词是 **hang 而不是报错**，因为 `except: print("ChatGPT ERROR"); return "ChatGPT ERROR"`（`:54-56`、`:79-81`）会把错误吞成字符串继续往下走，产生垃圾记忆。改造时把这层裸 except 换成显式重试 + 抛错。

### 结论

**原样 `pip install -r requirements.txt` + `python reverie.py` 今天跑不通**，三个不可回避的阻塞点：(a) `openai==0.27.0` 依赖已被移除的旧 API，须重写 `gpt_structure.py` 的 4 个出口函数；(b) `gensim==3.8.0` 与 `numpy==1.25.2` 的 ABI 组合大概率 import 失败（gensim 疑似未使用）；(c) 各种 `sys.path` hack 要求必须从 `reverie/backend_server/` 启动。

**最低成本的啃法：不要试图把它跑起来，把它当成一篇可以逐行读的论文附录。** 精读 `persona.py` 的 `move()`（约 45 行）+ `retrieve.py` 的 `new_retrieve`（约 70 行），这 115 行包含了这套认知架构 90% 的信息量。真要跑，建议先把 LLM 调用层换成 litellm 或新版 openai SDK，再回来验证行为。
