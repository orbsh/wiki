# Wiki 索引

按主题分类的文档索引。项目特定文档见 [projects/](projects/index.md)。

## 记忆系统

| 文档 | 内容 |
|:--|:--|
| [图谱化记忆](graph-memory.md) | **设计文档**：计算时机光谱、属性图定位、原子三元组、双层模型、聚簇策略、权重系统、checkpoint 压缩、编码场景 |
| [Agent 记忆选型](agent-memory.md) | **现有方案分析**：agentmemory vs cognee 深度对比、四象限技术全景、CodeGraph 代码结构层、与 graph-memory 方案的逐项对照 |
| [Agent 复利](agent-compound-interest.md) | 持久化如何改变 AI 工具本质：memory/skill/cron 的累积效应、跨项目联动 |

## Agent 架构与使用

| 文档 | 内容 |
|:--|:--|
| [Agent 原则](agent-principles.md) | AI Agent 通用设计原则：确定性/非确定性边界、反主流护城河、概率系统思维 |
| [Agent 使用模式](agent-usage-patterns.md) | 人类侧交互技巧和审查习惯——决定 Agent 质量上限的不是模型，是使用方式 |
| [Hermes Agent 设计](hermes-agent-design.md) | Hermes 特定的使用约定、配置细节和运维协议 |
| [AI Agent 选型](ai-agent-selection.md) | 轻量 CLI Agent 选型调研（jcode → Hermes 之间） |
| [多智能体批判](multi-agent-critique.md) | 多 Agent 模式的技术批判 |
| [用户画像：Orbit](orbit-profile.md) | Orbit（O）的偏好、习惯、技术立场 |

## LLM 与认知

| 文档 | 内容 |
|:--|:--|
| [LLM 基础](llm-fundamentals.md) | 训练涌现智能、三层局限、认知框架 |
| [LLM 过度思考批判](llm-overthinking-critique.md) | 表演拟人 vs 真正推理、草稿本纠偏 |
| [LLM 缓存破坏模式](llm-caching-destruction-patterns.md) | Prompt Caching 生效条件、前缀稳定性、IDE/Agent 场景的缓存杀手 |
| [LLM 隐藏行为模式](llm-hidden-behavior-patterns.md) | 模型未显式表达但影响输出的行为模式 |
| [认知心理学](cognitive-psychology.md) | 认知科学框架在 AI Agent 设计中的应用 |

## 架构与基础设施

| 文档 | 内容 |
|:--|:--|
| [AI 友好基础设施](ai-friendly-infrastructure.md) | 声明式 OS、全链路排查、NixOS 为何最适合 AI |
| [架构选择](architecture-choices.md) | 网关选择、仓库策略（Zot+WG）、Redis/数据库批判 |
| [统一数据层](unified-data-layer.md) | SurrealDB 多模型引擎、SurQL 人体工程学、Redis/PG 统一替代 |
| [湖仓研究](lakehouse-research.md) | Iceberg/LanceDB/Delta Lake 选型 |
| [LanceDB vs Fjall](lancedb-vs-fjall.md) | 列式向量存储 vs 嵌入式 KV 的定位对比 |
| [Fjall Openraft ADR](fjall-openraft-adr.md) | 嵌入式 KV + Raft 共识替代 Redis |
| [Fjall 多模态索引](fjall-multimodal-indexing.md) | Fjall 的多模态索引设计 |
| [Arrow HTAP 引擎](arrow-unified-htap-engine.md) | Arrow 统一 HTAP 引擎设计 |
| [存储方案](storage-solution.md) | 存储架构选型 |
| [拥塞控制设计](congestion-control-design.md) | 网络拥塞控制 |
| [轻量 FaaS 架构](lightweight-faas-architecture.md) | 轻量级函数即服务架构 |

## 批判文档

| 文档 | 内容 |
|:--|:--|
| [Redis 批判](redis-critique.md) | 网络延迟陷阱、单线程瓶颈、内存浪费 |
| [Nginx 批判](nginx-critique.md) | 为何 Nginx 过时：多进程低效、静态配置、Lua 复杂性 |
| [MySQL 批判](mysql-critique.md) | MySQL 的架构缺陷 |
| [等保标准批判](mlps-critique.md) | 等保标准与现代安全实践的差异：云环境适配、密码轮换、防火墙、审计 |
| [MCP 批判](mcp-critique.md) | MCP 协议的政治性妥协与工程代价 |
| [Dify 批判](dify-critique.md) | AI 应用领域的 Harbor 复制品：低代码 vs AI 范式矛盾 |
| [Harbor 批判](harbor-critique.md) | 企业膨胀、CNCF 政治 vs 工程现实 |
| [阿里云批判](aliyun-critique.md) | 阿里云的架构与商业问题 |
| [AI 编程乐观批判](ai-programming-optimism-critique.md) | AI 编程的能力边界 |
| [Vim 运动批判](vim-movement-critique.md) | Vim 运动模型的局限 |
| [为什么不写注释](why-not-write-comments.md) | 代码注释的反主流观点 |

## 设计与语言

| 文档 | 内容 |
|:--|:--|
| [Aura 架构](aura-architecture.md) | Event Realm、Actor、事件组合原语、interface_schema |
| [Aura Fluxora DevOps](aura-fluxora-devops.md) | Aura 与 Fluxora 的 DevOps 集成 |
| [响应式架构](reactive-architecture.md) | 响应式系统设计 |
| [WASM 统一运行时](wasm-unified-runtime.md) | WebAssembly 统一运行时架构 |
| [现代语言设计](modern-language-design.md) | 语言设计趋势与分析 |
| [嵌入式脚本语言](embedded-script-languages.md) | 嵌入式脚本语言选型 |
| [Lambda 到硅](lambda-to-silicon.md) | 从 Lambda 演算到硅片的计算模型演进 |
| [序列化协议决策](serialization-protocol-decision.md) | 序列化格式选型 |

## 工作流与技能

| 文档 | 内容 |
|:--|:--|
| [自动化 Skill 工作流](automation-skill-workflow.md) | 浏览器操作→可复用 Hermes Skill 的逆向工程 |
| [NL-to-SQL 技能架构](nl-to-sql-skill-architecture.md) | 自然语言到 SQL 的技能设计 |
| [Nushell 介绍](nushell-introduction.md) | Nushell 入门 |
| [Loop 工程分析](loop-engineering-analysis.md) | Loop 工程方法论框架 |

## AI 时代思考

| 文档 | 内容 |
|:--|:--|
| [AI 时代组织](ai-era-organization-individual.md) | 约束-解分析方法论、联邦化、个体崛起 |
| [AI 时代商业模式](ai-era-business-models.md) | AI 对商业模式的重塑 |
| [AI 流动性](ai-liquidity-camp.md) | AI 时代的流动性分析 |
| [Karpathy AI 编码方法论](karpathy-ai-coding-methodology.md) | Karpathy 的 AI 编码实践 |

## 工具与环境

| 文档 | 内容 |
|:--|:--|
| [Emacs 研究](emacs-research.md) | Emacs 配置与使用 |
| [编辑器选型 2026](editor-selection-2026.md) | 编辑器选型对比 |
| [区块编辑器架构](block-editor-architecture.md) | 区块编辑器设计 |
| [Linux SSD 优化](linux-ssd-optimization.md) | SSD 性能优化 |
| [Debian 容器沙箱](debian-container-sandbox.md) | 容器化沙箱方案 |
| [P2P 网格对比](p2p-mesh-networking-comparison.md) | P2P 网状网络方案对比 |
| [无状态认证选型](stateless-auth-selection.md) | 无状态认证方案 |
| [Gateway API 迁移](gateway-api-migration.md) | Istio 到 Envoy HTTPRoute 迁移 |

## 其他

| 文档 | 内容 |
|:--|:--|
| [Fractal](Fractal.md) | 核心自包含文档 |
| [技术哲学](tech-philosophy.md) | 应试工程 vs 范式转移 |
| [Zig 反智主义](go-zig-anti-intellectualism.md) | Zig 语言与反智主义 |
| [职场沟通](workplace-communication.md) | 职场沟通方法 |
| [坐姿工作范式](seated-work-paradigm.md) | 坐姿与工作方式 |
| [视觉神话](visual-myth.md) | 视觉相关的认知偏差 |
| [Qwen 到 MiMo 迁移](qwen-to-mimo-migration.md) | 模型迁移记录 |
| [Neovim AI 规则](neovim-0.12-ai-rules.md) | Neovim 0.12 AI 辅助规则 |

## 工具文档

| 文档 | 内容 |
|:--|:--|
| [Nushell](tools/nushell.md) | Nushell 使用与配置 |
| [Bx Vessel](tools/bx-vessel.md) | Bx 容器构建框架 |
| [Ferron](tools/ferron.md) | Ferron Web 服务器 |

## 项目文档

| 文档 | 内容 |
|:--|:--|
| [项目索引](projects/) | 项目特定文档目录 |
| [Fluxora 架构](projects/fluxora-architecture.md) | AI-native UI 框架完整架构 |
| [Fluxora 决策](projects/fluxora-decisions.md) | Fluxora ADR |
| [NixOS 配置](projects/nixos-config.md) | NixOS 主机配置与模块结构 |
| [SkillForge](projects/skillforge.md) | 多引擎 Agent 框架 |
| [知识库](projects/knowledge-base.md) | SurrealDB RAG 架构 |
| [部署 CI](projects/deployment-ci.md) | Helm 部署与 CI 数据配置 |
| [RisingWave CDC](projects/risingwave-cdc.md) | RisingWave CDC 方案 |
| [Fjall Openraft 设计](projects/fjall-openraft-design.md) | 原生分布式系统设计 |
