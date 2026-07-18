# Aura：存算一体的现代分布式 Actor 引擎

**状态**：架构设计完成
**日期**：2026-06-20
**核心哲学**：框架融合、计算下沉、零拷贝、确定性边界

> Aura 是一个存算一体的现代分布式 Actor 引擎。在 Aura 里，没有 SQL，没有外部缓存锁，没有多库同步。数据库完全隐形，状态由 Actor 自动管理。你只需在内存里修改上下文，剩下的事情，Aura 的 Rust 运行时自动处理高并发与多机备份。

## 0. 架构纲领：底层框架融合

当前工程实践的一条清晰主线：**网关、服务编排、Web 框架、存储、消息队列等底层框架正在走向融合**。Rivet 将 Actor 运行时与存储同机共生，Windmill 将任务编排与 FaaS 执行层合为一体，Dapr 试图用 Sidecar 统一服务间通信与状态管理——尽管其过于散碎，诉求是一致的。Aura 是这一趋势的工程实践：将上述五种能力全部编译进同一个 Rust 进程。

### 去微服务化的动因

微服务架构解决了单体的扩展性和团队协作问题，但引入了一套新的系统性代价：

| 问题 | 根因 |
|------|------|
| **网络 RTT 累积** | 服务间每次调用 1–3ms 延迟，复杂业务链路（用户 → 订单 → 商品 → 库存）的 N+1 跨服务查询将延迟线性放大 |
| **分布式事务** | 跨服务的数据一致性需要 Saga / TCC 等补偿机制，实现复杂且难以调试 |
| **运维膨胀** | 服务发现、负载均衡、熔断降级、链路追踪——每一项都需要独立基础设施（K8s + Istio + Jaeger + ...） |
| **数据搬运** | 服务间通过 RPC 传输序列化数据（JSON/Protobuf），应用端反复反序列化、拼装，CPU 和内存开销远超业务逻辑本身 |
| **Sidecar 开销** | 服务网格（Dapr、Istio）以 Sidecar 形式为每个服务附加代理层，诉求合理但过于散碎，运维成本与延迟收益不成比例 |

这些问题不是微服务的实现缺陷，而是分布式架构的结构性代价——只要服务间通信经过网络，上述代价不可避免。去微服务化的本质是：**用进程内通信替代网络通信，同时保留微服务的解耦能力**。

### 单体原有问题的当代解法

单体被淘汰不是因为"单体"本身有错，而是因为当时的工程手段无法在单体内部解决三个问题。这三个问题在今天已有成熟解法：

| 单体原有问题 | 当代解法 | Aura 中的对应 |
|-------------|---------|--------------|
| **语言锁定**：全系统被迫使用同一种语言，无法为不同场景选最优工具 | 嵌入式多语言运行时（进程内 VM，零 IPC） | Steel / Python / Wasm 三种语言通过 PyO3 和 Wasmtime 内嵌，Actor 按需选择 |
| **团队耦合**：修改一处可能影响全局，多团队并行开发冲突频繁 | 事件契约解耦（声明式接口，编译期/加载期校验） | `interface_schema()` 声明 receives/emits，Realm 强制白名单校验 |
| **无法独立伸缩**：整体部署，热点模块无法单独扩容 | Virtual Actor 按需激活，空闲自动 Scale-to-Zero | partition key 路由 + Fjall 磁盘休眠 + Tokio 事件循环 |

这三个解法恰好对应微服务当初试图解决的三个痛点。区别在于：微服务用"拆进程"来解决，Aura 用"进程内隔离"来解决——后者没有网络开销。

### Aura：标准化的融合方案

去微服务化不是回到传统单体，而是走向一种新的标准化形态：**一个进程内包含完整的分布式能力**。当前的去微服务化实践大多是 ad-hoc 的——每个团队根据自己的场景选择不同的组件拼凑（嵌入式 KV + 自建事件总线 + 自己写状态管理），缺乏统一的架构模式。

Aura 将这种融合标准化为一个可复用的引擎：

| 能力 | 传统拼凑方案 | Aura 标准化方案 |
|------|------------|----------------|
| 服务编排 | 自建 Actor 系统 + 手写状态机 | 场域模型（emit/on/on_join/on_batch）+ `interface_schema()` 自动路由 |
| 状态管理 | Redis / Memcached / 自建 HashMap | Fjall LSM-Tree（WAL 断电保护 + Scale-to-Zero） |
| 跨节点一致性 | 各服务自行实现 or 不实现 | Openraft 强一致共识，Actor 状态自动多机复制 |
| 多语言支持 | 各服务独立运行时（微服务模式） | 进程内嵌入（Steel/Python/Wasm），零序列化 |
| 部署 | Docker + K8s + Helm + 服务网格 | 单二进制，`scp` 即部署，加 `--raft-nodes` 即分布式 |

标准化的意义在于：开发者不再需要为每个项目重新设计"怎么把多个框架拼在一起"，而是直接使用一个经过验证的融合引擎，将精力集中在业务逻辑上。

### 融合的边界：不是分布式单体

融合不等于把多个服务共享一个数据库（分布式单体）。分布式单体同时承受微服务的部署复杂度和单体的耦合强度——改一个服务的 schema 全线牵连，但你享受不到单体的简单性。Aura 的做法是**物理单体，逻辑分布式**：

- **物理层面**：单二进制、进程内通信、零网络 RTT——取单体的性能与简洁
- **逻辑层面**：每个 Actor 拥有独立的状态分区（Fjall partition），通过场域事件（emit/on）松耦合通信，运行时按 partition key 激活和驱逐——取微服务的解耦与弹性

| 维度 | 微服务 | 分布式单体 | **Aura** | 单体 |
|------|--------|-----------|---------|------|
| 通信 | 网络 RPC | 共享数据库 JOIN | **进程内 emit/on** | 进程内函数调用 |
| 耦合 | 松 | 紧（共享 schema） | **松（事件契约）** | 紧 |
| 部署 | 多二进制 | 多二进制 | **单二进制** | 单二进制 |
| 语言 | 每服务可选 | 每服务可选 | **每 Actor 可选** | 全局统一 |
| 状态隔离 | 天然隔离 | **无隔离（共享 DB）** | **独立 Fjall 分区** | 共享内存 |

Aura 融合了微服务的解耦能力和单体的性能优势，同时规避了两者的固有缺陷——以及分布式单体这一反模式的全部代价。

## 1. 软件工程第一性原理：计算下沉与零拷贝

在现代通用开发环境中，盲目引入外部易失性内存缓存已被证明会带来严重的工程债务与认知负担。本架构基于第一性原理，确立了"计算紧贴存储"与"单体事务原子化"的核心原则，彻底排除了 Application-Side Joins（应用端拼装）的低效模式。

### 1.1 传统"多轮网络往返（N+1 Queries）"的失效分析

在处理复杂的领域模型时，外部缓存迫使开发者在应用端用代码扮演低效的"伪查询引擎"。

- **数据空耗**：为了组装一个包含多层嵌套关系的业务对象（例如：用户 → 订单 → 商品 → 库存），应用服务器必须通过网线高频发起多次网络往返（RTT）。
- **算力开销**：应用服务器耗费了大量的 CPU 周期，用于对传输过来的海量 RESP、JSON 或 Protobuf 格式数据流进行反复的序列化与反序列化，在应用内存中合并碎片。

### 1.2 本架构的设计机制

- **核心公式**：真实总延迟 = 网络 RTT + 数据库内部处理时间

由于现代局域网/VPC 内部的物理 RTT 往往高达 1.0ms 至 3.0ms，外部缓存零点几毫秒的纯内存优势在网络延迟面前毫无意义。

- **计算下沉（Compute Near Data）**：通过直接压榨 PostgreSQL 的共享内存（Shared Buffers）与 JSONB 强悍的聚合能力（或利用 plpython3u、SurrealDB 的原生多模态图遍历），实现单次请求，单次返回，直达本质。

→ 详见 [Redis 批判](redis-critique.md)、[MySQL 批判](mysql-critique.md)、[SurrealDB 分析](unified-data-layer.md)

## 2. 核心架构拓扑：Rust 异步 Actor + Steel Lisp 嵌入式大脑

为了移除 JavaScript/TypeScript 运行时（Rivet）带来的运行时开销，以及 Python 带来的执行损耗与环境污染，本架构采用 **Rust（高性能外壳）+ Steel（确定性核心）** 的双层解耦拓扑。

```
[网络请求 (REST/WS)]
│
▼
[Tokio 异步事件调度] ──► [唤醒对应 ID 的 Rust Actor] ──(从 Postgres 事务中捞出最新 Context)
│
▼
[Actor 落盘休眠 (Scale to Zero)] ◄──(单次事务写入 Postgres) ◄── [Steel Lisp 虚拟机内核运行]
```

Rust 原生实现的 Actor 引擎利用 Tokio MPSC 管道建立低开销的 Host 外骨骼，避免多线程数据踩踏。每个 Actor 贴身绑定内存状态（无共享锁），闲时冬眠（Scale to Zero），被请求唤醒时瞬间拉起轻量级 Steel Lisp 虚拟机作为最高策略核心。Host 与 Steel Lisp 之间通过内存指针直接内嵌打通，零拷贝映射内存状态至 Lisp 变量。完整的零共享锁 Actor 实现详见 [§6.2 多语言网关](#62-多语言网关纯-rust-混合-actor-实现)，其中 `exec_steel_lisp` 即为单一 Steel 版本的实现。

## 3. 全景解构：大厂基建与新锐智能体平台竞品分析

在将本架构投入真实的商业 Web 服务之前，我们需要跳出极客视角，以严苛的工业级标准对当前市场上几款最具代表性的智能体运行平台进行高维度对账。

### 3.1 跨代技术矩阵横向速查表

| 维度 / 平台 | Google AX + Substrate | Rivet Actors | Windmill | 本架构 (Rust+Steel+PG/Fjall) |
|------------|----------------------|--------------|----------|------------------------|
| **底层核心技术栈** | Go / K8s 控制面 / 容器沙箱 | Rust + V8 Isolates + FoundationDB | Rust (内核) + Python/TS (执行) + Postgres | 纯 Rust + 嵌入式 Steel VM + PostgreSQL（单体）或 Fjall+Openraft（分布式） |
| **语言开销与大小** | 厚重（容器 Pod 级）| 中等（V8 进程隔离）| 中等（依赖的多运行时环境较庞大）| 极轻（单二进制文件，单会话 ~27MB 内存）|
| **语言生态友好度** | 模型无关，全语系支持 | 偏向 JavaScript/TypeScript | 极度偏向 Python | 移除 JS 污染，对 Rust/Lisp 原生极佳 |
| **冷启动 / 恢复延迟** | ~200 毫秒 (Pod 内存快照解冻) | 几毫秒级 (V8 虚拟机瞬时激活) | 数十毫秒 (FaaS 工作进程调度) | 启动为 Tokio 运行时初始化（毫秒级）；Actor 唤醒微秒级 (嵌入式 VM 瞬时创建) |
| **状态持有与防失忆** | 事件日志回放 (WAL / Replay) | 计算与 SQLite 存储同机共生 | 分布式异步工作队列（偏向无状态短时任务）| 按规模选择：单体模式（PG+JSONB 事务原子化）或分布式模式（Fjall LSM-Tree + Openraft 强一致复制）|
| **空格敏感/边界歧义** | 视容器内运行的特定语言而定 | 视 JS/TS 闭包习惯而定 | 存在 Python 缩进与类型隐式转化断层 | 零二义性（S-表达式小括号确定性边界）|

> Rivet 的 DX 层面详细对比见 [§12.1](#121-rivet-actors-的-dx-基线与-aura-的改进点)。

### 3.2 核心竞品深度诊断剖析

#### ① Google 阵营 (AX + Agent Substrate) —— 工业级重型装甲车

- **架构本质**：Google 旨在通过重构 Kubernetes (K8s) 底层调度来接管大规模企业级智能体。AX 负责通过事件日志（Event Logging）回放实现长期任务的"断线原地复活"，Substrate 负责利用 Pod 级别的增量内存快照实现处于挂起/等待状态的智能体自动冷冻（Scale to Zero），从而实现 97% 的硬件利用率提升。
- **技术痛点**：由于其以 Pod 容器为最小隔离单位，架构极度庞大且严重依赖 K8s 生态。对于中小型灵活的 Web 服务而言，其部署运维成本、冷启动响应延迟（~200ms 级别延迟在实时 Web 场景中依然过高）极不划算。

#### ② Rivet 阵营 (Rivet Actors) —— 游戏级高并发轻量战斗机

- **架构本质**：由高并发多人联机游戏基建演进而来。它抛弃了容器，采用纯 Rust 开发的底层运行时，内嵌 V8 Isolates 虚拟机孤岛。每一个智能体或长时会话在内存中就是一个高敏捷的 Actor，身边绑着一个内存级 SQLite，冷启动和休眠恢复达到了毫秒级。
- **技术痛点**：它捆绑了 JS 生态。尽管其宣称底层由 Rust 压榨性能，但目前其官方 SDK 和暴露的应用层 API 几乎全向 JavaScript/TypeScript（Bun / Node.js）倾斜。如果你反感 JS 的运行时开销、动态类型的松散以及 V8 的隐式内存损耗，Rivet 就会成为你技术洁癖上的巨大障碍。

#### ③ Windmill —— FaaS 基因的 Python 军火库

- **架构本质**：一个 Code-first 的高并发分布式异步任务编排与轻量 FaaS 平台。其最大的亮点是零侵入性——直接利用 Python 的类型提示（Type Hints）在后台自动生成标准的 JSON Schema，从而让普通的 Python 脚本可以直接作为 Tool 回调被大模型完美识别。
- **技术痛点**：它在本质上不是一个智能体常驻平台。它的底层是为"短时任务（Short-lived Jobs）"和定时脚本设计的。一个 Python 任务跑完即释放，缺乏像 AX 那样的事件状态机死守，也缺乏像 Rivet 这样的内存级同机共生（Co-located State）。在面对密集的"多智能体高频长时交互（Long-running Swarm loops）"时，无状态拉起的进程摩擦力显得过于厚重。

## 4. 架构升级：多语言混合动力运行时（Polyglot Embedded Harness）

本架构已从单一的 Steel Lisp 策略层演进为多模态混合运行时控制台。通过在 Rust 主 Actor 内部建立统一的"多语言网关接口（Polyglot Bridge）"，针对不同业务场景派出最合适的"工具人"，同时保持快速启动与 Scale to Zero 的核心设计。

### 4.1 多语言混合动力：用户选择指南

框架提供两种嵌入式脚本语言 + 一种沙箱运行时。**每个 Actor 只选一种脚本语言**，但**语言用途无硬限制**——下表中的"适用场景"仅为建议，开发者可根据实际需求自由分配。框架不做智能分发。

| 技术选型 | 类型 | 底层虚拟机/隔离机制 | 核心价值 | 适用场景 |
|---------|------|-------------------|---------|---------|
| **Steel (Lisp/Scheme)** | 脚本语言 | 字节码虚拟机 + 卫生宏 | 100% 无二义性，天然沙箱隔离（默认不带危险系统 I/O）| 高危策略防线、权限路由、复杂元编程 DSL |
| **Python (PyO3 内嵌)** | 脚本语言 | CPython 解释器内嵌 (GIL 内存级捕获) | 零开销复用全球最庞大的 AI/Data 生态圈 | 遗留 AI 代码资产重组、Tokenizer 矩阵计算、Numpy/HuggingFace 调用 |
| **Wasm (Wasmtime)** | 沙箱运行时 | Cranelift JIT 编译器 / 硬件沙箱 | 接近原生 CPU 性能，硬件级隔离 | 第三方不信任代码托管。wasm-gc + WASI 成熟后可扩展为统一运行时（详见 [Wasm 统一运行时](wasm-unified-runtime.md)） |
| **Rust 主外壳** | 宿主 | Tokio 异步协程 + MPSC 状态管道 | 低开销启动，27MB 内存，Scale to Zero 核心设计 | 分布式网关、系统 I/O、事件派发 |
| **持久化** | 存储层 | PostgreSQL（单体）或 Fjall + Openraft（分布式） | Shared Buffers + JSONB / LSM-Tree + Raft 共识 | 无外部缓存中间层，单次事务原子固化（按规模选择） |

**关键澄清**：脚本语言（Steel/Python）是 Actor 的业务逻辑载体，每个 Actor 选一种。Wasm 是隔离运行时，用于托管第三方不信任代码——编写语言通常是 Rust，但 host 不关心，只消费 `.wasm` 二进制。语言用途没有硬性规定——Steel 可以写业务逻辑，Python 可以写策略，只要开发者认为合适即可。

**语言限制的设计理由**：框架限制为 Python/Steel/Wasm 三种语言，不是"不能支持更多"，而是**限制 AI 的生成空间**。AI 生成代码的质量和语言复杂度成反比——语言越少、边界越清晰，AI 生成的代码越可靠。组件集封闭有限（enum 门卫 + 手动过 enum 加组件），和 Aura 的 Actor 枚举设计一致：AI 需要完整词汇表才能正确生成 JSON，语言数量影响判断准确率。不限制语言看似灵活，实际上是把选择负担推给了 AI 和开发者——AI 要在无限选项中选最优，开发者要在无限组合中调试兼容性。限制语言 = 降低认知复杂度 = 提高生成质量。

**引擎 vs 应用的评价标准**：Aura 是引擎和平台，不是应用。评价应用问"你解决了什么问题"（"用 PostgreSQL 就行了"是合理的应用层回答），评价引擎问"你能让人解决什么问题"（引擎的价值在于它赋能的上层应用的数量和复杂度）。用应用标准评引擎——"你不需要造发动机，马车够用了"——是层次混淆。引擎的对手不是 PostgreSQL，是其他引擎（Akka、Orleans、Tempo）。引擎的客户不是终端用户，是用引擎构建应用的开发者。

**Steel 的选择理由**：Steel 不是"特殊地位"，是三种语言之一。但 S-表达式有独特的工程价值：小括号 `()` 在物理上将所有作用域和运算边界锁定，空格敏感语言（Koto、Python）的"空格 = 语义"设计在团队协作和 Git 合并中制造不可预测性。Steel 的零二义性让 AI 生成的代码更可靠——Lisp 的同构性（Homoiconicity）让 Transformer 天然擅长生成，S-表达式是 AI 最容易正确生成的语法。但 Steel 数据密度低（GitHub 上代码少），AI 裸写时会调用标准 R7RS Scheme 的直觉产生幻觉——需要在 `CLAUDE.md` 中用 Few-Shot 约束。

→ 详见 [嵌入式脚本语言选型](embedded-script-languages.md)

**wasm-gc 成熟后的简化路径**：Rust → Wasm 现在已可用（不需要 wasm-gc），异步场景可由 Rust → Wasm 覆盖。wasm-gc 落地后，Kotlin/C# 等语言涌入 Wasm 生态，语言选择进一步丰富。Steel 和 PyO3 不受影响——Steel 的 S-表达式确定性边界是 Wasm 不能替代的；Python 过于动态、编译到 Wasm 极难甚至不可行，但也不需要——PyO3 进程内零拷贝调用开销本身就低，即时执行无须编译的交互模式是 Python 的固有价值。详见 [Wasm 统一运行时](wasm-unified-runtime.md)。

**语言无硬限制**：框架通过 PyO3 和 WASM 两个通道接入外部语言，理论上任何能编译到 WASM 或能通过 C ABI 调用的语言都可以进入 Actor 进程。框架内置 Steel/Python/WASM 三种运行时，但不阻止用户通过 PyO3 的 C 扩展机制接入其他语言（如 Lua、Julia）。

**外部调用**：框架通过 `ctx.invoke()` + 统一调用注册表提供同步调用（详见 [§6.13](#613-ctxinvoke-统一调用原语)）。Host 管控调用生命周期（超时、审计、可观测），Fluxora 负责 HTTP 请求。Actor 不直接通过 PyO3 调用外部系统——这绕过 Host 管控。进程内调用（PyO3/WASM）始终优先于跨进程调用。

**AI Agent/Harness 场景的分层**：

Agent 的逻辑分为两层——编排层和工具层，分别用不同语言实现：

- **Agent 主循环（Wasm/Rust）**：接收 → LLM 调用 → 工具调用 → 返回。稳定，性能优先。Wasm Actor 通过 host 函数调用 `ctx.invoke()`，Host 在 Wasm 挂起时执行异步操作，对 Wasm 透明。
- **工具接入**有两种模式：
  - **基于 SKILLs**：所有工具统一走 `perform()` 接口——结构化、可组合、可发现。内置操作（文件 I/O、网络请求、HTTP 客户端）和业务逻辑都封装为 Skill，Agent 通过统一接口调用。涌现式 Skill 通过 graph-memory 高权重子图自动聚类，Agent 运行时发现并调用。
  - **不基于 SKILLs**：直接 host 函数（Rust/Wasm）+ 脚本（Python）。轻量场景，不需要 Skill 框架的开销。通用稳定操作走 host 函数，业务特定逻辑走 invoke.toml 注册。

Agent 主循环在单个 Actor 实例中运行（partition key = session_id）。同一会话内的工具调用串行（Actor 单线程语义），不同会话并行。Agent 处理完请求后进入 `on_sleep`，状态落盘 Fjall，内存归零（Scale-to-Zero）。下次消息到达时 `on_wake` 激活恢复。流式输出通过高频 emit `agent.stream` 事件实现，Fluxora 转为 SSE/WebSocket 推送。

工具和 Skill 不是静态文件，而是 graph-memory 中高工具指数的子图——使用数据自动聚类涌现 Skill 边界，Agent 运行时通过向量搜索发现相关 Skill 边，按权重排序注入上下文。

#### 4.1.1 多 Skill 编排：Pipeline as Code

**问题**：当前 Agent 框架（Hermes、Agno 等）的标准模式是 LLM 实时编排——每调一个 Skill 都是一次独立推理，上下文在对话中传递。调到第 3、4 个 Skill 时，LLM 可能已经"忘了"第 1 个 Skill 返回的关键字段。同样的输入，不同次运行可能走不同路径。编排过程是黑箱，出错不可见。

**约束**：Aura 禁止 LLM 拼字符串生成代码（§4.1 语言限制的设计理由：限制 AI 的生成空间，提高生成质量）。但多 Skill 编排需要确定性执行。

**解法**：三层分离——LLM 输出结构化意图，Rust Pipeline Runner 确定性执行。

```
用户请求 → LLM（规划，输出 YAML pipeline 定义）
              → Pipeline Runner（Rust，确定性执行：读 YAML → 调 Skill → 传递数据）
```

LLM 生成的是 YAML，不是代码：

```yaml
# LLM 输出的 pipeline 定义（结构化意图）
pipeline:
  - skill: order-fetch
    params: { status: 1 }
    output: orders
  - transform:
      type: filter
      source: orders
      expression: "amount > 100"
      output: large_orders
  - skill: pricing-calculate
    input: large_orders
    output: pricing
  - skill: payment-create
    input: pricing
```

Pipeline Runner（Rust）直接执行，不经过 bash：

```rust
// pipeline_runner.rs — 确定性执行器
async fn run_pipeline(yaml: &str) -> Result<Value> {
    let pipeline: Pipeline = serde_yaml::from_str(yaml)?;
    let mut ctx: HashMap<String, Value> = HashMap::new();

    for step in &pipeline.steps {
        match step {
            Step::Skill { name, params, output } => {
                // Skill 接收 JSON stdin，输出 JSON stdout
                // Runner 只做管道：stdout(A) → stdin(B)，不解序列化
                let input = resolve_refs(params, &ctx);  // 替换 $ref 引用
                let result = execute_skill(name, &input).await?;
                ctx.insert(output.clone(), result);
            }
            Step::Transform { source, expr, output } => {
                // 只有转换步骤需要 Runner 解析 JSON
                let data = ctx.get(source).unwrap();
                let transformed = apply_transform(data, expr)?;
                ctx.insert(output.clone(), transformed);
            }
        }
    }
    Ok(ctx)
}
```

**关键设计**：Skill-to-Skill 时 Runner 只做管道（stdout → stdin），不解序列化。Skill 脚本内部已经处理 JSON 解析。只有 Transform 步骤需要 Runner 读写 JSON。

**三层分离的约束满足**：

| 约束 | 怎么满足 |
|:---|:---|
| Aura 禁止 LLM 生成代码 | LLM 输出 YAML，不是 Rust/bash |
| 编排需要确定性执行 | Runner 是确定性的，输入 YAML 输出固定执行路径 |
| 需要可观测 | YAML 可审计，Runner 日志记录每步输入/输出/Exit Code |
| 需要可干预 | 改 YAML 或改 Runner，不改 LLM |
| 需要复用现有 Skill | Runner 通过 stdin/stdout JSON 调用 Skill CLI，兼容现有 Typer 接口 |

**与实时编排的对比**：

| 维度 | 实时编排（现状） | Pipeline Runner |
|:---|:---|:---|
| LLM 参与次数 | 每步一次推理 | 规划时一次 |
| 上下文 | 持续增长，可能漂移 | 规划时完整，执行时无 LLM |
| 确定性 | 概率性，同样输入可能不同路径 | 确定性，同样 YAML 同样执行 |
| 出错成本 | 每次执行都可能出错 | 生成时付一次，修正后永远对 |
| 可观测性 | 黑箱 | YAML + Runner 日志全部可见 |

**前置条件**：每个 Skill 的接口声明需要有显式的输入/输出 schema（结构化参数定义），Runner 才能校验 pipeline 中的数据流转。当前 Skill 的 perform() 接口已有参数类型声明，output schema 需要补充。

### 4.2 多语言网关：纯 Rust 混合 Actor 实现

将四大嵌入式语言运行时全部内嵌进同一个 Tokio 异步 Actor 实例，打造多模态混合执行环境。整个调用过程完全在进程内存（In-Memory）中流式交织：

```rust
// src/main.rs
use std::collections::HashMap;
use tokio::sync::{mpsc, oneshot};

// 1. 引入各大新锐与传统嵌入式语言运行时
use steel::steel_vm::engine::Engine as SteelEngine;           // Lisp
use wasmtime::{Engine as WasmEngine, Store, Module, Instance}; // Wasm Core
use pyo3::prelude::*;                                          // Python 3

// 定义多语言任务调度协议
#[derive(Debug)]
pub enum PolyglotTask {
    RunLisp { script: String, input: Vec<u8>, respond: oneshot::Sender<Vec<u8>> },
    RunWasm { bytecode: Vec<u8>, input: Vec<u8>, respond: oneshot::Sender<Vec<u8>> },
    RunPython { code: String, func: String, input: Vec<u8>, respond: oneshot::Sender<Vec<u8>> },
}

pub struct MasterAgentActor {
    agent_id: String,
    receiver: mpsc::Receiver<PolyglotTask>,
    memory_context: ciborium::Value, // 动态结构化状态，CBOR Value 树
}

impl MasterAgentActor {
    pub fn new(agent_id: String, receiver: mpsc::Receiver<PolyglotTask>) -> Self {
        let mem = ciborium::Value::Map(vec![
            ("agent_status".into(), "active".into()),
            ("engine_version".into(), "2026.06".into()),
        ]);
        Self { agent_id, receiver, memory_context: mem }
    }

    pub async fn run_loop(mut self) {
        while let Some(task) = self.receiver.recv().await {
            match task {
                PolyglotTask::RunLisp { script, input, respond } => {
                    let res = self.exec_steel_lisp(&script, &input);
                    let _ = respond.send(res);
                }
                PolyglotTask::RunPython { code, func, input, respond } => {
                    let res = self.exec_embedded_python(&code, &func, &input);
                    let _ = respond.send(res);
                }
                // Wasm 也可以顺着这个模式在内存中拉起、执行、释放
                _ => {}
            }
        }
    }

    // =================================────────────────====================
    // 引擎 A: Steel Lisp (策略大脑，利用括号边界防错)
    // =================================────────────────====================
    fn exec_steel_lisp(&self, script: &str, input: &[u8]) -> Vec<u8> {
        let mut vm = SteelEngine::new();
        // 从 CBOR Value 中取出 agent_status
        let status = self.memory_context.get("agent_status")
            .and_then(|v| v.as_text()).unwrap_or("idle");
        vm.register_value("current-status",
            steel::steel_vm::as_values::AsRefSteelVal::as_ref_steel_val(status).unwrap());
        // input 是 CBOR 编码的动态数据，Host 解码后注入
        let input_val: ciborium::Value = ciborium::from_reader(input).unwrap();
        vm.register_value("input-raw", /* ... */);
        match vm.run(script) {
            Ok(out) => {
                let result = out.last();
                // 返回 CBOR 编码的输出
                let mut buf = Vec::new();
                ciborium::serialize(&result.to_string(), &mut buf).unwrap();
                buf
            }
            Err(e) => {
                let mut buf = Vec::new();
                ciborium::serialize(&format!("Lisp Error: {:?}", e), &mut buf).unwrap();
                buf
            }
        }
    }

    // =================================────────────────====================
    // 引擎 B: PyO3 Python (复用 AI/数据生态，内嵌内存直接吞吐，无进程摩擦)
    // =================================────────====================
    fn exec_embedded_python(&self, code: &str, function_name: &str, input: &[u8]) -> Vec<u8> {
        Python::with_gil(|py| {
            let status = self.memory_context.get("agent_status")
                .and_then(|v| v.as_text()).unwrap_or("idle");

            let activators = PyModule::from_code_bound(py, code, "agent_script.py", "agent_script");
            match activators {
                Ok(module) => {
                    // input 是 CBOR 编码的动态数据
                    let input_val: ciborium::Value = ciborium::from_reader(input).unwrap();
                    // Host 将 CBOR Value 转为 Python dict 传入
                    let py_input = cbor_to_pyobj(py, &input_val);
                    let args = (status, py_input);
                    if let Ok(func) = module.getattr(function_name) {
                        if let Ok(py_res) = func.call1(args) {
                            // 返回值转回 CBOR 编码
                            let result_val = pyobj_to_cbor(py, &py_res);
                            let mut buf = Vec::new();
                            ciborium::serialize(&result_val, &mut buf).unwrap();
                            return buf;
                        }
                    }
                    let mut buf = Vec::new();
                    ciborium::serialize(&"Python execution failed", &mut buf).unwrap();
                    buf
                }
                Err(e) => {
                    let mut buf = Vec::new();
                    ciborium::serialize(&format!("Python VM Error: {:?}", e), &mut buf).unwrap();
                    buf
                }
            }
        })
    }
}
```

### 4.3 彻底打破 NoSQL 宿醉：多语言混合下的"单体关系型死守"

很多跟风的架构师一看到这个方案支持多种嵌入式语言，第一反应就是："那我是不是得搞一个复杂的状态同步层，或者引入一个外部内存数据库来让这几种语言进行数据共享？"

**不。这恰恰是本架构最清醒、最精益的地方：**

- **内存空间的统一**：Steel 虚拟机、Wasmtime 和 Python 的 PyO3 全部作为动态链接或 C-Binding 静态编译进同一个 Rust 进程的内存地址空间。Actor 状态按字段分存（每个字段一个 Fjall key），各语言通过 Host 桥接读写——Host 按 key 读取字段，转为各语言的原生类型注入虚拟机。进程内调用，无跨进程 IPC，无网络序列化开销。

**状态分字段存储**：Actor 状态不是单个大 CBOR blob，而是按字段拆分为独立的 Fjall key：

```
state:{actor_id}:history   = [10000条记录]
state:{actor_id}:profile   = {...}
state:{actor_id}:settings  = {...}
```

handler 按需读写，不加载不需要的字段：

```python
def handle(ctx, user_message=None):
    if user_message:
        # 只加载 history 的前 100 条，不碰 profile 和 settings
        recent = ctx.state.get_list("history", offset=0, limit=100)

        # 只改了 history
        ctx.state.append("history", new_record)

        # 已落盘（WAL + memtable）
```


- **数据载荷的动态性**：Actor 的输入/输出在进程内为 `ciborium::Value`（内存值树），零序列化。跨语言边界（Steel/Python/Wasm VM）时序列化为 CBOR `Vec<u8>`——schema 不固定，支持嵌套对象、数组、数字、布尔。Steel Lisp、Python（cbor2 库）、Rust Host 都能编解码。不同于 Postcard（需要编译期 Rust 类型），CBOR 自描述，适合跨语言动态数据交换。

**CBOR 改进方向**：当前 CBOR 的键名（key name）在高频小消息场景有开销——每个字段都带完整字符串键名。改进方向：消除键名开销（键名索引化或列族式存储），动态列式布局（类似 LSM-Tree 的列族设计），但不能拖累查询性能。这是工程优化，不是架构变更——CBOR 作为跨语言数据交换格式的选择不变，只是编码效率提升。注意：这不是走向"schema 约束"的方向（Rust 结构体状态已经是类型安全的），而是保持动态性的同时压缩编码体积。

- **PostgreSQL 事务的收拢**：多语言在一轮交互中通过 `ctx.state` 操作逐字段写 WAL 落盘。Fjall 的 LSM-Tree 天然支持高频小写入。分布式模式下，Openraft 将 Fjall 的写操作复制到所有节点，不需要 PostgreSQL。单体模式下，Actor 休眠时 Fjall 状态可选同步到 PostgreSQL 备份。

**网线里没有多轮的 RTT 消耗，没有外部缓存的易失性风险，没有多库同步的分布式 Bug。**

用 Rust 搭了外骨骼，用 Lisp 锁定了边界，用 Python 接入了生态，用 Wasm 隔离了黑盒，持久化层在 PostgreSQL（单体模式）和 Fjall + Openraft（分布式模式）之间按需选择。

## 5. 架构终极演进：分布式存算一体（Fjall + Openraft）

### 5.1 从单体 PostgreSQL 到分布式存算一体的逻辑跃迁

虽然 PostgreSQL 的 JSONB 聚合已经替代了外部缓存，但在某些极端场景下（如智能体状态高度 KV 化、需要多节点强一致性复制），继续背负 PostgreSQL 的 SQL 解析器、查询优化器、连接池和 MVCC 仍然是"大炮轰蚊子"。

**终极方案**：将 Fjall（纯 Rust LSM-Tree KV 引擎）+ Openraft（纯 Rust Raft 共识协议）绑定，在主程序内部直接构建出一个高可用、可复制、存算一体的"分布式智能体主权操作系统"。

### 5.2 分布式存算一体架构拓扑

```
[ 客户端网络请求 ]
│ (打向集群中任意节点)
▼
┌───────────────────────┐
│ OpenRaft 集群节点      │ ◄───► [ 分布式共识网络 (心跳/日志同步) ]
└───────────┬───────────┘
            │ 1. 验证 Leader 身份 / 提交状态提案 (Propose)
            ▼
┌───────────────────────┐
│ Raft 确定性状态机      │
└───────────┬───────────┘
            │ 2. 应用已达共识确认的指令 (Apply Committed Command)
            ▼
┌───────────────────────┐
│ Fjall LSM-Tree 存储    │ (紧贴本地 NVMe 磁盘快速写入)
└───────────┬───────────┘
            │ 3. 零网络延迟：直接在当前进程内存中拉起虚拟机
            ▼
┌───────────────────────┐
│ 多模态智能体核心运行时  │ ──► [ 解释执行 Steel / PyO3 / Wasm ]
└───────────────────────┘
```

### 5.3 为什么 Fjall + Openraft 是黄金组合？

- **Fjall 击败 RocksDB**：传统 RocksDB 是 C++ 写的，在 Rust 项目中带来跨平台编译地狱、臃肿内存开销和厚重 FFI 桥接。Fjall 是纯 Rust LSM-Tree 存储引擎，内存占用极小、自带布隆过滤器实现微秒级查找，且天生与 Tokio 异步生态 100% 融合。

- **Openraft 击败外部多库集群**：Openraft 负责接管分布式日志与多机复制。当客户端要求某个智能体更新状态时，Openraft 会在多台服务器之间广播这个变动。一旦集群多数派（Quorum）确认收到，该指令就会被原子化地应用到本地的 Fjall 中。

- **"零网络序列化损耗"**：底层数据库只是内嵌在 Rust 进程里的一行代码库（`struct Keyspace`），智能体读写状态是彻底的**进程内调用（In-Process Call）**。数据在物理磁盘到 Steel 虚拟机之间传递，走的是内存指针，速度直逼硬件物理极限。

→ 详见 [Redis 批判：RESP 协议 vs 二进制序列化](redis-critique.md#resp-协议-vs-二进制序列化)

### 5.4 核心源码实现：Openraft 状态机挂载 Fjall

```rust
// src/store.rs
use std::sync::Arc;
use openraft::{RaftStateMachine, StateMachineData, StateMachineError, LogId};
use fjall::{Config, Keyspace, PartitionHandle};
use serde::{Serialize, Deserialize};

// 1. 定义具备确定性、可全网广播的状态转换指令
#[derive(Serialize, Deserialize, Clone, Debug)]
pub enum RaftCommand {
    UpdateActorState {
        agent_id: String,
        serialized_context: Vec<u8>, // CBOR 编码的 Actor 状态快照
    },
    TerminateActor {
        agent_id: String,
    },
}

// 2. 核心分布式状态机结构体（内嵌包装 Fjall 物理分区）
pub struct FjallStateMachine {
    pub keyspace: Keyspace,
    pub actor_partition: PartitionHandle, // 专门存放智能体状态的分区
    pub meta_partition: PartitionHandle,  // 专门存放 Raft 元数据（如最后执行的 Log ID）的分区
}

impl FjallStateMachine {
    pub fn new(path: &str) -> Self {
        // 打开/创建纯 Rust 的 LSM-Tree 物理存储区
        let keyspace = Keyspace::open_default(path).expect("无法初始化 Fjall 物理存储区");
        let actor_partition = keyspace.open_partition("actors", Config::default())
            .expect("无法开辟智能体物理状态分区");
        let meta_partition = keyspace.open_partition("meta", Config::default())
            .expect("无法开辟集群元数据分区");
        Self { keyspace, actor_partition, meta_partition }
    }
}

// 3. 实现 Openraft 要求的相关核心数据 Trait
#[derive(Debug, Serialize, Deserialize)]
pub struct StateMachineResponse {
    pub success: bool,
}

impl StateMachineData for FjallStateMachine {}

// 4. 结合：将 Openraft 的日志应用钩子直接写入本地 Fjall 物理磁盘中
impl RaftStateMachine<openraft::TypeConfig> for Arc<FjallStateMachine> {
    async fn applied_state_id(
        &self,
    ) -> Result<Option<LogId<u32>>, StateMachineError<openraft::TypeConfig>> {
        // 从本地 Fjall 元数据分区中读取上一次成功执行的 Raft 日志 ID，防止状态漂移
        if let Ok(Some(bytes)) = self.meta_partition.get("last_log_id") {
            let log_id = postcard::from_bytes(&bytes).unwrap();
            Ok(Some(log_id))
        } else {
            Ok(None)
        }
    }

    async fn apply<I>(
        &self,
        entries: I,
    ) -> Result<Vec<StateMachineResponse>, StateMachineError<openraft::TypeConfig>>
    where
        I: IntoIterator<Item = openraft::Entry<openraft::TypeConfig>> + Send,
    {
        let mut responses = Vec::new();

        // 顺序遍历由分布式共识网络推流过来的、已达多数派确认的日志实体
        for entry in entries {
            let log_id = entry.log_id;
            if let openraft::EntryPayload::Normal(cmd) = entry.payload {
                match cmd {
                    RaftCommand::UpdateActorState { agent_id, serialized_context } => {
                        // 触发 Fjall 的单次写入，自带崩溃保护
                        self.actor_partition.insert(agent_id.as_bytes(), serialized_context)
                            .expect("Fjall 本地物理写入异常，触发系统级错误保护");
                        responses.push(StateMachineResponse { success: true });
                    }
                    RaftCommand::TerminateActor { agent_id } => {
                        self.actor_partition.remove(agent_id.as_bytes())
                            .expect("Fjall 本地擦除异常，触发系统级错误保护");
                        responses.push(StateMachineResponse { success: true });
                    }
                }
            }

            // 实时将当前最新执行成功的 LogId 固化落盘
            let log_bytes = postcard::to_stdvec(&log_id).unwrap();
            self.meta_partition.insert("last_log_id", log_bytes).unwrap();
        }

        // 命令 Fjall 执行一次物理持久化刷盘 (Sync to NVMe)
        self.keyspace.persist().unwrap();
        Ok(responses)
    }
}
```

### 5.5 用户意志主导的多模态路由机制（User-Driven Polyglot Routing）

在传统的 FaaS（如 Windmill）或重型智能体框架中，通常是由"系统架构或框架"死板地规定："这个步骤必须用 Python 跑，那个步骤必须用 JS 跑"。这本质上是对 AI 自由度和人类开发意志的束缚。

在本架构（Fjall + Openraft + Polyglot Core）中，我们彻底打破这种死板的框架绑架。**用什么语言来执行决策、重构代码或处理数据，完全由"用户下达的指令"或"AI 智能体自发生成的策略"动态决定。** Rust 的主 Actor 控制台只负责提供一个没有任何偏见的、纯粹的多语言执行沙箱（Polyglot Engine Room）。

#### 用户驱动的状态指令协议

```rust
// src/store.rs - 用户驱动的多语言执行提案
#[derive(Serialize, Deserialize, Clone, Debug)]
pub enum EngineType {
    Steel,  // 嵌入式脚本：Lisp 策略引擎
    PyO3,   // 嵌入式脚本：Python AI/数据引擎
    Wasm,   // 沙箱运行时：第三方不信任代码（通常用 Rust 编写 .wasm）
}

#[derive(Serialize, Deserialize, Clone, Debug)]
pub enum RaftCommand {
    UpdateActorState {
        agent_id: String,
        serialized_context: Vec<u8>,
    },
    TerminateActor {
        agent_id: String,
    },
    // 用户或 AI 动态发起的任意语言执行提案
    DispatchUserScript {
        agent_id: String,
        engine: EngineType,      // 用户决定的语言类型
        script: String,          // 用户提交的动态脚本
        target_function: String, // 要调用的目标函数
        input_payload: Vec<u8>,  // CBOR 编码的动态输入数据
    },
}
```

当 Openraft 集群对用户提交的 `DispatchUserScript` 达成共识后，本地状态机根据用户选择，将物理内存指针映射到对应的语言虚拟机。运行期交接棒流程为：从 Fjall LSM-Tree 中读出 CBOR 编码的 Actor 状态（零网络延迟）→ 解码为 `ciborium::Value` → 根据用户选择的 EngineType 拉起对应的嵌入式虚拟机（Steel/PyO3/Wasm），Host 从 CBOR Value 中取出字段注入虚拟机执行 → 更新结果状态 → 编码回 CBOR，作为提案提交给全网 Openraft 节点固化。具体的多语言执行逻辑已在 [§6.2](#62-多语言网关纯-rust-混合-actor-实现) 的 `exec_steel_lisp` 和 `exec_embedded_python` 中完整实现，此处不再重复。

#### 用户驱动模式的工程爽点

1. **用户拥有"语法免冲突权"**：
   如果你要写一段需要跟大模型频繁交互、且在 Neovim 里进行深度协作的核心控制策略，你可以立刻下达指令："这一步我用 Steel Lisp 跑"。这能保证你的大脑完全沉浸在括号的几何边界里，彻底免受 Koto 那种隐式空格语义地雷的折磨。

2. **AI 智能体拥有"生态自选权"**：
   当你托管在云端的 Hermes 大脑发现："接下来的任务需要去读取一个复杂的深度学习 .bin 权重文件，或者分析一段遗留的 PyTorch 矩阵"时，AI 会自己在分布式提案里写明：`engine: EngineType::PyO3`。它通过纯粹的内存指针，直接在当前 Rust 进程里无缝吃掉 Python 的 AI 生态。

3. **多语言在 Fjall 磁盘里的统一**：
   不管用户刚才任性地选了 Lisp 还是 Python，它们对智能体状态的修改（Mutation），最终都会被反序列化回最基础的二进制内存块（`Vec<u8>`）。经由 Openraft 的强一致性网络广播达成多数派共识后，单次落盘、高度压缩地锁进本地的 Fjall LSM-Tree 物理硬盘中。

**框架不再是法官，框架只提供执行能力；用户和 AI 的动态意志决定哪种语言在这一毫秒登上多模态内存舞台。**

### 5.6 为什么不用 Redis

通过将 Fjall + Openraft + 多模态嵌入式运行时揉进同一个单体二进制文件中，消除了现代架构中常见的冗余和嵌套。如果把这套架构里的 Fjall + Openraft 剔除换成 Redis，整个系统将发生严重的**底层架构退化（Structural Regression）**：

| 核心维度 | Fjall + Openraft + 多模态嵌入沙箱（原架构） | 替换为 Redis 的退化形态 |
|---------|---------------------------------------------|---------------------------|
| **集群内聚度** | 纯 Rust 单体。通过 Raft 日志实现全网自愈和强一致。 | 割裂的应用服务器矩阵 + 外部独立的 Redis 实例 + 复杂的 Redis Sentinel/Cluster 运维线。 |
| **计算局部性** | 计算紧贴存储（Compute Near Data）。多语言虚拟机内存指针直接映射磁盘 Buffer。零网络 RTT，走 CPU 总线速度。 | 计算远离存储。网关每次收请求必须打开 TCP 连接，数据打包成文本型 RESP 协议跨进程传输，重新背负 1.0ms–3.0ms 的网络往返延迟（RTT）。 |
| **零拷贝** | Rust 生命周期系统（`serde(borrow)`）让多语言虚拟机直接用指针读取磁盘 Buffer，无新内存申请。 | 数据必须在 Redis 侧打包、经 Socket 传输、在 Rust 客户端解包，在堆内存申请新空间大块拷贝。高频内存分配与 GC 开销。 |
| **多线程并行** | Openraft 共识日志基于 Tokio 异步协程多核高并发，Fjall 多线程 LSM 异步刷盘，Steel/PyO3 各走独立 OS 线程，全网无中心化吞吐卡死。 | Redis 单线程事件循环，一旦运行复杂 Lua 脚本或重度 CPU 计算，全球所有其他读写请求瞬间死锁卡死。 |
| **Scale-to-Zero** | Fjall LSM-Tree 将不活跃冷状态高度压缩为磁盘 SSTables。Agent 睡着时 RAM 消耗 0 字节。百万级 Agent 也无内存压力。 | Redis 纯内存数据库，所有数据全量躺在物理内存里。智能体扩大到 1 万或 100 万个时，硬件账单指数级爆炸。 |
| **Token 优化** | 后台静默触发"环境梦境整理（Ambient Consolidation）"，自动将长时文本 Sink 进 S3。 | 必须在应用端写复杂的定时任务（Cron Jobs），高频跨网络去捞内存数据再执行归档。 |
| **系统复杂度** | 极致极简。1 个可执行文件，0 个外部数据库配置文件，解压即组网。 | 高运维负荷。需要维护多套发布流水线、外部连接池监控以及缓存击穿/雪崩的防御代码。 |

**一句话总结**：把 Fjall + Openraft 换成 Redis，是用系统长期的"运行期高延迟、带宽开销、内存账单膨胀以及单线程死锁风险"，去仅仅换取"在第一周开发时少写几行 Openraft 节点连接代码"的短暂偷懒。没有 Redis 集群的心跳同步紊乱，没有 PostgreSQL 昂贵的连接池耗尽与 SQL 树解析开销，没有 JavaScript（Rivet）运行时的冗余与弱类型妥协。在 Rust 语言的底层安全原语之上，构建了一座全网络自动强一致性状态锁定的分布式智能体系统。

**Lua 脚本的工程断层**：Redis 为挽救吞吐量引入的 Lua 脚本，除了单线程死锁风险外，还导致主技术栈（Rust/Go）与脚本层发生工程学与调试断层——失去强类型保护、单元测试和 IDE 感知提示。

## 6. 场域模型：Actor 间交互与外部世界

### 6.1 核心设计

传统 Actor 框架（Akka、Erlang、Actix）的通信原语是 ActorRef 直发（tell/ask）——调用者必须知道目标 Actor 的地址。Aura 采用不同的原语：Actor 之间不直接寻址，而是通过共享的"场域"（Event Realm）用 `emit` / `on` 交互。

场域是引擎内部的事件空间。Actor 通过 `on(name, fn)` 订阅事件、`emit(name, data)` 发射事件。发射者不关心谁处理，处理者不关心谁发射。外部世界（HTTP/WS）的协议层由调用方（Fluxora、网关）处理——Fluxora 将外部请求转成 `emit()`，将 `on()` 事件转成 HTTP 响应或 WS 推送。Aura 引擎本身不碰 HTTP/WS。

**开发者体验方向**：Fluxora 不做 MQ（Kafka/NATS 已去掉），只做 HTTP/WS 协议桥接。进一步的 DX 目标：Web 控制台 + 嵌入 VSCode，开发者直接在浏览器里写 Python Actor。Actor 之间只管发消息，存数据由框架处理。这是 FaaS + Web 框架的融合形态——不是"给你一个数据库让你写 CRUD"，而是"给你一个事件空间让你编排 Actor"。

### 6.2 场域拓扑

```
┌─────────────── Event Realm (namespace: default) ───────────────┐
│                                                                │
│  Ingress（入口）                                                │
│  ┌──────────┐                                                  │
│  │ Fluxora  │  emit("order_created", data)                     │
│  │ Webhook  │───────┐                                          │
│  └──────────┘       │                                          │
│                      ▼                                         │
│               ┌────────────┐     emit("order_completed")      │
│               │  Actor A   │──────────────────┐                │
│               │ on("order_ │                   │                │
│               │  created") │                   ▼                │
│               └────────────┘          ┌────────────┐          │
│                                        │  Actor B   │          │
│  ┌──────────┐                          │on("order_  │          │
│  │  Actor C │◄──emit("inventory_upd.") │completed") │          │
│  │ on("inv_ │         ┌────────────┐   └────────────┘          │
│  │  updated")│        │  Actor D   │        │                  │
│  └──────────┘        │on("order_  │        │ emit("notif.")   │
│       │ emit("shipped")│completed")│        │                  │
│       ▼               └────────────┘        ▼                  │
│  ┌──────────┐                                   Egress（出口）  │
│  │  Actor E │                          ┌──────────┐            │
│  │on("ship- │                          │ Fluxora  │            │
│  │ ped")    │                          │on("notif"│ → WS push │
│  └──────────┘                          └──────────┘            │
└────────────────────────────────────────────────────────────────┘
```

- 事件名标记入口/出口语义（Fluxora 的模式）
- Actor 不感知协议（HTTP/WS），只收发事件
- 跨节点事件经 Openraft 共识网络传递，与状态复制共用同一通道

### 6.3 Actor 定义接口

```
set(<lang>, <script/wasm>)
```

提交或更新一个 Actor **定义**（类型）。`lang` ∈ {Steel, Python, Wasm}。脚本内导出 `interface_schema()` + 单一入口函数（事件名映射为函数参数）。运行时调用 `set()` 可热替换 Actor 实现——不仅换行为，还换语言。

`set()` 定义的是类型，不是实例。Actor 实例由 Realm 根据 partition key 按需激活（详见 [§6.11](#611-actor-实例化与分片)）。

**脚本持久化**：`set()` 提交的脚本内容（或 Wasm 字节码）存储在 Fjall 中，不从文件系统读取。这样 Actor 定义本身是持久化、可复制的状态——跨节点部署时，Openraft 自动将脚本同步到所有节点，不需要在每个节点上手动放置脚本文件。Fjall 的 LSM-Tree 天然支持版本化，每次 `set()` 保留新版本，旧版本可回滚。脚本条目附带元数据（提交时间、语言类型、版本号、提交者、内容哈希），存储结构：

```
fjall partition: "actor_defs"
  key:   <actor_type_name>
  value: CBOR { lang, script_bytes, version, content_hash, committed_at, committed_by }
```

**去重**：`set()` 提交前先计算 `script_bytes` 的哈希（content_hash），与 Fjall 中最新版本的 `content_hash` 比较——相同则忽略，不写入新版本。避免 CI 重复部署或无意义的热重载。

`on()` handler 在 Actor 实例激活时从 Fjall 读取最新版本的脚本，加载到对应 VM 执行。实例驱逐后，下次激活重新从 Fjall 读取。

### 6.4 interface_schema()

`interface_schema()` 是 Actor 的**统一契约**——声明接收事件，以及可选的发射事件清单。

**核心洞察：事件名就是引用。** 当 `interface_schema()` 声明 Actor B 接收 `"charge"` 事件时，任何人 emit `"charge"` 就是在引用 B。事件名 = 引用，partition key = 实例定位。不需要单独的 `invoke` / `direct_send` 原语——emit 本身就是引用调用。

```python
def interface_schema():
    return {
        "receives": {
            "add_to_cart": {
                "mode": "on",
                "key": "user_id",
                "params": {"type": "object", "properties": {
                    "user_id": {"type": "string"},
                    "item": {"type": "object"}
                }}
            },
            "remove_from_cart": {
                "mode": "on",
                "key": "user_id",
                "params": {"type": "object", "properties": {
                    "user_id": {"type": "string"},
                    "item_id": {"type": "string"}
                }}
            }
        },
        "returns": {
            "type": "object",
            "properties": {
                "cart_count": {"type": "integer"},
                "total": {"type": "number"}
            },
            "required": ["cart_count", "total"]
        },
        "emits": ["cart_updated"]
    }
```

**`receives` 中每个事件声明：**
- `mode`：触发模式——`on`（单事件）、`on_join`（多事件收齐）、`on_batch`（同类打包）、`on_debounce`（去抖）。详见 [§6.6 事件组合原语](#66-事件组合原语)
- `key`：partition key 字段——Realm 按此字段值路由到 Actor 实例
- `params`：入参 JSON Schema

**`returns` 是 Actor 级别的单一返回值声明。** 每个 Actor 有一个入口函数，多个事件映射到多个参数，返回一个值：

```python
# Actor 入口函数 — 多事件映射到多参数
def handle(ctx, add_to_cart=None, remove_from_cart=None):
    if add_to_cart:
        ctx.state["items"].append(add_to_cart["item"])
    if remove_from_cart:
        ctx.state["items"] = [i for i in ctx.state["items"]
                              if i["id"] != remove_from_cart["item_id"]]
    emit("cart_updated", {"user_id": ..., "items": ctx.state["items"]})
    return {"cart_count": len(ctx.state["items"]),
            "total": sum(i["price"] for i in ctx.state["items"])}
```

- 声明了 `returns` 的 Actor 支持 `ctx.invoke()` 同步调用——调用者通过 Actor 名获取返回值
- 不声明 `returns` 的 Actor 只能通过 `emit()` 异步触发（fire-and-forget）
- `ctx.invoke("cart_actor", data)` → 调用入口函数 → 返回单一值

| 用途 | 说明 |
|------|------|
| **内部事件** | 声明事件名即可，消费方的 `on` 已包含 schema |
| **安全管控** | Realm 强制白名单：只允许 `emits` 中声明的事件名被发射，未声明的拒绝 |
| **图表生成** | 声明的事件名可用于自动绘制 Actor 间事件流拓扑图 |

`emits` 声明的是**约束**——Actor 只能发射列表中的事件名。emit 时 Realm 校验事件名是否在 `emits` 声明中，未声明的拒绝发射。不声明 `emits` = 不能 emit 任何事件（纯接收型 Actor）。

**外部订阅者**：Fluxora、Webhook、分析系统等外部消费者是 Realm 的订阅者——和 Actor 共享同一个路由表，使用相同的 `RouteMode`（on / join / batch / debounce），只是投递目标不同（Actor → 入口函数，外部 → HTTP/WS/Webhook）。Realm 的 `emits` 约束了哪些事件可出界，外部系统从声明列表中选择订阅。

```
# Fluxora：去抖推 WS
Route { target: "fluxora", event: "cart_updated", mode: Debounce(300ms) }

# Webhook：单事件触发
Route { target: "webhook:order-service", event: "order_completed", mode: On }

# 分析系统：join 后批量发送
Route { target: "analytics", events: ["payment", "inventory"], mode: Join(5s) }
```

外部订阅者不需要声明 `interface_schema()`——它们是 Realm 的消费者，不是 Actor。路由表由 Realm 管理，外部系统通过配置（或 API）声明订阅关系。

**事件名 = 引用（路由机制）：**

Host 启动时调用 `interface_schema()`，构建事件路由表：

```
事件名 → partition key 字段 → params schema → returns? → Actor 定义 → on handler
```

当 A emit `"add_to_cart"` 时：
1. Realm 查路由表 → `"add_to_cart"` 只有 Cart Actor 注册了
2. 从事件数据提取 `key` 字段值（`user_id`）→ 路由到 Cart Actor 的对应实例
3. 校验载荷是否符合 `params` schema
4. 投递到 `on("add_to_cart")` handler

A 通过事件名精确指向了 Cart Actor——事件名就是引用，schema 就是类型签名。Host 在投递前可校验事件载荷是否符合 schema；Fluxora 可从 schema 自动生成 TypeScript 类型定义。

**ctx.invoke() 同步调用：**

Actor 声明了 `returns`（JSON Schema）时，调用者可以通过 `ctx.invoke()` 同步获取 Actor 入口函数的 return 值：

```python
# 调用方
result = await ctx.invoke("cart_actor", data={"user_id": "42", "item": {...}})
# result = Actor 入口函数的 return 值

# 被调用方（Cart Actor 入口函数）
def handle(ctx, add_to_cart=None, remove_from_cart=None):
    if add_to_cart:
        cart = add_item(add_to_cart["user_id"], add_to_cart["item"])
    return {"cart_count": len(cart), "total": sum(i["price"] for i in cart)}
```

Realm 内部通过 reply_to 机制实现同步：emit 事件 + `__reply_to` → 等待入口函数 return → 返回给调用者。入口函数只需正常 `return`，不需要关心 reply_to 细节。

入口函数同时可以 `emit` 通知其他 Actor——`return` 给调用者，`emit` 给系统，各走各的路。

**fire-and-forget：**

没有 `returns` 的 Actor 只能通过 `emit()` 触发。调用者不等待返回值，入口函数的 return 值被丢弃。

**emit 到有 `returns` 的 Actor（允许）：**

`returns` 声明的是**能力**（"这个 Actor 可以返回值"），不是**约束**（"只能同步调用"）。`emit()` 到有 `returns` 的 Actor 是合法的——入口函数正常执行，return 值丢弃。这解锁了触发但不等待的模式：

```python
# 你关心的是"这件事发生"，不关心结果
emit("charge", {"user_id": "42", "amount": 100})
# 入口函数执行了（扣款、发通知），return 值丢弃

# 同一 Actor 也可以同步调用
result = await ctx.invoke("charge_processor", {"user_id": "42", "amount": 100})
```

`interface_schema()` 也是热重载的入口：脚本修改 → 重新加载 → 重新 `interface_schema()` → 路由表更新。

### 6.5 事件 API

**emit(name, data)**：向 Realm 提交事件。data 是 `ciborium::Value`（内存值树）。进程内 Actor ↔ Actor 传递时保持 Value 形态，通过 Tokio MPSC channel clone，零序列化。序列化只在跨边界时发生：

| 边界 | 格式 | 说明 |
|------|------|------|
| 进程内 Actor ↔ Actor | `ciborium::Value` | 内存 clone，零编解码 |
| Actor → Fjall 持久化 | CBOR bytes | `ciborium::serialize()` 写入 LSM-Tree |
| 跨节点 Openraft 复制 | CBOR bytes | 网络传输需要序列化 |
| Actor → Fluxora（HTTP/WS） | JSON | 外部系统消费 JSON |
| Actor → Webhook | CBOR 或 JSON | 按配置选择 |

**on(name, fn)**：注册事件监听。每个 Actor 有一个入口函数，事件名映射为函数的 keyword 参数——多个 `on` 声明编译为单一 dispatch 函数。Realm 内部按事件名路由到对应的参数。状态共享通过 `ctx`（Actor 的统一状态树），不依赖闭包捕获。

**通配符监听**：`on()` 支持前缀通配符 `"prefix.*"`，匹配所有以 `prefix.` 开头的事件。与 etcd 的 key 前缀匹配类似——适用于日志、审计、投影 Actor 等需要监听一类事件的场景：

```python
# 监听所有 order 相关事件
@on("order.*")
def handle_order_events(ctx, data):
    log(ctx, data)
    # 投影 Actor：把 order.* 事件聚合到部门统计
    emit("dept_stats_updated", aggregate(data))
```

通配符声明和精确声明可以共存——事件同时匹配通配符参数和精确参数，各自独立路由到入口函数的对应参数。通配符声明不指定 partition key（它监听一类事件，不绑定具体实体），路由到场域中该 Actor 的单例实例。

**通配符声明**：通配符参数不在 `interface_schema()` 的 `receives` 中声明具体事件名，而是用 `wildcard_receives` 列表：

```python
def interface_schema():
    return {
        "receives": {
            "add_to_cart": {"key": "user_id", "schema": {...}},
            "remove_from_cart": {"key": "user_id", "schema": {...}}
        },
        "wildcard_receives": ["order.*"],  # 前缀通配符
        "emits": ["dept_stats_updated"]
    }
```

**前缀匹配规则**：`on("order.*")` → 前缀 `"order."`，匹配 `order_created`、`order_completed`、`order_cancelled`。不匹配 `order`（无 `.` 分隔）。不支持后缀通配（`*.created`）或中间通配（`order.*.created`）——只有前缀匹配，与 etcd 一致。

**路由表结构**：

```rust
pub struct EventRouter {
    exact: HashMap<String, Vec<Route>>,      // 精确匹配：事件名 → 路由规则
    wildcard: Vec<WildcardRoute>,             // 通配符匹配：前缀 → 路由规则
}

struct Route {
    actor_type: String,
    partition_key_field: String,              // 从事件数据中取哪个字段作为 key
    handler: HandlerRef,
}

struct WildcardRoute {
    prefix: String,                           // "order."（去掉 .* 后的前缀）
    actor_type: String,
    handler: HandlerRef,
    // 无 partition_key_field —— 通配符 handler 是单例实例
}
```

通配符用 `Vec` 而非 HashMap，因为匹配是反向的（给定事件名，找哪些前缀能匹配），HashMap 帮不上忙。通配符数量通常很少（几个投影 Actor），线性扫描足够。如果通配符多到成为瓶颈，再换 Trie。

**分发路径**：

```rust
impl RealmHost {
    async fn dispatch_event(&self, event_name: &str, data: Value) {
        let mut matched_routes = Vec::new();

        // 1. 精确匹配
        if let Some(routes) = self.router.exact.get(event_name) {
            matched_routes.extend(routes.iter().cloned());
        }

        // 2. 通配符匹配：线性扫描所有通配符前缀
        for wr in &self.router.wildcard {
            if event_name.starts_with(&wr.prefix) {
                matched_routes.push(Route {
                    actor_type: wr.actor_type.clone(),
                    partition_key_field: String::new(),  // 无 partition key
                    handler: wr.handler.clone(),
                });
            }
        }

        // 3. 无匹配 → dead event
        if matched_routes.is_empty() {
            self.dead_events.push(event_name, data);
            return;
        }

        // 4. 对每个匹配的路由，投递到对应 Actor 实例
        for route in matched_routes {
            if route.partition_key_field.is_empty() {
                // 通配符 handler → 单例实例（固定 key "__singleton__"）
                let instance = self.get_or_activate(
                    &route.actor_type, "__singleton__",
                );
                instance.mailbox.send(Event::new(event_name, data.clone()));
            } else {
                // 精确 handler → 按 partition key 路由
                let key = data.get(&route.partition_key_field)
                    .and_then(|v| v.as_str())
                    .unwrap_or("__default__");
                let instance = self.get_or_activate(
                    &route.actor_type, key,
                );
                instance.mailbox.send(Event::new(event_name, data.clone()));
            }
        }
    }
}
```

**通配符参数的实例化**：通配符参数不绑定 partition key，路由到固定 key `"__singleton__"` 的实例——整个 Actor 类型只有一个实例。这与投影 Actor 的场景一致：一个 DeptStatsActor 实例监听所有 `order.*` 事件，持续聚合。

**精确 + 通配符同时匹配**：一个事件可以同时命中精确参数和通配符参数，各自独立投递：

```
emit("order_created", {"user_id": "A", ...})

→ 精确匹配：CartActor 的 on("order_created")，partition key = "A"
→ 通配符匹配：DeptStatsActor 的 on("order.*")，单例实例

两个 Actor 实例各自独立处理，互不阻塞。
```

```python
from aura import emit

# Actor 入口函数 — 多事件映射到多参数
def handle(ctx, add_to_cart=None, remove_from_cart=None):
    if add_to_cart:
        ctx.state["items"].append(add_to_cart["item"])
        # 已落盘（WAL + memtable）
    if remove_from_cart:
        ctx.state["items"] = [i for i in ctx.state["items"]
                              if i["id"] != remove_from_cart["item_id"]]
        # 已落盘（WAL + memtable）
    emit("cart_updated", {"user_id": ..., "items": ctx.state["items"]})
```

```scheme
;; Steel Lisp — schema 内嵌在 on 调用中，Realm 从 AST 自动提取

(on "add_to_cart"
  (schema
    (key "user_id")
    (params (hash 'type "object"
                  'properties (hash 'user_id (hash 'type "string")
                                      'item (hash 'type "object"))
                  'required '("user_id" "item"))))
  (lambda (ctx data)
    (ctx-update! ctx "items"
      (lambda (items) (append items (list (hash-ref data "item")))))
    ;; 已落盘（WAL + memtable）
    (emit "cart_updated"
      (list (cons "user_id" (hash-ref data "user_id"))
            (cons "items" (ctx-ref ctx "items"))))))

(on "remove_from_cart"
  (schema
    (key "user_id")
    (params (hash 'type "object"
                  'properties (hash 'user_id (hash 'type "string")
                                      'item_id (hash 'type "string"))
                  'required '("user_id" "item_id"))))
  (lambda (ctx data)
    (ctx-update! ctx "items"
      (lambda (items)
        (filter (lambda (i) (not (equal? (hash-ref i "id") (hash-ref data "item_id"))))
                items)))
    ;; 已落盘（WAL + memtable）
    (emit "cart_updated"
      (list (cons "user_id" (hash-ref data "user_id"))
            (cons "items" (ctx-ref ctx "items"))))))
```

```rust
// Wasm — interface_schema 是 well-known 导出函数
// Host 调用 module.call("interface_schema", "") 拿 JSON
// on() handler 通过 Wasm export 函数注册
```

### 6.6 事件组合原语

事件到达即处理。但很多业务场景需要跨事件的时间或空间聚合。Realm 内置四种触发模式，覆盖不可分解的事件组合需求。

#### 不可分解性分析

能否拆成多个事件映射 + actor 组合，是判断"是否需要 Realm 原语"的标准：

| 组合模式 | 能否分解 | 理由 |
|---------|---------|------|
| **合并同类事件** | ✅ 能 | 多个事件映射到同一入口函数的多个参数，共享逻辑 |
| **多事件收齐（join）** | ❌ 不能 | 需等待多个事件全部到达后合并投递，单个参数无法触发 |
| **同类事件打包** | ❌ 不能 | 计数/窗口状态横跨多次事件到达，actor 无跨事件状态能力 |
| **去抖** | ❌ 不能 | 计时器必须在 Realm 层，actor 无时间感知 |

#### 四种触发模式

**`on` — 单事件触发（现有）**

```python
@on("order_created")
def handle(ctx, data):
    # data = 单个事件的载荷
    ...
```

**`on_join` — 多事件收齐**

等待多个**不同类型**的事件全部到达后触发。入口函数的 join 参数接收 map，按事件名索引。

```python
@on_join("payment_received", "inventory_reserved",
         timeout_ms=5000,
         schema={
             "payment_received": {"type": "object", "properties": {
                 "amount": {"type": "number"}
             }},
             "inventory_reserved": {"type": "object", "properties": {
                 "sku": {"type": "string"}, "qty": {"type": "integer"}
             }}
         })
def handle(ctx, data):
    # data = {
    #     "payment_received": {"amount": 100},
    #     "inventory_reserved": {"sku": "abc", "qty": 2}
    # }
    process_order(data["payment_received"], data["inventory_reserved"])
```

Realm 内部维护 `Collector`：追踪每个事件是否到达，收齐后合并投递到入口函数。超时未收齐时仍触发，数据中带 `__partial` 标记。

**Collector 是瞬态状态，不持久化到 Fjall。** 崩溃恢复后，源 actor 从 Fjall 恢复状态并重新 emit 事件，Collector 从零开始重新收集。持久化的锚点是 Actor 状态（写入 Fjall），不是事件聚合的中间缓冲区。

```rust
struct Collector {
    expected: HashSet<String>,              // 期望的事件集
    collected: HashMap<String, Value>,      // 已收集的事件
    window: Duration,                       // 超时窗口
    created_at: Instant,
}

impl Collector {
    fn is_complete(&self) -> bool {
        self.expected.iter().all(|e| self.collected.contains_key(e))
    }

    fn is_expired(&self) -> bool {
        self.created_at.elapsed() > self.window
    }

    fn drain(&mut self) -> Value {
        // 返回合并后的 map，未到达的事件标记 null
        let mut result = Map::new();
        for event in &self.expected {
            let value = self.collected.remove(event)
                .unwrap_or(Value::Null);
            result.insert(event.clone(), value);
        }
        Value::Map(result)
    }
}
```

**`on_batch` — 同类事件打包**

收集**同一类型**事件，按数量或时间窗口打包后触发。入口函数的 batch 参数接收数组。

```python
@on_batch("order_created", count=5, window_ms=10000,
          schema={"type": "array", "items": {
              "type": "object",
              "properties": {
                  "user_id": {"type": "string"},
                  "item": {"type": "object"}
              }
          }})
def handle_batch(ctx, events):
    # events = [
    #     {"user_id": "A", "item": {...}},
    #     {"user_id": "B", "item": {...}},
    #     ...
    # ]
    # 攒够 5 个或 10 秒窗口到期，取先到者触发
    bulk_insert(events)
```

**`on_debounce` — 去抖**

最后一次事件到达后，静默指定时间才触发。入口函数的 debounce 参数接收单个对象（最后一次事件的载荷）。

```python
@on_debounce("search_keystroke", delay_ms=300,
             schema={"type": "string"})
def handle_search(ctx, query):
    # 用户停止输入 300ms 后触发，query 是最后一次输入
    results = search(query)
    emit("search_results", results)
```

#### 触发模式与数据结构

| 模式 | 事件类型 | 入口函数参数 | Realm 状态 |
|------|---------|-------------|-----------|
| `on` | 单一 | 单个对象 | 无 |
| `on_join` | 多种不同 | map（key = 事件名） | Collector + timer |
| `on_batch` | 同一种 | 数组 | Counter + window timer |
| `on_debounce` | 单一 | 单个对象 | Timer + pending value |

#### 自动推导的 schema 格式

无论 Python 装饰器还是 Steel AST 提取，Realm 最终构建的路由表 schema 格式统一如下：

```python
def interface_schema():
    return {
        "receives": {
            "add_to_cart": {
                "mode": "on",
                "key": "user_id",
                "params": {"type": "object", "properties": {
                    "user_id": {"type": "string"},
                    "item": {"type": "object"}
                }}
            },
            "order_ready": {
                "mode": "join",
                "events": ["payment_received", "inventory_reserved"],
                "timeout_ms": 5000,
                "params": {
                    "type": "object",
                    "properties": {
                        "payment_received": {"type": "object"},
                        "inventory_reserved": {"type": "object"}
                    }
                }
            },
            "bulk_orders": {
                "mode": "batch",
                "source": "order_created",
                "count": 5,
                "window_ms": 10000,
                "params": {
                    "type": "array",
                    "items": {"type": "object", "properties": {
                        "user_id": {"type": "string"},
                        "item": {"type": "object"}
                    }}
                }
            },
            "search_input": {
                "mode": "debounce",
                "source": "search_keystroke",
                "delay_ms": 300,
                "params": {"type": "string"}
            }
        },
        "emits": ["cart_updated"]
    }
```

`interface_schema()` 由 Realm 自动推导，开发者不需要手写独立的 schema 声明。两种语言的机制不同，但目标一致：**schema 跟着事件声明走，单点维护，不可能不同步**。

**Python**：装饰器收集。`set("python", "actor.py")` 时 Realm import 模块 → 扫描所有 `@on` / `@on_join` / `@on_batch` / `@on_debounce` 装饰器 → 自动构建 `interface_schema()` → 注册路由表。

**Steel**：AST 提取。S-表达式天然可遍历，Realm 解析 `(on ...)` 调用的 AST，从内嵌的 `(schema ...)` 表达式中提取事件名和 schema。`set("steel", "actor.scm")` 时 Realm 读取源码 → sexp parser 解析 → 扫描所有 `(on ...)` 顶层调用 → 提取事件名 + schema → 构建路由表。不需要开发者手写独立的 `(define (interface-schema) ...)`。

```scheme
;; Steel — schema 内嵌在 on 调用中
(on "add_to_cart"
  (schema
    (key "user_id")
    (params (hash 'type "object"
                  'properties (hash 'user_id (hash 'type "string")
                                      'item (hash 'type "object"))
                  'required '("user_id" "item"))))
  (lambda (ctx data)
    (ctx-update! ctx "items"
      (lambda (items) (append items (list (hash-ref data "item")))))
    (emit "cart_updated"
      (list (cons "user_id" (hash-ref data "user_id"))
            (cons "items" (ctx-ref ctx "items"))))))

(on_join ("payment_received" "inventory_reserved")
  (schema
    (timeout_ms 5000)
    (params (hash 'type "object"
                  'properties (hash 'payment_received (hash 'type "object")
                                      'inventory_reserved (hash 'type "object")))))
  (lambda (ctx data)
    (let ((payment (hash-ref data "payment_received"))
          (inventory (hash-ref data "inventory_reserved")))
      (process-order payment inventory))))

(on_batch "order_created"
  (schema
    (count 5)
    (window_ms 10000)
    (params (hash 'type "array"
                  'items (hash 'type "object"
                               'properties (hash 'user_id (hash 'type "string")
                                                   'item (hash 'type "object"))))))
  (lambda (ctx events)
    (bulk-insert events)))
```

#### 路由表结构扩展

```rust
enum RouteMode {
    On,                                              // 单事件，现有逻辑
    Join { events: Vec<String>, timeout_ms: u64 },   // 多事件收齐
    Batch { source: String, count: usize, window_ms: u64 },  // 同类打包
    Debounce { source: String, delay_ms: u64 },       // 去抖
}

struct Route {
    actor_type: String,
    mode: RouteMode,
    partition_key_field: Option<String>,  // on 有，join/batch/debounce 可选
    handler: HandlerRef,
}
```

路由表从"事件名 → Route"变为"事件名 → Vec<Route>"——同一事件可以被多个不同模式的 Route 匹配：

```
"order_created" → [
    Route { mode: On, handler: CartActor },           // 精确单事件
    Route { mode: Batch, handler: BulkWriter },        // 5 个打包
]
```

Realm 分发时，对每个匹配的 Route 按 mode 分别处理：On 直接投递，Join/Batch/Debounce 交给 Collector。

### 6.7 投递语义

| 维度 | 选择 | 说明 |
|------|------|------|
| **可靠性** | at-least-once | 事件不丢，可能重复。消费端需幂等 |
| **命名空间** | namespace（默认 "default"） | 场域按 namespace 隔离，跨 namespace 的事件不投递 |
| **顺序保证** | 因果一致性 | 如果 e1 因果先于 e2（e1 的处理导致 e2 的发射），则任何订阅者收到 e1 必在 e2 之前。无因果关系的并发事件可乱序 |

因果一致是甜区：保证逻辑正确性（因先于果），不需要全序的共识开销。进程内事件天然因果有序（同一线程内的 emit 序列）；跨节点时 Openraft 日志本身是全序的（跨节点事件实际拿到比因果更强的保证）；同节点并发 Actor 的事件用向量时钟标记 happened-before 关系。

### 6.8 传统 Actor 模型可借鉴的设计

| 机制 | 来源 | Aura 对应 |
|------|------|-----------|
| **Supervision** | Erlang/OTP | Actor 崩溃时从 Fjall 恢复状态、重新加载脚本和入口函数。策略：one-for-one（独立重启）/ one-for-all（关联 Actor 一起重启，防止状态不一致） |
| **Location Transparency** | Akka | emit/on 不暴露目标在本地还是远程。Openraft 网络在背后透明投递 |
| **Become/Unbecome** | Akka | `set(lang, script)` 是更激进的版本——运行时切换行为函数和语言。可实现状态机：收到事件后 `set("python", "active_handler.py")` 切换入口函数 |
| **Stash** | Akka | Actor 刚唤醒、Fjall 状态还在恢复时，先 stash 事件，恢复完成后回放 |
| **Dead Letters** | Akka/Erlang | emit 的事件如果没有匹配的接收 Actor，或目标 Actor 崩溃且无 supervisor 重启，进入 dead event log。用于调试和事件审计 |
| **Backpressure** | Reactive Streams | Actor 入口函数的 mailbox（Tokio bounded MPSC）满了时，emit 方收到压力信号（阻塞/降级/丢弃） |
| **Passivation / Virtual Actor** | Akka / Orleans | 空闲 Actor 从内存驱逐（Scale-to-Zero），按需激活。Aura 的实例化机制基于此（详见 [§6.11](#611-actor-实例化与分片)） |

### 6.9 与现有 Actor 框架的对比

| 框架 | 类似点 | 关键差异 |
|------|--------|---------|
| **Akka EventStream** | 进程内事件总线，pub/sub | JVM/GC；EventStream 是 Actor 通信的补充，主通信仍 ActorRef 直发；无嵌入式存储，无 Scale-to-Zero |
| **Erlang gen_event** | 事件管理器，多订阅者 | BEAM 限定；无嵌入式多语言；跨节点靠分布式 Erlang，无强一致共识 |
| **Orleans Virtual Actor** | Actor 按需激活，"一直存在" | .NET 运行时；Grain 间通信是直接方法调用，不是事件总线；无嵌入式 KV |
| **Proto.Actor** | Go 实现，有 EventStream | Go runtime；事件总线是辅助；无嵌入式存储 |
| **Actix（Rust）** | Rust Actor 框架 | 无事件总线（Actor 间直发）；无分布式；无嵌入式脚本/沙箱；无共识 |

没有现有框架同时做到：事件总线作为主通信原语 + 嵌入式多语言 + 嵌入式 KV（Scale-to-Zero）+ Raft 共识跨节点 + 无外部消息队列。

### 6.10 Aura 的根本区别

**1. 事件总线是主通信原语，不是辅助**

传统 Actor 框架的主通信是 ActorRef 直发（tell/ask）。EventStream 是补充机制。Aura 把事件总线提到中心位置——Actor 之间只通过 emit/on 在场域中交互，完全解耦：发射者不关心谁处理，处理者不关心谁发射。

**2. 替代消息队列**

传统架构中服务间通信靠 Kafka/NATS。传统 Actor 框架跨节点靠框架自带 RPC（Akka Remote、Erlang dist）。Aura 的场域事件跨节点走 Openraft 共识日志——与状态复制共用同一通道。不需要外部消息队列，不需要独立 RPC 层。Fluxora 去掉 Kafka/NATS，由 Aura 场域替代。

**3. 嵌入式多语言 + 嵌入式存储 = 零外部依赖**

其他框架要么绑定一种运行时（BEAM、JVM、.NET），要么绑定一种语言（Actix/Rust）。Aura 的 Actor 实现可 Steel/Python/Wasm 任意切换，且 `set()` 运行时热替换。存储是进程内 Fjall。整个场域是一个单体二进制。

**4. 事件驱动 + 共识 = 反应式架构的进程内实现**

反应式架构主张"数据变更主动推送"，传统实现需要 CDC + Flink + Kafka + WS 网关的完整链路。Aura 的场域把这条链路压缩到进程内：Actor 写入 Fjall → emit 事件 → 订阅者进程内收到 → 零网络跳数。

### 6.11 Actor 实例化与分片

`set()` 定义的是 Actor **类型**（脚本 + interface_schema）。运行时，Realm 根据事件的 partition key 激活对应的 Actor **实例**。

**问题**：不同用户同时 emit("add_to_cart")，如果一个 Actor 实例串行处理所有请求，用户 B 要等用户 A 处理完——瓶颈。正确做法是按 user_id 分区，每个用户一个实例，互不阻塞。

**机制**：

```
emit("add_to_cart", {"user_id": "A", "item": "X"})
emit("add_to_cart", {"user_id": "B", "item": "Y"})

Realm 从 interface_schema 查到 add_to_cart 的 key = "user_id"
  → 事件 1: partition key = "A" → CartActor 实例 #A
  → 事件 2: partition key = "B" → CartActor 实例 #B
  → 两个实例并行处理，互不阻塞

同一用户后续事件：
emit("remove_from_cart", {"user_id": "A", "item_id": "X"})
  → partition key = "A" → 同一个 CartActor 实例 #A（串行，保证状态一致）
```

- 同一 partition key 的事件始终路由到同一 Actor 实例，保证该实体的状态一致性
- 不同 partition key 的事件路由到不同实例，并行处理
- 跨节点时，partition key 经一致性哈希映射到集群中的某个节点

**实例生命周期**（Virtual Actor 模式，与 Orleans 一致）：

| 阶段 | 行为 |
|------|------|
| **激活** | 事件到达，Realm 按 partition key 查找实例 → 不存在则从 Fjall 恢复状态（或新建空状态）→ 加载脚本和入口函数 |
| **运行** | 处理事件，可 emit，状态立即持久化。事件进入实例的 bounded MPSC mailbox，单线程串行消费 |
| **空闲** | 超时无事件 → 状态落盘 Fjall → 内存驱逐（Scale-to-Zero） |
| **再激活** | 新事件到达 → 从 Fjall 恢复 → 继续 |

Actor "一直存在"（逻辑上），按需激活/驱逐（物理上）。开发者不显式创建实例——`set()` 定义类型，`emit()` 触发激活。

**领域映射**：DDD 的聚合根天然对应 Actor 类型，聚合根 ID 作为 partition key：

| Actor 类型 | 实例 key | receives | emits |
|-----------|---------|----------|-------|
| CartActor | user_id | add_to_cart, remove_from_cart | cart_updated |
| OrderActor | order_id | place_order, cancel_order | order_completed, order_cancelled |
| InventoryActor | product_id | reserve_stock, release_stock | stock_reserved, out_of_stock |
| PaymentActor | payment_id | charge, refund | payment_confirmed, payment_failed |

跨聚合的交互通过场域事件完成，聚合内部直接操作 ctx.state。

### 6.12 跨 Partition 查询

Actor 实例的状态是隔离的——CartActor #A 看不到 CartActor #B 的数据。企业场景需要跨 partition 聚合查询（如统计部门所有用户的购物车）时，不能直接 JOIN。

| 方案 | 原理 | 延迟 | 适用场景 |
|------|------|------|---------|
| **投影 Actor**（推荐） | 独立 Actor 订阅事件流，持续维护聚合视图 | 低（预计算） | 常用聚合，可预定义 |
| **Arrow HTAP** | Fjall KV blob 导出为列式格式，Polars 执行 ad-hoc 查询 | 中（列式扫描） | 任意维度临时查询 |
| **Scatter-gather**（不推荐） | emit 查询事件，各实例响应后汇总 | 高（等最慢的实例） | 实例数已知且少的场景 |

**投影 Actor**：一个独立的 Actor（如 DeptStatsActor），按 dept_id 分片，`on("cart_updated")` 持续把用户级数据聚合到部门级 ctx.state。查询时直接读该 Actor 的状态。这是场域模型的自然延伸——投影 Actor 就是一个普通的事件接收 Actor，不需要额外基础设施。原理与反应式架构的流计算预聚合一致：不查询时计算，而是持续监听事件流维护聚合状态。

**Arrow HTAP**：[Arrow 大一统 HTAP 引擎](arrow-unified-htap-engine.md) 解决了 ad-hoc 查询问题——Fjall 的 KV blob 可以通过 Arrow 列式化 + Polars 执行多维度扫描、过滤、聚合。适合报表、BI、后台管理等无法预先定义的查询场景。

**Scatter-gather**：emit 一个查询事件，所有相关 Actor 实例各自响应 partial 结果，由汇总 Actor 收集。问题：不知道有多少实例、不知道何时收齐、延迟取决于最慢的实例。仅在实例数已知且少的场景使用。

### 6.13 ctx.invoke() 统一调用原语

Actor 不直接通过 PyO3 调用外部系统（`httpx.get()` 等）——这绕过了 Host 的管控，不可审计、不可限流、不可观测。所有同步调用（HTTP、Actor 间）通过 `ctx.invoke()` 统一入口。

**两种通信，分离**：

- **`emit`/`on`**：场域事件，Actor 间的异步通信（fire-and-forget、pub/sub、partition-keyed）。不承担请求-响应语义。
- **`ctx.invoke()`**：同步调用，阻塞等待返回值。目标可以是外部 HTTP 服务，也可以是场域内的 Actor。

**统一调用注册表（invoke.toml）**：

所有可调用目标通过声明式配置注册，统一目录：

```toml
# invoke.toml

# 外部 HTTP 服务
[[call]]
name = "user_info"
target = "https://api.example.com/users/{user_id}"
method = "GET"
timeout = 5000

[[call]]
name = "send_email"
target = "https://api.mailgun.net/v3/send"
method = "POST"
timeout = 10000

# 场域内 Actor（interface_schema 中声明了 returns 的事件）
[[call]]
name = "charge_processor"
target = "actor:charge_processor"
partition_key = "user_id"  # data 中哪个字段是 partition key

[[call]]
name = "db_query"
target = "actor:db_bridge"
partition_key = "query_id"
```

注册表是声明式配置，Fluxora 读取后负责实际路由——HTTP 目标走 Fluxora HTTP 请求，Actor 目标走 Realm 路由。

**Actor 侧**：

```python
# Actor 入口函数
async def handle(ctx, add_to_cart=None):
    if add_to_cart:
        # 调用外部 HTTP 服务
        user = await ctx.invoke("user_info", {"user_id": add_to_cart["user_id"]})

        # 调用场域内 Actor（同步等待返回值）
        charge = await ctx.invoke("charge_processor", {"user_id": add_to_cart["user_id"], "amount": 100})

        ctx.state["items"].append(add_to_cart["item"])
        # 已落盘（WAL + memtable）
        emit("cart_updated", {"user_id": add_to_cart["user_id"], "items": ctx.state["items"]})
```

`ctx.invoke()` 是同步语义——`await` 期间当前 Actor 实例阻塞（串行语义符合预期），其他实例不受影响。HTTP 和 Actor 调用在调用者视角完全一致：`ctx.invoke(name, data)` 拿到结果。

**Realm Actor 调用的内部机制**：

`ctx.invoke("charge_processor", data)` 对 Actor 目标的执行路径：

1. 查 `invoke.toml` → `target = "actor:charge_processor"`，`partition_key = "user_id"`
2. 从 `data` 中提取 `partition_key` 字段值 → 路由到 `charge_processor` 实例
3. Realm emit 事件 + `__reply_to` 到目标实例
4. 等待入口函数的 `return` 值（reply_to 机制）
5. 返回给调用者

入口函数只需正常 `return`，不需要关心 `__reply_to` 细节。入口函数同时可以 `emit` 通知其他 Actor——`return` 给调用者，`emit` 给系统，各走各的路。

**Webhook Ingress**：外部 HTTP 请求进来时，Fluxora 查注册表或路由配置，转成 `emit()` 投递到场域。Ingress 方向走 emit/on，因为这是业务事件入口，不是 RPC。

### 6.14 ctx.invoke 的 Rust 宿主实现

`ctx.invoke()` 的核心是 oneshot channel + pending call 表 + Actor 实例状态机。call 根据 `invoke.toml` 中的 target 前缀分派：HTTP 目标交给 Fluxora，Actor 目标走 Realm 路由。

**实例状态机**：

```
┌─────────┐     event 到达      ┌─────────┐
│  Idle   │──────────────────►│  Busy   │
└─────────┘                    └────┬─────┘
     ▲                              │
     │ 入口函数完成                │ ctx.invoke()
     │                              ▼
     │                        ┌──────────────────┐
     │                        │ WaitingForResponse │
     │                        │ (不消费 mailbox)    │
     │                        └────┬─────┬───────┘
     │              响应到达        │     │ 超时
     │◄─────────────────────────────┘     │
     │◄───────────────────────────────────┘
```

`WaitingForResponse` 状态下不消费 mailbox 中的新事件——保证同一实例的串行语义。其他实例（不同 partition key）不受影响。

**Host 核心结构**：

```rust
pub struct RealmHost {
    // call_id → 响应投递通道
    pending_calls: Arc<Mutex<HashMap<Uuid, PendingCall>>>,
    // 统一调用注册表（invoke.toml）
    call_registry: HashMap<String, CallTarget>,
    router: EventRouter,
    instances: Arc<Mutex<HashMap<String, ActorInstance>>>,
}

struct PendingCall {
    actor_id: String,
    responder: Responder,
    deadline: Instant,
}

enum Responder {
    Async { tx: oneshot::Sender<Value> },   // Python async / Rust actor
    Callback { cb: CallbackHandle },         // Steel Lisp
}

enum CallTarget {
    Http { endpoint: String, method: String, timeout: u64, schema: Option<Schema> },
    Actor { actor_type: String, partition_key: String, timeout: u64 },
}

struct ActorInstance {
    partition_key: String,
    state: Value,                            // CBOR 状态树
    mailbox: mpsc::Receiver<Event>,
    status: InstanceStatus,
}

enum InstanceStatus {
    Idle,
    Busy,
    WaitingForResponse,
}
```

**ctx.invoke 发起侧**：

```rust
impl Context {
    pub async fn call(
        &mut self,
        service: &str,
        data: Value,
    ) -> Result<Value, CallError> {
        let call_id = Uuid::new_v4();
        let (tx, rx) = oneshot::channel();

        // 查统一调用注册表
        let target = self.host.call_registry.get(service)
            .ok_or(CallError::UnknownService)?;

        let timeout_ms = match target {
            CallTarget::Http { timeout, .. } => *timeout,
            CallTarget::Actor { timeout, .. } => *timeout,
        };
        let deadline = Instant::now() + Duration::from_millis(timeout_ms);

        // 注册 pending call
        self.host.pending_calls.lock().insert(
            call_id,
            PendingCall {
                actor_id: self.actor_id.clone(),
                responder: Responder::Async { tx },
                deadline,
            },
        );

        // 标记实例状态
        self.instance.status = InstanceStatus::WaitingForResponse;

        match target {
            CallTarget::Http { endpoint, method, .. } => {
                // HTTP 目标 → 交给 Fluxora 执行
                self.host.call_dispatcher.send(CallRequest::Http {
                    call_id,
                    endpoint: endpoint.clone(),
                    method: method.clone(),
                    data,
                    deadline,
                });
            }
            CallTarget::Actor { actor_type, partition_key, .. } => {
                // Actor 目标 → Realm 路由 + reply_to
                let pk_value = data.get(partition_key)
                    .ok_or(CallError::MissingPartitionKey)?;
                self.host.router.emit_to_actor(
                    actor_type,
                    pk_value.as_str().unwrap_or_default(),
                    data,
                    Some(call_id),  // reply_to channel
                );
            }
        }

        // 挂起，等 tx.send() 或超时
        match tokio::time::timeout(
            Duration::from_millis(timeout_ms), rx,
        ).await {
            Ok(Ok(response)) => {
                self.instance.status = InstanceStatus::Busy;
                Ok(response)
            }
            Ok(Err(_)) => {  // tx 被 drop
                self.instance.status = InstanceStatus::Busy;
                Err(CallError::Cancelled)
            }
            Err(_) => {  // 超时
                self.host.pending_calls.lock().remove(&call_id);
                self.instance.status = InstanceStatus::Busy;
                Err(CallError::Timeout)
            }
        }
    }
}
```

**响应投递**：

```rust
impl RealmHost {
    // HTTP 响应到达 或 Actor 入口函数 return 值到达
    async fn resolve_call(&self, call_id: Uuid, response: Value) {
        let mut pending = self.pending_calls.lock();
        if let Some(req) = pending.remove(&call_id) {
            match req.responder {
                Responder::Async { tx } => {
                    let _ = tx.send(response);  // 唤醒 await 的 Actor
                }
                Responder::Callback { cb } => {
                    // Steel Lisp：向 Actor mailbox 投递 ResumeEvent
                    if let Some(inst) = self.instances.get(&cb.actor_id) {
                        inst.mailbox.send(Event::Resume {
                            callback: cb,
                            data: response,
                        });
                    }
                }
            }
        }
    }
}
```

HTTP 响应和 Actor return 值走同一个 `resolve_call` 通道——call 的响应不污染事件命名空间。

**Python 桥接**：Python 的 `await ctx.invoke()` 通过 PyO3 桥接为 Rust future。`await` 时 Python coroutine 挂起并释放 GIL，Tokio runtime 调度其他 task。oneshot 解锁 → Rust future 完成 → Python coroutine 恢复。

**Steel Lisp 桥接**：Steel 没有 async/await，用 callback。`ctx-invoke` 调用后立即返回，入口函数暂停，Actor 进入 `WaitingForResponse`。响应到达时 Host 不直接调用 Steel VM（跨线程不安全），而是向 Actor 的 mailbox 投递 `ResumeEvent`，事件循环收到后恢复执行 callback：

```scheme
(ctx-invoke ctx "user_info" (hash 'user_id "123")
  (lambda (response)        ; 响应到达时调用
    (ctx-update! ctx "items" ...)
    ;; 已落盘（WAL + memtable）
    (emit "cart_updated" ...))
  (lambda ()                ; 超时时调用
    (emit "error" ...)))
```

**Wasm**：Wasm 通过 host 函数调用 `ctx.invoke()`——Host 在 Wasm 挂起时执行 async 操作（emit + 等待 reply_to），结果返回后恢复 Wasm 执行。对 Wasm 来说 `ctx.invoke()` 是一个普通的同步 host 函数调用，内部异步由 Host 封装。Wasmtime 的 host 函数调用天然支持阻塞，不需要"中间 Actor 桥接"。

**超时扫描**：Host 后台 task 每 100ms 扫描 `pending_calls`，清理过期条目，向对应 Actor 投递超时 `ResumeEvent`（Callback）或 send 超时值（Async）。

## 7. 工业适用性诊断与通用场景

这套由 Fjall（本地存储）+ Openraft（分布式强一致共识）+ 多模态嵌入式内核（Steel/PyO3/Wasm 用户自主驱动沙箱）铸造的纯 Rust 存算一体架构，在工业落地中具有极强的普适性。它不仅兼容网关、FaaS、任务平台和游戏场景，甚至能在这几个场景里引发架构颠覆。

### 7.1 四大核心场景的工程适用性诊断

#### API 网关场景：适合（实现"动态策略、零网络开销"）

传统的 API 网关（如 Kong 或 APISIX）在处理高级路由（如：动态限流、AB 测试分流、用户鉴权）时，通常必须频繁地去读取外部缓存，或者在 Nginx 内部塞满晦涩的 Lua 脚本。

**在本架构下**：每个 API 路由策略或每个租户都是集群里的一个分布式 Actor。网关收到请求后，不需要经过任何网络跳转，直接在进程内存里、通过 Fjall 快速捞出该路由的最新规则，并由用户动态指定的 Steel Lisp 沙箱一秒执行。

**关键点**：由于 Openraft 保证了配置的全网强一致性复制，你修改网关规则后，全球所有分布式边缘节点会秒级同步，且没有外部缓存单线程死锁或集群脑裂的风险。

#### FaaS 平台场景：互补（打通"有状态 Serverless"的死穴）

传统 FaaS（如 AWS Lambda 或 Vercel Functions）最致命的痛点是"无状态（Stateless）"。函数每次拉起都要重新去数据库连线、握手、查配置（冷启动长达数百毫秒），这在行业里催生了对类似 Cloudflare Durable Objects（有状态持久化对象）的强烈渴望。

**在本架构下**：你不需要像 Rivet 那样去捆绑重型臃肿的 V8 和 JS 生态。用户自己决定这一毫秒是用 Steel 还是 Python 来跑 FaaS。

**关键点**：它提供了Scale-to-Zero（闲时内存归零）与系统低开销启动后，Actor 唤醒为微秒级（嵌入式 VM 瞬时创建）。函数不活动时，状态在 Fjall 磁盘中以紧凑的 SSTables 形式冬眠，不侵占 1 字节的 RAM；请求命中时，Openraft 达成共识，Fjall 瞬间把 Vec 二进制 blob 拍进内存，多语言引擎就地复活执行。

#### 任务编排平台（如 Windmill/Prefect）：超越（移除调度队列膨胀）

像 Windmill 这样的任务平台，其底层为了维护任务调度队列、并发锁和重试状态，必须高度依赖一个吞吐量极大的 PostgreSQL 或外部队列。当遇到百万级微型任务并发时，外部数据库连接池会瞬间枯竭。

**在本架构下**：每一个长时流式任务（Workflows）本身就是一个独立的 Actor。任务的每一个 Step、每一步重试状态，都被作为 RaftCommand 直接原子化地写进本地物理 Fjall 引擎中。

**关键点**：它彻底消除了外部调度队列。任务状态就在计算引擎的贴身进程内存里，不需要跨进程 IPC，不需要外部锁。它比 Windmill 更轻、更快，且天然具备跨机房的断线防失忆防猝死（Fault-tolerance）能力。

#### 游戏服务器场景：它的老本行（游戏级高并发 Actor 的 Rust 回归）

Rivet Actors 为什么要用这一套设计？因为他们原本就是做多人联机游戏房间（Game Rooms）和玩家实时状态（Player Stateful）托管出身的。

**在本架构下**：你用纯 Rust 实现了比 Rivet 更纯粹、更干净的原生游戏运行时。

**关键点**：一个游戏房间就是一个 Actor，玩家的所有走位、血量、装备变动（State Mutate），直接高频（每秒 60 次）缓存在该 Actor 的内存中，通过 Openraft 实现多副本冗余。游戏结束时，Fjall 异步执行一次大块物理刷盘。由于彻底踢出了 JS/V8 的垃圾回收（GC）开销，主程序可以跑到上千帧的平滑度，单机挂载数万玩家房间系统也绝不卡顿。

### 7.2 三个全新硬核应用场景

除了上述四个经典领域，这套 Fjall + Openraft + 用户主导多语言沙箱的全栈架构，还能在以下三个涉及通用、前沿开发的环境中展现出适用性：

#### 分布式工业物联网与边缘计算（Edge AI & IoT Gateways）

在风力发电厂、车联网、无人机编排或自动化工厂机房中，硬件设备往往处于"弱网、低功耗、本地磁盘寸土寸金"的恶劣物理环境里。你不可能在边缘机房里塞进庞大的 K8s 集群或者 PostgreSQL 数据库。

**怎么玩**：把这个单文件二进制程序直接扔在边缘网关（如树莓派或工业单板机）上。由于 Fjall 的 LSM 结构极度抗断电损耗，Openraft 可以在多台网关之间自愈组网。采集到高频传感器数据时，PyO3 Python 直接在本地内存中运行 Numpy 异常检测；发现危机时立刻切换到 Steel Lisp 运行确定性的关断策略脚本。微秒级响应，完全不需要连向云端。

#### 现代 Web 3.0 / 联盟链与可信去中心化账本（Consensus Ledgers）

传统的区块链底层（如以太坊节点）在执行智能合约时，架构笨重。Openraft 本身就是分布式共识的代名词，通过将用户的转账逻辑、合约条件写成 Steel Lisp 或 Wasm 字节码，全球 5 台或 11 台受信服务器运行这套二进制程序，用户提交的脚本在 Openraft 达成全网多数派共识后，在本地 Fjall 物理安全落盘。用不到 2000 行的 Rust 核心，实现了一个响应速度突破上万 TPS、且具备跨国多机房灾备能力的超高性能专属去中心化状态账本。

#### 企业私有化"无头"AI 程序员 Agent 矩阵（Headless Coding Matrix）

当公司需要部署 1000 个无处不在、24/7 在后台自动审查 Git 代码、跑自动测试并修改 Bug 的"AI 程序员"时，每个 AI 程序员就是一个常驻或冬眠的分布式 Actor。AI 需要去外网爬取 API 文档时，调用 Python（PyO3）异步网络库；当它要结合本地文件跑 PyTorch 权重或者语义分析时，调用 PyO3 Python；当它要执行危险的本地编译时，直接锁死在 Wasm 沙箱里防止它误删真实硬盘。所有的思考逻辑、进度、避坑备忘录，完全不需要依赖外部队列，被 Openraft 多数派共识后持久化在本地 Fjall 数据库中。

### 7.3 架构师的核心优势

这套架构之所以能在这么多南辕北辙的通用、极限场景里同时展现出适用性，不是因为它的功能多，恰恰相反，是因为它把不该存在的中介全部移除了（The Best Part is No Part）。

它在物理层面上消灭了网线两端的序列化和反序列化，把多核 CPU、NVMe 物理磁盘和多语言虚拟机的内存空间直接贴合在同一个机位上。在通用开发环境里，这是一种简洁的设计取向。

---

## 8. 单机起手与分布式演进

### 10.1 核心洞察：起手式 ≠ 终极数据宿命

在主程序里用一个轻量的内存状态管理组件起手，从来都不耽误、也不妨碍系统未来去连接和使用任何垂直领域的专业外部数据库。

正如在常规开发中，我们在主进程里手写一个 `Mutex<HashMap>` 或者拉起一个 Tokio 状态通道作为执行期的缓存和业务状态机，这不妨碍我们在需要持久化的时候，用一条单次事务连接（Single-Trip）把数据顺手写入 PostgreSQL、或者归档进 S3 的 LanceDB 里。

如果顺着这层"起手式不等于终极数据宿命"的最高务实哲学，来重新审视"任何项目直接以 Fjall + Openraft 起手"的合理性，核心的技术分水岭就不再是"能不能用别的数据引擎"，而是你从第一天开始，往你的二进制文件里注入的**"架构心智负荷与锁定代价"**有多重。

### 10.2 三种起手模式对比表

| 维度 | Mutex<HashMap> | Openraft 起手 | **Tokio Actor + Fjall** ✨ |
|------|---------------|--------------|---------------------------|
| **编码摩擦** | 极低 | 极高（状态机抽象） | **低**（简单 KV 接口） |
| **断电保护** | ❌ 无 | ✅ WAL + RaftLog | **✅ WAL** |
| **Scale-to-Zero** | ❌ 无（10 万 Actor 撑爆内存） | ✅ LSM-Tree | **✅ LSM-Tree** |
| **持久化** | ❌ 无（重启后状态丢失） | ✅ RaftLog | **✅ Fjall 磁盘** |
| **未来扩展性** | ✅ 无限 | ❌ 锁死 CP 强一致 | **✅ 无限** |
| **心智负荷** | 极低 | 极高 | **低** |
| **推荐场景** | 原型验证 | 明确需要分布式 | **通用起手式** ✨ |

**模式一：Mutex<HashMap> 起手（零依赖体验）**——项目刚敲下第一行代码时，状态就是 Rust 原生类型，不需要写任何序列化宏。致命缺陷：无断电保护、无 Scale-to-Zero、无持久化。

**模式二：Openraft 起手（分布式铁笼枷锁）**——状态不能再任性地通过指针直接修改，必须强制把每一个业务动作定义为可全网广播的 `RaftCommand`。如果后续发现项目需要 AP 最终一致性系统（如 CRDT），整个业务控制路由早已和 RaftLog 紧密绑定。不推荐项目早期、业务边界未定型时使用。

**模式三：Tokio Actor + 单机 Fjall 起手（工程学的最高折中）✨**——用 Fjall 替代 HashMap 几乎没有增加编码摩擦，却带来了：✅ 本地 bare-metal 级别的断电崩溃物理保护（WAL）、✅ 闲时内存自动归零（Scale-to-Zero）、✅ 读写速度快到物理硬件的极限、✅ 布隆过滤器微秒级定位。如果项目做大了需要多机灾备，由于已经是 Actor + Fjall 架构，随时可以轻松地把 Openraft 的分布式共识日志作为一层"轻量保护膜"套在 Fjall 的外面。**起手式不锁定终极宿命。**

### 10.3 起手式代码示例

#### Mutex<HashMap> 起手（零摩擦）

```rust
use std::collections::HashMap;
use std::sync::Mutex;
use tokio::sync::mpsc;

struct ActorState {
    data: Mutex<HashMap<String, Vec<u8>>>,
}

impl ActorState {
    fn new() -> Self {
        Self {
            data: Mutex::new(HashMap::new()),
        }
    }

    fn upsert(&self, key: &str, value: Vec<u8>) {
        self.data.lock().unwrap().insert(key.to_string(), value);
    }

    fn get(&self, key: &str) -> Option<Vec<u8>> {
        self.data.lock().unwrap().get(key).cloned()
    }
}
```

#### Tokio Actor + Fjall 起手（最佳折中）✨

```rust
use fjall::{Config, Keyspace, PartitionCreateOptions};
use std::sync::Arc;

struct ActorState {
    keyspace: Keyspace,
    partition: Arc<fjall::PartitionHandle>,
}

impl ActorState {
    fn new(path: &str) -> anyhow::Result<Self> {
        let keyspace = Config::new(path).open()?;
        let partition = keyspace.open_partition(
            "actor_state",
            PartitionCreateOptions::default(),
        )?;

        Ok(Self {
            keyspace,
            partition: Arc::new(partition),
        })
    }

    fn upsert(&self, key: &str, value: Vec<u8>) -> anyhow::Result<()> {
        self.partition.insert(key.as_bytes(), value)?;
        self.keyspace.persist(fjall::PersistMode::SyncAll)?;
        Ok(())
    }

    fn get(&self, key: &str) -> anyhow::Result<Option<Vec<u8>>> {
        Ok(self.partition.get(key.as_bytes())?.map(|v| v.to_vec()))
    }
}
```

#### 未来演进：套上 Openraft 保护膜

```rust
// 当需要多机灾备时，只需在 Fjall 外面套一层 Openraft
use openraft::Raft;

struct DistributedActorState {
    raft: Raft<FluxarrowTypeConfig>,
    fjall: Arc<ActorState>,  // 复用上面的单机 Fjall 实现
}

// 业务代码几乎不需要修改
impl DistributedActorState {
    async fn upsert(&self, key: &str, value: Vec<u8>) -> anyhow::Result<()> {
        // 通过 Raft 共识后，写入本地 Fjall
        let cmd = RaftCommand::UpsertState {
            agent_id: key.to_string(),
            value,
        };
        self.raft.client_write(cmd).await?;
        Ok(())
    }
}
```

### 10.4 技术现实主义的胜利

状态管理框架只是工具，它不是禁锢业务宿命的牢笼。在项目的第一天，直接引入 Openraft 这样厚重的分布式网络共识逻辑，往往会因为过度设计（Over-engineering）而把早期的业务演进速度活生生拖垮。

但如果采用**"最轻量的 Tokio Actor 消息管道 + 单机内嵌 Fjall"**作为新项目的通用起手核心——这既提供了启动速度、内存安全和冬眠效率，又把面向所有外部数据库（Postgres, S3, LanceDB）进行后期大一统演进的大门敞开着。单机运行完全没有问题，它是这套架构走向工业化最稳健、最清醒、低开销的第一步。

---

## 9. 序列化协议抉择

→ 详见 [序列化协议抉择：IDL vs Code-First](serialization-protocol-decision.md)

### Aura 的分层序列化策略（2026-06-20 更新）

| 层级 | 序列化格式 | 理由 |
|------|-----------|------|
| **Actor 状态持久化（Fjall）** | CBOR | 动态结构化数据，自描述，跨语言（Steel/Python/Rust）编解码 |
| **Actor 输入/输出载荷** | `ciborium::Value`（进程内）/ CBOR bytes（跨边界） | 进程内为内存值树，零序列化；持久化/复制/外部投递时序列化为 CBOR bytes |
| **Raft 元数据（LogId 等）** | Postcard | Openraft 内部用，与 Raft 控制流统一格式 |
| **Raft 控制流与 RPC** | Postcard | 纯 Rust 声明式、Postcard-Schema 宏支持 Schema 演进 |
| **分析查询路径** | Arrow IPC | 列式对齐，Fjall 读出后 Polars 零拷贝转铸 DataFrame |
| **云端长期记忆 Lakehouse** | Lance | 内嵌向量与倒排索引，原生支持远程 S3 流式检索 |
| **海量冷历史冬眠** | Parquet | 高压缩率冷存储 |
| **UI ↔ Gateway** | CBOR | 跨语言（WASM），自描述 |

**核心原则**：技术不应该是束缚开发者手脚的繁文缛节。看清物理硬件的边界，去掉无谓的 IDL 嵌套，才能让 Aura 引擎在多核 CPU 和 NVMe 之间保持高效运行。

---

## 10. 开发体验（DX）：从 Rivet Actors 的启发到 Aura 的改进

> 灵感来源于 Rivet Actors 的 Actor 模型开发体验，但 Aura 在类型安全、冷启动性能、状态持久化和多语言支持上做了根本性改进。

### 10.1 Rivet Actors 的 DX 基线与 Aura 的改进点

Rivet Actors 提供了优秀的 Actor 开发体验：TypeScript SDK、自动 HTTP 端点生成、内置状态管理、WebSocket 连接管理、本地开发服务器。但它的核心限制在于：

| 维度 | Rivet Actors | Aura 的改进 |
|------|-------------|------------|
| **语言** | TypeScript/JavaScript（V8 隔离） | Rust 核心 + Steel Lisp/Python/Wasm 嵌入（进程内，无 IPC） |
| **系统启动** | 数百毫秒（Node.js 进程 + V8 初始化） | 毫秒级（Tokio 运行时 + Fjall 打开） |
| **Actor 唤醒** | 几毫秒（V8 虚拟机激活） | 微秒级（Steel 字节码 VM 瞬时创建；Python PyO3 ~1ms） |
| **状态存储** | SQLite（同机共生，但单机瓶颈） | Fjall LSM-Tree（嵌入式，支持 Openraft 分布式复制） |
| **类型安全** | TypeScript（运行时类型，编译期弱） | Rust 编译期强类型 + Steel Lisp 的 S-表达式零二义性 |
| **多语言** | 仅 JS/TS | Rust/Steel/Python/Wasm 四语言进程内混合 |
| **状态持久化** | SQLite 文件 | Fjall KV + CBOR 序列化 |

### 10.2 Actor 定义与生命周期

Actor 通过 `set(lang, script)` 提交实现（详见 [§4.3](#43-actor-定义接口)）。脚本内导出 `interface_schema()` 声明事件契约，定义单一入口函数（事件名映射为参数）。

生命周期：

1. **唤醒（on_wake）**：有事件到达时，Host 从 Fjall 恢复 Actor 状态到内存
2. **注册**：调用 `interface_schema()` 构建路由表，绑定入口函数
3. **处理**：进入事件循环，收到事件 → 路由到入口函数对应参数 → 入口函数内可 `emit()`，状态立即持久化
4. **休眠（on_sleep）**：空闲后状态落盘到 Fjall，内存归零（Scale-to-Zero）

定时任务通过声明式注解，不需要外部 CronJob：

```python
@cron("0 2 * * *")
def daily_sync(ctx):
    sync_to_remote(ctx.state["pending_orders"])
```

**与 Rivet 的关键差异**：
- Rivet 的 Actor 状态是 JS 对象序列化到 SQLite，Aura 的状态是 CBOR 映射到 Fjall
- Rivet 的 cron 是外部调度器触发，Aura 的 cron 是 Actor 内声明式注解
- Rivet 需要 `actor.setState()` 手动调用，Aura 的 set/append 立即写 WAL 落盘
- Rivet 的 Actor 逻辑只能用 JS，Aura 的 `set()` 支持运行时切换语言

### 10.3 事件发现：interface_schema() 约定

脚本导出 `interface_schema()`，返回 receives/emits 列表。Host 启动时调用一次，构建事件路由表。详见 [§6.4](#64-interface_schema)。

**热重载**：脚本修改 → 重新加载 → 重新 `interface_schema()` → 路由表更新。无需重编译 Rust host。

**Rust actor 的情况**：Rust 写的 actor 同样遵循 `interface_schema()` 约定，但宏在编译期自动生成，不需要手写：

```rust
// #[aura::schema] 宏在编译期扫描，自动生成 interface_schema() 导出
#[aura::schema]
#[derive(Serialize)]
struct Order { id: String, item: String }

// 事件处理通过宏注册
#[aura::on("order_created")]
async fn handle_order_created(ctx: Context, data: Order) -> Result<()> { ... }
```

**唯一约束**：脚本必须导出 `interface_schema()` 且返回符合结构的 JSON。违反则拒绝加载。脚本即文档——看到 `interface_schema()` 就知道这个 Actor 接收什么事件、发射什么事件。

### 10.4 本地开发体验

Rivet 有 `rivet worker dev` 本地开发服务器。Aura 的本地开发更轻量：

```bash
# 单文件启动——不需要 Docker、不需要 etcd、不需要数据库
$ aura dev order_actor.py

# 输出：
# [Aura] Actor order_actor 已启动
# [Aura] 场域: default
# [Aura] 状态: Fjall 本地模式 (./data/actors/)
# [Aura] 热重载: 监听文件变化，修改即生效
```

**对比 Rivet**：Rivet 本地开发需要 Node.js 运行时 + V8 隔离层。Aura 是单二进制，下载即用，无运行时依赖。

### 10.5 测试体验

```rust
#[cfg(test)]
mod tests {
    use aura::test::RealmHarness;

    #[aura::test]
    async fn test_order_flow() {
        // 测试框架自动创建临时 Fjall 存储和场域，测试结束后自动清理
        let mut realm = RealmHarness::new("default").await;
        realm.set("python", "order_actor.py").await;

        // emit 事件，验证响应
        realm.emit("order_created", {"item": "widget"}).await;
        let events = realm.collect("order_completed").await;
        assert_eq!(events.len(), 1);

        // 状态持久化验证——重启后状态仍在
        realm.restart().await;
        realm.emit("order_created", {"item": "gadget"}).await;
        let events = realm.collect("order_completed").await;
        assert_eq!(events.len(), 2);
    }
}
```

**与 Rivet 的差异**：Rivet 测试需要启动本地开发服务器或 mock SQLite。Aura 的测试是纯进程内——Actor 在测试进程中直接运行，Fjall 用临时目录，无网络、无外部依赖。

### 10.6 部署体验

```bash
# 构建——单二进制，无运行时依赖
$ aura build --release
# 输出: target/release/order_service (12MB)

# 部署到目标机——不需要 Docker、不需要 K8S
$ scp target/release/order_service user@server:/opt/aura/
$ ssh user@server "aura serve order_service --port 8080"

# 多机部署——加一行 Openraft 配置即可分布式
$ aura serve order_service --raft-nodes "node1:9004,node2:9004,node3:9004"
```

**与 K8S 部署的对比**：

| 步骤 | K8S 部署 | Aura 部署 |
|------|---------|----------|
| 构建 | Dockerfile → docker build → push registry | `aura build --release` |
| 配置 | Deployment YAML + Service YAML + Ingress YAML | `aura serve --port 8080` |
| 状态存储 | PVC + StorageClass + PV | Fjall 内嵌（自动） |
| 多副本 | StatefulSet + etcd + headless Service | `--raft-nodes` 一行配置 |
| 扩缩容 | HPA + Metrics Server + CPU/内存阈值 | Actor 自动 Scale-to-Zero |
| 证书 | cert-manager + ClusterIssuer + Certificate CRD | 内置 Let's Encrypt（可选） |

### 10.7 Rivet 没有但 Aura 有的

**1. 编译期状态安全**
Rivet 的 `actor.setState()` 是运行时 API——传错类型、忘记调用、状态结构变更后旧数据反序列化失败，都要到运行时才暴露。Aura 的状态是 Rust 结构体，编译器在编译期就拦截所有类型错误。

**2. 多语言脚本 + 沙箱运行时**
Rivet 的 Actor 逻辑只能用 JS。Aura 的每个 Actor 可自由选择脚本语言——Python（AI/数据）、Steel（元编程）——不同 Actor 用不同语言，通过 `interface_schema()` 统一发现事件契约。此外，Wasm 沙箱运行时用于托管第三方不信任代码，硬件级隔离，与脚本语言互补。

**3. 确定性休眠/唤醒**
Rivet 的 Actor 休眠依赖 V8 堆快照，唤醒时需要反序列化整个堆。Aura 的 Actor 休眠是将 Rust 结构体通过 CBOR 序列化写入 Fjall，唤醒时从磁盘直接反序列化到内存——不依赖 V8 堆格式，不受 GC 暂停影响。

**4. 分布式状态复制**
Rivet 的状态是单机 SQLite，多副本需要外部同步。Aura 通过 Openraft 实现多节点强一致性复制，Actor 状态在集群内自动同步，无需外部组件。

### 10.8 DX 设计原则总结

1. **单二进制，零依赖**：开发者不需要安装 Docker/K8S/Node.js/Python，下载 Aura 二进制即可开发、测试、部署
2. **状态即代码**：Actor 状态是 CBOR Value 树（`ctx.state`），通过 `ctx.state` 操作立即写 WAL 落盘。Rust Actor 享有编译期类型安全，脚本 Actor 享有 JSON Schema 校验
3. **声明式生命周期**：`on_wake`/`on_sleep`/`cron` 注解声明 Actor 行为，不需要外部调度器
4. **本地即生产**：本地开发用 Fjall 临时目录，生产用 Fjall 持久目录，行为 100% 一致——没有"本地能跑线上炸了"的问题
5. **渐进式复杂度**：单机 → 分布式只需加一行 `--raft-nodes`，不需要重写代码或引入新组件

### 10.9 auractl：CLI 管理工具

Aura 提供两个一级子命令的 CLI 工具 `auractl`：

**`auractl realm`** — 场域管理操作：

```bash
# 提交/更新 Actor 定义
auractl realm set cart_actor python cart_actor.py

# 查看所有 Actor 定义
auractl realm list

# 查看某个 Actor 的 interface_schema
auractl realm schema cart_actor

# 向场域发射事件（调试用）
auractl realm emit add_to_cart '{"user_id": "A", "item": {"id": "X"}}'

# 查看活跃 Actor 实例
auractl realm instances

# 查看死信事件
auractl realm deadletters

# 查看待处理 call
auractl realm pending-calls
```

**`auractl fjall`** — Fjall 数据维护：

```bash
# 查看所有 partition
auractl fjall partitions

# 查看某个 Actor 的状态
auractl fjall get actor_state cart_actor:user_A

# 扫描某个 partition 的 key
auractl fjall scan actor_defs --prefix cart

# 压缩 LSM-Tree（手动触发 compaction）
auractl fjall compact

# 查看存储统计
auractl fjall stats

# 导出 Actor 定义（备份）
auractl fjall export actor_defs --output cart_actors.cbor

# 回滚 Actor 定义到上一版本
auractl fjall rollback cart_actor --version 2
```

`auractl` 直接读写本地 Fjall 实例（单机模式）或通过 Host API 远程操作（分布式模式）。两个子命令的职责分离：`realm` 管理运行时状态和事件，`fjall` 管理底层存储和版本化数据。

## 11. 常规 Web 服务适用性

Aura 的核心场景是高并发有状态流（游戏、IM、秒杀、AI 智能体）。对于常规 Web 服务（CRUD、电商、企业管理系统），需要判断哪些子场景适合 Aura、哪些应该回退到传统关系型数据库。

### 13.1 数据映射

常规 Web 的业务数据按特性映射到 Aura 的双轨记忆：

| 业务数据类型 | 传统做法 | Aura 映射 |
|---|---|---|
| 高频状态事务（购物车、库存、Session） | Redis 或 PostgreSQL UPDATE | 工作记忆（Fjall + Openraft）：Actor 进程内 LSM-Tree，Raft 多机一致，0 网络 RTT |
| 流水事件（订单历史、审计日志） | 日志表或 Kafka | Openraft WAL：共识达成即安全，确定性时序 |
| 海量历史只读分析（报表、全文检索） | 读写分离、Elasticsearch | 长期记忆（LanceDB + S3）：窗口结单转列式 Lance，S3 托管 |

### 13.2 适合的子场景

- **高并发状态变动**（秒杀、抢单）：Actor 进程内 MPSC 排队，单机承载传统架构需要多机加外部缓存才能扛住的并发量
- **无盘化弹性部署**：工作记忆 Scale-to-Zero，长期记忆托管 S3，节点可随时拉起或销毁

### 13.3 不适合的子场景

- **Ad-hoc 任意维度 SQL 查询**：热数据是 Fjall 的 KV blob，冷数据是 S3 列式切片，无法执行多表 JOIN + GROUP BY。注：[Arrow 大一统 HTAP 引擎](arrow-unified-htap-engine.md) 已部分解决此问题
- **后台管理 / ERP / CRM**：80% 工作是表格报表关联查询
- **团队门槛**：Raft 共识、Bincode 序列化、嵌入式脚本沙箱对初级开发者不友好

### 13.4 混合架构

常规 Web 服务同时包含"高并发核心模块"和"常规 CRUD"时，采用混合：
- CRUD 剥离给 PostgreSQL
- 高并发状态引擎用 Aura

不要让技术情怀变成业务迭代的绊脚石。

---

## 参考资料

- [1] 针对 NoSQL 盲目狂热与多库联合带来的数据一致性雪崩研究 (2025/2026 技术综述)
- [2] 现代高速多核 CPU 与 RDBMS 共享内存缓冲的吞吐量基准测试 (2026/03)
- [3] InfoQ 2026/05: Google 开源 AX 与 Agent Substrate 构建 K8s 智能体专属控制平面

---

## 交叉引用

本文档是架构设计的终极落地方案，与以下详细分析形成完整的决策闭环：

- **[Arrow 大一统 HTAP 引擎](arrow-unified-htap-engine.md)**：Fjall + Arrow + Polars 全链路存算一体，含 7 种数据格式底层字节排布对撞。
- **[Aura + Fluxora DevOps](aura-fluxora-devops.md)**：脚本即代码（Script-as-Code）的工程实践，含 GitOps 工作流、CI/CD 配置、脚本测试与调试。
- **[Redis 批判](redis-critique.md)**：详细论证不用 Redis 的原因，§7.6 是摘要版。
- **[MySQL 批判](mysql-critique.md)**：MySQL 的 SQL 反模式与分片幻觉，本架构选择 PostgreSQL 的 JSONB 聚合。
- **[SurrealDB](unified-data-layer.md)**：现代多模型数据库的替代方案，本架构可选择 SurrealDB 替代 PostgreSQL。
- **[嵌入式脚本语言选型](embedded-script-languages.md)**：Rune/Steel/Koto 的技术对比，本架构坚定选择 Steel Lisp。
- **[编辑器选型](editor-selection-2026.md)**：Helix vs Neovim 的深度分析，本架构可在 Neovim 中使用 Fennel/Steel 元编程。
- **[Nginx 批判](nginx-critique.md)**：Nginx 的遗留设计，本架构可选择 Envoy/Pingora 作为现代网关。
- **[反应式架构](reactive-architecture.md)**：Aura §8 场域模型是反应式架构的进程内实现——Actor 写入 Fjall → emit 事件 → 订阅者进程内收到，零网络跳数，不依赖外部流处理引擎。
- **[Wasm 统一运行时](wasm-unified-runtime.md)**：Rust → Wasm 现在已可用（不需要 wasm-gc），覆盖异步场景。wasm-gc 后 Kotlin/C# 涌入，语言选择进一步丰富。Steel 和 PyO3 保留。

