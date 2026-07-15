# 记忆架构设计

跟 AI 聊天时，每说一句话，AI 都要从头读一遍之前的所有对话。对话越长，AI 读得越慢、越贵。

我们的做法：把旧对话压缩成一段摘要固定住，后面新对话直接追加。因为摘要不动（前缀不变），AI 不用重新理解旧内容——就像看书翻到夹了书签的那页，不用从第一页重新读。这叫 **Prefix Checkpoint**。

这个机制还让记忆系统能完全控制 AI 的行为：通过在对话末尾追加指令，驱动 AI 调用记忆工具完成存储和整理，而 AI 本身不需要任何改造。详见 [Surface 设计](#二surface设计)。

## 一、两层架构

记忆系统分为**Surface**和**Engine**。Surface与 Agent 框架耦合，Engine框架无关。

```
┌─────────────────────────────────┐
│         Surface（框架相关）         │
│  触发时机 │ 注入方式 │ 框架适配   │
├─────────────────────────────────┤
│         Engine（框架无关）         │
│  提取     │ 存储     │ 检索      │
└─────────────────────────────────┘
```

| 层 | 职责 | 演进时改什么 |
|:--|:--|:--|
| **Surface** | 什么时候捕获、怎么注入 prompt、怎么适配 Agent 框架 | 换框架时改 |
| **Engine** | 对话→记忆的转换、存储格式、检索算法 | 换存储/检索策略时改 |

分层的价值：**现有方案（agentmemory、cognee）的组件耦合导致换任何一层都要动其他层**。分层后每层可独立演进——从 KV 迁移到图存储只改Engine，换框架只改Surface。

### Surface

| 决策点 | 选项 | 代表方案 |
|:--|:--|:--|
| **触发时机** | auto hooks（每轮系统自动） / 手动调用（Agent 判断） / 阈值触发 | agentmemory 用 hooks，skillforge 用阈值 |
| **注入方式** | 全量注入 / top-K 检索注入 / checkpoint + 增量 | Hermes 全量，agentmemory top-K，skillforge checkpoint |
| **框架适配** | 适配层接口（get_session_messages / add_message） | SnapshotSessionDB 适配 Agno |

### Engine

| 决策点 | 选项 | 代表方案 |
|:--|:--|:--|
| **提取** | flat KV（content + category + tags） / graph triples（subject + predicate + object） | skillforge 当前用 flat，图谱化用 triples |
| **存储** | SQLite / SurrealDB / Postgres / 三存储分离 | agentmemory 用 SQLite，skillforge 用 SurrealDB |
| **检索** | 向量 / BM25 / 图遍历 / RRF 融合 / 多策略路由 | agentmemory 向量+关键词，skillforge HNSW+BM25+RRF，cognee 9 种策略 |

### 计算时机光谱

Engine的核心区别在**计算发生在哪个阶段**：

```
写时重 ─────────────────────────────────── 读时重

LLM Wiki        KG (属性图)       KV 模式         向量模式
                graph-memory      skillforge      agentmemory
                SurrealDB graph   SurrealDB KV    SQLite + vector

写: LLM 整理/嵌入/拓扑推断    写: 简单切分/存储
读: 直接载入/遍历              读: 匹配/排序/向量搜索
```

光谱位置取决于使用方式，不取决于存储产品。SurrealDB 的多模型架构覆盖全部区间。

---

## 二、Surface设计

### Prefix Checkpoint

跟 AI 聊天时，每说一句话，AI 都要从头读一遍之前的所有对话。对话越长，AI 读得越慢、越贵。

传统做法是"记住最近 N 条"——但每次新消息来了，原来的 N 条就变了（最旧的被挤出去，最新的被加进来），AI 相当于每次都要重新读一遍。

Prefix Checkpoint 的做法不同：

1. 对话积累到一定长度（比如 100 条），AI 自己总结一段摘要："用户在讨论项目架构，决定用 SurrealDB……"
2. 这段摘要**固定下来，永远不变**。后续所有对话都在它后面追加。
3. 因为开头不动，AI 可以"跳过"已经理解的部分，只处理新消息——就像看书翻到夹了书签的那页，不用从第一页重新读。

摘要就是"前缀"（Prefix），固定不变；"Checkpoint"是存档点。每次新存档后，旧存档清空，重新累积。

**为什么传统压缩无法利用缓存**：传统做法是把 100 条消息发给一个单独的模型（或单独发起一次 API 调用）来生成摘要。这个调用和之前的对话没有缓存关系——即使你用同一个模型，它看到的是一段全新的文本，KV cache 从零开始计算，缓存命中率 0%。

Prefix Checkpoint 不同：100 条消息已经在缓存中（之前每轮对话都在用）。压缩提示词只是在末尾追加了一小段新文本。模型处理时，前面 99% 的 token 直接走缓存，只有最后的提示词 + 用户问题是新计算。

```
传统压缩：
  [100 条消息] → 新 API 调用 → 全部重新计算 → 输出摘要
  缓存命中率：0%

Prefix Checkpoint：
  [100 条消息（缓存中）] + [压缩提示词 + 用户问题]
  缓存命中率：~99%
```

结果：又快又好，消耗少。快是因为缓存，好是因为用的是同一个强模型（不是小模型），消耗少是因为大部分 token 不需要重新计算。而且不需要改造 Agent loop——压缩通过正常 tool call 完成。

**"辅助模型"≈传统模式**：能配置独立辅助模型的 Agent 系统，本质上就是传统压缩——需要单独发起一次 LLM 调用，无法利用缓存。所谓"快速模型"只是相对的：100 条消息发过去，小模型也要从头算一遍，延迟可能只少 30-50%，但质量差很多。Prefix Checkpoint 连辅助模型都不需要——压缩就是 Agent 正常回答的一部分。

### 记忆控制

Prefix Checkpoint 不仅是一个缓存优化机制，更是一个**记忆控制框架**。

AI 每次回复前都会读一遍完整的对话。如果我们想让 AI 做某件事（比如"把刚才聊的偏好记下来"），只需要在对话末尾加一句话就行了。

具体来说，当对话累积到 100 条时，记忆系统会在用户下一次提问时，在提问前面插入一段指令：

```
[之前的所有对话]           ← AI 已经理解了（缓存中）
[记忆系统插入的指令]        ← "分析以上对话，提取用户的偏好和事实，
                              调用 memory_store 工具存储，
                              然后回答用户的问题"
[用户的问题]               ← 用户真正想问的
```

AI 看到这段指令后，会自动调用记忆工具完成存储，然后正常回答用户。用户完全无感——他只是问了一个问题，AI 回答了，但后台顺便完成了记忆整理。

**控制机制**：Agent 是被动的——它只看到 prompt 和可用工具，按照 prompt 中的指令调用工具。记忆系统通过控制两件事实现完全控制：
1. **注入什么** — prompt 末尾追加的提示词（决定 Agent 做什么）
2. **提供什么工具** — Agent 可调用的工具列表（决定 Agent 能做什么）

**这使得记忆系统可以做到**：
- 控制压缩时机（阈值到达时注入压缩指令）
- 控制提取内容（提示词指定提取偏好/事实/决策）
- 控制存储方式（工具决定写入哪个表、什么格式）
- 不改 Agent loop — 所有控制通过 prompt + 工具实现

**类比**：Agent 是一个"没有主见的执行者"——它有能力（LLM 推理 + 工具调用），但没有意图。记忆系统通过 prompt 注入意图，Agent 负责执行。这和操作系统的"系统调用"类似：内核提供能力（syscall），用户态程序决定何时调用。

### 触发时机

| 方式 | 机制 | 优点 | 缺点 |
|:--|:--|:--|:--|
| **主动调用** | Agent 判断"这条值得记住"时调 memory_store | 精准，只存有价值的 | 依赖 Agent 判断力，可能遗漏 |
| **阈值触发** | 消息累积到 N 条时 LLM 批量提取 | 低频调用，LLM 开销可控 | N 条内未压缩（但原始消息仍在 session 中，Agent 可直接读取） |

### 主动触发

Prefix Checkpoint 的记忆控制不仅用于压缩，还用于**主动记忆**——Agent 在对话过程中判断"这条值得记住"时，主动调用 `memory_store` 工具。

触发方式是**用户要求或暗示**：
- 用户明确说"记住这个"、"以后都这样做"
- 用户表达了偏好、习惯、决策（Agent 判断值得长期保存）
- Agent 发现了重要的事实或上下文

Agent 不主动捕捉执行过程中产生的知识——它用 LLM 的判断力筛选，只存储有价值的信息，精准但依赖判断力。

这是记忆系统的两种触发方式，互补：

| 方式 | 时机 | 谁决定 | 存什么 |
|:--|:--|:--|:--|
| **Prefix Checkpoint** | 阈值到达时 | 记忆系统 | 对话摘要 + 提取的偏好/事实 |
| **主动触发** | 对话过程中 | Agent | Agent 判断值得记住的 |

### 注入方式

| 方式 | 机制 | cache 友好度 |
|:--|:--|:--|
| **全量注入** | 每轮把所有历史注入 prompt | 差（内容每轮变化，KV cache 失效） |
| **top-K 检索** | 每轮检索相关记忆注入 | 中（检索结果可能变化） |
| **checkpoint + 增量** | 不可变 checkpoint + 最近 N 条消息 | 高（checkpoint 固定，可永久缓存） |

#### checkpoint + 增量机制

类似 event-sourcing 的快照模式。消息逐条写入 session_messages 表，达到阈值时压缩为 checkpoint（写入 session_checkpoints 表），后续所有未压缩消息即为增量：

```
msg_001 ... msg_100    ← 消息追加到 session_messages 表
              ↓ 达到阈值（100 条）
ckpt_0: summary="用户讨论了项目架构，决定用 SurrealDB..."
              ↓ 写入 session_checkpoints 表，不可变
msg_101 ... msg_150    ← 全部是增量（当前所有未压缩消息）
              ↓ 再次达到阈值
ckpt_1: summary="..."
msg_151 ...            ← 新的增量
```

**读取流程**：

消息一条条追加到 prompt 中（不是每次重新拼接）。LLM API 的多轮对话机制天然缓存前面所有 tokens：

```
Turn 1: [ckpt] + [msg_101]
Turn 2: [ckpt] + [msg_101] + [msg_102]       ← 前面的 tokens 走 KV cache
Turn 3: [ckpt] + [msg_101] + [msg_102] + [msg_103]
...
Turn N: 达到阈值 → 融入下一个 Agent turn 完成压缩
Turn N+1: [ckpt_1] + [用户问题 X] + [Agent 回答 Y]
```

**写入流程**：
1. 每条消息追加到 session_messages 表
2. 检查距上次 checkpoint 的消息数是否 >= 阈值
3. 达到阈值 → 标记需要压缩 → 下一个用户提问时，将压缩提示词作为普通消息追加到 prompt 末尾

**Prefix Checkpoint**：阈值到达后，下一个用户提问时，记忆系统在 prompt 末尾追加一条特殊消息（不是 system prompt）：

```
KV cache 中（不变）：
  [ckpt] + [msg_101..msg_150]

新追加的消息（唯一未缓存的部分）：
  "先回答用户的问题，然后分析以上对话，调用 memory_store
   提取偏好/事实，标记 checkpoint"

Agent 一次 turn 完成三件事（注意顺序）：
  1. "Y"                              ← 先回答用户问题（用户体验优先）
  2. tool_call: memory_store(...)    ← 再做记忆操作
  3. tool_call: checkpoint(...)       ← 最后标记 checkpoint
```

**历史纯净性**：压缩提示词是临时的——只存在于当次 turn 的 prompt 中，从不写入 session。session 里永远是纯净的历史：

```
session 中存储的：
  [checkpoint_1] + [用户原始消息 X] + [Agent 回答 Y]

而不是：
  [checkpoint_1] + [压缩提示词 + 用户原始消息 X] + [Agent 回答 Y]
```

即使 prompt 中为了触发压缩而修改了用户消息（在其末尾追加指令），turn 结束后也需要将 session 中的该条消息**还原为原始内容**。这保证了：
- 历史可重放——任何时候重新加载 session，内容都是真实的对话
- 无伪指令——session 中不会残留"分析以上对话，调用 memory_store"这样的系统指令
- 逻辑连贯——checkpoint + 原始消息 + 回答，构成完整的因果链

**顺序很重要**：必须先回答用户问题，再做记忆操作。用户问了一个问题，如果先看到一堆工具调用在跑，体验很差。提示词中明确要求"先回答，再处理记忆"——用户看到的第一个输出就是答案，记忆操作是"顺便"完成的。

**为什么这样设计**：
- **不改 Agent loop** — 压缩通过正常 tool call 完成，记忆系统完全控制
- **不单独调 LLM** — 复用 Agent 的正常 turn，answer + compress 共享 KV cache
- **cache 天然命中** — 历史部分全部缓存，只有压缩提示词 + 用户问题是新 token
- **用户体验无感** — 正常回答用户，同时后台完成压缩和记忆提取
- **历史自动清理** — turn 结束后旧消息不再需要，历史变为 `checkpoint_1 + X + Y`

**为什么 cache 友好**：checkpoint 一旦写入不可变，后续所有 turn 共享同一个 checkpoint 文本。LLM API 的 prompt caching 将 checkpoint 部分缓存在 GPU 显存中，只有增量部分每次变化。随着增量消息增多、下一次 checkpoint 触发，增量被压缩为新的固定摘要，缓存再次命中。

**与 Agno 滑动窗口的对比**：Agno 的 `add_history_to_context` 每次从 DB 读最近 N 条完整消息。随着新消息到来，N 条的组成不断变化（旧的被挤出、新的被加入），KV cache 每轮失效。checkpoint 模式下，历史被压缩为固定摘要，只有未压缩的增量部分变化，cache 持续命中。

### 框架适配

适配层隔离框架差异，核心接口只有两个方法：

```python
class SessionAdapter:
    def get_session_messages(session_id) → list[dict]  # checkpoint + 所有增量
    def add_message(session_id, message)                 # 追加消息
```

`get_session_messages` 返回 checkpoint + 后续所有未压缩消息，长度由阈值决定，不需要 limit 参数。

换框架时只重写适配层（几十行），Engine完全复用。

---

## 三、Engine设计

### 提取

从对话到记忆的转换，两种格式：

| 格式 | 结构 | 适用场景 |
|:--|:--|:--|
| **flat KV** | content + category + importance + tags | 简单偏好/事实，当前实现 |
| **graph triples** | subject + predicate + object + category | 有关系结构的知识，图谱化演进方向 |

两种格式可以在同一次 LLM 调用中同时输出（双层输出）：

```
100 条消息 → [LLM 单次调用]
                ├→ checkpoint 摘要（注入 prompt）
                └→ flat memories / graph triples（写入存储）
```

#### 提取时机：三种方案

**方案 A：每轮并行提取（⚠️ 过时）**

在 system prompt 中指示 LLM 同时输出用户回复和抽取三元组的函数调用，一次响应完成两件事。

问题：(1) 需要改造 Agent 框架；(2) 函数调用干扰 LLM 对用户回复的注意力；(3) 每轮引入新的函数调用 token，KV cache 命中率低。→ 详见 [图谱化记忆](graph-memory.md) §并行提取机制

**方案 B'：纯追加指令（⚠️ 过时，被 B 替代）**

阈值到达后，在 prompt 末尾追加压缩指令，LLM 直接输出 checkpoint + memories（不通过 tool call）。

```
Turn 1..N: 正常对话，[ckpt] + 增量逐条追加，KV cache 持续命中
              ↓ 达到阈值
Turn N+1:  prompt 末尾追加 "分析以上对话，输出 checkpoint 和 memories"
           → LLM 直接输出结果，指令本身不写入 session
Turn N+2:  新 checkpoint + 新增量，重新开始
```

优势：LLM 注意力完全集中在压缩任务上（不需要同时回答用户），提取质量可能更高。
问题：(1) 需要改 Agent loop（识别特殊输出并处理）；(2) 压缩和回答分开，两次 LLM 调用；(3) 压缩时机不在记忆系统控制下。

**方案 B：Prefix Checkpoint（当前方案）**

替代方案 B'。压缩不是单独的操作，而是融入 Agent 的正常对话 turn——下一个用户提问时，在 prompt 末尾追加压缩提示词（不是 system prompt）。Agent 一次 turn 同时完成提取和回答：

```
Turn 1..N: 正常对话，[ckpt] + 增量逐条追加，KV cache 持续命中
              ↓ 达到阈值
Turn N+1:  用户提问 X
           prompt 末尾追加: "分析以上对话，调用 memory_store 提取记忆，
                           标记 checkpoint，然后回答：{X}"
           → Agent 一次 turn: tool_call(memory_store) + tool_call(checkpoint) + 回答 Y
Turn N+2:  历史变为 [ckpt_1] + [X] + [Y]，重新开始
```

优势：(1) 不改 Agent loop；(2) answer + compress 共享 KV cache；(3) 用户体验无感；(4) 压缩完全由记忆系统控制（通过 tool call）。

| 维度 | A（并行提取） | B'（纯追加指令） | B（融入 Agent turn） |
|:--|:--|:--|:--|
| Agent 改造 | 需要 | 需要 | 不需要 |
| LLM 调用 | 每轮 | 压缩单独一次 | 复用正常 turn |
| 注意力 | 分散 | 集中在压缩 | 分散（但 tool call 机制保障） |
| 控制权 | Agent | 不确定 | 记忆系统 |

### 存储

| 后端 | 特点 | 适用 |
|:--|:--|:--|
| **SQLite** | 嵌入式，零部署 | 个人工具、本地 Agent |
| **SurrealDB** | KV + 向量 + 全文 + 图，多模型 | 企业服务、多用户 |
| **Postgres** | pgvector + SQL + graph backend | 已有 PG 基础设施 |

### 检索

| 算法 | 机制 | 确定性 |
|:--|:--|:--|
| **向量（HNSW）** | 语义相似度 | 概率性 |
| **BM25** | 关键词匹配 | 确定性 |
| **图遍历** | 实体关系链 | 确定性 |
| **RRF 融合** | 多路排序融合 | — |

RRF（Reciprocal Rank Fusion）是融合多路检索结果的标准做法——各路独立排序，按排名倒数加权合并。不依赖分数归一化，鲁棒性强。

---

## 四、skillforge 实现

### 当前方案（Phase 2.5）

在光谱中间偏右——比 agentmemory 检索质量高、cache 友好，比 cognee 部署轻、LLM 成本低。

```
写时重 ─────────────────────────────────── 读时重

cognee          skillforge        agentmemory
(知识图谱)       (当前实现)         (轨迹压缩)
SurrealDB graph  SurrealDB KV      SQLite + vector
```

#### Surface

| 决策点 | 选择 |
|:--|:--|
| 触发时机 | 阈值触发（100 条）+ Agent 手动调用 |
| 注入方式 | checkpoint + 增量（SnapshotSessionDB） |
| 框架适配 | SnapshotSessionDB（适配 Agno session_db） |

#### Engine

| 决策点 | 选择 |
|:--|:--|
| 提取 | flat KV（LLM 双层输出：checkpoint + memories） |
| 存储 | SurrealDB（session_messages + session_checkpoints + memories） |
| 检索 | HNSW + BM25 + RRF 三路融合 |

#### 核心接口

三个：`store` / `search` / `forget`。辅助接口 `list_all`（导出/备份）和 `get_stats`（仪表盘/运维）不参与 Agent 对话流程。

| 方法 | 用途 |
|:--|:--|
| `store(content, category, importance, tags)` | 存储记忆（embedding 由 SurrealDB 内部生成） |
| `search(query, top_k)` | 混合检索 |
| `forget(memory_id)` | 删除记忆 |

#### 关键设计决策

| 决策 | 原因 |
|:--|:--|
| 100 条以内 LLM 无感知 | 不膨胀 prompt，不破坏 KV cache |
| Checkpoint 不可变 | 写入后固定，可永久缓存 |
| 不依赖框架 MemoryManager | 黑盒提取不可控、框架绑定、每 turn 额外 LLM 调用 |
| Embedding 在 SurrealDB 内部生成 | Python 侧零传输零存储 |
| 不做过期清理 | 存储不是瓶颈，向量检索天然让不相关旧记忆排在后面 |

#### 框架迁移

SnapshotSessionDB 是唯一与框架耦合的部分（实现 `get_session_messages` + `add_message`）。MemoryManager 和 MemoryTools 完全复用。迁移成本：几十行适配代码。

---

## 五、图谱化演进

Engine的演进方向：提取从 flat KV 升级为 graph triples，检索增加图遍历。Surface不变。

```
当前：  flat KV + 向量/BM25/RRF
        ↓
演进：  graph triples + 向量/图遍历/RRF
```

### 提取变化

LLM 双层输出的第二层从 flat memories 变为 graph triples：

```
当前输出：
  memories: [{"content": "...", "category": "fact", "importance": 3, "tags": [...]}]

图谱化输出：
  triples: [{"subject": "...", "predicate": "...", "object": "...", "category": "fact|rule|logic|preference"}]
```

一次 LLM 调用双层输出的模式不变，只是第二层的输出格式从 flat 变成 structured。

### 检索变化

在现有 HNSW + BM25 + RRF 基础上增加图遍历：

```
当前：query → 向量检索 + BM25 → RRF 融合
演进：query → 向量检索 + BM25 + 图遍历（多跳） → RRF 融合
```

### SurrealDB 的支撑

SurrealDB 的多模型架构让迁移自然——graph record 和 KV record 共存，检索时根据 query 类型路由。不需要换存储引擎。

### 详细设计

图谱化的完整设计（聚簇策略、权重系统、双层模型）见 [图谱化记忆](graph-memory.md)。

---

## 六、外部方案对比

### agentmemory（24k★，TypeScript + Rust）

**定位**：跨 Agent 的持久记忆层，通过 MCP 适配商业 IDE。

| 维度 | 实现 |
|:--|:--|
| 存储 | SQLite + 向量索引（`all-MiniLM-L6-v2` 本地 embedding） |
| 触发 | 12 个 auto hooks（每轮自动捕获） |
| 检索 | 向量 + 关键词（二元） |
| 注入 | 按需检索注入（92% token 节省） |

**优势**：MCP 通用性（任何支持 MCP/hook 的 Agent 都能接入）；auto hooks 零侵入。

**局限**：绑定 MCP 协议（端口常驻）；embedding 质量上限（小模型）；捕获内容本身已在 session 里（hooks 的增量价值有限）；无图结构、无权重系统。

### cognee（~22k★，Python）

**定位**：知识图谱 memory platform，ingest 任意格式数据构建 knowledge graph。

| 维度 | 实现 |
|:--|:--|
| 存储 | 三存储分离（relational + vector + graph） |
| 触发 | 手动 remember() |
| 检索 | 9 种策略路由（graph / vector / lexical / temporal / cypher 等） |
| 提取 | LLM 逐 chunk 提取 entity + relationship |

**优势**：图原生（多跳推理）；多策略检索路由；Postgres 统一栈（v1.0）；`improve()` + feedback_weight 自我改进。

**局限**：LLM 成本重（每 chunk 一次调用）；部署复杂（三存储）；无事实级冲突解决（旧事实和新事实并存）；无组织层（聚簇预聚合，靠 LLM 在检索时临时拼凑）。

### 对比矩阵

| 维度 | agentmemory | cognee | skillforge |
|:--|:--|:--|:--|
| **触发** | auto hooks（每轮） | 手动 remember() | 阈值触发 + 手动调用 |
| **提取** | iii-engine 压缩（黑盒） | LLM entity extraction | LLM 双层输出 |
| **存储** | SQLite + 向量 | 三存储分离 | SurrealDB（KV + 向量 + 全文） |
| **检索** | 向量 + 关键词 | 9 种策略路由 | HNSW + BM25 + RRF |
| **注入** | 按需检索 | GRAPH_COMPLETION 调 LLM | checkpoint + 增量 |
| **cache 友好** | 差（每轮变化） | 差（调 LLM） | 高（checkpoint 固定） |
| **框架绑定** | MCP | Python SDK + MCP | 无（适配层隔离） |
| **LLM 开销** | 低（压缩可配置） | 高（ingest 每 chunk） | 低（100 条才调一次） |

---

## 七、参考

### CodeGraph：代码结构层

本地语义代码知识图谱，代码修改时自动同步图谱，AI 查询时直接遍历。在光谱中属于"写时重"——inotify 触发自动建图，不需要人工维护。

与记忆系统互补：CodeGraph 回答"代码怎么组织的"（what），设计文档回答"为什么这样组织"（why），记忆系统回答"用这段代码学到了什么"（experience）。

### 纯记忆层技术全景（2026 开源）

四象限拓扑：

```
[象限 I：痕迹捕获]          [象限 II：情景事件/时序衰减]
  agentmemory (iii-engine)    Mnemosyne (Rust, Ebbinghaus)
  职责：捕获-压缩-检索闭环     HippoRAG (PPR 多跳联想)
───────┼──────────────────────────┼──────────────────
       │                          │
[象限 III：语义事实/冲突去重]   [象限 IV：向量/图谱引擎底座]
  Mem0 (实体-关系图谱)          sqlite-vec (纯 C, 单文件)
  cognee (知识图谱+多策略检索)  KuzuDB (嵌入式图数据库)
  职责：长效事实维护             职责：嵌入式免部署存储
```

### MCP 与记忆层

agentmemory 和 cognee 通过 MCP 适配商业 IDE，是"政治性妥协"。记忆层走 MCP 可接受（低频操作），工具执行层走 MCP 不可接受（高频操作）。skillforge 用框架适配层（SnapshotSessionDB）替代 MCP，进程内调用无网络开销。

### 合成闭环的缺口

理想的记忆闭环：捕获 → 压缩 → 冲突去重 → 持久化 → 技能固化。

**关键断裂：验证层缺失。** 每一步都可能引入错误——压缩丢失上下文、反思误判事实、冲突检测漏判。没有验证层，错误累积不是复利，是复亏。agentmemory 用 hooks 解决触发但绑定 MCP；cognee 用 improve() 部分解决进化但 LLM 成本重；skillforge 用人工审查保证质量但牺牲自动化。三难尚未被开源方案真正解决。

---

## 八、总结

### 当前状态

skillforge 已实现两层架构的Surface（阈值触发 + checkpoint 注入 + SnapshotSessionDB 适配）和Engine的基础形态（flat KV 提取 + HNSW/BM25/RRF 检索 + SurrealDB 存储）。核心接口三个：store / search / forget。

### 演进方向

| 阶段 | 改什么 | 不改什么 |
|:--|:--|:--|
| **图谱化** | Engine：flat KV → graph triples，检索增加图遍历 | Surface（触发、注入、适配）不动 |
| **框架迁移** | Surface：重写 SnapshotSessionDB（几十行） | Engine（MemoryManager、MemoryTools）不动 |
| **权重系统** | Engine：增加 read_count/write_count 追踪和排序 | Surface不动 |

### 设计原则

1. **Surface 和 Engine 分离**：换框架只改Surface，换存储/检索只改Engine
2. **LLM 调用最小化**：100 条以内无感知，压缩融入 Agent 正常 turn，不单独调 LLM
3. **框架无关**：记忆逻辑不依赖任何特定 Agent 框架
4. **渐进式演进**：当前 flat KV 够用就用 flat KV，需要图结构时再升级
