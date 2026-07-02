# MemPalace 的记忆机制（Memory in MemPalace）

> 本文基于 [`MemPalace/mempalace`](https://github.com/MemPalace/mempalace) 的真实源码撰写，分析锁定 commit [`da5a48c`](https://github.com/MemPalace/mempalace/tree/da5a48caf5d8a843df7568a00e44c714bd91ab11)（2026-06-30）。文中所有实现细节、常量、文件路径与行号区间均可回溯到该版本源码，未做臆测；凡代码与文档不一致处均已明确标注。

前三篇拆解的 [OpenClaw](./memory-in-openclaw.md)、[Hermes](./memory-in-hermes-agent.md) 与 [mem0](./memory-in-mem0.md) 各自代表了一种记忆哲学：OpenClaw 用 Markdown 文件 + FTS5/向量混检并辅以"做梦"整理；Hermes 用严格容量上限的冻结快照 + 学习回路；mem0 则把记忆抽象成一个靠 LLM **抽取事实**的通用中间件。它们有一个共同点——**都要用 LLM 去"理解"并改写你的原始内容**（抽取、摘要、合并、替换）。

MemPalace 走的是一条**几乎完全相反**的路。它的自我定位是：

> **A local-first memory palace for AI agents. Store verbatim. Retrieve with zero LLM calls.（给 AI Agent 用的本地优先记忆宫殿：逐字存储，零 LLM 检索。）**

这句话里藏着 MemPalace 两条最核心、也最"叛逆"的设计信条：

1. **逐字存储（Verbatim / store-what-you-see）**：绝不让 LLM 抽取、摘要或改写你的原文。存进去的是什么，取出来的就是什么。
2. **零 API 检索（Zero-LLM / local-first）**：检索链路里没有任何一次模型推理调用，连向量化都用本地 ONNX 模型跑，整台机器断网也能工作。

这套信条之上，MemPalace 借用了两个古老的隐喻来组织记忆：**记忆宫殿（Memory Palace）** 与 **卡片盒笔记法（Zettelkasten）**。它把记忆空间建模成一座宫殿——分**厢房（wing）**、**房间（room）**、**抽屉（drawer）**，再配上一套叫**壁橱（closet）**的索引卡系统。

本文自底向上拆解 MemPalace 的记忆架构：

1. [宫殿数据模型：厢房 / 房间 / 抽屉 / 壁橱](#1-宫殿数据模型厢房--房间--抽屉--壁橱)
2. [逐字存储与确定性抽屉 ID](#2-逐字存储与确定性抽屉-id)
3. [分层记忆栈：Layer 0-3 与"唤醒预算"](#3-分层记忆栈layer-0-3-与唤醒预算)
4. [壁橱：AAAK 索引卡与虚拟行号](#4-壁橱aaak-索引卡与虚拟行号)
5. [挖掘管线：从文件到抽屉](#5-挖掘管线从文件到抽屉)
6. [本地 ONNX 嵌入:零 API 的向量化](#6-本地-onnx-嵌入零-api-的向量化)
7. [检索管线:语义 + BM25 + 壁橱加权的混合打分](#7-检索管线语义--bm25--壁橱加权的混合打分)
8. [去重、WAL 审计与幂等](#8-去重wal-审计与幂等)
9. [后台钩子:离线挖掘与 v4 的"零 token"哲学](#9-后台钩子离线挖掘与-v4-的零-token-哲学)
10. [单时态知识图谱与规则式实体识别](#10-单时态知识图谱与规则式实体识别)
11. [可插拔后端、MCP 服务与记忆动力学](#11-可插拔后端mcp-服务与记忆动力学)
12. [整体数据流与设计取舍](#12-整体数据流与设计取舍)

---

## 1. 宫殿数据模型:厢房 / 房间 / 抽屉 / 壁橱

MemPalace 把记忆空间建模成一座**四级层次的宫殿**,这套隐喻贯穿全部代码:

| 隐喻 | 实体 | 实质 |
| --- | --- | --- |
| **厢房 wing** | 顶层命名空间 | 一个大的记忆分区(如某个项目、某个知识域) |
| **房间 room** | 二级分类 | 厢房内的主题分组(由 `detect_room` 规则判定) |
| **抽屉 drawer** | 最小存储单元 | 一段**逐字**存下的文本块(chunk) + 元数据 |
| **壁橱 closet** | 索引卡 | 对一个来源文件的**紧凑索引摘要**,加速定位(见 [第 4 节](#4-壁橱aaak-索引卡与虚拟行号)) |

物理上,这一切都落在本地目录 `~/.mempalace/` 下:抽屉与壁橱进 ChromaDB(默认后端),知识图谱进 SQLite,配置进 `config.json`。`mempalace/config.py` 的解析优先级是 **环境变量 > `~/.mempalace/config.json` > 内置默认值**,默认宫殿路径 `DEFAULT_PALACE_PATH = ~/.mempalace/palace`。

### 1.1 双集合:抽屉库 vs 壁橱库

在 ChromaDB 后端里,MemPalace 开了**两个物理集合**:

- **主集合**(抽屉库):存所有逐字文本块的向量 + payload。
- **壁橱集合**:`palace.py` 的 `get_closets_collection`(`:262`)把集合名硬编码为 `mempalace_closets`,专门存索引卡。

这与 mem0 的"内容库 + `_entities` 实体库"双集合布局形似,但语义完全不同:mem0 的第二个集合存 spaCy 抽出的**实体**,MemPalace 的第二个集合存**文件级索引卡**。

### 1.2 嵌入身份锁

一个容易被忽略但很关键的工程细节:`palace.py` 的 `_enforce_embedder_identity`(`:79-160`)会在每个集合的元数据里**写死当初创建它时用的嵌入模型指纹**。如果你后来换了嵌入模型再来读写同一个集合,它会直接报错——因为不同模型产出的向量根本不可比。这个"嵌入身份锁"避免了"向量维度对得上、语义对不上"的隐蔽数据污染。

---

## 2. 逐字存储与确定性抽屉 ID

这是 MemPalace 与前三个系统**最根本的分野**,值得单独成节。

### 2.1 存什么就是什么

mem0 的写入管线核心是**让 LLM 从对话里抽取事实**(`ADDITIVE_EXTRACTION_PROMPT`),Hermes 会把记忆压进严格的字符预算,OpenClaw 会"做梦"整理。MemPalace **一个都不做**:它把来源文本切成块(chunk),**原封不动**地存进抽屉。没有抽取,没有摘要,没有改写。

这背后的理由,`MISSION.md` 说得很直白:**任何 LLM 改写都是有损压缩,都会引入幻觉或丢失细节**。MemPalace 宁可多存一点、检索时多花点力气,也要保证"取出来的字节和存进去的一模一样"。这对代码库、法律文档、聊天记录这类**细节敏感**的场景是决定性的优势。

### 2.2 确定性、内容寻址的抽屉 ID

因为不改写内容,MemPalace 得以用一套**确定性 ID**来做幂等——同样的内容,永远算出同样的抽屉 ID。`mempalace/ids.py` 的 `make_drawer_id_from_chunk`(`:56-68`):

```
drawer_{wing}_{room}_{sha256("len:source_file|len:chunk_index")[:24]}
```

其中 `_delimited_sha256`(`:40-53`)在拼接前给每个字段加了**长度前缀**(`len:value`),避免 `("ab","c")` 与 `("a","bc")` 撞哈希这类分隔符歧义。ID 配方版本号 `ID_RECIPE = "v3"`(`:25`)显式写在文件里,方便未来演进时区分。

这套内容寻址 ID 带来两个直接好处:

- **天然幂等**:同一个文件重复挖掘,算出的抽屉 ID 完全一致,直接覆盖而非产生重复(见 [第 8 节](#8-去重wal-审计与幂等))。
- **无需 LLM 判重**:mem0 要靠"哈希去重 + LLM 语义去重"两层来防冗余,MemPalace 靠确定性 ID + 文件 mtime 就把绝大多数重复挡在门外。

---

## 3. 分层记忆栈:Layer 0-3 与"唤醒预算"

`mempalace/layers.py`(516 行)是 MemPalace 面向 Agent 的门面。它的顶部 docstring(`:6-13`)道出了整个设计的量化目标:

> *"Load only what you need... Wake-up cost: ~600-900 tokens (L0+L1). Leaves 95%+ of context free.(按需加载……唤醒成本约 600-900 token,给上下文留出 95% 以上的空间。)"*

也就是说,MemPalace 把"往 Agent 上下文里塞多少记忆"当成一个**显式的 token 预算问题**来管理,分成四层,逐层加深:

| 层 | 类 | 职责 | 成本 | 关键实现 |
| --- | --- | --- | --- | --- |
| **Layer 0** | 身份层 | 加载 Agent 自我认知(约 100 token) | 极低 | 读 `~/.mempalace/identity.txt`(`:39-73`) |
| **Layer 1** | 唤醒层 | 拉最相关的一小批抽屉"热身" | 低 | `MAX_DRAWERS=15`、`MAX_SCAN=2000`、`MAX_CHARS=3200`(`:81-184`) |
| **Layer 2** | 回忆层 | 按元数据精确取指定抽屉 | 中 | 元数据过滤 get(`:192-244`) |
| **Layer 3** | 搜索层 | 向量语义检索 | 按需 | vector query(`:252-363`) |

`MemoryStack`(`:371-448`)把这四层封装成统一门面。

哲学上,这与 Hermes/mem0 的"检索即注入"有微妙区别:MemPalace 认为 **Agent 的上下文窗口是稀缺资源**,默认只花 L0+L1 的 600-900 token 把 Agent"唤醒",把 95% 的窗口留给实际任务;只有当 Agent 真需要深挖时,才主动调 Layer 2/3。这是一种**吝啬地花上下文**的记忆观。

> **一个命名上的坑**:虽然文档里满是"wake-up"(唤醒)和"recall"(回忆)的说法,但 MCP 服务(见 [第 11 节](#11-可插拔后端mcp-服务与记忆动力学))里**并没有**一个字面叫 `wake_up` 或 `recall` 的工具。这些是分层栈的**内部概念**,对外暴露的是 `mempalace_search` 等更通用的接口。

---

## 4. 壁橱:AAAK 索引卡与虚拟行号

如果说抽屉是"逐字存下的原文",那**壁橱(closet)** 就是 MemPalace 为了在海量逐字内容里快速定位而发明的**索引卡系统**。

### 4.1 AAAK:一张索引卡上写什么

`MISSION.md` 与相关 prompt 把壁橱卡的信息结构概括为 **AAAK**——注意,**"AAAK" 是设计文档/提示词里的助记词,并不是代码里的字面常量**。一张壁橱卡本质上是对**一个来源文件**的紧凑索引:它记录这个文件里有哪些关键锚点、大致讲了什么、对应到哪些抽屉,从而让检索时不必扫全部原文就能先命中"哪个文件的哪一段最相关"。

构造壁橱卡的逻辑在 `palace.py`:

- `build_closet_lines`(`:539-624`):从一个文件的内容里提取索引行。
- `upsert_closet_lines`(`:681`):把索引卡写进 `mempalace_closets` 集合,单卡上限 `CLOSET_CHAR_LIMIT = 1500` 字符(`:447-448`),提取窗口 `CLOSET_EXTRACT_WINDOW = 5000`。
- `purge_file_closets`(`:668`):当一个文件被重新挖掘时,先清掉它旧的壁橱卡再重建,保证索引与内容同步。

### 4.2 虚拟行号:给逐字内容一套稳定坐标

`docs/virtual-line-numbering.md` 与 `searcher.py`(`:1339+`)描述了一套**虚拟行号(virtual line numbering)** 机制。因为抽屉是按 chunk 切的、跨块可能打断原始行,MemPalace 给检索结果**叠加一套虚拟的、稳定的行号坐标**,让 Agent 能精确引用"第几行到第几行",而不受切块边界影响。这对"逐字存储 + 精确引用"的定位场景是关键补强——它让 MemPalace 既能整段还原原文,又能像代码编辑器那样定位到行。

---

## 5. 挖掘管线:从文件到抽屉

把外部内容变成抽屉的过程,MemPalace 叫**挖掘(mine)**,主体在 `mempalace/miner.py`(2040 行)。核心入口 `mine` / `_mine_impl` 大致跑这么一条流水线:

1. **可读性判定**:`READABLE_EXTENSIONS`(`:111-144`)白名单 + `MAX_FILE_SIZE = 500MB` 上限,过滤二进制/超大文件。
2. **切块**:`chunk_text`(`:546-640`),`DEFAULT_CHUNK_SIZE = 800`、重叠 `overlap = 100`——重叠是为了避免关键句被切块边界腰斩。
3. **房间判定**:`detect_room`(`:499-538`)用**规则**(不是 LLM)判断这个块该归到哪个房间。
4. **过道判定**:`detect_hall`(`:895-916`)识别块内的过道信号(见下文 hallway)。
5. **构造抽屉元数据**:`_build_drawer_metadata`(`:1294-1316`),连同确定性 ID 一起。
6. **落盘**:批量写进主集合。
7. **挖后处理**:构建**过道(hallway)**、**隧道(tunnel)**、FTS5 全文索引。

### 5.1 过道 vs 隧道:两种记忆间的连接

MemPalace 用两个空间隐喻描述记忆间的关联,二者尺度不同:

- **过道(hallway)**:`mempalace/hallways.py`(388 行)。**厢房内部**、基于**实体共现**的软连接边——两个抽屉如果反复提到同一批实体,就在它们之间连一条过道。`min_count=2`(至少共现两次才连),边 ID 对称,存成 JSON `~/.mempalace/hallways.json`。
- **隧道(tunnel)**:**跨厢房**的房间级连接,把不同分区里主题相关的房间打通。

这套"过道 + 隧道"和 mem0 的"共享实体 → 检索加权"异曲同工——都是**无 schema、基于共现的轻量关联图**,而非带类型边的知识图谱。区别在于 MemPalace 还额外维护了一个**真正的**知识图谱(见 [第 10 节](#10-单时态知识图谱与规则式实体识别))。

---

## 6. 本地 ONNX 嵌入:零 API 的向量化

MemPalace"零 API"信条最硬核的落地,是它**连向量化都在本地跑**。`mempalace/embedding.py`(475 行):

- **默认模型**:`all-MiniLM-L6-v2`,384 维,ONNX 格式(`:9`、`:157-200`)。
- **可选模型**:`embeddinggemma-300m` 的 ONNX 版(`:206-360`),用 **Matryoshka** 表示裁到 384 维以对齐。
- **执行后端**:ONNX Runtime,provider 支持 `cpu / cuda / coreml / dml`(`:41-58`)——即能吃上 NVIDIA CUDA、苹果 CoreML、Windows DirectML 的硬件加速。

关键结论:**除了首次从 HuggingFace 下载模型权重那一下,整个嵌入过程零网络、零 API。** 这与 mem0(默认走 OpenAI embedding API)、以及绝大多数记忆库形成鲜明对比。断网、内网、隐私敏感环境,MemPalace 都能完整工作。这也是它敢自称 **local-first** 的底气。

---

## 7. 检索管线:语义 + BM25 + 壁橱加权的混合打分

生产检索主体在 `mempalace/searcher.py`(1379 行)的 `search_memories`(`:1049-1321`)。它是一条**混合检索**流水线:

1. **语义检索**:用本地 ONNX 嵌入把 query 向量化,在主集合里做向量近邻。
2. **BM25 关键词检索**:`_bm25_scores`(`:75-131`),经典参数 `k1=1.5`、`b=0.75`。
3. **混合排序**:`_hybrid_rank`(`:184-237`),`vector_weight=0.6`、`bm25_weight=0.4`——语义与关键词六四开。
4. **壁橱加权**:命中壁橱卡的结果按名次加分,`CLOSET_RANK_BOOSTS = [0.40, 0.25, 0.15, 0.08, 0.04]`(`:1172`)——排第 1 的壁橱命中加 0.40,依次递减。
5. **虚拟行号叠加**:给结果附上稳定坐标(`:1339+`)。

### 7.1 壁橱是"信号"而非"闸门"

这里有一个**重要的代码与文档落差**,任何要基于 MemPalace 做技术判断的人都必须知道:

- `docs/CLOSETS.md` 描述过一个 **"壁橱优先闸门"(`_closet_first_hits`)** ——先用壁橱把候选**筛一遍**再检索。
- **但在本文锁定的 commit 上,这个闸门已经不存在了。** 现役代码里,`searcher.py`(`:1121-1127`)有明确注释说明:壁橱现在只作为**排序加权信号(rank boost)**,**不是过滤闸门(not a gate)**。也就是说,壁橱命中会让结果更靠前,但不会把没命中壁橱的结果直接排除掉。

### 7.2 哪些"特性"其实只在跑分里

同样必须澄清的是,README/文档里提到的**时间邻近加权(temporal-proximity)** 和 **偏好模式加权(preference-pattern)**,在生产 `searcher.py` 里**并不存在**——它们只出现在 `benchmarks/` 的评测脚本里。README 把跑分专用的增强和生产链路混为一谈了。

### 7.3 零 LLM,而且是真的零

回到 MemPalace 的招牌信条:**整条检索链路没有任何一次 LLM 调用。** 向量化是本地 ONNX,BM25 是纯算法,壁橱加权是查表,排序是加权求和。项目自己给出的量化说法是 Layer 3 原始检索路径可达 **96.6% 零 LLM**——检索这件事上,它比 mem0(检索本身不调 LLM,但抽取阶段重度依赖 LLM)走得更彻底:MemPalace **写入和读取两端都不需要 LLM**。

---

## 8. 去重、WAL 审计与幂等

### 8.1 三道防线,只有一道用到向量

MemPalace 的去重是分层的,而且**主力根本不靠相似度计算**:

1. **确定性 ID(主力)**:同内容 → 同抽屉 ID → 直接覆盖。这挡住了绝大多数重复(见 [第 2 节](#2-逐字存储与确定性抽屉-id))。
2. **文件 mtime 幂等**:`palace.py` 的 `file_already_mined`(`:1204`)记录文件修改时间,没变过的文件直接跳过,连切块都省了。
3. **近重复清道夫(可选)**:`mempalace/dedup.py`(239 行)才真正用余弦相似度找"内容相近但不完全一样"的抽屉,`DEFAULT_THRESHOLD = 0.15`(`:36-39`)。这是个**兜底的 janitor**,不是主路径。

对比 mem0:mem0 的去重(哈希精确 + LLM 语义)是写入管线的**必经环节**,MemPalace 则把"防重复"这件事**前移到了 ID 设计层**,让绝大多数重复在算 ID 的那一刻就被自然消解。

### 8.2 WAL:审计日志,不是重放日志

`mempalace/wal.py`(97 行)写一个 `~/.mempalace/wal/write_log.jsonl`。名字虽叫 WAL(Write-Ahead Log),但**它不是数据库那种崩溃重放日志**——它是一份**审计 + 脱敏日志**:记录发生过哪些写操作,并在落盘前**对 `content`/`query`/`text` 等敏感键做脱敏(redaction)**。这符合 MemPalace 对隐私的一贯重视:连自己的操作日志都不留明文内容。

### 8.3 并发写锁

挖掘是有锁的:`palace.py` 的 `mine_lock`(`:721`)、`mine_palace_lock`(`:1077`)串行化写操作,配合确定性 ID 的幂等性,让"多个进程同时挖同一批文件"也不会写坏数据。

---

## 9. 后台钩子:离线挖掘与 v4 的"零 token"哲学

这是 MemPalace 的**招牌特性**,也是它区别于所有前作的最大亮点。

前三个系统的记忆写入,或多或少都发生在**对话主循环里**——要么占 Agent 的 token(让它决定存什么),要么在回合间同步触发。MemPalace v4 的思路是:**把记忆挖掘彻底赶出对话链路,让它零 token、异步、在后台发生。**

实现靠一组 **Claude Code 钩子(hooks)**,注册在 `.claude-plugin/hooks/hooks.json`,由 `mempalace/hooks_cli.py` 与 `hooks/` 驱动:

- **触发时机**:`Stop`(一次回答结束)、`SessionEnd`(会话结束)、`PreCompact`(上下文即将压缩前)。
- **节流**:`SAVE_INTERVAL = 15`,避免过于频繁地触发。
- **分离进程**:`_spawn_mine` 用 `start_new_session=True` 把挖掘进程**彻底 detach** 成后台守护进程——它脱离 Agent 会话独立运行,**不占用 Agent 的任何 token,也不阻塞对话**。

`MISSION.md` 把这个设计的动机讲得很清楚:**记忆的形成不应该跟 Agent 抢上下文预算。** 让 Agent 专心对话,等它闲下来(Stop/SessionEnd)或上下文快满了(PreCompact),后台进程再默默把这轮对话挖进宫殿。这就是"离线挖掘(off-chat mining)"。

此外还有一个**可选**的 HTTP 守护/服务(`daemon.py` / `service.py`),给需要常驻服务形态的场景用,但默认路径是上面这套 detached 钩子。

> **与 mem0 Proxy 的对照**:mem0 的 `Mem0` proxy 也用后台线程异步存记忆(`_async_add_to_memory`),思路相近。但 mem0 是在**同一进程内**开后台线程,MemPalace 是**spawn 一个完全独立的操作系统进程**——后者更彻底地把记忆挖掘和 Agent 运行时解耦,代价是要处理跨进程的锁与幂等(正好由 [第 8 节](#8-去重wal-审计与幂等)的确定性 ID + 文件锁兜住)。

---

## 10. 单时态知识图谱与规则式实体识别

除了"过道/隧道"这种轻量共现图,MemPalace 还内建了一个**正儿八经的知识图谱**,`mempalace/knowledge_graph.py`(577 行)。

### 10.1 一个本地、免费的 Zep/Neo4j 替身

知识图谱存在 SQLite:`~/.mempalace/knowledge_graph.sqlite3`(注意:`docs/schema.sql` 的注释里写的是 `.db`,与运行时实际文件名 `.sqlite3` **对不上**,属文档漂移)。它的自我定位(`:14-15`)很明确:**一个本地跑、零成本的 Zep / Neo4j 竞品**。

三张核心表:

- **entities**:实体节点。
- **triples**:`(主语, 谓语, 宾语)` 三元组,即带类型的关系边。
- **attributes**:**在 `schema.sql` 里定义了,但代码里从未使用**——又一处文档/schema 与实现的落差。

### 10.2 单时态,不是双时态

图谱带时间维度,但要精确描述:它是**单时态(uni-temporal)** 的,记录的是**有效时间(valid time)**,而**不是双时态(bitemporal)**——没有"事务时间(transaction time)"那一维。

- 每条边有 `valid_from` / `valid_to`。
- **当前有效**的判据是 `valid_to IS NULL`。
- `invalidate()`(`:356`)不删除边,而是给它盖上 `valid_to` 时间戳——这样历史就完整保留了。
- 支持 **as-of 时间点查询**:`_temporal_filter_sql`(`:122-126`)能查"某个历史时刻,图谱长什么样"。
- 边有 `confidence`,默认 `1.0`。

这套"软失效 + as-of 查询"和 mem0 的 SQLite 审计账本(不可变、可回溯)精神相通,但 MemPalace 把它做成了**可查询的时序知识图谱**,而不只是审计日志。

### 10.3 规则式实体识别:又一次拒绝 LLM

图谱里的实体从哪来?`mempalace/entities.py` / `entity_detector.py` / `entity_registry.py` 给出的答案再次贯彻了"零 LLM"信条:**纯规则 + 正则 + 启发式**,**不用 LLM,也不用 spaCy**。它内建多语言(i18n)模式,唯一的**可选**网络路径是查 Wikipedia 来消歧/补充实体信息——而且是可选的。

对比 mem0 用 spaCy 抽实体、更早的版本甚至用 LLM,MemPalace 连一个 NLP 依赖都不想背,把实体识别压到纯规则层面。这在准确率上肯定不如 LLM,但换来了**零依赖、零成本、可离线、完全可预测**。

---

## 11. 可插拔后端、MCP 服务与记忆动力学

### 11.1 可插拔存储后端(RFC 001)

MemPalace 的存储层是抽象的,`mempalace/backends/` 下按 RFC 001 契约(`spec_version="1.0"`)定义了 `BaseBackend` / `BaseCollection` 两个抽象基类(`base.py`),再实现四种后端:

| 后端 | 文件 | 特点 |
| --- | --- | --- |
| **chroma** | `chroma.py` | 默认,本地 ChromaDB |
| **sqlite_exact** | `sqlite_exact.py` | 纯 SQLite 精确匹配 |
| **qdrant** | `qdrant.py` | 服务端向量库,支持命名空间隔离 |
| **pgvector** | `pgvector.py` | Postgres 向量扩展,支持命名空间隔离 |

后端用**能力标志(capability flags)** 声明自己支持什么——比如 `supports_namespace_isolation` 只有 qdrant / pgvector 的 server 模式为真。注册中心 `registry.py` 按配置挑后端,并统一强制 [第 1.2 节](#12-嵌入身份锁)提到的嵌入身份锁。

这套"一切皆后端"的可插拔性,和 mem0 的"23 种向量库工厂"是同一种工程追求,只是 MemPalace 的后端数量克制得多(4 种),但每种都自带能力协商。

### 11.2 手搓的 MCP 服务

`mempalace/mcp_server.py` 是一个**手写的 JSON-RPC 2.0** MCP 服务(没有用现成 SDK),对外暴露约 40 个工具,采用**只读 / 写锁**模型:

- 检索类:`mempalace_search`、`mempalace_traverse`(走过道/隧道)。
- 图谱类:`mempalace_kg_query` / `kg_add` / `kg_invalidate` / `kg_timeline` / `kg_stats`。
- 写入类:`mempalace_mine`、`mempalace_add_drawer`。
- 日记/检查点:`mempalace_diary_write` / `diary_read` / `checkpoint`。

如前所述,这里**没有**字面叫 `wake_up` / `recall` 的工具——分层栈是库内概念,对外是这套更细粒度的工具集。

### 11.3 记忆动力学:Hebbian / Ebbinghaus / Cepeda

`mempalace/dynamics.py` 给记忆强度引入了一套**认知科学启发**的动力学模型:

- **Hebbian 强化**(Hebb 1949):两条记忆被一起访问,就加强它们的连接,`+0.05` / 次共同访问,上限 `5.0`——"一起激活的神经元连在一起"。
- **Ebbinghaus 遗忘**(Ebbinghaus 1885):记忆强度随时间指数衰减,`strength *= exp(-days/stability)`,地板值 `0.05`——不用就慢慢淡忘,但不会归零。
- **Cepeda 间隔效应**(Cepeda 2006):间隔重复比集中重复记得牢。

这套动力学让 MemPalace 的记忆强度**会随访问模式和时间动态涨落**,和 OpenClaw 的"做梦"整理一样,都是想给静态存储加上一层**类生物的记忆演化**。

---

## 12. 整体数据流与设计取舍

把所有环节串起来:

```
  外部文件 ──▶ ┌───────────────────────────────────────────┐
              │  挖掘管线 miner.py (后台钩子异步触发, 零token) │
              └───────────────────────────────────────────┘
                 │ 可读性过滤 (500MB, 扩展名白名单)
                 │ chunk_text (800字/重叠100)
                 │ detect_room / detect_hall (纯规则, 无LLM)
                 │ 确定性ID: drawer_{wing}_{room}_{sha256[:24]}
                 │ 本地ONNX嵌入 (all-MiniLM-L6-v2, 384维, 零API)
                 ▼
   ┌────────────────────────┐   ┌──────────────────────────┐
   │  主集合 (抽屉库)          │   │  壁橱集合 mempalace_closets │
   │  逐字文本块 + 向量 + 元数据 │   │  AAAK 索引卡 (≤1500字)      │
   │  ← 从不摘要/改写           │   │  加速文件级定位             │
   └────────────────────────┘   └──────────────────────────┘
        │            │                        │
        │            │挖后处理                  │
        │            ▼                        │
        │   ┌─────────────────┐  ┌──────────────────────────┐
        │   │ 过道(厢房内共现)   │  │ 知识图谱 SQLite (单时态)    │
        │   │ 隧道(跨厢房房间)   │  │ entities/triples          │
        │   │ FTS5 全文索引     │  │ valid_from/to, as-of查询   │
        │   └─────────────────┘  │ 规则式实体识别(无LLM/spaCy) │
        │                        └──────────────────────────┘
        ▼
  ┌─────────────────────────────────────────────────────┐
  │  分层记忆栈 layers.py                                   │
  │  L0 身份(~100tok) → L1 唤醒(≤15抽屉) → L2 回忆 → L3 搜索 │
  │  唤醒预算 600-900 token, 留 95% 上下文                   │
  └─────────────────────────────────────────────────────┘
        │
  search_memories() 混合检索 (searcher.py)
        │ 语义(0.6) + BM25(0.4, k1=1.5/b=0.75)
        │ + 壁橱加权 [0.40,0.25,0.15,0.08,0.04] (信号非闸门)
        │ + 虚拟行号叠加
        │ ★ 全程零 LLM 调用 (Layer3 原始路径 96.6% zero-LLM)
        ▼
   top_k 逐字记忆 ──▶ 注入 Agent 上下文
```

拆解至此,MemPalace 的记忆机制有几处极具借鉴价值,也有几处需要警惕的"文档滞后":

1. **逐字存储,反对一切 LLM 改写**。这是 MemPalace 与 mem0/Hermes 最根本的哲学对立。mem0 用 LLM 抽取事实、Hermes 压进字符预算,都承认"有损压缩"是可接受的代价;MemPalace 则认为**任何改写都是幻觉的温床**,宁可原样存、检索时多花力气。**细节敏感场景(代码、法律、聊天原文)是它的主场,海量泛化知识则是它的短板**(不摘要意味着库会更大、更"原始")。

2. **两端零 LLM**。写入靠规则切块 + 确定性 ID,检索靠本地 ONNX + BM25 + 查表加权,连嵌入都在本地跑。这让 MemPalace 成为**唯一能完全离线工作**的一个——断网、内网、隐私敏感环境的独门优势。代价是实体识别、房间判定的准确率不如 LLM。

3. **确定性内容寻址 ID 替代 LLM 判重**。mem0 要两层去重(哈希 + LLM),MemPalace 把防重前移到 ID 设计,让重复在算 ID 那一刻自然消解——这是很干净的"用数据结构消除一类问题"的工程思路。

4. **v4 后台钩子:记忆形成与对话彻底解耦**。用 detached 进程 + Stop/SessionEnd/PreCompact 钩子实现零 token、异步、离线挖掘,让记忆写入完全不跟 Agent 抢上下文。这是本系列里对"记忆成本"处理得最彻底的方案。

5. **唤醒预算:把上下文当稀缺资源**。L0+L1 只花 600-900 token 把 Agent"唤醒",95% 窗口留给任务——一种吝啬而务实的记忆注入观。

6. **文档与代码的落差是最大陷阱**。这一点和 mem0 惊人地相似,且必须逐条记住:
   - `docs/CLOSETS.md` 说的**壁橱优先闸门(`_closet_first_hits`)已不存在**,现役壁橱只是**排序加权信号**;
   - README 提到的**时间邻近 / 偏好模式加权只在 benchmarks 里**,不在生产检索;
   - 知识图谱是**单时态(valid-time)而非双时态**;
   - `attributes` 表**定义了但代码没用**;
   - 图谱运行时文件是 **`.sqlite3`**,schema 注释却写 `.db`;
   - **"AAAK" 是设计文档的助记词,不是代码常量**;
   - WAL 是**审计脱敏日志,不是崩溃重放日志**。

   **任何基于 MemPalace 的技术判断,都必须以你实际使用的 commit 源码为准,而非博客或 docs/ 描述。** 这也是本系列坚持"锁定 commit、回溯源码"的原因。

---

### 附:可直接查阅的源码入口

| 主题 | 关键文件(相对 mempalace 仓库根) |
| --- | --- |
| 分层记忆栈(L0-L3) | `mempalace/layers.py` |
| 宫殿数据模型 / 壁橱 / 嵌入身份锁 | `mempalace/palace.py` |
| 混合检索打分 / 虚拟行号 | `mempalace/searcher.py` |
| 挖掘管线 / 切块 / 房间判定 | `mempalace/miner.py` |
| 确定性抽屉 ID | `mempalace/ids.py` |
| 本地 ONNX 嵌入 | `mempalace/embedding.py` |
| 近重复清道夫 | `mempalace/dedup.py` |
| WAL 审计脱敏日志 | `mempalace/wal.py` |
| 后台钩子 / 离线挖掘 | `mempalace/hooks_cli.py`、`hooks/`、`.claude-plugin/hooks/hooks.json` |
| 单时态知识图谱 | `mempalace/knowledge_graph.py`、`docs/schema.sql` |
| 规则式实体识别 | `mempalace/entities.py`、`entity_detector.py`、`entity_registry.py` |
| 过道(厢房内共现) | `mempalace/hallways.py` |
| 可插拔后端(RFC 001) | `mempalace/backends/`(`base.py`/`chroma.py`/`sqlite_exact.py`/`qdrant.py`/`pgvector.py`/`registry.py`) |
| MCP 服务(手搓 JSON-RPC) | `mempalace/mcp_server.py` |
| 记忆动力学(Hebbian/Ebbinghaus/Cepeda) | `mempalace/dynamics.py` |
| 配置系统 | `mempalace/config.py` |
| 设计理念 / AAAK / v4 动机 | `MISSION.md`、`docs/CLOSETS.md`、`docs/virtual-line-numbering.md` |

> 本文为《Memory in Action》系列的一章,基于源码撰写,仅供技术学习与研究。所有代码常量、路径与行号以 mempalace commit `da5a48c` 为准,后续版本可能演进。
