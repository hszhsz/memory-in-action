# deer-flow 的记忆机制（Memory in deer-flow）

> 本文基于 [`bytedance/deer-flow`](https://github.com/bytedance/deer-flow) 的真实源码撰写，分析锁定 commit [`4e6248f`](https://github.com/bytedance/deer-flow/tree/4e6248f013aaed84ad5250fe4969de44beebeb7e)（2026-07-02）。文中所有实现细节、常量、文件路径与行号区间均可回溯到该版本源码，未做臆测；凡代码与文档不一致处均已明确标注。

前六篇拆解的 [OpenClaw](./memory-in-openclaw.md)、[Hermes](./memory-in-hermes-agent.md)、[mem0](./memory-in-mem0.md)、[MemPalace](./memory-in-mempalace.md)、[Codex](./memory-in-codex.md) 与 [opencode](./memory-in-opencode.md) 里，我们见过两种极端：mem0/Codex 有专门的 LLM 事实抽取管线，opencode 干脆**没有**长期记忆子系统、全靠声明式文件 + 会话历史 + 快照。deer-flow 是字节跳动开源的深度研究(deep research)Agent，它给出了**第三种答案**——而且很不一样：

> **deer-flow 建在 [LangGraph](https://langchain-ai.github.io/langgraph/) 之上,但它没有用 LangGraph 自带的 store(向量记忆)来做长期记忆;它自己手写了一套 mem0 风格、但零向量的 LLM 事实抽取记忆子系统,落成纯 JSON 文件。**

这句话里藏着 deer-flow 记忆架构的三个反直觉之处，也是本章要逐一拆开的：

1. **它同时挂着 LangGraph 的两套官方持久化设施**(checkpointer 存线程状态、store 存跨线程 KV),**但 store 几乎没被当记忆用**——真正的语义记忆是另起炉灶手写的。
2. **它的语义记忆是 mem0 血统的**(读当前记忆→LLM 抽取事实→ADD/UPDATE/DELETE),**但检索端零向量、零嵌入**——注入时把整个记忆文件按置信度排序、塞进 token 预算,更像 MemPalace 的"薄索引注入"而非 mem0 的向量召回。
3. **它有多层记忆,却散落在四个互不相通的子系统里**:LangGraph checkpointer、LangGraph store、自研关系型持久化(runs/events)、以及那套 JSON 文件语义记忆。理解 deer-flow 记忆的第一课,就是**先分清这四套谁管谁**。

更妙的是,deer-flow 把"记忆写入"这件事**接在了上下文压缩(summarization)的钩子上**——当对话即将被摘要、旧消息即将被丢弃前的最后一刻,记忆子系统抢先把这段"要被遗忘的对话"抽成事实存下来。**遗忘的出口,正是长期记忆的入口。** 这是本章最漂亮的一处设计。

本文自底向上拆解:

1. [四套持久化子系统总览](#1-四套持久化子系统总览)
2. [LangGraph 的 checkpointer 与 store:被"冷落"的官方记忆](#2-langgraph-的-checkpointer-与-store被冷落的官方记忆)
3. [自研关系层:runs 与只增事件日志](#3-自研关系层runs-与只增事件日志)
4. [语义记忆的数据模型:六段摘要 + 原子事实](#4-语义记忆的数据模型六段摘要--原子事实)
5. [写入触发:遗忘的出口即记忆的入口](#5-写入触发遗忘的出口即记忆的入口)
6. [抽取管线:单阶段 LLM 与 ADD/UPDATE/DELETE](#6-抽取管线单阶段-llm-与-addupdatedelete)
7. [反射:让 Agent 记住自己的错误](#7-反射让-agent-记住自己的错误)
8. [读取端:零向量的置信度排序注入](#8-读取端零向量的置信度排序注入)
9. [隐私与提示注入防御:OWASP LLM01](#9-隐私与提示注入防御owasp-llm01)
10. [工作记忆、声明式 SOUL.md、技能与子 Agent](#10-工作记忆声明式-soulmd技能与子-agent)
11. [整体数据流与设计取舍](#11-整体数据流与设计取舍)

---

## 1. 四套持久化子系统总览

deer-flow 的持久化不是一层,而是**四套彼此独立**的子系统,在 Gateway 启动时于 `app/gateway/deps.py:169-213` 一一接线。理清它们是理解全章的地基:

| 子系统 | 存什么 | 后端 | 是不是"记忆" |
| --- | --- | --- | --- |
| **LangGraph checkpointer** | 图的线程状态(消息历史 + 节点状态) | memory / sqlite / postgres | 情景记忆(会话可恢复) |
| **LangGraph store** | 跨线程 KV | memory / sqlite / postgres | **名义上是长期记忆,实际几乎没用** |
| **自研关系层(SQLAlchemy)** | runs / run_events / threads_meta / users / feedback / channel_* | sqlite / postgres | 运行审计(情景记忆的元数据) |
| **agents/memory(JSON 文件)** | 用户画像摘要 + 原子事实 | 纯 JSON 文件 | **真正的语义长期记忆** |

`langgraph.json` 里同时声明了图入口和 checkpointer:

```json
"graphs": { "lead_agent": "deerflow.agents:make_lead_agent" },
"checkpointer": { "path": ".../runtime/checkpointer/async_provider.py:make_checkpointer" }
```

一个反直觉的关键结论:**这四套里,LangGraph 官方提供的 store(专为长期记忆设计、支持向量索引)在 deer-flow 里几乎是摆设**;而 deer-flow 真正倚重的语义记忆,是它自己在 `agents/memory/` 里手写的、落成 JSON 文件的一套。为什么放着官方轮子不用、自己造一个?下文见分晓。

---

## 2. LangGraph 的 checkpointer 与 store:被"冷落"的官方记忆

### 2.1 checkpointer:情景记忆,交给 LangGraph 全权代管

checkpointer 是 deer-flow 的**情景记忆**——你关掉再回来,靠它恢复线程状态。deer-flow 几乎没做定制:`runtime/checkpointer/async_provider.py` 按配置三选一——`memory`→`InMemorySaver`(`:94`)、`sqlite`→`AsyncSqliteSaver`(`:101`)、`postgres`→`AsyncPostgresSaver`(`:111`),然后调 `.setup()` 让 LangGraph **自己建它那套表**(`checkpoints`、`checkpoint_blobs`、`checkpoint_writes`、`checkpoint_migrations`)。**默认是 `memory`——即进程内、不持久化。**

关键的一致性设计:LangGraph 的 checkpointer 表和 deer-flow 自研 ORM 的表**共处同一个 SQLite 文件**(`{sqlite_dir}/deerflow.db`,`config/database_config.py:83-90`),但**井水不犯河水**——`persistence/base.py:7` 明确写着"LangGraph's checkpointer tables are NOT managed by this Base",Alembic 迁移用 `include_object` 把它们排除在外(`migrations/_env_filters.py:24-34`)。两套表同库共存,靠"归谁管"划清边界。

### 2.2 store:支持向量索引,却被构造成纯 KV

LangGraph 的 `BaseStore` 本是为"跨线程长期记忆"设计的,支持嵌入向量索引。但在 deer-flow 里:

- 三种后端(`InMemoryStore` / `AsyncSqliteStore` / `AsyncPostgresStore`)**全部不带 `index=` 参数构造**(`runtime/store/async_provider.py:44-77`)——**即纯 KV,没有任何向量/嵌入索引。**
- 它**唯一的消费者**是 `MemoryThreadMetaStore`(`persistence/thread_meta/memory.py`),而且**只在 `database.backend=memory` 时**才启用——即当没有 SQL 会话工厂时,拿它临时存线程元数据,命名空间 `("threads",)`。一旦配了 SQL,线程元数据就走关系层的 `ThreadMetaRepository` 了,store 彻底靠边站。
- **它和 `agents/memory` 语义记忆毫无关联**(在 `agents/memory/` 里 grep `get_store`/`BaseStore` 无一命中)。

换句话说,**任何"LangGraph store = 长期记忆"的直觉,在 deer-flow 这里都是错的。** deer-flow 放弃官方 store 的向量记忆能力,选择自己手写一套零向量的文件记忆——这是一个刻意的架构取舍,理由在第 8 节会更清楚:deer-flow 想要的是**可读、可审计、零基础设施依赖**的记忆,而非"跑起来就得挂个向量库"的重方案。

> **文档落差**:`make_store` 的 docstring 说自己"镜像所配置的 checkpointer",但它检查的是**遗留的 `checkpointer:` 配置段**,而非现代的统一 `database:` 段(`runtime/store/async_provider.py:106-113`)。结果:只配 `database:` 时,checkpointer 会变成 SQLite/PG,而 store 却静默回退成 `InMemoryStore` 并打一条 WARNING。因为 store 本就只是 memory 模式的兜底,实际无害,但 docstring 是误导的。

---

## 3. 自研关系层:runs 与只增事件日志

第三套持久化是 deer-flow 自己的 **SQLAlchemy 2.0 异步 ORM**(`persistence/`),用 Alembic 做迁移,存的是"运行审计"——这是情景记忆的结构化元数据。9 张表里与记忆最相关的两张:

### 3.1 runs:每次运行的账本

`RunRow`(`persistence/run/model.py:14`)一行记录一次 run:`status`(`pending/running/success/error/timeout/interrupted`)、`model_name`、首条人类消息 / 末条 AI 消息,以及一整组 token 计费字段(`total_input_tokens`、`lead_agent_tokens`、`subagent_tokens`、`middleware_tokens`、`token_usage_by_model` JSON)。运行中用 `update_run_progress` 尽力写快照,完成时 `update_run_completion` 落最终状态与 token 总量。

### 3.2 run_events:严格递增序列号的只增日志

`RunEventRow`(`persistence/models/run_event.py:13`)是一份**只增(append-only)事件日志**,契约写在 `runtime/events/store/base.py:17-27`:"seq is strictly increasing within the same thread"。每条事件有 `event_type`、`category`(`message`/`trace`/`lifecycle`)、`content`、`seq`,并有 `UniqueConstraint(thread_id, seq)`。

`seq` 的分配是并发安全的:非 PG 用 `SELECT max(seq) FOR UPDATE`、Postgres 用 `pg_advisory_xact_lock` 保证同线程序列号无冲突(`runtime/events/store/db.py:92-112`)。事件日志有三种后端:`memory` / `db` / `jsonl`——`jsonl` 落成 `{base_dir}/threads/{thread_id}/runs/{run_id}.jsonl`,每行一个 JSON。

这套"严格递增序列号的只增事件日志"和 [opencode](./memory-in-opencode.md) V2 的 `session_message.seq`、[Codex](./memory-in-codex.md) 的 rollout JSONL 精神相通——都是"历史只追加、可回溯"的账本哲学。区别在 deer-flow 把它当成**运行审计**(哪次 run 发了什么事件),而非直接当会话真相源(那是 checkpointer 的活)。

> **默认非持久化的陷阱**:`database.backend` 和 `run_events.backend` **默认都是 `memory`**。更隐蔽的是——即便你把 `run_events.backend` 配成 `db`,只要 ORM 引擎没初始化(`database.backend=memory`),事件存储会**静默降级回 memory**(`runtime/events/store/__init__.py:12-15`),不报错却不持久化。

---

## 4. 语义记忆的数据模型:六段摘要 + 原子事实

现在进入本章的主角——`agents/memory/` 手写的语义长期记忆。先看它存什么。

存储抽象是 `MemoryStorage`(`storage.py:43`),默认实现 `FileMemoryStorage`(`storage.py:62`)——**纯 JSON 文件,一个作用域一个文件**,原子写入(先写临时文件再 `replace`,`storage.py:173-176`),`indent=2, ensure_ascii=False`。落盘位置默认按用户隔离:`{base_dir}/users/{user_id}/memory.json`,还可细到 per-agent。内存里带 mtime 失效缓存 + 线程锁。

记忆文件的 schema(`create_empty_memory()`,`storage.py:24-40`)是一个**混合体**:

```python
{
  "version": "1.0",
  "lastUpdated": "<iso-z>",
  "user":    { "workContext": {summary, updatedAt},     # 工作背景
               "personalContext": {...},                 # 个人背景
               "topOfMind": {...} },                      # 当前关注
  "history": { "recentMonths": {...},                    # 近几月
               "earlierContext": {...},                   # 更早
               "longTermBackground": {...} },             # 长期背景
  "facts": []                                             # 原子事实列表
}
```

两部分性质迥异:

- **六段滚动摘要**(user × 3 + history × 3):每段是一坨自由文本 summary,由 LLM **整段重写**维护。
- **原子事实列表**(`facts`):每条是结构化记录(`updater.py:658-671`):

```python
{"id": "fact_<8位hex>", "content": ..., "category": ..., "confidence": ...,
 "createdAt": <iso>, "source": <thread_id>, "sourceError": <可选,仅纠错时>}
```

所以 deer-flow 的记忆既是 mem0 式的"原子事实",又叠加了一层 MemPalace/Codex 式的"分段人类可读摘要"。**它是语义的**(内容经 LLM 蒸馏),**但组织形态是结构化 JSON**,而非向量。

---

## 5. 写入触发:遗忘的出口即记忆的入口

deer-flow 记忆最精巧的设计,是**它在什么时机抽取记忆**。答案不是"每轮结束",而是**上下文压缩的前一刻**。

deer-flow 用一个 `before_summarization` 钩子把记忆子系统挂在压缩流程上(`agents/lead_agent/agent.py:126-127`,仅当 `memory.enabled` 时注册):

```python
if resolved_app_config.memory.enabled:
    hooks.append(memory_flush_hook)
```

`memory_flush_hook`(`summarization_hook.py:12`)在**消息即将被摘要、旧对话即将从状态里丢弃之前**被触发:它过滤消息、检测纠错/强化信号,然后 `queue.add_nowait(...)` 把这段"要被遗忘的对话"推进后台队列。

这个设计的哲学非常干净:**工作记忆放不下、必须遗忘的那部分,恰恰是长期记忆该提炼的那部分。** 遗忘的出口,就是记忆的入口。对比 [Codex](./memory-in-codex.md)(记忆抽取在根会话启动时后台跑)、mem0(每轮 add 时触发),deer-flow 把抽取时机精准绑定在"信息即将流失的临界点",既不浪费 token 抽取还没沉淀的对话,也不遗漏任何将被压缩掉的内容。

**写入是完全后台异步的**,分两层:

- **去抖队列**(`queue.py`):用 `threading.Timer` 实现,去抖身份是 `(thread_id, user_id, agent_name)`。有意思的是——**生产路径其实跳过了去抖**:钩子调的是 `add_nowait()`→`_schedule_timer(0)`(立即在后台跑),配置里那个 `debounce_seconds=30` 只对没被使用的 `add()` 路径生效。所以实际是"立即但离线程",而非延迟 30 秒。
- **同步 LLM 甩到线程池**(`updater.py`):检测到事件循环在跑时,把阻塞的同步 `model.invoke()` 甩进 `ThreadPoolExecutor(max_workers=4)`。docstring 解释这是**故意用同步 HTTP 路径**,以绕开 langchain 全局缓存的异步 httpx 连接池的跨事件循环 bug(issue #2615)。

> **命名陷阱**:`aupdate_memory`/`update_memory` 名字带"async",但内部**故意跑同步** `model.invoke()` 于工作线程,不是真正的 asyncio LLM 调用。

---

## 6. 抽取管线:单阶段 LLM 与 ADD/UPDATE/DELETE

### 6.1 一次 LLM 调用搞定一切

deer-flow 的抽取是 **mem0 风格但单阶段**:一个 `MEMORY_UPDATE_PROMPT`(`prompt.py:22-138`)同时干四件事——读入当前记忆 JSON + 读入对话 + 抽取新事实 + 决定摘要更新与事实删除。模型在 `updater.py:518` 一次 `model.invoke()` 产出。抽取模型 `thinking_enabled=False`(`updater.py:390`)。

> **命名陷阱**:模块里还导出了一个 `FACT_EXTRACTION_PROMPT`(`prompt.py:142-167`),看着像 mem0 那种"抽取→巩固"两阶段的第一阶段,但它**在更新管线里从未被调用**,是死代码。deer-flow 只有一次 LLM 调用。别被它骗以为是两阶段。

### 6.2 输出 schema 与增删改

LLM 被要求产出一个四键 JSON(`prompt.py:104-119`,解析器强制这四个键,`updater.py:225`):

```json
{ "user": {...带 shouldUpdate 标志的三段摘要...},
  "history": {...三段...},
  "newFacts": [{"content","category","confidence"}],
  "factsToRemove": ["fact_id_1", ...] }
```

- **事实类别**(`prompt.py:77-83`):`preference | knowledge | context | behavior | goal | correction`。
- **置信度分档**(`prompt.py:84-87`):0.9-1.0 显式 / 0.7-0.8 强暗示 / 0.5-0.6 推断。

这是一套**粗粒度的 ADD/UPDATE/DELETE**:

- **UPDATE(摘要)**:每段带 `shouldUpdate` 标志,为真才**整段覆盖**(`updater.py:614-632`)——不是追加,而是让 LLM 自己把新旧整合后重写整段。
- **ADD(事实)**:置信度 `>= fact_confidence_threshold`(默认 **0.7**)才收(`updater.py:639`)。
- **DELETE(事实)**:`factsToRemove` 里的 id 直接删(`updater.py:634`),规则是"删除被新信息矛盾掉的旧事实"。
- **去重**:内容归一化(`content.strip().casefold()`)后比对,已存在的 key 跳过(`updater.py:366, 640`)——**纯字符串去重,不调 LLM 判重、不算相似度**(对比 mem0 的"哈希 + LLM 语义"两层去重)。

### 6.3 一条"拒绝不安全部分更新"的铁律

一个很讲究的健壮性设计(`updater.py:293-298`):如果响应里**同时**出现了 `factsToRemove` 且有 `newFacts` 条目解析失败/被丢弃,整个更新**直接抛 `JSONDecodeError` 中止**——**宁可不更新,也不基于半个解析结果去删事实。** 这避免了"因为一半 JSON 坏了,却把好端端的事实误删"的灾难。上限方面:`max_facts` 默认 **100**,超了按**置信度**保留 top-N(`updater.py:675-682`,注意是纯置信度、无 recency/LRU)。

---

## 7. 反射:让 Agent 记住自己的错误

> **先澄清一个大坑**:仓库里有个 `reflection/` 模块,名字很像"反思记忆",但它**跟记忆毫无关系**——它是一个通用的**点分路径导入解析器**(`resolve_class`/`resolve_variable`,`reflection/resolvers.py:25/73`),给"从字符串加载类/变量"用的。真正与记忆有关的"反射",在抽取 prompt 里。

deer-flow 记忆的"反射(reflection)"是一段**内嵌在 prompt 里的自省逻辑**(`MEMORY_UPDATE_PROMPT`,`prompt.py:39-46`),要求抽取 LLM 在提炼前先做三件事:

1. **错误/重试检测**→ 把根因 + 正确做法记为 `correction` 类事实。
2. **用户纠正检测**→ 记为 `correction`,并带 `sourceError`(仅当用户明确纠正时)。
3. **项目约束发现**→ 记为事实。

这套自省在运行时还有**正则信号检测**加持(`message_processing.py`):`detect_correction` 扫最近 6 条人类消息,匹配中英双语纠错模式(`不对`、`你理解错了`、`重试`、`换一种`…);`detect_reinforcement` 匹配强化模式(`完全正确`、`就是这个意思`…)。命中后灌进 prompt 的 `{correction_hint}`:纠错→强制类别 `correction`、置信度 ≥ 0.95;强化→类别 `preference`/`behavior`、置信度 ≥ 0.9(`updater.py:392-415`)。

于是"反射"= **Agent 对自己的错误与成功做内省,把它们写成高置信度的 `correction`/`preference` 事实**,形成一个轻量的自我改进回路。这与 [OpenClaw](./memory-in-openclaw.md) 的"做梦"整理、[MemPalace](./memory-in-mempalace.md) 的 Hebbian 强化精神相通——都想让记忆的去留和强度由"实际经历"动态塑造。而第 8 节会看到,这些 `correction` 事实在注入时享有**特权保护**。

---

## 8. 读取端:零向量的置信度排序注入

写得再好,读法才决定记忆有没有用。deer-flow 的读取端有一个鲜明特点:**没有查询、没有嵌入相似度、没有关键词检索——把整个记忆文件格式化后整份注入,事实仅按置信度排序、贪心塞进 token 预算。**

注入路径(`agents/lead_agent/prompt.py:594-626 _get_memory_context()`):加载 `get_memory_data()`→`format_memory_for_injection()`→包进 `<memory>...</memory>`。由 `DynamicContextMiddleware` 在首条用户消息前**一次性冻结注入**(每次会话一份快照)。

格式化与预算(`format_memory_for_injection`,`prompt.py:420-682`):

- `max_tokens` 默认 **2000**(`max_injection_tokens`,范围 100-8000)。
- 输出 `User Context:`(工作/个人/当前关注)+ `History:`(近期/更早/背景)+ 一个 `Facts:` 块。
- 事实行:`- [{category} | {confidence:.2f}] {content}`;纠错事实带 `sourceError` 时追加 `(avoid: {sourceError})`。
- **排序 = 置信度降序**,贪心选取,超预算即停。
- **保底类别特权**:`guaranteed_categories`(默认 **`["correction"]`**)从一份**独立的** `guaranteed_token_budget`(默认 **500**)里优先选取、置于最前——**保证"纠错记忆"永远不被挤掉。** 这正是第 7 节自省写下的 `correction` 事实的兑现:Agent 犯过的错,下次一定看得到。
- **结构感知截断**:溢出时 `Facts:` 块是**受保护后缀**,只裁前面的 user/history 前缀。
- token 计数默认 tiktoken `cl100k_base`,失败降级为 **CJK 感知的字符估算**(非 CJK 除 4、CJK 除 2),带 600 秒冷却重试。

这种"注入极省 + 保底特权 + 零向量"的读法,和 [MemPalace](./memory-in-mempalace.md) 的"唤醒预算 600-900 token"、[Codex](./memory-in-codex.md) 的"≤2500 token 摘要注入"是同一路数:**默认只给一份薄薄的、按重要性排好序的记忆,把上下文窗口留给任务本身。** 与 mem0 的向量语义召回相比,deer-flow 在读取端**连一次嵌入都没有**——这也解释了第 2 节它为什么放弃 LangGraph 官方 store:它压根不需要向量库。

---

## 9. 隐私与提示注入防御:OWASP LLM01

作为一个官方开源 Agent,deer-flow 把隐私与安全刻进了记忆管线,这是它区别于多数玩具实现的地方。

**门禁(纵深防御)**:`config.enabled=False` 时全链路熄火——入队、钩子、prompt 构造、注入各处都有独立 early-return。

**脱敏 / 隐私**:

- **上传文件绝不入记忆**。上传是会话级的,每次 finalize 都跑 `_strip_upload_mentions_from_memory`(`updater.py:343-363`),从所有摘要里剔除上传相关句子、丢弃上传类事实;prompt 也明令"Do NOT record file upload events in memory"。
- **框架消息排除**。`filter_messages_for_memory`(`message_processing.py:56-92`)丢弃带 `hide_from_ui` 的消息(TodoMiddleware、图片查看、以及**注入的 `__memory` 载荷本身**)——后者尤其关键:防止**注入的记忆被再次抽取**形成自我放大回环。
- 只保留用户输入 + **最终**助手回复(带 tool_calls 的中间消息丢弃);超长消息在 prompt 里截到 1000 字符。

**提示注入防御(OWASP LLM01)——最锋利的一笔**:

> 注入的记忆是作为 **`HumanMessage` 而非 `SystemMessage`** 递送的,这样"受用户影响的记忆内容永远拿不到系统权限"。

`dynamic_context_middleware.py:154` 的注释直言:注入的日期走 SystemMessage,而记忆走独立的 `{id}__memory` HumanMessage——**记忆内容可能被用户话术污染,所以刻意不给它系统级权威**。此外置信度值被 clamp 到 `[0,1]`、拒绝 NaN/inf,防止"被投毒的置信度"霸占排序。这种"把不可信数据严格降权"的意识,是本系列里对提示注入防御做得最显式的一个。

---

## 10. 工作记忆、声明式 SOUL.md、技能与子 Agent

最后补齐 deer-flow 的其余记忆层。

### 10.1 工作记忆:LangChain 摘要中间件

上下文压缩由 `DeerFlowSummarizationMiddleware`(`agents/middlewares/summarization_middleware.py:65`)负责——它**继承自 LangChain 内置的 `SummarizationMiddleware`**(不是 LangMem 的 `SummarizationNode`),在 `before_model` 钩子里判断是否超阈值,超了就用一个专用 LLM 把旧消息摘成一段、`RemoveMessage(REMOVE_ALL_MESSAGES)` 清掉再拼回保留的尾部消息。

配置(`config/summarization_config.py` 代码默认 vs `config.example.yaml` 出厂默认有出入,以出厂为准):出厂 `enabled: true`、**触发阈值 32000 tokens**、**保留最近 10 条消息**(代码默认 20)、`trim_tokens_to_summarize: 15564`。压缩前那一刻,正是第 5 节 `memory_flush_hook` 把对话抢救进长期记忆的时机——**工作记忆的压缩和长期记忆的写入,在这里咬合成一个动作。**

### 10.2 声明式记忆:是 SOUL.md,不是 AGENTS.md

这是一个**必须纠正的直觉**:deer-flow 仓库里有 `AGENTS.md`、`CLAUDE.md`,但——

> **它们是纯开发者文档,运行时从不加载。** 全仓 grep 确认没有任何 Python 代码读取/解析 `AGENTS.md`/`CLAUDE.md`;`backend/AGENTS.md:3` 自述"为 AI 编码助手(Claude Code、Codex 等)提供指引"。

deer-flow 真正的**运行时声明式记忆文件是 `SOUL.md`**(`config/agents_config.py:23`)。`load_agent_soul()` 读每个自定义 Agent 目录下的 `SOUL.md`——"定义 Agent 的人格、价值观与行为护栏,注入到 lead agent 的系统提示"。系统提示由 `SYSTEM_PROMPT_TEMPLATE`(`agents/lead_agent/prompt.py:364`)组装,`{soul}` 是其中一个槽位。

**所以:任何把 deer-flow 的 AGENTS.md 类比成 [Codex](./memory-in-codex.md)/[opencode](./memory-in-opencode.md) 那种"项目声明式记忆"的判断,都是错的。** deer-flow 的项目记忆叫 SOUL.md,且系统提示刻意做成**跨用户/会话静态**以复用前缀缓存——per-turn 的记忆和日期是另由中间件作为 `<system-reminder>` 注入的,不进静态系统提示。

### 10.3 技能:程序性记忆 + 渐进披露 + LLM 安全扫描

技能(`skills/`)是 deer-flow 的**程序性记忆**——"如何做某类事"的可复用知识,以 `SKILL.md`(带 YAML front-matter:`name`/`description`/可选 `allowed-tools`)承载。内置技能在仓库根 `skills/public/`(22 个,含 deep-research、ppt-generation 等),沙箱挂载于 `/mnt/skills`。

三个亮点:

- **渐进披露(progressive disclosure)**:系统提示里只列技能的 `name/description/location`,**从不塞完整正文**;模型匹配到需求时才用 `read_file` 读那份 SKILL.md(`prompt.py:629-663`)。斜杠激活(`/skill-name`)则把该技能全文注入当轮。
- **LLM 安全扫描(fail-closed)**:从 `.skill` ZIP 安装时,`security_scanner.py:70` 用审核模型把每个文本/脚本文件分类为 `allow|warn|block`,专门拦"提示注入、系统角色越权、提权、外泄、危险可执行代码";**扫描器不可用或输出无法解析→直接判 `block`**(`:105-109`)。ZIP 本身也做了防遍历、防符号链接、512 MiB 解压上限的加固。
- **`allowed-tools` 门禁**:技能可在其激活期间限制 Agent 可用的工具集。

### 10.4 子 Agent:干净的上下文隔离

子 Agent(`subagents/`)的记忆隔离很彻底:`_build_initial_state`(`executor.py:441-503`)只给子 Agent 一个 `[SystemMessage(自己的系统提示), HumanMessage(委派任务)]`——**父 Agent 的对话历史完全不传递**,跨界的只有 sandbox 与 thread_data。回流也极简:只把子 Agent 的**最后一条 AIMessage 文本**返回父 Agent,中间过程全部丢弃。父 Agent 则通过 `DurableContextMiddleware` 把委派记录(任务 + 结果摘要 + sha256,截断上限 2000 字符)存进 checkpoint 的 `delegations` 通道,每轮临时重注入并标注"已委派、勿重复委派、复用此结果"。这是一种"任务级的委派记忆",避免重复劳动。

---

## 11. 整体数据流与设计取舍

把所有环节串起来:

```
  声明式记忆(运行时)              程序性记忆                  (开发者文档,非运行时)
  SOUL.md → 系统提示 {soul}       SKILL.md (skills/public/)   AGENTS.md / CLAUDE.md
  (人格/价值观/护栏)              渐进披露 + LLM安全扫描(block)  ← 运行时从不加载
        │                              │ /mnt/skills
        ▼                              ▼
  ┌──────────────────── 一次 deer-flow 运行(LangGraph 图) ─────────────────────┐
  │  工作记忆: DeerFlowSummarizationMiddleware(继承 LangChain)                   │
  │    触发 32000 tokens → LLM 摘要 → RemoveMessage + 保留最近 10 条             │
  │                        │ before_summarization 钩子                          │
  │                        ▼  ★ 遗忘的出口 = 记忆的入口                          │
  │  memory_flush_hook: 过滤消息 + 正则检测纠错/强化 → queue.add_nowait()        │
  │  子Agent: 全新 [System+Human], 父历史不传; 只回流最后一条 AIMessage          │
  └───────────────────────────────┬────────────────────────────────────────────┘
                                   │ 后台异步(线程池, 同步HTTP绕#2615)
                                   ▼
   ┌───────────────────────────────────────────────────────────────────────┐
   │  语义记忆抽取 updater.py: 单阶段 LLM(MEMORY_UPDATE_PROMPT)                │
   │   读当前记忆 + 对话 → {六段摘要 shouldUpdate, newFacts, factsToRemove}    │
   │   ADD(conf≥0.7) / UPDATE(整段重写) / DELETE ; 字符串去重; max_facts=100   │
   │   反射: 错误→correction(≥0.95) 用户纠正→correction 强化→preference(≥0.9)  │
   │   隐私: 剥离上传/框架消息; 拒绝不安全部分更新                              │
   └───────────────────────────────┬───────────────────────────────────────────┘
                                    │ 落盘
                                    ▼
   ┌──────────────────────────────┐   ┌──────────────────────────────────────┐
   │  agents/memory JSON 文件       │   │  三套官方/自研持久化(与语义记忆无关)    │
   │  users/{uid}/memory.json      │   │  · LangGraph checkpointer(线程状态)    │
   │  六段摘要 + 原子事实           │   │  · LangGraph store(纯KV, 几乎摆设)     │
   │  ← 零向量/零嵌入               │   │  · SQLAlchemy: runs + 只增事件日志     │
   └──────────────┬───────────────┘   │    (seq严格递增, 默认memory不持久化)   │
                  │ 读取                └──────────────────────────────────────┘
                  ▼
   format_memory_for_injection: 置信度降序 + 贪心塞入 max 2000 token
   correction 类享独立 500 token 保底预算, 永不被挤掉; 结构感知截断保护 Facts:
                  │ DynamicContextMiddleware 一次性冻结注入
                  ▼
   <memory>...</memory> 作为 HumanMessage 注入 (★ OWASP LLM01: 不给系统权限)
```

拆解至此,deer-flow 的记忆机制有几处极具借鉴价值,也有几处需要警惕的"命名陷阱"与"架构分裂":

1. **建在 LangGraph 上,却弃用官方向量记忆、自造零向量文件记忆**。这是 deer-flow 最反直觉的取舍。LangGraph store 本可开启向量索引做长期记忆,deer-flow 偏不用,自己手写一套落成 JSON 文件的语义记忆。理由在读取端一目了然:它要的是**可读、可审计、零基础设施依赖**(不用挂向量库)的记忆,读取靠置信度排序 + token 预算而非嵌入召回。对一个要开源、要让人一眼看懂 `memory.json` 的官方项目,这个选择很务实。

2. **遗忘的出口即记忆的入口**。把长期记忆抽取挂在上下文压缩的 `before_summarization` 钩子上,让"工作记忆放不下、必须遗忘"的那部分精准地流入长期记忆。抽取时机绑定在信息流失的临界点——既不早抽(浪费在未沉淀的对话)、也不漏抽(不放过任何将被压缩的内容)。这是本系列里对"何时抽取"给出的最优雅答案。

3. **mem0 血统的单阶段 ADD/UPDATE/DELETE + 反射自省**。抽取是一次 LLM 调用完成读记忆/抽事实/定增删,并内嵌"错误/纠正/约束"三类自省,把 Agent 犯过的错写成高置信度 `correction` 事实;读取时这些 `correction` 又享有独立保底预算永不被挤掉——**自省与召回形成闭环**,Agent 真能"记住教训"。

4. **安全与隐私刻进管线**。上传文件绝不入记忆、框架消息排除以防自我放大回环、拒绝"不安全的部分更新"以防误删、技能安装 LLM 安全扫描且 fail-closed,尤其是**把注入记忆降级为 HumanMessage 以剥夺系统权限(OWASP LLM01)**——这是官方 Agent 该有的严谨,也是本系列里提示注入防御做得最显式的一个。

5. **四套持久化并存是最大的认知陷阱**。这一点必须逐条记住:
   - **记忆分散在四套子系统**:LangGraph checkpointer(线程状态)、LangGraph store(纯 KV,几乎没用)、自研 SQLAlchemy 关系层(runs/事件)、agents/memory(JSON 文件语义记忆)——谈"deer-flow 的记忆"必先分清是哪一套;
   - **LangGraph store ≠ 长期记忆**,真正的语义记忆是 `agents/memory/` 的 JSON 文件,与 store/checkpointer 毫无关联;
   - **`reflection/` 模块不是反思记忆**,是点分路径导入解析器;真正的"反射"内嵌在抽取 prompt 里;
   - **`FACT_EXTRACTION_PROMPT` 是死代码**,抽取只有一次 LLM 调用,不是两阶段;
   - **`AGENTS.md`/`CLAUDE.md` 是开发者文档、运行时不加载**,真正的声明式记忆文件是 `SOUL.md`;
   - **"去抖队列"的生产路径其实跳过了去抖**(`add_nowait` 立即执行);**"async" 更新内部跑同步 HTTP**;
   - **默认后端是 `memory`(不持久化)**,且 `run_events=db` 在无 ORM 时会静默降级回 memory。

   **任何基于 deer-flow 的技术判断,都必须以你实际使用的 commit 源码为准,而非文件名字面含义或 docs/ 描述。** 这也是本系列坚持"锁定 commit、回溯源码"的原因。

---

### 附:可直接查阅的源码入口

| 主题 | 关键文件(相对 deer-flow 仓库根,harness 在 `backend/packages/harness/deerflow/`) |
| --- | --- |
| 四套持久化接线 | `backend/app/gateway/deps.py`、`backend/langgraph.json` |
| LangGraph checkpointer | `.../runtime/checkpointer/async_provider.py`、`provider.py` |
| LangGraph store(纯 KV) | `.../runtime/store/async_provider.py`、`_sqlite_utils.py`、`.../persistence/thread_meta/memory.py` |
| 自研关系层 / 迁移 | `.../persistence/base.py`、`engine.py`、`bootstrap.py`、`persistence/run/model.py`、`persistence/models/run_event.py` |
| 只增事件日志 | `.../runtime/events/store/`(`base.py`/`db.py`/`jsonl.py`) |
| 语义记忆:数据模型 / 存储 | `.../agents/memory/storage.py`、`config/memory_config.py`、`config/paths.py` |
| 语义记忆:写入触发钩子 | `.../agents/memory/summarization_hook.py`、`agents/lead_agent/agent.py:126-127` |
| 语义记忆:去抖队列 | `.../agents/memory/queue.py` |
| 语义记忆:抽取/增删改/去重 | `.../agents/memory/updater.py` |
| 语义记忆:prompt / 反射 / 注入格式化 | `.../agents/memory/prompt.py` |
| 纠错/强化正则 + 消息过滤 | `.../agents/memory/message_processing.py` |
| 注入 + OWASP LLM01 权限分离 | `.../agents/lead_agent/prompt.py:594-626`、`.../agents/middlewares/dynamic_context_middleware.py` |
| 工作记忆:摘要中间件 | `.../agents/middlewares/summarization_middleware.py`、`config/summarization_config.py`、`config.example.yaml` |
| 声明式记忆:SOUL.md | `.../config/agents_config.py`、`.../agents/lead_agent/prompt.py:364` |
| 程序性记忆:技能 | `.../skills/`(`parser.py`/`types.py`/`installer.py`/`security_scanner.py`)、`skills/public/` |
| 子 Agent 隔离与回流 | `.../subagents/executor.py`、`.../agents/middlewares/delegation_ledger.py` |
| `reflection/`(无关的导入解析器) | `.../reflection/__init__.py`、`resolvers.py` |

> 本文为《Memory in Action》系列的一章,基于源码撰写,仅供技术学习与研究。所有代码常量、路径与行号以 deer-flow commit `4e6248f` 为准,后续版本可能演进。
