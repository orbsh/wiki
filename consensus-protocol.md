# 共识协议：Raft 的本质、边界与正确用途

> **语言：** [English](consensus-protocol-en.md)（主文档）· [中文](consensus-protocol.md)

**Status:** 初始版本
**核心论点：** Raft 是元数据共识协议（metadata consensus），不是数据存储协议。将其用于批量数据存储是范畴错误。
**架构上下文：** [KV 存储引擎](kv-storage-engine.md) — 双引擎模式中 Raft 的角色定位
**序列化关联：** [序列化协议分析对比](serialization-protocol-comparison.md) — Raft 控制流的编码选择

---

## Raft 的核心机制

Raft 是 Diego Ongaro 和 John Ousterhout 在 2014 年提出的共识算法（consensus algorithm），目标是可理解性（understandability）——相比 Paxos，Raft 将共识问题分解为三个独立子问题，每个子问题有明确的解。

### Leader Election（领导者选举）

集群中的节点有三种角色：**Leader**、**Follower**、**Candidate**。

- 正常状态下只有一个 Leader，其余为 Follower
- Leader 定期向 Follower 发送心跳（heartbeat），维持权威
- Follower 在超时时间内未收到心跳 → 转为 Candidate → 发起选举
- Candidate 向所有节点请求投票（RequestVote RPC）
- 获得多数派（majority）投票 → 成为新 Leader

**关键约束**：每个 term（任期）至多一个 Leader。term 是逻辑时钟，递增编号，用于检测过期信息。

### Log Replication（日志复制）

Leader 接收客户端写请求 → 追加到本地日志 → 通过 AppendEntries RPC 复制到 Follower → 多数派确认后 **commit** → 应用到状态机。

```
Client → Leader: put("key", "value")
Leader: 本地日志追加 → AppendEntries RPC → Follower 1, Follower 2
Follower 1: 写入本地日志 → ACK
Follower 2: 写入本地日志 → ACK
Leader: 收到 2/3 ACK → commit → apply 到状态机 → 回复 Client
```

**日志匹配原则（Log Matching Property）**：如果两个节点在某个 index 有相同 term，则该 index 之前的所有条目也相同。这是 Raft 正确性的数学基础。

### Majority Confirmation（多数派确认）

Raft 的安全性依赖一个简单数学事实：N 个节点的集群，任意两个多数派必有交集。这意味着：

- 一个 term 内不可能有两个 Leader（两个 Leader 必须各自获得多数派投票，而多数派必有重叠节点，该节点不会投两票）
- 已 commit 的日志不会被覆盖（新 Leader 必然包含所有已 commit 日志，因为它获得了多数派，而 commit 需要多数派确认）

**容错能力**：N 个节点容忍 ⌊(N-1)/2⌋ 个故障。3 节点容忍 1 故障，5 节点容忍 2 故障。

---

## Raft 适合什么：元数据共识

Raft 的本质是**元数据共识协议**——为少量、关键的决策数据提供强一致性保证。

### etcd / Kubernetes：Raft 的典型主场

etcd 是 Kubernetes 的唯一数据存储后端，使用 Raft 保证元数据一致性。典型数据规模：

| 维度 | 典型值 |
|:---|:---|
| 数据总量 | 数十 MB 到数百 MB |
| 单条记录 | 数百字节到数千字节（Pod、Service、ConfigMap） |
| 写入频率 | 数十到数百 QPS（集群规模相关） |
| 读取频率 | 数千 QPS（所有 kubelet/watcher 轮询） |
| 节点数 | 3 或 5（etcd 官方推荐） |

etcd 存储的是**数据关于数据的描述**（metadata about data）：哪些 Pod 运行在哪个节点、Service 的端点映射、ConfigMap 的配置内容。这些数据量小、变更频率低、但必须高度一致——一个过期的 Service 端点可能导致请求打到已销毁的 Pod。

### 分布式锁与 Leader Election

Raft 的 leader election 机制天然支持分布式锁：

- Leader 本身就是"锁持有者"——集群中只有一个 Leader
- etcd 的 `lease` + `put` 实现分布式锁：创建 lease → 绑定 key → Leader 故障时 lease 过期 → 其他节点竞争
- 用途：分布式任务调度、Leader 选举、分布式协调

### 流式元数据（Streaming Metadata）

Kafka 使用类 Raft 协议（KRaft，基于 Raft 的变体）管理分区元数据：

- 哪些分区存在、Leader 在哪个 Broker
- 消费者组的 offset 信息
- Topic 配置和分区分配

同样是"数据关于数据的描述"——Kafka 的实际消息数据（GB 级到 TB 级）存储在 Broker 本地磁盘，不经过 Raft。Raft 只管理"哪些数据在哪个位置"这一层元数据。

### 共识协议的通用特征

| 适用场景 | 核心需求 | Raft 如何满足 |
|:---|:---|:---|
| 配置管理 | 所有节点看到同一配置 | Log replication + majority commit |
| 服务发现 | 节点状态一致 | Leader 维护成员列表，心跳检测 |
| 分布式锁 | 互斥访问 | Leader election 天然互斥 |
| 分布式事务协调 | 决策一致 | Two-phase commit 的 coordinator 用 Raft 选主 |

---

## Raft 不适合什么：批量数据存储

### 写放大问题（Write Amplification）

Raft 的每次写入需要复制到 N 个节点。存储成本随节点数线性增长：

```
Raft 3 节点集群写入 1 GB 数据：
  节点 1: 1 GB（Leader 本地写入）
  节点 2: 1 GB（Follower 复制）
  节点 3: 1 GB（Follower 复制）
  总存储: 3 GB（3x 写放大）
  总网络传输: 2 GB（Leader → 2 Follower）

Raft 5 节点集群写入 1 GB 数据：
  总存储: 5 GB（5x 写放大）
  总网络传输: 4 GB（Leader → 4 Follower）
```

**这不是理论问题，是物理现实**。每多一个节点，存储成本和网络带宽消耗就多一倍。对于元数据（MB 级），这可以接受；对于用户数据（GB 到 TB 级），这是灾难。

### Raft 复制 vs S3 复制：成本对比

S3（以及所有主流对象存储）在后台自动进行 3 副本复制，存储成本已包含在价格中：

| 维度 | Raft 3 节点 | S3（3 AZ 复制） |
|:---|:---|:---|
| 副本数 | 3（可配更多） | 3（固定，跨 AZ） |
| 复制成本 | 显式：每节点一台服务器 | 隐式：包含在存储价格中 |
| 客户端增加时 | 副本数不变（N 固定） | 副本数不变（3 AZ 固定） |
| 每 GB 月价 | 服务器成本分摊（远高于 $0.023） | ~$0.023（us-east-1） |
| 扩展方式 | 加节点 = 加存储成本 | 不需要操作，自动处理 |
| 一致性 | 强一致（linearizable read） | 最终一致（但 S3 提供 read-after-write） |

**核心区别**：Raft 的复制成本随集群规模线性增长（每个新节点 = 新的完整副本）；S3 的复制成本固定在 3 副本，与客户端数量无关。Raft 是"我管我自己的复制"，S3 是"你用我就行，复制我管"。

### 用 Raft 存数据 = 给每个节点买一台服务器

假设集群有 3 个 Raft 节点，每个节点需要：

- 与数据总量等额的存储空间（每个节点存完整副本）
- 足够的网络带宽处理复制流量
- 足够的 CPU 处理 consensus 开销

当数据量增长到 1 TB 时：
- Raft：3 台服务器 × 1 TB 存储 = 3 TB 总存储成本
- S3：1 TB 存储 × $0.023/GB/月 = ~$23/月（含 3 副本复制）

**结论**：Raft 的存储成本是 S3 的数十到数百倍，这不是工程优化能解决的——是架构选择的必然结果。

### Scale 特征：为什么 3-5 节点是上限

| 节点数 | 容错 | 写延迟 | 适用场景 |
|:---|:---|:---|:---|
| 3 | 1 节点故障 | 1 RTT（Leader → 2 Follower） | etcd/K8s、小型协调服务 |
| 5 | 2 节点故障 | 1 RTT（Leader → 4 Follower） | 关键元数据、高可用要求 |
| 7 | 3 节点故障 | 1 RTT（但网络开销显著） | 理论可行，实践中罕见 |
| >7 | 更多容错 | 写放大严重，延迟上升 | **不推荐** |

**为什么不推荐 >7 节点**：

1. **写放大**：7 节点意味着每次写入复制 6 次，网络带宽消耗巨大
2. **延迟上升**：虽然 commit 只需多数派确认，但日志复制的网络开销随节点数增长
3. **运维复杂度**：7 台服务器的协调、监控、故障恢复远比 3 台复杂
4. **收益递减**：从 5 节点到 7 节点只多容忍 1 个故障，但成本增加 40%

---

## Raft vs 其他共识协议

### Raft vs Paxos

| 维度 | Raft | Paxos |
|:---|:---|:---|
| 可理解性 | 高（三个独立子问题） | 低（证明复杂，直觉反常） |
| Leader 依赖 | 强 Leader 模型 | 无 Leader（对称） |
| 日志复制 | 严格有序 | 可乱序（Multi-Paxos） |
| 工业实现 | etcd、TiKV、CockroachDB | Google Chubby、ZooKeeper（ZAB 变体） |
| 理论完备性 | 与 Paxos 等价 | 更通用（容忍更复杂的故障模型） |

**Paxos 的真实优势**：Paxos 不要求 Leader——理论上可以在任何节点并发提案。这在极端场景下（如 Leader 频繁故障）有优势。但实践中 Leader 模型更简单，Raft 的 Leader Election 已经处理了 Leader 故障场景。

**Paxos 的真实劣势**：Multi-Paxos 的工程实现极其复杂，Google 内部的 Spanner 团队花了数年才稳定。etcd 团队选择 Raft 的核心原因是"能招到能读懂代码的工程师"。

### Raft vs EPaxos

EPaxos（Egalitarian Paxos）允许无 Leader 的并发提案，理论上吞吐量更高：

| 维度 | Raft | EPaxos |
|:---|:---|:---|
| 并发写入 | 串行（Leader 序列化） | 并发（任意节点提案） |
| 吞吐量 | 受 Leader 瓶颈限制 | 理论更高（无 Leader 瓶颈） |
| 实现复杂度 | 中等 | 极高 |
| 工业采用 | 广泛（etcd、TiKV） | 几乎无（仅学术原型） |

**现实**：EPaxos 在论文中表现优异，但工业界几乎无人采用。根因不是理论不够好，而是工程复杂度太高——EPaxos 的 conflict detection 和 rollback 逻辑让代码审查成为噩梦。Raft 的"够用就好"哲学在工程实践中胜出。

### Raft vs CRDT

CRDT（Conflict-free Replicated Data Types）是完全不同的范式：

| 维度 | Raft | CRDT |
|:---|:---|:---|
| 一致性模型 | 强一致（linearizable） | 最终一致（eventual consistency） |
| 网络要求 | 需要多数派可达 | 任意节点可独立写入 |
| 数据冲突 | 不允许（Leader 序列化） | 自动合并（数学保证无冲突） |
| 适用场景 | 关键元数据、协调 | 高可用、低延迟、允许暂时不一致 |
| 代表系统 | etcd、etcd | Redis CRDB、Apple 时钟同步 |

**CRDT 的正确用途**：用户状态同步（在线文档协作）、配置热更新（最终一致即可）、边缘计算（离线写入后同步）。CRDT 不是 Raft 的替代品——它们解决不同的问题。

**关键区分**：Raft 回答"谁是对的"（强一致），CRDT 回答"如何合并分歧"（最终一致）。两者不可互换。

---

## etcd：Raft 的教科书案例

etcd 是 Kubernetes 的唯一数据存储后端，也是 Raft 在工业界最成功的应用。

### 为什么 etcd 用 Raft

| 需求 | etcd 如何满足 |
|:---|:---|
| 强一致性 | Raft majority commit 保证 linearizable read |
| 高可用 | 3-5 节点，容忍 1-2 故障 |
| 小数据量 | MB 级元数据，Raft 写放大可接受 |
| 低延迟 | 本地读取（Leader），Follower 可处理 stale read |
| 简单运维 | etcd 的运维比 MySQL 集群简单一个量级 |

### etcd 的数据规模

一个典型 Kubernetes 集群的 etcd 数据：

- 100 个 Pod：~100 KB 元数据
- 1000 个 Pod：~1 MB 元数据
- 10000 个 Pod：~10 MB 元数据
- 包括所有 ConfigMap、Secret、Service、Endpoints

**这就是 Raft 的甜蜜点**：数据量小（MB 级）、变更频率低（数十 QPS）、但一致性要求极高（过期数据 = 集群故障）。

### etcd 的反模式

etcd 不应该被用作：

- **通用 KV 存储**：数据量超过 GB 级时，Raft 写放大不可接受
- **缓存层**：etcd 的读延迟（毫秒级）远高于 Redis（微秒级）
- **消息队列**：etcd 不支持高吞吐写入（数百 QPS 级别）
- **大对象存储**：单条 Value 建议 < 1.5 MB（etcd 默认限制）

---

## 架构决策：何时用 Raft，何时不用

### 决策矩阵

| 场景 | 推荐方案 | 理由 |
|:---|:---|:---|
| K8s 集群元数据 | Raft (etcd) | 强一致、小数据量、高可用 |
| 分布式锁 / Leader 选举 | Raft | Leader 天然互斥 |
| 服务配置中心 | Raft | 配置变更必须一致 |
| 小规模数据存储（<1 GB） | Raft (Fjall + Openraft) | 单集群可接受 |
| 大规模数据存储（>10 GB） | SlateDB + S3 | S3 固定 3 副本，成本线性可控 |
| 海量数据（TB 级） | SlateDB + S3 | 对象存储是唯一经济选择 |
| 高可用缓存 | 独立方案（Redis/Valkey） | etcd 延迟过高 |
| 消息队列 | Kafka / NATS | etcd 吞吐量不足 |

### 核心原则

> **Raft 是"data about data"的共识协议，不是"data"的存储协议。**

Raft 的价值在于：**多数派活着 = 系统可用，少数派故障可恢复**。这个保证对元数据至关重要——K8s 集群的 etcd 挂了，整个集群失控。但对于用户数据，S3 的"3 副本 + 自动修复"已足够，且成本低两个数量级。

**Raft + Fjall 不适合做通用数据存储**。3 倍写放大、节点数线性增长的存储成本、运维复杂度——这些都是 Raft 用于数据存储的固有代价。对于数据存储：

- **小规模**（<1 GB）：单节点 Fjall 即可，不需要 Raft
- **大规模**（>10 GB）：SlateDB + S3，利用 S3 的固定 3 副本复制

Raft 的正确使用姿势：etcd 做元数据、K8s 做编排、分布式锁做协调。不要用它存你的用户数据。

### 分布式数据库的 Raft 选择：TiDB

TiDB 是"Raft 用于数据存储"的代表性架构——也是理解 Raft 存储代价的最佳案例。它不是反面案例，而是**强一致分布式数据库的正确架构**，只是有明确的适用边界。

**TiDB 架构**：
```
SQL 层（TiDB）
    ↓
PD（Placement Driver）← 路由：key → region → 节点
    ↓
TiKV 节点（每个节点存一部分 region）
    ├── Region 1: Raft Group（3 副本）
    ├── Region 2: Raft Group（3 副本）
    ├── ...
    └── Region N: Raft Group（3 副本）
        ↓
    RocksDB（本地存储）
```

**关键设计：不是 1 个 Raft 管所有数据，是 N 个小 Raft 各管一个 shard。** 每个 Region 有独立的 3 副本 Raft Group，写入只走 1 个 Region 的 Raft（3 节点），不是全部节点。写放大始终是 3x，不会随集群规模增长。

**节点数限制的解法**：

| 问题 | TiDB 的解法 |
|:---|:---|
| 单个 Raft 不能超过7 节点 | 每个 Region 只有3 副本，不随集群增长 |
| 数据量太大放不下 | Region 自动 split（类似分库分表） |
| 热点 Region | PD 检测 → 迁移到空闲节点 |
| 节点扩缩容 | Region 自动 rebalance |

**但3 副本 = 3 倍存储成本，这个没有变。**

| | TiDB（Raft 3x） | SlateDB + S3 |
|:---|:---|:---|
| 100TB 数据实际存储 | 300TB（3 副本） | 100TB（S3 内部复制） |
| 存储单价 | 本地 SSD $100-200/TB/月 | S3 $23/TB/月 |
| 月成本 | $30,000-60,000 | $2,300 |
| 写延迟 | < 1ms（本地 Raft） | 1-10ms（S3 PUT） |

**适用边界**：TiDB 是 2018 年的最优解——低延迟 + 强一致 + 私有化部署。2026 年 SlateDB + S3 成熟后，大部分场景不再需要这个架构。但对于金融交易、实时网关等延迟敏感场景，TiDB 仍是正确选择。

→ 分布式 KV 架构选型详见 [KV 存储引擎](kv-storage-engine.md) §11。

### Fjall + Raft vs SlateDB + S3：存储架构对比

| 维度 | Fjall + Raft | SlateDB + S3 |
|:--|:--|:--|
| 真理源 | 本地 NVMe（Raft 多数派确认） | S3 桶（11 个 9 可靠性） |
| 写延迟 | μs 级（MemTable，与 SlateDB 相同） | μs 级（MemTable，与 Fjall 相同） |
| 存储成本 | 3x（Raft 副本）+ 本地 SSD 单价 | 1x（S3 内部复制）+ S3 单价（低 20 倍） |
| Scale-to-Zero | 需保护本地磁盘 | 销毁 Pod 即可，S3 持久 |
| 跨区域复制 | Raft 跨机房（复杂） | S3 CRR（配置项） |
| 运维复杂度 | 高（Raft 集群 + 分片） | 低（无状态计算 + S3） |
| S3 依赖 | 无 | 必需 |

**判定**：两者写入性能本质相同（都是 MemTable 攒批）。差距在部署模型和成本——SlateDB + S3 的存储成本低 20 倍，运维简单，是大部分场景的默认选择。Fjall 仅在不能用 S3 时（私有化、离线）考虑，且在大 Value 场景（KV 分离）、复杂本地事务、极致本地性能方面有结构性优势。Fjall 官方无 S3 支持计划。

---

## 实现视角：Openraft 模块、主线与注意点

从"用 Openraft 实现一个 Raft 应用"看：先看模块地图，再沿一条主线（日志 → reducer → 业务数据 → 快照）对应到接口，然后辅助模块，最后注意点。以 **0.9.x** 为准，精确签名以 [docs.rs/openraft](https://docs.rs/openraft) 为准。为便于理解，用一个最简单的 KV 命令集举例：业务命令 `Put(key, value)` / `Delete(key)`，状态为 `HashMap<key, value>`。

### 全局：主要模块

- `openraft` + `openraft-macros` — **库**：共识引擎本体 + 定义类型用的宏。
- `Raft`（`raft` 模块）— **引擎句柄**：组装后的共识运行时；对外提交命令、查询、成员变更。
- `RaftTypeConfig`（`type_config`）— **类型绑定**：`declare_raft_types!` 把 NodeId / Node / Entry / 快照等泛型定死。
- `RaftNetwork` + `RaftNetworkFactory`（`network`）— **传输钩子**：节点间 RPC（append_entries / vote / snapshot）。
- `RaftLogStorage`（`storage`，含 `RaftLogReader`）— **日志仓库**：存日志 + 硬状态 + 边界账。
- `RaftStateMachine`（`storage`，含 `RaftSnapshotBuilder`）— **业务状态**：`apply` reducer + 快照。
- `Config` — **引擎参数**：选举超时 / 心跳 / 快照策略。
- **支持类型** — 数据：`Entry` / `EntryPayload` / `LogId` / `Vote` / `RaftMetrics` / `Membership`。

**一句话分工**：引擎帮你想清"复制、提交、选举"——你只通过四个钩子（TypeConfig + Network + LogStorage + StateMachine）把"类型、传输、日志、业务"接进去。

### 主线：日志 → reducer → 业务数据 → 快照

理解 Openraft 的主线是这四个概念一环扣一环：

```
日志（有序命令流）──reducer──→ 业务数据（应用后的状态）──快照──→ 控制日志规模
```

- **日志 = 命令 + 一致性元数据**。每条 entry = `log_id`（term + leader + index）+ `context`（可选）+ `payload`（命令）。日志仓库另有硬状态（`current_term` + `voted_for`，防脑裂重复投票）与边界记账（`last_log_id` / `last_purged_log_id`）。只有 `payload` 是业务命令，其余都是让所有节点"对同一串命令、同一顺序、同一任期"达成共识的元数据。
- **业务数据 = 应用后的状态**。日志是"怎么来的"（命令流），业务状态是"现在是多少"（结果）。
- **reducer(`apply`) = `(state, command) → state`**：把日志命令按 index 顺序折叠进业务状态；确定性的核心——同命令、同初态必同结果。
- **快照 = 固化业务数据、控制日志规模**。日志只增不减会无限膨胀，故把截至 `last_applied` 的状态固化成快照、删掉那段日志前缀；落后节点用快照追平而不重放整条日志。

> 术语：`apply`（reducer）形式上是（大状态空间、可无限）确定性自动机的转移函数——但在"状态机模式"的工程语域里它不算"状态机"，用 **reducer** 更贴切。完整辨析见 [自动机、状态机与 reducer](automaton-vs-state-machine.md)。

快照压缩闭环：

```
日志 [1][2]...[X] | [X+1][X+2]...      ← X = 快照的 last_applied
apply 推进到 X
(1) build_snapshot    把"到 X 为止的状态"固化成快照（本例 = 序列化 HashMap）
(2) purge_logs        删 ≤ X 的日志条目 ── 快照已接管这段前缀
(3) install_snapshot  落后/新节点不重放 [1..X]，直接灌快照，设 last_applied=X 续追
```

要点：快照是业务数据的固化（结果），不是命令的副本；`purge_logs` 的触发点是快照点 `last_applied` 而非 quorum（quorum 管"哪些被提交"，压缩管"删到哪"，两者正交）；删前缀安全是因为快照接管了它。代价：删掉前缀后失去命令级回放 / 审计史，要审计需另存命令（如 S3 归档）。

**主线的本质：这就是事件溯源（Event Sourcing）。** 把三层对应起来——

| 事件溯源概念 | 主线（你实现的 Openraft 层） | 作用 |
|:---|:---|:---|
| 事件日志（真源） | `RaftLogStorage`（日志仓库） | append-only 有序变更，唯一真源 |
| 投影（当前状态） | `RaftStateMachine::apply`（reducer） | 把日志 fold 成业务状态 |
| 快照 checkpoint | `build_snapshot` / `purge_logs` | 固化投影、控日志规模 |

也就是说：**用户要实现的主线——日志仓库 + reducer(apply) + 快照——本身就是一个完整的 event-sourced 系统**：真源是日志，状态是投影，快照是压缩。openraft 框架在这套 event sourcing 之上补的是**共识与一致性**（leader 选举、多数派确认、quorum、成员变更、日志复制与追平、快照分发），让它在多副本、分布式下，所有副本对同一日志、同一顺序、同一任期达成一致。

这条线可以延伸更远——**PostgreSQL 这类传统数据库在日志层面也一直是 event sourcing**：

| | 日志（真源） | 业务状态 | 共识? |
|:---|:---|:---|:---|
| Raft / Openraft | 复制日志 | `apply` 出的状态 | ✅ 多数派确认 + 选主 + 线性一致 |
| PostgreSQL | WAL | 表 / 索引 | ❌ 只能单向复制（主 → 从），从库回放 WAL 的投影 |

PostgreSQL 的 WAL 是事件日志，表是 reducer（对 WAL 的确定性重放）投影出的状态，主从复制是把日志单向广播给从库重放。它缺的只是**共识**——从库没有裁决权、没有选主、没有 quorum，谈不上多副本对"提交"达成多数派。所以 **Raft 复制存储与 PostgreSQL 的差别不在"是不是 event sourcing"（两者都是），而在日志之上有没有一层共识来保证多副本线性一致。**

### 主线对应的 Openraft 模块与接口

- **日志数据 / 命令流** → `RaftLogStorage`：`get_log_state` / `get_log_entries` / `try_append_entry` / `save_vote`；`purge_logs` 由压缩调用。
- **业务数据（状态）** → `RaftStateMachine`：状态在你实现的 `apply` 里持有。
- **reducer** → `RaftStateMachine::apply`：`apply(entries)` 按日志 index 折叠进业务状态。
- **快照（控制日志规模）** → `RaftStateMachine` / `RaftSnapshotBuilder`：`build_snapshot` / `get_current_snapshot` / `install_snapshot`；`purge_logs` 在 `RaftLogStorage`；`Config.snapshot_policy` 定节奏。
- **命令提交入口（生产端）** → `Raft<C>`：`client_write(cmd)` 提交 → 引擎复制 / 提交 → 触发 `apply`。

主线的职责边界：日志仓库与网络是**盲存盲读**（引擎传原始条目，你原样存 / 取，别反序列化命令）；真正"懂"业务的只有两处——`apply`（跑命令）与快照（读写业务状态）。

**你要写的：**
- `RaftLogStorage` — 哑持久化，盲存盲读。
- `RaftStateMachine::apply` — **业务核心**：反序列化命令 → 状态迁移。
- `build / install_snapshot` — 序列化 / 反序列化业务状态。
- `RaftNetwork` — 机械接线。
- `RaftTypeConfig` — 纯类型胶水。

### 辅助模块

- **网络**：`RaftNetwork`（`append_entries` / `vote` / `snapshot`）把引擎要发的共识消息包装成你的 wire；`RaftNetworkFactory` 为每个目标节点建一个实例。引擎调你，机械接线。
- **类型绑定**：`RaftTypeConfig`（`declare_raft_types!`）定 NodeId / Node / Entry / 快照类型——纯类型胶水，无逻辑。
- **引擎句柄 `Raft<C>`**（你调引擎）：`initialize()` 首启建集群；`metrics()` 查角色 / leader / term（**定位 leader** 就用它）；`change_membership` / `add_learner` / `remove_learner` 在线调容；`trigger_log_compaction` / `purge_log` 主动推进快照；`shutdown()` 下线。
- **参数 `Config`**：选举超时 / 心跳 / `snapshot_policy`（如每 N 条 applied 触发快照）等。
- **支持类型**：`Entry` / `EntryPayload` / `LogId` / `Vote` / `RaftMetrics` / `Membership`。

你完全不用管（引擎自动）：term / index / leader 选举 / 投票 / quorum、日志复制 / 提交推进 / follower 追平 / 快照触发节奏。

### 注意点

- **确定性**：`apply` 必须纯——同命令、同初态必同结果，否则副本发散。这是全部正确性的前提。
- **持久化**：日志 + 硬状态必须 durable（fsync / 落盘），生产不能用纯内存 store。
- **版本差异**：0.8 用 `RaftTypes` + 单一 `RaftStorage`；0.9+ 用 `RaftTypeConfig`（trait）+ 拆成 `RaftLogStorage` / `RaftStateMachine`。写前对照所用版本（openraft 自带 raft-kv-rocksdb 等 example）。
- **单 leader 语义**：`client_write` 只能发 leader；用 `metrics()` 定位并做重定向。
- **错误分流**：`StorageError`（落盘 / IO）与 `ApplyError`（业务拒绝）分开处理，别混成一路。
- **快照安全性链**：`purge_logs` 只删 ≤ last_applied——"已提交 → 已 apply → 已被快照覆盖 → 才可 purge"；任何未提交 / 未 apply 的 index 都不得删除。
- **别在 LogStorage 反序列化业务命令**：日志仓库盲存盲读，命令解读只在 `apply`。
- **网络 RPC 可重试 / 超时**：follower 会短暂落后，靠重试与快照追平，别把"一次 RPC 失败"当致命。

---

## 交叉引用

本文档与以下架构分析形成完整的决策闭环：

- **[KV 存储引擎](kv-storage-engine.md)**：Fjall/SlateDB 双引擎模式（元数据每节点独立，无共识）
- **[序列化协议分析对比](serialization-protocol-comparison.md)**：Raft 控制流的编码选择（Protobuf vs Bincode）
- **[分布式协作拓扑](distributed-collaboration-topology.md)**：Raft 定位的拓扑推广——控制面/数据面分工、大节点联邦与柔性事务、CRDT 的主权分界
- **[Aura 架构](aura-architecture.md)**：元数据单写（无共识）、数据面本地的完整架构位置
- **[HelixDB vs LanceDB 对象存储 AI 栈](helixdb-vs-lancedb.md)**：对象存储上 AI 数据栈的两种路线对比
- **[Redis 批判](redis-critique.md)**：Redis 为何被 KV 替代
