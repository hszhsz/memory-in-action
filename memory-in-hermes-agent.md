# Hermes Agent 的记忆机制（Memory in Hermes Agent）

> 本文基于 Nous Research 的 [`NousResearch/hermes-agent`](https://github.com/NousResearch/hermes-agent) 真实源码撰写，分析锁定 commit [`88d1d62`](https://github.com/NousResearch/hermes-agent/tree/88d1d6206f399c134d1f4c0b7db27733aaa3c50c)（2026-07-02）。文中所有实现细节、常量、文件路径与行号区间均可回溯到该版本源码,未做臆测;凡不确定处均已明确标注。

Hermes 是 Nous Research 打造的**自我改进型 AI Agent**,官方称它是"唯一内建学习闭环的 Agent——它从经验中创建技能、在使用中改进技能、periodically 提醒自己固化知识、检索自己过往的对话,并在多次会话之间构建一个愈发深入的'你是谁'的模型"。它的 slogan 一语中的:

> **The agent that grows with you.（与你一同成长的 Agent。)**

如果说上一篇拆解的 [OpenClaw](./memory-in-openclaw.md) 把记忆哲学落在"记忆即纯文本文件、无隐藏状态",那么 Hermes 的独特之处在于把记忆升维成一个**闭合的学习循环(closed learning loop)**:记忆不只是被读写的数据,更是一套会自我巩固、自我提炼、自我维护的活系统。

本文自底向上拆解 Hermes 的记忆架构:

1. [三层记忆模型:一张全景图](#1-三层记忆模型一张全景图)
2. [语义记忆:MEMORY.md 与 USER.md 的策展式存储](#2-语义记忆memorymd-与-usermd-的策展式存储)
3. [冻结快照与 memory 工具:如何写、如何注入](#3-冻结快照与-memory-工具如何写如何注入)
4. [安全边界:注入扫描与写入审批](#4-安全边界注入扫描与写入审批)
5. [情景记忆:基于 FTS5 的 session_search](#5-情景记忆基于-fts5-的-session_search)
6. [程序性记忆:技能即经验的固化](#6-程序性记忆技能即经验的固化)
7. [学习闭环:后台复盘、Nudge 与 Curator](#7-学习闭环后台复盘nudge-与-curator)
8. [工作记忆:上下文压缩与 on_pre_compress](#8-工作记忆上下文压缩与-on_pre_compress)
9. [可插拔的外部记忆后端](#9-可插拔的外部记忆后端)
10. [学习图谱:让成长可见](#10-学习图谱让成长可见)
11. [整体数据流与设计取舍](#11-整体数据流与设计取舍)

---

## 1. 三层记忆模型:一张全景图

Hermes 的记忆不是单一存储,而是借鉴认知科学的**多层记忆系统**,每一层解决不同问题:

| 记忆类型 | 载体 | 容量 | 特点 |
| --- | --- | --- | --- |
| **语义记忆 / Semantic**（事实与偏好） | `MEMORY.md` + `USER.md` | 严格上限(2200 / 1375 字符) | 策展式、始终在系统提示中 |
| **情景记忆 / Episodic**（发生过什么） | SQLite `state.db` + FTS5 | 无限（所有会话） | 按需检索、零 LLM 成本 |
| **程序性记忆 / Procedural**（怎么做） | `~/.hermes/skills/*/SKILL.md` | 由 Curator 维护 | 从经验自动创建、随用改进 |
| **工作记忆 / Working**（当前上下文） | 会话消息 + 压缩 | 受上下文窗口约束 | 超阈值时 LLM 摘要压缩 |

这四层由一个**学习闭环**串起来:后台复盘不断把工作记忆里的收获固化进语义/程序性记忆,Curator 定期修剪程序性记忆,而情景记忆则默默归档一切。下面逐层拆解。

---

## 2. 语义记忆:MEMORY.md 与 USER.md 的策展式存储

Hermes 的语义记忆由两个 Markdown 文件构成,都存放在 `<HERMES_HOME>/memories/`(默认 `~/.hermes/memories/`,解析逻辑在 `tools/memory_tool.py:55-57` 的 `get_memory_dir()`):

| 文件 | 用途 | 字符上限 |
| --- | --- | --- |
| **`MEMORY.md`** | Agent 的个人笔记:环境事实、约定、学到的经验 | **2200** 字符(约 800 token) |
| **`USER.md`** | 用户画像:偏好、沟通风格、期望 | **1375** 字符(约 500 token) |

这两个上限是 `MemoryStore.__init__` 的默认参数(`tools/memory_tool.py:130`,`memory_char_limit=2200, user_char_limit=1375`),可被配置 `memory.memory_char_limit` / `memory.user_char_limit` 覆盖。文件名映射硬编码在 `agent/learning_mutations.py:23`:`_MEMORY_FILES = {"memory": "MEMORY.md", "profile": "USER.md"}`。

### 2.1 为什么要"严格上限 + 不自动压缩"

这是 Hermes 一个鲜明的设计立场:**记忆是稀缺资源,必须保持聚焦**。字符上限不是软约束——当一次写入会超限时,`memory` 工具**返回错误**,而不是悄悄丢弃条目。Agent 必须在**同一轮**里自己腾地方(合并或删除条目)再重试。

条目在磁盘上以 `\n§\n`(section 符,`ENTRY_DELIMITER`,`tools/memory_tool.py:59`)分隔存储。加载时 `MemoryStore.load_from_disk()`(`:168`)读文件、按分隔符切分、用 `list(dict.fromkeys(...))` 去重、做安全清洗,然后**冻结快照**。

> **给读者的对照**:OpenClaw 的 `MEMORY.md` 超限时会保留磁盘原文、只截断注入副本;Hermes 则反过来——它不截断,而是把"腾地方"的责任交给模型,逼迫记忆始终保持高信噪比。两种哲学,各有取舍。

---

## 3. 冻结快照与 memory 工具:如何写、如何注入

### 3.1 冻结快照(Frozen Snapshot)模式

Hermes 的系统提示分三层:`stable`(可缓存)、`context`(cwd 相关)、`volatile`(每会话/每轮变化)。记忆属于 **volatile 层**(`agent/system_prompt.py:423-435`)。关键设计是:

> **系统提示中的记忆快照在会话开始时捕获一次,会话中途永不改变。**

`MemoryStore` 内部同时维护两份状态(`tools/memory_tool.py:117-122`):`_system_prompt_snapshot`(加载时冻结)与实时的 `memory_entries` / `user_entries`。`format_for_system_prompt(target)` 只返回**快照**,永不返回实时状态(`:615-626`)。

为什么这样设计?**为了保护 LLM 的前缀缓存(prefix cache)。** 如果记忆块随每次写入而变化,整个系统提示的 KV 缓存就会在每轮失效,推理成本骤增。因此中途写入会**立即持久化到磁盘**(不丢),但**要到下一次会话才出现在系统提示里**;当轮的工具响应则始终显示实时状态。甚至连时间戳都刻意精确到"日"而非"分",以保证系统提示全天字节稳定(`:448-454`)。

渲染格式(`_render_block()`,`:664-680`)是一个用 `═`×46 包裹、头部带用量百分比的块:

```
══════════════════════════════════════════════
MEMORY (your personal notes) [67% — 1,474/2,200 chars]
══════════════════════════════════════════════
User's project is a Rust web service at ~/code/myapi using Axum + SQLx
§
This machine runs Ubuntu 22.04, has Docker and Podman installed
§
User prefers concise responses, dislikes verbose explanations
```

头部把用量百分比直接暴露给模型,是为了让 Agent **自知容量**——接近上限时主动合并条目。

### 3.2 memory 工具:三个动作,没有 read

Agent 通过 `memory` 工具管理自己的记忆,动作集恰好是 **add / replace / remove**,**没有 read**(枚举在 `MEMORY_SCHEMA`,`:1087/:1113`;分发在 `memory_tool()`,`:1016-1026`)。为什么没有 read?因为记忆已经在系统提示里了——Agent"看得见"自己的记忆,无需再读。

几个精巧的工程点:

- **子串匹配**:`replace` / `remove` 用 `old_text in e`(`:408`、`:469`)定位条目,只需一个能唯一命中的短子串,不必给出完整原文。若命中多条则报错要求更具体(`:417-426`)。
- **超限即报错,当轮消化**:`add` / `replace` / 批量操作超限时返回 `success:False`,附上 `current_entries` 与 `usage`,并指示模型"在这一轮里"合并整理再重试(`:367-380` 等)。为防死循环,还有每轮失败上限 `_MAX_CONSOLIDATION_FAILURES_PER_TURN = 3`(`:128`),超过就返回终止性结果,不让失败写入拖垮整轮预算。
- **成功响应是终止性的**:成功响应刻意带 `note: "do not repeat it"` 并省略条目列表,避免模型反复 thrash。
- **批量原子性**:`operations=[...]` 批量形态会针对**最终**预算原子地应用(`apply_batch`,`:497-602`)。

### 3.3 去重

三层去重:(a) 加载期 `list(dict.fromkeys(...))`(`:192-193`);(b) `add()` 拒绝完全重复——`if content in entries: return "Entry already exists (no duplicate added)"`(`:360-361`);(c) 批量 `add` 幂等——`if content in working: continue`(`:543`)。

---

## 4. 安全边界:注入扫描与写入审批

记忆会被注入系统提示,因此它是 **prompt injection 的高危面**。Hermes 对此有两道防线。

### 4.1 威胁扫描(strict 作用域)

单一事实来源是 `tools/threat_patterns.py`。记忆使用**最广的 "strict" 作用域**,因为条目会跨会话持久驻留在冻结快照里:

- **写入时**:`_scan_memory_content()`(`:78-80`)→ `first_threat_message(content, scope="strict")`,在 `add`(`:343`)、`replace`(`:398`)、批量操作落盘前逐条调用。
- **加载时(纵深防御)**:`_sanitize_entries_for_snapshot()`(`:207-241`)对每条跑 `scan_for_threats(entry, scope="strict")`;命中者**仅在快照中**被替换为 `[BLOCKED: ... use memory(action=remove)...]`,实时状态仍保留原文供用户查看/删除。

strict 作用域覆盖:数据外泄(`send_to_url`、`context_exfil`)、SSH/持久化后门(`authorized_keys`、`~/.ssh`)、Agent 配置篡改、硬编码密钥;此外 `scan_for_threats` 单独检查**不可见/双向 unicode 字符**(内容先做 NFKC 归一化再正则)。

### 4.2 写入审批(write_approval)

默认情况下 Agent 自由写记忆(包括后台复盘的自动写入)。若担心"Agent 存了一个关于我的错误假设",可设 `memory.write_approval: true`,打开一个简单的开关门(`tools/write_approval.py`):

`evaluate_gate()`(`write_approval.py:253-312`)的决策矩阵:

- 门关(默认)→ `allow`,自由写入;
- **background_review 来源** → 一律**暂存**(后台线程无用户可问);
- 记忆 + **前台交互式 CLI** → 内联提示批准/拒绝;
- 记忆 + 网关/脚本/无监听者 → **暂存**。

暂存的写入通过 `/memory pending` 查看、`/memory approve|reject <id>` 处置。这正是对"后台自动写入把错误假设塞进画像"的解药:开启后每次保存——尤其是无提示的后台保存——都要等你点头。

---

## 5. 情景记忆:基于 FTS5 的 session_search

语义记忆容量有限(总共约 1300 token),但 Hermes 还能**检索自己所有的过往对话**——这就是情景记忆,由 `session_search` 工具提供。

- **存储位置**:`hermes_state.py:123` 的 `DEFAULT_DB_PATH = get_hermes_home() / "state.db"`(即 `~/.hermes/state.db`),所有 CLI 与消息平台会话都存进这个 SQLite。
- **FTS5 模式**(`hermes_state.py:802-855`,`SCHEMA_VERSION = 17`):两张 external-content 虚拟表,由 `messages` 表上的增删改触发器保持同步——`messages_fts`(默认 unicode61 分词)与 `messages_fts_trigram`(trigram 分词,支持 CJK/子串)。被索引的文本是 `content || ' ' || tool_name || ' ' || tool_calls`,连工具调用都可搜。
- **零 LLM**:`search_messages()`(`:4149`)直接 MATCH FTS5、按 BM25 `rank` 排序,返回**原始 DB 消息,不做 LLM 摘要、不截断**。文档给出的时延是 **~20ms 检索、~1ms 滚动**,且"免费——无 LLM 调用"。

`session_search` 的调用形态由参数推断(无 mode 参数,`session_search_tool.py:619`),源码实际实现**四种**:

1. **DISCOVERY**(`query=`):FTS5 → 按会话血缘去重 → Top-N 会话,每个带 `snippet`、命中点 ±5 消息窗口,以及 `bookend_start` / `bookend_end`(首尾各 3 条),并把 `cron` 来源降权到交互会话之下。
2. **SCROLL**(`session_id` + `around_message_id`):±window 切片,不走 FTS5。
3. **READ**(仅 `session_id`):导出整段会话(过大时首 20 + 尾 10)。
4. **BROWSE**(无参):按时间列出近期会话。

> **语义记忆 vs 情景记忆的分工**:前者是"始终该在上下文里的关键事实"(固定 token 成本);后者回答"我们上周是不是聊过 X?"(按需检索、成本几乎为零)。这是一种非常务实的成本-容量权衡。

---

## 6. 程序性记忆:技能即经验的固化

Hermes 最有野心的一层记忆是**程序性记忆**——它把"怎么做某件事"固化成**技能(Skill)**,兼容 [agentskills.io](https://agentskills.io) 开放标准。

- **存储**:技能位于 `~/.hermes/skills/<category>/<skill>/SKILL.md`(`skill_usage._skills_dir`,`:81-82`);内置/基础技能在仓库的 `<repo>/skills`。
- **自主创建**:复杂任务后,后台复盘 fork(或用户手动 `/learn`)会调用 `skill_manage(action="create")`,它进而调用 `mark_agent_created(name)`(`skill_manager_tool.py:1387-1389`),在 `.usage.json` 里写下 `created_by="agent"`。
- **agent-created vs base 的区分**:`is_agent_created`(`skill_usage.py:419-427`)= 既非内置(bundled)也非 Hub 安装。**只有 agent 创建的技能才归 Curator 管**(除非显式开 `curator.prune_builtins`);Hub/外部技能永不被触碰(`is_curation_eligible`,`:447-468`)。

`/learn`(`agent/learn_prompt.py`,`build_learn_prompt`,`:99`)是显式的同胞:它让**当前活跃**的 Agent 亲手写一个 `SKILL.md`,并强制"硬线"创作规范(描述 ≤60 字符、固定的章节顺序、作者署名 `Hermes`)。

这一层的意义在于:Hermes 不只是"记住事实",而是**把成功的做法沉淀为可复用的操作手册**,下次遇到同类任务时直接调用。这就是"从经验中学习"的具体落地。

---

## 7. 学习闭环:后台复盘、Nudge 与 Curator

这是 Hermes 的招牌机制,把前面几层记忆真正"盘活"。它由三个协作部件构成。

### 7.1 后台自我改进复盘(Background Review)

每轮回复交付后(未被打断、有 `final_response`),`turn_finalizer.py:474-478` 会 spawn 一次后台复盘(`agent/background_review.py`)。它由**计数器触发**:

- **记忆**:`_turns_since_memory` 每个用户轮 +1,达到 `_memory_nudge_interval`(默认 **10**)时置 `should_review_memory=True`。
- **技能**:`_iters_since_skill` 数本轮的工具调用迭代,达到 `_skill_nudge_interval`(默认 **10**,配置键 `creation_nudge_interval`)时触发。实际用到工具时计数器归零。

**模型**:复盘 fork 默认跑在**主模型**上(复用热的提示缓存);也可路由到更便宜的辅助模型(`auxiliary.background_review.{provider,model}`),路由时用压缩的历史摘要以控制冷写成本。

**机制**(`_run_review_in_thread`,`:572-869`):一个 fork 出来的 `AIAgent`(`max_iterations=16`、`skip_memory=True`)继承父级运行时与 `session_id`(缓存对齐),并被严格隔离(`_persist_disabled=True`、`compression_enabled=False`)。它的工具白名单被 `set_thread_tool_whitelist` 限制为 `["skills"]`(+ 启用时的 `"memory"`),并运行三种复盘提示之一(`_MEMORY_REVIEW_PROMPT` / `_SKILL_REVIEW_PROMPT` / `_COMBINED_REVIEW_PROMPT`)。写入都打上 `write_origin="background_review"` 标记,从而在 `write_approval` 打开时被暂存待审。

### 7.2 "Nudge" 到底是什么

一个反直觉但重要的发现:**Hermes 的 "nudge" 不是往主对话里注入提示文字**。`turn_context.py:98/290` 明确注释 "no nudge injection"。所谓 nudge,纯粹是**触发后台复盘 fork 的那个区间计数器**;持续施加"固化知识"压力的,是复盘 fork 内部的提示词。例如 `_SKILL_REVIEW_PROMPT` 写道:*"要 ACTIVE——大多数会话至少能产出一次技能更新……什么都不做的复盘是一次错失的学习机会。"*

### 7.3 Curator:技能收藏的园丁

技能会越攒越多,Hermes 用 **Curator**(`agent/curator.py`)定期修剪。它的不变式(文件头 `:1-20`)非常克制:

- **由不活跃触发,没有 cron 守护进程**:Agent 空闲、且距上次运行超过 `interval_hours` 时,`maybe_run_curator()` 才 fork 一个复盘 Agent。
- **只碰 agent 创建的技能**;
- **永不自动删除——只归档(可恢复)**;
- **Pinned(置顶)技能绕过一切自动转换**;
- 使用辅助客户端,绝不污染主会话的提示缓存。

默认阈值(`:56-64`):`DEFAULT_INTERVAL_HOURS = 24*7`(**7 天**)、`DEFAULT_MIN_IDLE_HOURS = 2`、`DEFAULT_STALE_AFTER_DAYS = 30`、`DEFAULT_ARCHIVE_AFTER_DAYS = 90`、`DEFAULT_CONSOLIDATE = False`(**合并默认关闭**,只跑确定性修剪)。

生命周期转换(`apply_automatic_transitions`,纯函数、不调 LLM):状态在 `active` / `stale` / `archived` 之间流转。以"最近活动时间"为锚:超过 90 天 → 归档;超过 30 天 → 标记 stale;重新使用则复活。**Pinned 永不被动**,从未用过的技能有一段宽限期保护。归档 `archive_skill`(`skill_usage.py:696-754`)把目录移到 `~/.hermes/skills/.archive/`,可用 `hermes curator restore` 恢复。

> **三者的协奏**:后台复盘负责"把当轮学到的写进记忆/技能";计数器 nudge 负责"何时该复盘";Curator 负责"长期把技能库保持健康"。这是一个真正意义上的**自我维护循环**。

---

## 8. 工作记忆:上下文压缩与 on_pre_compress

当对话变长,工作记忆(会话消息)会逼近上下文窗口。Hermes 的应对在 `agent/context_compressor.py` 与 `agent/conversation_compression.py::compress_context`(`:394`):

- **触发**:`should_compress()`(`:1096`)在提示 token ≥ `threshold_tokens` 时触发,阈值 = `context_length * threshold_percent`(默认 **0.50**)。
- **策略**:保护开头的若干轮 + 最后 N 轮,把中间可压缩段用 LLM 摘要成一条消息,其余保持原样。压缩会切分/轮转 SQLite 会话,并触发 `on_session_switch`。
- **`on_pre_compress` 钩子**(`conversation_compression.py:575-580`):在压缩器**丢弃消息之前**,先调用 `agent._memory_manager.on_pre_compress(messages)`,让当前的外部记忆后端有机会先提取并持久化洞见,再把返回文本折进摘要提示里。这保证了"要被压缩掉的上下文里的关键信息不会丢"。

> **注意**:仓库根目录还有一个 `trajectory_compressor.py`,那是一个**离线的训练/评测轨迹压缩工具**(依赖 HuggingFace tokenizer、OpenRouter 摘要),与上面这条**在线会话工作记忆**路径是两码事,不要混淆。

---

## 9. 可插拔的外部记忆后端

除了内建的两文件语义记忆,Hermes 还支持**可插拔的外部记忆 provider**,契约是 `agent/memory_provider.py` 的 `MemoryProvider` ABC。

- **发现**(`plugins/memory/__init__.py`):先扫内置 `plugins/memory/<name>/`,再扫用户安装的 `$HERMES_HOME/plugins/<name>/`;**同名时内置优先**。用户插件在合成命名空间 `_hermes_user_memory.<name>` 下加载以避免 `sys.modules` 冲突。
- **同时只有一个激活**:通过配置 `memory.provider` 选择。`MemoryManager.add_provider()`(`agent/memory_manager.py:374`)强制执行——内建总是接受,第二个非内建 provider 会被**拒绝并告警**(防止工具 schema 膨胀与后端冲突)。
- **委托生命周期**:`MemoryManager` 把 `prefetch_all`(轮前、内联)、`queue_prefetch_all`(轮后、后台)、`sync_all` 等委托给激活的 provider;sync/prefetch 跑在**单 worker 的守护 `ThreadPoolExecutor`** 上,确保卡住的 provider 永不阻塞主轮次(历史上曾有 provider 内联阻塞约 298 秒)。
- **完整钩子集**:`system_prompt_block`、`prefetch` / `queue_prefetch`、`sync_turn`、`on_turn_start`、`on_session_end`、`on_session_switch`、`on_pre_compress`、`on_memory_write`(镜像内建记忆写入)、`initialize_all` / `shutdown_all` 等。召回的上下文被 `<memory-context>` 围栏包裹并附系统说明,且从流式输出中剥除。

**Provider 生态一览**(源自各 `plugin.yaml` / README):

| Provider | 一句话定位 |
| --- | --- |
| **holographic**(内建,无依赖) | 本地 SQLite 事实库 + FTS5,带信任评分、实体消解,以及 **HRR 组合式检索**(见下) |
| **honcho** | AI 原生的跨会话**用户建模**,多轮辩证式(dialectic)推理、持久化结论 |
| **mem0** | 服务端 LLM 事实抽取、语义检索 + 重排、自动去重 |
| **byterover** | 持久化的分层知识树,经 `brv` CLI 做分层检索 |
| **hindsight** | 长期记忆 + 知识图谱、实体消解、多策略检索 |
| **openviking** | 火山引擎/字节的上下文数据库,文件系统式知识层级 |
| **retaindb** | 云 API,向量 + BM25 + 重排混合,7 种记忆类型 |
| **supermemory** | 语义长期记忆、画像召回,每会话一次全量摄入 |

其中 **holographic** 最值得一提:HRR 即 **Holographic Reduced Representations**(全息缩减表示,Plate 1995)。它用固定宽度的**相位向量**(角度在 `[0,2π)`,由 SHA-256 确定性生成)表示概念,以**循环卷积**做 `bind`(绑定)、**循环相关**做 `unbind`(解绑)、**叠加**做 `bundle`(捆绑)——一种把符号结构编码进定长向量的经典向量符号架构。`retrieval.py` 把 FTS5(0.4)+ Jaccard(0.3)+ HRR(0.3)混合打分,NumPy 缺失时自动把 HRR 权重降为 0。

---

## 10. 学习图谱:让成长可见

Hermes 还把"学到了什么"可视化成一张图(`agent/learning_graph.py`,`build_learning_graph`,`:254`),用于桌面端展示:

- **节点类型**:`kind="skill"` 与 `kind="memory"`。技能只取"学来的"(`source != "base"` 且 `created_by=="agent"` 或 `use_count>0`);记忆节点来自 `MEMORY.md`(`source="memory"`)与 `USER.md`(`source="profile"`),按 `\n§\n` 切分,每块一个节点 `memory:<source>:<idx>`。
- **边**:(a) 技能↔技能来自声明的 `related_skills`;(b) **记忆→技能边由词法重叠推导**(`_memory_skill_edges`):对记忆卡片文本分词(≥3 字符的 token),对每个技能打分——技能名出现在文本里 `+6`,再加与技能名 token 的交集大小,保留得分最高的 **top-4** 技能。

这张图能回答"我记住的东西,和我学会的哪些技能相关联?"——把抽象的"成长"变成可看、可点、可编辑的结构。

---

## 11. 整体数据流与设计取舍

把所有环节串起来,Hermes 的记忆是一个**多层 + 闭环**的活系统:

```
                         ┌─────────────────────────────┐
                         │        用户对话(一轮)         │
                         └─────────────────────────────┘
             (轮前 prefetch)│                     │(轮后)
                            ▼                     ▼
              ┌──────────────────────┐   ┌──────────────────────────┐
              │  语义记忆(冻结快照)    │   │  计数器 nudge(每 10 轮)   │
              │  MEMORY.md + USER.md  │   │  触发后台复盘 fork         │
              │  注入 volatile 层      │   └──────────────────────────┘
              └──────────────────────┘                 │
                    ▲   ▲                               ▼
        memory 工具  │   │(下次会话刷新)    ┌──────────────────────────────┐
        add/replace/ │   └─────────────────│  后台自我改进复盘              │
        remove       │                     │  ·写 MEMORY/USER(威胁扫描)    │
                     │                     │  ·创建/改进 SKILL(程序性记忆)  │
                     │                     └──────────────────────────────┘
                     │                                  │
    ┌────────────────┴───────┐          ┌──────────────▼───────────────┐
    │  情景记忆                │          │  程序性记忆                    │
    │  state.db + FTS5        │          │  ~/.hermes/skills/*/SKILL.md  │
    │  session_search(零 LLM) │          │      ▲                        │
    └─────────────────────────┘          │      │(7 天/不活跃触发)        │
                     ▲                    │  ┌───┴──────────────┐         │
       (所有会话自动归档)                   │  │  Curator          │         │
                     │                    │  │ stale 30d/归档 90d │         │
              ┌──────┴───────┐            │  │ 只归档不删除·置顶豁免│         │
              │  工作记忆      │            │  └──────────────────┘         │
              │  消息 + 压缩   │────────────┴───────────────────────────────┘
              │  on_pre_compress→外部 provider 抢救洞见
              └──────────────┘
        旁路:可插拔外部后端(holographic/honcho/mem0/… 同时仅一个激活)
```

拆解至此,Hermes 的记忆机制有几处极具借鉴价值:

1. **多层记忆各司其职**。语义(该常驻的事实)、情景(可检索的历史)、程序性(可复用的做法)、工作(当前上下文)四层分工清晰,而不是把一切都塞进一个向量库。这更贴近人类认知结构。

2. **闭合学习循环是灵魂**。后台复盘 + 计数器 nudge + Curator 三件套,让记忆**自我巩固、自我提炼、自我维护**——这才是 "grows with you" 的技术内核,也是 Hermes 与普通"带记忆的 Agent"的根本区别。

3. **前缀缓存驱动的冻结快照**。为了不破坏 KV 缓存,记忆中途只落盘不改提示、时间戳精确到天——把"性能"作为一等公民纳入记忆设计,是很多方案忽视的工程细节。

4. **稀缺即聚焦**。严格字符上限 + 超限报错 + 当轮消化,逼迫模型主动策展记忆,维持高信噪比;而非无限膨胀后再靠检索去噪。

5. **安全被认真对待**。记忆是注入面,strict 作用域的双向扫描 + NFKC 归一化 + 不可见 unicode 检测 + 可选写入审批,构成一套完整的记忆安全边界。

6. **零 LLM 的情景检索**。用 SQLite FTS5 提供"无限容量、~20ms、免费"的过往对话检索,把昂贵的语义容量留给真正该常驻的事实——一个漂亮的成本工程决策。

7. **克制的可插拔**。外部后端同时仅一个激活、后台线程隔离、卡死不阻塞主轮次——扩展性与稳健性兼顾。

---

### 附:可直接查阅的源码入口

| 主题 | 关键文件(相对 hermes-agent 仓库根) |
| --- | --- |
| 记忆用户文档 | `website/docs/user-guide/features/memory.md` |
| 语义记忆存储 + memory 工具 | `tools/memory_tool.py`、`agent/learning_mutations.py` |
| 系统提示注入(volatile 层) | `agent/system_prompt.py` |
| 威胁扫描 / 写入审批 | `tools/threat_patterns.py`、`tools/write_approval.py` |
| 情景记忆 / FTS5 | `hermes_state.py`、`tools/session_search_tool.py` |
| 程序性记忆(技能) | `tools/skill_usage.py`、`skill_manager_tool.py`、`agent/learn_prompt.py` |
| 学习闭环 | `agent/background_review.py`、`agent/curator.py`、`agent/turn_finalizer.py` |
| 工作记忆压缩 | `agent/context_compressor.py`、`agent/conversation_compression.py` |
| 外部记忆后端抽象 | `agent/memory_manager.py`、`agent/memory_provider.py`、`plugins/memory/__init__.py` |
| Provider 实现 | `plugins/memory/{holographic,honcho,mem0,byterover,hindsight,openviking,retaindb,supermemory}/` |
| 学习图谱 | `agent/learning_graph.py`、`agent/learning_graph_render.py` |

> 本文为《Memory in Action》系列的一章,基于源码撰写,仅供技术学习与研究。所有代码常量、路径与行号以 Hermes Agent commit `88d1d62` 为准,后续版本可能演进。
