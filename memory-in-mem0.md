# mem0 的记忆机制（Memory in mem0）

> 本文基于 [`mem0ai/mem0`](https://github.com/mem0ai/mem0) 的真实源码撰写，分析锁定 commit [`3b9aed8`](https://github.com/mem0ai/mem0/tree/3b9aed866ae70d29043388ed0ae5cc4e1844f3e8)（2026-07-01）。文中所有实现细节、常量、文件路径与行号区间均可回溯到该版本源码，未做臆测；凡不确定处均已明确标注。

mem0 是目前 GitHub 上 star 最多的**通用记忆层(universal memory layer)**开源项目,它的定位与前两篇拆解的对象截然不同:[OpenClaw](./memory-in-openclaw.md) 与 [Hermes](./memory-in-hermes-agent.md) 都是"自带记忆的完整 Agent",而 mem0 只做一件事——**把记忆抽象成一个可被任意 Agent 调用的独立中间件**。你把对话丢给它,它替你抽取事实、去重、向量化、存储、检索,再把最相关的记忆还给你。一句话:

> **The memory layer for AI Agents.（AI Agent 的记忆层。）**

这个定位决定了 mem0 的设计重心:它不关心 Agent 循环、工具调用或沙箱,而是把全部工程能力压在**"如何从对话中提炼出高质量的长期记忆,并高效地存取"**这一个问题上。

本文自底向上拆解 mem0 的记忆架构:

1. [整体架构:双存储 + 提供商工厂](#1-整体架构双存储--提供商工厂)
2. [写入管线:从"两阶段"到"V3 增量抽取"的演进](#2-写入管线从两阶段到v3-增量抽取的演进)
3. [事件模型与 infer 开关](#3-事件模型与-infer-开关)
4. [去重:哈希精确去重 + LLM 语义去重](#4-去重哈希精确去重--llm-语义去重)
5. [SQLite 历史库:不可变的审计日志](#5-sqlite-历史库不可变的审计日志)
6. [检索管线:语义 + BM25 + 实体加权的混合打分](#6-检索管线语义--bm25--实体加权的混合打分)
7. [实体存储:被移除的图数据库与"图 lite"替身](#7-实体存储被移除的图数据库与图-lite-替身)
8. [作用域、过期与程序性记忆](#8-作用域过期与程序性记忆)
9. [Proxy、Client 与配置系统](#9-proxyclient-与配置系统)
10. [整体数据流与设计取舍](#10-整体数据流与设计取舍)

---

## 1. 整体架构:双存储 + 提供商工厂

mem0 的核心是 `mem0/memory/main.py` 里的 `Memory` 类(`:444`,及其完整异步孪生 `AsyncMemory`,`:2095`)。构造函数(`:445-509`)把五个可插拔组件拼装起来:

| 组件 | 职责 | 实例化位置 |
| --- | --- | --- |
| **Embedder** | 把文本转成向量 | `EmbedderFactory.create(...)`(`:448`) |
| **Vector Store** | 存记忆内容(向量 + payload)、做语义检索 | `VectorStoreFactory.create(...)`(`:453`) |
| **LLM** | 从对话中抽取事实 | `LlmFactory.create(...)`(`:456`) |
| **SQLite History DB** | 记录每次增删改的审计日志 | `SQLiteManager(self.config.history_db_path)`(`:457`) |
| **Reranker**(可选) | 检索后重排 | `RerankerFactory.create(...)`(`:463-468`) |

### 1.1 双存储:内容 vs 历史

mem0 最基础的设计决策是把"记忆的当前状态"和"记忆的变更历史"分成**两个物理存储**:

- **向量库**回答"用户现在记得什么?哪些记忆和这条 query 语义相近?"——它是**可变的当前状态**。
- **SQLite 历史库**回答"这条记忆经历了什么?"——它是**不可变的事件账本**(旧值、新值、事件类型、时间戳)。

每次写入都会**双写**:例如 `_create_memory` 先向 `self.vector_store` 插入向量(`:1901-1905`),再向 `self.db` 追加一条 `"ADD"` 历史(`:1906-1915`)。这种"内容 + 审计"分离让你既能快速检索,又能完整回溯任意一条记忆的生命周期(见 [第 5 节](#5-sqlite-历史库不可变的审计日志))。

### 1.2 提供商工厂:极致的可插拔

mem0 用四个工厂(`mem0/utils/factory.py`)把所有后端都做成插件,数量之多堪称"生态级":

- **向量库 23 种**(`factory.py:179-204`):qdrant、chroma、pgvector、milvus、pinecone、redis、weaviate、faiss、elasticsearch、mongodb、opensearch、supabase、azure_ai_search、s3_vectors、neptune……
- **Embedder 11 种**(`factory.py:151-163`):openai、ollama、huggingface、azure_openai、gemini、vertexai、together、aws_bedrock、fastembed……
- **LLM 18 种**(`factory.py:40-59`):openai、anthropic、gemini、groq、together、aws_bedrock、litellm、deepseek、xai、vllm、ollama……
- **Reranker 5 种**(`factory.py:230-239`):cohere、sentence_transformer、zero_entropy、llm_reranker、huggingface。

这种"一切皆工厂"的架构,是 mem0 能被塞进任意技术栈的根本原因——它不绑定任何一家向量库或模型厂商。

---

## 2. 写入管线:从"两阶段"到"V3 增量抽取"的演进

这是本次拆解**最重要、也最容易踩坑**的一节。

mem0 的经典招牌一直被描述为**两阶段管线**:第一阶段用 LLM 从对话里抽取候选事实,第二阶段再用一次 LLM 把新事实和已有的相似记忆逐一比对,决定 **ADD / UPDATE / DELETE / NONE**。这个设计在无数博客和论文里被引用。

> **但在本文锁定的 commit `3b9aed8` 上,这套经典两阶段机制已经从活跃写入路径中退役了。**

经代码验证:经典的 `FACT_RETRIEVAL_PROMPT`(`prompts.py:15`)与 `DEFAULT_UPDATE_MEMORY_PROMPT`(`prompts.py:176`)依然躺在 `prompts.py` 里,但**已成孤儿**——`grep` 全仓库,它们只被测试和一个**从未被调用**的 `mem0/memory/utils.py::get_fact_retrieval_messages` 引用。真正在跑的是一套 **"V3 增量批处理管线"(V3 PHASED BATCH PIPELINE)**,由 `ADDITIVE_EXTRACTION_PROMPT` 驱动,**只用一次 LLM 调用**完成抽取 + 去重 + 链接。

### 2.1 V3 管线:七个阶段,一次 LLM 调用

入口是 `Memory.add()`(`:717`),规范化消息后调用 `_add_to_vector_store(...)`(`:821`)。当 `infer=True`(默认)时,代码头部赫然写着 `# === V3 PHASED BATCH PIPELINE ===`(`:868`),随后展开七个阶段:

- **Phase 0 — 上下文收集**:取本会话最近 10 条消息 `self.db.get_last_messages(session_scope, limit=10)`(`:872`),拼接当前对话 `parse_messages(messages)`(`:873`)。
- **Phase 1 — 检索已有相似记忆**:把整段对话作为一条 query 向量化,`vector_store.search(top_k=10, filters={user_id/agent_id/run_id})`(`:878-883`)。取回后做一个精巧的**"防幻觉"重编号**——把 UUID 映射成整数下标 `0,1,2...`,只把 `{"id": "0", "text": ...}` 喂给 LLM(`:885-890`),避免模型对着一长串 UUID 瞎编 ID。
- **Phase 2 — 单次 LLM 抽取**:system prompt = `ADDITIVE_EXTRACTION_PROMPT`(agent 场景再拼 `AGENT_CONTEXT_SUFFIX`,`:894-896`);user prompt 由 `generate_additive_extraction_prompt(...)` 组装,内含"已有记忆 + 最近消息 + 新消息 + 观察日期 + 自定义指令"(`:900-905`);以 `response_format={"type":"json_object"}` 强制 JSON 输出(`:908-914`)。
- **Phase 3 — 批量向量化**:对抽出的事实 `embed_batch(mem_texts, "add")`(`:946`)。
- **Phase 4-5 — 构造记录 + 去重**(见 [第 4 节](#4-去重哈希精确去重--llm-语义去重))。
- **Phase 6 — 批量落盘**:`vector_store.insert(...)`(`:997-1014`),并批量写入历史(全部 `"event":"ADD"`,`:1016-1036`)。
- **Phase 7 — 实体链接**:批量抽取实体、向量化、upsert 到实体库(`:1038` 起,见 [第 7 节](#7-实体存储被移除的图数据库与图-lite-替身))。

LLM 返回的 JSON 形如:

```json
{"memory": [
  {"id": "0", "text": "用户是一名 Rust 后端工程师", "attributed_to": "user", "linked_memory_ids": ["<uuid>"]}
]}
```

### 2.2 增量抽取 vs 经典两阶段:到底变了什么

`ADDITIVE_EXTRACTION_PROMPT`(`prompts.py:468-944`)的文件头注释一语道破:*"V3 Additive Extraction Prompt (ADD-only with memory linking) — Ported from platform"*(`:463-466`)。它与经典两阶段的核心差异:

| 维度 | 经典两阶段(已退役) | V3 增量抽取(现役) |
| --- | --- | --- |
| **LLM 调用次数** | 2 次(抽取 + 决策) | **1 次**(抽取 + 去重 + 链接一次搞定) |
| **操作集** | ADD / UPDATE / DELETE / NONE | **只有 ADD** |
| **输出形状** | `{"facts": [...]}` → `{"memory":[{event,...}]}` | `{"memory":[{id,text,attributed_to,linked_memory_ids}]}` |
| **抽取来源** | 用户消息为主 | **用户 + 助手消息都抽** |
| **记忆粒度** | 短事实 | 富上下文记忆(15-80 词),带时间锚定 |

这里有个关键的哲学转向:**V3 管线是"只增不改"的(ADD-only)**。它不再让 LLM 去 UPDATE 或 DELETE 旧记忆,而是靠去重(跳过重复)避免冗余,靠 `linked_memory_ids` 构建一张软性的记忆关联图。修改与删除被下放给**手动 API**(`update()` / `delete()`)。

> **给读者的对照**:这与 Hermes 的取舍恰好相反。Hermes 让 Agent 主动 `replace`/`remove` 记忆、严格控制容量;mem0 的 V3 则选择"只追加、语义去重、软链接",把"新旧冲突"交给检索时的排序去自然淘汰。前者追求精炼,后者追求召回。

---

## 3. 事件模型与 infer 开关

### 3.1 四种事件,但现役路径只发一种

mem0 的事件是四个**纯字符串字面量**(不是 Python enum):`"ADD" | "UPDATE" | "DELETE" | "NONE"`。

- **写入(add)路径**:V3 管线只会产出 `"ADD"`(`:861`、`:1022`、`:1034`)。
- **手动路径**:`_update_memory` 写 `"UPDATE"`(`:2007`)、`_delete_memory` 写 `"DELETE"`(`:2039`)——但这两个只被公开的 `update()`(`:1767`)、`delete()`(`:1808`)、`delete_all()`(`:1829`)调用,**不在 add 里**。
- `"NONE"` 与 UPDATE/DELETE 的**自动决策**只存在于已退役的 `DEFAULT_UPDATE_MEMORY_PROMPT` 里,现役代码中没有那个分发循环。

### 3.2 infer:要不要让 LLM 介入

`add()` 有一个决定性开关 `infer: bool = True`(`:727`):

- **`infer=True`(默认)**:走完整 V3 管线,LLM 抽取事实。
- **`infer=False`(`:832-866`)**:**原样存储,不调 LLM、不去重**。逐条消息(跳过 `system` 角色)直接向量化、`_create_memory`,返回 `{"event":"ADD"}`。这是给"我已经知道该存什么、别帮我瞎抽"场景准备的旁路。

---

## 4. 去重:哈希精确去重 + LLM 语义去重

mem0 的去重是**两层协作**的:

**第一层——哈希精确去重(无 LLM)**,在 V3 的 Phase 4-5(`:957-995`):
1. 收集 Phase 1 取回的 top-10 已有记忆的 `payload["hash"]` 存入 `existing_hashes`(`:959-963`)。
2. 批内维护 `seen_hashes`(`:966`)。
3. 每条新事实算 `hashlib.md5(text.encode()).hexdigest()`(`:972`),命中任一集合就跳过(`:973-976`)。

这只能拦截**逐字重复**。

**第二层——LLM 语义去重(在 prompt 里)**:`ADDITIVE_EXTRACTION_PROMPT` 明确把 `Recently Extracted Memories` 设为*"你的主要去重参照"*(`prompts.py:501-503`),把 `Existing Memories` 用于*"仅做去重与链接……语义等价则跳过"*(`prompts.py:507-511`)。也就是说,**语义级去重被委托给了 LLM 自己**。

值得注意:因为现役路径是 ADD-only,这里的"去重"意味着**跳过(skip)**,而**不是**经典两阶段里的"合并/更新(merge/update)"。

---

## 5. SQLite 历史库:不可变的审计日志

`mem0/memory/storage.py` 的 `SQLiteManager`(`:11`)管理两张表。

### 5.1 `history` 表——审计账本

DDL(`storage.py:108-119`):

```sql
CREATE TABLE IF NOT EXISTS history (
    id TEXT PRIMARY KEY, memory_id TEXT, old_memory TEXT, new_memory TEXT,
    event TEXT, created_at DATETIME, updated_at DATETIME,
    is_deleted INTEGER, actor_id TEXT, role TEXT)
```

每类事件记录的内容:

| 事件 | old_memory | new_memory | is_deleted |
| --- | --- | --- | --- |
| **ADD** | NULL | 新内容 | 0 |
| **UPDATE** | 旧内容 | 新内容 | 0 |
| **DELETE** | — | — | 1(软删除) |

读取时 `get_history(memory_id)` 按 `created_at ASC, updated_at ASC` 排序(`storage.py:235`),由 `Memory.history()`(`:1871`)对外暴露。这让"这条记忆什么时候被谁改成了什么"完全可追溯。

### 5.2 `messages` 表——滚动缓冲区

第二张表 `messages`(`storage.py:128-148`)是**每个会话作用域只保留最近 10 条**的滚动缓冲(插入后立即淘汰旧行,`storage.py:282-291`),供 V3 Phase 0 取上下文用。**它不是审计日志,别和 `history` 混淆。**

### 5.3 迁移与线程安全

- **迁移是"列集合 diff"式的**(`_migrate_history_table`,`storage.py:20-100`):没有版本号,而是用 `PRAGMA table_info` 读现有列,和硬编码的 `expected_cols` 求差集;不一致就 `RENAME TO history_old` → 建新表 → 只拷贝交集列。整个过程包在一个事务里。
- **线程安全**:连接开 `check_same_thread=False`(`:14`),再用**单个 `threading.Lock()`**(`:15`)串行化所有操作——每个方法体都 `with self._lock:`。简单粗暴但可靠。

---

## 6. 检索管线:语义 + BM25 + 实体加权的混合打分

`search()`(`:1331`)默认 `threshold=0.1, top_k=20`(`:1335-1341`),真正干活的是 `_search_vector_store`(`:1580`)。它是一条**混合检索**流水线:

1. **预处理**:`lemmatize_for_bm25(query)` + `extract_entities(query)`(`:1586-1587`)。
2. **向量化 query**:`embed(query, "search")`(`:1590`)。
3. **语义检索(过量取回)**:`internal_limit = max(limit*4, 60)`(`:1593-1596`)——多取回来一批做后续重排。
4. **关键词检索**:`vector_store.keyword_search(...)`(`:1599-1601`)。注意:**只有实现了 `keyword_search` 的向量库才有 BM25**,否则初始化时告警并降级为纯语义(`:500-507`)。
5. **BM25 归一化**:sigmoid 归一(`:1603-1611`)。
6. **实体加权**:query 含实体时算 `_compute_entity_boosts`(`:1613-1616`)。
7. **过滤过期**:`show_expired=False` 时跳过过期记忆(`:1618-1629`)。
8. **合并排序**:`score_and_rank(...)`(`:1631`)。

### 6.1 组合打分公式

核心在 `utils/scoring.py`:

```python
combined = min((semantic + bm25 + entity_boost) / max_possible, 1.0)
```

- **阈值在合并前就对语义分把关**——语义分低于 `threshold` 的候选直接丢弃,哪怕 BM25/实体想加分也没用(`scoring.py:113-115`)。
- `max_possible` 自适应:纯语义 = 1.0;有 BM25 再 +1.0;有实体再 +`ENTITY_BOOST_WEIGHT(0.5)`。三者全开时上限 2.5(`scoring.py:100-106`)。

### 6.2 可选重排

只有当 `search(rerank=True)` **且**配置了 reranker **且**有结果时才重排(`:1452-1457`),失败自动回退原结果。这是纯粹的锦上添花层。

---

## 7. 实体存储:被移除的图数据库与"图 lite"替身

如果你读过老版本 mem0 或它的博客,一定听说过 **Graph Memory**——把实体建成节点、关系建成边,外接 Neo4j / Memgraph / Kuzu / Neptune。

> **但在本文锁定的 commit 上,外接图数据库集成已被彻底移除。** 全仓库没有 `graph_memory.py`、没有 `MemoryGraph` 类、`MemoryConfig` 里没有 `graph_store` 字段、没有 `enable_graph` 开关。`grep "graph" mem0/memory/main.py` **零命中**。仅存的踪迹是 `exceptions.py:396` 里一句提到 `kuzu`/`graph_store` 的报错文案。

官方文档 `docs/platform/features/graph-memory.mdx` 交代了来龙去脉:*"早期版本通过 `enable_graph` 标志和 `graph_store` 配置块外接图数据库(Neo4j、Memgraph、Kuzu、Apache AGE 或 Neptune)。该集成已被原生内建的 Graph Memory 取代……即使你仍传 `enable_graph` 参数,它也会被忽略"*(`:90-94`)。

### 7.1 OSS 侧的替身:spaCy 实体库

在自托管的 `Memory` 里,取代图数据库的是一个**"图 lite"实体存储**——本质是**第二个向量库集合**(集合名 = 主集合名 + `_entities`,`:377`),惰性创建(`:515-537`):

- **实体抽取用 spaCy,不用 LLM**:`extract_entities(text)`(`utils/entity_extraction.py`)抽取专有名词、引号文本、名词短语等,返回 `(entity_type, entity_text)` 元组;spaCy 不可用则返回 `[]`。
- **链接**:`_link_entities_for_memory`(`:664`)把实体 upsert 到实体库,`_upsert_entity`(`:562`)对同名实体做精确匹配(或语义相似 ≥ 0.95)后,把 `memory_id` 追加进该实体的 `linked_memory_ids`。
- **检索加权**:`_compute_entity_boosts`(`:1685`)在检索时,对 query 里的实体(最多 8 个)去实体库搜相似实体(相似度 ≥ 0.5),命中后给关联记忆加分。公式里还有个巧思——`memory_count_weight = 1/(1 + 0.001*(num_linked-1)²)`(`:1753`),**关联记忆越多的实体权重越低**,以抑制"烂大街"的通用实体。

所以现在的 mem0 OSS **不再存"张三 --管理--> 项目 A"这样的带类型边**,而是靠"共享实体 → 检索时互相加分"实现记忆间的软关联。这是一种**无 schema、基于共现**的轻量图。

> **重要提醒**:写书或做技术选型时,务必把 mem0 的"图记忆"讲清楚是**历史 / 平台托管特性**——OSS 运行时如今是 spaCy 实体库 + 检索加权,而非图数据库。

---

## 8. 作用域、过期与程序性记忆

### 8.1 三级作用域

mem0 用 `user_id` / `agent_id` / `run_id` 三个维度隔离记忆(`_build_filters_and_metadata`,`:281-364`):

- 三者**至少提供一个**,否则抛 `Mem0ValidationError`(`:351-357`)。
- 每个 id 同时写进"存储元数据"和"查询过滤器",作用域在向量库查询层强制生效。
- `_build_session_scope`(`:367`)把三个键**排序后**拼成确定性字符串(如 `agent_id=x&user_id=y`),作为 `messages` 缓冲区的 key——排序保证了传参顺序无关。

### 8.2 记忆过期(TTL)

`add(..., expiration_date="YYYY-MM-DD")` 可给记忆设生存期。`_payload_is_expired`(`:397`)在检索时判断是否过期。注意这是**查询时过滤**(`search`/`get_all` 会跳过过期项),不是硬删除;而单条 `get()` **不做过期过滤**,会返回原始项。

### 8.3 程序性记忆

当 `agent_id` 存在且 `memory_type == "procedural_memory"` 时(`:805-806`),走 `_create_procedural_memory`(`:1918`)。它用 `PROCEDURAL_MEMORY_SYSTEM_PROMPT`(`prompts.py:326`)让 LLM 把 Agent 的**执行历史**总结成一份逐步编号、逐字保留动作结果的操作摘要——**绕过事实抽取和去重**,整段存为一条记忆。这与 Hermes 的"技能即程序性记忆"异曲同工,但 mem0 的做法更轻:它只是把执行轨迹压缩成一条可检索的文本。

---

## 9. Proxy、Client 与配置系统

### 9.1 三种使用形态

mem0 提供三层入口,适配不同集成深度:

| 形态 | 类 | 说明 |
| --- | --- | --- |
| **OSS 自托管** | `Memory`(`main.py:444`) | 全流程本地跑,自带 LLM/向量库/SQLite |
| **托管平台** | `MemoryClient`(`client/main.py:71`) | 薄 HTTP 封装,一切交给 `api.mem0.ai` 云端 |
| **OpenAI 直插** | `Mem0`(`proxy/main.py`) | OpenAI 兼容的 `chat.completions.create` |

### 9.2 Proxy:最丝滑的集成

`proxy/main.py` 的 `Mem0` 类提供了一个 **OpenAI 兼容的 drop-in**:你把 `openai.chat.completions.create` 换成它,它会在每次对话时**自动读写记忆**——发请求前拉取相关记忆注入到 system prompt(`_fetch_relevant_memories`),发完后用后台守护线程异步把新对话存进记忆(`_async_add_to_memory`)。底层用 litellm 转发。对开发者而言,加记忆几乎"零改动"。

### 9.3 配置系统

`MemoryConfig`(`configs/base.py:29-57`)是一个 Pydantic 模型,可配 `vector_store` / `llm` / `embedder` / `reranker` / `history_db_path` / `custom_instructions`,以及 `version`(默认 `"v1.1"`)。`from_config(config_dict)`(`:688`)让你用一个 dict 声明式地拼出整套记忆栈。

`version` 决定**返回格式**:`v1.1+` 把结果包成 `{"results": [{"id", "memory", "event"}]}` 信封(`:760`),老的 `v1.0` 返回裸列表。

---

## 10. 整体数据流与设计取舍

把所有环节串起来:

```
                    ┌─────────────────────────────────────┐
       add(msgs) ──▶│  V3 增量管线 (infer=True, 1 次 LLM)   │
                    └─────────────────────────────────────┘
                       │Phase0 取最近10条(messages表)
                       │Phase1 向量检索 top-10 已有记忆 ──┐
                       │Phase2 ADDITIVE_EXTRACTION 抽取   │(防幻觉:UUID→整数)
                       │       (用户+助手, 只产 ADD)       │
                       │Phase3 批量 embed                 │
                       │Phase4-5 哈希去重 + LLM 语义去重   │
                       ▼                                  │
        ┌──────────────────────┐        ┌────────────────▼──────────┐
        │  向量库 (23 种后端)     │        │  SQLite 历史库              │
        │  内容 + payload + hash │        │  history 表: 不可变审计账本  │
        │  ← 可变当前状态         │        │  ADD/UPDATE/DELETE + 时间戳 │
        └──────────────────────┘        │  messages 表: 10 条滚动缓冲  │
                 ▲   │                   └───────────────────────────┘
                 │   │Phase7 实体链接
                 │   ▼
                 │  ┌──────────────────────┐
                 │  │  实体库 (_entities 集合)│  spaCy 抽取, 非图数据库
                 │  │  实体→linked_memory_ids│  ("图 lite" 替身)
                 │  └──────────────────────┘
                 │           │
      search() ──┘           │实体加权 boost
        │ 语义 + BM25 + 实体 混合打分 (阈值先卡语义分)
        │ 可选 reranker 重排
        ▼
     top_k 记忆 ──▶ 注入 Agent 上下文
```

拆解至此,mem0 的记忆机制有几处极具借鉴价值,也有几处需要警惕的"文档滞后":

1. **记忆作为独立中间件**。mem0 把记忆从 Agent 里剥离成一个可被任意栈调用的层,靠四大工厂支持 23 种向量库、18 种 LLM——这种"一切皆插件"的架构是它成为 star 王的根本。

2. **双存储分离**。可变的向量内容 + 不可变的 SQLite 审计账本,让"快速检索"和"完整回溯"两个诉求各得其所,是很干净的关注点分离。

3. **V3 只增不改的取舍**。现役管线放弃了经典的"LLM 决策 UPDATE/DELETE",改为"只追加 + 语义去重 + 软链接",把一致性交给检索排序去自然淘汰。这降低了写入复杂度和 LLM 成本(2 次调用→1 次),代价是记忆库会缓慢膨胀、旧的错误事实不会被主动纠正。**这是与 Hermes"严格容量 + 主动 replace"截然相反的哲学。**

4. **混合检索 + 实体加权**。语义 + BM25 + 实体三路打分、阈值先卡语义分、过量取回再重排——是一套成熟的召回工程。

5. **"图 lite"取代图数据库**。用第二个向量集合 + spaCy 实体 + 检索加权,以极低运维成本换取记忆间的软关联。**但要清醒:老文档里的 Neo4j/图边在 OSS 已不复存在。**

6. **文档与代码的落差是最大陷阱**。经典两阶段 prompt、agent/user 双抽取分支、外接图数据库——这些被广泛引用的"mem0 特性"在 `3b9aed8` 上要么成了孤儿代码,要么被整体移除。**任何基于 mem0 的技术判断,都必须以你实际使用的 commit 为准,而非博客描述。** 这也是本系列坚持"锁定 commit、回溯源码"的原因。

---

### 附:可直接查阅的源码入口

| 主题 | 关键文件(相对 mem0 仓库根) |
| --- | --- |
| 核心引擎(Memory / AsyncMemory) | `mem0/memory/main.py` |
| 写入 prompt(V3 增量 + 已退役经典) | `mem0/configs/prompts.py` |
| SQLite 历史 / 消息缓冲 | `mem0/memory/storage.py` |
| 混合检索打分 | `mem0/utils/scoring.py` |
| 实体抽取(spaCy "图 lite") | `mem0/utils/entity_extraction.py` |
| 提供商工厂(LLM/Embedder/向量库/Reranker) | `mem0/utils/factory.py` |
| 配置模型 | `mem0/configs/base.py` |
| 托管平台 SDK | `mem0/client/main.py` |
| OpenAI 兼容 Proxy | `mem0/proxy/main.py` |
| 遥测 | `mem0/memory/telemetry.py` |
| 图记忆概念与移除说明 | `docs/platform/features/graph-memory.mdx` |

> 本文为《Memory in Action》系列的一章,基于源码撰写,仅供技术学习与研究。所有代码常量、路径与行号以 mem0 commit `3b9aed8` 为准,后续版本可能演进。
