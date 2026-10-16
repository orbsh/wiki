# 记忆架构选型

*Agent 记忆系统的通用架构分析与方案对比：Surface/Engine 两层、计算时机光谱、注入方式、外部开源方案。本项目实现设计见 [Krystallizer](krystallizer.md)，图谱概念设计见 [图谱化记忆](graph-memory.md)，Agent 组件架构（turn 模型、压缩、注入的架构语义）见 [无状态 Agent 架构](stateless-agent-architecture.md)。*

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
| **框架适配** | 适配层接口（push_user_message / get_context / write_assistant_message） | _AgnoAgentWrapper 适配 Agno |

### Engine

| 决策点 | 选项 | 代表方案 |
|:--|:--|:--|
| **提取** | flat KV（content + category + tags） / graph triples（subject + predicate + object） | skillforge 当前用 flat，图谱化用 triples |
| **存储** | SQLite / SurrealDB / Postgres / 三存储分离 | agentmemory 用 SQLite，skillforge 用 SurrealDB |
| **检索** | 向量 / BM25 / 图遍历 / RRF 融合 / 多策略路由 | agentmemory 向量+关键词，skillforge HNSW+BM25+RRF，cognee 9 种策略 |

计算时机光谱的完整展开（写时/读时代表方案、光谱本质、属性图定位）见 [图谱化记忆](graph-memory.md)。

## 二、Surface 设计要点

Surface 的完整架构语义——Prefix Checkpoint 机制、记忆控制框架、六种触发方式、checkpoint+增量注入、历史纯净性——属于 Agent 架构层，独立成篇见 [无状态 Agent 架构](stateless-agent-architecture.md)。分层视角的要点：

| 决策点 | 选项 | 代表方案 |
|:--|:--|:--|
| **触发时机** | auto hooks（每轮系统自动） / 手动调用（Agent 判断） / 阈值触发 | agentmemory 用 hooks，skillforge 用阈值 |
| **注入方式** | 全量注入 / top-K 检索注入 / checkpoint + 增量 | Hermes 全量，agentmemory top-K，skillforge checkpoint |
| **框架适配** | 适配层接口（push_user_message / get_context / write_assistant_message） | _AgnoAgentWrapper 适配 Agno |

注入方式中 **checkpoint + 增量** 的 cache 友好度最高（checkpoint 不可变、可永久缓存），是本架构的默认选择。尾提示词（临时注入、用后即弃、不写 session）的缓存旁路机制见 [缓存树和尾提示词优化](tail-prompt-optimization.md)。

## 三、Engine 设计

### 提取

从对话到记忆的转换，两种格式：

| 格式 | 结构 | 适用场景 |
|:--|:--|:--|
| **flat KV** | content + category + importance + tags | 图谱化之前的长期记忆形态（文本行）；图落地后撤掉——文本记忆没有逐条身份，按条更新、加权、枚举都做不了 |
| **graph triples** | 定长 8 元组：head/tail 各 `[值, 类型, 属性]`，关系 `[关系名, 属性]` | 有关系结构的知识，图谱化演进方向 |

两种格式可以在同一次 LLM 调用中同时输出（双层输出）：

```
100 条消息 → [LLM 单次调用]
                ├→ checkpoint 摘要（注入 prompt）
                └→ flat memories / graph triples（写入存储）
```

提取时机方案（并行提取 → 纯追加指令 → 混入 turn → Prefix Checkpoint 并行旁路）的演进记录完整保留在 [Krystallizer](krystallizer.md)。当前方案即 Prefix Checkpoint：压缩从不可变前缀分叉并行旁路，主分支照常回答，用户零感知。

### 提取规格：三轴分离

写压缩指令（如 `memory_checkpoint` 的提示词）前，先把三个问题拆开——**提取什么、怎么调用、写入后如何更迭**。三者分属两层架构的不同决策点，绑在一起会互相牵制；解耦后每轴独立演进：提升提取规则只改规格，换缓存策略只改调用方式，改冲突策略只改写入语义。

**提取规格（Engine·提取）**：压缩指令回答"从对话抽出什么"。一份合格规格至少覆盖四件事：
- **指代消解**——摘要内代词一律回指到具体对象，杜绝断链；这是高保真的第一保证。
- **角色归属**——每条事实标明来源主体（`[User]`/`[Agent]`；第三方转发另设来源维度）。现有 flat KV 无归属维度，须显式补。
- **时序与状态更迭**——保留版本轨迹，后期推翻前期以 latest-wins 覆盖，最新需求最高权重。
- **决策依据**——不只留"最终结论"，还要留"为什么这么决定"；过度剔除工具中间输出会切断因果链。

**调用方式（Surface·触发与注入）**：规格不决定经济性，调用方式才决定。压缩须跑在 [Prefix Checkpoint](#prefix-checkpoint) 的**原地生成**上（历史已在上下文缓存），不得把 `{RAW_DIALOGUE}` 原样重发给一次独立调用——后者缓存命中率 0%，即传统压缩。

**写入语义（Engine·存储）**：状态更迭必须"数据上生效"，而非"文本上的更强"。跨多个 checkpoint 后"当前最新状态"要为真，须按 session/实体**物化 latest-wins 的当前状态**（由 DB 派生，类比任务完成判定），而非每次新 checkpoint 在文本里重述。规格须先声明：append-only 不可变 checkpoint，还是逐键覆盖。

**越界即返工**：三处常见混界——
- TODO 不进记忆摘要——属独立任务表（`tasks`/`task_items`，属独立任务表（`tasks`/`task_items`），完成判定归 DB），完成判定归 DB；
- 实体与偏好同一种东西——都是原子事实（偏好就是 `user PREFERS X`），不存在「两套提取模式」；flat 只是图谱化之前的载体，不是偏好的固定桶；
- 无验证层——压缩错误累积是复亏（见「合成闭环的缺口」节），须留"提取对不对"的校验台阶。

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

## 四、外部方案对比

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

| 维度 | agentmemory | cognee | Krystallizer（前身 skillforge mem） |
|:--|:--|:--|:--|
| **触发** | auto hooks（每轮） | 手动 remember() | 阈值触发 + 手动调用 |
| **提取** | iii-engine 压缩（黑盒） | LLM entity extraction | LLM 双层输出 |
| **存储** | SQLite + 向量 | 三存储分离 | SurrealDB（KV + 向量 + 全文） |
| **检索** | 向量 + 关键词 | 9 种策略路由 | HNSW + BM25 + RRF |
| **注入** | 按需检索 | GRAPH_COMPLETION 调 LLM | checkpoint + 增量 |
| **cache 友好** | 差（每轮变化） | 差（调 LLM） | 高（checkpoint 固定） |
| **框架绑定** | MCP | Python SDK + MCP | 无（适配层隔离） |
| **LLM 开销** | 低（压缩可配置） | 高（ingest 每 chunk） | 低（100 条才调一次） |


## 五、参考

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

agentmemory 和 cognee 通过 MCP 适配商业 IDE，是"政治性妥协"。记忆层走 MCP 可接受（低频操作），工具执行层走 MCP 不可接受（高频操作）。Krystallizer 前身（skillforge mem）用框架适配层（_AgnoAgentWrapper）替代 MCP，进程内调用无网络开销。

### 合成闭环的缺口

理想的记忆闭环：捕获 → 压缩 → 冲突去重 → 持久化 → 技能固化。

**关键断裂：验证层缺失。** 每一步都可能引入错误——压缩丢失上下文、反思误判事实、冲突检测漏判。没有验证层，错误累积不是复利，是复亏。三个现成解法各押一端：agentmemory 用 hooks 解决触发但绑定 MCP；cognee 用 improve() 部分解决进化但 LLM 成本重；skillforge 用人工审查保证质量但牺牲自动化。三难尚未被开源方案真正解决。

**落地方案（2026-08）：人审门控的做梦沉淀。** 把三难拆成"自动生成 + 人审操作 + 驳回回流"三段，避开三端各自代价：每晚离线做全局社区检测，对 SKILL 主体的边界提 merge/relink 操作集（自动、便宜、写时重摊销）；管理者以 diff 审核**操作**而非最终文本（准确度兜底，且只审低频操作、不陪跑每条记忆）；被拒操作写回做梦输入避免重复提出（闭环反馈）。完整设计见 [图谱化记忆](graph-memory.md) §「做梦沉淀」。

## 六、总结

### 当前状态

skillforge 已实现两层架构的 Surface（阈值触发 + checkpoint 注入 + _AgnoAgentWrapper 适配）和 Engine 的基础形态（flat KV 提取 + HNSW/BM25/RRF 检索 + SurrealDB 存储），实现现状已迁移至 Krystallizer（记忆核心独立为 krystallizer 项目）。

### 演进方向

| 阶段 | 改什么 | 不改什么 |
|:--|:--|:--|
| **图谱化** | Engine：flat KV → graph triples，检索增加图遍历 | Surface（触发、注入、适配）不动 |
| **框架迁移** | Surface：重写 _AgnoAgentWrapper（~20 行三步调用） | Engine（WorkingMemory、MemoryManager、MemoryTools）不动 |
| **权重系统** | Engine：增加 read_count/write_count 追踪和排序 | Surface不动 |
| **后台压缩** | Surface：后台进程检测空闲，主动触发压缩 | Engine不动 |

### 设计原则

1. **Surface 和 Engine 分离**：换框架只改Surface，换存储/检索只改Engine
2. **mem 模块厚，agent 适配层薄**：所有记忆逻辑封装在 WorkingMemory，适配层只做三步调用
3. **LLM 调用最小化**：100 条以内无感知，压缩走并行旁路（共享前缀缓存，不占关键路径、不混入回答 turn）
4. **框架无关**：记忆逻辑不依赖任何特定 Agent 框架
5. **渐进式演进**：当前 flat KV 够用就用 flat KV，需要图结构时再升级

---

## 交叉引用

- **[无状态 Agent 架构](stateless-agent-architecture.md)**：Surface 机制的架构篇——Prefix Checkpoint 详解、记忆控制、触发时机、注入方式、压缩并行旁路。
- **[Krystallizer](krystallizer.md)**：本项目记忆系统的实现设计——会话控制原语、full/assist 模式、提取三方案演进、事实序列化、skillforge 实现现状。
- **[图谱化记忆](graph-memory.md)**：图谱的记忆概念设计——计算时机光谱详述、双层模型、聚簇策略、权重系统、涌现式 Skill、做梦沉淀。
- **[缓存树和尾提示词优化](tail-prompt-optimization.md)**：尾提示词的缓存旁路机制。
- **[Agent 复利](agent-compound-interest.md)**：跨会话累积的价值——记忆是复利资产的载体。
