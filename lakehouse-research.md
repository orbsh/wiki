# Lakehouse Research: 表格式与对象存储选型

## Delta Lake 的物理本质

Delta Lake 本质上是**分布式文件状态机**，不是传统数据库引擎。它将**事务隔离**和**脑裂防御**的重任完全卸载到底层 S3 存储层的条件语义（`PUT` with `If-None-Match`）。

这意味着对象存储不是"哑存储"，而是数据湖的**共识协议**。如果存储的一致性模型薄弱，Delta 日志就会断裂。详见下文[并发控制：为什么需要 Catalog](#并发控制为什么需要-catalog)章节。

---

## Overview

本文围绕**湖仓表格式选型**和**对象存储选型**展开，覆盖格式对比、场景断言、对象存储适配、并发控制机制、迁移路径。面向使用 Python/Polars/DuckDB 轻量栈的现代化团队，评估 LanceDB、Delta Lake、Iceberg 在不同场景下的适用性。

### 分析型负载全景

分析型负载（Analytical Workload）不是单一技术路线。OLAP 是其中一类——特指接受用户交互查询的分析型数据库。还有其他分析型数据库不走 OLAP 路线。

#### OLAP 数据库

接受用户提交的查询，引擎实时计算结果。延迟取决于数据量和查询复杂度，从亚秒到数分钟不等。适合探索性分析、BI 报表、数据科学家交互式查询。

**MPP 分析型数据库**

ClickHouse、Snowflake、Databend。列式存储 + 向量化执行引擎 + 分布式并行查询。查询延迟极低（亚秒级）、SQL 生态成熟。代价是需要独立集群运维，存储和计算耦合（或半耦合），扩展成本高。

**Lakehouse**

Delta Lake / Iceberg + 对象存储（S3/OSS）。把表格式叠加在廉价对象存储上，查询引擎（DuckDB/Polars/Spark）按需启动。目标是**用廉价对象存储替代专用集群存储，计算节点按需扩展**。存算分离是实现这个目标的手段，不是分类维度。存储成本极低（对象存储单价远低于 MPP 集群）、计算可弹性扩展。代价是查询延迟略高、需要管理 compaction 和元数据。

MPP 和 Lakehouse 的延迟/并发特征相似（都是 Ad-hoc 查询，秒级响应），区别在于存储成本和扩展方式：MPP 用专用集群存储，扩展靠加节点；Lakehouse 用对象存储，计算和存储独立扩展。

这也是 Lakehouse 更有前途的原因。对象存储（S3/OSS）相对文件存储的优势是碾压性的——成本低一到两个数量级、无限弹性扩展、原生多副本容灾、无需运维存储集群。对象存储的短板是延迟和元数据：单次请求延迟高于本地磁盘，目录列举（ListObjects）在海量文件下性能退化。但 OLAP 恰恰是对延迟不敏感的负载——用户提交查询后等待数秒到数分钟是可接受的。Lakehouse 只需解决元数据问题（通过表格式的 metadata layer 或 REST Catalog 提供高效快照和文件索引），即可继承对象存储对文件存储的全部碾压性优势。MPP 的专用集群存储在成本和扩展性上无法与之竞争。

**Apache Kylin（数据立方体/Cube）**：预计算所有维度组合的物化视图，理论优雅，实际两者的短处都很显眼——Cube 构建慢且存储爆炸，新增维度要重建 Cube，扩展性差。该技术已不再生长，处于技术生命周期的凋零期。

#### 流数据库（分析型数据库，非 OLAP）

RisingWave、Materialize。同为分析型数据库，但不是传统 OLAP——不接受用户交互查询，而是持续维护物化视图。SQL 声明式定义，数据库自动管理增量计算和状态。适合需要实时聚合但不愿编写流处理代码的场景。本质是**牺牲灵活性，换取性能、延迟和并发**——在数据写入时实时计算聚合结果，查询时直接读预计算表，延迟从分钟级降到毫秒级。

#### 流计算框架（管道层，非数据库）

Flink、Spark Streaming、Arroyo、Fluvio。通用流处理引擎，灵活但需要自行管理状态、窗口、exactly-once 语义。非数据库，属数据管道。适合需要复杂流处理逻辑的场景（CEP、多流 join、复杂窗口聚合）。

#### 核心区分

| 类型 | 是否数据库 | 是否接受用户查询 | 代表技术 |
|:---|:---|:---|:---|
| OLAP（MPP） | 是 | 是（交互式） | ClickHouse, Snowflake |
| OLAP（Lakehouse） | 是 | 是（交互式） | Delta Lake, Iceberg + DuckDB/Polars |
| 流数据库 | 是 | 否（维护物化视图） | RisingWave, Materialize |
| 流计算框架 | 否 | 否（管道） | Flink, Spark Streaming |

**⚠️ 常见混淆**：OLAP 是数据库的一个子类，不是"分析型负载"的同义词。RisingWave 是分析型数据库但不是 OLAP。

---

## ADR：选用 LanceDB 作为湖仓基础（轻量起步，远期红利）

> 本文档的 ADR 适用于**使用 Python/Polars/DuckDB 轻量栈、运行在容器环境中的现代化团队**。对于使用 Spark/Trino/Tableau 的传统 BI 团队，或 PB 级规模的数据平台，本文的选型结论不直接适用。更广泛的场景断言见下文[最终选型断言](#最终选型断言)。

### 1. 背景

在软件架构中，**"今天极致轻量、明天享受技术红利"**的决策取向，决定了项目是两周上线还是陷入无尽的运维泥潭。

对于**数万到数百万行**的数据资产，LanceDB 在纯 OLAP 场景下的当前劣势（压缩率或扫描吞吐量对比 Parquet）可以忽略。无论是 5 万行还是 200 万行，Polars 和 DuckDB 聚合耗时均在 **<50ms**，物理文件读取差异对用户不可感知。

### 2. 决策

**全面采用 LanceDB（Lance 格式）；放弃 Delta Lake/Iceberg。** 本决策限定于轻量容器化 Python 栈场景，不适用于传统 BI 栈或 PB 级平台。

### 3. 理由（为什么不是 Delta？为什么选 Lance？）

*   **今天（Day 1）：避开 OSS 兼容性泥潭与运维负担**
    *   **为什么不是 Delta**：`delta-rs` 1.6 在阿里云 OSS 上存在 `If-None-Match` 头部 bug，且 Delta 强依赖 Catalog 服务。
    *   **Lance 的优势**：为短生命周期容器提供**纯无状态、零服务**的运行方式。Append-only 机制实现无锁并发，彻底规避重型企业级技术栈。
*   **明天（Day 2）：零成本继承性能红利**
    *   6-12 个月后，随着 Lance 将底层优化（如 **BtrBlocks、自适应压缩协议**）推送到 Python 生态，系统将自动获得显著的 OLAP 性能提升，无需改动任何 Skill 代码或 DDL。
*   **后天（Day 3）：原生 AI/向量就绪**
    *   若财务 Skill 下季度引入**向量审计（GraphRAG / Semantic Search）**，底层存储已具备成熟的向量检索能力，无需存储层重构。

## Lance vs Vortex — 核心对比

| 特性 | Lance | Vortex |
|---|---|---|
| **定位** | AI/ML 原生格式：多模态数据、向量搜索、LLM 训练 | 下一代 Parquet 替代：通用高性能列式格式 |
| **文件结构** | 目录结构（Fragments + Manifest） | 单体文件（可配置内部布局） |
| **元数据** | Protocol Buffers | FlatBuffers（零解析，O(1) 尾部加载） |
| **AI 索引** | 内置向量（IVF_PQ, HNSW）+ 全文（BM25） | 无——聚焦编码 + 算子下推 |
| **版本控制** | 内置 MVCC + time travel | 无——依赖 Iceberg/表格式层 |
| **随机读** | 比 Parquet 快一到两个数量级（行地址 + 微块寻址） | 比 Parquet 快约一个数量级（级联自适应压缩 + 细粒度 ZoneMap） |
| **全扫描** | 中等——非核心设计目标 | 比 Parquet 快一个数量级左右（Clickbench 超越部分原生数据库格式） |
| **GPU** | 通过 Arrow 标准支持 | 原生 GPU 解压 + 零拷贝加载 |
| **治理** | LanceDB（商业） | LF AI & Data（完全中立） |

### 设计哲学
- **Lance**：行级寻址 + 自包含系统（MVCC、向量索引）。定位为轻量嵌入式数据库。
- **Vortex**："文件格式的 LLVM"（自适应编码、SIMD 对齐）。纯粹的存储格式，目标是 Iceberg File Format API。

## Lance 在分析性业务中的真实表现与权衡

对于纯分析性业务（报表、多维聚合、大宽表分析、全表扫描），Lance 格式呈现出明显的长板极长、短板明显的非对称特征。

### 三大长板

#### 1. 稀疏列随机读取（Point Lookup / Random Access）性能显著优于 Parquet

**Parquet 的痛点**：Parquet 的最小读取单元是 Row Group（行组，通常包含几万行）。如果你在一个上亿行的表里只想查询特定几行的分析数据（如 `SELECT * FROM order WHERE order_id = 1001`），Parquet 必须把该行所在的整个 Row Group 全部通过网络下载并解压，造成大量的网络和内存浪费。

**Lance 的改进**：Lance 重新设计了文件内部的页级（Page-level）元数据和索引布局，支持行级的随机快速存取。在针对特定维度、特定主键的稀疏过滤查询时，网络 I/O 吞吐量和速度比 Parquet 高出一个数量级以上。

#### 2. "分析+更新"混合场景下，零运维成本（不需要扫地）

**传统湖仓（Iceberg/Delta）的痛点**：在订单状态高频更新的分析业务中，要想保持白天查询不卡顿，必须在深夜用服务器算力执行 COMPACT 和 VACUUM 脚本来合并碎文件，否则白天查询会因积压过多补丁文件而出现严重延迟（如 50 秒）。

**Lance 的解决方案**：Lance 的底层元数据版本链由 Rust 驱动并直接嵌在文件内部。白天发生高频更新时，文件内部通过类似页覆写的机制完成处理，白天和晚上的读取性能天然保持高度稳定，无需在架构中引入额外的定时任务去合并和清理，极大精简了技术栈。

#### 3. 完美兼容未来 AI / 算法团队的特征分析（多模态分析天花板）

如果分析业务未来计划引入 AI 推荐、用户画像特征（Embeddings）、或订单流的相似度聚类分析。Lance 允许在同一张分析表里，左手跑传统的 SQL 过滤（DuckDB 查状态），右手跑 Rust 级别的向量相似度检索（Vector Search），这是 Parquet 做不到的。

### 两大短板

#### 1. 纯粹的吞吐量与压缩率略逊于成熟的 Parquet

**技术本质**：Parquet 经历了全球工业界超过 10 年的持续打磨（各种激进的位打包 Bit-Packing、字典编码 Dictionary Encoding 以及 ZSTD 极限压缩算法）。在大规模全表扫描（如 `SELECT status, SUM(amount) FROM table GROUP BY status`，需要硬啃几百 GB 数据的场景）下，Parquet 的二进制列式压缩率更高，单核执行引擎搬运连续 Parquet 字节流的吞吐量目前仍是行业天花板。

**Lance 的妥协**：为了照顾"高频随机更新"和"向量检索"，Lance 的文件物理布局比 Parquet 稍显松散，在大规模纯硬算的吞吐量上，比最纯粹的 Parquet 略逊一筹。

#### 2. 计算生态（计算引擎优化器）的成熟度差距

**Parquet/Iceberg 的生态**：目前几乎所有的现代数仓和查询引擎（包括 DuckDB 的核心研发团队、Polars 的优化器、ClickHouse/Databend 的底层内核）都把 90% 的精力花在了针对 Parquet 文件读取的"谓词下推（Predicate Pushdown）"和"多线程流式预取"的极致优化上。

**Lance 的生态**：虽然 DuckDB、Polars、DataFusion 目前都有原生插件可以直接 `scan_lance()`，但由于 Lance 是一种较新的格式，这些外部计算引擎的查询优化器在面对极其复杂的多表多层嵌套 Join 聚合时，对 Lance 内部索引的利用效率，往往不如对传统 Parquet/Iceberg 元数据树利用得那样成熟。

### 最终选型断言

#### 场景一：最传统的报表分析、财务对账、经营大宽表（T+1 模式）

**不要选 Lance，坚守基于 Parquet 的方案**（如 T+1 凌晨 Polars 去重合并脚本，或者阿里云的 OSS Tables）。在 T+1 且不再变动的场景下，Parquet 连续大文件的扫描性能和全生态支持率最高，DuckDB 读取仅需毫秒级。

#### 场景二：实时、高频更新、且包含点状特征查询的分析业务

**大胆尝试 Lance 格式**。它能让你在完全不启动任何 Catalog 服务（不需要自建本地 SQL、不需要 Polaris）的前提下，仅靠纯 Python 驱动在阿里云 OSS 上运行一个天生抗更新、查单条极快、零碎文件隐形计费账单的现代轻量化数据底座。

#### 场景对照（合并）

| 维度 | 选用 LanceDB（AI 原生栈） | 选用 Delta/Iceberg + Parquet（传统 BI 栈） |
|---|---|---|
| **数据类型** | AI 驱动（Embedding、长文本、多模态） | 纯结构化（数字、日期、字符串） |
| **团队技术栈** | Python / Polars / DuckDB | Spark / Trino / Tableau |
| **数据规模** | 数百 GB 至数 TB | 多 TB 至 PB |
| **运维偏好** | 零服务、无 Catalog | 企业级 Catalog + ACID 治理 |
| **BI 工具集成** | 仅自定义仪表盘 | Tableau、Power BI、传统 BI |

## 对象存储选型：MinIO vs RustFS vs Garage

面向容器化/自托管湖仓场景，三个候选方案：

| 维度 | MinIO | RustFS | Garage |
|---|---|---|---|
| **一致性模型** | 强一致（CP） | 强一致（CP） | 最终一致（AP/CRDT） |
| **Delta CAS 支持** | ✅ 成熟 | ✅ 原生 | 🔴 概率性（LWW 风险） |
| **S3 Tables / REST Catalog** | ❌ 不支持 | ✅ 原生（2026+） | ❌ 不支持 |
| **资源占用** | 高（Go 运行时） | 低（<100MB 二进制） | 低（<100MB 二进制） |
| **最佳场景** | 企业标准、零风险 | 轻量 + 湖仓一体 | 静态资源、归档、CDN |
| **Delta Lake 风险** | 无 | 低（代码库较新） | **高（日志损坏）** |

- **Garage 不适用的原因**：无主架构，CRDT + Dynamo 风格仲裁（AP 倾向）。仅在**静态集群无节点变动**时保证 Read-After-Write 一致性——在网络分区、重平衡或拓扑变更下，一致性退化为最终一致。存储层无原子 CAS；"Last Write Wins"静默覆盖事务日志。Delta 的 MVCC 状态在物理上分裂 → 数据损坏。
- **RustFS 的优势**：强一致性（CP 倾向），确定性元数据管理 / Raft 共识。原生 CAS（`If-None-Match` 返回严格 412），轻量二进制（<100MB），适合 sidecar/临时 Pod。使用**内联纠删码（Inline Erasure Coding）**保护数据，无需完整副本集的复杂性。**2026 新增：支持 S3 Tables 协议**，存储桶自身可作为 Iceberg REST Catalog，内置 Auto-Compaction，从"轻量对象存储"升级为"湖仓一体化存储"。
- **MinIO 的优势**：久经考验，企业标准。需要 >2GB 内存。

**推荐**：
- **稳妥选择**：MinIO（成熟、零风险）。
- **轻量选择**：RustFS（精简 S3 子集、CAS 就绪、适合紧凑容器）。
- **不推荐**：Garage（破坏 MVCC 保证，不用于 Delta/Iceberg）。

## 工业级数据分析 Skill 架构

### 1. Schema 定义策略

LanceDB 使用 Apache Arrow。显式 schema 定义对于空表和严格财务治理是**强制性**的。

| 数据类型 | 使用场景 | 推荐 |
|---|---|---|
| **财务金额** | 金额、余额 | `pa.float64()`（不用 `float32`，避免精度损失） |
| **时间/审计** | 时间戳、日期 | `pa.date32()`、`pa.timestamp('us', tz='Asia/Shanghai')` |
| **固定嵌套（Struct）** | 发票、明细行 | `pa.struct([...])`。支持**投影下推**（仅下载相关列） |
| **动态标签（Map）** | 不可预测的 JSON/标签 | `pa.map_(pa.string(), pa.string())`。防止上游 Schema 漂移错误 |

**初始化代码：**
```python
import pyarrow as pa
import lancedb

def initialize_blank_financial_lakehouse(db_uri: str, oss_options: dict):
    db = lancedb.connect(db_uri, storage_options=oss_options)
    if "financial_records" not in db.table_names():
        explicit_schema = pa.schema([
            pa.field("tx_id", pa.string(), nullable=False),
            pa.field("amount", pa.float64(), nullable=False),
            pa.field("year", pa.int32(), nullable=False),
            pa.field("month", pa.int32(), nullable=False),
            pa.field("invoice_detail", pa.struct([
                pa.field("invoice_no", pa.string()),
                pa.field("tax_rate", pa.float64())
            ])),
            pa.field("custom_tags", pa.map_(pa.string(), pa.string()))
        ])
        db.create_table("financial_records", schema=explicit_schema, storage_version="2.0")
```

### 2. 容器化 Skill 代码模板

- **延迟导入（Lazy Import）**：不在顶层导入 `polars`/`lancedb`。冷启动从 ~500ms 降至 <10ms。
- **零拷贝查询（Zero-Copy Query）**：使用 `table.to_lance().to_lazy()` 创建 Polars LazyFrame，无需下载数据。
- **盲追加（Blind Append）**：并发写入创建独立物理 fragment，避免锁竞争。

```python
import os
from datetime import datetime

# Configuration
OSS_OPTIONS = {
    "aws_access_key_id": os.environ.get("OSS_ACCESS_KEY_ID"),
    "aws_secret_access_key": os.environ.get("OSS_SECRET_ACCESS_KEY"),
    "aws_endpoint": "https://oss-cn-hangzhou.aliyuncs.com",
    "aws_region": "cn-hangzhou"
}
DB_URI = "s3://your-finance-bucket/lancedb_core_lake/"

def skill_data_sync_lancedb(incoming_pl_df):
    """将增量数据追加到 OSS，作为独立 .lance fragment。"""
    import polars as pl
    import lancedb
    
    # Ensure table exists
    initialize_blank_financial_lakehouse(DB_URI, OSS_OPTIONS)
    db = lancedb.connect(DB_URI, storage_options=OSS_OPTIONS)
    table = db.open_table("financial_records")
    
    # Add ingestion timestamp and append
    payload = incoming_pl_df.with_columns(pl.lit(datetime.now()).alias("ingested_at"))
    table.add(payload)
    return {"status": "success", "version": table.version}

def skill_analytical_query_lancedb(sql_filter: str):
    """通过 Polars LazyFrame 执行 AI 生成的 SQL 过滤。"""
    import polars as pl
    import lancedb
    
    db = lancedb.connect(DB_URI, storage_options=OSS_OPTIONS)
    table = db.open_table("financial_records")
    
    # Zero-copy LazyFrame（本地内存操作，0ms）
    lazy_lance = table.to_lance().to_lazy()
    
    # Apply filter (Data Skipping on OSS) and Aggregate
    # Note: pl.sql() allows executing SQL-like expressions directly
    result = (
        lazy_lance.filter(pl.sql(sql_filter))
        .group_by(["year", "month"])
        .agg(pl.col("amount").sum())
        .collect()
    )
    return {"status": "success", "data": result}
```

### 3. 维护与 Compaction

异步执行（如 CronJob）。LanceDB 的 MVCC 确保前台查询不受 compaction 影响。

```python
def skill_weekly_maintenance_compact():
    import lancedb
    db = lancedb.connect(DB_URI, storage_options=OSS_OPTIONS)
    table = db.open_table("financial_records")
    
    # Merge small fragments
    table.compact_files()
    # Cleanup old snapshots (save storage costs)
    table.cleanup_old_versions(older_than_days=7)
```

## Serverless Parquet-on-OSS + DuckDB — 无锁分析模式

### 适用场景

当 `delta-rs` 在非 AWS S3 兼容存储（如阿里云 OSS 条件写入问题、锁错误）上失败时，DuckDB 直接查询原生 Parquet 目录提供了更简单、更稳定的替代方案。DuckDB 的 `httpfs` 扩展对阿里云 OSS 有成熟支持，规避了 `delta-rs` 的编译、签名和文件锁问题。

### 架构

```
┌─────────────────────┐        ┌─────────────────────┐
│  增量写入            │        │   分析查询            │
│  (Python / Polars)  │        │  (DuckDB + httpfs)   │
└────────┬────────────┘        └────────┬────────────┘
         │                              │
         ▼                              ▼
  ┌──────────────────────────────────────────┐
  │  s3://bucket/finance_table/              │
  │    append_20260101_120000_111111.parquet │
  │    append_20260101_120000_222222.parquet │
  │    append_20260102_093000_333333.parquet │
  │    ...                                   │
  └──────────────────────────────────────────┘
```

**写入端**：Polars 将带时间戳的 `.parquet` 文件直接写入 OSS。无锁、无目录服务、无覆写——纯追加。
**查询端**：DuckDB 用通配符扫描 `*.parquet`，自动合并所有文件。投影下推和谓词下推最小化 S3/OSS I/O。

### 实现

#### 1. 增量写入（Polars → OSS）

```python
from datetime import datetime
import polars as pl

oss_options = {
    "aws_access_key_id": "YOUR_ACCESS_KEY",
    "aws_secret_access_key": "YOUR_SECRET_KEY",
    "endpoint": "oss-cn-hangzhou.aliyuncs.com",
    "region": "cn-hangzhou",
}

def sync_incremental_data(new_rows: pl.DataFrame):
    """将新 Parquet 文件追加到 OSS。设计上并发安全。"""
    ts = datetime.now().strftime("%Y%m%d_%H%M%S_%f")
    path = f"s3://your-bucket/finance_table/append_{ts}.parquet"
    new_rows.write_parquet(path, storage_options=oss_options)
    print(f"Synced: {path}")
```

#### 2. 分析查询（DuckDB → OSS）

```python
import duckdb

def run_monthly_summary():
    con = duckdb.connect()
    con.execute("""
        SET s3_endpoint='oss-cn-hangzhou.aliyuncs.com';
        SET s3_access_key_id='YOUR_ACCESS_KEY';
        SET s3_secret_access_key='YOUR_SECRET_KEY';
        SET s3_url_style='path';  -- 关键：阿里云 OSS 要求 path-style URL
    """)

    sql = """
        SELECT month_col, employee_id, SUM(amount) AS total_sum
        FROM 's3://your-bucket/finance_table/*.parquet'
        GROUP BY month_col, employee_id
        ORDER BY month_col DESC, total_sum DESC
    """
    return con.execute(sql).pl()
```

### 为什么适合财务数据

| 特性 | 收益 |
|---|---|
| **零运维** | 无 Delta Log 链、无 Catalog 服务、无 8181 端口守护进程 |
| **规避 delta-rs bug** | DuckDB 的 `httpfs` 在国内云厂商上有充分验证 |
| **天然写入并发** | 两次同时同步仅产生两个不同的 `.parquet` 文件——零锁竞争 |
| **列裁剪** | DuckDB 仅下载查询引用的列，而非整个文件 |

### Compaction 注意事项

随着时间推移，大量小 `.parquet` 文件累积。DuckDB 的性能因重复 HTTP 往返（网络延迟主导计算）而下降。

**缓解措施**：运行定期 compaction 脚本（每周/每月），读取所有文件，合并为单个大 Parquet 文件，然后删除碎片：

```python
# Compaction sketch
df_all = con.execute("SELECT * FROM 's3://your-bucket/finance_table/*.parquet'").pl()
df_all.write_parquet(
    "s3://your-bucket/finance_table/combined_v1.parquet",
    storage_options=oss_options
)
# Then safely remove historical append_*.parquet files
```

**安全提示**：删除碎片时，确保没有并发写入正在进行（例如，仅合并当天之前的文件，或使用临时重命名模式）。

## delta-rs 1.6 阿里云 OSS 条件写入失败的 Workaround

### Bug 描述

`deltalake` 1.6.0 在写入事务日志时硬编码了 `If-None-Match` HTTP 头部。阿里云 OSS 对此头部返回 `400 NotImplemented`，使 Python 包无法直接写入 OSS。社区推荐的 `AWS_COPY_IF_NOT_EXISTS` 环境变量在此版本无效。

### Workaround 配置

三个存储选项可强制 delta-rs 回退到标准 PUT 协议，绕过条件写入 bug：

```python
storage_options = {
    "aws_access_key_id": "YOUR_ACCESS_KEY",
    "aws_secret_access_key": "YOUR_SECRET_KEY",
    "aws_endpoint": "oss-cn-hangzhou.aliyuncs.com",
    "aws_region": "cn-hangzhou",
    # 核心修复：禁用条件写入，回退到普通 PUT
    "aws_unsafe_disable_conditional_write": "true",
    # 禁用虚拟主机风格（阿里云 OSS 优先使用 path style）
    "aws_virtual_hosted_style_tables": "false",
    # 跳过 V4 集群签名（避免不必要的认证往返）
    "aws_skip_signature_v4_cluster": "true",
}
```

### 验证

应用这些选项后，delta-rs 应能成功写入阿里云 OSS，不再出现 `400 NotImplemented` 错误。测试：

```python
import polars as pl
from deltalake import write_deltalake

df = pl.DataFrame({"id": [1, 2, 3], "val": ["a", "b", "c"]})
write_deltalake(
    "s3://your-bucket/test_delta/",
    df,
    storage_options=storage_options,
    mode="overwrite"
)
```

**注意**：禁用条件写入意味着并发写入同一 Delta 表可能产生冲突。此 workaround 适用于单写入者场景（如 append-only 增量同步）。多写入者场景下，原生 Parquet + DuckDB 模式更安全。

---

## 并发控制：为什么需要 Catalog

> 为什么不能简单用 `latest.json` 覆盖方案？Delta Lake 和 Iceberg 如何解决对象存储的并发控制问题。

### 问题：latest.json 覆盖方案的两个致命缺陷

假设方案：在 OSS/S3 里放一个 `latest.json` 文件，每次更新用新指针覆盖它。技术上可行，早期很多自建轻量数仓都这么干过。但对象存储有两个无法逾越的物理铁律。

**缺陷一：Lost Update**——两个写入脚本同时覆盖 `latest.json`，对象存储无法提供行级锁，最终结果取决于网络延迟，旧内容可能后发先至覆盖新数据，新写入的物理文件变成永久孤儿。Catalog 的价值：通过内存锁、Raft 共识或数据库事务在控制面排队，保证绝对不丢数据。

**缺陷二：缓存污染**——高频覆盖同一文件名导致 CDN/网关缓存同步出现最终一致性延迟，查询引擎拿到的版本错乱。

### 两种解决方案

**路线 A：Delta Lake**——永远不覆盖任何文件，用 `PutIfAbsent` 原子操作做单向追加。代价是长长的日志链拖慢查询（50 秒延迟的根因）。

**路线 B：Iceberg**——把指针覆盖踢给外层 Catalog。Catalog 在内存或数据库事务里瞬间完成指针切换，查询直接读最新快照。OSS Tables / S3 Tables 把 Catalog 内化为存储桶原生能力，消除了自建 Catalog 的部署门槛。

### 核心对比

| 维度 | latest.json 覆盖 | Delta Lake | Iceberg + Catalog |
|:---|:---|:---|:---|
| 并发控制 | 无（Lost Update） | PutIfAbsent 原子操作 | Catalog 内存锁/数据库事务 |
| 文件策略 | 覆盖同一文件 | 单向追加不覆盖 | 不重名不覆盖 + 外层指针切换 |
| 查询性能 | 快（但会丢数据） | 慢（扫描长日志链） | 快（直接读最新指针） |
| 高频更新 | 不可用 | CoW 写放大严重 | MoR 近零写放大 |
| 架构复杂度 | 最低 | 中等（无需 Catalog） | 传统：需自建 Catalog；S3 Tables 后：零部署 |

### 架构洞察

`latest.json` 覆盖方案的本质是**在数据面对象存储内部做并发控制**，这违反了对象存储"高并发、大吞吐、不可变"的底层硬件设计初衷。Iceberg + OSS Tables / S3 Tables 把 `latest.json` 的指针文件从 OSS 慢速磁盘挪到表格桶网关层的超高速分布式内存缓存中——既实现"一步到位找指针"，又通过网关层上锁避开高并发写入撞车。

### 关键认知

1. **对象存储的设计初衷**：高并发、大吞吐、不可变。在数据面做并发控制违反这个初衷。
2. **Delta vs Iceberg 的本质区别**：Delta 在存储层用 PutIfAbsent 解决并发（代价是长日志链拖慢查询）；Iceberg 把并发控制踢给外层 Catalog（代价是需要额外服务，但查询快）。
3. **OSS Tables / S3 Tables 的价值**：把 Catalog 做成对象存储的原生能力，让用户不用自己部署 Catalog 服务。
4. **S3 Tables 改变了选型权重**：此前 Iceberg 最大痛点是"需要自建 Catalog"。S3 Tables / OSS Tables 把 Catalog 内化为存储桶原生能力，让 Iceberg 在大多数场景下成为默认选择，Delta 退守 append-only 细分场景。

---

## 迁移案例：Delta → Iceberg（高频更新场景）

> 订单系统高频点状更新场景下，Delta Lake CoW 模式导致 50s 查询延迟的根因分析与迁移方案。

### 根因：Delta CoW 的两个死穴

**写放大**：更新 1 条数据 → 读入整个 200MB 文件 → 重写出新 200MB 文件。CoW 固有缺陷。

**碎文件爆炸**：频繁重写导致 OSS 积压大量无序小文件 + 极长元数据日志。DuckDB 全表扫描被迫发起 60+ 次 HTTP 请求串行下载，50 秒延迟的绝大部分死在与 OSS 建立网络握手和解析海量元数据上，而非实际计算。

### 迁移方案：Iceberg MoR + OSS Tables

- **MoR 根治写放大**：更新时不碰原始文件，仅追加几 KB 的 Positional Delete File + 一条新记录，写入毫秒级完成
- **REST Catalog 消除元数据瓶颈**：OSS Tables 内置 Iceberg REST Catalog，查询引擎直接请求 REST API 获取当前快照，无需遍历目录
- **Auto-Compaction 委托云端**：阿里云后台自动调度合并，查询永远命中干净 Parquet 大文件

### 什么时候不该迁移

| 场景 | 选 Delta | 选 Iceberg |
|:---|:---|:---|
| 时序日志/事件流（只追加不更新） | ✅ PutIfAbsent 足够，无需 Catalog | 过度工程 |
| 批量 ETL 一次性灌入后只读 | ✅ 写入简单 | 差异不大 |
| IoT 传感器数据（高频写入，极少更新） | ✅ 追加天然适配 CoW | 可选但非必要 |
| 订单/用户状态（高频更新+删除） | ❌ CoW 写放大致命 | ✅ MoR 正解 |
| 时间旅行+行级删除的审计场景 | ❌ 日志链查询慢 | ✅ 快照隔离+谓词下推 |

**判断原则**：数据写入后基本不修改，Delta 更简单。只有当更新/删除成为常态时，Iceberg 的架构投入才值得。

**当前选型比例**（2026-06-23）：现代化团队的典型分配——Iceberg 50%（主力）、LanceDB 30%（向量+多模态，受阿里云 OSS 兼容性限制）、Delta Lake 20%（append-only）。

### 落地路线图

1. **测试验证**：开通 OSS Tables 邀测 → 创建测试表格桶 → Python 模拟订单高频更新写入 MoR 格式 Iceberg
2. **基准对撞**：DuckDB 通过 REST Catalog 读取表格桶 → 验证频繁更新后全表扫描聚合是否从 50s → sub-second
3. **正式割接**：生产环境写入链路改造为 Iceberg 追加 + 开启云端自动合并

### 关键认知

- **MoR 的代价**：读取时需要合并原始文件 + Delete 补丁，查询有额外开销。但 Auto-Compaction 在后台持续把补丁合回主文件，保证查询永远命中干净大文件。MoR 是"写时省、读时稍贵但被后台治理抹平"——对高频更新场景净收益巨大。
- **REST Catalog 是 Iceberg 的标准化方向**：不再依赖 Hive Metastore / Glue / 自建锁服务，存储桶自身化身元数据中心，架构最简。
- **OSS Tables ≈ S3 Tables**：阿里云 OSS Tables 和 AWS S3 Tables 是同一理念——对象存储桶内置 Iceberg REST Catalog + Auto-Compaction，把湖仓治理下沉为存储原生能力。

---

**交叉引用**：
- **[统一数据层](unified-data-layer.md)**：SurrealDB 的计算下推原则与对象存储的共识协议本质同源——减少不必要的中转，让逻辑在数据所在的地方执行。
- **[Redis 批判](redis-critique.md)**：架构批判方法论——从"架构鼓励什么"的视角审视技术选型，而非"工具没问题开发者用错了"。
- **[反应式架构](reactive-architecture.md)**：Lakehouse 的 Catalog 层管理数据版本和快照，与反应式架构的版本号接口思路一致——把版本治理从存储层提升到控制面。
