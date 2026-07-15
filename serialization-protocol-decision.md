# 序列化协议抉择：IDL vs Code-First

**状态**：架构决策完成  
**日期**：2026-06-20  
**核心哲学**：技术不应该是束缚开发者手脚的繁文缛节

---

## 概述

在分布式系统设计中，序列化协议的选择直接影响开发体验、跨语言兼容性、向后兼容性和性能。本文档深入分析三种主流方案的优劣，并提供清晰的决策框架。

→ 本决策影响 [Aura 架构](aura-architecture.md) 的 Raft 控制流和 [Arrow 大一统 HTAP 引擎](arrow-unified-htap-engine.md) 的数据存储层。

---

## 一、核心痛点：PB 的 IDL 与 Rust Code-First 哲学断层

Protocol Buffers (PB) 乃至绝大多数传统工业级通信协议在 Rust 生态中存在最大的"断层"和"痛点"——即 **IDL（接口定义语言）驱动的声明方式**，与 Rust 强烈的、以可导出的结构体（Struct / Enum with Derive Macros）为主导的**代码内聚哲学（Code-First）**，存在着天然的"代沟"与不兼容。

在传统微服务中，PB 强迫你必须：
1. 先写一个独立的 `.proto` 文件
2. 配置晦涩的 `build.rs` 插件（如 `prost-build` 或 `tonic-build`）
3. 在编译时去动态生成一堆不归你手写的、极其丑陋且难以在 Neovim 里直接跳转、重构的外部 Rust 源码

这种设计直接砸碎了 Rust 程序员最引以为傲的**"编译期强类型内聚"**的视觉爽感。

---

## 二、三种方案深度对账

### 方案 A：prost-derive 实现"纯 Rust 声明式（Code-First PB）"

谁说写 PB 必须先写 `.proto`？借助 Rust 强大的过程宏（Procedural Macros）和 prost 生态，你完全可以直接用 Rust 原生语法去定义你的分布式命令状态机，并在编译期由宏直接将其自动翻译映射为符合 PB 标准的二进制流，从而彻底干掉 IDL。

#### 纯正 Rust 极客流：0 外部文件、100% 声明式 PB 状态机

```rust
// src/store.rs
use serde::{Serialize, Deserialize};

// 🔥 核心大招：利用 prost 过程宏，在 Rust 结构体上直接标记 PB 字段 ID
// 彻底蒸发 .proto 文件和 build.rs！你的代码就是唯一的真实源（Single Source of Truth）

#[derive(Clone, PartialEq, ::prost::Message, Serialize, Deserialize)]
pub struct UpdateActorStateCommand {
    // 标记为 PB 字段 ID 1，编译期自动执行 Varint 高压缩
    #[prost(string, tag = "1")]
    pub agent_id: String,
    
    // 标记为 PB 字段 ID 2，存放用户最高意志驱动的嵌入式多语言 Arrow 字节流 Payload
    #[prost(bytes = "vec", tag = "2")]
    pub arrow_payload: Vec<u8>,
}

#[derive(Clone, PartialEq, ::prost::Message, Serialize, Deserialize)]
pub struct TerminateActorCommand {
    #[prost(string, tag = "1")]
    pub agent_id: String,
}

// 通过 Rust 的原生 Enum 映射为 PB 的 OneOf 联合体
#[derive(Clone, PartialEq, ::prost::Oneof, Serialize, Deserialize)]
pub enum RaftCommandPayload {
    #[prost(message, tag = "1")]
    Update(UpdateActorStateCommand),
    
    #[prost(message, tag = "2")]
    Terminate(TerminateActorCommand),
}

#[derive(Clone, PartialEq, ::prost::Message, Serialize, Deserialize)]
pub struct AuraRaftCommand {
    #[prost(uint64, tag = "1")]
    pub term: u64,
    
    #[prost(oneof = "RaftCommandPayload", tags = "2, 3")]
    pub payload: ::core::option::Option<RaftCommandPayload>,
}
```

#### Code-First 方案的降维打击

1. **Neovim 的全绿灯跳转**：因为所有的状态结构体都是原生的 Rust 代码，你在 Neovim 里可以通过最熟悉的 `gd` (Go to Definition) 自由重构、随时加减字段、无缝获得 LSP 智能提示。没有多余的外部文件折磨你的眼睛。

2. **多重格式大一统**：我们在结构体上同时挂载了 `::prost::Message`（负责高并发的集群间 Raft 选票与控制网络传输）和 `Serialize/Deserialize`（负责将数据极其干净地作为 Bincode/MsgPack 锁进 Fjall 物理磁盘分区）。数据在两套系统之间流转，不需要任何手动转义或拷贝函数（Mapping Functions），类型在内核空间里绝对安全地天然一致。

### 方案 B：纯 Rust 环境下的 Bincode/Postcard

PB（Protocol Buffers）的核心价值在于**"跨语言生态的绝对兼容"**（例如你的主集群是用 Rust 写的，但明天需要允许团队用 Go/Java 编写的微服务节点接入共识网）。

如果您在推理和决策中已经认定："我这个全新的 Web 服务/Agent 集群，从前端网关到后端分布式节点，100% 都是纯正的 Rust 单体代码"：

- **立刻停止在 PB 上雕花**。果断一票否决所有 PB 组件。
- **彻底倒向 Bincode**（或专为极简紧凑设计的 Postcard 格式）。

因为在纯 Rust 环境下，Bincode 根本不需要任何 IDL，不需要任何宏标记，也不包含任何字段 ID 冗余。它能根据 Rust 结构体本身的空间排布（Memory Layout），在编译期直接生成硬件级的纯原始字节编解码代码。它的元数据开销是绝对的 **0%**（比带有 Tag ID 的 PB 还要轻量、还要快，它才是真正的裸金属肌肉）。

#### Fluxora 实际踩坑：Bincode 的致命缺陷

Bincode 的"裸金属肌肉"在 Fluxora（AI-native UI 框架）项目中被实际验证为不可用。[→ 详见 ADR-001](projects/fluxora-decisions.md)

**根因**：Bincode **无法反序列化 `#[serde(tag = "...")]` 枚举**——它调用 `deserialize_any`，而 bincode 不支持。这影响 `Brick`/`Content` 类型的所有反序列化，不仅是线路协议。

**附带致命伤**：
- serde 2.x 不兼容（v1.x 损坏，v2.x API 不稳定）
- 无类型自描述（Gateway 无法部分解析路由元数据）
- 无跨语言支持（bincode 无 JS 库）

**结论**：Bincode 的零元数据设计是一把双刃剑——它在纯 Rust 内部 RPC 场景下确实极致轻量，但一旦涉及 serde 高级特性（内部标记枚举、`deserialize_any`）或跨语言需求，就会直接崩盘。Fluxora 最终迁移到 CBOR，Aura 则选择 Postcard（继承 serde 生态兼容性 + 自带 Schema 演进）。

### 方案 C：Apache Avro 的 Schema Evolution

Avro 最恐怖、也是最迷人的物理特性——数据在通过网线传输或写进物理日志时，里面**没有包含任何字段名**（不像 CBOR），**甚至连字段 ID 都没有**（不像 PB/Thrift）！

它是怎么做到的？

Avro 在序列化时，会严格按照你用 JSON 定义的 Schema 顺序，将数据一个接一个紧挨着排列成纯粹的物理字节流。

```
PB 的排布：   [Tag ID + 类型] -> [值] （依然有一丁点 Tag 冗余）
Avro 的排布： [值 1] -> [值 2] -> [值 3] （纯粹到极致的肌肉数据）
```

#### 降维的宇宙级向后兼容性（Schema Evolution）

这是 Avro 称霸大数据管道（如 Kafka/流式计算）的绝对大杀器，也是它比 Bincode 强悍得多的地方。

**痛点**：写 Rust 时，如果你用 Bincode 存了数据，明天你的 Actor 升级了，在结构体里加了一个新字段，旧节点读到新数据会瞬间崩掉（Panic）。

**Avro 的解法**：在 Avro 中，只要新老节点手里各有一份对应的 JSON 说明书，Avro 的解码器（基于 `apache-avro` crate）会在内存里自动执行 **"Schema 校验与对齐（Schema Resolution）"**。

- 如果新节点发现老数据少了一个字段，它会自动用 JSON 里写好的 `default` 默认值填补
- 如果老节点读到了新数据，它会自动无视并瞬间"跳过"多出来的未知字节

整个分布式集群可以极其平滑地在不停止任何服务器的前提下，实现**在线无感滚动升级**。

#### 无缝的"多语言沙箱大一统"契约

因为它的说明书是标准的 JSON，这意味着您那套用户动态驱动的多语言引擎（Steel Lisp / PyO3 Python）可以在完全不修改任何底层 Rust 代码的前提下，天然、原生、百分之百看懂当前的通信契约！

- **Lisp 联动**：内嵌的 Steel Lisp 虚拟机可以直接用最擅长的 S-表达式去解析和动态组合这个 JSON 说明书
- **Python 联动**：Python 更是不需要任何额外的 C-Binding 编译，原生调用 `json.loads()` 就能完美接管

彻底实现了应用层、脚本层和共识网络层的跨语言零障碍沟通。

#### 冰冷的物理现实：为什么 Avro 不适合替代 Arrow？

虽然 Avro 听起来完美无瑕，但必须极其清醒地死守其物理边界：

- ✅ **它只适合用于**：Raft → Avro（接管多节点共识网络的信令与命令广播包）
- ❌ **它绝对不适合用于**：Fjall → Avro（不能用它来代替 Apache Arrow 跑热数据分析）

**致命短板（行式与列式的物理天堑）**：

Avro 无论多么精简，其本质依然是**行式存储（Row-oriented）**。数据在内存里是按照一个用户一个用户挨着放的。

一旦你进入到需要用 Polars 进行任意维度的即兴复杂多表 JOIN 和批量数据框合并的阶段，行式排布的 Avro 会瞬间败下阵来。因为多核 CPU 无法在 Avro 内存里激活 SIMD 硬件向量化并联加速。如果强行用 Avro 跑分析，Polars 被迫必须在运行期把数据整块拆解，速度会比原生 Arrow 列式对齐**慢上百倍**。

---

## 三、技术现实主义的终极裁判

### 三种方案对比表

| 决策维度 | 方案 A（Code-First PB） | 方案 B（Bincode） | 方案 C（Avro） |
|---------|------------------------|------------------|---------------|
| **跨语言兼容** | ✅ 支持 Go/Java/Python 接入 | ❌ 纯 Rust 生态 | ✅ 标准 JSON Schema |
| **IDL 文件** | ❌ 不需要（宏标记替代） | ❌ 不需要 | ✅ JSON 文件（标准化） |
| **编码摩擦** | 中（需学习 prost 宏） | **极低**（零配置） | 低（JSON Schema） |
| **压缩率** | 高（Varint + Tag） | **极高**（0 字节元数据） | 极高（0 Tag，纯数据） |
| **向后兼容** | ✅ 字段 ID 稳定即可 | ❌ 加字段即崩 | ✅ **Schema Evolution** |
| **Neovim 体验** | ✅ 全绿灯跳转 | ✅ 全绿灯跳转 | ✅ JSON 可读 |
| **适用场景** | 未来可能需要多语言接入 | **纯 Rust 极客帝国** | **需要平滑滚动升级** ✨ |

### 决策树

```
你的集群未来有死红线，必须支持跨语言（Go/Java）微服务通信？
├── 是 → 你的集群需要在线无感滚动升级（Schema Evolution）？
│        ├── 是 → 选用方案 C（Avro）
│        │        标准 JSON Schema，0 Tag 冗余
│        │        完美向后兼容，跨天平滑升级
│        │
│        └── 否 → 选用方案 A（Prost Code-First）
│                 用纯 Rust 过程宏给字段打上数字 Tag
│                 在 Neovim 里死守 100% 的内聚爽感
│
└── 否 → 你的系统就是纯正的、由您主导的 Rust 极客自愈帝国
         直接选择 Bincode
         享受 0 字节冗余、0% 元数据税、硬件级内存转铸的最高大一统高平滑体感
```

---

## 四、Avro Schema 示例

```json
{
  "type": "record",
  "name": "AuraRaftCommand",
  "namespace": "aura.raft",
  "fields": [
    {
      "name": "term",
      "type": "long"
    },
    {
      "name": "payload",
      "type": [
        "null",
        {
          "type": "record",
          "name": "UpdateActorStateCommand",
          "fields": [
            {
              "name": "agent_id",
              "type": "string"
            },
            {
              "name": "arrow_payload",
              "type": "bytes"
            }
          ]
        },
        {
          "type": "record",
          "name": "TerminateActorCommand",
          "fields": [
            {
              "name": "agent_id",
              "type": "string"
            }
          ]
        }
      ],
      "default": null
    }
  ]
}
```

---

## 五、Rust 代码示例

### Avro 序列化与反序列化

```rust
use apache_avro::{Schema, Reader, Writer};
use serde::{Serialize, Deserialize};

// Schema 从 JSON 文件加载
const AVRO_SCHEMA: &str = include_str!("../schemas/aura_raft_command.avsc");

#[derive(Debug, Clone, Serialize, Deserialize)]
pub struct UpdateActorStateCommand {
    pub agent_id: String,
    pub arrow_payload: Vec<u8>,
}

#[derive(Debug, Clone, Serialize, Deserialize)]
pub struct TerminateActorCommand {
    pub agent_id: String,
}

#[derive(Debug, Clone, Serialize, Deserialize)]
pub enum RaftCommandPayload {
    Update(UpdateActorStateCommand),
    Terminate(TerminateActorCommand),
}

#[derive(Debug, Clone, Serialize, Deserialize)]
pub struct AuraRaftCommand {
    pub term: u64,
    pub payload: Option<RaftCommandPayload>,
}

// 序列化
fn serialize_command(cmd: &AuraRaftCommand) -> anyhow::Result<Vec<u8>> {
    let schema = Schema::parse_str(AVRO_SCHEMA)?;
    let mut writer = Writer::new(&schema, Vec::new());
    writer.append_ser(cmd)?;
    writer.flush()?;
    Ok(writer.into_inner())
}

// 反序列化（自动 Schema Resolution）
fn deserialize_command(bytes: &[u8]) -> anyhow::Result<AuraRaftCommand> {
    let schema = Schema::parse_str(AVRO_SCHEMA)?;
    let mut reader = Reader::with_schema(&schema, bytes)?;
    
    // Avro 自动处理 Schema Evolution
    // 如果数据缺少字段，会用 default 值填补
    // 如果数据有多余字段，会自动跳过
    let cmd: AuraRaftCommand = reader.next().unwrap()?;
    Ok(cmd)
}
```

---

## 六、Aura 的最终抉择（2026-06-20 更新）

在 Aura 架构中，我们采用**分层序列化策略**：

| 层级 | 序列化格式 | 理由 |
|------|-----------|------|
| **Raft 控制流与 RPC** | Postcard | 纯 Rust 声明式、Varint 压缩、Postcard-Schema 宏支持 Schema 演进，彻底干掉 Bincode 加字段就崩的致命缺陷 |
| **本地事务工作记忆** | Arrow IPC | 列式对齐，Fjall 磁盘与 Polars 零拷贝对接 |
| **云端长期记忆 Lakehouse** | Lance | 内嵌向量与倒排索引，原生支持远程 S3 流式点杀检索，替代 Parquet 的 AI 时代列式标准 |
| **海量冷历史冬眠** | Parquet | 高压缩率冷存储，当数据进冷库时从 Lance 一键转为 Parquet 字典压缩 |
| **UI ↔ Gateway** | CBOR | 跨语言（WASM），自描述 |

### Postcard 替代 Bincode 的核心理由

Bincode 在纯 Rust 环境下虽然零元数据开销，但有两个致命缺陷：

1. **结构体加字段即崩**：分布式集群滚动升级时，新老节点的 Bincode 编解码格式不兼容，反序列化直接 Panic
2. **serde 高级特性不兼容**：Fluxora 实际踩坑——Bincode 无法反序列化 `#[serde(tag = "...")]` 枚举（调用 `deserialize_any` 而 bincode 不支持），导致所有内部标记枚举类型反序列化失败 [→ ADR-001](projects/fluxora-decisions.md)

Postcard 的解法与边界：
- `#[derive(Serialize, Deserialize)]` 挂载 `postcard-schema` 宏，编译期自动生成字段布局元数据
- 反序列化时根据 Rust 结构体实际变化，自动执行"安全跳过（Skip）"和"默认值补全"
- 新老节点不需要传递任何网络元数据，实现跨天无感滚动升级
- 底层 Zigzag/Varint 压缩，传输体积比 Avro 更轻

#### ⚠️ Postcard 同样不支持 `#[serde(tag = "...")]`

Postcard 和 Bincode 属于同一个流派（零自描述的极致轻量 Row 格式）。它的字节流里同样不包含字段名字符串和自描述 Key 标记，因此同样**不支持 `deserialize_any`**。强行把 `#[serde(tag = "type")]` 枚举喂给 Postcard 会抛出 `WontImplement` 错误。

**正解：改用 Externally Tagged 默认枚举模式**

```rust
// ✅ 正确：默认外部标签，变体用数字索引（0, 1, 2），不触发 deserialize_any
#[derive(Serialize, Deserialize, Debug, Clone, PartialEq)]
pub enum Content {
    Text(String),        // tag = 0
    Vector(Vec<f32>),    // tag = 1
    ArrowBlob(Vec<u8>),  // tag = 2
}

#[derive(Serialize, Deserialize, Debug, Clone, PartialEq)]
pub struct Brick {
    pub brick_id: String,
    pub timestamp: u64,
    pub payload: Content, // 确定性嵌套，零 ambiguity
}
```

**Gateway 延迟解包外壳**：网关层只需解析路由元数据，Payload 用 `&'a [u8]` 零拷贝转发给下游 Actor：

```rust
#[derive(Serialize, Deserialize)]
struct GatewayRouterEnvelope<'a> {
    pub route_target_id: String,   // 网关直接读取，执行路由
    pub auth_token: &'a str,       // 零拷贝借用
    pub brick_payload: &'a [u8],   // 完全不解包，直接转发
}
```

### Lance 替代 Parquet 的核心理由

Parquet 诞生于 Hadoop 时代，对现代 AI 工作负载有三个先天缺陷：
1. **向量索引缺失**：不支持 HNSW 向量索引，多维 Embedding 检索需要外部维护
2. **随机写入恐惧**：列式布局对流式追加写入极不友好
3. **无倒排文本**：全文检索需要额外的索引层

Lance 的解法：
- 内嵌 HNSW 向量索引 + 倒排文本索引，单一格式覆盖多模态检索
- 支持微秒级远程 S3 流式点杀（LanceDB 的底层魔法）
- 磁盘压缩率与 Parquet 持平，但 AI 工作负载性能碾压

**核心原则**：技术不应该是束缚开发者手脚的繁文缛节。看清物理硬件的边界，撕掉无谓的 IDL 套娃，才能让你的 Aura 引擎在多核 CPU 和 NVMe 之间爆发出最干净的能量。

---

## 交叉引用

本文档是序列化协议抉择的完整分析，与以下详细分析形成完整的决策闭环：

- **[Aura 架构](aura-architecture.md)**：存算一体的现代分布式 Actor 引擎，采用分层序列化策略。
- **[Arrow 大一统 HTAP 引擎](arrow-unified-htap-engine.md)**：Fjall + Arrow + Polars 全链路存算一体，含 7 种数据格式底层字节排布对撞。
- **[块级编辑器架构](block-editor-architecture.md)**：类 Notion 的 Block Editor，Yjs CRDT 协同 + Fjall 存储。

**统一的第一性原理**：不搞技术崇拜，不吃开源画的大饼，只看真实的硬件物理限制与团队生产力。
