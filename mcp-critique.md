# MCP：应用层协议的过度工程

**结论**：在拥有技术主权的纯终端环境中，拒绝 MCP，采用 CLI + Skill。
**状态**：用 Web 思维解决系统工程问题，在 UNIX 哲学面前是倒退。

## 执行摘要

Model Context Protocol（MCP）被 Anthropic 和各大商业 IDE（Cursor、Windsurf）包装成解决"AI 工具连接与动态发现"的终极方案。但从操作系统和分布式系统架构的本质来看，MCP 是一套在工程上极度肥胖的方案：它用应用层的 JSON-RPC 常驻进程，重写了操作系统进程管道就能解决的事情。

本文不讨论 MCP 的商业动机。焦点在技术：MCP 宣称的三大优势，在计算机科学原理面前全部站不住脚，CLI + Skill 路线是对齐且更优的。

---

## 一、技术栈路径依赖

当前主流 AI 客户端基于 Electron 和 Node.js 堆栈。这条技术路径的惯性导致了一个偏差：用 Web 思维解决系统工程问题。

Web 工程师的直觉是"起一个服务、跑 JSON-RPC 握手、常驻端口"——这在 Web 场景下合理，但在本地工具调用场景下是过度工程。操作系统的进程控制、标准信号量交互、管道和编程语言的 spawn API 已经完美解决了工具执行的所有需求。MCP 把这些原生能力用 JavaScript 在应用层重写了一遍，代价是：常驻进程的内存开销、握手延迟、Schema 全量加载的 Token 浪费。

---

## 二、MCP 的真实价值：生态标准化

MCP 有一个文档必须承认的正面价值：**标准化**。一个 MCP server 写一次，所有兼容客户端都能用。这是真实的生态价值——不是技术上的"更优"，而是集成成本上的"更低"。

但这不改变 MCP 在技术上过度工程的事实。标准化不等于唯一正确实现——USB 协议标准化了外设连接，但 USB-C 的 alternate mode 地狱证明标准化本身也可以是过度工程的产物。MCP 的标准化价值在**多客户端场景**成立，在**单一技术栈的纯终端环境**不成立——如果只有一个 Agent 框架，不需要跨客户端标准，CLI + Skill 的轻量优势立刻压倒 MCP 的生态优势。

---

## 三、三大技术伪优势

### 伪优势一：动态工具发现与 Schema 约束

**MCP 的主张**：工具通过 JSON-Schema 声明自身接口，Agent 动态发现并约束调用。

**问题**：大模型不可能凭空知道工具如何调用——Schema 声明逃不掉。MCP 协议规范本身支持 `tools/list` 和 `tools/call` 分离——Agent 可以先 list 再按需 call。但主流 MCP 客户端通常在初始化时全量拉取 Schema，冗余的 JSON-Schema 严重消耗 Token。这是生态惯性（客户端实现习惯），不是协议强制——但结果是：实际使用中 MCP 的 Schema 加载确实比按需检索浪费。

**CLI + Skill 对齐**：Skill 同样可以规定特定命令（如 `my-skill --schema`）毫秒级吐出强类型参数描述。AI 框架按需渐进式加载，只在需要时路由到对应 Skill 并获取 Schema，比 MCP 全量加载省得多。这是信息论意义上的优势：按需检索 > 全量广播。

### 伪优势二：有状态长连接避免并发文件锁

**MCP 的主张**：长连接的有状态进程避免多实例并发写文件的锁冲突。

**问题**：MCP 集成了并发安全，但集成的方式（内存 Lease 锁）不如直接用底层 DB。每个 IDE 窗口各自 fork 出独立的 MCP 实例，它们之间不共享内存。面对高并发时，照样得依赖底层数据库（如 SQLite WAL 模式、行级锁）的锁机制。

**CLI + Skill 对齐**：无状态 CLI + 有状态 DB。高并发安全彻底交给成熟的嵌入式数据库引擎（SQLite / LibSQL），不需要上层协议在内存里重复造锁。

### 伪优势三：异步长任务与流式传输

**MCP 的主张**：异步执行长任务，流式返回结果，不打断 Agent 主循环。

**问题**：让大模型发一条 `memory_save` 消息然后立刻继续输出，本质上就是后台异步执行。这不值得一个协议。

**CLI + Skill 对齐**：用编程语言的 spawn API 启动子进程，框架完全控制生命周期——可以 wait、可以 kill、可以读 stdout 流式返回。这是编程语言运行时的标准能力，不需要 MCP 来"施舍"。

### 补充：反向控制与中途交互

**MCP 的主张**：CLI 的单向 Standard I/O 无法在执行中途向用户请求交互（如确认、表单）。

**问题**：spawn 的子进程可以通过 stdout 向框架返回结构化消息（如 `{"action":"confirm","message":"..."}`），框架解析后弹窗给用户，用户选择再通过 stdin 写回。双向交互不需要常驻的 JSON-RPC 连接。

---

## 四、MCP vs CLI + Skill 对比

| 维度 | MCP | CLI + Skill |
|:---|:---|:---|
| **架构** | JSON-RPC 常驻进程 + 端口监听 | spawn 子进程，用完即退 |
| **启动延迟** | 握手 + Schema 全量拉取 | 毫秒级冷启动 |
| **Schema 加载** | 客户端通常全量拉取 | 渐进式（`--schema` 按需） |
| **Token 消耗** | 高（冗余 JSON-Schema） | 低（按需路由） |
| **并发安全** | 内存 Lease 锁（不如直接用 DB） | 嵌入式 DB（SQLite WAL） |
| **异步能力** | 协议内置 | spawn API（生命周期可控） |
| **依赖** | 主流实现依赖 Node.js + 端口 | 零依赖（单二进制 / Python 脚本） |
| **可迁移性** | 协议绑定，跨客户端需适配 | 直接复制目录 |
| **主权** | 协议绑定，工具在服务端 | 本地文件，工具主权在开发者 |
| **生态兼容** | ✅ 跨客户端标准 | ❌ 每框架自行实现 |

---

## 五、Skill 实现语言选型

Skill 的实现语言选择不是技术审美问题，是**自主进化场景的摩擦问题**——AI 需要读、写、改 Skill 代码，语言的选择直接影响摩擦大小。

| 语言 | 优势 | 劣势 | 适合场景 |
|:---|:---|:---|:---|
| **Python** | AI 极度熟悉，环境普及，读写改摩擦最低 | 性能一般 | **首选**——Skill 的瓶颈不在执行性能 |
| **Rust** | 性能最好，安全 | 环境重，编译慢，二进制不利 AI 审查，自主进化摩擦大 | 对性能敏感的核心 Skill |
| **Node / Bun** | 生态大 | 包碎，npm 下载量大，部署麻烦，优势不大 | 不推荐 |
| **Go** | 编译快 | 性能对 Skill 场景无优势，可靠性不如 Rust | 不推荐 |
| **Bash** | — | 全硬伤（无类型、无错误处理、注入风险） | **禁止** |

**结论**：Skill 首选 Python。理由不是"Python 最好"，而是"Python 在自主进化场景摩擦最小"——AI 写 Python 的准确率最高，审查 Python 代码的能力最强，Python 环境在服务器和开发机上最普及。Rust 性能更好但自主进化需要 AI 反复读写改代码，编译周期和二进制不透明性在这个场景下是劣势。

> 这个选型逻辑和 [AI Agent 选型](ai-agent-selection.md) 一致：Agent 的瓶颈不在执行性能，在复利积累。Skill 用 Python 不是因为快，是因为 AI 改起来快。

---

## 六、安全设计

### 框架层安全：框架拼命令

框架收到 LLM 的结构化输出（工具名 + 参数列表）后，用框架自身语言的 spawn API 构造命令——LLM 不直接拼命令字符串，框架控制执行。这与 MCP 的安全模型没有本质区别——MCP 也是 LLM 提供参数、框架/Server 执行。注入风险在两条路线上相同。

### Skill 层安全：供应链问题

Skill 启动后，用的是 Skill 自身所用语言的全部能力——Python 的 `subprocess`、Rust 的 `std::process`，包括 spawn 后台进程、读写文件、网络请求。Skill 本身的安全性是**供应链问题**（你信任这个 Skill 吗？它的来源可靠吗？），和 MCP Server 的信任问题完全一样。MCP 没有在安全层提供任何额外保障。

---

## 七、选型落脚点

**必须在商业 IDE 里工作**（Cursor / Windsurf）：不得不吃 MCP，因为这些 IDE 不开放 Shell 权限。此时跨客户端共享记忆的最成熟生态是 [agentmemory](https://github.com/rohitg00/agentmemory)（24k★，TypeScript，通过 MCP 协议适配商业 IDE）。这是政治性妥协——接受 MCP 的开销换取生态兼容。

**拥有技术主权的纯终端环境**（Claude Code / 独立 CLI Agent）：坚守 CLI + Skill。工具和记忆以 Python 脚本和明文 / LibSQL 的形式沉淀在项目目录（`.agent_skills/`）中，随 Git 走，具备可追溯性与环境隔离性。迁移时直接复制目录。记忆层可外挂（参见 [AI Agent 选型](ai-agent-selection.md) 中记忆外挂架构），不需要绑定 MCP。

---

## 交叉引用

- **[AI Agent 选型](ai-agent-selection.md)**：CLI Agent + 外挂记忆的分层架构，记忆层独立于 Agent。Skill 用 Python 不是因为快，是因为 AI 改起来快。
- **[Agent 复利](agent-compound-interest.md)**：工具主权在本地 = 复利归用户；协议绑定 = 平台锁定。Skill 直接复制目录 = 零迁移成本 = 复利不断裂。
- **[agentmemory](agent-memory.md)**：记忆层走 MCP 的妥协方案，低频操作开销可接受。
- **[Karpathy 的 AI 编程方法论](karpathy-ai-coding-methodology.md)**：SYSTEM.md / TODO.md 是 Skill 的雏形，文档驱动是对抗 LLM 无状态性的工程适配。
