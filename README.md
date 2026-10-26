# Wiki 索引

按主题分类的文档索引。项目特定文档见 [projects/](projects/index.md)。

## 记忆系统

| 文档 | English | 内容 |
|:--|:--|:--|
| [图谱化记忆](graph-memory.md) | [Graph Memory](graph-memory-en.md) | **设计文档**：计算时机光谱、属性图定位、原子三元组、双层模型、聚簇策略、权重系统、编码场景 |
| [无状态 Agent 架构](stateless-agent-architecture.md) | [Stateless Agent Architecture](stateless-agent-architecture-en.md) | **架构总纲**：turn 即执行单位、组件四分（Prism/Gravity/Krystallizer/Effector）、Surface 设计（注入与触发）、压缩并行旁路、skill 涌现闭环、传输裁决 |
| [Krystallizer](krystallizer.md) | [Krystallizer](krystallizer-en.md) | **实现设计**：会话控制原语（branch/tail/summarize）、full/assist 双模式、提取机制三方案、事实序列化、存储实现、skillforge 实现现状 |
| [无状态 Agent 架构](stateless-agent-architecture.md) | [Stateless Agent Architecture](stateless-agent-architecture-en.md) | **架构设计**：turn 即执行单位、组件四分（Prism/Gravity/Krystallizer/Effector）、skill 涌现闭环、HTTP+SSE 传输裁决、与传统架构对照 |
| [记忆选型](agent-memory.md) | [Memory Architecture](agent-memory-en.md) | **选型综述**：Surface/Engine 两层、计算时机光谱、注入方式、外部开源方案对比、合成闭环缺口 |
| [Agent 复利](agent-compound-interest.md) | [Agent Compound Interest](agent-compound-interest-en.md) | 持久化如何改变 AI 工具本质：memory/skill/cron 的累积效应、跨项目联动 |

## Agent 架构与使用

| 文档 | English | 内容 |
|:--|:--|:--|
| [Agent 原则](agent-principles.md) | [Agent Principles](agent-principles-en.md) | AI Agent 通用设计原则：确定性/非确定性边界、反主流护城河、概率系统思维 |
| [Agent 使用模式](agent-usage-patterns.md) | [AI Agent Usage Patterns](agent-usage-patterns-en.md) | 人类侧交互技巧和审查习惯——决定 Agent 质量上限的不是模型，是使用方式 |
| [Hermes Agent 设计](hermes-agent-design.md) | [Hermes Agent Design](hermes-agent-design-en.md) | Hermes 特定的使用约定、配置细节和运维协议 |
| [AI Agent 选型](ai-agent-selection.md) | | 轻量 CLI Agent 选型调研——已定案：Hermes 主力 + jcode 救援，Open Interpreter 移除 |
| [多智能体批判](multi-agent-critique.md) | [Multi-Agent Critique](multi-agent-critique-en.md) | 多 Agent 模式的技术批判 |
| [辩论验证](debate-validation.md) | | Prefix Checkpoint 方案的辩论验证过程：帕累托最优论证、层次隔离、揭穿 AI 幻觉 |
| [用户画像：Orbit](orbit-profile.md) | | Orbit（O）的偏好、习惯、技术立场 |
| [智力金字塔模型](intelligence-pyramid.md) | [Intelligence Pyramid](intelligence-pyramid-en.md) | 多级智力架构与卸载策略：LLM→ML→算法→代码的梯度平衡 |

## LLM 与认知

| 文档 | English | 内容 |
|:--|:--|:--|
| [LLM 基础](llm-fundamentals.md) | [LLM Fundamentals](llm-fundamentals-en.md) | 训练涌现智能、三层局限、认知框架 |
| [LLM 过度思考批判](llm-overthinking-critique.md) | [LLM Overthinking Critique](llm-overthinking-critique-en.md) | 表演拟人 vs 真正推理、草稿本纠偏 |
| [LLM 缓存破坏模式](llm-caching-destruction-patterns.md) | [LLM Caching Destruction Patterns](llm-caching-destruction-patterns-en.md) | Prompt Caching 生效条件、前缀稳定性、IDE/Agent 场景的缓存杀手 |
| [LLM 隐藏行为模式](llm-hidden-behavior-patterns.md) | [LLM Hidden Behavior Patterns](llm-hidden-behavior-patterns-en.md) | 模型未显式表达但影响输出的行为模式 |
| [缓存树和尾提示词优化](tail-prompt-optimization.md) | [Tail Prompt Optimization](tail-prompt-optimization-en.md) | 上下文缓存旁路分支机制，用于压缩/汇总/提取 |
| [认知心理学](cognitive-psychology.md) | [Cognitive Psychology](cognitive-psychology-en.md) | 个人认知与成长原则：自驱力、无痛重复、认知陷阱、副产品思维；专注的连续性与资源约束下的取舍 |
| [紧急陷阱](urgency-vs-importance.md) | | 紧急为何不自动累积成重要：四种失效、Goodhart 定律与伪造成本判据、表演棘轮（代价作为最后一种可信信号）、反馈回路断裂、三档杠杆 |
| [意义来源于连接](meaning-as-connection.md) | | 知识论：意义是连接的涌现、难度即概念到既有网络的距离、小说与数学的不对称、情绪自锁循环、枢纽拓扑与神童寓言、教育推论、成年人靠库存 |
| [终生学习与环境工程](lifelong-learning.md) | | 成年人的学习：点火而非坚持、代理指标必然败坏、散兵线、沉浸环境与接触流、信念的现货与期货、奶头乐分叉 |
| [自上而下](top-down-learning.md) | | 学习顺序：自下而上的两重代价、真实问题自带次序、两个西格玛、递归补洞、AI 私教、反馈密度、作品与代理信号 |

## 架构与基础设施

| 文档 | English | 内容 |
|:--|:--|:--|
| [AI 友好基础设施](ai-friendly-infrastructure.md) | [AI-Friendly Infrastructure](ai-friendly-infrastructure-en.md) | 声明式 OS、全链路排查、NixOS 为何最适合 AI |
| [架构选择](architecture-choices.md) | [Architecture Choices](architecture-choices-en.md) | 网关选择、仓库策略（Zot+WG）、Redis/数据库批判 |
| [统一数据层](unified-data-layer.md) | [Unified Data Layer](unified-data-layer-en.md) | SurrealDB 多模型引擎、SurrealQL 人体工程学、与 PG 的分层选型 |
| [SQL/PGQ 属性图查询](property-graph-queries.md) | [Property Graph Queries](property-graph-queries-en.md) | 在表上声明图、一图多关系、跨图查询、权限可达性、与图数据库/递归 CTE 的关系 |
| [SurrealDB 评估档案](query-language-design.md) | [Query Language Design](query-language-design-en.md) | SurrealDB 评估档案：SQL 的根本性缺陷、SurrealQL 的架构级解法、挑战者逻辑 |
| [湖仓研究](lakehouse-research.md) | [Lakehouse Research](lakehouse-research-en.md) | Iceberg/LanceDB/Delta Lake 选型 |
| [HelixDB vs LanceDB](helixdb-vs-lancedb.md) | [HelixDB vs LanceDB](helixdb-vs-lancedb-en.md) | 对象存储上 AI 数据栈的两种路线对比 |
| [Aura 架构 §5](aura-architecture.md) | [Aura Architecture](aura-architecture-en.md) | Fjall + 独立元数据（单写无共识）分布式存算一体 |
| [Arrow HTAP 引擎](arrow-unified-htap-engine.md) | [Arrow Unified HTAP Engine](arrow-unified-htap-engine-en.md) | Arrow 统一 HTAP 引擎设计 |
| [文件系统方案](filesystem-solution.md) | | 文件系统选型 |
| [拥塞控制设计](congestion-control-design.md) | [Congestion Control Design](congestion-control-design-en.md) | 网络拥塞控制 |
| [轻量 FaaS 架构](lightweight-faas-architecture.md) | | 轻量级函数即服务架构 |

## 批判文档

| 文档 | English | 内容 |
|:--|:--|:--|
| [Redis 批判](redis-critique.md) | [Redis: the "Network RAM" Trap](redis-critique-en.md) | 网络延迟陷阱、单线程瓶颈、内存浪费 |
| [高并发批判](high-concurrency-critique.md) | | 「高并发」作为简中特有话语的解剖：词汇量纲缺失、面试循环繁殖、fpm 防御性焦虑 |
| [0→1 话语批判](innovation-discourse-critique.md) | | 「0→1 / 1→100」话术的解剖：用跨度冒充高度、应用遥遥领先的证伪、市场对创新课税、体量≠能力、应用工程≠计算机科学、可替代性判据 |
| [Nginx 批判](nginx-critique.md) | [Nginx Critique](nginx-critique-en.md) | 为何 Nginx 过时：多进程低效、静态配置、Lua 复杂性 |
| [MySQL 批判](mysql-critique.md) | [MySQL Critique](mysql-critique-en.md) | MySQL 的架构缺陷 |
| [等保标准批判](mlps-critique.md) | | 等保标准与现代安全实践的差异：云环境适配、密码轮换、防火墙、审计 |
| [MCP 批判](mcp-critique.md) | [MCP Over-Engineering](mcp-critique-en.md) | MCP 协议的政治性妥协与工程代价 |
| [Dify 批判](dify-critique.md) | [Dify Critique](dify-critique-en.md) | AI 应用领域的 Harbor 复制品：低代码 vs AI 范式矛盾 |
| [Harbor 批判](harbor-critique.md) | [Harbor Critique](harbor-critique-en.md) | 企业膨胀、CNCF 政治 vs 工程现实 |
| [阿里云批判](aliyun-critique.md) | [Aliyun Critique](aliyun-critique-en.md) | 阿里云的架构与商业问题 |
| [AI 编程乐观批判](ai-programming-optimism-critique.md) | [AI Programming Optimism Critique](ai-programming-optimism-critique-en.md) | AI 编程的能力边界 |
| [Vim 运动批判](vim-movement-critique.md) | [Vim Movement Logic Critique](vim-movement-critique-en.md) | Vim 运动模型的局限 |
| [为什么不写注释](why-not-write-comments.md) | [Why You Shouldn't Write Comments](why-not-write-comments-en.md) | 代码注释的反主流观点 |

## 设计与语言

| 文档 | English | 内容 |
|:--|:--|:--|
| [Aura 架构](aura-architecture.md) | [Aura Architecture](aura-architecture-en.md) | Event Realm、摊位、事件组合原语、interface_schema |
| [长连接工程挑战](long-lived-connection-engineering.md) | [Long-Lived Connection Engineering](long-lived-connection-engineering-en.md) | WS/WT 长连接的物理限制（背压、碎片化、连接倾斜）与解法 |
| [Aura Fluxora DevOps](aura-fluxora-devops.md) | [Aura + Fluxora DevOps](aura-fluxora-devops-en.md) | Aura 与 Fluxora 的 DevOps 集成 |
| [Flux 架构](flux-architecture.md) | [Flux Architecture](flux-architecture-en.md) | 响应式通信模式：从被动拉取到主动推送，Fluxora = Flux + Aura |
| [WASM 统一运行时](wasm-unified-runtime.md) | [WASM Unified Runtime](wasm-unified-runtime-en.md) | WebAssembly 统一运行时架构 |
| [现代语言设计](modern-language-design.md) | [Modern Language Design](modern-language-design-en.md) | 语言设计趋势与分析 |
| [适合 LLM 编写的编程语言](llm-language-selection.md) | [Language Selection for LLM-Generated Code](llm-language-selection-en.md) | 生成者侧判据：验证成本、逃生通道与治理、按语言的评级 |
| [编程语言选型](language-selection.md) | [Programming Language Selection](language-selection-en.md) | 人类侧：认知负荷、四象限领地、生态与工具链、被淘汰语言 |
| [原型/快速开发迷思](prototype-myth.md) | [The Prototype / Rapid-Development Myth](prototype-myth-en.md) | 「先用 Python 验证再重写」的前提为何不成立：类型是契约，AI 改变类型成本 |
| [ECS 实体组件系统](entity-component-system.md) | [Entity-Component-System (ECS)](entity-component-system-en.md) | ECS 三层模型、数据驱动收益、网/树拓扑定位、Bevy 0.19 实践 |
| [Go, Zig 与反智主义](go-zig-anti-intellectualism.md) | [Go, Zig and Anti-Intellectualism](go-zig-anti-intellectualism-en.md) | Go, Zig 与反智主义 |
| [嵌入式脚本语言](embedded-script-languages.md) | [Embedded Script Language Selection](embedded-script-languages-en.md) | 嵌入式脚本语言选型 |
| [Lambda 到硅](lambda-to-silicon.md) | [Lambda to Silicon](lambda-to-silicon-en.md) | 从 Lambda 演算到硅片的计算模型演进 |
| [自动机、状态机与 reducer](automaton-vs-state-machine.md) | [Automaton vs State Machine](automaton-vs-state-machine-en.md) | 术语的两种语域：形式语域（自动机 = 状态机）与工程语域（单实体受限操作）、RSM 定位 |
| [序列化协议分析对比](serialization-protocol-comparison.md) | [Serialization Protocol Comparison](serialization-protocol-comparison-en.md) | 序列化格式选型 |
| [文档格式对比](document-format-comparison.md) | | Markdown 与下一代纯文本标记语言：按笔记/排版/出版用途分轴对比 Neorg、Org-mode、Typst、AsciiDoc |
| [KDL vs 配置格式](kdl-vs-config-formats.md) | | 语义树 vs 数据网络：KDL 节点模式与 TOML/YAML/JSON/HTML 全景对比、工程坑点；KDL 作为配置生成中间层的复用价值 |
| [标签森林](tag-forest.md) | [Tag Forest](tag-forest-en.md) | 多棵树互斥维度组成的内容组织：situation/importance·urgency 评分树、隔离实例、邻接表+递归CTE查询 |

## 工作流与技能

| 文档 | English | 内容 |
|:--|:--|:--|
| [自动化 Skill 工作流](automation-skill-workflow.md) | [Automation Skill Workflow](automation-skill-workflow-en.md) | 浏览器操作→可复用 Hermes Skill 的逆向工程 |
| [NL2SQL 架构](nl2sql-architecture.md) | | OLTP 上做分析是架构设计错误：批判性分析 + 技能实现 |
| [Nushell 介绍](nushell-introduction.md) | | Nushell 入门 |
| [Loop 工程分析](loop-engineering-analysis.md) | [Loop Engineering Analysis](loop-engineering-analysis-en.md) | Loop 工程方法论框架 |
| [迭代闭环](iteration-closed-loop.md) | [Iteration Closed Loop](iteration-closed-loop-en.md) | 行动、复盘、转向：迭代闭环的结构骨架、心理阻塞与元能力 |
| [信息流精炼](infoflow-refinement.md) | | 信息流管道捕获→属性化→精炼→消费：NB 评分·标题党一致性检测、html→md·链接挖掘、RSS、聚合摘要/态势感知 |
| [任务管理架构](task-management.md) | [Task Management](task-management-en.md) | scratch 任务模型（结构化 = 树形 = 脑图）、与 neorg 对比、改进计划与 agent 集成模式 |

## AI 时代思考

| 文档 | English | 内容 |
|:--|:--|:--|
| [AI 时代组织](ai-era-organization-individual.md) | [AI Era Organization and Individual](ai-era-organization-individual-en.md) | 约束-解分析方法论、联邦化、个体崛起 |
| [液态化](liquefaction.md) | | **科普向**：故事开篇。文件/频道/职级/页面四层边界同源；一个窗口、拖钉成流、夜间扫描投递、可导航三维图；知识、组织、软件是同一张图的四种读法 |
| [AI 时代商业模式](ai-era-business-models.md) | [AI Era Business Models](ai-era-business-models-en.md) | AI 对商业模式的重塑 |
| [风口的三种姿态](windfall-arbitrage.md) | [Windfall Arbitrage](windfall-arbitrage-en.md) | 渠道控制者 / 有准备的人 / 投机者三种姿态与对比：复利资产的差异 |
| [市场形态全谱系](market-form-spectrum.md) | | 服务/产品/创作者/资产/投机/中介六形态与个人适配、认知操作系统轴、天赋勘探提示词 |
| [AI 流动性](ai-liquidity-camp.md) | [AI Liquidity Camp](ai-liquidity-camp-en.md) | AI 时代的流动性分析 |
| [Karpathy AI 编码方法论](karpathy-ai-coding-methodology.md) | [Karpathy AI Coding Methodology](karpathy-ai-coding-methodology-en.md) | Karpathy 的 AI 编码实践 |
| [国内 AI 产业制度性缺陷](ai-industry-institutional-critique.md) | | 蒸馏/刷分现象、评审体制逆向选择、自由市场 vs 国家主导 |
| [AI 辅助视频制作与游戏引擎](ai-creative-production-workflow.md) | [AI Creative Production Workflow](ai-creative-production-workflow-en.md) | AI 视频 10 秒魔咒、UE6/Bevy 路线、AI+Blender 工作流 |
| [3D 工具能否替代 2D 工具](blender-replace-2d-tools.md) | [Can 3D Tools Replace 2D Tools?](blender-replace-2d-tools-en.md) | 历史成因、Blender 的 2D 能力现状、收敛与分叉的力量、判断框架 |
| [3D 数据可视化](3d-data-visualization-aesthetics.md) | | 3D 作为叙事语言：物理流动/空间体积/纵深层级/材质语义 + 通用可视化问题的 3D 解法 |
| [3D 渲染库选型](3d-rendering-library-selection.md) | | 自定义空间展示的渲染库之争：three.js/three-d/Bevy/deck.gl API 风格 + wasm 体积实测 |

## 工具与环境

| 文档 | English | 内容 |
|:--|:--|:--|
| [Emacs 研究](emacs-research.md) | | Emacs 配置与使用 |
| [编辑器选型 2026](editor-selection-2026.md) | [Editor Selection Analysis](editor-selection-2026-en.md) | 编辑器选型对比 |
| [区块编辑器架构](block-editor-architecture.md) | [Block Editor Architecture](block-editor-architecture-en.md) | 区块编辑器设计 |
| [Linux 桌面工作流](linux-desktop-workflow.md) | | 平铺 WM 到 COSMIC DE 的架构选择：外层堆叠+内层平铺、Nushell+Pop-Launcher 数据流 |
| [Linux SSD 优化](linux-ssd-optimization.md) | | SSD 性能优化 |
| [Debian 容器沙箱](debian-container-sandbox.md) | | 容器化沙箱方案 |
| [P2P 网格对比](p2p-mesh-networking-comparison.md) | | P2P 网状网络方案对比 |
| [无状态认证选型](stateless-auth-selection.md) | | 无状态认证方案 |
| [Gateway API 迁移](gateway-api-migration.md) | | Istio 到 Envoy HTTPRoute 迁移 |
| [浏览器键盘模式](keyboard-wm-browser-mode.md) | | Niri + chromium --app + CDP 的全键盘浏览器模式：信息流精炼定位、tag 森林组织、mudra 底座 |
| [键盘键位重映射设计](keyboard-keymap-design.md) | [Keyboard Keymap Design](keyboard-keymap-design-en.md) | 修饰键重映射：evdev vs XKB、kanata 的取舍与配置 |

## 其他

| 文档 | English | 内容 |
|:--|:--|:--|
| [Fractal](Fractal.md) | | 核心自包含文档 |
| [技术哲学](tech-philosophy.md) | | 应试工程 vs 范式转移 |
| [职场沟通](workplace-communication.md) | [Workplace Communication](workplace-communication-en.md) | 职场沟通方法 |
| [坐姿工作范式](seated-work-paradigm.md) | | 坐姿与工作方式 |
| [工程思维与工程素养](engineering-mindset-and-competence.md) | | 工程师双核心能力：认知决策模型（思维）与职业行为规范（素养） |
| [可视化迷思](visual-myth.md) | [The Myth of Visualization](visual-myth-en.md) | 可视化形式的结构性批判 |
| [图形语法与 AI 可视化](grammar-of-graphics.md) | | 图形语法理论（Data→Scale→Aesthetics→Geom→Coord）+ AI 出图选型（Vega-Lite vs AntV F2 vs ApexCharts）|
| [Qwen 到 MiMo 迁移](qwen-to-mimo-migration.md) | [Qwen to MiMo Migration](qwen-to-mimo-migration-en.md) | 模型迁移记录 |
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
| [Fluxora 架构](projects/fluxora-architecture.md) | [Fluxora Architecture](projects/fluxora-architecture-en.md) | AI-native UI 框架完整架构 |
| [Flex UI](flex-ui.md) | [Flex UI](flex-ui-en.md) | 传输 UI 的事件驱动声明式框架：UI 作为数据结构、Accrete DSL、Bind 系统、封闭组件集的取舍 |
| [Flex UI 的 CSS 方案](flex-ui-css.md) | [Flex UI CSS](flex-ui-css-en.md) | 变量即协议 + Trait 组合：从 fluxora 分离的设计史、现代 CSS 的三个能力、与 Flex UI 的关系 |
| [Fluxora 决策](projects/fluxora-decisions.md) | | Fluxora ADR |
| [NixOS 配置](projects/nixos-config.md) | [NixOS Configuration](projects/nixos-config-en.md) | NixOS 主机配置与模块结构 |
| [SkillForge](projects/skillforge.md) | | 多引擎 Agent 框架 |
| ψ 语言设计（`~/world/psi`） | | wasm 原生的方法链语言（假想设计）：调用单原语、统一访问、标签模型、Json 与 Path、泛型与种类。文档已迁入项目仓库 |
| [知识库](projects/knowledge-base.md) | [Knowledge Base](projects/knowledge-base-en.md) | SurrealDB RAG 架构 |
| [部署 CI](projects/deployment-ci.md) | | Helm 部署与 CI 数据配置 |
| [RisingWave CDC](projects/risingwave-cdc.md) | | RisingWave CDC 方案 |
| [KV 存储引擎架构](kv-storage-engine.md) | [KV Storage Engine](kv-storage-engine-en.md) | 嵌入式 KV 架构设计、复合键编码与用例 |
| [共识协议](consensus-protocol.md) | [Consensus Protocol](consensus-protocol-en.md) | Raft 共识协议的本质、边界与正确用途：元数据共识 vs 数据存储 |
| [分布式协作拓扑](distributed-collaboration-topology.md) | [Distributed Collaboration Topology](distributed-collaboration-topology-en.md) | 拓扑/信任边界与一致性算法的正交选择：单体红利、大节点联邦、TCC 柔性事务、CRDT 主权分界 |
| [分布式锁反模式](distributed-lock-anti-pattern.md) | [Distributed Lock Anti-Pattern](distributed-lock-anti-pattern-en.md) | 术语澄清（哪些不是锁）、文件锁的系统层/平台层分界、无锁分布式事务、共识替代锁 |
| [OKM：Object-Keyspace Mapping](https://github.com/orbsh/okm) | | 对标 ORM 的 KV 键空间映射过程宏（文档已并入项目仓库） |
