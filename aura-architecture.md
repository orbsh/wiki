# Aura：存算一体的现代分布式摊位引擎

**状态**：架构设计完成
**日期**：2026-06-20
**核心哲学**：框架融合、计算下沉、零拷贝、确定性边界

> Aura 是一个存算一体的现代分布式摊位引擎。在 Aura 里，没有 SQL，没有外部缓存锁，没有多库同步。数据库完全隐形，状态由摊位自动管理。你只需在内存里修改上下文，剩下的事情，Aura 的 Rust 运行时自动处理高并发与多机备份。

## 0. 架构纲领：底层框架融合

当前工程实践的一条清晰主线：**网关、服务编排、Web 框架、存储、消息队列等底层框架正在走向融合**。Rivet 将摊位运行时与存储同机共生，Windmill 将任务编排与 FaaS 执行层合为一体，Dapr 试图用 Sidecar 统一服务间通信与状态管理——尽管其过于散碎，诉求是一致的。Aura 是这一趋势的工程实践：将上述五种能力全部编译进同一个 Rust 进程。

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

### 扩展单元：地址 vs 程序（方法论框架）

去微服务化上升到"可扩展应用如何组织扩展点"的抽象层，归结为一元判定：**扩展单元是"地址"还是"程序"**。

- **地址（RPC / 进程外）**：核心配置的是**接口端点**（URL / socket / 协议），运行时去调用远处。Webhook 只是其远程体，本地 socket / stdio 是近端体。跨进程、各自成语言、松耦合，但每次调用付网络/进程往返。若异步、来/回不一一对应（如事件订阅 pub/sub），则缺乏**确定性契约**——它只是 RPC 的异步组织形态，扛不起整体扩展点的职责，不是独立范式。
- **程序（内嵌动态代码）**：核心配置的是**逻辑**，进程内就地执行。按「表达力 × 装载」分一谱系：
  - **通用脚本**：Steel / Python（PyO3）等全表达力语言；
  - **受限 DSL / 配置即程序**：核心 = 该受限语言的解释器，配置 = 该语言源码（规则表、路由策略）；
  - **编译产物**：so/dll → **WASM**（跨平台 + 沙箱；so/dll 既不跨平台又无隔离）。

**耦合与安全的收束**：RPC 松（扩展在协议外、崩溃隔离），内嵌紧（进程内）；但 **WASM 用沙箱把"紧"的代价收回**——内嵌的高级档反而兼得轻耦合与安全。这是 WASM 取代 so/dll 的另一理由。

**Aura 的选择正是这个裁决**：反对「地址 / 网络 RPC」（上节去微服务化动因）、采用「程序 / 进程内多语言内嵌」（Steel / Python / Wasm，零 IPC、单二进制），并用 WASM 沙箱承接「第三方不信任代码」。内嵌谱系的语言级深化见 [embedded-script-languages](embedded-script-languages.md)。

### 单体原有问题的当代解法

单体被淘汰不是因为"单体"本身有错，而是因为当时的工程手段无法在单体内部解决三个问题。这三个问题在今天已有成熟解法：

| 单体原有问题 | 当代解法 | Aura 中的对应 |
|-------------|---------|--------------|
| **语言锁定**：全系统被迫使用同一种语言，无法为不同场景选最优工具 | 嵌入式多语言运行时（进程内 VM，零 IPC） | Steel / Python / Wasm 三种语言通过 PyO3 和 Wasmtime 内嵌，摊位按需选择 |
| **团队耦合**：修改一处可能影响全局，多团队并行开发冲突频繁 | 事件契约解耦（声明式接口，编译期/加载期校验） | `interface_schema()` 声明 receives/emits，Realm 强制白名单校验 |
| **无法独立伸缩**：整体部署，热点模块无法单独扩容 | Virtual Actor 按需激活，空闲自动 Scale-to-Zero | instance key 路由 + Fjall 磁盘休眠 + Tokio 事件循环 |

这三个解法恰好对应微服务当初试图解决的三个痛点。区别在于：微服务用"拆进程"来解决，Aura 用"进程内隔离"来解决——后者没有网络开销。

### Aura：标准化的融合方案

去微服务化不是回到传统单体，而是走向一种新的标准化形态：**一个进程内包含完整的分布式能力**。当前的去微服务化实践大多是 ad-hoc 的——每个团队根据自己的场景选择不同的组件拼凑（嵌入式 KV + 自建事件总线 + 自己写状态管理），缺乏统一的架构模式。

Aura 将这种融合标准化为一个可复用的引擎：

| 能力 | 传统拼凑方案 | Aura 标准化方案 |
|------|------------|----------------|
| 服务编排 | 自建摊位系统 + 手写状态机 | 场域模型（emit/on/on_join/on_batch）+ `interface_schema()` 自动路由 |
| 状态管理 | Redis / Memcached / 自建 HashMap | Fjall LSM-Tree（WAL 断电保护 + Scale-to-Zero） |
| 跨节点协调 | 各服务自行实现 or 不实现 | 元数据每节点独立（数据面 okm 实例内的 meta 表，控制平面单写，无共识——ADR-0025）；摊位数据走 Fjall+落湖 或 SlateDB+S3；联邦节点间经 well-known 协议认证 |
| 多语言支持 | 各服务独立运行时（微服务模式） | 进程内嵌入（Steel/Python/Wasm），零序列化 |
| 部署 | Docker + K8s + Helm + 服务网格 | 单二进制，`scp` 即部署；多机组网走联邦（well-known 协议认证），无需共享元数据设施 |

标准化的意义在于：开发者不再需要为每个项目重新设计"怎么把多个框架拼在一起"，而是直接使用一个经过验证的融合引擎，将精力集中在业务逻辑上。

### 融合的边界：不是分布式单体

融合不等于把多个服务共享一个数据库（分布式单体）。分布式单体同时承受微服务的部署复杂度和单体的耦合强度——改一个服务的 schema 全线牵连，但你享受不到单体的简单性。Aura 的做法是**物理单体，逻辑分布式**：

- **物理层面**：单二进制、进程内通信、零网络 RTT——取单体的性能与简洁
- **逻辑层面**：每个摊位类型占据一个类型 ns（okm 键段，声明 collections，ADR-0026），实例是 ns 内的 document，通过场域事件（emit/on）松耦合通信，运行时按 instance key 激活和驱逐——取微服务的解耦与弹性

| 维度 | 微服务 | 分布式单体 | **Aura** | 单体 |
|------|--------|-----------|---------|------|
| 通信 | 网络 RPC | 共享数据库 JOIN | **进程内 emit/on** | 进程内函数调用 |
| 耦合 | 松 | 紧（共享 schema） | **松（事件契约）** | 紧 |
| 部署 | 多二进制 | 多二进制 | **单二进制** | 单二进制 |
| 语言 | 每服务可选 | 每服务可选 | **每摊位可选** | 全局统一 |
| 状态隔离 | 天然隔离 | **无隔离（共享 DB）** | **独立 Fjall 分区** | 共享内存 |

Aura 融合了微服务的解耦能力和单体的性能优势，同时规避了两者的固有缺陷——以及分布式单体这一反模式的全部代价。


## 1. 计算下沉与核心架构拓扑

在现代通用开发环境中，盲目引入外部易失性内存缓存已被证明会带来严重的工程债务与认知负担。本架构基于第一性原理，确立了"计算紧贴存储"与"单体事务原子化"的核心原则，彻底排除了 Application-Side Joins（应用端拼装）的低效模式。

### 1.1 传统"多轮网络往返（N+1 Queries）"的失效分析

在处理复杂的领域模型时，外部缓存迫使开发者在应用端用代码扮演低效的"伪查询引擎"。

- **数据空耗**：为了组装一个包含多层嵌套关系的业务对象（例如：用户 → 订单 → 商品 → 库存），应用服务器必须通过网线高频发起多次网络往返（RTT）。
- **算力开销**：应用服务器耗费了大量的 CPU 周期，用于对传输过来的海量 RESP、JSON 或 Protobuf 格式数据流进行反复的序列化与反序列化，在应用内存中合并碎片。

### 1.2 本架构的设计机制

- **核心公式**：真实总延迟 = 网络 RTT + 数据库内部处理时间

由于现代局域网/VPC 内部的物理 RTT 往往高达 1.0ms 至 3.0ms，外部缓存零点几毫秒的纯内存优势在网络延迟面前毫无意义。

- **计算下沉（Compute Near Data）**：通过直接压榨嵌入式 KV 的 LSM-Tree Block Cache 与进程内零拷贝路径，实现单次请求，单次返回，直达本质。

→ 详见 [Redis 批判](redis-critique.md)、[MySQL 批判](mysql-critique.md)、[SurrealDB 分析](unified-data-layer.md)



为了移除 JavaScript/TypeScript 运行时（Rivet）带来的运行时开销，以及 Python 带来的执行损耗与环境污染，本架构采用 **Rust（高性能外壳）+ Steel（确定性核心）** 的双层解耦拓扑。

```
[网络请求 (REST/WS)]
│
▼
[Tokio 异步事件调度] ──► [唤醒对应 ID 的 Rust 摊位] ──(从 Fjall 本地磁盘读出最新 Context)
│
▼
[摊位落盘休眠 (Scale to Zero)] ◄──(单次 WAL 写入 Fjall) ◄── [Steel Lisp 虚拟机内核运行]
```

Rust 原生实现的摊位引擎利用 Tokio MPSC 管道建立低开销的 Host 外骨骼，避免多线程数据踩踏。每个摊位贴身绑定内存状态（无共享锁），闲时冬眠（Scale to Zero），被请求唤醒时瞬间拉起轻量级 Steel Lisp 虚拟机作为最高策略核心。Host 与 Steel Lisp 之间通过内存指针直接内嵌打通，零拷贝映射内存状态至 Lisp 变量。完整的零共享锁摊位实现详见 [§2.2 多语言网关](#22-多语言网关纯-rust-混合-摊位-实现)，其中 `exec_steel_lisp` 即为单一 Steel 版本的实现。


## 2. 架构升级：多语言混合动力运行时（Polyglot Embedded Harness）

本架构已从单一的 Steel Lisp 策略层演进为多模态混合运行时控制台。通过在 Rust 主摊位内部建立统一的"多语言网关接口（Polyglot Bridge）"，针对不同业务场景派出最合适的"工具人"，同时保持快速启动与 Scale to Zero 的核心设计。

### 2.1 多语言混合动力：用户选择指南

框架提供两种嵌入式脚本语言 + 一种沙箱运行时。**每个摊位只选一种脚本语言**，但**语言用途无硬限制**——下表中的"适用场景"仅为建议，开发者可根据实际需求自由分配。框架不做智能分发。

| 技术选型 | 类型 | 底层虚拟机/隔离机制 | 核心价值 | 适用场景 |
|---------|------|-------------------|---------|---------|
| **Steel (Lisp/Scheme)** | 脚本语言 | 字节码虚拟机 + 卫生宏 | 100% 无二义性，天然沙箱隔离（默认不带危险系统 I/O）| 高危策略防线、权限路由、复杂元编程 DSL |
| **Python (PyO3 内嵌)** | 脚本语言 | CPython 解释器内嵌 (GIL 内存级捕获) | 零开销复用全球最庞大的 AI/Data 生态圈 | 遗留 AI 代码资产重组、Tokenizer 矩阵计算、Numpy/HuggingFace 调用 |
| **Wasm (Wasmtime)** | 沙箱运行时 | Cranelift JIT 编译器 / 硬件沙箱 | 接近原生 CPU 性能，硬件级隔离 | 第三方不信任代码托管；存储型 Rust 服务（k10r/gravity 一类，存储经帧上抛由 host 承载）。wasm-gc + WASI 成熟后可扩展为统一运行时（详见 [Wasm 统一运行时](wasm-unified-runtime.md)） |
| **Rust 主外壳** | 宿主 | Tokio 异步协程 + MPSC 状态管道 | 低开销启动，27MB 内存，Scale to Zero 核心设计 | 分布式网关、系统 I/O、事件派发 |
| **持久化** | 存储层 | Fjall/SlateDB（data/meta 两实例分离，元数据单写无共识） | LSM-Tree + Raft 共识 | 无外部缓存中间层，单次事务原子固化 |

**关键澄清**：

- **脚本语言（Steel/Python）是摊位的业务逻辑载体**，每个摊位选一种。
- **Wasm 是隔离运行时**，定位经历了一次扩展：
  - 初版定位是「托管第三方不信任代码」——编写语言通常是 Rust，但 host 不关心，只消费 `.wasm` 二进制。
  - 随着 k10r、gravity 一类 Rust 服务以 wasm 摊位形态进场域，定位扩展为**重隔离的服务承载，且是 Rust 服务的唯一发布形态**：服务编译为 `.wasm` 运行时上传（`set(lang="wasm", bytes)`），不编译进 host。
  - 编译进 host 的后果：每个应用 fork 一份 aura，加服务就要重打包，平台退化成框架。
- **引擎不提供进程内 Rust 摊位**：框架自身机制（evictor 类）就是 realm 内的普通逻辑，包装成摊位绕一圈没有意义。Rust 代码成为摊位只有一条路：编译为 wasm 上传——不是 dylib、不是编译期、不是闭包。
- **存储也不进沙箱**：服务自带的 OKM schema 原样编译进 wasm，`VirtualStorage` 的实现替换为帧上抛（okm-wire 帧），host 侧 NestStorage 执行器在 registry 分配的 app ns 前缀下承载物理存储（Krystallizer ADR-0007 存储承载分流；静态 derive，不需要 okm-dynamic）。
- **wasm carrier ABI 已落地**：
  - handler 以事件名导出（多入口）。
  - 值以 CBOR 过线性内存：host 经 guest 的 `aura_alloc` 写入参数、调用 `(ptr, len) -> i64`，返回打包 `(ptr:u32)<<32|len:u32`。
  - ctx bridge host imports 注册在 `aura_host` 命名空间，统一签名。
  - `interface_schema` 显式导出优先，否则由导出清单推导。
  - JSON 只在 host 侧 `ResidentSession` 边界出现，存储层与 wasm 桥零 JSON 文本。
- **语言用途没有硬性规定**：Steel 可以写业务逻辑，Python 可以写策略，只要开发者认为合适即可。

**持久化选型补充**：摊位消息有强**时间局部性**——热点集中在最近写入的状态，SlateDB 刚写入即落在 memtable / 缓存，冷读 S3 只在偶发点查触发，可接受。因此 **SlateDB 可取代「Fjall + Lakehouse」两件套**（单真相、免 flush 管线）；仅当热路径对进程内 ns-μs 有硬要求且读取无时间局部性时才退回 Fjall。若需强一致分布式复制，不自建——直接用 FoundationDB / TiKV。详见 [KV 存储引擎](kv-storage-engine.md) §两条分布路径/强一致分布式 与 [统一数据层](unified-data-layer.md)。

**语言限制的设计理由**：框架限制为 Python/Steel/Wasm 三种语言，不是"不能支持更多"，而是**限制 AI 的生成空间**。AI 生成代码的质量和语言复杂度成反比——语言越少、边界越清晰，AI 生成的代码越可靠。组件集封闭有限（enum 门卫 + 手动过 enum 加组件），和 Aura 的摊位枚举设计一致：AI 需要完整词汇表才能正确生成 JSON，语言数量影响判断准确率。不限制语言看似灵活，实际上是把选择负担推给了 AI 和开发者——AI 要在无限选项中选最优，开发者要在无限组合中调试兼容性。限制语言 = 降低认知复杂度 = 提高生成质量。

**引擎 vs 应用的评价标准**：Aura 是引擎和平台，不是应用。评价应用问"你解决了什么问题"（"用 PostgreSQL 就行了"是合理的应用层回答），评价引擎问"你能让人解决什么问题"（引擎的价值在于它赋能的上层应用的数量和复杂度）。用应用标准评引擎——"你不需要造发动机，马车够用了"——是层次混淆。引擎的对手不是 PostgreSQL，是其他引擎（Akka、Orleans、Tempo）。引擎的客户不是终端用户，是用引擎构建应用的开发者。

**Steel 的选择理由**：Steel 不是"特殊地位"，是三种语言之一。但 S-表达式有独特的工程价值：小括号 `()` 在物理上将所有作用域和运算边界锁定，空格敏感语言（Koto、Python）的"空格 = 语义"设计在团队协作和 Git 合并中制造不可预测性。Steel 的零二义性让 AI 生成的代码更可靠——Lisp 的同构性（Homoiconicity）让 Transformer 天然擅长生成，S-表达式是 AI 最容易正确生成的语法。但 Steel 数据密度低（GitHub 上代码少），AI 裸写时会调用标准 R7RS Scheme 的直觉产生幻觉——需要在 `CLAUDE.md` 中用 Few-Shot 约束。

→ 详见 [嵌入式脚本语言选型](embedded-script-languages.md)

**wasm-gc 成熟后的简化路径**：Rust → Wasm 现在已可用（不需要 wasm-gc），异步场景可由 Rust → Wasm 覆盖。wasm-gc 落地后，Kotlin/C# 等语言涌入 Wasm 生态，语言选择进一步丰富。Steel 和 PyO3 不受影响——Steel 的 S-表达式确定性边界是 Wasm 不能替代的；Python 过于动态、编译到 Wasm 极难甚至不可行，但也不需要——PyO3 进程内零拷贝调用开销本身就低，即时执行无须编译的交互模式是 Python 的固有价值。详见 [Wasm 统一运行时](wasm-unified-runtime.md)。

**语言无硬限制**：框架通过 PyO3 和 WASM 两个通道接入外部语言，理论上任何能编译到 WASM 或能通过 C ABI 调用的语言都可以进入摊位进程。框架内置 Steel/Python/WASM 三种运行时，但不阻止用户通过 PyO3 的 C 扩展机制接入其他语言（如 Lua、Julia）。

**外部调用**：框架通过 `ctx.invoke()` + 统一调用注册表提供同步调用（详见 aura 仓库 [`docs/design/realm.md`](https://github.com/orbsh/aura/blob/main/docs/design/realm.md) §ctx.invoke 统一调用原语）。Host 管控调用生命周期（超时、审计、可观测），Fluxora 负责 HTTP 请求。摊位不直接通过 PyO3 调用外部系统——这绕过 Host 管控。进程内调用（PyO3/WASM）始终优先于跨进程调用。

**流式调用（iterate）**：与 emit/on/invoke 并列的第四个原语（ADR-0034）：`ctx.iterate(target, handler, args)` 返回游标，逐条拉取类型化信封（ADR-0036 统一：`done` 恒布尔、非终止带 `item`、终止可带 `value`——invoke 即首轮即终止的 1-流，`ctx.invoke` 是它的取值糖）——终止、背压、消费方活性由拉取结构内建，不是约定。生产方形态按语言分流：python 写 `yield` 生成器（框架停放驱动，耗尽即 done）；steel/wasm/bgi（nushell 的 PTY 载体已退役，nu 摊位走 bgi 双 fifo 形态）无宿主可驱动生成器，handler 显式写信封（Rust-wasm guest 在模块状态内映射 `Iterator`）。消费方形态同样按语言：python 脚本用原生 `ctx_iterate(...)` generator（break 自动 dispose），Rust 用游标，其余拉到 done；每次拉取是 hot 调用，骑既有调用机件（流不持久、不可重放——要 at-least-once 用事件）。动机场景：provider 摊位（python `httpx.stream` 消费 SSE → yield per token）供 wasm 兄弟摊位消费，落实 ADR-0031「对外访问归摊位代码」。

**AI Agent/Harness 场景的分层**：

Agent 的逻辑分为两层——编排层和工具层，分别用不同语言实现：

- **Agent 主循环（Wasm/Rust）**：接收 → LLM 调用 → 工具调用 → 返回。稳定，性能优先。Wasm 摊位通过 host 函数调用 `ctx.invoke()`，Host 在 Wasm 挂起时执行异步操作，对 Wasm 透明。
- **工具接入**有两种模式：
  - **基于 SKILLs**：所有工具统一走 `perform()` 接口——结构化、可组合、可发现。内置操作（文件 I/O、网络请求、HTTP 客户端）和业务逻辑都封装为 Skill，Agent 通过统一接口调用。涌现式 Skill 通过 graph-memory 高权重子图自动聚类，Agent 运行时发现并调用。
  - **不基于 SKILLs**：直接 host 函数（Rust/Wasm）+ 脚本（Python）。轻量场景，不需要 Skill 框架的开销。通用稳定操作走 host 函数，业务特定逻辑走 invoke.toml 注册。

Agent 主循环在单个摊位实例中运行（instance key = session_id）。同一会话内的工具调用串行（摊位单线程语义），不同会话并行。Agent 处理完请求后进入 `on_sleep`，状态落盘 Fjall，内存归零（Scale-to-Zero）。下次消息到达时 `on_wake` 激活恢复。流式输出通过高频 emit `agent.stream` 事件实现，Fluxora 转为 SSE/WebSocket 推送。

主循环本身可以进一步收缩为纯函数——会话即数据：摊位按 `session_id` 从记忆系统取完整会话 → 跑 turn → 存会话，会话持久化与恢复全部沉到记忆系统，摊位在两次调用之间无状态，`on_sleep`/`on_wake` 退化为存取两个动作。循环不再持有会话状态、不改写历史，所有函数调用对它都是普通记录；工具调用格式的裁剪发生在记忆系统的序列化视图层，存储层始终完整。

无状态 Agent 的完整组件架构独立成篇：循环组件（Gravity）、入口（Prism）、执行器（Effector，Aura 内嵌/远程执行器两形态）、skill 涌现闭环与传输裁决，见 [无状态 Agent 架构](stateless-agent-architecture.md)；记忆侧设计见 Krystallizer ADR-0006（无状态 Agent 集成）。

工具和 Skill 不是静态文件，而是 graph-memory 中高工具指数的子图——使用数据自动聚类涌现 Skill 边界，Agent 运行时通过向量搜索发现相关 Skill 边，按权重排序注入上下文。

#### 2.1.1 多 Skill 编排：Pipeline as Code

**问题**：当前 Agent 框架（Hermes、Agno 等）的标准模式是 LLM 实时编排——每调一个 Skill 都是一次独立推理，上下文在对话中传递。调到第 3、4 个 Skill 时，LLM 可能已经"忘了"第 1 个 Skill 返回的关键字段。同样的输入，不同次运行可能走不同路径。编排过程是黑箱，出错不可见。

**约束**：Aura 禁止 LLM 拼字符串生成代码（§2.1 语言限制的设计理由：限制 AI 的生成空间，提高生成质量）。但多 Skill 编排需要确定性执行。

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

### 2.2 多语言网关：纯 Rust 混合摊位实现

将四大嵌入式语言运行时全部内嵌进同一个 Tokio 异步摊位实例，打造多模态混合执行环境。整个调用过程完全在进程内存（In-Memory）中流式交织：

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

pub struct MasterAgentBooth {
    agent_id: String,
    receiver: mpsc::Receiver<PolyglotTask>,
    memory_context: ciborium::Value, // 动态结构化状态，CBOR Value 树
}

impl MasterAgentBooth {
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

### 2.3 彻底打破 NoSQL 宿醉：多语言混合下的"单体关系型死守"

很多跟风的架构师一看到这个方案支持多种嵌入式语言，第一反应就是："那我是不是得搞一个复杂的状态同步层，或者引入一个外部内存数据库来让这几种语言进行数据共享？"

**不。这恰恰是本架构最清醒、最精益的地方：**

- **内存空间的统一**：Steel 虚拟机、Wasmtime 和 Python 的 PyO3 全部作为动态链接或 C-Binding 静态编译进同一个 Rust 进程的内存地址空间。摊位状态住在类型声明（ADR-0026 §3）的 collections 里，实例是类型 ns 内的 document，document 的动态段按字段读写，各语言通过 ctx.store.emit 指令桥接——Host 解析指令后读写本类型 ns，转为各语言的原生类型注入虚拟机。进程内调用，无跨进程 IPC，无网络序列化开销。

**状态按 collection/document 分存**：摊位状态不是单个大 CBOR blob，而是按类型声明的 collections 组织：

```
type ns (注册时分配) /
  history/{instance_key}   = [10000条记录]
  profile/{instance_key}   = {...}
  settings/{instance_key}  = {...}
```

handler 按需 emit 指令读写，不加载不需要的 collection：

```python
def handle(ctx, user_message=None):
    if user_message:
        # 只读本类型 collections 里的 history，不碰其它 collection
        recent = ctx.store.emit(get_document("history", ctx.self_id.key))

        recent["records"].append(new_record)
        ctx.store.emit(put_document("history", ctx.self_id.key, recent))

        # 已落盘（WAL + memtable）
```


- **数据载荷的动态性**：摊位的输入/输出在进程内为 `ciborium::Value`（内存值树），零序列化。跨语言边界（Steel/Python/Wasm VM）时序列化为 CBOR `Vec<u8>`——schema 不固定，支持嵌套对象、数组、数字、布尔。Steel Lisp、Python（cbor2 库）、Rust Host 都能编解码。不同于 Postcard（需要编译期 Rust 类型），CBOR 自描述，适合跨语言动态数据交换。

**CBOR 改进方向**：当前 CBOR 的键名（key name）在高频小消息场景有开销——每个字段都带完整字符串键名。改进方向：消除键名开销（键名索引化或列族式存储），动态列式布局（类似 LSM-Tree 的列族设计），但不能拖累查询性能。这是工程优化，不是架构变更——CBOR 作为跨语言数据交换格式的选择不变，只是编码效率提升。注意：这不是走向"schema 约束"的方向（Rust 结构体状态已经是类型安全的），而是保持动态性的同时压缩编码体积。

- **状态落盘的原子化**：多语言在一轮交互中通过 `ctx.store.emit` 指令写 WAL 落盘（ADR-0026 §3：状态住在本类型声明的 collections，实例是类型 ns 内的 document）。Fjall 的 LSM-Tree 天然支持高频小写入。摊位状态写入本地 Fjall（不走网络路径）；元数据（摊位注册、路由、本节点配置）写数据面 okm 实例内的 meta 表（摊位Def，ADR-0025），控制平面单写 + 节点缓存——**用户数据（含登录态）绑定所属节点，不做全局同步**：用户登录其它节点，该节点没有它的历史数据，跨节点只按 well-known 协议认证身份。

**网线里没有多轮的 RTT 消耗，没有外部缓存的易失性风险，没有多库同步的分布式 Bug。**

用 Rust 搭了外骨骼，用 Lisp 锁定了边界，用 Python 接入了生态，用 Wasm 隔离了黑盒，持久化层用 Fjall/SlateDB（data/meta 两实例分离）。


## 3. 存储架构

存储层通过 trait 实现引擎可插拔——存储引擎（KV 读写）与分发层（多节点协调）正交组合。要点：

- **双轨实现**：`FjallEngine`（本地 NVMe，同步 I/O 经 spawn_blocking 包装，私有部署亚毫秒延迟）与 `SlateEngine`（SlateDB → S3，天生 async，云原生无状态计算）
- **单 API**：`ctx.store.emit`（摊位状态，本类型声明的 collections，不走网络路径——ADR-0026 §3）；注册表/分片映射等是引擎内部结构，不对摊位暴露读写面（ADR-0025）
- **单机起手 ≠ 分布式宿命**：Fjall 单机起步，数据不复制；SlateDB+S3 是云原生态；元数据不引入共识——每节点独立，控制平面单写；跨节点按联邦形态（well-known 协议认证）演进
- 摊位状态有强时间局部性（热点集中在最近写入），SlateDB memtable 命中即可接受冷读 S3，可取代「Fjall + Lakehouse」两件套

完整设计细节（trait 签名、AuraCollection 绑定容器、分发层、配置评估、多模态路由）见 **aura 仓库 [`docs/design/storage.md`](https://github.com/orbsh/aura/blob/main/docs/design/storage.md)**。


## 4. 序列化协议分析对比

→ 详见 [序列化协议分析对比：IDL vs Code-First](serialization-protocol-comparison.md)

### Aura 的分层序列化策略（2026-06-20 更新）

| 层级 | 序列化格式 | 理由 |
|------|-----------|------|
| **摊位状态持久化（Fjall）** | CBOR | 动态结构化数据，自描述，跨语言（Steel/Python/Rust）编解码 |
| **摊位输入/输出载荷** | `ciborium::Value`（进程内）/ CBOR bytes（跨边界） | 进程内为内存值树，零序列化；持久化/复制/外部投递时序列化为 CBOR bytes |
| **Raft 元数据（LogId 等）** | Postcard | 仅外部强一致方案（TiKV 等）用；本架构默认无共识层 |
| **分析查询路径** | Arrow IPC | 列式对齐，Fjall 读出后 Polars 零拷贝转铸 DataFrame |
| **KV value 只读点查（候选）** | rkyv | 纯 Rust mmap 零拷贝，O(1) 点查；仅进程内热路径，跨语言/出口仍走 CBOR/Arrow（见序列化文档方案 B3 落点边界） |
| **云端长期记忆 Lakehouse** | Lance | 内嵌向量与倒排索引，原生支持远程 S3 流式检索 |
| **海量冷历史冬眠** | Parquet | 高压缩率冷存储 |
| **UI ↔ Gateway** | CBOR | 跨语言（WASM），自描述 |

**核心原则**：技术不应该是束缚开发者手脚的繁文缛节。看清物理硬件的边界，去掉无谓的 IDL 嵌套，才能让 Aura 引擎在多核 CPU 和 NVMe 之间保持高效运行。

---


## 5. 场域模型：摊位间交互与外部世界

场域（Event Realm）是引擎内部的事件空间：摊位不直接寻址，通过 `emit(name, data)` / `on(name, fn)` 交互——发射者不关心谁处理，处理者不关心谁发射。事件名就是引用，instance key 就是实例定位。要点：

- **ctx 边界**（ADR-0011）：ctx 只收实例绑定 + Host 管控的能力（store-emit / interface_schema / invoke——点状态面 ctx.state 已退役，ADR-0026）；emit/on、契约、钩子留在场域层/静态契约
- **interface_schema()**：由 `@on` 装饰器推导（receives：事件名 → key 字段；通配符入 wildcard_receives）——emits 不声明、不收集、不校验（ADR-0012）：接收者集合是运行时事实，无订阅者的 emit 落入 dead-event ring，那是可观测的审计面
- **实例化与分片**：同一 instance key 串行（状态一致），异 key 并行；**数据跟随所属节点（联邦裁决）**——instance key 作用域是节点内部，无全局放置、无跨节点重分片
- **跨 Partition 查询**：投影摊位（持续聚合，推荐）/ Arrow HTAP（ad-hoc 列式扫描）
- **统一调用模型**：`ctx.invoke()` 是唯一受控调用面；冷调用（触达人类/外部系统）wait 不进入 park，挂起写事件流，`resolve_call` 重入
- **投递语义**：进程内事件因果有序；无共识层——跨节点按联邦形态，域间消息显式寻址（不做全局事件排序）；物理依据与尺度量化见 [分布式协作拓扑](distributed-collaboration-topology.md) §2
- **持久事件队列（ADR-0014）**：事件投递的存储形态是 okm 内嵌持久分区（`[mq-data][event][part_id][time]` + `[mq-cursor][event][part_id][摊位]`），不是内存 channel——emit 即落盘（被动保存与摊位状态的主动保存同引擎双轨）、实例驱逐期间的积压在重新激活后照常送达（scale-to-zero 不丢触发）、慢消费者积压可见且可「跳到最新」（skip-to-now 兜底阀门）、N 个订阅者 = 一个分区 + N 个游标（per-摊位 mailbox 的 N 份复制从结构上消失）。**保留 = 活跃订阅者的最小水位线**（分母来自路由注册表，陈旧游标不钉死水位线；积压深度是 mq-data 前缀的 okm reduce 实时计数）；队列是缓冲不是存储——at-least-once 仅在订阅者保持注册期间成立，摊位状态（类型 collections）是 durable truth。「不引入队列组件」指不引入外部重型队列；嵌入式持久分区是事件被动保存的自然形态。当前实现为 broadcast 过渡形态，持久化重写在 4.5c step 2b

完整设计细节（路由表结构、四种触发模式 on_join/on_batch/on_debounce、Collector、通配符匹配、投递代码、宿主实现、MQ 分解）见 **aura 仓库 [`docs/design/realm.md`](https://github.com/orbsh/aura/blob/main/docs/design/realm.md)**。

## 6. 开发体验（DX）：从 Rivet Actors 的启发到 Aura 的改进

> 灵感来源于 Rivet Actors 的 Actor 模型开发体验，但 Aura 在类型安全、冷启动性能、状态持久化和多语言支持上做了根本性改进。

### 6.1 Rivet Actors 的 DX 基线与 Aura 的改进点

Rivet Actors 提供了优秀的摊位开发体验：TypeScript SDK、自动 HTTP 端点生成、内置状态管理、WebSocket 连接管理、本地开发服务器。但它的核心限制在于：

| 维度 | Rivet Actors | Aura 的改进 |
|------|-------------|------------|
| **语言** | TypeScript/JavaScript（V8 隔离） | Rust 核心 + Steel Lisp/Python/Wasm 嵌入（进程内，无 IPC） |
| **系统启动** | 数百毫秒（Node.js 进程 + V8 初始化） | 毫秒级（Tokio 运行时 + Fjall 打开） |
| **摊位唤醒** | 几毫秒（V8 虚拟机激活） | 微秒级（Steel 字节码 VM 瞬时创建；Python PyO3 ~1ms） |
| **状态存储** | SQLite（同机共生，但单机瓶颈） | Fjall LSM-Tree（嵌入式；元数据每节点独立单写，数据落湖/S3） |
| **类型安全** | TypeScript（运行时类型，编译期弱） | Rust 编译期强类型 + Steel Lisp 的 S-表达式零二义性 |
| **多语言** | 仅 JS/TS | Rust/Steel/Python/Wasm 四语言进程内混合 |
| **状态持久化** | SQLite 文件 | Fjall KV + CBOR 序列化 |

### 6.2 摊位定义与生命周期摊位通过 `set(lang, script)` 提交实现（详见 aura 仓库 [`docs/design/realm.md`](https://github.com/orbsh/aura/blob/main/docs/design/realm.md) §摊位定义接口）。摊位是**多入口**的：每个 handler 用 `@on(event, key=...)` 装饰器声明监听的事件（python 装饰器 / steel `on` 函数 / wasm 导出约定），事件名即 handler 寻址名；`interface_schema` 由装饰器推导（receives 不再手工声明；emits 不声明不收集，ADR-0012），声明了 key 的事件按 `(event, partition)` 队列投递，无 key 按 event 单队列——事件不属于任何摊位，一个事件队列可有多个订阅者摊位（一对多是结构性的）。上传时 Host 自省一次并把元数据持久化到 meta store；执行路径永不调用 `interface_schema`。

生命周期：

1. **上传（set）**：独立生命周期，可以永远不执行——Host 自省 `interface_schema()`（或从 `@on` 装饰器推导）一次，元数据（receives/lifecycle）与脚本一并持久化到 meta store，receives 派生投递路由
2. **唤醒（on_wake）**：有事件到达时，Host 激活实例（定义已持久化，状态按需经 ctx.store.emit 从本类型 ns 的 collections 读写）；持久队列里该实例未消费的积压事件按游标继续送达
3. **处理**：按事件名寻址 handler 执行；handler 内可 `emit()`，状态立即持久化；永不调用 `interface_schema`
4. **休眠（on_sleep）**：空闲后状态落盘到 Fjall，内存归零（Scale-to-Zero；per-type idle_ttl 决定驻留窗口）
5. **版本变更**：新 `set` 重新自省一次、更新持久化元数据与路由；此前旧元数据治理

定时任务通过声明式注解，不需要外部 CronJob：

```python
@cron("0 2 * * *")
def daily_sync(ctx):
    orders = ctx.store.emit(get_document("orders", ctx.self_id.key))
    sync_to_remote(orders["pending_orders"])
```

**与 Rivet 的关键差异**：
- Rivet 的摊位状态是 JS 对象序列化到 SQLite，Aura 的状态是类型 collections 里的 document（nTLV 编码）落在 Fjall
- Rivet 的 cron 是外部调度器触发，Aura 的 cron 是摊位内声明式注解
- Rivet 需要 `摊位.setState()` 手动调用，Aura 的 set/append 立即写 WAL 落盘
- Rivet 的摊位逻辑只能用 JS，Aura 的 `set()` 支持运行时切换语言

### 6.3 事件发现：interface_schema() 约定

脚本以 `@on` 装饰器声明接收的事件；Host 上传时自省一次（或从装饰器推导）构建事件路由表，元数据持久化到 meta store，执行路径永不自省。详见 aura 仓库 [`docs/design/realm.md`](https://github.com/orbsh/aura/blob/main/docs/design/realm.md) §interface_schema。

**热重载**：脚本修改 → 重新加载 → 重新 `interface_schema()` → 路由表更新。无需重编译 Rust host。

**Rust 摊位的情况**：Rust 写的摊位同样遵循 `interface_schema()` 约定，但宏在编译期自动生成，不需要手写：

```rust
// #[aura::schema] 宏在编译期扫描，自动生成 interface_schema() 导出
#[aura::schema]
#[derive(Serialize)]
struct Order { id: String, item: String }

// 事件处理通过宏注册
#[aura::on("order_created")]
async fn handle_order_created(ctx: Context, data: Order) -> Result<()> { ... }
```

**唯一约束**：脚本必须导出 `interface_schema()` 且返回符合结构的 JSON。违反则拒绝加载。脚本即文档——看到 `interface_schema()` 就知道这个摊位接收什么事件、发射什么事件。

### 6.4 本地开发体验

Rivet 有 `rivet worker dev` 本地开发服务器。Aura 的本地开发更轻量：

```bash
# 单文件启动——不需要 Docker、不需要 etcd、不需要数据库
$ aura dev order_booth.py

# 输出：
# [Aura] 摊位 order_booth 已启动
# [Aura] 场域: default
# [Aura] 状态: Fjall 本地模式 (./data/摊位/)
# [Aura] 热重载: 监听文件变化，修改即生效
```

**对比 Rivet**：Rivet 本地开发需要 Node.js 运行时 + V8 隔离层。Aura 是单二进制，下载即用，无运行时依赖。

### 6.5 测试体验

```rust
#[cfg(test)]
mod tests {
    use aura::test::RealmHarness;

    #[aura::test]
    async fn test_order_flow() {
        // 测试框架自动创建临时 Fjall 存储和场域，测试结束后自动清理
        let mut realm = RealmHarness::new("default").await;
        realm.set("python", "order_booth.py").await;

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

**与 Rivet 的差异**：Rivet 测试需要启动本地开发服务器或 mock SQLite。Aura 的测试是纯进程内——摊位在测试进程中直接运行，Fjall 用临时目录，无网络、无外部依赖。

### 6.6 部署体验

```bash
# 构建——单二进制，无运行时依赖
$ aura build --release
# 输出: target/release/order_service (12MB)

# 部署到目标机——不需要 Docker、不需要 K8S
$ scp target/release/order_service user@server:/opt/aura/
$ ssh user@server "aura serve order_service --port 8080"

# 多机部署——联邦形态：各节点独立部署，well-known 协议认证互认（数据不全局同步，跟随所属节点）
$ aura serve order_service --config /etc/aura/aura.kdl
```

**与 K8S 部署的对比**：

| 步骤 | K8S 部署 | Aura 部署 |
|------|---------|----------|
| 构建 | Dockerfile → docker build → push registry | `aura build --release` |
| 配置 | Deployment YAML + Service YAML + Ingress YAML | `aura serve --port 8080` |
| 状态存储 | PVC + StorageClass + PV | Fjall 内嵌（自动） |
| 多机元数据同步 | StatefulSet + etcd + headless Service | 无需同步——元数据每节点独立（控制平面单写），联邦间 well-known 协议认证 |
| 扩缩容 | HPA + Metrics Server + CPU/内存阈值 | 摊位自动 Scale-to-Zero |
| 证书 | cert-manager + ClusterIssuer + Certificate CRD | 内置 Let's Encrypt（可选） |

### 6.7 Rivet 没有但 Aura 有的

**1. 编译期状态安全**
Rivet 的 `摊位.setState()` 是运行时 API——传错类型、忘记调用、状态结构变更后旧数据反序列化失败，都要到运行时才暴露。Aura 的状态是 Rust 结构体，编译器在编译期就拦截所有类型错误。

**2. 多语言脚本 + 沙箱运行时**
Rivet 的摊位逻辑只能用 JS。Aura 的每个摊位可自由选择脚本语言——Python（AI/数据）、Steel（元编程）——不同摊位用不同语言，通过 `interface_schema()` 统一发现事件契约。此外，Wasm 沙箱运行时用于托管第三方不信任代码，硬件级隔离，与脚本语言互补。

**3. 确定性休眠/唤醒**
Rivet 的摊位休眠依赖 V8 堆快照，唤醒时需要反序列化整个堆。Aura 的摊位休眠是将 Rust 结构体通过 CBOR 序列化写入 Fjall，唤醒时从磁盘直接反序列化到内存——不依赖 V8 堆格式，不受 GC 暂停影响。

**4. 分布式状态复制**
Rivet 的状态是单机 SQLite，多副本需要外部同步。Aura 的元数据每节点独立（控制平面单写，无共识），摊位状态走 SlateDB+S3 或本地 Fjall，无需外部组件；联邦节点间 well-known 协议认证，数据跟随所属节点。

### 6.8 DX 设计原则总结

1. **单二进制，零依赖**：开发者不需要安装 Docker/K8S/Node.js/Python，下载 Aura 二进制即可开发、测试、部署
2. **状态即代码**：摊位状态是类型 collections 里的 document（ADR-0026 §3），通过 `ctx.store.emit` 指令立即写 WAL 落盘。Rust 摊位享有编译期类型安全，脚本摊位享有 JSON Schema 校验
3. **声明式生命周期**：`on_wake`/`on_sleep`/`cron` 注解声明摊位行为，不需要外部调度器
4. **本地即生产**：本地开发用 Fjall 临时目录，生产用 Fjall 持久目录，行为 100% 一致——没有"本地能跑线上炸了"的问题
5. **渐进式复杂度**：单机 → 多机组网走联邦形态（各节点独立 + well-known 协议认证；数据侧本地落湖或 SlateDB+S3），不需要重写代码或引入新组件

### 6.9 auractl：CLI 管理工具

Aura 提供两个一级子命令的 CLI 工具 `auractl`：

**`auractl realm`** — 场域管理操作：

```bash
# 提交/更新摊位定义
auractl realm set cart_booth python cart_booth.py

# 查看所有摊位定义
auractl realm list

# 查看某个摊位的 interface_schema
auractl realm schema cart_booth

# 向场域发射事件（调试用）
auractl realm emit add_to_cart '{"user_id": "A", "item": {"id": "X"}}'

# 查看活跃摊位实例
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

# 查看某个摊位的状态
auractl fjall get cart_booth.cart cart_booth:user_A

# 扫描某个 partition 的 key
auractl fjall scan booth_defs --prefix cart

# 压缩 LSM-Tree（手动触发 compaction）
auractl fjall compact

# 查看存储统计
auractl fjall stats

# 导出摊位定义（备份）
auractl fjall export booth_defs --output cart_booths.cbor

# 回滚摊位定义到上一版本
auractl fjall rollback cart_booth --version 2
```

`auractl` 直接读写本地 Fjall 实例（单机模式）或通过 Host API 远程操作（分布式模式）。两个子命令的职责分离：`realm` 管理运行时状态和事件，`fjall` 管理底层存储和版本化数据。


## 7. 全景解构：大厂基建与新锐智能体平台竞品分析

在将本架构投入真实的商业 Web 服务之前，我们需要跳出极客视角，以严苛的工业级标准对当前市场上几款最具代表性的智能体运行平台进行高维度对账。

### 7.1 跨代技术矩阵横向速查表

| 维度 / 平台 | Google AX + Substrate | Rivet Actors | Windmill | 本架构 (Rust+Steel+PG/Fjall) |
|------------|----------------------|--------------|----------|------------------------|
| **底层核心技术栈** | Go / K8s 控制面 / 容器沙箱 | Rust + V8 Isolates + FoundationDB | Rust (内核) + Python/TS (执行) + Postgres | 纯 Rust + 嵌入式 Steel VM + Fjall/SlateDB |
| **语言开销与大小** | 厚重（容器 Pod 级）| 中等（V8 进程隔离）| 中等（依赖的多运行时环境较庞大）| 极轻（单二进制文件，单会话 ~27MB 内存）|
| **语言生态友好度** | 模型无关，全语系支持 | 偏向 JavaScript/TypeScript | 极度偏向 Python | 移除 JS 污染，对 Rust/Lisp 原生极佳 |
| **冷启动 / 恢复延迟** | ~200 毫秒 (Pod 内存快照解冻) | 几毫秒级 (V8 虚拟机瞬时激活) | 数十毫秒 (FaaS 工作进程调度) | 启动为 Tokio 运行时初始化（毫秒级）；摊位唤醒微秒级 (嵌入式 VM 瞬时创建) |
| **状态持有与防失忆** | 事件日志回放 (WAL / Replay) | 计算与 SQLite 存储同机共生 | 分布式异步工作队列（偏向无状态短时任务）| Fjall/SlateDB 嵌入式存储（元数据每节点独立单写，无共识） |
| **空格敏感/边界歧义** | 视容器内运行的特定语言而定 | 视 JS/TS 闭包习惯而定 | 存在 Python 缩进与类型隐式转化断层 | 零二义性（S-表达式小括号确定性边界）|

> Rivet 的 DX 层面详细对比见 [§6.1](#61-rivet-摊位-的-dx-基线与-aura-的改进点)。

### 7.2 核心竞品深度诊断剖析

#### ① Google 阵营 (AX + Agent Substrate) —— 工业级重型装甲车

- **架构本质**：Google 旨在通过重构 Kubernetes (K8s) 底层调度来接管大规模企业级智能体。AX 负责通过事件日志（Event Logging）回放实现长期任务的"断线原地复活"，Substrate 负责利用 Pod 级别的增量内存快照实现处于挂起/等待状态的智能体自动冷冻（Scale to Zero），从而实现 97% 的硬件利用率提升。
- **技术痛点**：由于其以 Pod 容器为最小隔离单位，架构极度庞大且严重依赖 K8s 生态。对于中小型灵活的 Web 服务而言，其部署运维成本、冷启动响应延迟（~200ms 级别延迟在实时 Web 场景中依然过高）极不划算。

#### ② Rivet 阵营 (Rivet Actors) —— 游戏级高并发轻量战斗机

- **架构本质**：由高并发多人联机游戏基建演进而来。它抛弃了容器，采用纯 Rust 开发的底层运行时，内嵌 V8 Isolates 虚拟机孤岛。每一个智能体或长时会话在内存中就是一个高敏捷的摊位，身边绑着一个内存级 SQLite，冷启动和休眠恢复达到了毫秒级。
- **技术痛点**：它捆绑了 JS 生态。尽管其宣称底层由 Rust 压榨性能，但目前其官方 SDK 和暴露的应用层 API 几乎全向 JavaScript/TypeScript（Bun / Node.js）倾斜。如果你反感 JS 的运行时开销、动态类型的松散以及 V8 的隐式内存损耗，Rivet 就会成为你技术洁癖上的巨大障碍。

#### ③ Windmill —— FaaS 基因的 Python 军火库

- **架构本质**：一个 Code-first 的高并发分布式异步任务编排与轻量 FaaS 平台。其最大的亮点是零侵入性——直接利用 Python 的类型提示（Type Hints）在后台自动生成标准的 JSON Schema，从而让普通的 Python 脚本可以直接作为 Tool 回调被大模型完美识别。
- **技术痛点**：它在本质上不是一个智能体常驻平台。它的底层是为"短时任务（Short-lived Jobs）"和定时脚本设计的。一个 Python 任务跑完即释放，缺乏像 AX 那样的事件状态机死守，也缺乏像 Rivet 这样的内存级同机共生（Co-located State）。在面对密集的"多智能体高频长时交互（Long-running Swarm loops）"时，无状态拉起的进程摩擦力显得过于厚重。

#### ④ Dapr —— Sidecar 抽象层的诱惑与代价

- **架构本质**：CNCF 孵化项目，通过 Sidecar 模式为每个服务附加"构建块"（State Management、Pub/Sub、Service Invocation 等），以 RESTful/gRPC 接口屏蔽底层实现差异。切换 Redis → Kafka → NATS 不需要改业务代码。
- **技术痛点**：每个调用经过 Sidecar 的一跳，延迟叠加。抽象层过重——每个构建块是一层 REST API 包装，而各领域已有事实标准协议（Kafka 的协议、Postgres 的协议、S3 的协议），在标准之上再抽象一层是退化。Dapr 的核心价值是跨云迁移和厂商锁定规避，但在技术栈可控的系统中，直接使用原生协议始终优于抽象层。Aura 的做法是进程内集成，没有 Sidecar，没有额外跳数。


## 8. 单机起手与分布式演进

### 8.1 核心洞察：起手式 ≠ 终极数据宿命

在主程序里用一个轻量的内存状态管理组件起手，从来都不耽误、也不妨碍系统未来去连接和使用任何垂直领域的专业外部数据库。

正如在常规开发中，我们在主进程里手写一个 `Mutex<HashMap>` 或者拉起一个 Tokio 状态通道作为执行期的缓存和业务状态机，这不妨碍我们在需要持久化的时候，用一条单次事务连接（Single-Trip）把数据顺手写入 PostgreSQL、或者归档进 S3 的 LanceDB 里。

如果顺着这层"起手式不等于终极数据宿命"的最高务实哲学，来重新审视"任何项目直接以 Fjall 起手"的合理性，核心的技术分水岭就不再是"能不能用别的数据引擎"，而是你从第一天开始，往你的二进制文件里注入的**"架构心智负荷与锁定代价"**有多重。

### 8.2 三种起手模式对比表

| 维度 | Mutex<HashMap> | 外部共识起手 | **Tokio 摊位 + Fjall** ✨ |
|------|---------------|--------------|---------------------------|
| **编码摩擦** | 极低 | 极高（状态机抽象） | **低**（简单 KV 接口） |
| **断电保护** | ❌ 无 | ✅ WAL + RaftLog | **✅ WAL** |
| **Scale-to-Zero** | ❌ 无（10 万摊位撑爆内存） | ✅ LSM-Tree | **✅ LSM-Tree** |
| **持久化** | ❌ 无（重启后状态丢失） | ✅（元数据 Raft + 数据 Fjall/S3） | **✅ Fjall 磁盘** |
| **未来扩展性** | ✅ 无限 | ⚠️ 全域绑共识（联邦语义下是方向性倒退） | **✅ 无限** |
| **心智负荷** | 极低 | 中（共识依赖） | **低** |
| **推荐场景** | 原型验证 | 确定需要数据级强一致复制（直接用 TiKV 等） | **通用起手式** ✨ |

**模式一：Mutex<HashMap> 起手（零依赖体验）**——项目刚敲下第一行代码时，状态就是 Rust 原生类型，不需要写任何序列化宏。致命缺陷：无断电保护、无 Scale-to-Zero、无持久化。

**模式二：外部共识起手（元数据协调枷锁）**——引入共识（Openraft 或 etcd）做跨节点协调，即使用途收窄到元数据（摊位注册/路由/配置）、不复制摊位数据，仍为协调逻辑绑进 Raft 的共识依赖。本架构已明确**不引入共识**：元数据每节点独立（控制平面单写，`ctx.metadata` 只读面使写入者永远单一），多机走联邦形态——联邦内部没有通向共识的路径，多控制面部署是方向性倒退而非扩展点。共识只在「数据级强一致复制」真实出现时由外部现成方案（TiKV 等）承担。

**模式三：Tokio 摊位 + 单机 Fjall 起手（工程学的最高折中）✨**——用 Fjall 替代 HashMap 几乎没有增加编码摩擦，却带来了：✅ 本地 bare-metal 级别的断电崩溃物理保护（WAL）、✅ 闲时内存自动归零（Scale-to-Zero）、✅ 读写速度快到物理硬件的极限、✅ 布隆过滤器微秒级定位。如果项目做大了需要多机形态，由于已经是摊位 + Fjall 架构，直接演进到联邦模式（各节点独立 + well-known 协议认证，数据仍本地 + 落湖/S3），无需引入共识。**起手式不锁定终极宿命。**

### 8.3 起手式代码示例

#### Mutex<HashMap> 起手（零摩擦）

```rust
use std::collections::HashMap;
use std::sync::Mutex;
use tokio::sync::mpsc;

struct BoothState {
    data: Mutex<HashMap<String, Vec<u8>>>,
}

impl BoothState {
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

#### Tokio 摊位 + Fjall 起手（最佳折中）✨

```rust
use fjall::{Config, Keyspace, PartitionCreateOptions};
use std::sync::Arc;

struct BoothState {
    keyspace: Keyspace,
    partition: Arc<fjall::PartitionHandle>,
}

impl BoothState {
    fn new(path: &str) -> anyhow::Result<Self> {
        let keyspace = Config::new(path).open()?;
        let partition = keyspace.open_partition(
            "booth_state",
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

#### 元数据演进：每节点独立，联邦认证

```rust
// 多机形态：元数据每节点独立（数据面 okm 实例内的 meta 表，控制平面单写），数据直写本地 Fjall（后落湖）
// 跨节点不做共识——联邦节点间经 well-known 协议认证身份，用户数据跟随所属节点

struct DistributedBoothState {
    meta: Arc<MetaStore>,               // 元数据：摊位注册/路由（本节点实例）
    摊位: Arc<BoothState>,             // 复用上面的单机 Fjall 实现
    store: Option<ObjectStoreClient>,   // 数据兜底：落湖 / SlateDB+S3
}

impl DistributedBoothState {
    async fn upsert(&self, key: &str, value: Vec<u8>) -> anyhow::Result<()> {
        // 数据直写本地 Fjall；落湖 / S3 异步兜底——无共识写路径
        self.摊位.upsert(key, value).await?;
        Ok(())
    }

    async fn locate(&self, 摊位: &str) -> anyhow::Result<NodeId> {
        // 元数据本节点读（控制平面单写 + 节点缓存）
        self.meta.locate(摊位).await
    }
}
```

### 8.4 技术现实主义的胜利

状态管理框架只是工具，它不是禁锢业务宿命的牢笼。在项目的第一天，直接引入分布式网络共识这样厚重的逻辑，往往会因为过度设计（Over-engineering）而把早期的业务演进速度活生生拖垮。

但如果采用**"最轻量的 Tokio 摊位消息管道 + 单机内嵌 Fjall"**作为新项目的通用起手核心——这既提供了启动速度、内存安全和冬眠效率，又把面向所有外部数据库（Postgres, S3, LanceDB）进行后期大一统演进的大门敞开着。单机运行完全没有问题，它是这套架构走向工业化最稳健、最清醒、低开销的第一步。

---


## 9. 工业适用性诊断与通用场景

这套由 Fjall（本地存储）+ 独立元数据实例（单写无共识）+ 多模态嵌入式内核（Steel/PyO3/Wasm 用户自主驱动沙箱）铸造的纯 Rust 存算一体架构，在工业落地中具有极强的普适性。它不仅兼容网关、FaaS、任务平台和游戏场景，甚至能在这几个场景里引发架构颠覆。

### 9.1 四大核心场景的工程适用性诊断

#### API 网关场景：适合（实现"动态策略、零网络开销"）

传统的 API 网关（如 Kong 或 APISIX）在处理高级路由（如：动态限流、AB 测试分流、用户鉴权）时，通常必须频繁地去读取外部缓存，或者在 Nginx 内部塞满晦涩的 Lua 脚本。

**在本架构下**：每个 API 路由策略或每个租户都是集群里的一个分布式摊位。网关收到请求后，不需要经过任何网络跳转，直接在进程内存里、通过 Fjall 快速捞出该路由的最新规则，并由用户动态指定的 Steel Lisp 沙箱一秒执行。

**关键点**：由于元数据每节点独立（控制平面单写 + 节点缓存），你修改本节点网关规则后立即生效，且没有外部缓存单线程死锁或集群脑裂的风险。

#### FaaS 平台场景：互补（打通"有状态 Serverless"的死穴）

传统 FaaS（如 AWS Lambda 或 Vercel Functions）最致命的痛点是"无状态（Stateless）"。函数每次拉起都要重新去数据库连线、握手、查配置（冷启动长达数百毫秒），这在行业里催生了对类似 Cloudflare Durable Objects（有状态持久化对象）的强烈渴望。

**在本架构下**：你不需要像 Rivet 那样去捆绑重型臃肿的 V8 和 JS 生态。用户自己决定这一毫秒是用 Steel 还是 Python 来跑 FaaS。

**关键点**：它提供了Scale-to-Zero（闲时内存归零）与系统低开销启动后，摊位唤醒为微秒级（嵌入式 VM 瞬时创建）。函数不活动时，状态在 Fjall 磁盘中以紧凑的 SSTables 形式冬眠，不侵占 1 字节的 RAM；请求命中时，Fjall 瞬间把 Vec 二进制 blob 拍进内存，多语言引擎就地复活执行。

#### 任务编排平台（如 Windmill/Prefect）：超越（移除调度队列膨胀）

像 Windmill 这样的任务平台，其底层为了维护任务调度队列、并发锁和重试状态，必须高度依赖一个吞吐量极大的 PostgreSQL 或外部队列。当遇到百万级微型任务并发时，外部数据库连接池会瞬间枯竭。

**在本架构下**：每一个长时流式任务（Workflows）本身就是一个独立的摊位。任务的每一个 Step、每一步重试状态，都被原子化地写进本地物理 Fjall 引擎中。

**关键点**：它彻底消除了外部调度队列。任务状态就在计算引擎的贴身进程内存里，不需要跨进程 IPC，不需要外部锁。它比 Windmill 更轻、更快，且天然具备跨机房的断线防失忆防猝死（Fault-tolerance）能力。

#### 游戏服务器场景：它的老本行（游戏级高并发摊位的 Rust 回归）

Rivet Actors 为什么要用这一套设计？因为他们原本就是做多人联机游戏房间（Game Rooms）和玩家实时状态（Player Stateful）托管出身的。

**在本架构下**：你用纯 Rust 实现了比 Rivet 更纯粹、更干净的原生游戏运行时。

**关键点**：一个游戏房间就是一个摊位，玩家的所有走位、血量、装备变动（State Mutate），直接高频（每秒 60 次）缓存在该摊位的内存中，随写随落 Fjall WAL（同房间状态不跨节点复制，房间归属固定节点——强一致复制的诉求交给外部 TiKV 类方案）。游戏结束时，Fjall 异步执行一次大块物理刷盘。由于彻底踢出了 JS/V8 的垃圾回收（GC）开销，主程序可以跑到上千帧的平滑度，单机挂载数万玩家房间系统也绝不卡顿。

### 9.2 三个全新硬核应用场景

除了上述四个经典领域，这套 Fjall + 独立元数据 + 用户主导多语言沙箱的全栈架构，还能在以下三个涉及通用、前沿开发的环境中展现出适用性：

#### 分布式工业物联网与边缘计算（Edge AI & IoT Gateways）

在风力发电厂、车联网、无人机编排或自动化工厂机房中，硬件设备往往处于"弱网、低功耗、本地磁盘寸土寸金"的恶劣物理环境里。你不可能在边缘机房里塞进庞大的 K8s 集群或者 PostgreSQL 数据库。

**怎么玩**：把这个单文件二进制程序直接扔在边缘网关（如树莓派或工业单板机）上。由于 Fjall 的 LSM 结构极度抗断电损耗，边缘节点各自独立自治（元数据本地单写，联邦间 well-known 协议认证）。采集到高频传感器数据时，PyO3 Python 直接在本地内存中运行 Numpy 异常检测；发现危机时立刻切换到 Steel Lisp 运行确定性的关断策略脚本。微秒级响应，完全不需要连向云端。

#### 现代 Web 3.0 / 联盟链与可信去中心化账本（Consensus Ledgers）

传统的区块链底层（如以太坊节点）在执行智能合约时，架构笨重。需要全网共识的账本场景，共识层直接采用成熟方案（把多数派共识交给专门的区块链/账本引擎），合约逻辑写成 Steel Lisp 或 Wasm 字节码在本架构执行；用不到 2000 行的 Rust 核心，实现了一个响应速度突破上万 TPS、且具备跨国多机房灾备能力的超高性能专属去中心化状态账本。

#### 企业私有化"无头"AI 程序员 Agent 矩阵（Headless Coding Matrix）

当公司需要部署 1000 个无处不在、24/7 在后台自动审查 Git 代码、跑自动测试并修改 Bug 的"AI 程序员"时，每个 AI 程序员就是一个常驻或冬眠的分布式摊位。AI 需要去外网爬取 API 文档时，调用 Python（PyO3）异步网络库；当它要结合本地文件跑 PyTorch 权重或者语义分析时，调用 PyO3 Python；当它要执行危险的本地编译时，直接锁死在 Wasm 沙箱里防止它误删真实硬盘。所有的思考逻辑、进度、避坑备忘录，完全不需要依赖外部队列，直接持久化在本地 Fjall 数据库中。

### 9.3 架构师的核心优势

这套架构之所以能在这么多南辕北辙的通用、极限场景里同时展现出适用性，不是因为它的功能多，恰恰相反，是因为它把不该存在的中介全部移除了（The Best Part is No Part）。

它在物理层面上消灭了网线两端的序列化和反序列化，把多核 CPU、NVMe 物理磁盘和多语言虚拟机的内存空间直接贴合在同一个机位上。在通用开发环境里，这是一种简洁的设计取向。

---


## 10. 常规 Web 服务适用性

Aura 的核心场景是高并发有状态流（游戏、IM、秒杀、AI 智能体）。对于常规 Web 服务（CRUD、电商、企业管理系统），需要判断哪些子场景适合 Aura、哪些应该回退到传统关系型数据库。

### 10.1 数据映射

常规 Web 的业务数据按特性映射到 Aura 的双轨记忆：

| 业务数据类型 | 传统做法 | Aura 映射 |
|---|---|---|
| 高频状态事务（购物车、库存、Session） | Redis | 工作记忆（SlateDB+S3）：摊位进程内 KV，S3 自动复制，0 网络 RTT |
| 流水事件（订单历史、审计日志） | 日志表或 Kafka | Fjall WAL 本地落盘（存算分离下由 S3 承担长期安全） |
| 海量历史只读分析（报表、全文检索） | 读写分离、Elasticsearch | 长期记忆（LanceDB + S3）：窗口结单转列式 Lance，S3 托管 |

### 10.2 适合的子场景

- **高并发状态变动**（秒杀、抢单）：摊位进程内 MPSC 排队，单机承载传统架构需要多机加外部缓存才能扛住的并发量
- **无盘化弹性部署**：工作记忆 Scale-to-Zero，长期记忆托管 S3，节点可随时拉起或销毁

### 10.3 不适合的子场景

- **Ad-hoc 任意维度 SQL 查询**：热数据是 Fjall 的 KV blob，冷数据是 S3 列式切片，无法执行多表 JOIN + GROUP BY。注：[Arrow 大一统 HTAP 引擎](arrow-unified-htap-engine.md) 已部分解决此问题
- **后台管理 / ERP / CRM**：80% 工作是表格报表关联查询
- **团队门槛**：Bincode 序列化、嵌入式脚本沙箱对初级开发者不友好（共识已不在架构内，不再是门槛）

### 10.4 混合架构

常规 Web 服务同时包含"高并发核心模块"和"常规 CRUD"时，采用混合：
- CRUD 剥离给外部数据库（如需要关系型查询）
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
- **[Redis 批判](redis-critique.md)**：详细论证不用 Redis 的原因，§9.6 是摘要版。
- **[MySQL 批判](mysql-critique.md)**：MySQL 的 SQL 反模式与分片幻觉，本架构选择嵌入式 KV。
- **[SurrealDB](unified-data-layer.md)**：现代多模型数据库的替代方案，本架构可选择 SurrealDB 作为关系型补充。
- **[嵌入式脚本语言选型](embedded-script-languages.md)**：Rune/Steel/Koto 的技术对比，本架构坚定选择 Steel Lisp。
- **[编辑器选型](editor-selection-2026.md)**：Helix vs Neovim 的深度分析，本架构可在 Neovim 中使用 Fennel/Steel 元编程。
- **[Nginx 批判](nginx-critique.md)**：Nginx 的遗留设计，本架构可选择 Envoy/Pingora 作为现代网关。
- **[反应式架构](flux-architecture.md)**：Aura §5 场域模型是反应式架构的进程内实现——摊位写入 Fjall → emit 事件 → 订阅者进程内收到，零网络跳数，不依赖外部流处理引擎。
- **[Wasm 统一运行时](wasm-unified-runtime.md)**：Rust → Wasm 现在已可用（不需要 wasm-gc），覆盖异步场景。wasm-gc 后 Kotlin/C# 涌入，语言选择进一步丰富。Steel 和 PyO3 保留。

