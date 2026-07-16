# 缓存树与缓存藤蔓

## 一、缓存树（Cache Tree）

多个 turn 共享同一段 KV cache 前缀，形成树状结构：

```
                    [system prompt + 历史对话]          ← trunk（共享 cache）
                   /                                  \
    [branch A: 问题A]    [branch B: 问题B]    [branch C: 问题C]   ← branches（新计算）
```

核心特性：
- **共享 trunk**：所有 branch 复用父节点的 KV cache，不重复计算
- **独立 branch**：每个 branch 的新 token 独立计算，互不干扰
- **缓存复用**：切换 branch 时，common prefix 的缓存仍然可用（TTL 窗口内）

典型场景：IM 平台的 thread 机制。每个 thread 是一个 branch，共享主线程的前缀缓存。

```
主线程（trunk）
  ├── thread_A：修 bug      → 继承 trunk cache + bug 描述
  ├── thread_B：加功能      → 继承 trunk cache + 需求描述
  └── thread_C：写测试      → 继承 trunk cache + 测试范围
```

每个 thread 的成本 ≈ 任务描述的 token（项目上下文走 cache）。

---

## 二、缓存藤蔓（Cache Vine）

缓存树的变种。一条主干持续生长，每隔一段长出一片叶子，叶子凋落后主干继续。

```
[system prompt + 历史对话] ─── trunk（持续积累）
       │
       ├── [尾提示词A + 问题]  ← 叶子（一次 turn，凋落）
       │
    trunk 继续生长...
       │
       ├── [尾提示词B + 问题]  ← 叶子（一次 turn，凋落）
       │
    trunk 继续生长...
       │
       └── [尾提示词C + 问题]  ← 叶子（一次 turn，凋落）
```

与缓存树的区别：

| 维度 | 缓存树 | 缓存藤蔓 |
|:--|:--|:--|
| 主干 | 共享 trunk | 同，但主干持续积累 |
| 分支 | 持久 thread | 临时叶子（一次 turn） |
| 生命周期 | 跨 turn，用户驱动 | 单 turn，系统注入 |
| 控制方 | 用户选择 branch | 系统决定何时长叶 |
| 典型用途 | 多话题并行 | 后台任务（压缩、提取、审查） |

### 尾提示词 = 藤蔓的叶子

尾提示词利用藤蔓的特性：共享 trunk（历史 cache），只计算新叶子（尾提示词）。

```
KV cache（不变）：[历史对话]                          ← trunk
旁路分支（新计算）：[尾提示词 + 用户问题]              ← leaf
  → LLM 一次 turn 同时完成：回答用户 + 执行尾提示词指定的任务
  → turn 结束后，叶子凋落（尾提示词丢弃），只保留结果
```

类比 TCO（尾调用优化）：尾调用复用当前栈帧，尾提示词复用当前 KV cache。都是"尾部"操作，都为了复用已有状态。

### 核心特性

- **利用 cache**：历史部分走 KV cache，只有尾提示词是新计算
- **旁路分支**：不修改主干（历史对话），只在末尾追加临时指令
- **一次性**：turn 结束后从 prompt 中移除，不写入 session
- **控制权**：注入方决定何时注入、注入什么，Agent 只负责执行

### 压缩场景：叶子凋落时原地替换

一般尾提示词是旁路分支（追加临时指令，历史不变）。压缩场景不同——叶子凋落时，前面的历史也被替换（checkpoint 替换旧历史）。不需要分支（连之前的旁路分支都不需要），只需要在同一次 turn 内完成替换。

```
KV cache 中（不变）：
  [ckpt_0] + [msg_101..msg_150]

新追加的消息（唯一未缓存的部分）：
  尾提示词："先回答用户的问题，然后分析以上对话，调用 memory_store
            提取偏好/事实，标记 checkpoint。"
  用户消息："Fluxora 的组件集为什么是封闭的"

Agent 一次 turn 完成三件事（注意顺序）：
  1. "Fluxora 组件集封闭是因为..."          ← 先回答用户问题
  2. tool_call: memory_store(...)          ← 再做记忆操作
  3. tool_call: memory_checkpoint(...)     ← 最后标记 checkpoint
```

turn 结束后，session 中存储的是纯净历史：

```
session 中存储的：
  [checkpoint_1] + ["Fluxora 的组件集为什么是封闭的"] + ["Fluxora 组件集封闭是因为..."]

而不是：
  [checkpoint_1] + [尾提示词 + "Fluxora 的组件集为什么是封闭的"] + ["Fluxora 组件集封闭是因为..."]
```

尾提示词已丢弃，不残留。

### 与传统方案的对比

| 维度 | 传统压缩 | 尾提示词 |
|:--|:--|:--|
| LLM 调用 | 独立 API 调用 | 复用 Agent 正常 turn |
| KV cache | 从零计算（命中率 0%） | 历史部分走 cache（命中率 ~99%） |
| 注意力 | 全部集中在压缩任务 | 分散（回答用户 + 执行任务） |
| 控制权 | 压缩器控制 | 注入方控制（通过 prompt + 工具） |

## 参考

- [agent-memory.md](agent-memory.md) — Prefix Checkpoint 中的尾提示词应用
- [graph-memory.md](graph-memory.md) — 图谱化记忆中的提取 prompt
- [LLM 缓存破坏模式](llm-caching-destruction-patterns.md) — 缓存分支模式（thread）与破坏模式对比
