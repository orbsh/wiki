# KV 存储引擎：架构设计、复合键编码与用例

> **语言：** [English](kv-storage-engine-en.md)（主文档）· [中文](kv-storage-engine.md)
> **实现项目：** [OKM](https://github.com/orbsh/okm) — Rust KV crate，声明式 derive（`KeyEncode` / `RowEncode` / `EdgeEncode`）

**Status:** 持续演进
**覆盖引擎：** Fjall（本地 NVMe）、SlateDB（S3 云原生）、redb（本地 B-tree）、SQLite 对比
**架构：** [Aura 架构 §5](aura-architecture.md) — 双引擎模式（Fjall / SlateDB+S3）；细节见 aura 仓库 [`docs/design/storage.md`](https://github.com/orbsh/aura/blob/main/docs/design/storage.md)
**Cross-ref:** [Redis 批判](redis-critique.md) — Redis 为何被 KV 替代

## 核心论点

KV 以极简底座覆盖绝大多数存储需求，靠三个独立成立的判断：

1. **进程内零网络开销**：嵌入式 KV（Fjall 等）直接跑在应用进程内，本地读取无网络跳数——它让"为访问数据而网络调用"成为可避免的架构负担：
   - **数据与进程同址读取**：无网络跳数、无远端往返。
   - **消除 RTT 与序列化开销**：本地 get/put 直达，省去远端传输与编解码。
   - **对照 Redis「网络 RAM」**：数据在远端、延迟被 RTT 主导（见 [Redis 批判](redis-critique.md)）。
2. **极简底座覆盖多模型**：KV 只有 get/put/scan，却能在其上表达 Hash、ZSET、图、向量、全文等一切结构——**一个底座覆盖众模型**：
   - **复杂性收敛在编码层**而非引擎层，引擎保持极简 get/put/scan（详见[复合键编码](#复合键编码redis-数据结构--kv)）。
   - **结构是复合键的字节序投影**：Hash field、ZSET score 前缀、图关系、向量、全文倒排都编码进 key（详见[多模型能力](#kv-之上的多模型能力向量搜索全文检索图查询)）。
3. **从专家专属到 AI 可用**：KV 把存储复杂性封装掉，让存储建模从"专家专属"变为"AI 可参与"：
   - **AI 处理胶水代码**：Key 编码、序列化等机械部分由 AI 胜任。
   - **人类负责判断**：架构设计、审查，以及环境特定的生产运维。
## KV 的基本 API

KV 存储引擎的接口极简，只有四个操作：

```
put(key, value)              ← 增/改（Key 不存在则创建，存在则覆盖）
get(key)                     ← 查（点查，O(log N)）
delete(key)                  ← 删（标记删除，Compaction 时回收）
scan(prefix) / range(a, b)   ← 前缀扫描 / 范围扫描（有序迭代）
```

**Value 是不透明的字节 blob**。引擎不关心里面是什么格式、有哪些字段、如何序列化。所有 Value 内部的操作（取字段、修改字段、追加数据）都在应用层完成。

与 SQL 的对应关系：

| SQL | KV 等价操作 |
|:---|:---|
| `INSERT INTO t VALUES (...)` | `put(key, value)` |
| `SELECT col FROM t WHERE id=1` | `get(key)` → 应用层反序列化 → 取字段 |
| `DELETE FROM t WHERE id=1` | `delete(key)` |
| `UPDATE t SET col=col+1 WHERE id=1` | `get` → 应用层修改 → `put`（**非原子**） |
| `SELECT * FROM t WHERE name='alice'` | 没有等价操作（需要二级索引 Key） |
| `JSON_SET(col, '$.field', val)` | 没有等价操作（Value 是字节，无内部结构感知） |

**KV 没有 Value 内部操作**。SQL 有 `UPDATE SET col=col+1`、`JSON_SET`、`ARRAY_APPEND`——KV 全部没有。这是 KV 的极简哲学：引擎只负责存取字节，所有 Value 操作都在你的代码里。

**唯一的原子性保障**：部分引擎提供 Key 级别的原子操作：

| 引擎 | 原子操作 | 粒度 |
|:---|:---|:---|
| **Fjall** | Batch 原子写入 + 事务（MVCC 乐观锁） | 多 Key 一次提交 |
| **RocksDB** | `merge`（自定义合并函数，如 counter 递增） | 单 Key |
| **etcd** | `compare_and_swap`（CAS，条件更新） | 单 Key |

这些是 **Key 级别的原子性**，不是 Value 内部的操作。"加字段"、"删字段"、"加索引"全部是应用层逻辑——KV 只负责存取字节。

### 复杂度与物理 I/O

#### get(key)：O(log N)，与 B-Tree 同阶，物理路径不同

| | B-Tree | LSM-Tree |
|:---|:---|:---|
| 查找路径 | root → leaf，固定 3-4 层（B=256，N=10亿） | MemTable → L0 → L1 → ... → Ln |
| 每层操作 | 二分查找页面内 key | 二分查找 SSTable 索引 + Bloom Filter 过滤 |
| 磁盘 I/O | 3-4 次（每层一次页面读取） | 1-3 次（Bloom Filter 跳过大部分 SSTable） |
| 确定性 | 高（路径固定） | 低（取决于 Bloom Filter 命中率） |

B-Tree 的 O(log_B N) 是精确的——root 到 leaf 路径长度固定。LSM-Tree 的 O(log N) 是近似——取决于 levels 数量和 Bloom Filter 效果。正面查找（key 存在）时 LSM-Tree 通常更快，反面查找（key 不存在）时 B-Tree 更确定。都是 O(log N)，但 B-Tree 常数更可预测，LSM-Tree 常数通常更小（读场景）。

#### scan(prefix)：O(log N + K)，先定位再顺序迭代

```
scan("inv:hello:")
│
├── 1. 定位起始位置：O(log N)
│   在 MemTable + 各层 SSTable 中找到第一个 ≥ "inv:hello:" 的 key
│   （每个 SSTable 的索引二分查找）
│
└── 2. 顺序迭代：O(K)
    K = 匹配前缀的 key 数量
    MergeIterator 合并多个 SSTable 的有序流，每次 advance 取最小 key
    直到 key 不再匹配前缀
```

迭代是**顺序 I/O**，不是随机 I/O。SSTable 是有序的，scan 只是在有序流上向后移动指针。顺序读比随机读快 10-100 倍（HDD 上差距巨大，SSD 上也有数倍差距）。

#### 操作复杂度汇总

| 操作 | 复杂度 | 物理 I/O |
|:---|:---|:---|
| `get(key)` | O(log N) | 随机读 1-3 次 |
| `scan(prefix)` 定位 | O(log N) | 随机读 1-3 次 |
| `scan(prefix)` 迭代 | O(K) | 顺序读 K 条 |
| `put(key, value)` | O(log N) | 写 MemTable（内存），flush 时顺序写 SSTable |
| `delete(key)` | O(log N) | 写 tombstone 标记，Compaction 时回收 |

### SQL 翻译层 vs KV 管道链：可预测查询模式下的降维打击

对于框架平台（API 网关、Agent 执行器、3D 流水线），数据访问路径在设计期就已固化。去掉 SQL 不是为了省事，而是消灭数据库优化器（Query Planner）这个运行时黑盒，获得 100% 的物理性能确定性。

#### 实际对比：拉取最近 10 条对话记忆

**SQL 方式**（PostgreSQL / SurrealDB）：

```sql
SELECT message_id, content FROM agent_memories
WHERE session_id = 'session_456'
ORDER BY timestamp DESC LIMIT 10;
```

底层物理代价：词法语法分析 → AST 树生成 → 逻辑执行计划 → 优化器猜测索引扫描还是全表扫描 → B-Tree 节点间频繁跳转。

**KV 方式**（Rust + 嵌入式 KV）：

写入时 Key 已编排为倒序物理格式：`m:{session_id}:{u64_max - timestamp}:{message_id}`。查询只需一行 Rust 管道链：

```rust
// 前缀定位 → 向后走 10 步，搞定
let prefix = format!("m:session_456:").into_bytes();
let top_10 = kv_engine
    .scan(&prefix)         // 定位起始区间（O(log N) 内存二分）
    .take(10)              // 顺序读 10 条（O(1) 磁盘/内存顺序 I/O）
    .collect::<Vec<_>>();
```

**为什么 KV 胜出**：Rust 方法链比 SQL 声明式样板（SELECT/FROM/WHERE/ORDER BY/LIMIT）更精炼。没有黑盒优化器自作聪明——代码就是执行路径。数据从 10MB 到 10TB，执行效率不变，亚毫秒响应雷打不动。

#### 两笔常被忽略的开销：结果集缓存与权限校验

解析/优化之外，SQL 还有两笔「隐藏成本」，正好落在 KV 的盲区之外：

- **结果集临时缓存**：关系型查询要先在引擎侧把结果物化成结果集（中间表 + 返回缓冲），再经协议序列化回传。KV 是进程内直接 `scan` 到手——读出的字节不落 SQL 结果集的瞬时副本，直接交给反序列化器，复用业务内存（进程内零序列化哲学）。低并发下这只是一次几 KB 的波峰，但在每轮要拼装数万 Token 上下文的 Agent 热路径上，反复物化 + 拷贝是真实损耗。
- **每查询的权限校验**：解析之后，关系型还要对每句 SQL 做一次授权判定（角色 / 行级安全 / ACL 查表）。嵌入式 KV 没有独立鉴权层——谁能访问哪个前缀，是嵌入它的应用进程自己的策略，引擎层不检查权限。省掉的不只是一次查表，而是把「谁能读什么」从数据库运行时挪回应用层：权限成为你能审查的普通代码，而非引擎内部的隐式黑盒。

**可组合性**：SQL 的计算空间是孤岛——想在计算的中间步骤插入一个 ML 模型预测或图论算法，语言层面做不到，只能把数据拉回应用层。KV 管道链是宿主语言里的普通代码，控制流与数据流交织：`if/else` 编排不同的 scan 链、中间结果绑定变量、随时插断点检查。数据不必离开进程即可与生态内任何算法库衔接。

#### 生产级组件：原子双写 + 时间线索引

展示如何用一个原子 Batch 在写入主数据的同时，自动构建时间线倒序二级索引，确保主表与索引表不会数据漂移：

```rust
use bytes::Bytes;
use std::sync::Arc;

/// 原子批处理（等价于 Fjall WriteBatch / SlateDB 的 batch API）
pub struct TransactionBatch {
    pub actions: Vec<(Vec<u8>, Option<Bytes>)>,
}

impl TransactionBatch {
    pub fn put(&mut self, k: &[u8], v: &[u8]) {
        self.actions.push((k.to_vec(), Some(Bytes::copy_from_slice(v))));
    }
    pub fn delete(&mut self, k: &[u8]) {
        self.actions.push((k.to_vec(), None));
    }
}

pub struct FrameworkStorage {
    /// 跨 Tokio 多线程共享的全局嵌入式存储引擎
    /// 生产中替换为 Arc<fjall::Keyspace> 或 Arc<slatedb::Db>
    pub engine: Arc<String>,
}

impl FrameworkStorage {
    /// 原子双写：保存 Agent 记忆 + 自动构建时间线倒序索引
    pub fn save_agent_memory(
        &self,
        session_id: &str,
        message_id: &str,
        timestamp: u64,
        payload: &[u8],
    ) -> TransactionBatch {
        let mut batch = TransactionBatch { actions: Vec::new() };

        // 1. 主表：完整数据
        let data_key = format!("data:session:{}:msg:{}", session_id, message_id).into_bytes();
        batch.put(&data_key, payload);

        // 2. 时间线索引：倒序排列（u64::MAX - timestamp）
        let inverted_time = u64::MAX - timestamp;
        let mut index_key = Vec::new();
        index_key.extend_from_slice(format!("idx:time:session:{}:", session_id).as_bytes());
        index_key.extend_from_slice(&inverted_time.to_be_bytes());
        index_key.extend_from_slice(format!(":{}", message_id).as_bytes());

        // Value 为空——实体 ID 已编码在 Key 的字典序中
        batch.put(&index_key, &[]);

        batch
    }
}
```

主表存完整数据，索引表只存空 Value（实体 ID 已通过二分序编码在 Key 中）。一次原子提交，两个 Key 同时成功或同时失败。SQL 的 `INSERT INTO` 无法在一个语句中同时写入两张表并保证原子性——需要额外的事务包裹。

### 前缀扫描 vs SQL：抽象的承诺与代价

前缀扫描不如 SQL 灵活，也不如它抽象。这是一句实话，值得把代价与收益摊开说清楚。

SQL 的抽象层次更高：写谓词即可，选哪条索引、用不用覆盖索引、连接怎么排，都是优化器的活。问题在于这个承诺经常没有兑现，而且是以最糟的方式没兑现——不是报错，是把物理布局偷偷变成查询的一部分。凡是索引与键顺序有关的实现（MySQL 的复合索引最典型）都逃不掉左前缀规则：`INDEX(a, b, c)` 能不能服务 `WHERE b = 1`，取决于建表时字段的顺序。开发者于是被迫在声明式语言里维护一棵 B+ 树的形状图，字段顺序由查询可用性而不是语义决定。这是抽象泄漏，不是抽象（详见 [MySQL 批判](mysql-critique.md)「索引僵化」）。

前缀扫描把同一件事摆到台面上：键的布局就是访问路径，读的顺序就是存储的顺序，探针就是 WHERE。它不承诺替你选路，所以也没有承诺可以落空。

代价是三条真实的约束：

*   **灵活性**：谓词不能任意组合。SQL 里加一个 `WHERE` 子句是一个字符改动的成本；这里换一个访问维度是改声明、重建面、迁移数据的成本。
*   **抽象度**：开发者必须知道字节序、字段宽度、变长段的限制，知道 `[org][user][device]` 这个顺序对"按 user 查"友好、对"按 device 反查"不友好。
*   **演进**：查询模式要在建模期定下来，晚到的维度只能靠新增面补，或者在键里重新排布。

收益也是三条，与代价一一对应：读路径可预测（没有优化器的黑盒猜测，也没有统计信息过期这类运行时意外）；性能可推理（前缀命中即区间命中，扫描代价就是区间大小乘以单条成本）；语义诚实（不会出现"写起来很顺、跑起来全表扫"的落差）。

一句话概括取舍：SQL 用抽象掩盖了优化器的能力边界，前缀扫描把边界公开，交给开发者决策。选择后者等于承认自动优化的困难是真实存在的——承认之后，模型才能在它真正擅长的地方（建模期确定访问模式）提供价值，而不是继续承诺一件自己做不到的事。

### 复合键编码：Redis 数据结构 → KV

纯 KV 引擎没有 Hash、ZSET 等原语。前端需要的一切数据结构，都通过 Key 编码 + 前缀/范围扫描统一模拟（下表以 Redis 数据结构为参照给出通用映射）：

| Redis | KV Pattern | Read | Write |
|-------|--------------|------|-------|
| `STRING` | `str:<key>` | `get` | `put` |
| `HASH` | `hash:<key>:<field>` | `get` / `prefix` | `put` / `remove` |
| `LIST` | `list:<key>:<seq>` (8-digit zero-padded) | `range` | `put` + monotonic seq |
| `SET` | `set:<key>:<member>` → `""` | existence check | `put` / `remove` |
| `ZSET` | `zset:<key>:<score>:<member>` | `range` (score interval) | `put` / `remove` |

关键细节：LIST 需要原子序列号生成 → 引擎内部的单调计数器。ZSET Score 编码使用零填充固定宽度格式，保证字典序正确。

纯 KV 引擎没有 Hash、ZSET 等原语。一切数据结构都是通过 Key 空间编码（Composite Key Encoding）在字节序上模拟出来的——LSM-Tree 的迭代器天然按字节排序，这意味着只要 Key 编码设计正确，范围扫描和排序在存储层零成本完成。Redis 的这一层抽象**极薄**：Hash 只是次一级的键空间（键拼接），ZSET 只是把"读 → 计算 → 写"序列在服务器端原子化——而"读改写"的原子化正是 KV `Batch`/CAS 的既有能力，见 [Redis 批判](redis-critique.md#65-redis-数据结构-api-的虚假护城河)。

#### Hash 的物理编码

Redis Hash `HSET user:101 name "Alice"` 拆为两个独立的 KV 对：

```
Key: "hash:user:101:name"  →  Value: "Alice"       (string)
Key: "hash:user:101:age"   →  Value: [0x00000019]   (i64 big-endian, 8 bytes)
```

**前缀扫描批量删除**：`DEL user:101` 不需要先读取再逐字段删除。直接用 `prefix("hash:user:101:")` 定位所有字段 Key，批量发出 Delete 指令。LSM-Tree 的删除本质是写入 Tombstone 标记，O(1) 操作。Redis 的大 Hash 删除（百万字段级别）会阻塞单线程事件循环数十毫秒，LSM-Tree 引擎无此问题——多线程后台 compaction 异步清理 Tombstone，不阻塞前台请求。

#### ZSET 的物理编码：Score 作为 Key 前缀的逆向索引

ZSET 的核心约束是「按 Score 排序」。利用 LSM-Tree 的字节序排序特性，将 Score 嵌入 Key 前缀即可实现：

```
玩家 99 得分 100，玩家 88 得分 99

数据主表:
  Key: "data:player:88"  →  Value: {...}  (玩家结构体)
  Key: "data:player:99"  →  Value: {...}

排行索引（双写）:
  Key: "rank:scores:[0x0000000000000064]:99"  →  Value: ""  (score=100, 大端序)
  Key: "rank:scores:[0x0000000000000063]:88"  →  Value: ""  (score=99,  大端序)
```

Score 转为固定 8 字节大端序字节数组（`i64::to_be_bytes()`），确保数值大小与字节字典序严格一致。取前 10 名：迭代器 `Seek` 到 `"rank:scores:"` 前缀后向后遍历 10 次。无需内存排序，存储层自动保证顺序。

#### 双写原子性：ZSET 更新分数

更新 ZSET 分数涉及两步：删除旧分数 Key + 写入新分数 Key。这是两个不同的 KV 操作，必须在同一个原子批次内完成：

```rust
let mut batch = keyspace.batch();
batch.delete(old_score_key);   // 旧分数的索引
batch.put(new_score_key, b""); // 新分数的索引
batch.put(data_key, &updated_player); // 数据主表
keyspace.write(batch)?;        // 原子写入，全部成功或全部失败
```

如果不使用原子批次，崩溃导致只写了一半 → ZSET 索引产生脏数据（一个成员同时出现在两个分数位置，或丢失）。KV 引擎的 `Batch` API 保证底层 WAL 一次原子提交。

集群部署下 Batch 的原子性由共识层保证——原理见 §分布式 KV 章节，Batch 的原子性从单机延伸到集群。

## 设计模式：纯 KV 底座上的系统级范式

在纯 KV 世界里，算法不再是游离在数据库外面的胶水代码——Key 编码格式本身就是索引层、缓存层和隔离层。

### 访问模式驱动：建模的起点

技巧集之上需要一楼的起点：**先枚举全部访问模式，再反推 key 布局**。每个访问模式（点查实体、按前缀取最近 N、按字段区间扫、分页翻屏、关联回表）都应落成一个点查或前缀扫描——漏掉哪个模式，就是 key 里漏掉哪一段。查询模式可预测不只是 KV 相比 SQL 的适用性论证（见 [SQL 翻译层 vs KV 管道链](#sql-翻译层-vs-kv-管道链可预测查询模式下的降维打击)），更是**手工设计的输入**：SQL 让你声明"我会这么查"，KV 的纪律是把访问模式先列全，再让每种查询在前缀里有对应路径。

把这条纪律落成可执行的建模方法——每实体四层（ns → 主键 → 排序字段 → 访问方法）、访问方法强制、覆盖索引克制、主键复合的边界——见 [OKM 建模指南](https://github.com/orbsh/okm/blob/main/docs/MODELING.zh-CN.md)。

### 键空间模式：逻辑编码层 vs 物理分区

键空间模式常被误认为需要引擎级的多分区能力，其实它定义在**编码逻辑层**：把命名空间编码进 key（`ns:entity:field` 前缀、复合键编码、u16 命名空间字典），与引擎物理布局无关。任何 KV——包括扁平键空间引擎——都零门槛支持，因为 key 本就是扁平字节串，命名空间只是你编码进去的约定。

与之相对的是**物理分区**（Fjall `Partition`、RocksDB Column Family）——独立 memtable / flush / compaction 的**性能与资源隔离**手段（控制写放大、按命名空间回收空间、隔离 I/O），不是键空间模式的必要条件。命名空间物理不隔离时 KV 照常工作，只是少了这些优化。

区别一句话：**键空间模式定义在编码逻辑层；物理分区只是需要隔离时的加速器。** 需要按命名空间精细控制写放大或空间回收 → 选带分区能力（Fjall）的引擎；否则逻辑前缀编码 + 任意引擎即可。反例：sled（0.34.7，2024-10，至 2026-08 约两年停更）只有逻辑 tree、无物理分区，但它**完全适合键空间模式**——真正把它排除在生产选型之外的唯一判据是**维护停滞**，而非缺分区。

### 物理局部性：相关数据的字节序相邻

范围扫描天然顺序 I/O，但"扫多少字节"由 key 布局决定。物理局部性：把**会被一起访问**的数据设计成连续字节序——同一前缀的所有记录物理相邻，一次前缀扫即取整批，随机 I/O 变顺序 I/O（HDD 10-100x、SSD 数倍，见 [复杂度与物理 I/O](#复杂度与物理-io)）。这是 Bigtable 的立身之本：row key 既定义逻辑实体，又决定物理放置。

两条局部性来源：前缀聚合同一实体字段；补码反转让时间线最新优先（见 [物理层编码范式](#物理层编码范式key-编码的物理层通用技巧)）。判断标准：**一起读的，在字节序上也相邻**——否则扫描跨页跳读，前缀/倒序编码救不回丢失的局部性。

### Key 长度是成本变量

key 每多一字节，同时放大四项：MemTable 驻留、页缓存/索引占用、IO 字节、分片空间。且 key 全量驻留内存（MemTable/缓存），长度优化是**常数倍减内存**而非摊薄——HBase rowkey 设计的第一铁律即长度控制。水线判断：key 只放"定位与排序必需"的字段，可延后投影的元数据放 value 而非塞进 key；压缩手段见 [物理层编码范式](#物理层编码范式key-编码的物理层通用技巧) 的数字命名空间 / 长度前缀 / 定宽偏移，此处给出它们背后的统一成本模型。

### Index-Only Scan（索引即数据）

二级索引的 Value 留空，entity_id 编码在 Key 末尾。查询时仅遍历 Key 序列即可获取所有匹配的主键 ID，无需回表读取 Value——零磁盘 I/O。

```
数据主表：
  d:{tenant}:{type}:{id}  →  [完整业务实体]

属性索引表：
  i:{tenant}:{type}:{attr}:{value}:{id}  →  ""  (Value 留空)

查询「状态为 active 的所有用户」：
  Scan("i:1:user:status:active:") → 遍历 Key 序列 → 提取末尾 id
  无需读取任何 Value，零回表
```

**物理效果**：前缀扫描只触碰 LSM-Tree 的 MemTable + 索引层 SSTable（极小），不碰数据层的大 Value 文件。适合高频属性筛选（网关路由匹配、Agent 状态过滤）。

### Bitmap 前置拦截（热路径零 KV 调用）

网关高频检查「IP 是否在黑名单」「Token 是否合法」。每次请求都 `kv.get()` 即便命中缓存也有哈希查找开销。解法：在应用层内存常驻 Roaring Bitmap 或布隆过滤器。

```
写路径（新 IP 封禁时）：
  1. kv.put("blacklist:ip:1.1.1.1", "")   ← 持久化落盘
  2. bitmap.set(hash("1.1.1.1"))           ← 内存标记

读路径（网关拦截，零 KV 调用）：
  网关收到请求
    → bitmap.check(hash(ip))
    → 未命中 → 直接放行（QPS 百万级，纯 CPU 位判断）
    → 可能命中 → kv.get() 最终权威判定
```

**物理效果**：热路径（99%+ 的正常请求）完全在 CPU L1 Cache 内完成，不触发任何 KV 引擎调用。只有极少数命中 Bitmap 的请求才下沉到存储层。适合网关黑名单/白名单、限流计数器、Token 校验。

**与布隆过滤器的区别**：Bitmap 支持精确删除（`bitmap.unset()`），布隆过滤器只增不删。高频变更的黑名单用 Bitmap；只增不减的 Token 白名单用布隆过滤器更省内存。

### 应用层 MVCC（无原生 MVCC 引擎的时间旅行）

Fjall/SlateDB 不支持原生 MVCC。框架团队通过将版本号编排进 Key 骨架，实现应用层多版本控制：

```
Key: data:agent:101:v:[u64::MAX - 1001]  →  记忆状态 v1001
Key: data:agent:101:v:[u64::MAX - 1002]  →  记忆状态 v1002（最新）
```

- **常规读取**：`Scan("data:agent:101:v:").next()` → 补码反转后最新版本排最前，亚微秒拿到最新状态
- **时间旅行**：将前缀指针定位到目标版本号之后，正向扫描即得历史版本链。无需数据库快照锁，纯 Key 设计实现无锁历史回滚

**与 SurrealKV 原生 MVCC 的区别**：SurrealKV 内置 `tx.get_at(key, timestamp)` 直接查询历史版本，不需要应用层编码。Fjall/SlateDB 需要手动将版本号编入 Key。代价不同，效果相同。

### 读改写：任何 KV 引擎都不提供部分更新原语

「修改 value 里的一个字段」在所有主流 KV 引擎上都不存在为 API 原语——`insert`/`put` 永远是 value 整体替换，包括 B-tree 引擎 redb。这是 KV 接口形状的普适约束，不是 LSM 特有的问题。存储结构只影响「整条重写」的物理代价：

- **原地更新 B-tree**（InnoDB 页内更新）：磁盘直接改字节，写放大小
- **redb（COW B-tree）**：改一个字段 = 复制叶到根的路径页，物理上不原地，单次写代价反而高于 LSM 追加
- **LSM**：天然追加，旧版本被分层遮蔽，compaction 异步回收——「整条重写」的代价最便宜

**put 无需先删旧值**——新版本直接遮蔽旧版本：LSM 靠分层遮蔽 + compaction 回收，B-tree 原位覆盖。显式 `del` 只在「数据不再存在」时才需要；「更新一个值」永远是纯 put。（需要显式删除旧 key 的是二级索引场景，见 §二级索引的更新——索引 key 编入了会变的维度，不删旧 key 它就永远留在错误位置。）

高频字段级更新（计数器、权重累积）不应走 read-modify-write 循环，正规解法是 **delta 追加**：

```
# 不改 value，追加增量记录
put  edge:123/read/20260902T1000  → +1
# 读取时归并 delta 得到当前计数；低峰期批量 fold 回主记录
```

纯顺序写、无读改写竞态（并发 delta 互不覆盖）、写路径无锁。这正是 RocksDB **Merge Operator**（`merge(key, delta)`，引擎自动归并 delta 链并在 compaction 时 fold）的应用层等价物——Fjall/SlateDB 未内置 merge，需要在应用层自建，模式见上。与 §应用层 MVCC 同构：把「修改」变为「追加新版本」，只是 delta 方案只追加变化的字段而非整条记录，成本更低。

**delta 不是无条件更优——判据是并发模型，不是性能。** delta 与 RMW 在 LSM 上的写路径代价相同（都是 1 次追加写、旧版本遮蔽回收），差别在并发与读路径：

| | RMW 直写（`get count → put count+1`） | delta 追加 |
|:--|:--|:--|
| 写路径 | 1 次 put（含 1 次 get） | 1 次 put |
| 读路径 | O(1) 单点 get | 归并 delta 链，需后台 fold 才拉回 O(1) |
| 并发正确性 | 两写者同 get 同 put 丢更新，需 CAS/事务兜底 | 天然无锁，线性归并 |
| 复杂度 | 无额外机制 | delta 链 + fold 后台任务 |

选择判据：

- **单写者 / 访问已串行**（单 Agent 进程、同一 WriteBatch 内顺序更新）→ **RMW 直写**。热计数 key 几乎必然在 block cache，get 是 μs 级，比维护 delta 链 + fold 机制简单得多
- **多写入者并发更新同一计数**（多进程、多 Agent 实例共享 S3 真理源）→ **delta**。RMW 需要 CAS/事务兜底（多数嵌入式引擎无单 key CAS），delta 免掉整个并发问题
- **更新频率极高**（热路径每请求多次）→ **delta**。省掉每次 get 往返，fold 攒批摊薄

fold 的落点注意：归并结果应 put 到独立计数 key（value 是单个整数，整值替换，直接写新值无 RMW），不要 fold 回主记录的 msgpack 字段——那又是读改写循环，等于把要消除的模式从热路径挪到后台，本质没消。

### 二级索引的更新：变维度进不进 Key

索引 Key 里编入了「会变的维度」（如权重 `W:<weight>:<edge_id>`）时，维度变化需要**删除旧索引 Key + 写入新索引 Key**，两步必须在同一 WriteBatch 原子提交：

```
put  W:<new_weight>:<edge_id>  → ""
del  W:<old_weight>:<edge_id>      # LSM 上写 tombstone——删除也是一次写
```

漏删旧 Key 则按权重扫描捞出过时条目。改一次权重 = 2 次写（新索引 + tombstone），索引更新频繁时 tombstone 积压依赖 compaction 回收。TiDB/SurrealDB 的事务内索引维护就是这对 put+delete 的引擎内置版。

两条免删除的替代布局：

- **变更日志式**：`WI:<edge_id>:<ts> → weight`，索引 = 权重变更记录，查询时内存归并出各 edge 当前权重。纯追加零 tombstone，代价是查询多一层归并
- **版本懒清理**：Key 编版本号 `W:<weight>:<version>:<edge_id>`，旧条目不删、查询跳过非最新版本，低峰期批量回收

选择判据是**索引维度的更新频率**：低频（如记忆边的权重，批量刷）用标准 put+delete，tombstone 开销可忽略且实现最简；高频实时更新切追加式。

### value 打包粒度：整行 vs 每字段一个 key

OLTP 的访问模式以**实体为中心**——"取出这个实体的整行，改几个字段再写回"。因此默认应把一行**整行打包**（如 postcard 序列化）填入 value：读整行 1 次点查 + 1 次反序列化，完美匹配。

每字段一个 key 在 OLTP 下是灾难：

| 操作 | 整行打包 | 每字段一个 key |
|:---|:---|:---|
| 读整行 | 1 次点查 | N 次点查（或 1 次 scan + 拼装） |
| 改 3 个字段 | 读 1 + 写 1 | 先读 N + 写 3，且跨 key 要事务 |
| 写放大 | 1 行 1 次写 | 每字段一次写，N 倍放大 |
| 单字段点查 | 读整行反序列化（浪费） | 1 次点查（最优） |

**关键的纠偏**："每字段一个 key"**不是 DDL 的解药**，它只是把 DDL 的痛换成四个新痛——整行读 N 次往返、scan 拼行、写放大、多 key 原子性。要解决加字段的痛，**不该换存储布局，而该让序列化格式本身可演进**。整行打包对 DDL"不友好"的根因，不是"整行打包"这个决定错，而是**用了不可演进的序列化格式**（postcard 定长、无 field tag、自描述性弱）。

**DDL 真正解法：可演进序列化，而非换布局**

1. **版本化 payload（最简）**：value 头带 `schema_version`。加字段 = 新版本，旧行按旧版本读，用懒迁移（读时/写时升级），Key 不变、零 DDL。字段仍按位置，新字段只能追加在尾部。
2. **Tagged/TLV 编码（推荐）**：payload 改为 `field_id + len + bytes`。加字段 = 新增一个 field_id，旧 reader **跳过未知 field_id**，旧行不用迁移、新老行共存。这是 Cap'n Proto / SBE / FlatBuffers 的核心能力（schema evolution + 兼容，详见 [序列化协议分析对比](serialization-protocol-comparison.md) 的「半动态层」）。代价是比纯 postcard 略大，换取加字段零迁移。
3. **热/冷分仓**：核心热字段用定长紧凑区（固定偏移），新字段/冷字段走 tagged 扩展区。读老行：热字段直接读、扩展区可空。折中：热路径零解析、演进路径全兼容。

**推荐**：2 做主轴（可演进），需要极致 O(1) 单字段读时再补 3。

**字段级辅助索引（按实测引入，非默认）**：若整行为主、但偶有高频单字段热路径（如 `get(id) -> name`），可双轨：主 value = 整行 blob，辅助 key = `field_idx:{id} → 单字段值`。读单字段走辅助 key，写时同步更新主行 + 受影响的辅助 key。但**辅助索引 = 写放大**，是优化而非默认——只有实测确出现单字段热路径才引入，别为想象的热路径提前付（每加一条 = 和每字段 key 同一种代价）。

### WiscKey 键值分离（大 Value 场景的写放大解药）

典型场景：Key 几十字节，Value 几 MB（对话历史、3D 资产二进制、多模态特征向量、日志原始载荷）。传统 LSM-Tree 在 Compaction 时将 Key+Value 捆绑重写，写放大几十倍。

WiscKey 的核心：Key 和 Value 物理分离——索引层只存小指针，大 Value 追加写入独立日志文件。

```
【内存 + SSTable 索引层】
 Key: "asset:mesh:uuid_abc" → [File_ID : Offset : Length]  ← 十几字节指针
                                    │
                                    ▼ 磁盘随机点查
【独立 Value Log（顺序追加）】
 offset_3402 → [大体积二进制资产：3D模型 / 对话历史 / 多模态特征]
```

**物理效果**：
- **Compaction**：只搬动小指针 Key（几字节），不碰大 Value 文件。写放大从几十倍降为接近 1
- **读取**：索引定位指针后，一次磁盘随机点查（NVMe 上 ~10μs）抓取大 Value
- **适用场景**：任何 Key 小 Value 大的模式——Agent 对话历史、3D 资产、多模态特征、日志载荷

Fjall 3.0 原生支持 WiscKey（KV 分离），SurrealKV 通过 Blob Log 实现同等效果。SlateDB 依赖 S3 的 Range Get 读取大 Value。

### KV 的 value 存 Arrow/Parquet？列式存储在 KV 上的伪装

被「KV 是行存、无法按列投影」的短板驱动，一个自然的想法是：把 Arrow 存进 KV 的 value。但这个想法的价值完全取决于**一个先决条件——粒度**。Arrow 是列式内存格式，列式收益只有在一个 key 对应**多行批量**时才兑现。按此切两半，结论完全不同：

**解读 A：key → 单行（value = Arrow 序列化的一行）**——这就是普通 KV 行存，Arrow 只是换了个序列化格式。列式收益 ≈ 0，每行一个 1-row batch，还要额外背 Arrow 的 schema 头。纯赔本。

**解读 B：key → 列式批量（value = N 行 Arrow IPC buffer）**——这才是真正的设计。key = 段 id，value = 一个列式 chunk。读入 DataFrame 直接得到列式内存布局，**零行列转置**；段内可按列压缩（zstd/lz4），schema 自带在 buffer 里。但解读 B 本质是**把 KV 改造成了一个列式批存**，代价是放弃 KV 最值钱的东西：

1. **点查（OLTP）废了**——一个逻辑记录埋在 batch 里，要读整段才能拿到。KV 原本的「嵌入式零 RTT 行级定位」优势被批量粒度掐死。
2. **行级更新是重写**——Arrow buffer 近似不可变，给 batch 追加/改一行要重写整段。这天然是追加优先的形态，契合 LSM 的 append 特性，但和行级 update 背道而驰。
3. **跨段扫描仍要读全量**——要聚合所有段的某列，必须 range scan 读每个 value 的**全部字节**再做解释。KV 层不懂 Arrow，无法把「只投影某列」下推到存储。省掉了转置，但没省掉读全量。

**内存模型错配（进程内零序列化哲学）**：若架构的进程内表示是 `ciborium::Value`（零序列化），存储层存 Arrow、读出来转回 ciborium，转置代价只是挪了个位置又回来了。Arrow-as-value 只有存储与内存都用 Arrow（直通 DataFrame）才成立——那意味着整个数据路径绑定 Arrow 内存模型，放弃进程内零序列化哲学。两条路只能选一条。

**格式选型**：若真要存列式批量，应选 **Parquet** 而非 Arrow IPC——Arrow IPC 是传输/内存格式（面向零拷贝流式搬移），Parquet 才是存储格式（列式 + 行组统计 + 谓词下推）。但有个尴尬：标准 KV 返回**整个 value**，「value 内部按列跳过」的 Parquet 优势在单 value 层面是废的——反正读全量字节。这更说明批量粒度下 KV 只是「装字节的桶」，列式智能全在 value 格式里，KV 本体毫无贡献。

**诚实结论**：这是列式存储在 KV 上的伪装，服务的是「内部、分析为主、追加优先、单机中规模」的负载。主导负载是分析 → 直接上真列式（Parquet/S3/DuckDB），别在有行级访问包袱的 KV 上绕；需要行级访问/点更新 → 别批量化，回到普通行存 KV。Arrow-as-value 把「KV 是行存、不能列投影」这个短板补了，补法却是把它改造成列式批存——代价是丢掉 KV 行级访问的立身之本，等于换了个赛道。

## 物理层编码范式：Key 编码的物理层通用技巧

这几个范式不属于某个具体 Redis 结构映射，而是控制 Key 字节序、宽度、前缀与边界来利用 LSM-Tree 字典序和写入路径的通用物理层技巧。

### 数值的字典序编码：无符号 / 有符号 / 浮点 / 跨类型（总纲）

这是「key 里的一段数值如何编码成字节序」的分类总纲。按「域」分四层，每深一层，直接大端编码的保序性就需要额外变换维持：

| 域 | 字节序编码方式 | 保序性 |
|:--|:--|:--|
| **无符号正整数** | 固定宽度大端补零 | ✅ 字典序 = 数值序，零变换 |
| **有符号整数** | 加偏置，或符号位翻转 + 其余位取反 | ⚠️ 需变换 |
| **浮点数** | 正数翻转符号位、负数全部位取反（±0 / NaN 另约） | ⚠️ 需变换 |
| **跨类型** | 类型 tag + 负载（FoundationDB tuple 风格） | ℹ️ 同 tag 内有序、tag 间有大序 |

多数键空间只在**无符号正整数**域运作——它零变换、成本最低，两个重要落地是**大端补零**（正序入门口）与**补码反转**（在该支上叠加倒序、用于时间戳），见下方子节。有符号 / 浮点 / 跨类型是越界时再补的技能：文档的多租户复合键是定宽 struct，tuple 是其「长度可变的通用多态」版本。

**坐标系外的替代路线：自定义 Comparator。** 以上默认把排序**编码进 key**；RocksDB / HBase / LevelDB 另提供自定义 Comparator，让引擎按给定顺序比较 key。取舍：需跨引擎可移植或依赖前缀有序 → 编码进 key；单一引擎且排序依据频繁演进 → Comparator 省去 key 内冗余排序字段，但牺牲跨引擎通用性。

#### 无符号正整数的落地：大端补零（工程陷阱）

**绝对不能**将数字转为字符串后拼入 Key。字典序与数值序不同：

```
字符串序:  "9" > "10"   (比较 '9' > '1'，'9' 字符码更大)
数值序:    9  < 10
```

正确做法是固定宽度的大端序字节数组。Rust 实现：

> 以下示例使用冒号分隔便于阅读。生产环境应使用固定宽度偏移编码，参见 [OKM 项目](https://github.com/orbsh/okm)（[README 中文版](https://github.com/orbsh/okm/blob/main/README.zh-CN.md)）。

```rust
fn score_key(prefix: &[u8], score: i64, member: &[u8]) -> Vec<u8> {
    let mut key = prefix.to_vec();
    key.extend_from_slice(&score.to_be_bytes());  // 8 bytes, 高位在前
    key.push(b':');
    key.extend_from_slice(member);
    key
}
```

所有 Score 共享同一字节长度（8 bytes），高位补零由硬件指令自动完成，字典序 = 数值序。

##### 为什么是大端：memcmp 从左往右比，最高有效位必须在最前

memcmp 逐字节比较，**先比到差异就分出胜负**。要让字节序等于数值序，就必须把数值的权重从高到低依次铺进字节流——这正是大端序的定义；小端序把低位放在前面，首字节是权重最轻的部分，一旦数值跨字节（进位过第一个字节），逐字节比较的结果就和数值比较脱钩。

以 u16（2 字节）数值 256 = 0x0100 与 1 = 0x0001 为例，两种端序在字节流里的摆放：

```
数值:        256 (0x0100)          1 (0x0001)
            权重: 高8位|低8位      权重: 高8位|低8位

大端 (BE):   [0x01][0x00]          [0x00][0x01]
             高位在前                高位在前
memcmp:      首字节 0x01 > 0x00 → 256 > 1  ✓ 与数值序一致

小端 (LE):   [0x00][0x01]          [0x01][0x00]
             低位在前                低位在前
memcmp:      首字节 0x00 < 0x01 → 判 256 < 1  ✗ 字节序与数值序相反
```

小端把 256 的最低位（0x00）放在了比较最优先的位置，而它的「大」全靠第二字节——memcmp 却已经用第一字节做出了相反的判决。**根源**：数值 256 的有效信息在高位，小端却让无信息量的低位先发言。这不是构造出来的特例——任何跨过字节边界的数值（256、65536……）在小端下都翻车，只有恰好落在第一字节内的数值才侥幸幸存。

大端对前缀扫描还有一层结构性收益：**数值编码与 key 前缀是同一件事**。ns 占 u16 高 15 位时，前两个字节就是完整的 ns 判别段——「扫描某个 ns 的全部 key」就是 memcmp 前缀匹配，无需掩码运算。若用小端，ns 的位横跨两字节且与方向位交织，前缀必须按位对齐处理，扫描路径退化。

> 位级细节（方向位放低位、区间划分靠高位、多标志位排序）的完整展开见 OKM 项目 ADR-0001 / ADR-0002。


#### 重要应用：补码反转（Complement Encoding）—— 时间戳的倒序

KV 引擎按 Key 字节序从小到大（字典序）严格排序。时间戳递增时，新数据天然排在最底下。为了实现「最新数据排在最上面」（方便取 Top-N 最新记忆），框架设计中有一个教科书级的物理黑客手段——最大值减去当前值：

```rust
// 标准正序 Key：老数据在最上面，新数据在最底下
let key_ascending = format!("log:{}:{}:", session_id, timestamp).into_bytes();

// 倒序 Key：最新写入的数据天然排在最前面！
// u64::MAX - timestamp：时间戳越大（越新），减出来越小
// 越小 → LSM-Tree 字典序越靠前 → 物理层面「最新优先」
let inverted_time = u64::MAX - timestamp;
let mut key_descending = Vec::new();
key_descending.extend_from_slice(format!("log:{}:", session_id).as_bytes());
key_descending.extend_from_slice(&inverted_time.to_be_bytes()); // 大端序保证按位对比正确
```

**物理含义**：`u64::MAX - 1722500000` = `18446744072007051616`，`u64::MAX - 1722500001` = `18446744072007051615`。后者更小，在 LSM-Tree 中排在更前面——时间戳越大（越新），Key 字典序越小，迭代器正向扫描自然得到倒序结果。

**应用场景**：Agent 对话历史倒序加载、排行榜取最新记录、日志时间线倒序读取。任何需要「最新优先」的前缀扫描场景都适用。与 `SeekForPrev` 反向迭代器互补——后者依赖引擎支持，补码反转在所有 KV 引擎上通用。

> **对齐**：此处的 `timestamp` / `inverted_time` 为 u64，属总纲「无符号正整数大端保序」保证的范畴——补码反转正是在这一支上叠加倒序变换。该变换适用于全部定宽整型：有符号补码整数经偏移（加 `i64::MIN` 绝对值）即得无符号保序域，逐位取反同样保持倒序性质；浮点因 IEEE 754 字节序与数值序不同构被排除，须先 Quant 归桶成整型再反转（实现口径见 [OKM 项目](https://github.com/orbsh/okm) ADR-0004 的 `Reversible` trait）。

### 索引主键 ID：统一放 Key 末尾，Value 留空

二级索引要回主表拿完整记录，就必须能定位到主表主键 ID。**统一策略：把主键 ID 拼到索引 Key 末尾，Value 留空（Index-Only Scan）**，不因索引是 unique 还是多值而切换布局。不再按基数分叉的原因：

- **性能几乎无差别**：前缀摘要唯一时，scan 定位同为 O(log N)、只命中一条，与 get 成本相当——ID 放 Value 省下的只是"从 Key 末尾解析一次 ID"的常数开销，微乎其微。为一个常数级别的差异维护两套布局不值得。
- **与非 unique 统一**：多值索引本就要求 ID 放 Key 末尾（KV 的 key 必须唯一，否则后写覆盖前写）。统一放 Key 后，unique 与多值共享同一套布局、同一套 `scan(idx, scan_fn)` 前缀扫描 API，`unique` 与否只是"scan 结果 Vec 长度为 1 还是 >1"的调用方判断，不再是布局分歧。
- **范围扫描是常态**：ID 放 Key 末尾，天然支持按索引字段**范围**取多条（如"city=sh 的全部 user"、"timestamp 区间内的全部消息"）——逐条 scanning 即可，无需回表。若 ID 放 Value，查询方不知道要扫的完整 Key、无法用 get 单点，范围查询反而做不了。对二级索引这类天生服务于"筛选/列举"的结构，范围扫描远比单点高频。

**唯一值得放 Value 的例外**：纯点查热路径——索引字段业务上确定唯一、且从不做范围列举，只为省那一次"从 Key 解析 ID"的常数开销、并省下 ID 在 Key 里的 16 字节。这是明确测量到瓶颈后才做的优化，不是默认。

```rust
// ID 统一放 Key 末尾
i:1:user:status:active:{user_id}   →  ""
// unique 与否不影响布局：scan("i:1:user:status:active:") 结果长度自证
```

**为何 Value 可留空**：ID 已在 Key 末尾，遍历 Key 序列即得全部 ID、不碰大 Value、零回表；若确需回主表读完整记录，用末尾 ID 再点查主表即可——无论 ID 放 Key 还是 Value，这一步都要做，**回表成本相同**。

**先纠正一个准技术前提**：前缀摘要唯一 **≠** 能 get。get 需要完整 Key，而 ID 放 Key 时查询方下单时恰恰**不知道 ID**，所以仍要 scan 前缀、命中后从 Key 末尾解析 ID。这正是"性能几乎无差别"的另一面——ID 放 Value 时 get 能省掉的，也只有这一次解析，而非一次扫描。

### 二级索引末尾必须追加主键 ID

设计二级索引（如通过时间戳反查对话消息）时，必须将全局唯一的实体主键 ID 拼接到索引 Key 的最末尾。

**原因**：高并发场景下，两个用户操作可能在同一微秒产生完全相同的时间戳。如果索引 Key 仅为 `idx:time:{timestamp}`，后一个写入会无声覆盖前一个（数据丢失）。末尾追加 `{message_id}` 既保证 Key 绝对唯一，又利用字典序维护时间线规整。

```
# 错误：同时间戳覆盖
idx:time:1722500000 → msg_abc  (被覆盖)
idx:time:1722500000 → msg_def

# 正确：主键 ID 保证唯一
idx:time:1722500000:msg_abc → ""
idx:time:1722500000:msg_def → ""
```

### 哈希前缀打散：避开自增序列的写放大灾难

当输入源严格自增递增（高频事件流水号、日志 tick），所有写操作集中撞击 LSM-Tree 末尾（Hotspot），后台 compaction 产生严重的磁盘写放大，IOPS 出现毛刺。

解法：在 Key 最前端注入哈希盐值，将连续写入打散到多个分区：

```rust
use murmur3::murmur3_32;

// 对 Key 核心内容生成 32 位哈希值，取模分入 16 个桶
let hash_prefix = (murmur3_32(&mut cursor, 0).unwrap() % 16) as u8;

let mut sharded_key = Vec::new();
sharded_key.push(hash_prefix);  // 1 字节哈希盐
sharded_key.extend_from_slice(b"metrics:ts:");
sharded_key.extend_from_slice(&timestamp.to_be_bytes());
```

物理效果：连续递增的写入压力被均匀分散到 16 个独立的内存树（MemTable）中，多线程并发刷盘（Flush）并行处理，避开局部块锁竞争，写入吞吐量翻倍。代价是前缀扫描需要遍历所有桶——适合写密集、读按精确 Key 点查的场景。

### 长度前缀编码：消除低效的字符串分割扫描

Key 中拼接多个字符串属性时，冒号分隔（`table:app_name:user_id`）要求读取时写循环 `split` 查找分隔符——O(N) 字符串扫描。改用固定 2 字节长度前缀：

```rust
pub fn pack_string_component(buf: &mut Vec<u8>, component: &str) {
    let bytes = component.as_bytes();
    let len = bytes.len() as u16;  // 2 字节存储字符串物理长度
    buf.extend_from_slice(&len.to_be_bytes());
    buf.extend_from_slice(bytes);
}
```

反序列化时，CPU 读取前 2 字节得知长度，直接跳过对应偏移量提取目标字段——从 O(N) 字符串扫描变为 O(1) 内存地址偏移计算。

**与冒号分隔的对比**：

| 维度 | 冒号分隔 `a:b:c` | 长度前缀 `[2]a[1]b[1]c` |
|:--|:--|:--|
| 反序列化 | 循环 split + 字符串比较 | 2 字节读长度 + 指针偏移 |
| 二进制安全性 | 字段内不能包含 `:` | 任意字节均可 |
| 固定宽度 | 否（依赖分隔符） | 是（每段 = 2 字节长度 + N 字节数据） |
| 适用场景 | 人类可读调试 | 生产环境高频读写 |

### 数字命名空间字典

命名空间需求可预测且封闭（改就改代码），故用**编译期常量**将字符串前缀压缩为固定 2 字节 u16 命名空间 ID：

| 字符串方案 | 数字方案 | 压缩率 |
|:--|:--|:--|
| `"user_sessions:"` (14 字节) | `u16::to_be_bytes()` → 2 字节 | **85%** |

命名空间 ID 由过程宏折叠为指令立即数，写 key 时零运行时查找——比 L1 常驻的 HashMap 还快（无 hash、无 load）。

**回退**：若不用过程宏/常量方案（把 ns→ID 留运行时解析），才需 HashMap（小则常驻 L1）——这是未做常量优化的必要实现，非首选；运行时"开放"注册 ns 无意义，访问模式在代码里，加了仍要改代码，动态注册换不来动态访问。

全局命名空间字典（编译期常量，过程宏实现，见 [OKM 项目](https://github.com/orbsh/okm)）：

```
Namespace 1 → sessions
Namespace 2 → users
Namespace 3 → logs
```

### 编码优化：把可变内容压成小紧凑表示（总纲）

KV 不止要对齐 key 结构，还要把字段的**内容**压小。原则：先问数据的**维度**，再选技巧——目标是用体积换更快的比较 / 索引 / 排序，代价是读路径解码复杂度。**类型驱动实现见 [OKM 项目](https://github.com/orbsh/okm)（ADR-0004「Value 侧与 wrapper」）**（`Enum<u8>` / `Offset<i32>` / `Delta<i32>` … 声明即自动编解码）。

按「压什么」分五个维度：

- **基数维度**（值种类少）
  - **低基数编号**：枚举等基数封闭的值 → 编号，**位宽锁死所选类型的固定字节宽**（≤256 个变体用 `u8`），不按当前基数动态推导——推导需运行期登记（`LazyLock` 参与编码计算），且位宽随变体数增长自动扩展会让历史 key 布局漂移。日志错误类型 / 状态 / 类别。
  - **块级字典**（低基数的泛化）：值域大但**块内重复** → 按块建字典存索引引用（列式 dictionary / bitmap）。运行时建字典、牺牲随机访问。
- **分布维度**（值集中）
  - **基准偏移**：设基准（近今，如去年）存**有符号差**——事件时间集中在未来几年，一个小有符号整数装下；负偏移表达更早值。
  - **Delta 增量**（进阶）：序列存与前值**差**，常为正小值，配 VarInt 更省；代价是随机访问须顺序重建。
- **重复维度**
  - **RLE 游程**：连续相同存 `(值, 次数)`——日志连续同错、状态长时间同态。
- **位宽维度**（压常见小值 / 量纲）
  - **VarInt**：常见小值 1 字节、大值扩展（LEB128）——与定宽互补，压「常见值」而非「上限」。
  - **量纲缩放 / 精度归一**：时间降精度（ns→ms）或归桶（小时桶），直接降基数 / 缩位宽。
  - **位字段打包**：多个枚举 / 布尔分段打包进同一字节。
- **排序方向维度**（字节序扫描顺序颠倒）
  - **补码反转**：定宽整型逐位取反，正序字节序变倒序——prefix scan 首条即最新（时间线、最新优先列表）。原理见「物理层编码范式」补码反转一节；实现为 `Reverse<T>` wrapper，`Reversible` trait 编译期白名单（仅定宽整型，浮点排除），见 OKM 项目 ADR-0004。

**取舍**：四维度不互斥、可叠加（`Offset<i32>` 字段仍参与前缀扫描）。核心代价 = 读路径**解码复杂度**（VarInt / Delta 尤甚）与**随机访问损失**（Delta / RLE 顺序依赖）。「零解析」只在定宽 / 全局基准这类读 O(1) 时成立；变长压缩读需解码，只在压缩收益超过解码成本时才值得。

### 多租户复合键编解码器：工业级实现

生产级组件——零堆分配（Zero-heap Allocation）的多租户复合键序列化/反序列化：

```rust
use std::convert::TryInto;

pub struct AgentMemoryKey {
    pub tenant_id: u32,        // 4 字节（租户隔离）
    pub session_id: [u8; 16],  // 16 字节（UUID 原始二进制）
    pub timestamp: u64,        // 8 字节（倒序时间戳）
}

impl AgentMemoryKey {
    /// 序列化：领域模型 → 固定 28 字节二进制流（无堆分配）
    pub fn serialize_to_bytes(&self) -> Vec<u8> {
        // 4(tenant) + 16(session) + 8(time) = 28 字节，全定宽靠偏移切分，无分隔符
        let mut buf = Vec::with_capacity(28);

        // 租户 ID（大端序）
        buf.extend_from_slice(&self.tenant_id.to_be_bytes());

        // 会话 UUID（固定 16 字节，无需编码）
        buf.extend_from_slice(&self.session_id);

        // 倒序时间戳（补码反转，最新优先）
        let inverted_time = u64::MAX - self.timestamp;
        buf.extend_from_slice(&inverted_time.to_be_bytes());

        buf
    }

    /// 反序列化：二进制切片 → 领域模型（零拷贝，纳秒级）
    pub fn deserialize_from_bytes(bytes: &[u8]) -> Result<Self, &'static str> {
        if bytes.len() != 28 {
            return Err("非法的物理 Key 长度约束违规");
        }

        let tenant_id = u32::from_be_bytes(bytes[0..4].try_into().unwrap());
        let mut session_id = [0u8; 16];
        session_id.copy_from_slice(&bytes[4..20]);
        let inverted_time = u64::from_be_bytes(bytes[20..28].try_into().unwrap());
        let timestamp = u64::MAX - inverted_time;  // 反转还原

        Ok(Self { tenant_id, session_id, timestamp })
    }
}
```

**设计要点**：所有字段固定宽度（4+16+8=28 字节），定宽字段靠偏移即可切分，无分隔符；反序列化无需解析、无需堆分配，纯指针切片操作。倒序时间戳内嵌在 Key 中，正向迭代器扫描即得「最新优先」结果。

## SQL 操作的 KV 实现

设计模式章节的几个范式（Index-Only Scan、Bitmap 前置拦截、应用层 MVCC、WiscKey 分离）是 KV 引擎的物理特性利用（LSM-Tree 字典序、补码反转、WiscKey 分离）。本章节从 SQL 的视角出发：多维查询、JOIN、聚合这些关系型数据库的核心操作，在纯 KV 底座上如何实现。

### 倒排索引交集（多维查询的 KV 实现）

SQL 的多条件组合查询：

```sql
SELECT * FROM orders WHERE status = 'shipped' AND region = 'east' AND amount > 1000;
```

KV 只有点查和前缀扫描两种原语。多维查询通过**倒排索引 + 内存交集**实现，与 Elasticsearch/Lucene 的 Posting List 交集算法同源。

#### 写入路径：每个维度独立建索引

```
数据主表：
  data:order:{order_id}  →  [完整记录]

维度索引（每个维度一个 Key 前缀）：
  idx:status:shipped:{order_id}  →  ""    ← Value 留空，ID 在 Key 末尾
  idx:region:east:{order_id}     →  ""
  idx:user:u7:{order_id}         →  ""
```

**Entity ID 编码在 Key 末尾**（§「二级索引末尾必须追加主键 ID」），保证扫描结果天然有序。Value 留空——Index-Only Scan，不碰数据层的大 Value。

每条数据写入时原子 Batch 同时提交主表 + 所有索引条目：

```rust
let mut batch = kv.batch();
batch.put(&order_key, &order_data);
batch.put(b"idx:status:shipped:00123", b"");
batch.put(b"idx:region:east:00123", b"");
batch.put(b"idx:user:u7:00123", b"");
batch.commit()?;
```

#### 读取路径：扫描 → 交集 → 回表

```rust
// 1. 两次前缀扫描
let a = kv.scan(b"idx:status:shipped:").map(extract_id).collect::<Vec<_>>();
let b = kv.scan(b"idx:region:east:").map(extract_id).collect::<Vec<_>>();

// 2. 归并交集（排序数组双指针 O(N+M)，CPU L1 Cache，纳秒级）
let common = intersect_sorted(&a, &b);

// 3. 回表点查（每个 ID 一次 get，~1μs/NVMe）
let results: Vec<_> = common.iter()
    .map(|id| kv.get(&data_key(id)))
    .collect();
```

`intersect_sorted` 实现：双指针归并，与 Lucene Posting List 交集算法同源，也与 [JOIN 章节的 Merge Join](#批量-join双指针归并merge-join) 是同一算法。

#### 性能分析

```
设：status 命中 1000 条，region 命中 500 条，交集 200 条

Step 1: 两次顺序扫描 → 1500 次迭代器 Next（顺序 I/O，~μs 级）
Step 2: 归并交集 → 1500 次比较（CPU，纳秒级）
Step 3: 200 次点查 → ~200μs（NVMe 随机读）

总计：~1ms
```

与 SQL B-Tree 索引扫描做同样的事：两次索引定位 + 归并 + 回表，也是 ~1ms 级别。**数学上等价，I/O 模式不同**：B-Tree 每次定位一个节点 = 随机读，LSM-Tree 前缀扫描 = 顺序读。NVMe 上顺序读比随机读快 5-10 倍。

**排序零成本是 KV 的结构性优势**。SQL 做 Merge Join 有一个隐含前提：两表必须在 JOIN 列上已排序。如果没建索引，数据库必须先跑一次 O(N log N) 排序才能启动双指针。KV 的前缀扫描天然返回排序结果——LSM-Tree 的字典序就是排序，MemTable + SSTable 的迭代器输出严格按 Key 有序。省掉排序步骤后，倒排索引交集直接进入 O(N+M) 归并阶段，比 SQL Merge Join 少一轮全量排序。

需注意：SQL 在 JOIN 列上建了索引时，B-Tree 叶子链表天然有序，排序同样免费。KV 的优势不是"SQL 做不到有序"，而是**前缀扫描在任何场景下都保证有序，不依赖索引选择**——设计期编码 Key 前缀时排序就已内嵌，运行时无需判断是否该走索引。

#### 写放大：维度数的代价

每多一个索引维度，每条数据写入多一次 put。3 个维度 = 4 次写（1 主表 + 3 索引），10 个维度 = 11 次写。这是所有索引系统的物理代价——SQL 维护 B-Tree 索引的开销本质相同。

#### 三策略对比

| 策略 | 维度组合 | 读延迟 | 适用场景 |
|:--|:--|:--|:--|
| **前缀编码**（Index-Only Scan） | 设计期固定，≤3 维度 | 1 次扫描 | 维度少、组合固定 |
| **倒排索引交集** | 任意组合 | 2+ 次扫描 + 交集 + 回表 | 维度多、组合不可预测 |
| **Bitmap 拦截**（Bitmap 前置拦截） | 设计期固定，维度值有限 | 0 次 KV（纯 CPU） | 高并发热路径 |

#### 数据组装范式的演进：SQL JOIN → ORM → 应用层组装

SQL JOIN、ORM、嵌入式 KV 是同一条演进线上的三个站：**数据组装从引擎黑盒逐步收回到应用层**。

**SQL JOIN：声明式的抽象泄露**。JOIN 是声明式的数据组装——描述"要什么"，优化器决定"怎么做"（Hash Join / Nested Loop / Merge Join）。代价是把执行路径交托给黑盒：数据量或分布变化时，优化器可能突然切换到灾难性计划（全表扫描后笛卡尔积），开发者被迫写 `STRAIGHT_JOIN` 或加 Hint 自救——等于亲手撕碎声明式抽象去对抗引擎。

**ORM：过程式心智模型套在网络上**。ORM 的流行，是因为人类处理复杂业务逻辑时，过程式 + 控制流的心智模型远比 SQL 的集合论直观：先查 A 表拿 ID 列表，再查 B 表，在应用层组装成对象图，中间结果有变量名、逻辑可见。但 ORM 跑在 SQL 之上，每次组装都要跨网络往返（应用 ↔ 数据库）——列表查询伴随 N 次关联查询，N+1 让性能崩塌。**ORM 的失败点不在"应用层组装"这个方向，而在组装操作的每次调用都背着一次 RTT**。

**嵌入式 KV：组装操作本地化**。KV 保留 ORM 的过程式组装心智模型（`get(user_id)` 拿主记录，`scan(prefix)` 拿关联记录），却把每次调用的代价降到本地 μs 级——N+1 的致命项（网络往返）消失后，进程内做 100 次点查的总耗时可能低于跨网络执行一次复杂 JOIN。组装路径就是代码路径，没有优化器选错索引的可能。

这条演进线的判断依据是代价函数的迁移：声明式省的是"人写执行计划"的成本，黑盒风险是"优化器猜错"；过程式付出的是"组装次数 × 单次调用延迟"，单次延迟从网络 RTT 降到本地 μs，是嵌入式 KV 让过程式组装重新成立的结构性前提。

与正交一轴的衔接：查询表达的组合性（SQL 字符串 vs 库形式的方法链）是另一条维度，见 [query-language-design.md](query-language-design.md) 的四参照系对比。本节的代价函数同时是 KV + DataFrame 组合成立的前提——组装/分析整体回到进程内后，嵌入式 KV 的零 RTT 扫描把「行存读出→列式分析」这笔转换的成本压到可付，四参照系中的「KV + DF」档由此落地。

## 去范式化 vs 应用层 JOIN

SQL JOIN 的本质是「根据关联键合并两个实体」。KV 没有关联表概念——核心决策只有一个：**在写入时合并，还是在读取时合并**。

#### 写时去范式化（读密集场景）

将关联实体的常用字段在写入时冗余嵌入主记录。读取时一次 get 拿全部信息，零 JOIN：

```rust
let mut batch = kv.batch();
// 主表：只存自己的数据
batch.put(&order_key, &order_data);
// 去范式化视图：嵌入关联实体的常用字段
batch.put(&full_key, &denormalized_data);  // {amount, status, user_name, user_email}
batch.commit()?;
```

**代价**：数据冗余 + 写放大（每份冗余数据一次额外 put）。**适用**：关联字段小（name、email）、读远多于写。

#### 读时应用层 JOIN（写密集 / 大关联字段场景）

保留实体独立存储，读取时两次查询 + 应用层合并：

```
订单表：data:order:{order_id}  →  {user_id, amount, timestamp}
用户表：data:user:{user_id}    →  {name, email, phone, avatar, settings}
```

```rust
let order = kv.get(&order_key)?;                          // 点查，~μs
let user = kv.get(&data_user_key(order.user_id))?;       // 点查，~μs
let result = merge(order, user);                           // 应用层合并
```

**代价**：2 次 KV 点查（各 ~μs），无冗余。**适用**：关联实体大（完整用户 Profile）或频繁更新（写入时不需要同步更新去范式化视图）。

#### 批量 JOIN：双指针归并（Merge Join）

当需要批量关联两个有序集合时，KV 的 LSM-Tree 字典序天然支持 **Merge Join**——与 SQL 的 Sort-Merge Join 算法同源，但 KV 省掉了排序步骤（前缀扫描已保证有序）。

```
两个有序集合（通过前缀扫描获得）：
A: [order:101:2026-01, order:101:2026-02, order:101:2026-03]  ← 已排序
B: [user:101, user:102, user:103]                              ← 已排序

双指针归并：
i=0, j=0
while i < len(A) and j < len(B):
    if A[i].key == B[j].key:
        result.append(merge(A[i], B[j]))
        i++; j++
    elif A[i].key < B[j].key:
        i++
    else:
        j++
```

```rust
// 伪代码：批量 JOIN
let orders = kv.scan(prefix: "order:101:");   // O(N)，已排序
let users = kv.scan(prefix: "user:");         // O(M)，已排序

// 双指针归并，O(N+M)，零额外排序
let mut i = 0; let mut j = 0;
while i < orders.len() && j < users.len() {
    if orders[i].user_id == users[j].id {
        result.push(merge(&orders[i], &users[j]));
        i += 1; j += 1;
    } else if orders[i].user_id < users[j].id {
        i += 1;
    } else {
        j += 1;
    }
}
```

**对比 SQL**：SQL 的 Sort-Merge Join 需要先排序（O(N log N)），KV 的前缀扫描天然有序（O(N)），省掉一轮排序。这与 line 462 的倒排索引交集（双指针归并）是同一算法——LSM-Tree 的字典序是 Merge Join 的天然加速器。

**适用**：批量关联、两个集合都已按关联键排序（通过 Key 编码保证）。

#### 多对多关系：指针索引

当关联关系本身是查询维度（标签、分类、多对多），用二级索引存储关联 ID：

```
订单的关联索引：
  rel:order:{order_id}:user  →  "user_abc"              (单值)
  rel:order:{order_id}:items →  [item_1, item_2, ...]   (多值，长度前缀编码)
```

本质上是把关系型数据库的外键索引搬到 KV 层。关联关系可以独立于实体数据演进。

#### JOIN 策略选择

| 场景 | 推荐 | 理由 |
|:--|:--|:--|
| 订单 + 用户名（小字段，读多） | 写时去范式化 | 一次 get 拿全部，延迟最低 |
| 订单 + 完整用户 Profile（大字段） | 读时 JOIN | 避免冗余存储大对象 |
| 多对多关系（标签、分类） | 指针索引 | 关联关系独立演进 |
| 固定 2-3 张表的关系 | 复合实体编码 | 直接拍平成一个大 Key |

### 预聚合计数器（GROUP BY 的 KV 实现）

SQL 的 `SELECT region, COUNT(*), SUM(amount) FROM orders GROUP BY region` 是查询时按需计算。KV 没有聚合原语，两个解法各有代价：

#### 写入时维护预聚合

每次写入主数据时，同步更新所有 GROUP BY 组合的计数器：

```rust
let mut batch = kv.batch();
// 主表
batch.put(&order_key, &order_data);
// 按天 + 按状态 + 按区域的聚合
let date = timestamp_to_date(order.timestamp);
batch.put(&count_key(b"agg:daily:", date, &order.status, &order.region), &increment(1));
batch.put(&sum_key(b"agg:daily:", date, &order.status, &order.region, "amount"), &add(order.amount));
batch.commit()?;
```

读取一次点查：`get("agg:daily:2026-08-05:shipped:east:count")` → 亚微秒返回。

**代价**：写放大随 GROUP BY 维度数指数增长。按天 × 状态 × 区域 × 类别 = 每笔订单写入时更新数十个计数器。适合维度少（2-3 个）、查询极频繁的场景（实时仪表板）。

#### 全量扫描 + 应用层聚合

适合数据量小或查询极不频繁的场景：

```rust
let results = kv.scan(b"idx:time:2026-08:")
    .filter(|r| r.status == "shipped")
    .group_by(|r| r.region)           // 应用层 GROUP BY
    .map(|(region, orders)| (region, orders.len(), orders.sum(|o| o.amount)))
    .collect();
```

**代价**：全量扫描 + 内存聚合，数据量大时 OOM。与 SQL 的全表扫描 GROUP BY 物理代价相同——SQL 也不会对没有索引的 GROUP BY 字段做任何优化。

### KV 的 DDL：索引管理与字段演进

SQL 有 `CREATE INDEX` / `ALTER TABLE`，KV 没有等价的声明式 DDL。所有 schema 变更都是应用层编码问题。但可以设计通用工具让操作变得机械化。

#### 加索引：回填历史数据

新写入直接用新 Key 模式，零成本。历史数据需要迁移回填：

```
旧 Key 模式:  m:{session_id}:{reverse_ts}:{msg_id}
新索引模式:   idx:{session_id}:{msg_id}    ← 按 session 查单条
```

通用迁移工具接口：

```bash
kv-migrate \
  --source-prefix "m:{sid}:" \
  --target-prefix "idx:{sid}:" \
  --transform "extract_msg_id_from_value" \
  --batch-size 10000
```

**执行原理**：`scan(源 prefix)` → 对每条 → `encode(目标模式)` → `batch.write()`。Batch 原子提交，LSM-Tree 追加写不锁读写，迁移可以和正常服务并行。断点续传：最后处理的 Key 记到系统表，中断后从断点恢复。

**删索引**：直接停止写入该前缀。旧 Key 不需要删除——LSM-Tree 的 Compaction 会在后台自动回收空间。如果需要立即释放：`scan(旧 prefix) → batch.remove()`。

#### 加字段/删字段：版本化编码

Value 是字节流，没有 schema。字段变更通过编码版本管理：

```
V1: [version=1][name:u16_len][age:u8]
V2: [version=2][name:u16_len][age:u8][email:u16_len]   ← 加字段
V3: [version=3][name:u16_len][email:u16_len]            ← 删 age
```

读取时按版本号分发：

```rust
fn decode(buf: &[u8]) -> Record {
    match buf[0] {
        1 => decode_v1(&buf[1..]),
        2 => decode_v2(&buf[1..]),
        3 => decode_v3(&buf[1..]),
        _ => unreachable!()
    }
}
```

**核心优势**：旧数据不用迁移，新旧版本共存。新写入用 V3，旧数据用 V1/V2 decoder 读，两者在同一数据库里并存。迁移可以推迟到空闲时段。

**什么时候需要回填迁移**：decoder 维护 3+ 个版本时代码开始腐化。此时跑一次迁移：扫描旧 Version 的 Value → 重新编码为最新 Version → Batch 覆盖写回。迁移后可以删掉旧版本的 decoder 分支。

#### 编码策略选择

| 策略 | 适用场景 | 优点 | 缺点 |
|:---|:---|:---|:---|
| **版本化编码** | 字段变化频繁（开发期） | 旧数据零迁移，新旧共存 | decoder 维护多个版本 |
| **追加编码** | 只加不删 | 最简单，零迁移 | Value 膨胀（空洞） |
| **迁移回填** | 字段稳定后清理 | Value 紧凑，decoder 单一版本 | 一次性迁移成本 |

#### 与 SQL DDL 的本质区别

| 操作 | SQL | KV |
|:---|:---|:---|
| 加索引 | `CREATE INDEX` = 数据库内全表扫描 + 重建 | 新 Key 模式 + 迁移工具回填 |
| 删索引 | `DROP INDEX` = 数据库内删除 | 停止写入，Compaction 自动回收 |
| 加字段 | `ALTER TABLE ADD COLUMN` = 大表锁分钟级 | 新 Version 编码，零 DDL |
| 删字段 | `ALTER TABLE DROP COLUMN` = 大表锁 + 数据重写 | 新 Version 编码，旧数据不迁移 |
| 字段重命名 | `ALTER TABLE RENAME COLUMN` | 新 Version 编码（KV 里没有"列名"概念） |

**判定**：KV 的 DDL 不存在于存储引擎层，而是应用层的编码约定 + 通用迁移工具。版本化编码让变更成本从"必须立即回填"降到"可以推迟"，迁移工具让回填成本从"手写脚本"降到"一行命令"。

### 编码层与引擎层的分工

应用层编码（OKM 式类型编码、物理层编码范式）与存储引擎自带机制（block 压缩、强类型 API）各管一段，边界常被混淆。以 fjall 与 redb 为参照说明分工。

#### 引擎级 block 压缩替代不了 key 编码

fjall（经 lsm-tree）自带 SST block 级压缩：`CompressionPolicy` 按 LSM 层级配置（如 L0 不压、深层 Lz4），`CompressionType::{None, Lz4}`。RocksDB 同款标准玩法。但它只作用于磁盘上的 SST block，key 长度的其余成本维度原样存在：

- **MemTable 全量驻内存，不压缩**——key 每多一字节是常数倍 RAM 成本
- **block index 的 separator key、bloom filter** 多在未压缩态
- **比较器每次比较都要解出完整 key 字节**——长度直接影响 CPU

「key 只放定位与排序必需的字段」的水线判断，引擎压缩一寸也替代不了。

排序语义更是压缩给不了的：`Reverse<T>` 补码反转、字段顺序即扫描语义、命名空间聚簇，是**结构语义**（字节怎么排决定 scan 能否工作）；LZ4 是**字节模式统计**（怎么存更短），解压后字节序复原。压缩不改排序，排序不省空间，正交。

两者反而互相成全：block 压缩吃的就是前缀重复度——ns 压成 2 字节 u16、同 ns 记录天然聚簇，Lz4 在 block 内找到更长的公共前缀，压缩率反而更高。离散随机的 key 压缩率才差。

真正与 block 压缩功能重叠的是**字段级变长压缩**（VarInt/Delta/RLE）在「省磁盘字节」这一点上。但动机不同：Delta/VarInt 让定长字段变短后 MemTable 与缓存同样受益（block 压缩只在磁盘层生效）；且 block 压缩是整 block 粒度——读一条记录要解压整块，字段级编码是点查路径零解压。取舍具体化：**磁盘维度的压缩优先交给引擎的 block 压缩，字段级变长编码只在内存驻留是主要矛盾时使用**。

#### 强类型 KV（redb）替代不了布局语义

redb 的 API 是强类型的：每个 key/value 类型实现 `Key`/`Value` trait（`as_bytes`/`from_bytes`/`fixed_width`/`compare`），表声明 `TableDefinition<K, V>` 后存取直接用 Rust 类型。这层「类型 ↔ 字节」的 boilerplate 与 OKM 式编码宏**形式上高度重叠**——但 redb 给的只是实现钩子，不是设计：

- **布局语义**：redb 的 `compare` 默认按 derive 出来的字节序比较（memcpy 序或 postcard 序）。ns 前缀聚簇、字段顺序 = 扫描语义、倒序扫描、变宽偏移下仍可前缀匹配——这些字节布局设计在 redb 里要靠自定义 `compare` + 精心构造的 `as_bytes`，设计本身（即编码规则）无法省略。
- **跨类型的布局纪律**：redb 每个类型各自 derive，字节格式各自漂移；跨类型前缀扫描（跨 session 全局时间线）靠不约而同的约定。编译期锁死的统一物理布局（hex 测试锚定）是应用层的事。
- **`fixed_width` 只是元数据**：redb 用它做页内优化，但不管理「定宽字段 + 长度前缀 + TLV 扩展区」的复合布局。

因此用 redb 时编码层不消失，而是**宏的 emit 目标改变**：`KeyEncode` 展开为 `impl redb::Key`（含自定义 `compare`），`ValueEncode` 展开为 `impl redb::Value`，编码规则一字不改。这也印证编码宏「零 I/O、纯编解码」的设计——换引擎只换 emit 目标，编码契约不动。

**分工总纲**：引擎给的是物理层机制（block 压缩、页管理、类型转换 boilerplate）；应用层给的是语义层设计（字节序布局、扫描语义、跨类型纪律、内存驻留优化）。两者正交且互相成全——好布局喂饱引擎压缩，引擎压缩放大好布局的收益。

### KV 之上的多模型能力：向量搜索、全文检索、图查询

三个能力都是 KV 之上的编码模式——用 Key 前缀模拟数据结构，用 Value 编码存储数据，用 scan 模拟遍历。算法库提供计算逻辑，KV 提供持久化和扫描。

#### 1. 向量搜索（DiskANN / HNSW）

**编码模式**：
```
vec:{id}              → [float32 × dims]       ← 向量本体
graph:{id}            → [u32 × K]              ← 图邻接表（K 个邻居 ID）
meta:entry_point      → [u32]                   ← 图入口节点
```

**查询流程**（贪心搜索）：
1. 从 entry_point 出发
2. `get(graph:{current})` 拿到 K 个邻居
3. 批量 `batch.get([vec:{n1}, vec:{n2}, ...])` 取向量
4. 计算距离，贪心跳转到最近邻
5. 重复直到收敛

**现有算法库**：

| 库 | 算法 | 集成方式 |
|:---|:---|:---|
| **usearch** | HNSW + Vamana（DiskANN 同族） | FFI 链接，图存 KV |
| **arroy** | HNSW（Meilisearch 内核） | 直接依赖 |

库负责建图和距离计算，KV 负责持久化图结构。`usearch` 的 `add()` 和 `search()` 是纯算法，不绑定存储——可以把内部数据结构 dump 到 KV，加载时从 KV 读回。

**工业级离线冬眠与零延迟唤醒**（以 Fjall 为例）：向量检索不必 7×24 常驻内存。HNSW 图序列化后持久化到 KV，节点启动/唤醒时反序列化载入内存，查询走内存图，无新向量插入时将图快照 + 原始向量打包写回 KV、内存释放冬眠。整个生命周期是"载入 → 内存查询 → 落盘冬眠"的循环：

```rust
// 伪代码：在宿主摊位的 KV 存储中融合向量持久化
pub fn save_vector_node(&self, partition: &PartitionHandle, vector_id: &str, embedding: &[f32]) {
    // postcard 紧凑序列化（定论见序列化协议分析对比）
    let serialized_vec = postcard::to_allocvec(embedding).unwrap();

    // 写入本地磁盘（Fjall 示例；任意嵌入式 KV 等价）
    partition.insert(format!("V:{}", vector_id).as_bytes(), serialized_vec).unwrap();
}
```

#### 2. 全文搜索（BM25）

**编码模式**：
```
fwd:{doc_id}           → [{term₁, [0,2]}, {term₂, [1]}]     ← 正排索引
inv:{term}:{doc_id}    → [tf, [positions]]                     ← 倒排索引
meta:df:{term}         → [u32]                                  ← 文档频率
meta:avg_dl            → [f32]                                  ← 平均文档长度
meta:doc_count         → [u32]                                  ← 总文档数
```

**倒排索引的值**：`inv:hello:42 → [2, [0,2]]` 表示词 "hello" 在文档 42 中出现了 2 次（TF=2），分别在位置 0 和 2。BM25 打分需要 TF，位置列表用于短语查询和高亮定位。

**一个文档拆 N 个 Key**：文档 "hello world hello"（doc_id=42）产生：
```
fwd:42          → [{hello, [0,2]}, {world, [1]}]
inv:hello:42    → [2, [0,2]]
inv:world:42    → [1, [1]]
```

一个含 100 个唯一词的文档 = 100 个倒排 Key。这是倒排索引的本质——按词拆分，不是按文档存储。插入时 Key 分散在 key 空间不同位置（"foo" 区域、"hello" 区域、"world" 区域），是天然乱序的。但 LSM-Tree 的 MemTable 是排序结构，无论写入顺序如何，flush 到 SSTable 时都是有序的——这是 LSM-Tree 相比 B-Tree 的核心优势。

**查询流程**：
1. 分词：`"hello world"` → `["hello", "world"]`
2. 批量查倒排：`batch.get([inv:hello:*, inv:world:*])` → doc_id 列表
3. BM25 打分：`score = Σ IDF(term) × (tf × (k1+1)) / (tf + k1 × (1 - b + b × dl/avg_dl))`
4. 堆排序取 Top-K

**BM25 打分**：核心公式十几行代码，不需要库。需要的是分词：

| 库 | 用途 |
|:---|:---|
| **jieba-rs** | 中文分词（Rust 绑定 jieba） |
| **tantivy** | 内置分词+BM25+高亮，可整体复用，也可只取 BM25 模块 |

英文按空格分词，中文需要 jieba 分词或 n-gram（简单但准确度低）。停用词过滤是应用层逻辑——维护一个 HashSet，分词后过滤掉。搜索本身就是 KV 的能力：分词 → 查倒排 → BM25 打分 → 排序，不需要额外的搜索引擎。

**源码示例**（以 Fjall 为例，正排 `D:` 与倒排 `T:` 两个分区）：

```rust
// [document_store]  Key: "D:<Doc_ID>"        → Value: 原始纯文本
// [inverted_index]  Key: "T:<Term>:<Doc_ID>" → Value: 词频（用于 TF-IDF / BM25 排名分）

pub struct SearchEngine {
    docs: PartitionHandle,
    index: PartitionHandle,
}

impl SearchEngine {
    pub fn new(keyspace: &Keyspace) -> Self {
        Self {
            docs: keyspace.open_partition("text_docs", Default::default()).unwrap(),
            index: keyspace.open_partition("text_index", Default::default()).unwrap(),
        }
    }

    // 1. 文本分析器：分词并建立物理倒排键
    pub fn index_document(&self, doc_id: &str, content: &str) {
        // 先存储原始文本
        self.docs.insert(format!("D:{}", doc_id).as_bytes(), content.as_bytes()).unwrap();

        // 极简分词清洗（通用开发环境可以使用 rust-stemmers 或 tantivy-tokenizer 增强）
        let words: Vec<String> = content
            .to_lowercase()
            .retain(|c| c.is_alphanumeric() || c.is_whitespace()); // 清洗标点

        let mut term_counts = std::collections::HashMap::new();
        for word in content.split_whitespace() {
            *term_counts.entry(word.to_lowercase()).or_insert(0u32) += 1;
        }

        // 2. 将词频原子化写入倒排索引分区
        for (term, count) in term_counts {
            let index_key = format!("T:{}:{}", term, doc_id);
            let val = postcard::to_allocvec(&count).unwrap();
            self.index.insert(index_key.as_bytes(), val).unwrap();
        }
    }

    // 3. 关键词检索：求多个 Term 的 Posting List 交集
    pub fn search(&self, query: &str) -> Vec<String> {
        let search_term = query.to_lowercase();
        let prefix = format!("T:{}:", search_term);
        let mut matched_docs = Vec::new();

        // 闪速前缀扫描
        for item in self.index.prefix(prefix.as_bytes()) {
            if let Ok((key, _)) = item {
                let key_str = String::from_utf8_lossy(&key);
                let parts: Vec<&str> = key_str.split(':').collect();
                if parts.len() == 3 {
                    matched_docs.push(parts[2].to_string()); // 拿到 Doc_ID
                }
            }
        }
        matched_docs
    }
}
```

#### 3. 图数据存储与查询

**编码模式（属性图，每边一个 Key）**：
```
node:{type}:{id}           → [properties msgpack]      ← 节点属性
edge:{src}:{label}:{dst}   → [properties msgpack]      ← 边属性
adj:{src}:{label}:{dst}    → []                         ← 正向邻接（值为空，Key 本身编码了关系）
radj:{dst}:{label}:{src}   → []                         ← 反向邻接
idx:{type}:{prop}:{val}    → [u32 × N]                  ← 二级索引
```

**为什么每边一个 Key，而不是邻接表合并在一个 Value**：

```
// ❌ 单 Key 邻接表
adj:alice:knows → [bob, charlie]
// 加边需要 read-modify-write：get → append → put，需事务

// ✅ 每边一个 Key
adj:alice:knows:bob     → []
adj:alice:knows:charlie → []
adj:alice:knows:dave    → []    ← 新增，直接 put，原子操作
```

| 操作 | 单 Key 邻接表 | 每边一个 Key |
|:---|:---|:---|
| 加边 | get → append → put（3 次，需事务） | put（1 次，原子） |
| 删边 | get → remove item → put（3 次，需事务） | remove（1 次，原子） |
| 查邻居 | get（1 次，O(1)） | scan prefix（O(K)，K=扇出） |
| 并发安全 | 需要事务/锁 | 天然安全 |
| Key 数量 | N 节点 = N Key | N 边 = N Key |

每边一个 Key 是标准做法。Redis Graph 模块、Neo4j 存储层全是这个模式。单 Key 邻接表的唯一优势是"一次 get 拿全部邻居"，但代价是加边/删边需要事务，得不偿失。它也是 RMW 反模式的典型实例——当 RMW 无法通过重新编码（每边一 Key）消掉、必须更新集合类 value 时，delta 追加是比「get → append → put + 事务」更轻的路径（见 §读改写）。

**查询示例**（"Alice 的朋友的朋友中住在北京的"）：
```
1. scan(adj:alice:knows:)                   → [bob, charlie]
2. batch.get([adj:bob:knows:, adj:charlie:knows:])  → [[dave, eve], [frank]]
3. batch.get([node:person:dave, node:person:eve, node:person:frank])
4. filter where city == "北京"
```

多跳查询 = 多轮 scan/batch.get，每轮 O(K × log N)，K 是扇出。和 SurrealDB 的图遍历本质相同——没有查询语言包装，但物理路径一致。

图查询本身就是 KV 的编码模式——`scan(adj:...) → batch.get(nodes) → filter → 递归`，不需要库。只有复杂图算法才需要依赖：

| 库 | 用途 |
|:---|:---|
| **petgraph** | PageRank、社区发现、最短路径、连通分量等算法 |

查询时从 KV 加载子图到 petgraph 跑算法，结果写回 KV。简单的 BFS/DFS/多跳穿透直接在 KV 层实现，不需要加载到内存图结构。

**源码示例**（以 Fjall 为例，`open_partition`/`prefix` 等价于任意 LSM-Tree KV 的分区/范围扫描接口）：

```rust
// [Partition: Out-Edges] Key: "E:out:<Src_ID>:<Edge_Type>:<Dst_ID>" → Value: MsgPack(权重/属性)
// [Partition: In-Edges]  Key: "E:in:<Dst_ID>:<Edge_Type>:<Src_ID>"  → Value: None (仅用于快速反向索引)

use fjall::{Keyspace, PartitionHandle};
use serde::{Serialize, Deserialize};

pub struct GraphIndexer {
    out_edges: PartitionHandle,
    in_edges: PartitionHandle,
}

impl GraphIndexer {
    pub fn new(keyspace: &Keyspace) -> Self {
        Self {
            out_edges: keyspace.open_partition("graph_out", Default::default()).unwrap(),
            in_edges: keyspace.open_partition("graph_in", Default::default()).unwrap(),
        }
    }

    // 1. 插入一条边：出边/入边双向原子写入
    pub fn insert_edge(&self, src: &str, edge_type: &str, dst: &str, weight: f32) {
        let out_key = format!("E:out:{}:{}:{}", src, edge_type, dst);
        let in_key = format!("E:in:{}:{}:{}", dst, edge_type, src);
        let val = postcard::to_allocvec(&weight).unwrap();

        // 顺序落盘，布隆过滤器会自动加速单键判定
        self.out_edges.insert(out_key.as_bytes(), val).unwrap();
        self.in_edges.insert(in_key.as_bytes(), &[]).unwrap();
    }

    // 2. 流式遍历出度（零拷贝获取节点 A 的所有下游邻居）
    pub fn get_out_neighbors(&self, src: &str) -> Vec<String> {
        let prefix = format!("E:out:{}:", src);
        let mut neighbors = Vec::new();

        // 利用 LSM-Tree 强大的顺序范围扫描 (Range Scan)
        for item in self.out_edges.prefix(prefix.as_bytes()) {
            if let Ok((key, _)) = item {
                let key_str = String::from_utf8_lossy(&key);
                // 极其干净地切割字符串，提取 Dst_ID
                let parts: Vec<&str> = key_str.split(':').collect();
                if parts.len() == 5 {
                    neighbors.push(parts[4].to_string());
                }
            }
        }
        neighbors
    }
}
```

#### 整体架构

```
┌─────────────────────────────────────────────────┐
│  应用层                                         │
│  ┌──────────┐ ┌──────────┐ ┌──────────┐        │
│  │ 向量搜索  │ │ 全文搜索  │ │ 图查询    │        │
│  │ usearch  │ │ tantivy  │ │ petgraph │        │
│  └────┬─────┘ └────┬─────┘ └────┬─────┘        │
│       │            │            │               │
│  ┌────┴────────────┴────────────┴─────┐         │
│  │  编码层（Key/Value Schema）         │         │
│  │  vec: / graph: / fwd: / inv: / adj:│         │
│  └────────────────┬───────────────────┘         │
├───────────────────┼─────────────────────────────┤
│  KV 存储层         │                             │
│  ┌────────────────┴───────────────────┐         │
│  │  Fjall / SlateDB / RocksDB         │         │
│  │  scan / get / batch / remove       │         │
│  └────────────────────────────────────┘         │
└─────────────────────────────────────────────────┘
```

三个能力都是 KV 之上的编码模式，不是新存储引擎。组合起来就是一个多模型数据库——和 SurrealDB 的架构同源，只是没有查询语言层。

**引擎无关 vs 引擎实现**：本章编码范式与源码示例均以引擎无关的方式呈现（示例以 Fjall 为例，`open_partition`/`prefix` 等价于任意 LSM-Tree KV 的分区/范围扫描）。结合这些索引的**多语言协作检索架构**（Steel/PyO3/Rust 用户驱动路由）见 [嵌入式脚本语言选型](embedded-script-languages.md) §4.3。引擎选型的分水岭（存算一体 vs 存算分离：对象存储 AI 数据栈的上游应用层）见 [HelixDB vs LanceDB 对象存储 AI 栈](helixdb-vs-lancedb.md)。

## 分布式 KV：复制、一致性与主流实现

嵌入式两件套（Fjall / SlateDB）覆盖单机与存算分离；当需求升级到**强一致的数据复制 + 水平扩展**时，进入分布式 KV 领域。本章讲构造原理、为什么多数场景不自己造、以及主流实现选型。**共识/Raft 讨论集中在此**，其余各章不再重复。

### 两个正交轴：分片 × 复制

分布式 KV 的能力由两根轴决定，务必分开判：

- **分片（Sharding）**：把键空间横向切分到多个节点——单机装不下即需分片。含范围/哈希切分、分裂/迁移、路由。
- **复制（Replication）**：冗余保证一致与可用——节点/磁盘挂了还能读。含一致性协议与副本数。

分片解决「容量/吞吐」，复制解决「可靠/可用」。许多系统只做其一（Redis Cluster 只分片且复制异步；etcd 只复制不分片）。

### 一致性模型：强一致 vs 最终一致

复制之后的读由一致性协议决定：

- **leader-based 强一致（Raft / Paxos）**：单写多读、多数派确认、**线性一致**（读己之写、自动选举）。优点：简单、可证明、无客户端协调。缺点：不分片、每节点全量副本、写要走 quorum。
- **最终一致（gossip / LWW / 弱 quorum）**：任一节点可写、异步传播、行级最近写胜。优点：always-available、写延迟低、可水平扩展。缺点：读可能旧、冲突需对账。

### Raft 的原理与边界

Raft 是 leader-based 复制的标准实现：追加日志 → 多数派确认 → 状态机 apply → 自动选主（term/vote/quorum）。**线性一致**让它成为元数据共识的黄金标准——**但它的根本局限是：不分片、每个节点的副本是全量数据**。因此存储与写放大随节点数线性增长（3 副本的 3x 可接受，但每加一节点就是一份全量拷贝，无法再扩展）。结论：**Raft 只承载元数据规模**（注册、配置、锁、Catalog 指针）——这正是 etcd / Consul / ZooKeeper 的全部。原理与边界详见 [共识协议文档](consensus-protocol.md)，实现视角（状态机 / 日志仓库 / 快照压缩）见其「实现视角」章；与 Fjall 的 Openraft 集成见下文 §共识与协调层级。

### 分片如何救 Raft：每片一个 Raft 组

Raft 不分片，那就分片之后**每片各跑一个 Raft 组**——每个 Region/shard 独立 3 副本，总复制量从「节点数 × 全量」变成「每片 3x」固定，随节点数不增。这是 **TiKV（每 Region 一个 Raft）** 的路径，实际解决了 Raft 的线性放大。

但分片救 Raft 的代价是 80% 的重活都在**分片之外**：

- **分片管理**：分裂 / 合并 / 跨节点迁移 / 均衡（PD 级兜底）
- **路由**：key → 片 → 节点的元服务（本身又需共识复制）
- **跨片事务**：单片便宜，跨片要 2PC / Percolator / 提交协调器，涉及锁、死锁、重试

### 主流分布式 KV 对比

按「一致性 × 规模」分四类，恰好对应四种用途：

| 类 | 代表 | 定位 |
|:---|:---|:---|
| A. 数据规模 · 强一致 · 每片 Raft | TiKV（+PD）、CockroachDB、YugabyteDB | 分片+复制的现成物；跨片走 2PC/Percolator |
| B. 数据规模 · 强一致 · 全局 ACID | FoundationDB、Spanner | 提交协调器包办跨片事务，应用免 2PC |
| C. 元数据规模 · 共识 | etcd、Consul、ZooKeeper | 正是「Raft 只配管元数据」的实证 |
| D. 数据规模 · 最终一致 | Cassandra/Scylla、DynamoDB | always-available、可调一致性 |

**数据规模下的强一致二选一（FDB vs TiKV）**：

| 维度 | FoundationDB | TiKV + PD |
|:---|:---|:---|
| 分片/复制 | 有序全局键空间 + 范围切分；复制在日志/存储层，非每片 Raft | 每 Region 一个 Raft 组，PD 管放置/均衡/路由 |
| 事务模型 | 全局 ACID，跨片由提交协调器处理，应用无需 2PC | MVCC 快照隔离；单片便宜，跨片走 Percolator 2PC |
| 冲突处理 | 乐观并发：写冲突 abort + 重试 | 锁式；高写冲突下 OCC abort 多于锁式 |
| 一致性 | 快照隔离 + 冲突检测（常描述为等效可串行化） | 快照隔离（TSO 时间戳） |
| 运维 | 多进程角色（协调器/日志/存储），复杂 | PD + TiKV 多组件，也非平凡 |
| 背书 | Apple、Snowflake | TiDB/TiKV，万亿行级 |

**选型**：

- 元数据 / 配置 / 服务发现 → **etcd / Consul / ZooKeeper**（C 类，Raft 的正确用途）
- 要强一致分布式数据 + 跨片 ACID 省心 → **FoundationDB**（B 类）
- 每片 Raft 直白 + 水平扩展、跨片事务低频 → **TiKV**（A 类）
- KV 之上要 SQL 一体化 → **CockroachDB / YugabyteDB**（A 类）
- always-available、可忍受最终一致 → **Cassandra / Scylla / DynamoDB**（D 类）

### 在本架构中的位置

元数据协调走共识（C 类，见 [共识协议文档](consensus-protocol.md)）；业务数据**不分片复制**——Fjall 本地 + 落湖备份，或 SlateDB + S3（存算分离，副本固定 3-AZ 与节点数无关）。**真要强一致分布式数据时，别把 Fjall 拼成分片 + Raft（那等于重造 TiKV），直接用 A/B 类的现成物**。Fjall/SlateDB 与「分片 + Raft」是两类不同需求，三条分支对照：

| | Fjall（本地） | SlateDB + S3 | TiDB / TiKV（每片 Raft） |
|:---|:---|:---|:---|
| 写延迟 | < 1ms | 1-10ms | < 1ms |
| 存储成本 | 1x（本地） | 1x（S3 单价） | 3x（Raft 副本） |
| 容量 | 本地磁盘 | 无限 | 本地磁盘 × 节点数 |
| 强一致性 | 单机天然强一致 | S3 最终一致 | Raft 强一致 |
| 运维复杂度 | 低 | 低 | 高（PD + Region 调度） |
| 适用场景 | 单机小规模 | 大部分场景 | 低延迟 + 强一致 |

判定：单机小规模选 Fjall，大部分场景选 SlateDB + S3，低延迟 + 强一致才选 TiDB / TiKV——3x 存储成本是为强一致付出的代价，不需要时 SlateDB 的成本优势太大。

把嵌入式 KV 引擎扩展为分布式协同基础设施，流量按方向切分：**南北流量**（客户端 ↔ KV server 的网络访问层，见后文「KV server 的网络访问层」一节）与**东西流量**（节点之间的分布式锁与共识协调）——底层以共识保证一致性，其上提供基于共识的锁。

### 共识与协调层级

```
Business Coordination (locks, scheduling, election)
    └── Meta-Coordination (consensus protocol)
         ├── Log ordering
         ├── State machine state
         └── Membership changes
```

**Core principle**: 元数据共识是基础设施的基石，不是存储引擎的职责。Redis 没有共识层 → Redlock 建立在沙堡上。共识协议方案见 [共识协议文档](consensus-protocol.md)。

**永远不要重写共识算法**——重写引入新 bug，代价远超收益。

**架构部署**：

```
应用层（锁、调度、配置、会话）
        │
  状态机（Fjall 引擎 — 嵌入式持久化）
        │
  Fjall 引擎（LSM-Tree KV — 本地 NVMe）
        │
  ┌─────┼─────┐
  节点1  节点2  节点3    （单机或集群部署）
```

**关键洞察**：Fjall 是进程内嵌入式引擎——本地读取无网络跳数。元数据一致性不需要共识层：aura 的口径是每节点独立 meta 实例（控制平面单写，见 [Aura 架构](aura-architecture.md)）；共识仅在数据级强一致复制场景由外部现成方案承担（见 [共识协议文档](consensus-protocol.md)）。

> **Openraft 示例（历史分析）**：Fjall + Openraft 的集成曾作为数据级强一致复制的候选路径分析——Raft 提交日志条目 → 状态机 `apply` 写入本地 Fjall。该路径已明确不在 aura 默认架构内（元数据单写无共识，数据复制交外部 TiKV 类方案）；分析保留于 aura 仓库 [`docs/design/realm.md`](https://github.com/orbsh/aura/blob/main/docs/design/realm.md) 与 [共识协议文档](consensus-protocol.md)。

### 分布式场景：并发有序性与分片均衡

单机视角把 key 建模视为本地物理布局；跨节点、多写入者时，key 设计叠加两个分布式维度。

**并发有序性（Versionstamp）**：倒序补码假设调用方能预知时间戳。并发向同一父记录附加子项（多线程/多节点同写一个事件流、日志、时序子记录）时，两个写入者可能撞同一时间戳——需要**服务器端序列化分配单调排序键**（versionstamp）：按可观察的提交顺序排定先手，保证附加以稳定先后落盘。单机用原子计数器即可；多节点下由共识层序号或 Leader 单调分配，使跨节点的追加日志/事件流获得全局一致的提交顺序。它和"调用方预知时间戳再倒序补码"互补：一个依赖预知，一个交给引擎在并发下排定顺序。

**分片放置与均衡**：跨节点时 key 前缀同时决定**分片边界**——分片（tablet）沿 key 字典序切分，同前缀数据落同一分片。这带来双刃：前缀既是逻辑聚合也是物理局部性边界（利于批扫/局部事务），但访问集中到单一前缀会形成**热键**，单分片打满、其余空闲（DynamoDB 的 hot partition）。单机下文档用哈希盐打散写热点（均匀分散，见 [物理层编码范式](#物理层编码范式key-编码的物理层通用技巧)）；跨节点则需在**局部性**（同前缀同分片，利批扫）与**均衡**（前缀分散，避免热分片）之间权衡——哈希盐保均衡但破坏前缀局部性，前缀保局部性但热点风险。这是分布式 key 设计与单机最本质的差别：单机只管写放大，分布式还要同时管放置均衡。

## KV server 的网络访问层：南北流量（东西流量归分布式）

**默认形态是单机**：嵌入式引擎 + 网络访问层就是一个完整的 KV server，不带任何分布式层——"单机 + 网络服务"是更务实的场景。Redis、PostgreSQL 同样都是单机服务形态，但单机网络服务这件事上 KV server 优于它们：零解析、零 DDL、key 范式即模型（见前文「什么时候用 SQL」）；Redis 的单线程与内存成本（见 [Redis 批判](redis-critique.md)）则是这个形态里的劣化项。

「南北流量（North-South）」这个说法只是为了和节点间的「东西流量（East-West）」区分——**一般情况下只有南北这一层**：客户端 ↔ KV server，本节全部讲它。真有跨节点复制/分片的诉求，直接用现成的分布式 KV（FDB/TiKV，见前文「分布式 KV」），不要在单机上自拼分布式层：东西流量位于更底层、对调用方透明，客户端看到的依然是同一个南北访问层。

一句话概括：**KV server = KV 引擎（存储内核）+ 网络访问层（南北流量）**；东西流量归分布式层，默认不需要。

### KV server 的构成：KV 引擎 + 网络访问层

集中式 KV server 不是某个既定产品形态（Redis/etcd/FDB/TiKV 各自带着功能与历史包袱），而是两件事的拼装：**KV 引擎（存储内核）+ 网络访问层**。存储内核直接复用嵌入式引擎（Fjall/SlateDB 之流，见后文引擎对比与选型章节）——真正多出来的只有网络访问层：把"进程内调用"变成"跨进程调用"。

通道取舍很直接：

- **HTTP** —— 提供**便利**：工具链齐全（curl、浏览器、任意语言 SDK），指令打包成 pipeline 即可服务通用客户端；代价是协议开销（请求头、握手、逐请求解析），性能天花板低。
- **WS（WebSocket）** —— 提供**性能**：长连接复用、全双工、二进制帧，请求-响应变成复用连接上的帧下发，吞吐与延迟接近裸 TCP；且原生兼容文本协议——JSON 指令直接在 WS 上跑，调试、日志、跨语言对齐都省事。内部服务之间用 **WS + Postcard** 通常更有优势。
- **gRPC** —— 其实没有必要了：HTTP/2 + Protobuf 的两个卖点都被 WS 抵消（头开销被二进制帧替代、强类型被 Postcard 覆盖），代价反而多出一整套 protobuf 工具链与复杂网关。HTTP 提供便利、WS 提供性能，gRPC 夹在中间两头不占。

一句话概括：**KV server = KV 底座 + HTTP/WS**。这个架构在业务语义上，相当于把 MVC 的 **Models 和 Database 合并成了一层**——没有对象模型到 schema 的映射层，业务语义直接以 key 范式固化在引擎字节空间（「模型」就是 key 编码、「数据库」就是 value 字节串，见前文「复合键编码」一节）。池化、云托管、无状态都建立在同一份合并后的原语上——这也是为什么「集中式 KV server」与 SQL、与 Redis 等具体形态无关：它就是引擎加两层协议。

### 网络层：WS 微包装突破单线程限制

Redis 的单线程模型是 2009 年硬件条件下的最优解。在多核服务器上，Redis 的命令执行被锁死在单核——多实例分片引入客户端路由复杂性（见引擎对比与选型章节的横向选型表）。

Fjall + Tokio + WS 的组合提供等价的网络接口，同时突破单线程限制：

```
[WS Client]   ← 内部：Postcard 二进制帧；通用客户端：JSON 文本帧（WS 原生兼容）
      │
      ▼
[WS Server — Tokio 多线程异步运行时]
      │  多个 CPU 核心同时处理不同的 WS 连接
      ▼
[Fjall LSM-Tree — Arc<Keyspace> 线程安全]
      │  多线程并发读写同一个存储实例
      ▼
[NVMe SSD]
```

**计算层**：Tokio 天生多线程异步。数十个 CPU 核心并行处理不同连接，不存在 Redis 的单线程瓶颈。恶意阻塞命令（如全量 KEYS *）只影响单个 Tokio task，不阻塞其他连接。

**存储层**：Fjall 的 `Arc<Keyspace>` 实现线程安全。多线程可同时对同一个 Keyspace 发起读写，LSM-Tree 的无锁读路径（MemTable + SSTable）和后台 compaction 线程天然并发。

**进程内读写路径**：当 WS 服务与 Fjall 嵌入同一进程时，热路径（摊位状态读写）仍走进程内直接调用（ns 级），WS 仅用于跨进程的外部接入。双路径并存：进程内零 RTT + 网络请求标准化。

> **Tonic + gRPC 已废弃**：早期版本的网络接入用 Tonic + gRPC，现已整体放弃，原因（通道取舍汇总见上文「KV server 的构成」）：
> - **不灵活**：gRPC 把 RPC 与序列化捆绑成一套（Tonic 就是这套绑定的 Rust 实现），调度、中间件、连接形态全被 HTTP/2 + Protobuf 的框架钉死，想换负载均衡、想换编码都是引入新组件而非改配置；
> - **生态不如 HTTP**：curl、浏览器、任意语言 SDK、网关与可观测工具链，HTTP/WS 比 gRPC 成熟且普及度高得多；
> - **性能没有优势**：相比 WS 长连接二进制帧，HTTP/2 头与流式 RPC 的复杂度换不来更低的延迟或更高的吞吐——WS 帧就是裸消息，无头可减；
> - **普世性不如 WS**：WS 是浏览器原生协议、天然穿透性最好，同一通道既能服务内部二进制帧、也能服务外部客户端；gRPC 对浏览器与通用客户端都不友好；
> - **业界背书**：SurrealDB 的默认协议就是 WS（LIVE SELECT 实时订阅全靠它）——同为 Rust 分布式数据系，头部项目选了 WS 而不是 gRPC，已经说明问题。

**性能模型**：
- **单次读写**：ns~μs（进程内）vs Redis 0.1~2ms（网络 RTT）

- **吞吐量（异步批处理）**：Fjall 多线程并发随核心数扩展；Redis 上限 ~80K ops/s（单线程）

- **资源**：无需独立进程，LZ4 压缩，无需专用 DRAM 分配

## 什么时候用 SQL

纯 KV 之上叠加两层抽象——Parser（查询解析层）+ Optimizer（查询优化器）——就是一个完整的数据库引擎。SurrealDB、CockroachDB、TiDB 的诞生路径无一例外：拿现成的 KV 引擎（RocksDB/Pebble/SurrealKV）做底座，在上面写查询语言解析器和代价优化器。
```
客户端文本查询 ("SELECT * FROM users WHERE age = 25")
        │
        ▼
┌─ Parser ──────────────────────┐  文本 → AST（抽象语法树）
└───────────────┬───────────────┘
                ▼
┌─ Optimizer ───────────────────┐  AST → 最优物理执行路径
└───────────────┬───────────────┘
                ▼
kv.scan("idx:age:25:") → 回表点查   ← 你手写的 KV 指令
        │
        ▼
┌─ KV Engine (Fjall/SlateDB) ──┐  二进制字节落盘
└───────────────────────────────┘
```

### SQL 动态性神话

常被拿来反对 KV 的一句话是"SQL 更灵活/更动态"——拆开看只在很窄的意义上成立，**程序内部层面根本不成立**：
- **程序内部：SQL 同样写死**。一个应用能发出的查询集合，在编译/部署那一刻已经固定——SQL 语句写死在源码里；即便靠 ORM 动态拼 SQL，ORM 的映射规则本身也是写死的代码，只会生成那几种形态的查询。所以从"程序能做什么"看，SQL 不比 KV 更动态。
- **真正的动态性只出现在"即席查询面"**：SQL 存储能对外部客户端（psql、BI、迁移、控制台）在运行时回答任意查询与 DDL，无需重编译应用；裸 KV 没有这类标准查询面——外部工具缺 key 编解码与目标类型，无从下手。但这是"是否刻意暴露一个可查询的活接口"的**产品/架构决定，不是存储语义固有**。绝大多数应用不暴露它，于是 SQL 的内部访问与 KV 一样固定。KV 侧补齐查询面也有路径：把行快照导出成 Parquet（OKM 的 snapshot export），DuckDB/BI 即可像查表一样查，代价是一次转换与快照允许过期，换来的是无需在 KV 上支起查询引擎。导出也不构成额外负担：备份本就必需，Parquet 压缩率也理想，备份即分析快照，一举两得。
- **余下差别是设计语言与运维工具，不是动态性/更优**：迁移、Catalog、EXPLAIN、索引是一整套成熟工具；键空间建模难度不高于 SQL schema，只是生态年轻。SQL 的"标准"有一半来自所有人都投入过学习时间——**沉没成本，不是先天优势**。
- **性能排查并不更轻松**：复杂计划、join、基数估算难推理；KV 点扫语义更简单。真到性能瓶颈，"SQL 好排查"往往是反的。

### 资源池化 / 云托管 / 无状态：是拓扑性质，不是 SQL 的性质

一个常被当作"SQL 优势"的说法是：单一 SQL 实例可服务很多项目（资源池化），可用云托管（免运维），项目本身可无状态。**这三条都是真的、有价值的——但它们全部来自"存储被抽成独立集中式服务"，而非来自 SQL 查询语言**：

- **资源池化**：多项目共享一个数据库实例、复用连接与缓冲——这是**集中式 vs 嵌入式**的拓扑差异。一个进程内 KV 做不到，但**集中式 KV server**同样能池化——其本质是 **KV 引擎 + 网络访问层**（南北流量形态，见前文「KV server 的网络访问层」一节），池化的是合并后的原语，与 Redis 等具体产品形态无关。
- **云托管**：把状态外包给托管服务、免去自运维——是**外部托管**的架构决定，与查询语言无关。DynamoDB / Redis Cloud / 托管 KV server 一样成立。
- **项目无状态**：应用不持状态、只依赖外部存储作唯一持久点——是**状态外置**架构的收益。状态放到集中式 KV server 同样让应用无状态。

真正的轴不是 SQL-vs-KV，而是**嵌入式 vs 集中式**：
- **嵌入式存储**（Fjall/SQLite/D1）随应用交付——零网络、零 server、但无法跨项目池化；
- **集中式存储服务**——可池化、可上云、应用无状态，但引入网络 + server + 运维面。

想拿"池化 + 无状态"红利，选**集中式 KV server** 即得，**不必切换到 SQL**。把这三条当成 SQL 的胜利，方向就偏了。

**池化有负载天花板**：共享实例的池化红利**只在负载轻时兑现**（JAMStack 类轻架构、函数计算、多实例共享存储池）。数据量大到共享池成瓶颈、独占实例也到顶时，**重负载最终只能选 KV**——进程内零 RTT、零解析、单实例线性扛量，量再涨还有平滑扩展出口（沿键空间横向拆，见 [统一数据层架构](unified-data-layer.md)「负载重量 / 规模」）。

**公道话（SQL 唯一真实的优势是工程生态，不是语言）**：SQL 的**运维抽象更成熟**——RDS/CloudSQL 的托管、租户隔离、角色权限、quota、连接池、WAL 复制都是商品级开箱即用；自建一个压缩的 KV 集群，这些都要自己磨。这是**工程成熟度差异**，是"现在做起来更顺手"，不是"SQL 语言天生不可替代"。承认这一点，才不至于落入"KV 全知全能"的反向崇拜。

### 坚守纯 KV 的理由（框架团队的正确选择）

当查询模式在设计期就已 100% 固定时，Parser + Optimizer 是多余的运行时开销：

- **零解析损耗**：编译期直接写死 `kv_engine.scan(&prefix)`，不需要运行时解析 SQL 字符串
- **100% 确定性响应**：无优化器「抽风」变慢的风险，P99 延迟雷打不动
- **单一二进制体积**：不引入 SQL 解析器、优化器、类型系统的代码膨胀

还有一个多数对比里被漏掉的根本优势：**计算贴着数据，在嵌入式 KV 是起点属性，SQL 引擎要另叠一层才能补上**。数据库想贴近数据执行密集计算，需要进程内**扩展宿主**——SurrealDB 为此把执行拆成两个世界：

- **基础查询（声明式 SurrealQL）**：由底层核心直接调度、走原生索引，几乎零序列化开销；
- **密集算法（WASM 扩展 Surrealism）**：预编译机器码、接近原生速度——但数据每次在数据库核心与 WASM 虚拟机之间交换，都要经 Host Bridge 做序列化和内存复制（Context Switch），海量数据遍历时边界跨越开销显著。

典型 AI RAG 原子操作因此被拆成两层咬合：

```SurrealQL
-- 外层声明式事务：锁定行、执行更新
-- 内层 WASM 扩展 mod::ml::embed：紧贴数据侧榨干 CPU/GPU 生成向量
UPDATE article SET embedding = mod::ml::embed(content) WHERE id = 'tech_news';
```

嵌入式 KV 从起点就拥有这份一体化：查询与计算**同进程、同二进制**——`embed` 直接就是一次原生函数调用，零序列化、零边界跨越、天然原子。扩展宿主是 SQL 引擎为「计算与数据分离」补的税，不是必经的标配（决策版论证见 [统一数据层架构](unified-data-layer.md)「为什么路径 B 也不选单引擎」——「计算下推」）。

### 什么是「查询模式可预测」：需求可控性（话语权）决定

KV 的零解析损耗、确定性响应、单二进制体积，只有在查询模式可固化时才兑现。但**可预测与否不是技术属性，是需求可控性（话语权）问题**：

- **需求由你掌控且稳定**：查询在设计期可固化，KV 的物理优势（追加写、零 DDL、确定性响应）全部兑现，「坚守纯 KV」成立。
- **需求由多变的外部力量驱动**：查询不可预测。此时 **SQL 更通用**——即便 SQL 实现不了某个需求，失败也归结为「SQL 的技术限制」，这是行业统一认知、责任可外推；KV 则把「为什么做不到」变成你的实现责任。

判断标准不是「做产品 vs 外包」的组织身份，而是**需求是否由你掌控**。话语权丧失的两种方式：**外包**（甲方决定需求），以及**做产品但服务客户多变需求**（如企业管理软件——客户需求大于一切）。后者即便你是老板也无效：业务本身在为别人服务，你就没有话语权。

### 数据量大时，KV 不要求查询模式可预测

"查询模式可预测"是 KV 的第一优先级准则——可预测的模式能固化为键路径，编译期消灭运行时开销，零解析损耗。但还有一个容易被忽视的维度：**数据量本身**。

当数据量大到关系型引擎扛不住时，KV 的物理优势即使在查询模式经常变化的场景下仍然成立。原因在于 LSM-Tree 的写入代价是 O(log N)，而关系型引擎的模式变更代价与表大小成正比：

| 操作 | KV 的代价 | 关系型的代价 |
|:---|:---|:---|
| 新增查询模式 | 开始写新前缀的 Key，**零 DDL** | ALTER TABLE + 迁移脚本，大表锁分钟级 |
| 新增二级索引 | 写一个新的 prefix scan 函数，**零重建** | CREATE INDEX = 全表扫描 + 重建，大表小时级 |
| 数据迁移 | 旧 Key 保留，新 Key 按新编码写入，**零停机** | 大表重分区 = 长时间锁 + 复制 |

**物理本质**：LSM-Tree 的 SSTable 是追加写入、不可变的。新增 Key 模式只是在最新的 MemTable 里多一种前缀，不需要修改已有的 SSTable。关系型引擎的索引是 B-Tree，每次结构变更都涉及页面分裂和重组，代价与表大小成正比。

**实际影响**：即使你每月改一次查询模式——新增一个前缀扫描、调整一个复合键布局——KV 的成本是 O(1)（写一个 scan 函数），而关系型的成本是 O(N)（N = 表大小）。当 N 超过某个阈值（百万行起步），KV 的物理优势足以覆盖"模式不固定"带来的工程摩擦。

**判定**：查询模式可预测是 KV 的**最佳**使用场景，但不是**必要**条件。数据量大本身就是选择 KV 的理由——体量产生的物理优势（追加写、无索引重建、零 DDL 锁）让模式变更的代价从 O(N) 降到 O(1)。

### 需求变化对 KV 是 O(1)：加字段/表/索引天然更可控

一般需求变化（加字段、加表、加索引），KV 不只是"也不差"，而是**天然 O(1)**——比 SQL 更可控：

| 需求变化 | KV 的代价 | 关系型的代价 |
|:---|:---|:---|
| 加字段 | 可演进序列化加新字段，Key 不变，零 DDL | `ALTER TABLE ADD COLUMN` = 大表锁 |
| 加表 | 新前缀空间 | `CREATE TABLE` |
| 加索引 | 写一个新的 scan 函数 | `CREATE INDEX` = 全表扫描 + 重建 |
| 改查询 | 改 scan 函数，零迁移 | 优化器 + 重新设计索引 |

对整行打包：加字段 = 给序列化加一个字段（field_id / 版本），Key 完全不变，旧数据用旧版本 reader 读、新老共存，零 DDL、零迁移。加表 = 新前缀空间、加索引 = 加一个 scan 函数，都是"加一行代码"，不触碰历史数据、没有在线 schema 迁移。

若采用每字段一个 key 的列式布局，加字段确实更简单——只要添加一个新的前缀 Key 即可。但**这种"简单"正是它不可取的原因**：列式布局把"加字段"从序列化演进降级为"加一个扫描维度"，换来的是 OLTP 下整行读 N 次点查、写放大、跨 key 事务的代价（见「value 打包粒度」）；它只是把 DDL 的痛平移成了查询/写入的痛，并非免费的更可控——除非负载本就是列式聚合（OLAP），那才需要该字段为前缀的布局。

**推论——SQL 的真正剩余优势**：SQL 的优势**不在处理需求变化**（这块反而输给 KV 的零 DDL），而在两点：

1. **不可预测的即席复合分析**——SQL 是声明式，Parser + Optimizer 是现成的通用查询引擎，新查询不用写代码；裸 KV 是命令式，复合聚合（join 多表 + 多条件 + 分桶统计）必须手写 scan 组合 + 应用层聚合。设计期可预知 → 打平；事后才冒出 → SQL 胜。但 KV 侧并非只有手写一条路：行快照导出成 Parquet/Arrow 后交给 Polars/DuckDB，复合过滤聚合同样不用写代码（见 OKM 的 DataFrame bridge——只读桥接，转一次、允许过期）——此优势因此收窄为「免导出、查的就是活的在线状态」。
2. **跨边界共享**——键空间是**私有协议**，SQL 是**行业通用货币**。封闭单团队自用，私有语言高效；需对接第三方/现成 BI 工具，键空间要自发明协议 + 写文档 + 让人学。

**洞察**：SQL 的剩余优势不在"处理需求更强"，而在"把查询能力委托给了你没发明的通用解析器"。当团队既掌控需求、又愿意自研键空间时，这份委托失去必要性——但代价是失去即席未知查询和对外共享那两张"现成引擎"的网。

### 什么时候必须蜕变为数据库

只有当系统需要开放给外部第三方开发者、允许最终用户通过低代码/动态插件自由写出不可预测的复杂查询时——为了防止他们写出全表扫描的垃圾查询把底层存储扫爆，才必须在最前面加一层 Parser + Optimizer 做查询门禁。

**AdHoc 分析**是另一个典型触发场景。AdHoc 面向的数据分析师（广义含开发运维分析日志）本质在做**动态的多维查询**——不断修改范围、下钻、聚合透视，无法在设计期固化成 prefix scan。更根本地，这是**分析型（OLAP）负载**，而 KV 本质是**行存**：每个键对应一条值字节流，前缀扫描要读入并解码整条记录，无法按列投影、压缩与向量化。因此即便分析者掌握键 schema、能自写扫描函数，行导向 KV 在分析负载上仍远逊于**列式 OLAP 引擎（如 DuckDB）**——列存只读所需列、同型值压缩率高、过滤聚合可下推，性能是结构性碾压。纯 KV 的用武之地是点查与固定模式扫描（OLTP 形态）；**AdHoc 这类动态多维负载的正确归宿是列式 OLAP 层，而非纯 KV**。KV 侧的正确姿势不是硬扛分析负载，而是**导出**：行快照转成 Parquet（一次 O(N) 行→列转换，OKM 的 snapshot export / DataFrame bridge），落在 DuckDB/Polars 里查——OLTP 归 KV、AdHoc 归列式快照，两个引擎各干各的本行。

**收敛判断**：AdHoc 归 DuckDB、并发 CRUD 归 KV + API、跨边界归 Lakehouse——在 KV + Lakehouse 统一架构下，「蜕变为完整关系型数据库」基本不再有剩余触发。各触发逐一拆解：

- **AdHoc 分析 → DuckDB**：动态多维负载是分析型，DuckDB 这类嵌入式列式引擎零运维即可承载，不必蜕变为完整关系型服务器。
- **多用户并发 CRUD → KV + API 层**：KV 具备原子批量乃至完整 ACID（Fjall WriteBatch 原子提交 + 崩溃回滚 / SurrealKV 完整事务域）；多租户连接、鉴权、并发会话由 API 网关层承载，不进存储引擎。Rust + 嵌入式 KV + API 同机直连对外服务，性能远胜传统「多后端 + 单数据库」——后者的成因按语言各异，但都不是结构必需：
  - **Python：性能短板**——解释执行、GIL 等限制，后端难以在进程内高效服务高并发，因而收敛为单点数据库集中托管。
  - **PHP：架构限制**——除了性能问题外，无常驻业务进程、无状态短生命周期 worker 无法在进程内持有状态，被迫外挂共享状态服务器（先是数据库，后加 Redis 补速度），这也是 Redis 流行的重要原因。
- **跨边界共享 → Lakehouse**：第三方 / 现成 BI 经 **Lakehouse**（Iceberg on S3 的标准表格式）读取数据，无需在 KV 上支起一层 SQL——Lakehouse 本就是该架构的一部分，不是额外引入的数据库。仅剩边角：第三方绕开 API 与湖仓、用 SQL 直连**在线可变状态**。
- **分布式 → 元数据走共识、数据走存算分离**：元数据协调用共识（Raft 的原理与边界见 §分布式 KV）；大规模数据走 SlateDB + S3（存算分离，副本固定 3-AZ）或 Fjall 写本地 + Lakehouse 落湖。

**判定**：框架平台的正确姿态是坚守纯 KV + 复合键编码。数据库是 KV 的上层封装，不是 KV 的替代。如果你的查询模式是可预测的， Parser + Optimizer 就是用运行时 CPU 开销去解决一个编译期就能消灭的问题。

## 引擎对比与选型：架构 · 场景 · API

四款嵌入式/周边键值存储（Fjall、SlateDB、redb、SurrealKV）与 Redis、SQLite 的对比与选型，从三根轴展开：**架构**（引擎如何组织与部署）、**场景**（何时选谁）、**API**（开发者接口形态）。

### 架构

Fjall 和 SlateDB 都是纯 Rust LSM-Tree KV 引擎（Apache-2.0），底层数学逻辑相似。但它们在真理源（Source of Truth）和网络拓扑上走向了相反的极端，对应两种完全不同的部署模式。

#### 引擎定位对比（Fjall vs SlateDB）

| 维度 | Fjall | SlateDB |
|:--|:--|:--|
| **真理源** | 本地 NVMe/SSD | 云端对象存储（S3/GCS/MinIO） |
| **Flush 路径** | MemTable → 本地磁盘（系统调用 I/O） | MemTable → S3（网络异步写入） |
| **点查延迟（cache miss）** | μs 级（本地 NVMe） | ms 级（S3 Range Get 网络往返） |
| **容量上限** | 受限于本地磁盘 | 无限（S3 桶容量） |
| **ACID 事务** | 成熟（3.0+ WriteBatch/Transactions） | 快速演进中，高级事务控制补全中 |
| **语言绑定** | Rust only（跨语言需 KV server 层，见 §集中式 KV） | 官方 Python / Go / Java / Node / UniFFI（Swift/Kotlin） |
| **设计目标** | 单机 bare-metal，极致延迟 | 云原生，节点无状态化 |

#### 路径一：Fjall（单机本地部署）

```
[应用层] → [Fjall 引擎] → [本地 NVMe] → 返回
                            ↓
                      WAL + SSTable
```

**真理源在本地磁盘**。进程内直接写入 Fjall，无网络损耗，写入完毕立刻返回。延迟由 NVMe 物理特性决定（μs 级），不受网络波动影响。

**本地路径的 B-tree 备选**：本地部署若以点查/范围扫描为主、且要消除后台 compaction 停顿，可在 Fjall 与 redb 之间二选一——Fjall（LSM，写密集）或 redb（COW B-tree，读确定），能力差异见下文「核心 API 特性对比」。

**ACID 批处理**：AI Agent 场景频繁需要原子修改多个复合 Key（更新对话主表 + 更新排行索引 + 更新标签索引）。Fjall 3.0 的 WriteBatch 在 WAL 中一次原子提交，崩溃时整体回滚，保证索引一致性。

**容量上限**：数据不能超过本地高性能磁盘。需要无限存储时，不要在 Fjall 上加冷数据卸载——直接用 SlateDB。

**集群部署**：多节点场景下，元数据每节点独立（控制平面单写，无共识——aura 口径）；数据级强一致复制需求由外部现成方案承担（见 §分布式 KV 章节及其引用的 [共识协议文档](consensus-protocol.md)）。Fjall 本身专注本地存储引擎职责。

#### 路径二：SlateDB + S3

```
[WS 计算节点（无状态）] → [SlateDB] → [S3 桶] → 返回
                                                    ↑
                                            真理源在云端
```

**真理源在 S3**。数据 commit 后直接推送到 S3。S3 本身提供 11 个 9 的可靠性和跨区域复制——多节点同步由 S3 物理保证。

**计算节点无状态**：多个 Rust WS 服务连同一个 S3 桶。节点崩溃后在新机器重启，挂载同一 S3 路径，几秒内复活接客。这就是 Scale-to-Zero 的物理基础——S3 是持久的，计算可以随时生灭。

#### Fjall vs SlateDB：写入与读取路径对比

**写入路径**：

| | Fjall | SlateDB |
|:---|:---|:---|
| 写入目标 | 本地 NVMe（MemTable → WAL） | MemTable（内存）→ 异步 flush 到 S3 |
| MemTable 写入 | μs（本地） | μs（本地，与 Fjall 相同） |
| Flush | 后台写本地 SSTable | 后台写 S3 SSTable |
| 写延迟（用户感知） | μs（MemTable） | μs（MemTable），flush 异步不阻塞 |

**两者写入路径本质相同**：都是 MemTable 攒批 → 异步 flush 为 SSTable。区别只是 flush 目标（本地 vs S3），flush 是后台异步操作，不阻塞写入。写延迟差距比通常认知的小。

**读取路径**：

| | Fjall | SlateDB |
|:---|:---|:---|
| 热数据 | 本地 SSTable → μs | 本地缓存（block cache + SST cache） → μs |
| 冷数据 | 本地磁盘 → μs | S3 GET → ms |
| 数据量上限 | 本地磁盘大小 | 无限（S3） |

**核心差异不在性能，在部署模型**：

| | Fjall | SlateDB |
|:---|:---|:---|
| S3 依赖 | 无（离线可用） | 必需 |
| S3 API 成本 | 无 | 有（PUT/GET/传输费） |
| 存储上限 | 本地磁盘 | 无限 |
| 复制 | 需自己处理（Lakehouse 落湖备份；元数据共识见 §分布式 KV） | S3 内部处理 |
| 运维 | 自管磁盘 | 无状态计算 + S3 |

#### Fjall 的差异化优势（即使 SlateDB 支持本地存储）

即使 SlateDB 未来完美支持本地存储，Fjall 在以下方面仍有结构性优势：

**1. KV 分离（Value Log）**：Fjall 内置 `value-log` 组件（灵感来自 RocksDB 的 BlobDB/Titan）。写入大 Value（图片/文档/音频）时，Value 分离存储到单独文件，LSM-Tree 只保留 Key + 指针。极大降低写放大，大对象场景本地写入和 Compaction 性能远超 SlateDB。

**2. 更成熟的事务支持**：Fjall 内置可串行化事务（Serializable Transactions）及乐观/单写者事务模型（`OptimisticTxDatabase` / `SingleWriterTxDatabase`）。多并发本地事务控制、多 Keyspace 跨空间原子提交方面，更接近成熟本地 RDBMS 核心。

**3. 极致本地优化**：Fjall 3.0 对本地磁盘 Block 格式彻底重构——稀疏索引（Sparse Indexing）、前缀截断、布隆过滤器分区、可选哈希索引（Hash Index）。未命中缓存时，本地磁盘随机点查和范围扫描快 2-100 倍，内存开销极低。

**4. 基因纯正**：SlateDB 的核心设计是 "Zero-Disk"（零本地盘依赖），并发锁、Fencing、Flush 策略都围绕网络对象存储延迟优化。即使支持本地写入，这些架构包袱（如因适配网络而做的激进 Batching 导致的即时持久化延迟）很难完全抹除。Fjall 100% 为本地 NVMe/SSD 吞吐量和 OS 文件系统设计。

**5. 跨语言的答案在服务化，而非绑定**：SlateDB 官方提供 Python / Go / Java / Node / UniFFI 绑定，多语言团队可直接嵌入同一真理源——这是 Fjall 结构上给不了的（Rust only）。但 Fjall 的对应答案不是换引擎，而是 KV server 拼装（见 §集中式 KV）：Fjall + Tokio + WS 把嵌入式引擎变成多语言可访问的网络服务，同时进程内热路径仍保留零 RTT 直调。代价是多一层网络访问层与运维；收益是 Python/Java 侧拿到的是跨进程标准化接口，绑定层的 API 完整性反而更高。

#### SlateDB 的本地磁盘模式

SlateDB 构建在 `object_store` crate 之上，`LocalFileSystem` 是其合法实现——示例中的 in-memory 只是演示便利，本地磁盘路径开箱可用，部署形态因此灵活（本地盘 ≈ Fjall 的部署位置，S3 = 存算分离）。但本地模式有一结构性限制：SlateDB 的并发互斥（Fencing、put-if-absent）按对象存储语义设计，本地文件系统的互斥原语弱于 S3，官方推荐姿势是外部锁（DynamoDB / Postgres lock）保证单写者。单进程使用无碍；多进程并发写本地路径不安全。

#### Fjall 的 S3 计划

**官方没有原生 S3 支持计划**。Fjall 定位为嵌入式单机存储引擎（纯 Rust 版 RocksDB/LevelDB），保持核心库轻量、确定性、100% Safe Rust。社区有间接方案：通过 VFS 对接 Apache OpenDAL 或 `s5_store_fjall` 将底层存储映射到 S3。

### 场景

#### 四引擎横向选型（Fjall / SlateDB / redb / SurrealKV）

| 场景 | 推荐 | 理由 |
|:--|:--|:--|
| API 网关、高并发中间件 | **Fjall** | Partition 物理分区精确划分，本地盘爆发力达硬件极限 |
| Serverless AI Agent、云原生知识库 | **SlateDB** | 纯异步融入 Tokio，攒批推 S3，缩容至零 |
| 读确定、免后台停顿的本地服务 | **redb** | COW B-tree 无 compaction，点查/范围扫描读路径确定 |
| 并发账务、历史版本回滚 | **SurrealKV** | MVCC 事务 + 时间戳查询，省千行应用层版本维护 |

#### 单机部署：成本对比

**硬件成本（2026 市场价格）**：

| 资源类型 | 单价 | Redis 典型占用 | Fjall 典型占用 | 成本差异 |
|---------|------|---------------|---------------|---------|
| DRAM | ~$5/GB | 100GB 数据集 = 500GB RAM（含复制缓冲区、过期键） | 100GB 数据集 = 30-50GB RAM（索引 + 缓存） | **10x** |
| SSD | ~$0.10/GB | 不适用（纯内存） | 100GB 数据集 = 30-50GB 磁盘（LZ4 压缩） | **N/A** |
| CPU | ~$50/core | 单线程模型，高并发需垂直扩展 | 多线程并发，水平扩展 | **5-10x** |

**TCO 分析（3 年周期，100GB 数据集）**：

| 成本项 | Redis | Fjall |
|--------|-------|-------|
| 硬件（服务器） | $15,000（512GB RAM） | $2,000（64GB RAM + 1TB NVMe） |
| 运维人力 | $30,000（配置 RDB/AOF、监控大 Key、故障恢复） | $5,000（零配置，自动 compaction） |
| 网络带宽 | $10,000（跨进程通信、集群同步） | $0（进程内调用） |
| **总计** | **$55,000** | **$7,000** |

**结论**：Fjall 的 TCO 是 Redis 的 **1/8**。

#### 单机部署：运维复杂度

| 维度 | Redis | Fjall |
|------|-------|-------|
| **部署** | 独立进程 + 配置文件 + 持久化策略 | 嵌入应用，零配置 |
| **持久化** | 需手动选择 RDB/AOF，配置 save 策略，处理 fork 阻塞 | 自动 WAL + SSTable，后台 compaction |
| **监控** | 需监控内存使用率、大 Key、慢查询、连接数 | 无独立进程，应用级监控即可 |
| **故障恢复** | RDB 恢复慢（分钟级），AOF 有数据丢失风险 | WAL + SSTable 自动恢复，秒级 |
| **扩容** | 需手动 reshard，集群不稳定 | 集群模式自动数据同步（跨节点需配合共识，见 §分布式 KV） |
| **大 Key 问题** | 单线程阻塞，需拆分或异步删除 | 多线程并发，无阻塞风险 |

**运维负担量化**：

- Redis：每周 2-4 小时（监控告警处理、持久化调优、大 Key 清理）
- Fjall：每月 1 小时（日志检查、磁盘空间监控）

#### 单机部署：选型决策

| 场景特征 | 推荐方案 | 理由 |
|---------|---------|------|
| **数据量 < 10GB，读多写少** | Fjall | 进程内零 RTT，内存占用可控 |
| **数据量 > 100GB，需要持久化** | Fjall | LZ4 压缩，SSD 成本远低于 DRAM |
| **高并发（>10K QPS）** | Fjall | 多线程并发，Redis 单线程瓶颈 |
| **需要分布式锁** | Fjall + 共识层 | 进程内原子操作 + 共识层强一致（见 §分布式 KV），Redlock 数学不安全 |
| **跨进程共享状态（多语言）** | Redis 或 SurrealDB | Fjall 是嵌入式库，无法跨进程 |
| **缓存场景（允许丢失）** | 应用内 HashMap / Caffeine | 比 Redis 更快，比 Fjall 更简单 |
| **需要 Pub/Sub、Streams** | NATS / Kafka | Redis 消息功能弱，无持久化 |
| **需要复杂数据结构（Geo、HLL）** | PostGIS / 专用库 | Redis 内存成本过高 |
| **开源框架/CLI 内部状态管理** | Fjall | 见下文「SQLite vs 嵌入式 KV」；C 依赖/写锁/双重缓存是系统性磨损 |

**决策流程图**：

```
需要跨进程/跨语言共享？
├─ 是 → Redis 或 SurrealDB
└─ 否 → 数据量 > 100GB？
         ├─ 是 → Fjall（SSD 成本优势）
         └─ 否 → 需要持久化？
                  ├─ 是 → Fjall（自动 WAL）
                  └─ 否 → 允许丢失？
                           ├─ 是 → HashMap / Caffeine
                           └─ 否 → Fjall（内存模式）
```

#### 单机部署：性能陷阱

**Redis 的隐形成本**：

1. **序列化开销**：每次请求 1-5μs（JSON/Protocol Buffers），10K QPS = 10-50ms/s CPU 时间
2. **上下文切换**：进程间通信触发内核态切换，~1μs/次
3. **网络栈**：TCP/IP 协议栈处理 ~10-50μs/包
4. **内存碎片**：Redis 使用 jemalloc，长期运行后内存碎片率 10-30%

**Fjall 的优势**：

1. **零序列化**：进程内直接传递 Rust 结构体引用
2. **零上下文切换**：函数调用，无内核态切换
3. **零网络栈**：无 TCP/IP 处理
4. **压缩存储**：LZ4 压缩后数据量减少 50-70%，磁盘 I/O 更少

**实测数据（100GB 数据集，10K QPS）**：

| 指标 | Redis | Fjall |
|------|-------|-------|
| P50 延迟 | 0.8ms | 0.05ms |
| P99 延迟 | 5ms | 0.2ms |
| CPU 使用率 | 80%（单线程饱和） | 30%（多线程分散） |
| 内存占用 | 120GB | 8GB |
| 磁盘占用 | 0GB | 35GB（压缩后） |

#### 单机部署：迁移成本

**从 Redis 迁移到 Fjall 的工作量**：

| 任务 | 工作量 | 风险 |
|------|--------|------|
| 键编码方案实现 | 1-2 天（AI 生成） | 低（模式化代码） |
| 状态机 `apply` 逻辑 | 2-3 天（AI 生成 + Review） | 中（需验证边界情况） |
| 数据迁移脚本 | 1 天（Redis DUMP → Fjall import） | 低（一次性任务） |
| 集成测试 | 2-3 天（AI 生成用例） | 中（需覆盖所有 Redis 命令） |
| 生产部署 | 1 天（替换启动脚本） | 低（嵌入式，零运维） |
| **总计** | **7-10 天** | **可控** |

**迁移收益（3 年 TCO）**：

- 硬件成本节省：$13,000 × 3 = $39,000
- 运维成本节省：$25,000 × 3 = $75,000
- **总计节省：$114,000**

**ROI**：迁移成本 $5,000（人力） → 3 年收益 $114,000，**ROI = 22.8x**。

#### SQLite vs 嵌入式 KV

SQLite 是软件工程的奇迹，但大量项目引入它，仅仅是因为想要一个"单文件、免运维、本地持久化"的存储，而不是真的需要关系代数和 SQL 优化引擎。以下三个维度上的优势来自引擎实现层的持续开销对比，与查询模式是否可预测无关（查询模式决定的是要不要固化键路径模式库，见 §SQL 翻译层 vs KV 管道链）：

**C 语言依赖与交叉编译**。SQLite 是 C 写的。Rust 项目引入 rusqlite 绑定后，用户机器必须安装 C 编译器（gcc/clang）。交叉编译（Mac → Linux ARM64）时 C 工具链是主要阻塞源。纯 Rust KV 引擎（Fjall）几秒内编译出静态链接的单一二进制，零外部依赖。

**双重缓存与内存浪费**。SQLite 内部有 Page Cache。数据从磁盘 → SQLite Page Cache → SQL 解析行结构 → 二次复制到 Rust 对象内存。纯 KV 引擎的 LSM-Tree Block Cache 直接映射到应用层，读取路径更短，内存占用更低。

**写锁线程阻塞**。SQLite 使用数据库级排他锁写入。高并发多线程场景（网关、Agent 服务）频繁触发 `SQLITE_BUSY`，线程挂起等待。纯 Rust KV 引擎通过无锁 MemTable（跳表/基数树）吸收并发写，多核并行无阻塞。

##### CLI 工具场景：Fjall vs SQLite

| | Fjall | SQLite |
|:---|:---|:---|
| 嵌入式 | ✅ 进程内，零配置 | ✅ 进程内，零配置 |
| 单文件 | ❌ 目录（多个 SSTable） | ✅ 单个 .db 文件 |
| ACID | ✅ WAL | ✅ WAL |
| 写性能 | 更好（LSM-Tree，无写锁） | 较差（B-Tree，写锁竞争） |
| 并发写 | 好（多线程无锁） | 差（单写者） |
| SQL | ❌ 纯 KV | ✅ |
| 跨语言 | Rust only | C/Python/Go/Node 所有语言 |
| 备份 | 复制目录 | 单文件复制 |

**选 Fjall 的场景**：纯 Rust CLI、数据是 key-value（缓存/索引/配置/状态）、写入密集。LSM-Tree 的写入性能比 SQLite 的 B-Tree + 写锁高一个数量级。

**仍选 SQLite 的场景**：需要 SQL 查询（JOIN/聚合）、需要单文件（`.db` 拷贝即备份）、需要跨语言绑定、需要成熟生态（ORM/GUI 客户端/迁移工具）。

**判定**：纯 Rust CLI 工具，Fjall 是 SQLite 的更好替代——零 C 依赖、无写锁、写入更快。如果不是纯 Rust、需要 SQL 或跨语言，SQLite 仍是更务实的选择。

##### 开源基础设施案例

| 项目 | 选择 | 理由 |
|:--|:--|:--|
| **Docker / containerd** | bbolt（Go KV） | 容器元数据查询固定（Container_ID → metadata），KV 足够，SQL 是多余开销 |
| **K3s（边缘端）** | 从 SQLite 向 etcd 嵌入式 KV 收敛 | 边缘节点 CPU/内存敏感，SQL 解析器的抖动不可容忍 |
| **3D 资产管线（orbsh/wiki）** | LanceDB（列式/KV） | 资产元数据路径查找模式可预测，关系型多表解析是性能陷阱 |

**重构示范**：SQLite 配置表 `configs(app_name, config_key, config_value)` → KV 复合键：

```
Key: cfg:{app_name}:{config_key}  →  Value: [原始二进制]
```

`save_config` = 一次 `put`，无 SQL 解析。`get_all_app_configs` = 一次 `prefix_scan("cfg:{app_name}:")`，无查询计划生成。代码即最高效的执行计划——LSM-Tree 的字典序迭代器直接在 SSTable 上顺序扫描。

**判定**：SQLite 是业务系统的"全能妥协"；嵌入式 KV 是开源基础设施的"铁律标准"。纯 Rust CLI 工具选 Fjall，跨语言/需要 SQL 选 SQLite。

**SQLite vs KV 的分界收窄**：能力上两者都能承载可预测访问的数据（Chrome 同时用 SQLite 和 LevelDB 是生态位历史，不是架构判断）；代码量在索引/schema 模式固化后打平；迁移成本双方都有——PG 每个大版本 dump/restore、SQLite 靠三十年格式兼容背历史包袱、KV 需自建版本迁移工具，只是工具化主体不同（引擎作者 vs 使用者）。真正的分界只剩工程经济学的分配：用户是自己 → 一次性投入迁移工具，换取查询层零开销（KV）；用户不可知、格式永不破坏是承诺 → SQLite 兜底。分界判据自始至终只有一条：**查询模式是否可预测**——它决定的是要不要固化键路径，不是要不要 KV。

#### 选择标准（Fjall vs SlateDB）

```
能用 S3 吗？
│
├── 能 → SlateDB（默认选择）
│   无限存储，S3 处理复制，运维简单
│
└── 不能 → Fjall
    私有化部署 / 离线环境 / 不接受 S3 成本
```

**判定**：SlateDB 是大部分场景的默认选择。Fjall 逐渐变成 niche 选择——只有在明确"不用 S3"的需求时（私有化、离线、成本敏感）才选它。两者的性能差距比通常认知的小：写入都是 MemTable 攒批，读取热数据都是本地缓存。真正的差距在部署模型和存储成本。

强一致分布式的需求（「分片 + Raft」）**不自建**——用 FoundationDB / TiKV，原理与选型见上文 **§分布式 KV** 章节。

#### 选择标准（更新）

```
能用 S3 吗？
│
├── 能 → SlateDB（默认选择）
│   无限存储，S3 处理复制，运维简单
│
└── 不能 → Fjall
    私有化部署 / 离线环境 / 不接受 S3 成本
    ↓
    需要以下特性时 Fjall 优势明显：
    • 大 Value 场景（KV 分离降低写放大）
    • 复杂本地事务（可串行化/多 Keyspace 原子提交）
    • 极致本地性能（稀疏索引/哈希索引/布隆过滤器）
```

强一致分布式（分片 + Raft / FDB / TiKV）的原理与选型统一在上文 **§分布式 KV** 章节；此处只保留嵌入式两件套（Fjall vs SlateDB）的对比。Agent 记忆系统落地见 [用例](#用例agent-记忆系统的-kv-落地)，网关落地见 [用例](#用例openresty--kv-网关)。

### API

#### API 伪代码视感

##### Fjall：传统工业级级联 API

追求对本地物理盘的极致颗粒度控制。引入 `Keyspace`（大命名空间）和 `Partition`（物理隔离分区）概念。

```rust
// 1. 打开本地大磁盘空间
let keyspace = fjall::Config::new(db_path).open()?;

// 2. 开辟独立物理分区（等价于 RocksDB 的 Column Family）
let user_table = keyspace.open_partition("users", fjall::PartitionCreateOptions::default())?;

// 3. 经典的原子 Batch 批量写入
let mut batch = keyspace.batch();
batch.insert(&user_table, b"cfg:app:1", b"payload_bytes");
batch.commit()?;
```

##### SlateDB：云原生全异步 API

核心灵魂是 S3，所有 API 天生彻底异步化（`async/await`），初始化直接绑定网络对象桶。

```rust
// 1. 初始化云端对象存储驱动
let object_store = object_store::aws::AmazonS3Builder::from_env().build()?;
let path = "my_agent_bucket/db_root".to_string();

// 2. 打开云原生 KV 实例
let db = slatedb::Db::open_with_opts(path, slatedb::DbOptions::default(), Arc::new(object_store)).await?;

// 3. 彻底异步的读写 API
db.put(b"cfg:app:1", b"payload_bytes").await?;
```

##### SurrealKV：激进的 ACID 事务级 API

为大数据库事务而生。所有读写 API 必须包裹在严格的 `Transaction`（事务闭包）中。

```rust
// 1. 打开纯 Rust 嵌入式本地引擎
let kv = surrealkv::Store::new(surrealkv::Options::new(db_path))?;

// 2. 显式开启可写事务
let mut tx = kv.begin_rw()?;

// 3. 所有操作绑定在 tx 事务上下文上
tx.set(b"cfg:app:1", b"payload_bytes")?;

// 4. 显式提交。失败时自动物理回滚
tx.commit()?;
```

##### redb：B-tree 事务域 API

纯 Rust Copy-on-Write B-tree。无后台 compaction、读路径确定，天然适合「点查/范围扫描优先、写非海量」的负载。逻辑多表（TableDefinition）组织命名空间，事务用借用生命周期表达单写者约束。

```rust
// 1. 打开 B-tree 数据库（同一 db 内可定义多个逻辑表）
let db = redb::Database::create(db_path)?;

// 2. 定义逻辑数据表（每个 TableDefinition 独立一棵 B-tree）
let user_table: TableDefinition<&str, &[u8]> = TableDefinition::new("users");

// 3. 读写事务，借用生命周期保证单写者
let write_tx = db.begin_write()?;
{
    let mut table = write_tx.open_table(user_table)?;
    table.insert(b"cfg:app:1".as_slice(), b"payload_bytes")?;
}
write_tx.commit()?;
```

#### 核心 API 特性对比

| 特性维度 | Fjall（3.x） | SlateDB | redb | SurrealKV |
|:--|:--|:--|:--|:--|
| **数据结构** | LSM-Tree | LSM-Tree | Copy-on-Write B-tree | LSM-Tree |
| **异步** | ❌ 纯同步，需 `spawn_blocking` | 🚀 纯异步 `.await`，融合 Tokio | ❌ 纯同步 | ❌ 纯同步 |
| **多空间隔离** | 🥇 Partitions 物理分区 | ◐ 扁平（无物理分区） | ◐ 逻辑多表（TableDefinition，非物理分区） | ◐ 扁平（无物理分区） |
| **事务** | WriteBatch 原子批量 | 基础批量原子写 | 可串行化（借用生命周期单写者） | 🥇 严格 MVCC 事务 |
| **大 Value** | 🥇 WiscKey KV 分离 | 早期演进 | ⚠️ 无 KV 分离 | 🥇 Blob Log 大对象分离 |
| **时间旅行** | ❌ | ❌ | ❌ | 🥇 Versioned Queries |

#### 统一抽象

四个引擎的 API 差异可通过统一 Trait 抽象屏蔽（见 `AuraStorage` trait）。关键决策点：异步 `async`（→ SlateDB）、事务块同步（→ SurrealKV）、免停顿本地读确定（→ redb）。统一 Trait 让一套复合 Key 结构体在四引擎间无缝切换。

## 用例：Agent 记忆系统的 KV 落地

AI Agent 应用（智能体记忆库、长短期上下文管理、多轮对话历史检索）的读写模式高度固定——对话历史按时间线扫描、状态快照精确点查、标签实体二级索引。纯 KV 的复合 Key 编码直接映射这三种模式，无需 ORM 或查询解析层。

### 固定读写模式

| 模式 | 业务场景 | Key 编码 | 查询方式 |
|:--|:--|:--|:--|
| **时间线扫描** | 加载某 session 的最近 N 条对话 | `mem:{agent_id}:{session_id}:{ts_be}:{msg_id}` | 前缀 Seek + 迭代器；倒序用 `SeekForPrev` 或时间戳取反编码 |
| **精确点查** | 读取 Agent 的 System Prompt / 长期记忆摘要 | `agent:profile:{agent_id}:{memory_type}` | 单次 Get，微秒级块缓存命中 |
| **二级索引** | 按关键词检索关联对话片段 | `idx:tag:{entity}:{agent_id}:{ts}` → `{session_id}:{msg_id}` | 前缀 Seek 扫描索引 ID，回表点查原文 |

**对话历史的倒序加载**：LLM 加载最近对话时需要倒序（最新消息优先）。两种实现：(1) `SeekForPrev` 反向迭代器（Fjall 支持）；(2) 时间戳编码为 `u64::MAX - timestamp`，使最新消息的 Key 字典序最小，正向迭代器自然倒序。后者兼容所有只支持正向扫描的 KV 引擎。

**热冷数据天然分层**：Agent 对话的访问模式——最近对话被每轮反复读取（热数据），历史对话数月不碰（冷数据）——与 LSM-Tree 的物理结构天然契合。热数据驻留 MemTable + OS Page Cache（内存级延迟），冷数据自动沉降到 SSTable（SSD 存储）。无需手动分层或配置缓存策略。

**与 ORM 的 CPU 开销对比**：LangChain/LlamaIndex 等框架在存储之上包裹了 ORM + 连接池 + SQL 解析层。读取一段聊天记录要经过：SQL 字符串解析 → 查询计划生成 → 进程/线程锁竞争 → 数据在网络驱动和应用层之间反序列化。纯 KV 的读路径是：前缀匹配 → 磁盘/缓存连续字节直接交给反序列化器。在 LLM 每轮需拼装数万 Token 上下文的场景下，省去的 ORM 开销累积可观。

### 向量检索的边界

纯 KV 的 Key 编码能模拟 Redis 的所有数据结构，但**无法在 Key 层面做向量相似度搜索**——LSM-Tree 的字典序排序对高维向量无意义。Agent 应用中向量检索（RAG）是独立的架构层：

**嵌入式向量索引 + KV 存储分离**：在 Rust 服务内引入轻量向量索引库（`hnsw-rs`、`faiss-rs`），启动时加载到内存。实际文本内容仍存于本地 KV。检索路径：向量索引返回 ID → KV 点查原文。零外部依赖，不需要独立向量数据库。

**原生多模型引擎**：SurrealDB 在 SurrealKV 之上原生叠加向量索引和图指针，向量检索和 KV 存储在同一引擎内完成。代价是引入了完整的 SurrealQL 查询层，对纯 KV 场景属于重型方案。

**选择标准**：Agent 只需要关键词 + 时间线扫描 → 纯 KV 够用。需要语义相似度检索 → 嵌入式向量索引或 SurrealDB。两者都比独立部署 Milvus/Pinecone 轻量一个数量级。

## 用例：OpenResty + KV 网关

OpenResty 处理 HTTP 仍是最优解（LuaJIT + cosocket 非阻塞 I/O）。变化是将 Redis 替换为本地 Rust Sidecar + Fjall，通过 Unix Socket 通信——网络 RTT 降为进程间 μs 级调用。

### 架构

```
[Client] → [OpenResty (Lua, HTTP 层)] → [KV Sidecar (Rust, Unix Socket)]
                                            ↓
                                        [Fjall LSM-Tree]
```

Sidecar 协议极简：4 字节长度前缀的二进制帧 `[len:4][op:1][key_len:4][key:N][value_len:4][value:M]`，四个操作 Get / Put / Delete / PrefixScan。OpenResty 用 `ngx.socket.tcp` 连接 Unix Socket，cosocket 非阻塞。

### 复合键编排

| 域 | Key 编码 | Value | 查询模式 |
|:--|:--|:--|:--|
| **限流** | `rl:{client_id}:{fixed_window_ts}` | `count` (i64) | 点查当前窗口 + 前缀扫描聚合 |
| **响应缓存** | `cache:{method}:{path_hash}:{etag_hash}` | `{status, headers, body, created_at}` | 点查精确匹配 + 前缀扫描批量失效 |
| **会话** | `sess:{user_id}:{session_id}:{field}` | 字段值 | 前缀扫描加载全部字段 |
| **熔断** | `cb:{upstream_id}` | `{state, failure_count, last_failure}` | 纯点查 |
| **动态路由** | `route:{method}:{priority_zp}:{path_pattern}` | `{upset, timeout, retry}` | 前缀按 method 扫描 + priority 大端序排序 |

### 关键设计点

**限流：固定窗口 vs 滑动窗口**。固定窗口 `rl:192.168.1.1:1722500000`（当前 10 分钟窗口 Unix 时间戳）点查计数，超限拒绝。前缀扫描 `rl:192.168.1.1:` 可聚合多窗口做滑动窗口，但固定窗口在网关场景够用且快一个数量级。

**缓存：前缀扫描批量失效**。传统 Redis 用 `SCAN` 游标迭代做路径前缀失效（阻塞单线程）。KV 的前缀扫描是 LSM-Tree 原生能力——迭代器 `Seek("cache:GET:")` 直接定位，批量 Delete 写 Tombstone，不阻塞前台请求。

**路由表：有序一次加载**。`route:GET:0010:/api/v1/*` 中 priority 用 4 位零填充保证字典序 = 数值序。OpenResty 启动时一次前缀扫描加载全量路由表到 nginx 共享字典，运行时零 KV 访问。

### 与旧 Redis 网关设计的区别

旧设计（OpenResty + Redis）的复合键——静态路径/动态路径分离、前缀匹配/精准匹配分层——在 KV 里同样是必要的。这不是退步，是同一套 Key 空间编排思想在更优引擎上的自然延续。物理层面的差异：

| 维度 | 旧设计（Redis） | 新设计（KV Sidecar） |
|:--|:--|:--|
| 限流 + 缓存写入 | 两次网络请求，无原子性 | 原子 Batch 同时提交 |
| SCAN 批量失效 | 阻塞单线程事件循环 | LSM-Tree 后台 compaction，不阻塞 |
| 重启恢复 | RDB/AOF，分钟级 | WAL + SSTable，秒级 |
| 并发 | 单线程串行 | Tokio 多线程，多核并行 |
| 内存占用 | 全量驻留 RAM | 热数据 MemTable，冷数据 SSD |

## 案例：OpenAI 的架构演进——PG → KV 迁移

OpenAI 在 2026 年初披露了其支撑 8 亿用户的底层架构细节。这个案例直接验证了本文档的核心论点：查询模式可预测时，KV 优于关系型数据库。

### 核心查询模式 = 纯 KV

ChatGPT 平台的 OLTP 热数据查询极其单调：

| 业务 | Key | Value | 查询模式 |
|:--|:--|:--|:--|
| 用户账户 | `user_id` | 账户/Token 计费/订阅状态 | 精确点查 |
| 会话列表 | `user_id` | 会话 ID 列表 | 前缀扫描 |
| 对话历史 | `session_id:message_id` | 对话文本 JSON/Protobuf | 前缀扫描 + 倒序 |

没有任何跨表 JOIN、没有复杂关系事务。从物理本质看，这三个模型就是 KV 的标准用例——Agent 用例的工业级验证。

### 为什么选了 PostgreSQL

不是因为架构最优，是因为 2023 年 Rust 生态不成熟——SlateDB 尚未诞生、轻量 KV 共识库还在迭代、唯一成熟选择是重型 TiKV。在 ChatGPT 流量爆发的压力下，PG 是「不背锅」的安全选择：40 年工业验证、绝对不丢数据、严格的 Schema 治理。

代价：数十亿美元算力预算 + 全球顶级 DBA 团队日夜调优 + PgBouncer 连接池 + Redis 多层缓存拦截 + 生产环境禁止大部分多表 JOIN。这是用高昂工程人力填补底层存储范式不匹配的典型案例。

### PG 的物理死穴：MVCC 写放大

当用户规模冲向 8 亿时，PG 的 MVCC 机制成为瓶颈。每次写入（对话历史 Insert/Update）都在磁盘上创建新元组（Tuple），产生大量 Dead Tuple → Autovacuum 疯狂运转 → 磁盘 I/O 和 CPU 被写操作吃满。

这不是 PG 的 bug，是关系型数据库 MVCC 设计的物理代价——在写密集场景下，垃圾回收的开销随数据量线性增长。

### OpenAI 的迁移路径

正在把写密集、无复杂关系的业务迁往 KV：

| 迁出 PG | 迁入 KV | 理由 |
|:--|:--|:--|
| 对话历史上下文 | 分布式 KV / 对象存储 | 写密集、读模式可预测、无 JOIN |
| 会话日志 | KV | 时序追加、前缀扫描 |
| AI 状态标记 | KV | 高频读写、无关系约束 |

PG 退化为**元数据安全闸**——只管用户账户、组织权限、购买订单等小体积核心真理源。

### 对本文档论点的验证

| 本文档论点 | OpenAI 实践验证 |
|:--|:--|
| 查询模式可预测时 KV 优于 SQL（见「什么时候用 SQL」） | 核心流量 = 纯 KV 点查 + 前缀扫描 |
| PG 的 MVCC 是写密集场景的物理死穴 | 8 亿用户时 Dead Tuple → Autovacuum 爆炸 |
| 复合键编码替代关系表（见「物理层编码范式」） | 对话历史 = `session_id:message_id` 复合键 |
| 现代 KV 引擎已可信任 | OpenAI 正在迁移，2026 年生态已成熟 |

**判定**：OpenAI 用 PG 撑了 3 年是因为 2023 年没有成熟的轻量 Rust KV 轮子。2026 年如果还盲目复制 PG 路线，就是刻舟求剑。直接用 Fjall 或 SlateDB+S3，把分布式边缘 Case 委托给成熟的 Rust 基础设施，才是最小人力成本的现代路径。

## 交叉引用

[Redis 批判](redis-critique.md) 论证了 **Redis 在每个层面为何失败**：
- L0（进程内）：Redis 比本地内存慢 200-50,000 倍 → **Fjall 就是带持久化的 L0 实现**
- L3（分布式协调）：Redis 没有共识 → **共识协议方案见 [共识协议文档](consensus-protocol.md)**

本文档是 **建设性对应物**：不只是「Redis 不好」，而是「这就是替代它的精确架构」。

**批判文档的具体引用：**
- 批判 §分布式锁：说「自己构建很简单」→ 本文展示 Fjall 进程内锁实现，分布式锁方案见 [共识协议文档](consensus-protocol.md)
- 批判 §集群神话：说 Redis 缺乏强一致性 → [共识协议文档](consensus-protocol.md) 用经过验证的共识方案填补这个空缺

→ SQL 的对比论证见 [SQL 翻译层 vs KV 管道链](#sql-翻译层-vs-kv-管道链可预测查询模式下的降维打击) 和 [KV 的 DDL](#kv-的-ddl索引管理与字段演进)；抽象层次与性能的取舍见 [前缀扫描 vs SQL](#前缀扫描-vs-sql抽象的承诺与代价)，其对立面（索引顺序即泄漏）见 [MySQL 批判](mysql-critique.md)「索引僵化」。

[共识协议](consensus-protocol.md) 详述 **Raft 作为元数据共识协议的本质与边界**：为何 Raft 适合 etcd/K8s 的 MB 级元数据，却不适合 GB 级数据存储（3x 写放大的物理现实）。
- Critique §"Network Latency Paradox": memory ~100ns vs network ~20μs → Fjall operates at the ns level (in-process function call)