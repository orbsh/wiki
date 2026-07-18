# AI Agent 选型

## 选型需求

寻找一个轻量 CLI Agent，定位介于 jcode（纯执行）和 Hermes（全功能但重）之间。核心要求：

**硬需求**：
- **单可执行文件**：Rust / Zig / Go 编译，无运行时依赖
- **Skill 为核心**：执行以技能（SKILL.md）为主，不是裸 prompt 对话
- **轻量**：Hermes 乱七八糟功能太多，Python 运行时有延迟

**可选**：
- 看板（Kanban 任务分解 + 多 Agent 调度）
- 自主进化（自动发现模式、生成/更新 skill）

**记忆可外挂**：不强求 Agent 内置强记忆。记忆层可以外挂（如 [agentmemory](https://github.com/rohitg00/agentmemory)），跨工具共享——这样换 Agent 不换记忆，复利不丢失。

Hermes 的问题不是能力不够，是**太重**——Python 解释器启动延迟、功能堆砌导致维护复杂度膨胀。需要一个"Hermes 的核心能力，但用编译语言实现"的方案。

> Agent 工具的价值不在单次对话质量，而在跨会话累积（参见 [Agent 复利](agent-compound-interest.md)）。没有持久化的 Agent 是打工；有持久化的 Agent 是投资。复利的关键是记忆层独立于 Agent——记忆外挂 = 换工具不丢积累。

---

## 一、候选方案

### 1. gocode（Go）— AlleyBo55/gocode

Claude Code 的 Go 重写。单二进制，零依赖。MCP server + CLI 工具编排 + session 管理。定位是 harness runtime，不是对话助手。

- **语言**：Go，单二进制
- **记忆**：session 管理（具体持久化能力待验证）
- **Skill**：未提及原生 skill 系统
- **看板**：无
- **自主进化**：无

### 2. gopher-code（Go）— ProjectBarks/gopher-code

Claude Code 的 Go 重写，513K 行 TypeScript → Go。12ms 冷启动。零 Node.js / Electron。跨平台交叉编译。

- **语言**：Go，单二进制
- **记忆**：待验证
- **Skill**：无
- **看板**：无
- **自主进化**：无

### 3. hermes-agent-rs（Rust）— Lumio-Research/hermes-agent-rs

Hermes Agent 的 Rust 重写。单二进制，10 个 LLM provider，30+ 工具，17 平台适配器。Zero 配置。

- **语言**：Rust，单二进制
- **记忆**：unified memory（具体架构待验证）
- **Skill**：30+ 工具但不是 skill 系统
- **看板**：无
- **自主进化**：声称 self-evolving，具体机制待验证

### 4. epic-harness（Rust）— epicsagas/epic-harness

多工具 AI Agent harness。22 个 skill，self-evolving engine，unified memory，autonomous spec-to-PR pipeline。兼容 Claude Code / Codex / Cursor / OpenCode / Cline。

- **语言**：Rust
- **记忆**：unified memory
- **Skill**：22 个 skill，self-evolving engine（自动进化）
- **看板**：无
- **自主进化**：✅ 声称支持

### 5. wintermolt（Zig）— lupin4/wintermolt

OpenClaw 的 Zig 重写。~5MB 单二进制，零 Node.js。Agentic loop、SSE streaming、工具分发、SQLite history、多后端（Claude / Ollama / OpenAI 兼容）。

- **语言**：Zig，~5MB 单二进制
- **记忆**：SQLite history
- **Skill**：无
- **看板**：无
- **自主进化**：无

### 6. AgenticGoKit（Go）— AgenticGoKit/AgenticGoKit

Go 的 Agentic AI 框架。流式 API、workflow 编排、工具集成、memory。定位是框架（库），不是开箱即用的 CLI Agent。

- **语言**：Go（库，需自己构建 CLI）
- **记忆**：内置 memory
- **Skill**：workflow 编排
- **看板**：无
- **自主进化**：无

### 7. godmode（Shell）— arbazkhan9710/godmode

126 个 skill，7 个 subagent，5 平台。自动回滚、failure memory、并行多 Agent 执行。是 Claude Code / Cursor / Codex 的插件，不是独立 Agent。

- **语言**：Shell（非编译语言，不满足单二进制要求）
- **记忆**：failure memory
- **Skill**：126 个 skill
- **看板**：无
- **自主进化**：部分（iterative optimization + automatic rollback）

---

## 二、对比矩阵

| 方案 | 语言 | 单二进制 | 记忆 | Skill | 看板 | 自主进化 | 成熟度 |
|------|------|---------|------|-------|------|---------|--------|
| **gocode** | Go | ✅ | session | ❌ | ❌ | ❌ | 低 |
| **gopher-code** | Go | ✅ | ? | ❌ | ❌ | ❌ | 低 |
| **hermes-agent-rs** | Rust | ✅ | unified | ❌ | ❌ | ⚠️ 声称 | 低（61★） |
| **epic-harness** | Rust | ✅ | unified | ✅ 22个 | ❌ | ✅ | 极低（10★） |
| **wintermolt** | Zig | ✅ 5MB | SQLite | ❌ | ❌ | ❌ | 低（39★） |
| **AgenticGoKit** | Go | 需自建 | ✅ | workflow | ❌ | ❌ | 框架级 |
| **godmode** | Shell | ❌ | failure | ✅ 126个 | ❌ | ⚠️ | 插件级 |
| **Hermes** | Python | ❌ | ✅ 三层 | ✅ | ✅ | ✅ | 成熟（90k★） |
| **jcode** | Rust | ✅ | SQLite | ❌ | ❌ | ❌ | 中（7k★） |

---

## 三、结论

**硬需求（单二进制 + skill + 轻量）满足的方案**：

1. **epic-harness**（Rust，10★）— 有 skill（22个）+ self-evolving engine + unified memory。硬需求全中，可选项也有自主进化。唯一短板是极度早期、无看板。**最接近需求**。
2. **gocode / gopher-code**（Go）— 单二进制 + 快（12ms），但无 skill 系统，只是 Claude Code 的行为对齐重写。
3. **wintermolt**（Zig，39★）— 5MB 单二进制 + SQLite history，但无 skill 系统。

**记忆外挂方案**：记忆不强不再是否决项。[agentmemory](https://github.com/rohitg00/agentmemory)（24k★，TypeScript）提供跨工具持久记忆层，可以给任何 CLI Agent 外挂。架构是：

```
CLI Agent（单二进制，skill 执行）
  ↕ MCP / API
agentmemory（持久记忆，跨工具共享）
  ↕
[可选] kanban-md（文件式看板，多 Agent 协调）
```

这样**记忆层独立于 Agent**——换 Agent 不丢积累，复利不断裂（参见 [Agent 复利](agent-compound-interest.md) 中"数据主权与复利"的论述：积累的知识必须归用户所有，记忆外挂 = 记忆归用户不归工具）。

**现实路径**：

- 如果**skill 为核心**是硬需求 → **epic-harness**（Rust）是唯一选择，接受早期风险，记忆外挂 agentmemory
- 如果**稳定性优先** → **gocode**（Go）+ agentmemory 外挂，接受无 skill，靠 SYSTEM.md/CLAUDE.md 手动管理上下文
- 如果**看板是硬需求** → 任何 Agent + kanban-md（文件式 Kanban，Markdown + YAML frontmatter）
- 如果**现在就要用** → jcode（Rust，成熟）+ agentmemory 外挂，接受无 skill 系统

> 编译语言生态的 Agent 普遍把精力放在"对齐 Claude Code 的行为"，忽略了 Hermes 走通的核心路径：skill → 自主进化 → 复利。但记忆可以外挂解决了最关键的复利问题——剩下的是 skill 系统。epic-harness 是目前唯一在 skill + self-evolving 方向上探索的编译语言 Agent，值得观察。

---

## 四、Claw Code 事件与净室重写

### 泄露事件（2026-03-31）

Anthropic 向 npm 发布 claude-code 2.1.88 时，意外包含未压缩的 `.map` 源码映射文件，泄露 512,000 行 TypeScript 源码（近 2,000 个内部文件），包含 44 个未公开的 Agent 调度特性、多智能体协同引擎、动态环境记忆算法。

### 净室设计（Clean-room Design）

**法务风险**：直接复制/修改泄露源码触犯 DMCA，会被 Anthropic 删库。

**技术洗白流程**：
1. **A 组**（阅读源码）：提炼系统运行逻辑、状态机蓝图、Prompt 结构，输出元数据说明书
2. **B 组**（未接触源码）：根据说明书用 Python 编写 1500 行架构验证代码
3. **Rust 重构**：为消除 Python 解释器延迟和内存开销，全面转向 Rust 重写（4000 行核心代码）

**结果**：Claw Code（ultraworkers/claw-code）创下 GitHub 最快突破 3 万/10 万星纪录，最终 194k Stars。

### 现状：智能体托管的博物馆

Claw Code 已转变为 Agent-managed 模式——后台由 Gajae-Code 等智能体自发读取 Issue、修改 Rust 代码、跑 CI、合并 PR。定位从"工业生产力工具"转为"AI 自主演进的学术温床"。

---

## 五、技术对比：Claw Code vs jcode

| 维度 | Claw Code (194k Star) | jcode (7k+ Star) |
|------|----------------------|------------------|
| **哲学** | 官方 Claude Code 的高仿与行为对齐 | 实用主义大杀器，干掉原版臃肿与延迟 |
| **架构** | Python（元数据）+ Rust（核心）混合 | 纯 Rust 静态编译（Single Static Binary） |
| **启动时间** | 数十毫秒（Python 解释器初始化拖累） | 14ms（几近原生 Shell 物理极限） |
| **内存** | ~120MB | 27.8MB（原版 Claude Code 的 1/14） |
| **运行模式** | 纯 CLI 交互式 | Server/Client 解耦（内置 1000+ FPS TUI） |
| **并发** | 线性单会话 | 多会话蜂群并发（Agent Swarm）+ 自动冲突调解 |
| **SDK** | 无原生 SDK，依赖 Shell 管道 | 跨平台 Bridge SDK（JSON-RPC 通信） |

**技术路线差异**：
- Claw Code → 行为对齐（Parity）→ 线性单兵作战
- jcode → 性能压榨（14ms）→ 高并发蜂群协作

---

## 六、多智能体框架演进

### 早期框架（LangChain/AutoGen）失败原因

**核心缺陷**：用人类的臃肿管理思维绑架大模型。

1. **树状主管地狱（Supervisor Loop）**
   - 强行设计"主管 Agent → 程序员 Agent → 测试员 Agent"阶级制度
   - 底层报错时，主管 Agent 陷入理解混乱，反思中引发无限套娃死循环

2. **上下文雪崩（Context Window Bloat）**
   - 智能体间通过"打包发送聊天历史"传递进度
   - 输入 Token 呈指数级暴涨，撑爆 Context 窗口，导致记忆幻觉

3. **厚重的抽象阻抗**
   - 用厚重的 Python 类强行规范大模型
   - 框架越重，模型原生工具调用（Native Tool Calling）灵活性越差

### 2026 新一代：环境驱动型 Swarm

**① 极简平权工具交接（Tool-based Handoff）**

消灭"主管"阶级。每个 Agent 平等且极度原子化（Agent A 只读文件，Agent B 只改文件）。A 完成后直接调用原子工具 `transfer_to_coder`，将控制权瞬间切给 B。消灭主管 = 消灭套娃死循环。

**② 环境感知同步机制（Ambient Context）**

智能体间几乎"互不聊天"，共享同一个物理文件系统（Git 仓库状态）。

工作流：
```
[Agent A] ──(修改文件/生成Diff)──► [本地物理文件系统]
                                      │
                                      ▼
                          [Rust事件总线派发 file_changed_under_you]
                                      │
                                      ▼
                        [Agent B 原地读取Diff，自动感知并调解]
```

靠物理代码库的真实状态（而非聊天历史）建立认知同步。

**③ Token 控制曲线突破**

Token 消耗从"指数级"暴跌为"近乎线性"：

- **局部视图（Local View）**：跑测试的 Agent 3 Context 里只有测试命令 + 报错日志，不加载业务源码。10 个 4k 窗口的局部 Agent 并发成本 << 1 个 100k 上下文的"胖 Agent"反复高空思考。

- **异步梦境整理（Ambient Consolidation）**：系统空闲或多 Agent 交接时，后台利用图论检索记忆（Memory Graph）静默清洗。将前 20 轮"修改-失败-看日志"的十几万字历史提炼为 100 字"避坑备忘录"重新注入 System Prompt。确保连续工作几天几夜，Token 计费曲线依然平稳。

---

## 七、工程落地要点

### NixOS 声明式部署

```nix
# modules/dev/units/jcode.nix
pkgs.rustPlatform.buildRustPackage rec {
  pname = "jcode";
  version = "2026.03";
  src = pkgs.fetchFromGitHub {
    owner = "1jehuang";
    repo = "jcode";
    rev = "master";
    hash = pkgs.lib.fakeHash;  # 首次构建时 Nix 报错给出真实 sha256
  };
  cargoHash = pkgs.lib.fakeHash;
  nativeBuildInputs = with pkgs; [ pkg-config ];
  buildInputs = with pkgs; [ openssl stdenv.cc.cc.lib ];
  doCheck = false;
}
```

### 核心记忆资产（CLAUDE.md / SYSTEM.md）

```markdown
# Project Rules
## Architecture Context
- OS Environment: NixOS
- Package Manager: Flakes Native

## Operational Constraints
- All system alterations MUST be declared in NixOS configuration.
- NEVER run generic `apt`, `yum` or standard python `pip` commands.

## Automated Memory Consolidation Hook
- After resolving conflict loops, compress diagnostic error tracks into memory store.
```

### 低额度启动

```bash
# 注入 DashScope 兼容端点凭证
export OPENAI_API_KEY="«redacted:sk-…»"
export OPENAI_BASE_URL="https://aliyuncs.com"

# 低推理额度控制
export JCODE_MAX_THINKING_TOKENS=1024

# Server/Client 模式
jcode server --port 8080 --workspace /mnt &
jcode client --connect 127.0.0.1:8080 --model qwen/glm-5.2
```

**断线重连**：终端黑屏掉线后，重新执行 `jcode client --connect 127.0.0.1:8080`，救援现场和 AI 整理的记忆图谱完美重现。

---

## 交叉引用

- **[Agent 复利](agent-compound-interest.md)**：Agent 工具的价值在跨会话累积，不在单次对话质量。复利的前提是状态持久化——memory、skill、自动化、历史检索。当前编译语言生态的 Agent 普遍缺乏复利基础设施。
- **[MCP 批判](mcp-critique.md)**：MCP 的 JSON-RPC 常驻进程是过度工程，CLI + Skill 路线更符合 UNIX 哲学。记忆外挂不需要绑定 MCP。
- **[agentmemory](agent-memory.md)**：记忆层外挂的具体方案——TS 外壳 + Rust 内核（iii-engine）+ 本地 embedding + SQLite，跨 Agent 共享记忆。
- **[LLM 基础认知](llm-fundamentals.md)**：Agent 的记忆和自主进化受限于 LLM 的元认知缺失——无法自我校验输出质量，需要 Maker/Checker 分离。
- **[Karpathy 的 AI 编程方法论](karpathy-ai-coding-methodology.md)**：文档驱动（SYSTEM.md / TODO.md）是对抗 LLM 无状态性的工程适配。
- **[循环工程分析](loop-engineering-analysis.md)**：Agent 的自动化（cron/loop）是复利的执行层，但验证天花板限制了适用范围。
- **[终端 Agent 批判](terminal-agent-critique.md)**：TUI 视觉化、性能营销、工程壁垒夸大的系统性批判——本选型的哲学框架来源。
