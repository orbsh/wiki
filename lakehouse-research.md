# Lakehouse 研究

## 1. 分析型负载全景

分析型负载不等于 OLAP。OLAP 是其中一类——特指接受用户交互查询的分析型数据库。还有其他分析型负载不走 OLAP 路线。

| 类型 | 是否数据库 | 是否接受用户查询 | 代表技术 |
|:---|:---|:---|:---|
| OLAP（MPP） | 是 | 是（交互式） | ClickHouse, Snowflake |
| OLAP（Lakehouse） | 是 | 是（交互式） | Delta Lake, Iceberg + DuckDB/Polars |
| 流数据库 | 是 | 否（维护物化视图） | RisingWave, Materialize |
| 流计算框架 | 否 | 否（管道） | Flink, Spark Streaming |

**MPP**（ClickHouse、Snowflake）：列式存储 + 向量化引擎 + 分布式并行查询。延迟极低（亚秒级），但存储和计算耦合，扩展靠加节点，成本高。

**Lakehouse**（Delta Lake / Iceberg + 对象存储）：把表格式叠加在廉价对象存储上，查询引擎（DuckDB/Polars/Spark）按需启动。存算独立扩展，存储成本低一到两个数量级。代价是查询延迟略高、需要管理 compaction。

**流数据库**（RisingWave、Materialize）：持续维护物化视图，写入时实时计算，查询时直接读预计算表。延迟从分钟级降到毫秒级。不是 OLAP。

**流计算框架**（Flink、Spark Streaming）：通用流处理引擎，需要自行管理状态和窗口。非数据库，属数据管道。

**Apache Kylin（数据立方体/Cube）**：预计算所有维度组合的物化视图。理论优雅，实际短处明显——Cube 构建慢且存储爆炸，新增维度要重建 Cube，扩展性差。该技术已不再生长，处于技术生命周期的凋零期。

**⚠️ 常见混淆**：OLAP 是数据库的一个子类，不是"分析型负载"的同义词。RisingWave 是分析型数据库但不是 OLAP。

---

## 2. 为什么选 Lakehouse

对象存储（S3/OSS）相对文件存储的优势是碾压性的：成本低一到两个数量级、无限弹性扩展、原生多副本容灾、无需运维存储集群。

对象存储的短板是延迟和元数据——单次请求延迟高于本地磁盘，目录列举在海量文件下性能退化。但**分析型负载恰恰是对延迟不敏感的**：用户提交查询后等待数秒到数分钟完全可接受。Lakehouse 只需解决元数据问题（通过表格式的 metadata layer 或 REST Catalog），即可继承对象存储的全部优势。

MPP 的专用集群存储在成本和扩展性上无法与之竞争。**Lakehouse 可能是分析型架构的终局。**

---

## 3. 核心机制：并发控制

> 对象存储的铁律决定了所有湖仓表格式的设计空间。

### 对象存储的本质

对象存储的设计初衷是：**高并发、大吞吐、不可变**。它没有行级锁、没有文件锁、没有原子覆盖。在数据面做并发控制违反这个初衷。

### 为什么 latest.json 不行

假设在 OSS 里放一个 `latest.json`，每次更新用新指针覆盖它：

- **Lost Update**：两个写入脚本同时覆盖，对象存储无法提供行级锁，旧内容可能后发先至，新写入的文件变成永久孤儿。
- **缓存污染**：高频覆盖同一文件名导致 CDN/网关缓存同步出现最终一致性延迟，查询引擎拿到的版本错乱。

### 两种解决方案

| 维度 | Delta Lake | Iceberg + Catalog |
|:---|:---|:---|
| 并发控制 | PutIfAbsent 原子操作 | Catalog 内存锁 / 数据库事务 |
| 文件策略 | 单向追加不覆盖 | 不重名不覆盖 + 外层指针切换 |
| 查询性能 | 慢（扫描长日志链） | 快（直接读最新快照） |
| 高频更新 | CoW 写放大严重 | MoR 近零写放大 |
| 复杂度 | 中等（无需 Catalog） | 需要 Catalog 服务（或 S3 Tables 内置） |

**Delta**：永远不覆盖任何文件，用 PutIfAbsent 原子操作做单向追加。代价是长日志链拖慢查询。

**Iceberg**：把指针覆盖踢给外层 Catalog。Catalog 在内存或数据库事务里瞬间完成指针切换，查询直接读最新快照。

**OSS Tables / S3 Tables 的价值**：把 Catalog 做成对象存储的原生能力——存储桶自身化身元数据服务中心，用户不用部署任何额外服务。S3 Tables 改变了选型权重：此前 Iceberg 最大痛点是"需要自建 Catalog"，现在 Catalog 被内化为存储原生能力。

---

## 4. 表格式选型

### 核心对比

| 特性 | Delta Lake | Iceberg | Lance | Vortex |
|:---|:---|:---|:---|:---|
| **存储机制** | Append-only 日志 + Parquet | 快照 + Manifest + Parquet | 自包含目录（Fragments + Manifest） | 单体文件（可配置内部布局） |
| **并发控制** | PutIfAbsent（存储层） | Catalog 指针切换 | 内置 MVCC | 无（依赖表格式层） |
| **写入模式** | CoW（写放大）/ MoR | MoR（追加为主） | Append-only | 无原生写入 |
| **随机读** | 慢（日志链） | 快（快照直达） | 极快（行级寻址，比 Parquet 快 10-100x） | 快（级联自适应压缩） |
| **全扫描** | 快（Parquet 压缩） | 快（Parquet 压缩） | 中等 | 极快（比 Parquet 快一个数量级） |
| **AI/向量** | 无 | 无 | 内置（IVF_PQ, HNSW, BM25） | 无 |
| **元数据格式** | JSON 日志 | Protocol Buffers | Protocol Buffers | FlatBuffers（O(1) 尾部加载） |
| **GPU** | 通过 Arrow | 通过 Arrow | 通过 Arrow | 原生 GPU 解压 + 零拷贝 |
| **Catalog 依赖** | 可选 | 必须（或 S3 Tables 内置） | 无（自包含） | 无 |
| **生态成熟度** | 高（Spark/Databricks） | 高（Spark/Trino/DuckDB） | 中（Python/Rust 生态好，大数据引擎支持早期） | 低（LF AI & Data 项目） |

### 选型决策

**选用 Lance 的决策推理（ADR）**：

本决策限定于**使用 Python/Polars/DuckDB 轻量栈、运行在容器环境中的现代化团队**。对于使用 Spark/Trino/Tableau 的传统 BI 团队，或 PB 级规模的数据平台，本文的选型结论不直接适用。

对于**数万到数百万行**的数据资产，Lance 在纯 OLAP 场景下的当前劣势（压缩率或扫描吞吐量对比 Parquet）可以忽略。无论是 5 万行还是 200 万行，Polars 和 DuckDB 聚合耗时均在 **<50ms**，物理文件读取差异对用户不可感知。

三阶段推理：

- **今天（Day 1）：避开 OSS 兼容性泥潭与运维负担**——Delta Lake `delta-rs` 1.6 在阿里云 OSS 上存在 `If-None-Match` 头部 bug，且 Delta 强依赖 Catalog 服务。Lance 为短生命周期容器提供纯无状态、零服务的运行方式，Append-only 机制实现无锁并发。
- **明天（Day 2）：零成本继承性能红利**——6-12 个月后，随着 Lance 将底层优化（如 **BtrBlocks、自适应压缩协议**）推送到 Python 生态，系统将自动获得显著的 OLAP 性能提升，无需改动任何 Skill 代码或 DDL。
- **后天（Day 3）：原生 AI/向量就绪**——若财务 Skill 下季度引入**向量审计（GraphRAG / Semantic Search）**，底层存储已具备成熟的向量检索能力，无需存储层重构。

**Iceberg（主力，50%）**：高频更新 / 删除场景的正解。MoR 根治写放大，REST Catalog 消除元数据瓶颈。需要 Catalog 服务（Lakekeeper、Polaris、或 S3 Tables 内置）。

**Lance（30%）**：AI/ML 原生场景首选。同一张表里左手跑 SQL 过滤，右手跑向量检索。零 Catalog、零服务，适合容器化轻量栈。短板：纯全表扫描吞吐量不如 Parquet，大数据引擎（Spark/Trino）支持早期。

**Delta Lake（20%）**：append-only 批量写入场景。写入简单，无需 Catalog。高频更新场景 CoW 写放大致命，不适用。

**Vortex**：下一代 Parquet 替代格式。纯存储格式，目标是 Iceberg File Format API。目前生态尚早。

**设计哲学**：
- **Lance**：行级寻址 + 自包含系统（MVCC、向量索引）。定位为轻量嵌入式数据库。
- **Vortex**："文件格式的 LLVM"（自适应编码、SIMD 对齐）。纯粹的存储格式，目标是 Iceberg File Format API。

### Lance 的详细权衡

Lance 在分析性业务中呈现明显的非对称特征：

**长板：**

1. **稀疏列随机读取性能显著优于 Parquet**：Parquet 最小读取单元是 Row Group（几万行），查特定几行必须下载整个行组。Lance 支持行级随机存取，在稀疏过滤查询时 I/O 速度比 Parquet 快一个数量级以上。

2. **"分析+更新"混合场景零运维**：Iceberg/Delta 在高频更新后必须深夜执行 COMPACT 合并碎文件，否则白天查询严重延迟。Lance 的元数据版本链由 Rust 驱动嵌在文件内部，白天高频更新时读取性能天然稳定，无需额外定时任务。

3. **多模态分析天花板**：同一张表支持 SQL 过滤 + 向量相似度检索 + 全文搜索，避免在数仓和向量数据库之间搬运数据。

**短板：**

1. **全表扫描吞吐量略逊于 Parquet**：Parquet 经过 10+ 年打磨，列式压缩率（Bit-Packing、字典编码、ZSTD）目前仍是行业天花板。Lance 为照顾高频更新和向量检索，文件布局稍松散。

2. **计算引擎优化器成熟度差距**：DuckDB/Polars 对 Parquet 的谓词下推和多线程预取优化远比对 Lance 成熟。复杂多表嵌套 Join 聚合时差距明显。

**结论**：
- 纯 T+1 报表分析 → 坚守 Parquet（Lance 没有优势）
- 高频更新 + 点状查询 → 大胆用 Lance
- AI/向量场景 → Lance 是唯一选择

### 场景对照

| 维度 | 选用 LanceDB | 选用 Delta/Iceberg + Parquet |
|:---|:---|:---|
| **数据类型** | AI 驱动（Embedding、长文本、多模态） | 纯结构化（数字、日期、字符串） |
| **团队技术栈** | Python / Polars / DuckDB | Spark / Trino / Tableau |
| **数据规模** | 数百 GB 至数 TB | 多 TB 至 PB |
| **运维偏好** | 零服务、无 Catalog | 企业级 Catalog + ACID 治理 |
| **BI 工具集成** | 仅自定义仪表盘 | Tableau、Power BI、传统 BI |

---

## 5. 对象存储选型

面向容器化 / 自托管湖仓场景：

| 维度 | RustFS | MinIO | Garage |
|:---|:---|:---|:---|
| **语言/架构** | Rust，去中心化对等架构 | Go，Master/Agent 架构 | Rust，无主架构（CRDT） |
| **一致性模型** | 强一致（read-after-write） | 强一致（CP） | 最终一致（AP/CRDT） |
| **Delta CAS 支持** | ✅ 原生 | ✅ 成熟 | 🔴 概率性（LWW 风险） |
| **S3 Tables / REST Catalog** | ✅ 原生（2026+） | ❌ 不支持 | ❌ 不支持 |
| **资源占用** | 低（<100MB 二进制，Rust 无 GC） | 高（Go 运行时，>2GB 内存） | 低（<100MB 二进制） |
| **混合部署** | ✅ 分级存储（本地 NVMe 热层 + 云端冷层，自动分层） | ❌ 纯本地 | ❌ 纯本地 |
| **部署便捷性** | 单命令启动，Linux/Docker/K8s/macOS/Windows | 单命令启动，Linux/Docker/K8s | 较复杂 |
| **MCP Server** | ✅ 原生支持 | ❌ | ❌ |
| **协议** | Apache 2.0 | AGPL v3（商业限制） | AGPL v3 |

- **RustFS（首选）**：Rust 编写，性能与内存安全兼具。去中心化对等架构，无 Master 单点。内联纠删码保护数据。原生 S3 Tables 协议——存储桶自身可作为 Iceberg REST Catalog，内置 Auto-Compaction。本地部署极简，单二进制 + 单命令启动。Apache 2.0 协议，商业友好。
  - **分级存储（核心差异化）**：本地 NVMe SSD 作热数据层（Rust + io_uring 榨干磁盘 I/O），外部对象存储（S3/MinIO/COS）作冷数据层。通过生命周期规则自动分层——冷数据自动迁移到云端，热数据留在本地。本质是"本地极速写 + 云端低成本存"的混合模式。
  - **MCP Server**：原生支持，AI 生态集成好。
- **MinIO（备选）**：久经考验，企业标准。AGPL 协议有商业限制。Go 运行时资源占用较高。不支持 S3 Tables。
- **Garage（不推荐）**：无主架构，CRDT + Dynamo 风格仲裁（AP 倾向）。存储层无原子 CAS，"Last Write Wins"静默覆盖事务日志，Delta 的 MVCC 状态在物理上分裂 → 数据损坏。

**推荐**：首选 RustFS（全面超越 MinIO），稳妥选 MinIO，不选 Garage。

---

## 6. 落地模式

### 模式一：DuckDB 直查 Parquet（最简）

最轻量的分析模式——跟 SQLite 读本地文件一个思路。对象存储放 Parquet，DuckDB 按需拉起来查，零服务、零运维。

```
┌─────────────────────┐        ┌─────────────────────┐
│  增量写入            │        │   分析查询            │
│  (Python / Polars)  │        │  (DuckDB + httpfs)   │
└────────┬────────────┘        └────────┬────────────┘
         │                              │
         ▼                              ▼
  ┌──────────────────────────────────────────┐
  │  s3://bucket/finance_table/              │
  │    append_20260101_120000.parquet        │
  │    append_20260102_093000.parquet        │
  │    ...                                   │
  └──────────────────────────────────────────┘
```

**写入端**：Polars 将带时间戳的 `.parquet` 文件直接写入 OSS。无锁、无目录服务——纯追加。
**查询端**：DuckDB 用通配符 `*.parquet` 扫描，自动合并所有文件。投影下推和谓词下推最小化 I/O。

```python
# 写入
from datetime import datetime
import polars as pl

def sync_incremental(new_rows: pl.DataFrame):
    ts = datetime.now().strftime("%Y%m%d_%H%M%S_%f")
    path = f"s3://your-bucket/finance_table/append_{ts}.parquet"
    new_rows.write_parquet(path, storage_options=oss_options)

# 查询
import duckdb

def run_summary():
    con = duckdb.connect()
    con.execute("""
        SET s3_endpoint='oss-cn-hangzhou.aliyuncs.com';
        SET s3_access_key_id='YOUR_ACCESS_KEY';
        SET s3_secret_access_key='YOUR_SECRET_KEY';
        SET s3_url_style='path';
    """)
    return con.execute("""
        SELECT month_col, employee_id, SUM(amount) AS total
        FROM 's3://your-bucket/finance_table/*.parquet'
        GROUP BY month_col, employee_id
    """).pl()
```

**注意**：随时间推移小文件累积会导致 HTTP 往返增加。定期跑 compaction 脚本合并大文件。

**Compaction 脚本与安全提示：**

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

### 模式二：LanceDB（AI/向量就绪）

零 Catalog、零服务，适合容器化轻量栈。同一张表支持 SQL 过滤 + 向量检索。

**Schema 定义策略：**

| 数据类型 | 使用场景 | 推荐 |
|:---|:---|:---|
| 财务金额 | 金额、余额 | `pa.float64()`（不用 float32，避免精度损失） |
| 时间/审计 | 时间戳、日期 | `pa.date32()`、`pa.timestamp('us', tz='Asia/Shanghai')` |
| 固定嵌套 | 发票、明细行 | `pa.struct([...])`（支持投影下推） |
| 动态标签 | 不可预测的 JSON | `pa.map_(pa.string(), pa.string())`（防止 Schema 漂移） |

**完整代码模板（四个函数）：**

```python
import os
from datetime import datetime

OSS_OPTIONS = {
    "aws_access_key_id": os.environ.get("OSS_ACCESS_KEY_ID"),
    "aws_secret_access_key": os.environ.get("OSS_SECRET_ACCESS_KEY"),
    "aws_endpoint": "https://oss-cn-hangzhou.aliyuncs.com",
    "aws_region": "cn-hangzhou"
}
DB_URI = "s3://your-finance-bucket/lancedb_core_lake/"

# 1. Schema 初始化（显式定义，防止 Schema 漂移）
def initialize_blank_financial_lakehouse():
    import pyarrow as pa
    import lancedb
    db = lancedb.connect(DB_URI, storage_options=OSS_OPTIONS)
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

# 2. 增量同步（延迟导入，冷启动 <10ms）
def skill_data_sync_lancedb(incoming_pl_df):
    import polars as pl
    import lancedb
    initialize_blank_financial_lakehouse()
    db = lancedb.connect(DB_URI, storage_options=OSS_OPTIONS)
    table = db.open_table("financial_records")
    payload = incoming_pl_df.with_columns(pl.lit(datetime.now()).alias("ingested_at"))
    table.add(payload)
    return {"status": "success", "version": table.version}

# 3. 分析查询（零拷贝 LazyFrame）
def skill_analytical_query_lancedb(sql_filter: str):
    import polars as pl
    import lancedb
    db = lancedb.connect(DB_URI, storage_options=OSS_OPTIONS)
    table = db.open_table("financial_records")
    lazy_lance = table.to_lance().to_lazy()
    result = (
        lazy_lance.filter(pl.sql(sql_filter))
        .group_by(["year", "month"])
        .agg(pl.col("amount").sum())
        .collect()
    )
    return {"status": "success", "data": result}

# 4. 每周维护（异步 CronJob，MVCC 确保前台查询不受影响）
def skill_weekly_maintenance_compact():
    import lancedb
    db = lancedb.connect(DB_URI, storage_options=OSS_OPTIONS)
    table = db.open_table("financial_records")
    table.compact_files()
    table.cleanup_old_versions(older_than_days=7)
```

**关键点**：
- **延迟导入（Lazy Import）**：不在顶层导入 `polars`/`lancedb`，冷启动从 ~500ms 降至 <10ms
- **零拷贝查询（Zero-Copy Query）**：`table.to_lance().to_lazy()` 创建 Polars LazyFrame，无需下载数据
- **盲追加（Blind Append）**：并发写入创建独立物理 fragment，避免锁竞争
- **维护**：异步执行 compaction（CronJob），LanceDB 的 MVCC 确保前台查询不受影响

### 模式三：Iceberg + Catalog（高频更新）

高频更新/删除场景的正解。MoR 追加几 KB 的 Delete File，写入毫秒级完成。REST Catalog 提供高效快照索引。

**落地路线图：**
1. 开通 OSS Tables 邀测 → 创建测试表格桶 → Python 模拟高频更新写入 MoR 格式 Iceberg
2. DuckDB 通过 REST Catalog 读取 → 验证频繁更新后全表扫描是否从 50s → sub-second
3. 生产环境写入链路改造为 Iceberg 追加 + 开启云端自动合并

**关键认知：**
- MoR 是"写时省、读时稍贵但被后台治理抹平"——对高频更新场景净收益巨大
- REST Catalog 是 Iceberg 的标准化方向——不再依赖 Hive Metastore / Glue / 自建锁服务
- OSS Tables ≈ S3 Tables——对象存储桶内置 Iceberg REST Catalog + Auto-Compaction

---

## 7. 阿里云兼容性问题

### OSS 不支持 Delta Lake 的根因

`deltalake` 1.6 在写入事务日志时硬编码了 `If-None-Match` HTTP 头部。阿里云 OSS 不支持此头部（返回 400 `NotImplemented`），Rust 的 `object_store` 依赖 S3 完整 API，没有 fallback 路径。

OSS 有自己的条件写入头 `x-oss-forbid-overwrite`，但 `object_store` 不识别。

**本质**：阿里云有完整的 Lakehouse 方案（DLF + MaxCompute + Hologres），让 OSS 完美兼容 Delta Lake 意味着用户可以随意切换计算引擎，阿里云沦为"卖硬盘的"。保持边缘 API 差异是隐形的 Vendor Lock-in。

**Workaround**（单写入者场景）：

```python
storage_options = {
    "aws_access_key_id": "YOUR_ACCESS_KEY",
    "aws_secret_access_key": "YOUR_SECRET_KEY",
    "aws_endpoint": "oss-cn-hangzhou.aliyuncs.com",
    "aws_region": "cn-hangzhou",
    "aws_unsafe_disable_conditional_write": "true",
    "aws_virtual_hosted_style_tables": "false",
    "aws_skip_signature_v4_cluster": "true",
}
```

禁用条件写入后并发写入同一 Delta 表可能冲突。多写入者场景建议用原生 Parquet + DuckDB 或切换到 Iceberg。

### 替代方案

1. **MinIO 自建**：100% S3 兼容，Delta Lake 开箱即用
2. **原生 Parquet + DuckDB**：绕过 Delta 事务日志，用文件系统语义管理数据
3. **Iceberg**：通过 DLF Catalog 支持稍好，但 Rust 实现可能面临同样的 `object_store` 问题

---

## 8. 迁移路径：Delta → Iceberg

> 订单系统高频点状更新场景下，Delta Lake CoW 模式导致 50s 查询延迟的根因与迁移方案。

### 根因：Delta CoW 的两个死穴

- **写放大**：更新 1 条数据 → 读入整个 200MB 文件 → 重写出新 200MB 文件
- **碎文件爆炸**：频繁重写导致 OSS 积压大量无序小文件 + 极长元数据日志。DuckDB 全表扫描被迫发起 60+ 次 HTTP 请求串行下载，50 秒延迟绝大部分死在建立网络握手和解析海量元数据上

### 迁移方案

- **MoR 根治写放大**：更新时不碰原始文件，仅追加几 KB 的 Positional Delete File + 一条新记录
- **REST Catalog 消除元数据瓶颈**：查询引擎直接请求 REST API 获取当前快照
- **Auto-Compaction 委托云端**：后台自动调度合并，查询永远命中干净大文件

**MoR 代价说明**：读取时需要合并原始文件 + Delete 补丁，查询有额外开销。但 Auto-Compaction 在后台持续把补丁合回主文件，保证查询永远命中干净大文件。MoR 是"写时省、读时稍贵但被后台治理抹平"——对高频更新场景净收益巨大。

**迁移路线图：**
1. **测试验证**：开通 OSS Tables 邀测 → 创建测试表格桶 → Python 模拟订单高频更新写入 MoR 格式 Iceberg
2. **基准对撞**：DuckDB 通过 REST Catalog 读取表格桶 → 验证频繁更新后全表扫描聚合是否从 50s → sub-second
3. **正式割接**：生产环境写入链路改造为 Iceberg 追加 + 开启云端自动合并

### 什么时候不该迁移

| 场景 | 选 Delta | 选 Iceberg |
|:---|:---|:---|
| 时序日志 / 事件流（只追加不更新） | ✅ 够用 | 过度工程 |
| 批量 ETL 一次性灌入后只读 | ✅ 简单 | 差异不大 |
| IoT 传感器数据（高频写入，极少更新） | ✅ 适配 | 可选但非必要 |
| 订单/用户状态（高频更新+删除） | ❌ CoW 致命 | ✅ 正解 |
| 时间旅行 + 行级删除的审计场景 | ❌ 日志链慢 | ✅ 快照隔离 |

**判断原则**：数据写入后基本不修改 → Delta 更简单。更新/删除成常态 → Iceberg 才值得。

---

## 交叉引用

- **[统一数据层](unified-data-layer.md)**：SurrealDB 的计算下推原则与对象存储的共识协议本质同源——减少不必要的中转，让逻辑在数据所在的地方执行。
- **[Redis 批判](redis-critique.md)**：架构批判方法论——从"架构鼓励什么"的视角审视技术选型，而非"工具没问题开发者用错了"。
- **[反应式架构](reactive-architecture.md)**：Lakehouse 的 Catalog 层管理数据版本和快照，与反应式架构的版本号接口思路一致——把版本治理从存储层提升到控制面。
