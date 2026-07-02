# Claude Code 的记忆机制（Memory in Claude Code）

> 本文基于 [`claude-code-best/claude-code`](https://github.com/claude-code-best/claude-code) 的真实源码撰写,分析锁定 commit [`d0713bd`](https://github.com/claude-code-best/claude-code/tree/d0713bdd169eefffab4f3d6bc361cb93be66bf52)(2026-06-29)。这是一个对 Anthropic 官方 Claude Code CLI 做**逆向工程重建**的开源实现(`package.json` 里 `name: "claude-code-best"`、`description: "Reverse-engineered Anthropic Claude Code CLI"`)。文中所有实现细节、常量、文件路径与行号区间均可回溯到该版本源码,未做臆测;凡代码与文档不一致、命名误导、以及**桩代码(stub)/死代码**处均已明确标注——在一个逆向重建的代码库里,这类"陷阱"尤其多,是本章要反复提醒的。

前七篇拆解的 [OpenClaw](./memory-in-openclaw.md)、[Hermes](./memory-in-hermes-agent.md)、[mem0](./memory-in-mem0.md)、[MemPalace](./memory-in-mempalace.md)、[Codex](./memory-in-codex.md)、[opencode](./memory-in-opencode.md) 与 [deer-flow](./memory-in-deerflow.md) 里,我们见过各种取舍:有的专设 LLM 事实抽取管线(mem0/Codex/deer-flow),有的干脆不做长期记忆(opencode),有的逐字存(MemPalace)。到了 Claude Code,我们撞上一个**规模上的量变引起质变**:

> **Claude Code 不是"有一套记忆机制",而是同时跑着近十套彼此独立、职责不同的记忆子系统。** 它把"记忆"这件事拆得极细,每一层都单独成一个 service/命令/工具,且大量特性由 GrowthBook 特性开关与 `bun:bundle` 编译期 `feature()` 门控——很多在默认构建里根本没打开,甚至有的整层是 `Auto-generated stub`。

这句话是理解本章的钥匙。Claude Code 的记忆版图不能用"一个子系统"概括,而要按"记忆的类型"铺开来看。下面这张表是全章的地图:

| 记忆类型 | 子系统 | 实质 | 对标前作 |
| --- | --- | --- | --- |
| **声明式记忆** | `utils/claudemd.ts` | CLAUDE.md 分层加载 + `@include` | Codex/opencode 的 AGENTS.md 层 |
| **长期语义记忆** | `memdir/` + `extractMemories/` | 每条一个 `.md`,LLM 分叉写盘,**零向量** | mem0 的抽取,但无嵌入 |
| **"做梦"整合** | `autoDream/` | 后台合并/去重/剪枝记忆 | OpenClaw 的 dreaming |
| **会话摘要记忆** | `SessionMemory/` | 单会话一份 `summary.md`,喂给压缩 | (前作少见) |
| **本地便签记忆** | `local-memory/` + `LocalMemoryRecallTool` | 用户手写便签 + 只读召回工具 | (前作少见) |
| **远端记忆** | `memory-stores/` | 服务端 CRUD + 版本 + PII 脱敏 | (前作少见) |
| **团队共享记忆** | `teamMemorySync/` | 服务端按 repo 同步 + 密钥扫描 | (前作少见) |
| **工作记忆** | `compact/` (六套压缩) | 从全量摘要到 API 原生裁剪 | Codex/opencode 的 compaction |
| **情景记忆** | `sessionStorage.ts` | 每会话一份追加式 JSONL | Codex 的 rollout |
| **程序性记忆** | `skillLearning/` | 从观察中学"本能"→提炼成 SKILL.md | (前作少见) |

本文自"最稳定、最核心"往"最实验、最边缘"排,逐层拆:

1. [情景记忆:每会话一份追加式 JSONL](#1-情景记忆每会话一份追加式-jsonl)
2. [声明式记忆:CLAUDE.md 的分层加载与 @include](#2-声明式记忆claudemd-的分层加载与-include)
3. [长期语义记忆:memdir 与"分叉写盘"抽取](#3-长期语义记忆memdir-与分叉写盘抽取)
4. [做梦:autoDream 的后台整合](#4-做梦autodream-的后台整合)
5. [召回:LLM 选择器 + 用户角色注入的防注入设计](#5-召回llm-选择器--用户角色注入的防注入设计)
6. [会话摘要记忆:SessionMemory](#6-会话摘要记忆sessionmemory)
7. [本地便签与远端记忆:三个都叫-store-的东西](#7-本地便签与远端记忆三个都叫-store-的东西)
8. [团队共享记忆:服务端同步与密钥扫描](#8-团队共享记忆服务端同步与密钥扫描)
9. [工作记忆:六套并存的上下文压缩](#9-工作记忆六套并存的上下文压缩)
10. [程序性记忆:从"本能"到-skillmd](#10-程序性记忆从本能到-skillmd)
11. [整体数据流与设计取舍](#11-整体数据流与设计取舍)

---

## 1. 情景记忆:每会话一份追加式 JSONL

先从最朴素、也最像"账本"的一层说起。Claude Code 的对话历史落在**每会话一份的追加式 JSONL 文件**里,这是情景记忆(episodic memory)的物理真相源。

路径由 `src/utils/sessionStorage.ts` 计算:`<projectDir>/<sessionId>.jsonl`(`sessionStorage.ts:205`),子 Agent 则单独写 `agent-<agentId>.jsonl`(`:258`)。写入是纯行追加:

```ts
// sessionStorage.ts:2611-2622  appendEntryToFile
fs.appendFileSync(fullPath, jsonStringify(entry) + '\n', { mode: 0o600 })
```

有几个工程细节值得记:

- **延迟物化**。`appendEntry`(`:1151`)先把条目缓冲在 `pendingEntries` 里,直到第一条 user/assistant 消息到来才 `materializeSessionFile` 真正建文件,并对已写过的消息去重跳过。摘要、AI 标题、tag、`agent-*` 这类元数据条目则始终可追加(`:1181-1205`)。
- **历史曾在别处**。`config.ts:970-971` 有一句注释:"Removes history field from projects (**migrated to history.jsonl**)"——早期版本把历史塞在项目配置里,后来才迁到独立 JSONL。这和 opencode "storage.ts 从消息主存储降级为 JSON KV"是同一类"迁移活化石"。
- **恢复即回读**。`/resume` 走 `parseJSONL`(`:81`)把 JSONL 读回;持久化可被 `isSessionPersistenceDisabled` 关掉。

这一层的哲学与 **Codex 高度一致**:JSONL 作为不可变、可追加的真相源。但与 opencode 把 SQLite 当权威存储不同,Claude Code 这里**没有数据库**——纯文本文件就是全部。好处是可 grep、可 tail、坏一行不影响其余;代价是没有事务与索引。后面所有"高级"记忆子系统,本质都是从这份 JSONL(或其内存态)派生出来的。

---

## 2. 声明式记忆:CLAUDE.md 的分层加载与 @include

真正意义上"跨会话让 Agent 记住项目规矩"的机制,是声明式指令文件 **CLAUDE.md**。整个加载器就一个大文件:`src/utils/claudemd.ts`(1476 行),`getMemoryFiles()`(`:789`,`memoize` 缓存)把所有来源拼成一个有序的 `MemoryFileInfo[]`。

### 2.1 四层加载 + "反序优先级"

文件头的 docstring 把规则写得很明白(`claudemd.ts:1-25`),加载顺序与优先级是:

1. **Managed(受管)**:`/etc/claude-code/CLAUDE.md` 一类全局策略指令(`:803-822`),外加受管 `.claude/rules/*.md`。始终加载。
2. **User(用户)**:`~/.claude/CLAUDE.md`(`:824-846`),外加 `~/.claude/rules/*.md`。
3. **Project(项目)**:沿目录向上遍历,每层读 `CLAUDE.md`、`.claude/CLAUDE.md`、`.claude/rules/*.md`(`:886-909`)。
4. **Local(本地私有)**:`CLAUDE.local.md`(`:921-932`)。

优先级原则是一句反直觉的话:"**Files are loaded in reverse order of priority**"(`:9`)——越晚加载的优先级越高,模型对它越"上心"。实现上它先收集 `originalCwd` 到根的所有目录,然后 `dirs.reverse()`(`:877`)从**根往 CWD** 处理,于是离当前目录最近的文件排在数组最后=最高优先级。还有一个嵌套 git worktree 的守卫(`:867-883`),跳过主仓在 worktree 之上的 Project 文件,避免重复加载。

> **命名与废弃陷阱**:`getUserClaudeRulesDir` 返回的是 `~/.claude/rules`(不是 `.claude/rules`,因为 base 已经是配置主目录),与 Managed/Project 的 `.claude/rules` 命名不一致(`config.ts:1813-1819`)。而 `CLAUDE.local.md` **实质已半废弃**:代码仍加载它(`:921-932`),但 `/memory` 选择器再也不提供创建它的入口(只给 User 与 Project,`MemoryFileSelector.tsx:61-85`)。

### 2.2 @include:只在叶子文本节点生效

指令文件可以用 `@` 语法引入其他文件(`:18-25`,实现 `extractIncludePathsFromTokens` `:450-534`)。要点:

- 语法:`@path`、`@./path`、`@~/path`、`@/absolute`;裸 `@path` 相对于**引用它的文件所在目录**解析(`:485`)。正则 `/(?:^|\s)@((?:[^\s\\]|\\ )+)/g`(`:458`)。
- **只在叶子文本节点生效**:它用 `marked` 词法器解析,跳过 `code`/`codespan` 与非注释 `html` 节点(`:495-513`),只扫 `text` 节点——所以代码块里的 `@foo` 不会被当成 include。
- **防循环**:`processedPaths` 集合按规范化路径去重,连符号链接的真实路径也加入(`:628-647`);深度上限 `MAX_INCLUDE_DEPTH = 5`(`:536`)。
- **外部引用需批准**:CWD 之外的路径默认跳过,除非 `includeExternal`(`:666-668`);User 记忆始终允许外部(`:833`)。
- **仅文本类扩展名**:非 `TEXT_FILE_EXTENSIONS`(`:95-226`)的文件静默跳过,挡住二进制。

> **文档-代码落差**:docstring 说被引文件"added as separate entries **before** the including file"(`:23`),但代码实际是**先 push 父文件再 push 子文件**(`:663` 在 include 循环 `:665-681` 之前)。MemoryFileSelector UI 恰恰依赖这个"父先子后"的顺序来算缩进层级(`MemoryFileSelector.tsx:94-97`)。文档写反了。

### 2.3 大小限制与注入方式

- `MAX_MEMORY_CHARACTER_COUNT = 40000`(`:91`)**名不副实**:它只被 `getLargeMemoryFiles` 用来"标记"超大文件提醒用户(`:1131-1133`),**从不截断** CLAUDE.md。这与 Codex 的 `DEFAULT_PROJECT_DOC_MAX_BYTES=32KiB` 硬预算不同——Claude Code 的 CLAUDE.md 无硬上限。
- 真正会硬截断的只有自动记忆/团队记忆的 `MEMORY.md` 入口文件:`truncateEntrypointContent`(`:381-384`),上限 `MAX_ENTRYPOINT_LINES = 200`、`MAX_ENTRYPOINT_BYTES = 25_000`(`memdir.ts:35,38`)——因为 MEMORY.md 是"始终在场的索引",必须短。
- **注入为用户上下文,而非系统提示**:`getUserContext`(`context.ts:155-189`)计算 `claudeMd = getClaudeMds(filterInjectedMemoryFiles(await getMemoryFiles()))`,作为 user context 返回(`:170-172,184-187`);`getSystemContext` 只带 git 状态与缓存破坏符,**不含 CLAUDE.md**。三处都 `memoize`,并额外把渲染结果塞进 `setCachedClaudeMdContent`(`:176`)以打破一个 auto-mode 分类器的循环依赖。

> **AGENTS.md 是本章的红鲱鱼**。这个加载器**根本不读 AGENTS.md**——`isMemoryFilePath` 只认 `CLAUDE.md`/`CLAUDE.local.md`/`.claude/rules/*.md`(`:1434-1448`)。仓库里的 `AGENTS.md` 属于另一套"自治权限(autonomy authority)"子系统(`autonomyAuthority.ts:16` `AUTONOMY_AGENTS_FILENAME`),与声明式记忆注入无关。别被文件名骗了。

---

## 3. 长期语义记忆:memdir 与"分叉写盘"抽取

这是 Claude Code 真正的"跨会话长期记忆",核心目录 `src/memdir/` + 抽取服务 `src/services/extractMemories/`。它血缘上像 mem0(读记忆→LLM→增删改),但实现方式非常特别。

### 3.1 memdir:每条记忆一个 .md,零向量

**磁盘位置**:`getAutoMemPath()` = `<memoryBase>/projects/<sanitized-git-root>/memory/`(`paths.ts:229-232`),`memoryBase` 默认 `~/.claude`(`:85-90`)。它按**规范化 git 根**分目录,于是同一仓库的所有 worktree 共享一份记忆(`:203-205`)。

**磁盘格式 = 纯 Markdown,一条记忆一个文件**,带 YAML frontmatter(`memoryTypes.ts:176-186`)。类型是一个**闭合的四类分类法** `['user', 'feedback', 'project', 'reference']`(`memoryTypes.ts:14-19`)。另有一份 **`MEMORY.md` 是索引而非记忆**——每行一个指针 `- [Title](file.md) — hook`,始终注入上下文,硬上限 200 行 / 25KB(`memdir.ts:222-234`)。

> **关键结论:memdir 里没有任何 embedding / vector / cosine。** 跨 `src/memdir`、`extractMemories`、`autoDream` 穷举搜索,零命中。(向量/tf-idf 只存在于无关的 `skillSearch`/`searchExtraTools`。)记忆的检索靠"frontmatter 扫描 + LLM 选择器 + 关键字 grep",不是向量索引。`scanMemoryFiles` 只读每个文件的前 30 行 frontmatter,按新到旧排序,上限 200 个文件(`memoryScan.ts:21-73`)。这和 **deer-flow 的"mem0 血统但零向量"如出一辙**,也和 **MempPalace 的"薄索引"精神相通**。

### 3.2 extractMemories:不是抽取器,是"分叉出一个会自己写文件的 Agent"

这是本章最反直觉的一处。你会以为"记忆抽取"是个确定性的解析器,把对话解析成结构化事实——**不是**。

**触发**:每个完整查询循环结束时(最后一条 assistant 回复且无工具调用),经 `handleStopHooks → executeExtractMemories`(`extractMemories.ts:595-600`)触发,fire-and-forget。

**机制**:它 `runForkedAgent`**完美分叉当前对话**、共享父提示缓存(`:412-424`),`maxTurns: 5`、`skipTranscript: true`。被分叉的 Agent 被告知扮演"记忆抽取子 Agent",然后**自己用 Read/Grep/Glob + Edit/Write 去写记忆文件**——**没有 JSON 的 ADD/UPDATE/DELETE schema**。所谓增删改语义,全是 prompt 指令出来的:"更新已有文件而不是建重复的"、"更新或删除后来发现错误/过时的记忆"(`prompts.ts:32,64-65,80-81`)。工具白名单严格,**禁用 `rm`**(`:37`),且"不得进一步调查验证"(`:41`)。

**去重靠游标**。如果主 Agent 本轮已经写过某个记忆路径,分叉**直接跳过**并推进游标(`hasMemoryWritesSince`,`:120-147`)——主 Agent 与后台抽取每轮互斥。一个 UUID 游标(`lastMemoryMessageUuid`)保证每次只看新消息(`:337-340`),仅在成功后推进(`:429-432`)。节流由 GB `tengu_bramble_lintel` 控制(默认每轮都可,`:374-383`);**没有置信度阈值,全由 LLM 判断**。

> **"extract"是个误称**。这里没有确定性抽取器,而是一个**分叉出来、自己写文件的 LLM Agent**,机制和主 Agent 存记忆的路径完全一样。增删改、去重、"置信度"都是 prompt 指令的行为,不是代码强制的。对比 **Codex 的两阶段结构化抽取**、**mem0 的 `ADD/UPDATE/DELETE` JSON**,Claude Code 走的是"把写记忆这件事交还给一个受约束的子 Agent"——更像 opencode/deer-flow 那种"隐藏子 Agent 做危险活",但把决策权彻底下放给了模型。

---

## 4. 做梦:autoDream 的后台整合

`src/services/autoDream/` 是 Claude Code 版的"做梦(dreaming)"——一次反思性的整合(consolidation),把散落的记忆文件合并、去重、剪枝。这与 **OpenClaw 的 dreaming** 直接对标,但触发方式很不一样。

**是什么**:一次"dream"= 分叉一个跑 `/dream` prompt 的子 Agent(`autoDream.ts:225-235`,`maxTurns: 20`)。`consolidationPrompt.ts` 把它分成四阶段:Orient(ls、读 MEMORY.md)→ Gather(读日志、漂移的事实、窄范围 grep transcript)→ **Consolidate**(合并进已有主题文件、相对日期转绝对、删掉被推翻的事实)→ Prune & index(保持 MEMORY.md < 200 行 / 25KB)。

**何时运行——三道由廉到贵的门**(`autoDream.ts:126-191`):(1)**时间**:距 `lastConsolidatedAt` ≥ `minHours`(默认 **24 小时**);(2)**会话数**:距上次触碰的 transcript ≥ `minSessions`(默认 **5**,排除当前);(3)**锁**。阈值来自 GB `tengu_onyx_plover`。另有 10 分钟扫描节流防止每轮重扫(`:145-152`)。

**并发锁**很精巧:`consolidationLock.ts` 用一个 `.consolidate-lock` 文件,**它的 mtime 本身就是 `lastConsolidatedAt`**,文件体是持有者 PID(`:1,16`)。`tryAcquireConsolidationLock` 用"写后校验"的竞态守卫回收陈旧锁(>1h 或 PID 已死,`:46-84`);分叉失败时 `rollbackConsolidationLock` 把 mtime 倒回去,让时间门重新打开(`:91-108`)。

> **"做梦"并非空闲触发**。尽管有这个隐喻,autoDream 实际是在 **stop-hook 时刻**(一个查询循环结束)被检查触发的,由 24h/5-会话阈值把关,**不是后台空闲定时器,也没有独立调度线程**。此外 `isForced()` 是**死代码**——硬编码 `return false`(`:106-108`),整个 `force` 分支在发行构建里不可达。对比 OpenClaw 真正的后台 dreaming,Claude Code 的"梦"更像是"攒够了再顺手做一次"。

---

## 5. 召回:LLM 选择器 + 用户角色注入的防注入设计

存下来的记忆怎么读回给模型?`src/memdir/findRelevantMemories.ts` 给出的答案是——**用一次 LLM 侧查询在 frontmatter 清单上做选择**,而不是关键字或向量。

**检索**:`scanMemoryFiles` 建一份清单(`[type] filename (ISO ts): description`,`memoryScan.ts:84-94`),然后一次 **Sonnet 侧查询**按 JSON schema 选出 ≤5 个文件名(`findRelevantMemories.ts:19-25,102-136`)。排除 `MEMORY.md`(已在系统提示里)和已浮现过的路径。最近用过的工具也传进去,好让选择器抑制活跃工具的参考文档噪声。无置信度排序,顺序是扫描的"新到旧"。

**新鲜度**:`memoryAge.ts` 生成人类可读的"47 天前"(`:15-20`),对超过 1 天的记忆注入陈旧性告诫(`memoryFreshnessText`,`:33-42`)。

**预算化、非全量**:预取每轮触发一次(受 GB `tengu_moth_copse` 门控),穷模式/单词提示/会话字节耗尽时跳过。预算:每文件 `MAX_MEMORY_BYTES=4096`、`MAX_MEMORY_LINES=200`(`attachments.ts:280,2356`),每会话 `MAX_SESSION_BYTES=60KB`(`:291,2449`),硬上限 **5** 条。这与 **deer-flow 的"按置信度全量转储进 token 预算"不同**——Claude Code 是"LLM 选 5 条 + 严格字节预算",更克制。

### 5.1 关键:召回内容以"用户角色"注入,不是系统提示

这是 Claude Code 记忆安全设计里最值得学的一点:被召回的记忆是**以 `isMeta: true` 的 user 消息、包在 `<system-reminder>` 里**注入的,**不进系统提示**(`messages.ts` 的 `wrapMessagesInSystemReminder`,`attachments.ts:2385-2390` 的 `memoryHeader`)。

这正是**防提示注入(prompt injection)**的模式:召回的记忆内容是**不可信的历史数据**,以 user 角色 + system-reminder 框 + "断言前先验证"的告诫进入上下文,**绝不作为可信的系统指令**。`memoryTypes.ts:155-171` 的"## Before recommending from memory"章节进一步强调把记忆当作"带时间戳、需验证的声明"。

注意这里有个**刻意的不对称**:**MEMORY.md 索引在系统提示里**(它是可信的目录),但**被选中的记忆正文是 user 角色注入**(它们是不可信的历史)。这与 deer-flow 用 `HumanMessage` 而非 `SystemMessage` 注入记忆抵御 OWASP LLM01 是**同一个防御思想的两种实现**。

---

## 6. 会话摘要记忆:SessionMemory

`src/services/SessionMemory/` 是又一套独立机制:一个**自动后台记录员**,为每个会话维护一份 Markdown,主要用来"扛过上下文压缩"。

- **是什么**:`sessionMemory.ts:1-5` 自述——"自动维护一份 Markdown……用分叉子 Agent 定期在后台抽取关键信息"。
- **磁盘**:每会话一份固定文件 `{projectDir}/{sessionId}/session-memory/summary.md`(`filesystem.ts:261-271`),`0o600`/`0o700` 权限。**这不是** `~/.claude/local-memory/` 那棵树(见第 7 节的命名陷阱)。
- **LLM 用法(prompts.ts)**:抽取 = **用 Edit 工具做摘要**,不是召回、不是嵌入。分叉 Agent 被告知"你唯一的任务是用 Edit 工具更新笔记文件,然后停止"(`prompts.ts:55`),填一个固定的 10 段模板(Session Title、Current State、Task、Files and Functions、Errors & Corrections……)。预算:`MAX_SECTION_LENGTH=2000`、`MAX_TOTAL_SESSION_MEMORY_TOKENS=12000`(`:10-11`)。用户可在 `~/.claude/session-memory/config/` 覆盖模板/prompt。
- **触发**:阈值驱动,默认 `minimumMessageTokensToInit=10000`、`minimumTokensBetweenUpdate=5000`、`toolCallsBetweenUpdates=3`(`sessionMemoryUtils.ts:32-36`),可被远端 `tengu_sm_config` 覆盖。只在 `repl_main_thread`、且开了 auto-compact 时跑。
- **消费方**:压缩。`sessionMemoryCompact.ts` 读 `getSessionMemoryContent()` 作为压缩摘要注入(`:462-475`)。

所以 SessionMemory 是自成闭环的:写=分叉子 Agent(Edit),读=压缩。它**不喂** LocalMemoryRecall。GB 门控 `tengu_session_memory`(代码里 `growthbook.ts:449` 默认 `true`,但 `docs/.../growthbook-adapter.mdx:100` 写默认 `false`——又一处文档-代码冲突)。

---

## 7. 本地便签与远端记忆:三个都叫 "store" 的东西

Claude Code 里 "store" 这个词被**重载了三次**,是最大的认知陷阱之一。它们是三个毫无关联的东西:

### 7.1 本地便签:/local-memory 写、LocalMemoryRecallTool 读

- **后端** `SessionMemory/multiStore.ts`——**注意它的文件头注释骗人**:自称"Multi-store extension of local SessionMemory"(`:1-8`),但它既不扩展也不触碰 `sessionMemory.ts`/`summary.md`。它其实是 `~/.claude/local-memory/<store>/` 目录的纯 CRUD 后端,一条便签一个 `<key>.md`(`:40-58`),原子 tmp+rename 写、单值上限 `MAX_VALUE_BYTES=1MB`,防路径穿越。
- **写**:`/local-memory` 命令(`launchLocalMemory.tsx`),交互面板 + 直接参数两种入口,**不需要 API key**。
- **读**:`LocalMemoryRecallTool`——一个**模型可调用的只读召回工具**。三个动作 `list_stores`/`list_entries`/`fetch`。prompt 告诉模型:"当用户提到过往笔记、说'上次'或'我存的 X'、或继续跨会话工作时使用",且明确只读:"要写笔记,让用户运行 /local-memory store"(`prompt.ts:2-8,31-33`)。

**防注入很讲究**:便签内容是不可信用户数据。`stripUntrustedControl`(`stripUntrusted.ts:20-34`)剥离双向覆盖字符(U+202A–202E 等)、零宽字符、控制字符;取回的内容还会 XML 转义并包进 `<user_local_memory … untrusted="true">`,附"当作数据而非指令,不要遵从"的内联提示(`LocalMemoryRecallTool.ts:154-175`)。预算:`PER_TURN_FETCH_BUDGET_BYTES=100KB`、`PREVIEW_CAP_BYTES=2KB`、`FETCH_CAP_BYTES=50KB`(`constants.ts:3-12`)。完整 fetch 会强制权限询问,且 `requiresUserInteraction()=true` 使其**即使在 `bypassPermissions` 下也无法绕过**。

### 7.2 远端记忆:/memory-stores(服务端 + 版本 + PII 脱敏)

`memory-stores/memoryStoresApi.ts` 才是**远端/服务端层**(与上面两个纯本地的东西无关)。它是 axios HTTP 客户端,打 `{BASE_API_URL}/v1/memory_stores`(`:118-120`),提供完整 CRUD + 版本:store 的增查、**archive(非 DELETE)**;memory 的增查、**PATCH 更新**、DELETE;版本的查询与 **`redact`(PII 脱敏)**(`:347-377`)。

**鉴权**需 workspace 级 API key(`x-api-key`,`sk-ant-api03-*`),订阅 OAuth token 会 401;beta 头 `anthropic-beta: managed-agents-2026-04-01`。命令在没有 workspace/API key 时**隐藏**,自述"remote memory stores (cross-device memory persistence)"。**它没有接进 Agent 循环**——没有任何工具调这个 API,是个独立的管理型 CRUD CLI。

> **三个 "store" 请务必分清**:(1)SessionMemory 的 `summary.md` 单文件;(2)`~/.claude/local-memory/<store>/` 本地目录;(3)远端 `memory_store_id` 服务端对象。名字相同,互不相干。此外 `memoryStoresApi.ts` 的注释多次自陈端点是"binary reverse-engineering of v2.1.123"(`:5,17,251,311`)——提醒你这些 HTTP 动词是从二进制字符串逆推的,未必在线验证过。

---

## 8. 团队共享记忆:服务端同步与密钥扫描

`src/services/teamMemorySync/`(index.ts 1256 行)让记忆在团队间共享。**是服务端同步,不是 git 同步**(git 只用来识别 repo)。

- **契约**:`GET/PUT /api/claude_code/team_memory?repo={owner/repo}`(`index.ts:8-13`)。语义:**拉取=服务端按 key 覆盖本地;推送=只上传 sha256 与 `serverChecksums` 不同的 key**(增量);服务端 upsert;**本地删除不传播**(下次拉取会恢复)。上限 `MAX_FILE_SIZE_BYTES 250_000`、`MAX_PUT_BODY_BYTES 200_000`。
- **磁盘**:`getTeamMemPath()` = `<memoryBase>/projects/<...>/memory/team/`,是自动记忆的子目录(`teamMemPaths.ts:81-86`);需开自动记忆 + GB `tengu_herring_clock`(默认 false)。
- **密钥扫描(OWASP)**:`secretScanner.ts` 是 gitleaks 的精选子集(AWS/GCP/Azure/DO,以及运行时拼装的 Anthropic key 前缀 `['sk','ant','api'].join('-')` `:46`——把字面量藏起来防外泄)。`teamMemSecretGuard.ts:15` 的 `checkTeamMemSecrets` 挂在 FileWrite/FileEdit 的 `validateInput` 上,一旦检测到密钥就**阻止写入**团队记忆文件:"Team memory is shared with all repository collaborators"(`:38-40`)。
- **watcher**:`fs.watch` 监听团队目录,启动先拉取,之后 **2 秒去抖推送**(`DEBOUNCE_MS=2000`)。有熔断器 `pushSuppressedReason`——注释记着"一台 no_oauth 设备在 2.5 天里发了 16.7 万次推送事件"(`:45-51`)的真实事故。
- **路径安全**极重:`validateTeamMemKey`/`validateTeamMemWritePath` 拒绝空字节、URL 编码穿越、NFKC unicode 穿越、反斜杠、绝对路径,并用 `realpath` 挡符号链接逃逸(`:96-256`)。

这一层把 **deer-flow 的"记忆写入前扫密钥"防御做到了极致**:不仅扫,还把守写入钩子、熔断异常推送、硬化路径校验——因为"共享"意味着一旦泄密就是团队级事故。

---

## 9. 工作记忆:六套并存的上下文压缩

对话撑爆窗口怎么办?Claude Code 有**六套并存、分层的压缩机制**,从"最贵的全量 LLM 摘要"到"最便宜的服务端原生裁剪"。这是本系列里压缩机制**最庞杂**的一个。

| 机制 | 触发 | 做什么 |
| --- | --- | --- |
| **A. 全量 compact** | 手动/兜底 | 分叉 Agent 摘要整段对话,产出边界标记 + 摘要消息,重建上下文 |
| **B. autocompact** | token 阈值(API 调用前) | 先试 session-memory 压缩,再退回全量 compact |
| **C. reactive compact** | API 真的返回 too-long | 紧急兜底,ant-only |
| **D. microcompact** | 计数/时间 | **裁工具结果**(非整条消息),不做摘要 |
| **E. API 原生 microcompact** | 服务端策略 | 发 `clear_tool_uses_20250919` 等 context-edit 指令 |
| **F. snip** | 用户/模型驱动 | 按 UUID 选择性移除消息 |

几个要点:

- **全量 compact**(`compact.ts:411`)为摘要预留 `MAX_OUTPUT_TOKENS_FOR_SUMMARY = 20_000`(注释:"基于 compact 摘要输出 p99.99 为 17,387 tokens",`autoCompact.ts:29-30`)。重建后 `buildPostCompactMessages = [边界标记, 摘要, stripToolUseResults(保留的尾部), attachments, hookResults]`(`:336-343`);已在保留尾部可见的文件内容不再重附,"每次压缩省下高达 25K tokens"(`:1450-1461`)。
- **autocompact**(`autoCompact.ts:270`)阈值 = 有效窗口 − buffer,buffer 按窗口大小 50K/30K/13K 分档(`:62-82`)。有个真实生产修复:熔断器在连续失败 `MAX_CONSECUTIVE_AUTOCOMPACT_FAILURES = 3` 后停手——"1,279 个会话曾连续失败 50+ 次……每天浪费约 25 万次 API 调用"(`:96-99`)。
- **microcompact**(`microCompact.ts:257`)只裁 `COMPACTABLE_TOOLS`(Read/shell/Grep/Glob/WebSearch/WebFetch/Edit/Write)的**结果**,不摘要。有基于时间(缓存 TTL 过期即清)、基于缓存编辑(`cache_edits`/`delete_tool_result` 不失效前缀)两条子路径。
- **摘要 prompt 结构**(`prompt.ts:61-131`):强制模型先写 `<analysis>` 草稿(后被剥离),再产出含 **9 个编号段**的 `<summary>`:Primary Request/Intent、Key Technical Concepts、Files and Code Sections、Errors and fixes、Problem Solving、**All user messages**、Pending Tasks、Current Work、Optional Next Step。这份 schema 比 opencode 的七段模板还细,是本系列里**最工整的压缩摘要定义**。
- **postCompactCleanup**(`postCompactCleanup.ts:43`)清理被压缩失效的各种缓存,但**刻意不清已调用技能的内容**(必须跨压缩存活,`:24-33`)——呼应"技能=高价值程序性知识"。

> **压缩层的陷阱最多**:
> - **`contextCollapse/` 整层是桩代码**:三个文件都是 `// Auto-generated stub`,`applyCollapsesIfNeeded` 原样返回消息、`persist.ts` 什么都不存。可它仍被 autocompact/postCompactCleanup 引用并受 `feature('CONTEXT_COLLAPSE')` 门控,持久化类型也在 `types/logs.ts` 里定义好了——**设计意图存在,逻辑未发行**。这是全库最大的文档-现实落差。
> - **"microcompact"是误称**:它不摘要,而是清除/删除工具结果载荷;遗留路径已删,注释承认"tengu_cache_plum_violet is always true"(`microCompact.ts:292`)。
> - **`sessionMemoryCompact`/`cachedMicrocompact`/`timeBasedMC` 都是实验/默认关**;`sessionMemoryCompact.ts:1` 文件头直接写着 "EXPERIMENT"。
> - **`snip` 的 "IfNeeded"/`force` 是幌子**:`isSnipRuntimeEnabled()` 恒返回 true,"IfNeeded" 指的是"有没有边界",不是 token 阈值(`snipCompact.ts:79-156`)。

---

## 10. 程序性记忆:从"本能"到 SKILL.md

最后一层最"未来":`src/services/skillLearning/` 试图**从观察到的会话行为里学出可复用技能**。这是程序性记忆(procedural memory),前作里几乎没有对应物。

**两级模型**:**本能(Instinct)**是原子的"触发→动作"规则,带置信度(`types.ts:61-76`),是学习的原材料;**技能(Skill)**是从聚类的本能提炼出的、持久的 `SKILL.md` 产物。

- **门控**:编译期 `feature('SKILL_LEARNING')`,但**运行时默认关**——需运维跑 `/skill-learning start` 设 `SKILL_LEARNING_ENABLED=1`(`featureCheck.ts:32-35`)。
- **观察收集**(`observationStore.ts`):追加 `StoredSkillObservation` 到 `<HOME>/.claude/skill-learning/projects/<id>/observations.jsonl`;字段经 `scrubText` 用 `SECRET_PATTERNS` 脱敏成 `[REDACTED]`,截断到 5000 字符,30 天后清除。
- **LLM 驱动 + 非 LLM 兜底**(`llmObserverBackend.ts`):主分析后端用 **Haiku** 抽 ≤3 条原子本能(JSON),带熔断器(失败 3 次冷却 60s)在出错时**退回启发式后端**。
- **聚类/晋升**(`evolution.ts:33` / `promotion.ts:54`):按 `domain:trigger` 分组,要求 `minClusterSize`(默认 3)——"独立重复观察不可妥协"(`:44-51`);跨 ≥2 个项目、平均置信度 ≥0.8 的项目级本能被晋升为**全局**。
- **置信度动力学**(`instinctStore.ts`):矛盾时 −0.1,<0.3 转 `conflict-hold`;≥0.8 从 `pending` 转 `active`;每周衰减 0.02。生命周期状态:`pending/active/stale/superseded/retired/archived/conflict-hold`。
- **技能生成**(`skillGenerator.ts`):平均置信度 ≥0.75 才写 `SKILL.md`(YAML frontmatter,`MAX_SKILL_FILE_BYTES=50_000`)。

这套"观察→本能→聚类→晋升→技能"的管线,是 Claude Code 记忆版图里唯一**真正从经验里自动提炼可复用知识**的部分——恰恰是 opencode 明确放弃、而 Codex 用 Phase 2 子 Agent 部分实现的能力。但它默认关闭,更像一块"实验田"。

---

## 11. 整体数据流与设计取舍

把所有环节串起来:

```
声明式(跨会话)          长期语义记忆              程序性记忆            远端/团队
CLAUDE.md 分层          memdir/ (每条一个.md)     skillLearning/       memory-stores(服务端)
 +@include(depth5)       零向量·四类分类法         观察→本能→技能        teamMemorySync(按repo)
 首个不适用/反序优先级    MEMORY.md索引(200行/25KB)  Haiku抽本能+启发式兜底  密钥扫描+写入钩子拦截
     │注入=user上下文        │                        │默认关(需/start)      │2s去抖+熔断
     ▼                       │写: extractMemories        ▼                    ▼
┌──────────────────────────┼─分叉Agent自己写文件(非结构化抽取)──────────────────┐
│                            │  autoDream: stop-hook触发, 24h/5会话门, .lock=mtime │
│  工作记忆(六套压缩):        │  读: findRelevantMemories(Sonnet选≤5) 预算4K/文件   │
│   A全量compact(摘要9段)     │      注入=<system-reminder>+isMeta的USER消息(防注入)│
│   B autocompact(阈值,熔断3) │  ★ MEMORY.md索引在系统提示; 记忆正文在user角色      │
│   C reactive D micro E API原生 F snip                                          │
│   contextCollapse=整层桩代码(feature门控但未发行)                              │
│  SessionMemory: summary.md(10段模板), 喂给压缩                                 │
│  local-memory: /local-memory写 + LocalMemoryRecall只读(stripUntrusted防注入)   │
└───────────────────────────────┬──────────────────────────────────────────────┘
                                 │ 情景记忆真相源
                                 ▼
        每会话一份追加式 JSONL: <projectDir>/<sessionId>.jsonl (0o600, 行追加)
        (子Agent: agent-<id>.jsonl; 历史曾在config, 已迁出)
   ★ 全局: 无数据库·无向量·无嵌入; 记忆=纯文件系统(md/jsonl) + LLM分叉Agent
   ★ 大量特性由 GrowthBook(tengu_*) + bun:bundle feature() 门控, 默认多为关
```

拆解至此,Claude Code 的记忆机制有几处极具借鉴价值,也有几处必须警惕的陷阱:

1. **把"记忆"拆成近十套单一职责子系统**。这是 Claude Code 与所有前作最根本的分野:它不追求"一套统一的记忆",而是按记忆类型(声明式/语义/会话/便签/远端/团队/工作/情景/程序性)各起一套 service。好处是每层可独立演进、独立门控、独立开关;代价是**认知负担极重**——光 "store" 就有三种含义,"microcompact" 不 compact,"dream" 不在空闲时做,"extract" 不是抽取器。

2. **纯文件系统 + LLM 分叉 Agent,零数据库零向量**。与 opencode(SQLite 权威)、mem0/deer-flow(向量或至少结构化抽取)都不同,Claude Code 的记忆全是 `.md` + `.jsonl` 文件,检索靠"LLM 选择器 + 关键字 grep",写入靠"分叉一个会自己写文件的子 Agent"。它把"记忆是什么、该记什么"的判断权,几乎完全交给了 LLM 自身(通过 prompt 约束),而非代码 schema。这是"用模型能力替代工程管线"的极致。

3. **召回内容以用户角色注入,是最值得抄的安全设计**。MEMORY.md 索引(可信目录)进系统提示,记忆正文(不可信历史)以 `<system-reminder>` + user 角色注入,并附"验证后再断言"的告诫。加上 `stripUntrustedControl` 剥离双向/控制字符、`untrusted="true"` 包裹——这套防提示注入的分层,和 deer-flow 的 `HumanMessage` 注入殊途同归,但做得更细。

4. **团队记忆的"密钥扫描 + 写入钩子拦截 + 熔断"是共享记忆的教科书**。因为一旦泄密就是团队级事故,Claude Code 在写入钩子处直接拦截含密钥的团队记忆写入,并对异常推送熔断——把"共享"的风险控制前置到了写入时刻。

5. **压缩摘要 schema 是本系列最工整的**。九段固定结构(含"All user messages"这种别处没有的段)+ 强制 `<analysis>` 草稿再产出 `<summary>`,把"一次压缩该记住什么"编码得极清楚,很值得被其他 Agent 借鉴。

6. **逆向重建 + 特性开关 = 陷阱的温床,必须以源码为准**。这一点在本章反复出现,务必逐条记住:
   - **`contextCollapse/` 整层是 `Auto-generated stub`**,有门控、有类型、但逻辑未发行;
   - **`extractMemories` 不是确定性抽取器**,是分叉 LLM Agent 自己写文件;
   - **"dream" 在 stop-hook 触发**,非空闲;`isForced()` 是死代码;
   - **三个 "store"**(summary.md / local-memory 目录 / 远端对象)互不相干;
   - **`multiStore.ts` 文件头注释骗人**(自称扩展 SessionMemory,实为 local-memory 后端);
   - **`AGENTS.md` 不是声明式记忆**,属自治权限子系统;
   - **`CLAUDE.local.md` 半废弃**、**`MAX_MEMORY_CHARACTER_COUNT` 从不截断**、**"microcompact" 不摘要**、**snip 的 "force" 是幌子**;
   - **`tengu_session_memory` 代码默认 true、文档默认 false** 等多处文档-代码冲突;
   - 大量特性受 GrowthBook `tengu_*` 与编译期 `feature()` 双重门控,**默认构建里多数是关的**。

   **任何基于 Claude Code 的技术判断,都必须以你实际使用的 commit 源码 + 实际生效的特性开关为准,而非文件名字面含义、注释、或 docs/ 描述。** 在一个逆向重建的代码库里,这条纪律比任何一章都更重要。

---

### 附:可直接查阅的源码入口

| 主题 | 关键文件(相对仓库根,均在 `src/` 或 `packages/` 下) |
| --- | --- |
| 情景记忆:会话 JSONL | `utils/sessionStorage.ts`(`:205` 路径、`:2611` 追加、`:1151` 延迟物化) |
| 声明式记忆:CLAUDE.md 加载器 | `utils/claudemd.ts`(`:789` getMemoryFiles、`:450` @include、`:1141` filterInjected) |
| 声明式记忆:注入 | `context.ts`(`:155` getUserContext)、`bootstrap/state.ts`(缓存) |
| 声明式记忆:/memory UI | `commands/memory/memory.tsx`、`components/memory/MemoryFileSelector.tsx` |
| 长期语义记忆:memdir | `memdir/memdir.ts`、`memoryTypes.ts`、`memoryScan.ts`、`paths.ts` |
| 长期语义记忆:抽取(分叉Agent) | `services/extractMemories/extractMemories.ts`(`:412` runForkedAgent)、`prompts.ts` |
| 做梦:整合 | `services/autoDream/autoDream.ts`、`consolidationLock.ts`、`consolidationPrompt.ts` |
| 召回:LLM 选择器 | `memdir/findRelevantMemories.ts`、`memoryAge.ts`;注入 `utils/attachments.ts`(`:2385` memoryHeader)、`utils/messages.ts` |
| 会话摘要记忆 | `services/SessionMemory/sessionMemory.ts`、`prompts.ts`、`sessionMemoryUtils.ts` |
| 本地便签:后端 | `services/SessionMemory/multiStore.ts`(命名陷阱) |
| 本地便签:只读工具 | `packages/builtin-tools/src/tools/LocalMemoryRecallTool/`(`LocalMemoryRecallTool.ts`/`prompt.ts`/`stripUntrusted.ts`/`constants.ts`) |
| 本地便签:/local-memory | `commands/local-memory/launchLocalMemory.tsx` |
| 远端记忆 | `commands/memory-stores/memoryStoresApi.ts`、`launchMemoryStores.tsx` |
| 团队记忆:同步 | `services/teamMemorySync/index.ts`、`secretScanner.ts`、`teamMemSecretGuard.ts`、`watcher.ts` |
| 团队记忆:路径 | `memdir/teamMemPaths.ts`、`teamMemPrompts.ts` |
| 工作记忆:压缩栈 | `services/compact/`(`compact.ts`/`autoCompact.ts`/`microCompact.ts`/`snipCompact.ts`/`prompt.ts`/`postCompactCleanup.ts`) |
| 工作记忆:桩代码 | `services/contextCollapse/`(`index.ts`/`operations.ts`/`persist.ts` 均 `Auto-generated stub`) |
| 程序性记忆:技能学习 | `services/skillLearning/`(`instinctStore.ts`/`observationStore.ts`/`evolution.ts`/`promotion.ts`/`skillGenerator.ts`/`llmObserverBackend.ts`) |

> 本文为《Memory in Action》系列的一章,基于源码撰写,仅供技术学习与研究。所有代码常量、路径与行号以 claude-code commit `d0713bd` 为准,后续版本可能演进。
