# Wiki 索引

按主题分类的文档索引。项目特定文档见 [projects/](projects/index.md)。

## 记忆系统

| 文档 | English | 内容 |
|:--|:--|:--|
| [图谱化记忆](graph-memory.md) | | **设计文档**：计算时机光谱、属性图定位、原子三元组、双层模型、聚簇策略、权重系统、编码场景 |
| [无状态 Agent 架构](stateless-agent-architecture.md) | | **架构总纲**：turn 即执行单位、组件四分（Prism/Gravity/Krystallizer/Effector）、Surface 设计（注入与触发）、压缩并行旁路、skill 涌现闭环、传输裁决 |
| [Krystallizer](krystallizer.md) | | **实现设计**：会话控制原语（branch/tail/summarize）、full/assist 双模式、提取机制三方案、KDL 序列化、存储实现、skillforge 实现现状 |
| [无状态 Agent 架构](stateless-agent-architecture.md) | | **架构设计**：turn 即执行单位、组件四分（Prism/Gravity/Krystallizer/Effector）、skill 涌现闭环、HTTP+SSE 传输裁决、与传统架构对照 |
| [记忆选型](agent-memory.md) | [Memory Architecture](agent-memory-en.md) | **选型综述**：Surface/Engine 两层、计算时机光谱、注入方式、外部开源方案对比、合成闭环缺口 |
| [Agent 复利](agent-compound-interest.md) | | 持久化如何改变 AI 工具本质：memory/skill/cron 的累积效应、跨项目联动 |

## Agent 架构与使用

| 文档 | English | 内容 |
|:--|:--|:--|
| [Agent 原则](agent-principles.md) | | AI Agent 通用设计原则：确定性/非确定性边界、反主流护城河、概率系统思维 |
| [Agent 使用模式](agent-usage-patterns.md) | | 人类侧交互技巧和审查习惯——决定 Agent 质量上限的不是模型，是使用方式 |
| [Hermes Agent 设计](hermes-agent-design.md) | | Hermes 特定的使用约定、配置细节和运维协议 |
| [AI Agent 选型](ai-agent-selection.md) | | 轻量 CLI Agent 选型调研——已定案：Hermes 主力 + jcode 救援，Open Interpreter 移除 |
| [多智能体批判](multi-agent-critique.md) | | 多 Agent 模式的技术批判 |
| [辩论验证](debate-validation.md) | | Prefix Checkpoint 方案的辩论验证过程：帕累托最优论证、层次隔离、揭穿 AI 幻觉 |
| [用户画像：Orbit](orbit-profile.md) | | Orbit（O）的偏好、习惯、技术立场 |
| [智力金字塔模型](intelligence-pyramid.md) | [Intelligence Pyramid](intelligence-pyramid-en.md) | 多级智力架构与卸载策略：LLM→ML→算法→代码的梯度平衡 |

## LLM 与认知

| 文档 | English | 内容 |
|:--|:--|:--|
| [LLM 基础](llm-fundamentals.md) | | 训练涌现智能、三层局限、认知框架 |
| [LLM 过度思考批判](llm-overthinking-critique.md) | | 表演拟人 vs 真正推理、草稿本纠偏 |
| [LLM 缓存破坏模式](llm-caching-destruction-patterns.md) | | Prompt Caching 生效条件、前缀稳定性、IDE/Agent 场景的缓存杀手 |
| [LLM 隐藏行为模式](llm-hidden-behavior-patterns.md) | | 模型未显式表达但影响输出的行为模式 |
| [缓存树和尾提示词优化](tail-prompt-optimization.md) | [Tail Prompt Optimization](tail-prompt-optimization-en.md) | 上下文缓存旁路分支机制，用于压缩/汇总/提取 |
| [认知心理学](cognitive-psychology.md) | | 认知科学框架在 AI Agent 设计中的应用 |

## 架构与基础设施

| 文档 | English | 内容 |
|:--|:--|:--|
| [AI 友好基础设施](ai-friendly-infrastructure.md) | | 声明式 OS、全链路排查、NixOS 为何最适合 AI |
| [架构选择](architecture-choices.md) | | 网关选择、仓库策略（Zot+WG）、Redis/数据库批判 |
| [统一数据层](unified-data-layer.md) | | SurrealDB 多模型引擎、SurrealQL 人体工程学、与 PG 的分层选型 |
| [湖仓研究](lakehouse-research.md) | | Iceberg/LanceDB/Delta Lake 选型 |
| [HelixDB vs LanceDB](helixdb-vs-lancedb.md) | | 对象存储上 AI 数据栈的两种路线对比 |
| [Aura 架构 §5](aura-architecture.md) | | Fjall + 独立元数据（单写无共识）分布式存算一体 |
| [Arrow HTAP 引擎](arrow-unified-htap-engine.md) | | Arrow 统一 HTAP 引擎设计 |
| [文件系统方案](filesystem-solution.md) | | 文件系统选型 |
| [拥塞控制设计](congestion-control-design.md) | | 网络拥塞控制 |
| [轻量 FaaS 架构](lightweight-faas-architecture.md) | | 轻量级函数即服务架构 |

## 批判文档

| 文档 | English | 内容 |
|:--|:--|:--|
| [Redis 批判](redis-critique.md) | | 网络延迟陷阱、单线程瓶颈、内存浪费 |
| [高并发批判](high-concurrency-critique.md) | | 「高并发」作为简中特有话语的解剖：词汇量纲缺失、面试循环繁殖、fpm 防御性焦虑 |
| [0→1 话语批判](innovation-discourse-critique.md) | | 「0→1 / 1→100」话术的解剖：用跨度冒充高度、应用遥遥领先的证伪、市场对创新课税、体量≠能力、应用工程≠计算机科学、可替代性判据 |
| [Nginx 批判](nginx-critique.md) | | 为何 Nginx 过时：多进程低效、静态配置、Lua 复杂性 |
| [MySQL 批判](mysql-critique.md) | | MySQL 的架构缺陷 |
| [等保标准批判](mlps-critique.md) | | 等保标准与现代安全实践的差异：云环境适配、密码轮换、防火墙、审计 |
| [MCP 批判](mcp-critique.md) | [MCP Over-Engineering](mcp-critique-en.md) | MCP 协议的政治性妥协与工程代价 |
| [Dify 批判](dify-critique.md) | | AI 应用领域的 Harbor 复制品：低代码 vs AI 范式矛盾 |
| [Harbor 批判](harbor-critique.md) | | 企业膨胀、CNCF 政治 vs 工程现实 |
| [阿里云批判](aliyun-critique.md) | | 阿里云的架构与商业问题 |
| [AI 编程乐观批判](ai-programming-optimism-critique.md) | | AI 编程的能力边界 |
| [Vim 运动批判](vim-movement-critique.md) | | Vim 运动模型的局限 |
| [为什么不写注释](why-not-write-comments.md) | | 代码注释的反主流观点 |

## 设计与语言

| 文档 | English | 内容 |
|:--|:--|:--|
| [Aura 架构](aura-architecture.md) | | Event Realm、摊位、事件组合原语、interface_schema |
| [长连接工程挑战](long-lived-connection-engineering.md) | | WS/WT 长连接的物理限制（背压、碎片化、连接倾斜）与解法 |
| [Aura Fluxora DevOps](aura-fluxora-devops.md) | | Aura 与 Fluxora 的 DevOps 集成 |
| [Flux 架构](flux-architecture.md) | | 响应式通信模式：从被动拉取到主动推送，Fluxora = Flux + Aura |
| [WASM 统一运行时](wasm-unified-runtime.md) | | WebAssembly 统一运行时架构 |
| [现代语言设计](modern-language-design.md) | | 语言设计趋势与分析 |
| [ECS 实体组件系统](entity-component-system.md) | | ECS 三层模型、数据驱动收益、网/树拓扑定位、Bevy 0.19 实践 |
| [Go, Zig 与反智主义](go-zig-anti-intellectualism.md) | | Go, Zig 与反智主义 |
| [嵌入式脚本语言](embedded-script-languages.md) | | 嵌入式脚本语言选型 |
| [Lambda 到硅](lambda-to-silicon.md) | | 从 Lambda 演算到硅片的计算模型演进 |
| [序列化协议分析对比](serialization-protocol-comparison.md) | | 序列化格式选型 |
| [文档格式对比](document-format-comparison.md) | | Markdown 与下一代纯文本标记语言：按笔记/排版/出版用途分轴对比 Neorg、Org-mode、Typst、AsciiDoc |
| [KDL vs 配置格式](kdl-vs-config-formats.md) | | 语义树 vs 数据网络：KDL 节点模式与 TOML/YAML/JSON/HTML 全景对比、工程坑点；KDL 作为配置生成中间层的复用价值 |
| [标签森林](tag-forest.md) | | 多棵树互斥维度组成的内容组织：situation/importance·urgency 评分树、隔离实例、邻接表+递归CTE查询 |

## 工作流与技能

| 文档 | English | 内容 |
|:--|:--|:--|
| [自动化 Skill 工作流](automation-skill-workflow.md) | | 浏览器操作→可复用 Hermes Skill 的逆向工程 |
| [NL2SQL 架构](nl2sql-architecture.md) | | OLTP 上做分析是架构设计错误：批判性分析 + 技能实现 |
| [Nushell 介绍](nushell-introduction.md) | | Nushell 入门 |
| [Loop 工程分析](loop-engineering-analysis.md) | | Loop 工程方法论框架 |
| [迭代闭环](iteration-closed-loop.md) | | 行动、复盘、转向：迭代闭环的结构骨架、心理阻塞与元能力 |
| [信息流精炼](infoflow-refinement.md) | | 信息流管道捕获→属性化→精炼→消费：NB 评分·标题党一致性检测、html→md·链接挖掘、RSS、聚合摘要/态势感知 |

## AI 时代思考

| 文档 | English | 内容 |
|:--|:--|:--|
| [AI 时代组织](ai-era-organization-individual.md) | | 约束-解分析方法论、联邦化、个体崛起 |
| [AI 时代商业模式](ai-era-business-models.md) | | AI 对商业模式的重塑 |
| [市场形态全谱系](market-form-spectrum.md) | | 服务/产品/创作者/资产/投机/中介六形态与个人适配、认知操作系统轴、天赋勘探提示词 |
| [AI 流动性](ai-liquidity-camp.md) | | AI 时代的流动性分析 |
| [Karpathy AI 编码方法论](karpathy-ai-coding-methodology.md) | | Karpathy 的 AI 编码实践 |
| [国内 AI 产业制度性缺陷](ai-industry-institutional-critique.md) | | 蒸馏/刷分现象、评审体制逆向选择、自由市场 vs 国家主导 |
| [AI 辅助视频制作与游戏引擎](ai-creative-production-workflow.md) | | AI 视频 10 秒魔咒、UE6/Bevy 路线、AI+Blender 工作流 |
| [3D 数据可视化](3d-data-visualization-aesthetics.md) | | 3D 作为叙事语言：物理流动/空间体积/纵深层级/材质语义 + 通用可视化问题的 3D 解法 |
| [3D 渲染库选型](3d-rendering-library-selection.md) | | 自定义空间展示的渲染库之争：three.js/three-d/Bevy/deck.gl API 风格 + wasm 体积实测 |

## 工具与环境

| 文档 | English | 内容 |
|:--|:--|:--|
| [Emacs 研究](emacs-research.md) | | Emacs 配置与使用 |
| [编辑器选型 2026](editor-selection-2026.md) | | 编辑器选型对比 |
| [区块编辑器架构](block-editor-architecture.md) | | 区块编辑器设计 |
| [Linux 桌面工作流](linux-desktop-workflow.md) | | 平铺 WM 到 COSMIC DE 的架构选择：外层堆叠+内层平铺、Nushell+Pop-Launcher 数据流 |
| [Linux SSD 优化](linux-ssd-optimization.md) | | SSD 性能优化 |
| [Debian 容器沙箱](debian-container-sandbox.md) | | 容器化沙箱方案 |
| [P2P 网格对比](p2p-mesh-networking-comparison.md) | | P2P 网状网络方案对比 |
| [无状态认证选型](stateless-auth-selection.md) | | 无状态认证方案 |
| [Gateway API 迁移](gateway-api-migration.md) | | Istio 到 Envoy HTTPRoute 迁移 |
| [浏览器键盘模式](keyboard-wm-browser-mode.md) | | Niri + chromium --app + CDP 的全键盘浏览器模式：信息流精炼定位、tag 森林组织、mudra 底座 |

## 其他

| 文档 | English | 内容 |
|:--|:--|:--|
| [Fractal](Fractal.md) | | 核心自包含文档 |
| [技术哲学](tech-philosophy.md) | | 应试工程 vs 范式转移 |
| [职场沟通](workplace-communication.md) | | 职场沟通方法 |
| [坐姿工作范式](seated-work-paradigm.md) | | 坐姿与工作方式 |
| [工程思维与工程素养](engineering-mindset-and-competence.md) | | 工程师双核心能力：认知决策模型（思维）与职业行为规范（素养） |
| [可视化迷思](visual-myth.md) | [The Myth of Visualization](visual-myth-en.md) | 可视化形式的结构性批判 |
| [图形语法与 AI 可视化](grammar-of-graphics.md) | | 图形语法理论（Data→Scale→Aesthetics→Geom→Coord）+ AI 出图选型（Vega-Lite vs AntV F2 vs ApexCharts）|
| [Qwen 到 MiMo 迁移](qwen-to-mimo-migration.md) | | 模型迁移记录 |
| [Neovim AI 规则](neovim-0.12-ai-rules.md) | | Neovim 0.12 AI 辅助规则 |

## 工具文档

| 文档 | English | 内容 |
|:--|:--|:--|
| [Nushell](tools/nushell.md) | | Nushell 使用与配置 |
| [Bx Vessel](tools/bx-vessel.md) | | Bx 容器构建框架 |
| [Ferron](tools/ferron.md) | | Ferron Web 服务器 |

## 项目文档

| 文档 | English | 内容 |
|:--|:--|:--|
| [项目索引](projects/) | | 项目特定文档目录 |
| [Fluxora 架构](projects/fluxora-architecture.md) | | AI-native UI 框架完整架构 |
| [Fluxora 决策](projects/fluxora-decisions.md) | | Fluxora ADR |
| [NixOS 配置](projects/nixos-config.md) | | NixOS 主机配置与模块结构 |
| [SkillForge](projects/skillforge.md) | | 多引擎 Agent 框架 |
| [知识库](projects/knowledge-base.md) | | SurrealDB RAG 架构 |
| [部署 CI](projects/deployment-ci.md) | | Helm 部署与 CI 数据配置 |
| [RisingWave CDC](projects/risingwave-cdc.md) | | RisingWave CDC 方案 |
| [KV 存储引擎架构](kv-storage-engine.md) | [KV Storage Engine](kv-storage-engine-en.md) | 嵌入式 KV 架构设计、复合键编码与用例 |
| [共识协议](consensus-protocol.md) | | Raft 共识协议的本质、边界与正确用途：元数据共识 vs 数据存储 |
| [分布式协作拓扑](distributed-collaboration-topology.md) | | 拓扑/信任边界与一致性算法的正交选择：单体红利、大节点联邦、TCC 柔性事务、CRDT 主权分界 |
| [OKM：Object-Keyspace Mapping](https://github.com/orbsh/okm) | | 对标 ORM 的 KV 键空间映射过程宏（文档已并入项目仓库） |
