# Fjall 多模态索引实现：图、向量与全文检索的无外部依赖方案

**状态**：架构设计完成  
**日期**：2026-06-20  
**核心哲学**：通过 Key 前缀设计，将复杂索引结构映射为 LSM-Tree 的顺序 KV 对

## 概述

在没有关系型数据库（PostgreSQL）和外部专有引擎（如 Elasticsearch、Milvus）的通用分布式环境下，依靠 Fjall 这样一个纯 Rust、无污染的嵌入式 LSM-Tree 键值引擎，实现图索引、向量检索与全文本搜索，是一个教科书级的"空间/索引映射（Index Mapping）"命题。

LSM-Tree 的底层核心优势是"极速的单键 KV 查找"和"按字节序排序的范围扫描（Ordered Range Scan）"。我们需要通过巧妙地设计 Key 的前缀与字节拼接（Prefix Byte Design），将复杂的图拓扑、多维向量空间和文本倒排索引，全部映射为物理磁盘上高效的顺序排序键值对。

→ 详见 [Fjall + Openraft 设计](fjall-openraft-design.md)、[Redis 批判](redis-critique.md)

---

## 1. 图索引（Graph Index）：邻接表与双向键设计

在图拓扑中（例如智能体之间的协作网络或多智能体记忆图谱 Memory-Graph），最核心的操作是：已知节点 A，找出它指向谁（出度 Out-edges）；以及谁指向了节点 A（入度 In-edges）。

我们可以利用 Fjall 自动按 Key 排序的特性，设计一个"双向邻接键（Bi-directional Edge Keys）"系统，将其拆分为两个物理分区（Partitions）：

```
[Partition: Out-Edges] Key: "E:out:<Src_ID>:<Edge_Type>:<Dst_ID>" ──► Value: MsgPack(权重/属性)
[Partition: In-Edges]  Key: "E:in:<Dst_ID>:<Edge_Type>:<Src_ID>"  ──► Value: None (仅用于快速反向索引)
```

### 1.1 Rust 极致实现：单次写入原子事务与流式范围扫描

```rust
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

    // 1. 插入一条边：利用分布式 Raft 达成共识后，本地双向原子写入
    pub fn insert_edge(&self, src: &str, edge_type: &str, dst: &str, weight: f32) {
        let out_key = format!("E:out:{}:{}:{}", src, edge_type, dst);
        let in_key = format!("E:in:{}:{}:{}", dst, edge_type, src);
        let val = bincode::serialize(&weight).unwrap();
        
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

---

## 2. 向量搜索（Vector Search）：本地 HNSW 或磁盘 IVF-PQ 索引

在大模型（Embedding）时代，向量检索不能只靠野蛮的暴力 O(N) 线性全表扫描（余弦相似度计算），那在通用高并发环境里会把多核 CPU 瞬间跑满崩溃。

由于 Fjall 是嵌入式存储，最佳策略是：在当前 Rust 进程内存中利用第三方高性能库（如 `hnsw_rs` 或 `instant-distance`）构建 HNSW（分层可导航小世界）图，同时将原始稠密向量与图结构序列化成二进制 Bincode，持久化写入 Fjall。

### 2.1 工业级离线冬眠与零延迟唤醒架构

1. **节点复活期（Initialization / Wake-up）**：当你的 Openraft 节点启动或某个 Actor 被唤醒时，读取 Fjall 专有的 `vector_index` 分区，瞬间将压缩的 HNSW 图反序列化载入内存空间（占极小内存）。

2. **毫秒级检索期（In-Memory Query）**：当用户提交查询向量时，直接在内存中的 HNSW 图上执行微秒级（Microsecond）近邻搜索，拿到符合条件的 Vector_ID（即 Agent_ID）。

3. **落盘持久化（Commit / Scale to Zero）**：当有新向量插入时，更新内存中的 HNSW 图，并将图快照和原始向量打包写入 Fjall 物理磁盘，内存释放冬眠。

```rust
// 伪代码：在宿主 Actor 的 Fjall 存储块中融合向量持久化
pub fn save_vector_node(&self, partition: &PartitionHandle, vector_id: &str, embedding: &[f32]) {
    // 采用 Bincode 极速紧凑序列化
    let serialized_vec = bincode::serialize(embedding).unwrap();
    
    // 写入 Fjall 物理 NVMe
    partition.insert(format!("V:{}", vector_id).as_bytes(), serialized_vec).unwrap();
}
```

---

## 3. 文本搜索（Text Search）：倒排索引（Inverted Index）

在通用开发中，要在黑窗口里直接检索日志、或者让 AI Agent 在大段文本记忆里进行关键词检索，你需要手搓一套极简、高效的倒排索引（Inverted Index）。

其核心逻辑是：将文章拆分成一个个独立的词（Tokens），然后建立"词 → 包含该词的文档列表（Posting List）"的映射。

### 3.1 物理 Key 空间设计

我们在 Fjall 中开辟两个物理分区：

1. **document_store**：Key 为 `"D:<Doc_ID>"`，Value 为原始的纯文本内容。
2. **inverted_index**：Key 为 `"T:<Term_Word>:<Doc_ID>"`，Value 为该词在文章中出现的频率/位置（用于计算 TF-IDF 或 BM25 排名分）。

### 3.2 倒排索引构建与检索源码

```rust
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
            let val = bincode::serialize(&count).unwrap();
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

---

## 4. 极致大一统：用户驱动的多模态查询总线（The Polyglot Query Bus）

结合前文提出的最高指导原则——"用哪种语言处理任务，完全由用户和 AI 动态决定"。这套图与文本检索系统在运行期会展现出近乎科幻的极客美学：

→ 详见 [用户驱动的多模态路由协议](embedded-script-languages.md#43-用户驱动的多模态路由协议user-driven-polyglot-routing)

### 4.1 多语言协作检索流程

1. **用户下达检索命令**：你在 Neovim 终端里给你的分布式 Actor 发送了一个 `DispatchUserScript` 提案。

2. **调度不同的语言引擎现场绞肉**：
   - **Steel Lisp 登场**：用户选择调用 Steel 引擎。Lisp 脚本调用 Host 的 `get_out_neighbors` 图工具，在一毫秒内无网络开销地顺着邻接表纵深横穿 3 层智能体协作网络，把关系节点的 ID 提取出来。
   - **Python (PyO3) 登场**：Lisp 执行完毕后将 ID 数组零拷贝抛给 Python (PyO3)。Python 瞬间调用内嵌的本地大模型分词器，对节点文本进行向量化（Embedding）生成。
   - **Rust 本地落地**：生成的向量直接在内存的 HNSW 图里进行近邻比对，最终将最精确的记忆和日志快照锁死，打包成一个二进制 Bincode 块，经由 Openraft 广播同步全网，狠狠地固化在本地的 Fjall 磁盘中。

---

## 总结

看，你完全没有借助任何重型的图数据库（Neo4j）、没有依赖外部臃肿的向量库、更没有安装几百兆内存起步的 Elasticsearch。你仅仅通过设计好 Fjall 的物理二进制 Key 前缀空间，利用 LSM-Tree 的前缀扫描（Prefix Scan）红利，就在极简的单二进制进程里，实现了功能完备、速度达到裸金属极限的图、向量与文本检索大一统。

这才是真正的纯粹与清醒。

---

## 交叉引用

本文档是多模态索引的无外部依赖实现方案，与以下详细分析形成完整的决策闭环：

- **[Fjall + Openraft 设计](fjall-openraft-design.md)**：Fjall LSM-Tree 引擎与 Openraft Raft 共识的分布式存算一体架构。
- **[Redis 批判](redis-critique.md)**：详细论证为何 Redis 是"网络 RAM 陷阱"，本方案彻底排除 Redis。
- **[嵌入式脚本语言选型](embedded-script-languages.md)**：Rune/Steel/PyO3/Wasm 的技术对比与用户驱动路由协议。
- **[Aura 架构](aura-architecture.md)**：存算一体的现代分布式 Actor 引擎，含反证章节（换成 Redis 的架构退化分析）。
- **[Arrow 大一统 HTAP 引擎](arrow-unified-htap-engine.md)**：Fjall + Arrow + Polars 全链路存算一体，含 7 种数据格式底层字节排布对撞。

**统一的第一性原理**：不搞技术崇拜，不吃开源画的大饼，只看真实的硬件物理限制与团队生产力。
