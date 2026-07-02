# opencode 的记忆机制（Memory in opencode）

> 本文基于 [`opencode`](https://github.com/anomalyco/opencode) 的真实源码撰写，分析锁定 commit [`403164a`](https://github.com/anomalyco/opencode/tree/403164a6ac638549d5d9502a7fe8eccb8a79355d)（2026-07-02）。文中所有实现细节、常量、文件路径与行号区间均可回溯到该版本源码，未做臆测；凡代码与文档不一致处均已明确标注。

前五篇拆解的 [OpenClaw](./memory-in-openclaw.md)、[Hermes](./memory-in-hermes-agent.md)、[mem0](./memory-in-mem0.md)、[MemPalace](./memory-in-mempalace.md) 与 [Codex](./memory-in-codex.md) 里，有的把"记忆"做成了独立的语义提炼层（mem0 抽事实、MemPalace 逐字存、Codex 两阶段 LLM 抽取），有的把它揉进 Agent 主体（OpenClaw 的"做梦"、Hermes 的冻结快照）。到了 opencode，我们撞上一个**反直觉的结论**：

> **opencode 根本没有"专门的长期记忆子系统"。**

我在源码里对 `memory`、`embedding`、`vector`、`recall`、`cosine` 做了穷举式 grep（`core/src` + `opencode/src`），命中的全是假阳性——一句注释 `Test or embedding seam`、GitHub Copilot 的 `file-search` 工具类型定义而已。**没有向量库，没有嵌入模型，没有"从历史里提炼事实"的抽取管线。** 这与 Codex 的 `memories` crate、mem0 的 `ADDITIVE_EXTRACTION_PROMPT` 形成最鲜明的对照。

那 opencode 到底靠什么"记住东西"?答案是它把记忆需求**拆解回三种更朴素、更工程化的机制**，全部围绕"对话本身"和"文件系统"打转：

| opencode 的"记忆" | 实质 | 对标前作的哪一层 |
| --- | --- | --- |
| **声明式指令记忆** | `AGENTS.md` / `CLAUDE.md` / `CONTEXT.md` 分层加载 | Codex 的 AGENTS.md 层 |
| **情景记忆(会话历史)** | SQLite 持久化的消息/事件流 | Codex 的 rollout,但用 SQL 而非 JSONL |
| **工作记忆(上下文压缩)** | 两套并存的 compaction 栈 | Codex 的 compact 四策略 |
| **程序性记忆(技能)** | `SKILL.md` 文件 | (前作少见) |
| **快照记忆(检查点)** | git shadow repo 做 checkpoint/undo | (前作少见) |
| **待办工作记忆** | SQLite 里的 todo 表 | MemPalace 的 diary/checkpoint |

而 opencode 最值得细看的地方，不在"有没有长期记忆"，而在它**同时存在两套并行的会话架构**——经典的 `opencode/src/session/`(V1,消息/分片表)与更新的 `core/src/session/`(V2,事件溯源)——两套对"如何持久化对话、如何压缩上下文"给出了**不同的答案**，正处在新旧交替的迁移期。这才是本章的主线。

本文自底向上拆解：

1. [双架构总览:V1 经典 vs V2 事件溯源](#1-双架构总览v1-经典-vs-v2-事件溯源)
2. [持久化的真相:从 JSON 文件到 SQLite](#2-持久化的真相从-json-文件到-sqlite)
3. [V2 事件溯源:projector 与 durable 序列号](#3-v2-事件溯源projector-与-durable-序列号)
4. [上下文纪元:可复位的系统上下文基线](#4-上下文纪元可复位的系统上下文基线)
5. [情景记忆的读取:历史开窗](#5-情景记忆的读取历史开窗)
6. [工作记忆:两套并存的上下文压缩栈](#6-工作记忆两套并存的上下文压缩栈)
7. [声明式记忆:AGENTS.md 的两个加载器](#7-声明式记忆agentsmd-的两个加载器)
8. [程序性记忆与快照:技能与 git 检查点](#8-程序性记忆与快照技能与-git-检查点)
9. [系统上下文、待办与提醒:合成出来的记忆](#9-系统上下文待办与提醒合成出来的记忆)
10. [整体数据流与设计取舍](#10-整体数据流与设计取舍)

---

## 1. 双架构总览:V1 经典 vs V2 事件溯源

opencode 是一个 TypeScript / Bun 项目，重度使用 [Effect](https://effect.website/) 函数式框架。它的会话逻辑**同时存在两套实现**，是本章一切理解的起点：

| | **V1 经典** | **V2 core** |
| --- | --- | --- |
| 目录 | `packages/opencode/src/session/` | `packages/core/src/session/` |
| 模型 | 消息 + 分片(message/part)直接落库 | **事件溯源**(events → SQL state) |
| Schema | `packages/schema/src/v1/session.ts`(12 种 part) | `packages/schema/src/session-message.ts` |
| 压缩 | 隐藏 `compaction` 子 Agent + `filterCompacted` 重排 | 内联 LLM + 事件(`SessionEvent.Compaction`) |
| 指令加载 | `session/instruction.ts`(AGENTS.md/CLAUDE.md/CONTEXT.md) | `core/src/instruction-context.ts`(仅 AGENTS.md) |
| 状态 | 现役 | 新架构，正在接管 |

两套都往**同一个 SQLite 库**(`opencode.db`)写，但表结构与写入路径不同。V2 的 `core/src/session/message-updater.ts` 甚至用**一个 reducer + 两个适配器**(内存 immer 版 + DB 版)来统一"如何把一条消息变更应用到状态"——这是典型的事件溯源写法。

理解这一点极其重要：**任何关于"opencode 怎么存对话"的判断，都必须先问"你说的是 V1 还是 V2"。** 两者的持久化粒度、压缩哲学、可回溯性都不一样。下文凡涉及分歧处都会显式标注。

---

## 2. 持久化的真相:从 JSON 文件到 SQLite

### 2.1 "文件存储"是个历史误名

很多关于 opencode 的旧描述会说它"用 JSON 文件存会话"。**在本文锁定的 commit 上，这已经是过时的说法。** `packages/opencode/src/storage/storage.ts`(327 行)如今只是一个**通用的 JSON 键值存储**，落在 `~/.local/share/opencode/storage/` 下：

```ts
// storage.ts:64
return path.join(dir, ...key) + ".json"
```

它存的是杂项状态(项目元信息、session_diff 之类)，**已经不是消息的主存储**。真正的会话数据早已迁进 SQLite。

更能说明问题的是 `storage.ts` 里那一串**迁移函数**(`Storage.migration.1`、`Storage.migration.2`，`:82`、`:182`)：它们把老版本散落在 `storage/session/message/*/*.json`、`storage/session/part/.../*.json` 里的历史数据，**逐个搬进新的 SQL 表**。迁移进度用一个数字标记文件 `storage/migration` 记录。换句话说，源码里保留的这些 glob 扫描代码，本身就是"文件存储→SQLite"这次架构迁移的**活化石**。

### 2.2 SQLite/Drizzle:真正的会话主库

会话主库是 `Global.Path.data/opencode.db`，用 [Drizzle ORM](https://orm.drizzle.team/) 定义 schema(`packages/core/src/session/sql.ts`，176 行)。打开参数是很规矩的生产配置：**WAL 模式、`synchronous=NORMAL`、`busy_timeout=5000`、`foreign_keys=ON`**。

核心表：

| 表 | 作用 |
| --- | --- |
| `session` | 会话元信息(标题、目录、模型、token 计费、`revert` 状态、`time_compacting` 等) |
| `message` / `part` | V1 消息与分片 |
| `session_message` | **V2** 消息表，靠 `uniqueIndex session_message_session_seq_idx (session_id, seq)` 保证每会话内序列号唯一 |
| `session_input` | 输入队列,`delivery` 分 `"steer"`(插队) / `"queue"`(排队) |
| `session_context_epoch` | **上下文纪元**(见第 4 节),每会话一行 |
| `todo` | 待办事项(工作记忆,见第 9 节) |

这与 Codex "**JSONL 是不可变真相 + SQLite 只是可重建索引**"的双层哲学**恰好相反**：opencode 把 SQLite 直接当成**权威存储**，没有一份并行的纯文本真相源。好处是查询/事务/外键约束都由数据库原生保证；代价是失去了 Codex 那种"删掉索引重建、历史一行不丢"的纯文本透明性——在 opencode 里，SQLite 坏了就是真坏了。

---

## 3. V2 事件溯源:projector 与 durable 序列号

V2 架构的核心是 `packages/core/src/session/projector.ts`(458 行)——一个把**事件流投影成 SQL 状态**的投影器(projector)。这是 opencode 记忆机制里最"现代"的一块。

### 3.1 一切状态都是事件的投影

在事件溯源模型里，你不直接改状态，而是**追加事件**，再由 projector 把事件"重放"成当前状态。opencode 的 `projector.ts` 就是这个重放器：`SessionEvent.*` 进来，`session_message` / `session_input` / `session_context_epoch` 等表被相应更新。

### 3.2 durable 序列号:事件溯源的地基

projector 里有一条被**反复强调、且直接 `Effect.die`(致命崩溃)** 的铁律——每个需要持久化的事件**必须带聚合序列号**：

```ts
// projector.ts:117 / :194 / :352 / :366 (共 4 处)
if (event.durable === undefined) return Effect.die("Durable Session event is missing aggregate sequence")
```

这个 `event.durable.seq` 就是写进 `session_message.seq` 的单调递增序号。**它是整个事件溯源可回溯性的地基**：序列号保证事件全序、保证 `(session_id, seq)` 唯一、保证重放确定性。opencode 宁可让程序当场崩溃，也不允许一个没有序列号的持久事件溜进来——这是对"历史必须可靠"的强硬承诺。

### 3.3 revert:按序列号"砍掉"历史

事件溯源的一个天然红利是**干净的回退**。`RevertEvent.Committed` 的处理(`projector.ts`)不是去"反向操作"，而是：

- 删除 `session_message` / `session_input` 里所有 `seq > 边界` 的行;
- 同时 `SessionContextEpoch.reset` 复位上下文纪元(见下节)。

也就是说，回退 = **砍掉某个序列号之后的一切事件**。这比 V1 那种在消息里塞标记、检索时再过滤的做法干净得多，也和 Codex rollout "只追加、从不改写"形成有趣对比：Codex 靠 `Compacted` 标记 + 读时过滤模拟遗忘，opencode V2 则**真的把行删掉**——因为它的真相就是 SQL 状态，而非追加日志。

---

## 4. 上下文纪元:可复位的系统上下文基线

`session_context_epoch` 表和 `packages/core/src/session/context-epoch.ts`(174 行)是 opencode 一个**很少见于其他 Agent** 的设计，值得单独一节。

### 4.1 什么是"纪元"

每个会话在 `session_context_epoch` 里有**恰好一行**，记录三个关键字段：`baseline`(当前系统上下文基线)、`snapshot`、`baseline_seq`(基线对应的序列号)。它解决的问题是：**系统上下文(system prompt、AGENTS.md 指令、环境信息等)不是一成不变的**——AGENTS.md 会被改、模型会被换、环境会变。opencode 需要一个稳定的"基线"来界定"从哪条消息之后，系统上下文才算数"。

`prepare()`(每轮开始时调用)负责 `initialize / reconcile / replace` 三种动作：会话首次启动时初始化基线；系统上下文没变时对账保持；变了就替换基线并推进 `baseline_seq`。

### 4.2 纪元如何驱动历史开窗

`baseline_seq` 的真正用途在检索侧(第 5 节)：历史开窗时，**系统类消息只保留 `seq > baseline_seq` 的**——即只认"当前纪元之后"的系统上下文，旧纪元的系统消息自动失效。这和 Codex 的 AGENTS.md "变更即重注入 + REPLACEMENT_NOTICE(新指令覆盖旧指令)"是**同一个问题的两种解法**：Codex 靠在 prompt 里贴一句"替换声明"，opencode 靠序列号基线让旧系统消息**在检索层就查不出来**。opencode 的做法更结构化——遗忘旧系统上下文这件事，被下沉进了 SQL 查询条件。

---

## 5. 情景记忆的读取:历史开窗

存下来的对话，怎么读回给模型?逻辑在 `packages/core/src/session/history.ts`(101 行)。它做的是**历史开窗(windowing)**——从 `session_message` 里选出"这一轮该喂给模型的那些消息"。

核心 SQL 条件(`history.ts:38-46`)可以读成一句话：

> **保留 `seq >= compaction.seq` 的消息(即压缩点之后的所有内容);外加,即便在压缩点之前,只要是 `seq > baseline_seq` 的系统消息也保留。**

拆开看：

- `compaction.seq` 是上一次压缩的序列号——压缩之后的对话全部保留(这是"最近的活跃上下文")。
- 压缩点**之前**的普通消息被摘要替代、不再选出(已被压缩"遗忘")。
- 但**当前纪元的系统消息**(`seq > baselineSeq`)是例外,无论在不在压缩点之前都保留——因为系统上下文必须始终在场。

这套"压缩序列号 + 纪元序列号"的双阈值开窗，把"工作记忆(压缩)"和"系统上下文(纪元)"两条正交的裁剪逻辑，优雅地合并进了一条查询。对比 Codex 的 `build_compacted_history_with_limit`(保留最近用户消息 + 逐个丢最老 item)，opencode 的开窗是**声明式的 SQL 谓词**,而非命令式的逐项裁剪——这是 SQLite 权威存储带来的直接红利。

---

## 6. 工作记忆:两套并存的上下文压缩栈

和 Codex 一样，opencode 也要解决"对话撑爆上下文窗口怎么办"。但它**有两套并存、且设计分歧明显**的 compaction 实现——这正是双架构分裂在工作记忆层的投影。

### 6.1 触发判据:溢出检测

先看什么时候触发。经典侧 `packages/opencode/src/session/overflow.ts`(34 行)只有一条规则：

```
COMPACTION_BUFFER = 20_000
isOverflow  当  count >= usable   (usable = 输入上限 - 预留)
```

token 估算是全项目统一的粗糙口径——`packages/core/src/util/token.ts` 的 `CHARS_PER_TOKEN = 4`,`estimate = round(length/4)`。没有精确分词,纯按字符数除以 4。这与 mem0/Codex 依赖模型返回的真实 token 用量不同,opencode 在压缩触发上走的是**廉价近似**路线。

### 6.2 经典栈:隐藏子 Agent + filterCompacted 重排

经典压缩在 `packages/opencode/src/session/compaction.ts`(562 行)，一堆常量道出了它的裁剪哲学：

| 常量 | 值 | 含义 |
| --- | --- | --- |
| `PRUNE_MINIMUM` | 20_000 | 低于此不修剪 |
| `PRUNE_PROTECT` | 40_000 | 从尾部往回保护这么多 token 的工具输出 |
| `TOOL_OUTPUT_MAX_CHARS` | 2_000 | 单条工具输出截断上限 |
| `PRUNE_PROTECTED_TOOLS` | `["skill"]` | **skill 工具的输出永不被修剪** |
| `DEFAULT_TAIL_TURNS` | 2 | 默认保留最近 2 轮 |
| `MIN/MAX_PRESERVE_RECENT_TOKENS` | 2_000 / 8_000 | 保留最近内容的 token 区间(取 `usable*0.25` 夹在两者间) |

经典栈最有特色的两点：

1. **专门的隐藏压缩 Agent**。`agent.ts:219-233` 定义了一个 `compaction` agent，`hidden:true`、权限 `"*":"deny"`——它是一个**对用户不可见、被彻底锁死权限**的子 Agent，唯一职责就是生成摘要。这与 Codex Phase 2 那个"断网沙箱子 Agent"精神相通:**用一个被严格约束的子 Agent 去做记忆改写这件危险活**。
2. **`filterCompacted` 重排**。`message-v2.ts:521-572` 在压缩后把消息流**重新编排**成固定顺序：`[compaction-user(压缩触发消息), summary(摘要), ...保留的尾部..., continue-user(继续消息)]`。摘要被塞在恰当位置，让模型读起来像一段连贯的历史。

`PRUNE_PROTECTED_TOOLS=["skill"]` 这个细节很能说明设计意图:**技能调用的输出被当成不可丢弃的高价值信息**,哪怕压缩也要留着——呼应了下一节"技能=程序性记忆"的定位。

### 6.3 core 栈:内联 LLM + 事件溯源

V2 侧 `packages/core/src/session/compaction.ts`(246 行)则完全是另一套：

| 常量 | 值 |
| --- | --- |
| `DEFAULT_BUFFER` | 20_000 |
| `DEFAULT_KEEP_TOKENS` | 8_000 |
| `TOOL_OUTPUT_MAX_CHARS` | 2_000 |
| `SUMMARY_OUTPUT_TOKENS` | 4_096 |

区别在机制：

- **没有独立 Agent**,直接**内联**调一次 LLM 生成摘要(`summaryOutput = min(output, SUMMARY_OUTPUT_TOKENS)`)。
- **事件溯源**:压缩表现为 `SessionEvent.Compaction.Started` / `Ended` 两个事件,而非直接改消息——压缩本身也进了那条不可篡改的事件流。
- 之后靠第 5 节的 `compaction.seq` 历史开窗自动生效,无需 `filterCompacted` 那样的显式重排。

### 6.4 共享的摘要模板

两套栈虽然机制不同,但 core 栈的**摘要模板 `SUMMARY_TEMPLATE`**(`core/src/session/compaction.ts:16-51`)非常值得一看——它强制 LLM 输出一个**固定 Markdown 结构**:

```
## Goal
## Constraints & Preferences
## Progress (### Done / ### In Progress / ### Blocked)
## Key Decisions
## Next Steps
## Critical Context
## Relevant Files
```

并附硬规则:"每个 section 都保留(哪怕为空)"、"用简短 bullet 不要长段落"、"精确保留文件路径/命令/错误串/标识符"、"不要提及压缩过程本身"。这比 Codex local 策略那个自由格式的 `SUMMARY_PREFIX + 摘要` **结构化得多**——opencode 把"一次压缩该记住哪几类东西"(目标/约束/进度/决策/下一步/关键上下文/相关文件)显式编码进了模板。这其实是一种**对"什么值得被记住"的声明式定义**,是本系列里最工整的压缩摘要 schema。

> **命名陷阱**:`core/src/session/summary.ts` 这个文件**不是**对话摘要——它生成的是 **git diff 摘要**(改了哪些文件、加删多少行),对应 `session` 表里的 `summary_additions/deletions/files/diffs` 字段。真正的对话摘要在 `compaction.ts` 里。别被文件名骗了。

---

## 7. 声明式记忆:AGENTS.md 的两个加载器

opencode 真正意义上"跨会话、让 Agent 记住项目规矩"的机制，是**声明式指令文件**——`AGENTS.md` / `CLAUDE.md` / `CONTEXT.md`。这等价于 Codex 的 AGENTS.md 层、Claude Code 的 CLAUDE.md。但 opencode 有个坑:**它有两个不一致的加载器**,又是双架构分裂的产物。

### 7.1 经典加载器:三种文件 + 首个命中者通吃

`packages/opencode/src/session/instruction.ts`(237 行)是现役的完整加载器：

- **全局层**(`:60-63`):`~/.config/opencode/AGENTS.md`,以及(除非 `disableClaudeCodePrompt`)`~/.claude/CLAUDE.md`。
- **项目层文件名**(`:64-67`):`["AGENTS.md", "CLAUDE.md", "CONTEXT.md"(已废弃)]`。
- **首个命中者通吃**(`:122`,原注释:"The first project-level match wins so we don't stack AGENTS.md/CLAUDE.md from every ancestor"):在项目层,**先命中哪个文件名,就只用那个文件名**,然后沿 `findUp` 把该文件名的所有祖先目录版本都加载进来。这避免了把 AGENTS.md 和 CLAUDE.md 从每一层目录里重复叠加。
- **`config.instructions`**:一个 `string[]`,支持路径 / URL / glob,支持 `~/` 展开。
- **无大小上限**:与 Codex 的 `DEFAULT_PROJECT_DOC_MAX_BYTES=32KiB` 不同,opencode 经典加载器**不对指令文件设字节预算**。

### 7.2 嵌套注入:读文件时顺带塞指令

一个精巧设计:`instruction.resolve()` 支持**读时嵌套注入**。当 read 工具读取某个文件时,如果该目录下有指令文件,其内容会作为 `<system-reminder>` **拼进 read 工具的输出里**;而全局/项目级指令则作为 system 消息注入,格式为 `Instructions from: ${path}\n${content}`。这让"局部目录的规矩"能在 Agent 真正接触到那片代码时才浮现,而非一股脑全塞进开头。

### 7.3 core 加载器:只认 AGENTS.md

而 V2 侧 `packages/core/src/instruction-context.ts`(即 SystemContext V2)**只加载 AGENTS.md**——不认 CLAUDE.md、不认 CONTEXT.md、也不读 `config.instructions`。这是两个加载器的实质分歧:**迁移到 V2 后,指令来源被收窄为单一的 AGENTS.md**,CLAUDE.md 兼容层可能正在被淘汰。

> **多处文档/代码落差**(基于本 commit,任何技术判断都应以源码为准):
> - `CONTEXT.md` 身份双重:既是经典加载器里一个**已废弃的指令文件名**,又是 opencode 仓库**自己的架构文档名**——极易混淆。
> - 仓库内散落 **14 个内部 AGENTS.md**(如 `opencode/src/session/llm/AGENTS.md`),它们是**给开发者看的模块文档**,不是给 Agent 的运行时指令,别混为一谈。
> - 配置 schema 也有 v1/v2 分裂:`core/config.ts:96` 的 `instructions: Schema.String[]` 与 `skills: Schema.String[]`(`:90`)是数组,而某些旧描述里是对象形态。

---

## 8. 程序性记忆与快照:技能与 git 检查点

除了"陈述性"的指令记忆,opencode 还有两种更特殊的记忆形态。

### 8.1 技能:程序性记忆(SKILL.md)

`packages/opencode/src/skill/index.ts` 实现的技能系统,本质是**程序性记忆(procedural memory)**——"如何做某件事"的可复用知识,以 `SKILL.md` 文件承载。技能从 `~/.claude`、`~/.agents` 以及 config 里发现,还内建了一个 `customize-opencode` 技能。

回顾第 6.2 节:经典压缩把 `skill` 工具的输出列入 `PRUNE_PROTECTED_TOOLS`,**永不修剪**。两处一联系就清楚了:opencode 把技能视为高价值的程序性知识,写入(SKILL.md 文件)和运行时(压缩保护)两端都给它开了绿灯。这和 Codex `memories/skills/` 目录的定位相似,但 opencode 的技能是**用户/生态手写的静态文件**,而非 Codex 那样由 Phase 2 子 Agent 提炼产出。

### 8.2 快照:git shadow repo 做检查点

`packages/opencode/src/snapshot/index.ts`(V1)与 `packages/core/src/snapshot.ts`(V2)实现了**基于 git 的检查点/回退**:

- V1 用一个 **git 影子仓库**(独立的 `GIT_DIR`),把工作区状态拍成 git 提交。
- V2 改为**内容寻址**(content-addressed),未跟踪文件上限 `2MiB`。
- 两者都服务于 `revert.ts` 的撤销能力——配合第 3.3 节 V2 的"按序列号砍历史",opencode 能同时回退**对话状态**(SQL 事件)和**文件系统状态**(git 快照)。

这是一种"记住世界长什么样"的记忆:不是记对话,而是记**代码工作区的每个检查点**,让"撤销刚才那步改动"成为可能。前作里 Codex 有 `WorldState`,但用 git 影子仓库做文件级快照/undo 是 opencode 相对独特的一笔。

---

## 9. 系统上下文、待办与提醒:合成出来的记忆

最后一组机制,补齐 opencode"合成式"的短时记忆。

### 9.1 系统上下文:带类型、可刷新的 Sources

`packages/core/src/system-context/`(`index.ts` 320 行、`builtins.ts` 50 行、`registry.ts` 49 行)把系统上下文抽象成**一组带类型、可刷新的 Source**。内建两个:`core/environment`(环境信息)和 `core/date`(当前日期);指令则通过 `core/src/instruction-context.ts`(key 为 `core/instructions`)接入。每个 Source 能在需要时**重新求值刷新**——这正是第 4 节"上下文纪元"要去检测变更、复位基线的对象。

### 9.2 待办:SQLite 里的工作记忆

`packages/core/src/session/todo.ts` 把待办事项存进 SQLite 的 `todo` 表,由 `todowrite` 工具驱动。这是一种**结构化的工作记忆**:Agent 把"当前要做的几件事"外化成持久化的清单,而非全靠上下文窗口里的自然语言硬记。与 MemPalace 的 diary、Codex 的任务树不同,opencode 的 todo 就是朴素的"待办列表落库",服务于单会话内的任务跟踪。

### 9.3 提醒:合成出来的文本 part

`packages/core/src/session/reminders.ts`(92 行)会**合成一些文本 part**注入对话——比如切换 plan/build 模式时的提示。它们不是用户或模型产生的真实消息,而是系统**动态编造、临时插入**的上下文片段,用完即弃。这类似于本 prompt 体系里的 `<system-reminder>`:一种"临时贴在 Agent 眼前的便签",严格说不算持久记忆,但它是 opencode 上下文构造的有机部分。

---

## 10. 整体数据流与设计取舍

把所有环节串起来:

```
   声明式记忆(跨会话)                    程序性记忆              快照记忆
   AGENTS.md / CLAUDE.md / CONTEXT.md    SKILL.md              git shadow repo
   两个加载器(经典3种 / core仅AGENTS.md)  (~/.claude,~/.agents) 检查点 / undo
        │ 注入 system 消息 + 读时嵌套       │ 压缩时受保护          │ revert.ts
        │  <system-reminder>               │ (PRUNE_PROTECTED)     │
        ▼                                  ▼                      ▼
   ┌───────────────────── 一次 opencode 会话 ──────────────────────┐
   │  工作记忆: token 估算(CHARS_PER_TOKEN=4) → 溢出(buffer 20k)     │
   │   ├ V1 经典栈: 隐藏 compaction 子Agent + filterCompacted 重排   │
   │   │           PRUNE_PROTECT=40k, skill输出永不修剪              │
   │   └ V2 core栈: 内联LLM摘要 + 事件(Compaction.Started/Ended)     │
   │              共享 SUMMARY_TEMPLATE(Goal/进度/决策/下一步/文件)   │
   │  系统上下文: 带类型可刷新 Sources(environment/date/instructions)│
   │  待办: todo 表(工作记忆)   提醒: 合成文本 part(即用即弃)         │
   └──────────────────────────────┬────────────────────────────────┘
                                   │
        ┌──────────────────────────┴───────────────────────────┐
        │  V1 经典                          V2 core(事件溯源)     │
        │  message / part 表                events → projector    │
        │                                   ★ durable.seq 缺失即崩溃│
        ▼                                   ▼                      │
   ┌────────────────────────────────────────────────────────────┐ │
   │  SQLite 主库 opencode.db (WAL, foreign_keys=ON) ← 权威存储    │ │
   │  session / message / part / session_message(seq唯一)         │ │
   │  session_input(steer|queue) / todo                          │ │
   │  session_context_epoch: baseline / baseline_seq(纪元)        │◀┘
   └───────────────────────────┬────────────────────────────────┘
                               │ 读取: history.ts 历史开窗
                               ▼
        seq >= compaction.seq  OR  (system 且 seq > baseline_seq)
                               │
                               ▼
                   喂给模型的上下文窗口
   (旁路: storage.ts 已降级为 JSON KV + 迁移垫片, 不再是消息主存储)
   ★ 全局: 无向量库 / 无嵌入 / 无事实抽取 —— 没有专门的长期记忆子系统
```

拆解至此，opencode 的记忆机制有几处极具借鉴价值，也有几处需要警惕的"命名陷阱"与"架构分裂":

1. **刻意不做"长期记忆",把需求拆回工程原语**。这是 opencode 与 Codex/mem0/MemPalace 最根本的分野。它没有向量库、没有嵌入、没有 LLM 事实抽取管线;"记忆"被拆解成**声明式文件(AGENTS.md)+ 会话历史(SQLite)+ 快照(git)+ 技能(SKILL.md)** 四种朴素机制。取舍很清晰:**放弃"从历史里自动提炼可复用经验"的能力,换取整个系统零额外基础设施、完全可预测、易调试。** 对一个编码 Agent 来说,"项目规矩写进 AGENTS.md、想撤销就 git revert"往往比"神秘的向量召回"更符合工程师直觉。

2. **SQLite 即权威,与 Codex 的 JSONL 真相相反**。opencode 把对话直接存进 SQLite 并当成唯一真相源,享受事务/外键/查询的原生便利;代价是失去 Codex"纯文本可 grep、索引坏了重建"的透明性。两种哲学各有其理,选择取决于你更看重"数据库的一致性保证"还是"账本的可回溯透明"。

3. **事件溯源 + durable 序列号,把可回溯性做进地基**。V2 的 projector 用单调序列号驱动一切,`durable.seq` 缺失就当场崩溃——对"历史必须可靠"的强硬承诺。回退直接砍掉序列号之后的事件,干净利落,还能与 git 快照联动同时回退对话与文件。这是本系列里对"undo"支持得最彻底的一个。

4. **上下文纪元:把"遗忘旧系统上下文"下沉进 SQL 谓词**。`baseline_seq` 让历史开窗只认当前纪元的系统消息——Codex 靠 prompt 里贴"替换声明"解决的问题,opencode 用一条查询条件就结构化地解决了。这是很干净的"用数据结构消除一类问题"的思路。

5. **结构化压缩摘要模板**。core 栈的 `SUMMARY_TEMPLATE` 强制七段固定结构(目标/约束/进度/决策/下一步/关键上下文/相关文件),比 Codex 的自由格式摘要工整得多——它其实是一份"一次压缩该记住什么"的声明式 schema,很值得被别的 Agent 借鉴。

6. **双架构分裂是最大的认知陷阱**。这一点必须逐条记住:
   - **会话有 V1(`opencode/src/session/`)与 V2(`core/src/session/`)两套**,持久化粒度、压缩机制、指令加载都不同,谈任何细节都要先分清是哪一套;
   - **压缩有两套**:V1 隐藏子 Agent + `filterCompacted` 重排,V2 内联 LLM + 事件溯源;
   - **指令加载器有两个**:经典认 AGENTS.md/CLAUDE.md/CONTEXT.md 且首个命中通吃,core 只认 AGENTS.md;
   - **`summary.ts` 生成的是 git diff 摘要,不是对话摘要**;
   - **`storage.ts` 早已从消息主存储降级为 JSON KV + 迁移垫片**,"文件存储"是历史误名;
   - **`CONTEXT.md` 既是废弃指令文件名,又是仓库架构文档名**;
   - 仓库内 **14 个内部 AGENTS.md 是开发者文档,不是运行时指令**;
   - 指令文件**无大小上限**(与 Codex 的 32KiB 预算不同)。

   **任何基于 opencode 的技术判断,都必须以你实际使用的 commit 源码为准,而非文件名字面含义或旧博客描述。** 这也是本系列坚持"锁定 commit、回溯源码"的原因。

---

### 附:可直接查阅的源码入口

| 主题 | 关键文件(相对 opencode 仓库根,均在 `packages/` 下) |
| --- | --- |
| SQLite/Drizzle 会话 schema | `core/src/session/sql.ts` |
| V2 事件溯源投影器 | `core/src/session/projector.ts` |
| V2 消息 reducer(内存+DB 双适配) | `core/src/session/message-updater.ts` |
| V2 只读门面 | `core/src/session/store.ts` |
| 历史开窗 | `core/src/session/history.ts` |
| 上下文纪元 | `core/src/session/context-epoch.ts` |
| 通用 JSON KV + 迁移垫片 | `opencode/src/storage/storage.ts` |
| 工作记忆:经典压缩栈 | `opencode/src/session/compaction.ts`、`overflow.ts`、`message-v2.ts`(filterCompacted) |
| 工作记忆:core 压缩栈 + 摘要模板 | `core/src/session/compaction.ts` |
| 隐藏压缩子 Agent | `opencode/src/session/agent.ts`(:219-233) |
| token 估算 | `core/src/util/token.ts` |
| 声明式指令:经典加载器 | `opencode/src/session/instruction.ts` |
| 声明式指令:core 加载器(仅 AGENTS.md) | `core/src/instruction-context.ts` |
| 系统上下文 Sources | `core/src/system-context/`(`index.ts`/`builtins.ts`/`registry.ts`) |
| 程序性记忆:技能 | `opencode/src/skill/index.ts` |
| 快照/检查点(git) | `opencode/src/snapshot/index.ts`、`core/src/snapshot.ts`、`revert.ts` |
| 待办工作记忆 | `core/src/session/todo.ts` |
| 合成提醒 part | `core/src/session/reminders.ts` |
| git diff 摘要(命名陷阱) | `core/src/session/summary.ts` |

> 本文为《Memory in Action》系列的一章,基于源码撰写,仅供技术学习与研究。所有代码常量、路径与行号以 opencode commit `403164a` 为准,后续版本可能演进。
