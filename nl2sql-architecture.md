# NL2SQL 架构：OLTP 上做分析是架构设计错误

**结论**：NL2SQL 的架构选择问题，本质上是"有没有数据分析需求"的问题。

- **没有数据分析需求**：OLTP 走接口，AI 做智能路由，这不是 NL2SQL，是智能操作。
- **有数据分析需求**：绕不开 OLAP 基础设施。在此之上添加 NL2SQL 管线，边际成本极低。

OLAP 基础设施本身是数据业务的要求，不是 NL2SQL 的要求。NL2SQL 只是在已有湖仓上加一层 Skill 而已。
**状态**：从数据架构第一性原理出发的批判性分析。

## 为什么原始数据转储行不通

业务用户希望通过自然语言查询数据（"本月产品销量排名"）。最直觉的方法是将原始数据转储给 LLM 分析——这在小数据集上看似可行，但在企业场景下必然失败：

1. **速度**：大数据集延迟不可接受
2. **Token 成本**：上下文窗口被原始数据填满，成本呈线性增长
3. **准确性**：LLM 对原始数据聚合产生幻觉——它在"阅读"数据，而非"查询"数据

**认知**：以结构化形式存在的数据应该通过 SQL 查询，而非被 AI "阅读"。源和结果之间零信息损失。这本质上是 **NL-to-SQL** 任务，不是"AI 分析"任务。


## 1. 架构方案定义

### 语义中间层（SaaS 兜底）

```
用户提问 → LLM → SemanticPlan（语义映射）→ CapabilityCatalog（权限）→ PlanValidator（预检）→ OLTP 主库
```

**设计初衷**：应对千差万别的客户原生表结构，提供开箱即用的通用安全查询能力，避免强制客户搭建数据流水线。

**核心组件**：

| 组件 | 职责 | 本质 |
|------|------|------|
| SemanticPlan | 语义映射，将自然语言映射到表/字段 | 用应用层复杂度补偿 OLTP 表结构的混乱 |
| CapabilityCatalog | 权限控制，限制可查询的表和字段 | 用应用层限制补偿 OLTP 缺乏分析隔离的缺陷 |
| PlanValidator | 预校验，拦截危险操作 | 用应用层拦截补偿 OLTP 无法承受复杂查询的事实 |

### 宽表直查

```
用户提问 → LLM + Skill（表结构/字段说明/示例）→ SQL → OLAP 分析层（宽表）
```

**设计初衷**：顺应数据架构自然规律，将复杂度下沉至底层数据建模，应用层保持极简。

**核心机制**：
- 数据流水线（ETL/CDC）将主库数据同步至 OLAP 分析层
- LLM 通过 Skill 直接生成 SQL，依赖数据库原生机制进行语法校验和错误重试
- 分析层为只读，隔离生产数据风险

**具体实现架构**（企业级技能）：

```
┌─────────────────────────────────────────────────┐
│                   SKILL（无状态）                 │
│                                                  │
│  ┌──────────┐    ┌───────────┐    ┌───────────┐ │
│  │   用户    │───►│   AI LLM  │───►│  SQL 生成  │ │
│  │   查询    │    │ (NL→SQL)  │    │ (CTE 模式) │ │
│  └──────────┘    └───────────┘    └─────┬─────┘ │
│                                         │        │
│  ┌──────────────────────────────────────▼──────┐│
│  │           技能脚本 (run.py)                  ││
│  │  1. CTE 包装：注入 user_id + org_path 过滤  ││
│  │     WITH filtered_data AS (                 ││
│  │       SELECT * FROM data                    ││
│  │       WHERE user_id = ?                     ││
│  │         AND org_path LIKE ?                 ││
│  │     )                                       ││
│  │     <LLM_SQL 替换表名为 filtered_data>      ││
│  │  2. duckdb.sql(wrapped_sql)                 ││
│  └──────────────────────┬──────────────────────┘│
│                         │                        │
└─────────────────────────┼────────────────────────┘
                          │
              ┌───────────▼───────────┐
              │   Lakehouse (Iceberg) │
              │  s3://bucket/org/42/  │
              │    data/*.parquet     │
              └───────────────────────┘
```


### 执行摘要

NL2SQL 有两种架构路径：

- **语义中间层**：LLM 直连 OLTP 主库，引入语义计划、能力目录、预校验器等组件，在 SQL 生成前进行语义映射和危险操作拦截
- **宽表直查**：数据同步至 OLAP 分析层，LLM 通过 Skill 直接生成 SQL，在分析层执行

语义中间层试图在 OLTP 上解决 OLAP 问题，本质上是架构设计错误。宽表直查顺应数据架构自然规律，将复杂度下沉至数据层，应用层保持极简。


## 2. 推理链条：从技术细节到架构本质

### 逻辑节点 1：LLM 能力边界与纠错机制的冗余

**质疑**：语义中间层的 SemanticPlan 能理解复杂语义并在生成前纠错，这是普通 Agent 纠错（生成后报错再修）做不到的。

**反驳**：LLM 本身具备理解字段的能力（配合 Skill 里的说明和示例）；纠错循环是 Agent 的通用能力，没必要为 SQL 场景专门搞一套复杂架构。单纯为了"语义理解"和"纠错"去建一套中间层，是不必要的重复造轮子。

### 逻辑节点 2：业务定义的来源本质

**质疑**：数据库不知道"高价值客户"这种复合业务定义，SemanticPlan 可以处理这种复杂映射，防幻觉。

**反驳**：业务定义总要有个来源。语义中间层需要维护配置，宽表直查写在 Skill 里，本质没有信息增量。中间层只是把信息整理提前了，不如直接利用数据库原生报错机制可靠（跑不通就报错，跑得通就是对的）。

### 逻辑节点 3：工程视角的全面劣势

**结论**：在成本、复杂度、维护和可移植性上，语义中间层全面处于劣势，没有存在的工程价值。

### 逻辑节点 4：架构破案点——OLTP vs OLAP

**洞察**：OLTP 库高度范式化，多表 Join 性能极差且影响主业务。语义中间层的中间件本质是在"硬扛"OLTP 不适合 OLAP 查询的缺陷。

> "人类分析都需要数仓，LLM 同样需要。"

将复杂分析查询直接打在 OLTP 主库上，不仅会导致锁竞争和性能拖垮，其复杂的范式表结构也会极大地消耗 LLM 的注意力，增加幻觉概率。将数据同步加工为"宽表"供 LLM 查询，是降维打击。


## 3. 语义中间层为何存在：商业逻辑闭环

语义中间层不是架构师不懂 OLAP，而是在商业和通用性约束下做出的最务实的"防御性架构"。

### SaaS 的宿命：面对"现状"而非"理想"

作为云服务提供商（如 Cloudflare），面对的是形形色色的客户：
- 绝大多数客户只有主业务库（OLTP），没有专门的数仓
- 不懂也不想搞复杂的 ETL/ELT 数据流水线
- 表结构千奇百怪

Cloudflare 无法要求客户先建好湖仓再做 AI 查询——门槛太高，没人用。提供语义中间层作为"开箱即用"的退化方案，是商业上的必然选择。

### 语义中间层在 SaaS 场景下的真实价值

| 组件 | SaaS 场景下的真实价值 |
|------|---------------------|
| SemanticPlan | 客户表结构千差万别，没有统一宽表，必须加一层强制语义映射 |
| CapabilityCatalog | 无法控制客户底层数据库权限，必须中间层强制查询限制和熔断 |
| PlanValidator | 共享服务中执行失败代价高，提前拦截保护平台资源 |

**总结**：语义中间层用应用层的复杂度，去填补客户底层数据架构的缺失。


## 4. 核心架构原则

### OLTP 与 OLAP 必须分离

> "人类分析都需要 OLAP，没理由到了 LLM 反而不需要了。"

| 维度 | OLTP | OLAP |
|------|------|------|
| **设计目标** | 事务一致性 | 分析吞吐 |
| **数据模型** | 高度范式化 | 宽表/星型模型 |
| **查询模式** | 点查/短事务 | 复杂聚合/多维分析 |
| **适合 LLM？** | ❌ 表结构复杂，Join 性能差 | ✅ 宽表直接查询，语义清晰 |

### CQRS 与 API-Driven 原则

LLM 作为系统参与者，应严格遵循 CQRS（命令查询职责分离）：

| 操作类型 | 正确路径 | 错误路径 |
|---------|---------|---------|
| **写操作**（下单、修改） | 业务 API → 风控/校验/状态机 | ❌ LLM 直接写数据库 |
| **读操作**（查询分析） | 分析接口 → OLAP 引擎（只读） | ❌ LLM 直连 OLTP 主库 |

> "LLM 本质上是一个'数字员工'，你不能让员工直接去改财务系统的底层数据库，必须通过系统界面（API）来操作。"

### AI 设计，引擎执行

AI 生成 SQL；OLAP 引擎处理重计算。永远不让 AI 直接处理数据——读写分离是底线。

### 技能自包含

无外部 ETL、无 cron、无 catalog 服务。同步逻辑存在于技能脚本内，重用现有 API。冷启动延迟被首次请求吸收，后续是增量且快速的。

### 强类型优于测试

AI SQL 生成受模式上下文 + 强制租户过滤器 + 只读引擎 + `LIMIT` + 超时约束。不依赖"AI 不会犯错"的假设。


## 5. 实现宽表直查：工具选型

> 同步机制（数据如何进入 Lakehouse）尚无定论，见 §6 Serverless 落地路径中的多种方案对比。

### 数据持久化：Lakehouse（Iceberg 优先）

| 选项 | 结论 | 原因 |
|------|------|------|
| 本地技能路径（Parquet）| ❌ 拒绝 | 技能必须无状态且可移植。本地持久化在实例重启时中断。|
| 直接业务数据库 | ❌ 拒绝 | 在生产数据库上执行 AI 生成的 SQL 是不可控风险。注入、全表扫描、数据损坏。|
| 新数据库 + 同步 | ⚠️ 可行但重 | 需要专用数据库运维、连接池、备份、迁移。|
| **湖仓一体（Iceberg）** | ✅ **首选** | 开放表格式，S3 原生，无需专用服务。 |
| 湖仓一体（Delta Lake）| ✅ 备选 | 基于文件事务日志，无需 catalog 服务。但生态封闭度高于 Iceberg。|

**选择**：**S3/对象存储上的 Lakehouse**，优先 Iceberg。开放表格式，原生分区裁剪，与 DuckDB/Spark/Trino 等查询引擎同源。Schema evolution 由表格式原生支持，但宽表 schema 变更时需同步更新 Skill 提示词。

Parquet 被拒绝的原因——企业场景需要多技能交叉引用和并发写入：

| 关注点 | Parquet | Lakehouse |
|--------|---------|-----------|
| 多技能 JOIN | ❌ 无模式注册表 | ✅ 模式演进、事务日志 |
| 并发写入 | ❌ 文件损坏 | ✅ ACID 事务 |
| 碎片化 | ❌ 小文件累积 | ✅ 内置 compaction |
| 点更新/删除 | ❌ 重写整个文件 | ✅ 支持 MERGE / DELETE |

### 工具选择：写读分离

| 工具 | 结论 | 原因 |
|------|------|------|
| Spark/Flink（Java/Scala）| ❌ 拒绝 | 启动慢、JVM 预热、重集群维护。技能是非持久进程。|
| Polars | ⚠️ 部分 | 数据操作优秀，但 SQL 支持有限。AI 生成 SQL 比 Polars DSL 更好。|
| DuckDB | ✅ 读层 | 原生 SQL、快速 OLAP、Python 绑定成熟。完美适合 AI 生成的 SELECT 查询。|
| `deltalake` Python | ✅ 写层 | 基于 Rust、经过验证、处理 Delta Lake 写入。AI 永远不触碰写入路径。|

**选择**：**`deltalake`（Rust/Python）用于写入 + DuckDB 用于读取**。职责分离：技能的 `run.py` 拥有写入逻辑；AI 只生成 SELECT 查询。

### 数据安全：查询时过滤注入

**需求**：多租户企业系统。用户只能看到自己有权限的数据。安全性不依赖 LLM 的提示遵守率。

**核心原则**：过滤条件由脚本从认证上下文获取，**带外注入**，不经过 LLM。

#### 模式一：user_id 注入（基础）

每个查询必须带上当前用户的 `user_id`。SKILL 提示 LLM："你的 SQL 会从 `filtered_data` CTE 查询，它已包含当前用户的数据。"

```python
# 脚本层：模板引擎填充（非字符串替换）
# 用 Jinja2 或类似模板引擎，避免 SQL 注入风险
template = """
WITH filtered_data AS (
    SELECT * FROM data WHERE user_id = :user_id
)
{{ llm_sql }}
"""
# 渲染时绑定参数，而非字符串拼接
```

- LLM 无需记忆过滤条件
- 即使 LLM 生成复杂 SQL 也能保证过滤
- 过滤逻辑在脚本层，对 LLM 透明

**风险评估**：模板引擎填充避免了字符串替换的 SQL 注入风险，但仍需验证 LLM 生成的 SQL 语法正确性（通过数据库原生校验）。

#### 模式二：org_path 分区隔离（进阶）

当数据按组织分层存储时，用 `org_path` 作为分区列，实现路径枚举前缀过滤（如 `org/42/dept/7/`）。

```
WITH filtered_data AS (
    SELECT * FROM data
    WHERE user_id = '{user_id}'
      AND org_path LIKE '{user_org_path}%'
)
<LLM_SQL>
```

- **同步阶段**：所有记录从源 API 携带 `org_path`
- **查询阶段**：脚本注入 `user_id` + `org_path` 双重过滤
- **性能**：Lakehouse/DuckDB 原生分区裁剪 `LIKE 'prefix%'`——避免全表扫描

这实现了**行级安全（RLS）**而无需数据库级授权。


## 6. Serverless/受限环境的落地路径

在 Cloudflare 等无服务器环境中，构建轻量级 OLAP 的务实路径：

### 路径 A：离线增量同步

```
Cron Triggers → 拉取增量（updated_at 水位线）→ 转 Parquet → 写入 R2 → DuckDB 查询
```

- 不需要完整 ETL 框架
- Parquet + 对象存储 = 极低成本的湖仓分离
- DuckDB（WASM 或独立实例）直接扫描 R2 上的 Parquet

### 路径 B：实时物化视图

```
主库 CDC → CF Queues → Consumer 清洗聚合 → 写入 D1/OLAP 引擎 → 宽表
```

- 用 CF 基础设施手搓极简版流计算 Pipeline
- 没有 Polars 复杂的 DataFrame 语义，但对"宽表物化"足够

### 路径 C：D1 作为只读分析层（小数据量）

```
定时同步主库 → CF D1（SQLite 分布式）→ LLM 查询 D1
```

- 百 GB 以内数据量的降级方案
- SQLite 对单机分析查询响应速度极快
- 彻底隔离主库风险

### 路径 D：Arroyo 流计算引擎（推荐）

CDC 进来 → SQL 做清洗/Join/物化 → 落成 Parquet/Iceberg 给 DuckDB 查。这条链路不需要完整的流式数仓（Flink/RisingWave），需要的是一个**单二进制、SQL 描述、能直接对接对象存储和 Iceberg**的流计算引擎——这正是 Arroyo 的设计目标。

**Cloudflare 闭环**：Cloudflare 在 2025 年 4 月收购 Arroyo，用它驱动 Cloudflare Pipelines。到 2025 年 9 月 Developer Week，三件套拼成完整数据平台：

| 组件 | 功能 |
|------|------|
| **Pipelines**（= Arroyo 引擎） | HTTP/Workers 摄入 → SQL 转换 → 写 Iceberg/Parquet 到 R2 |
| **R2 Data Catalog** | 托管 Iceberg 元数据，自动 compaction |
| **R2 SQL** | 自研分布式 SQL 引擎，直接查 R2 上的 Iceberg 表 |

这等于把"宽表直查"的完整链路在 Cloudflare 内部实现了——语义中间层确实是退化的兜底，Cloudflare 自己也在用宽表直查。

#### 为什么 Flink/RisingWave 偏重

| 引擎 | 问题 |
|------|------|
| **Flink** | JVM 体系，状态后端 + JobManager + TaskManager + checkpoint 协调，运维链路长，单实例资源占用高，对"OLTP 增量同步成分析宽表"的需求是杀鸡用牛刀 |
| **RisingWave** | Rust 写、Postgres 兼容，比 Flink 轻，但本质上是"流式数仓"，自带存储和元数据服务，部署形态仍偏重，适合作为常驻分析层而非嵌入式同步管道组件 |

#### 流计算引擎横向对比

| 维度 | Cron + Polars | Fluvio | Arroyo | Flink / RisingWave |
|------|--------------|--------|--------|-------------------|
| **实现语言** | Python/Rust | Rust | Rust | JVM / Rust+自研存储 |
| **部署形态** | 定时函数 | 单集群，K8s 原生 | **单二进制 / 单容器** | 多组件集群 |
| **编程模型** | 手写 DataFrame | 主题 + 无状态处理 | **SQL 流式管道**（DataFusion） | SQL/DataStream |
| **CDC 能力** | 需自己接 | 需配合外部 CDC | **Postgres/MySQL 原生 connector** | Debezium/Kafka 生态成熟 |
| **落地分析层** | 手写 Parquet | 需自己写 sink | **原生 FileSystem sink + Iceberg sink，exactly-once** | 生态最全 |
| **NL2SQL 契合度** | 数据量小时最省 | 偏传输层 | **最契合** | 过重 |
| **维护风险** | 低但要自己造轮子 | InfinyOn 重心转向云服务 | **已被 CF 收购，引擎仍开源可自托管** | 成熟但重 |

#### Arroyo 在链路中的位置

```
OLTP 主库 (Postgres/MySQL)
  → 逻辑解码
    → Debezium (CDC connector)
      → Kafka / Fluvio topic
        → Arroyo SQL Pipeline (清洗/Join/物化宽表)
          → exactly-once
            → Iceberg / Parquet (R2 / S3 / GCS)
              → DuckDB / R2 SQL (只读分析查询)
                → LLM 生成 SQL + Skill 元数据
```

几个关键点：

- **Fluvio 和 Arroyo 是互补，不是竞品**。Fluvio 是传输层（边缘原生，冷启动快），Arroyo 把 Fluvio 作为 source/sink connector。没有现成消息层时，直接 Kafka 甚至 Debezium→Arroyo 也行。
- **Arroyo 和 DuckDB 共享 Arrow/DataFusion 生态**。流计算写入的 Parquet/Iceberg 和 DuckDB 读取的 Parquet/Iceberg 在类型系统和执行语义上同源，避免了实现差异。
- **性能**：Arroyo 对标 Flink 有数量级优势（滑动窗口场景吞吐近常数，整体 5x+），资源占用是单二进制级别。

#### 落地路径

| 场景 | 推荐方案 |
|------|---------|
| **已在 Cloudflare 生态** | Pipelines + R2 Data Catalog + R2 SQL，引擎托管，连部署都省了 |
| **自托管/多云** | Arroyo 单二进制 + DuckDB 查 Parquet/Iceberg，CDC 用 Debezium，传输层可选 Kafka 或 Fluvio |
| **数据量小、容忍延迟** | Cron + Polars 仍是最小可行版本，Arroyo 是自然升级路径 |

#### RisingWave 的资源爆炸：架构基因决定的

RisingWave 的资源消耗不是调参没调好，是架构基因决定的：

| 问题 | 机制 |
|------|------|
| **物化视图即存储** | MV 结果存在 Hummock（S3）里的"表"，每建一个 MV 就要长期维护一份状态 |
| **Actor 模型 + 行存状态** | 流式 job 切成 fragment → actor，每个 actor 持有自己的状态（hash table / agg buffer / join state）。Join 越多、key 越散，状态越大 |
| **Backfill 即爆炸点** | 建大 MV 或带多 CTE 的 sink 时，backfill 相当于"同时跑 N 个查询"，内存装不下所有 stateful operator → cache miss → 远程 IOPS 飙升 → 雪崩 |
| **默认吃满内存** | 不显式设 `RW_TOTAL_MEMORY_BYTES` 就会把节点内存吃光，K8s/Docker 下必须强制设限 |

**本质**：它把"流处理"和"数据库存储"焊在一起了——你只是想做个 ETL 同步，它却要你养一个常驻的流式数仓。

#### Arroyo vs RisingWave：状态生命周期是关键差异

| 维度 | RisingWave | Arroyo |
|------|-----------|--------|
| **定位** | 流式数仓（处理+存储+服务） | 纯流处理引擎（处理+sink，不服务查询） |
| **状态后端** | Hummock（S3）+ 内存 cache，行存 | 开源版：内存；Cloud 版：FoundationDB 远程 KV |
| **状态生命周期** | **长期**（MV 永驻） | **短期**（窗口/join 算完就 sink 走） |
| **结果服务** | 直接 SELECT 查 MV | 不服务查询，结果落 Parquet/Iceberg 给 DuckDB 查 |
| **最小资源** | 默认吃满节点内存 | 自称小用例几 MB RAM、fractional vCPU |
| **生态同源** | 自研 SQL | SQL 基于 Arrow + DataFusion，和 DuckDB 同源 |

**关键差异在"状态生命周期"**：NL2SQL 同步场景里，Arroyo 的状态主要是窗口聚合 + 流式 join 的中间态，算完就 sink 成 Parquet/Iceberg，状态随即释放；而 RisingWave 的 MV 状态是永久驻留的。同样一条"订单宽表"管道，Arroyo 的稳态内存占用理论上比 RisingWave 低一个数量级。

#### Arroyo 的风险点

| 风险 | 说明 |
|------|------|
| **开源版纯内存状态** | 官方承认。大维表 join 状态增长过快时同样会 OOM。区别在于没有"常驻 MV"包袱，爆的概率和烈度低，但不是不会爆 |
| **分布式 state backend 仍在演进** | v0.7 开始做，往多 TB 状态、快速 restart、高频 checkpoint 方向走，但目前成熟度需要实测 |
| **被 CF 收购后的重心** | 引擎保持开源自托管，但团队精力明显在 CF Pipelines（托管版）。自托管用户遇到深坑，社区响应速度是问号 |
| **CDC connector 生态** | 走 Debezium→Kafka 路线，原生 CDC source 还在补，不如 RisingWave 的 Postgres/MySQL CDC 开箱即用 |

#### 低风险 POC 验证路线

NL2SQL 同步管道的状态规模可控——CDC 流 + 维表 join + 窗口聚合，状态量通常在 GB 级而非 TB 级。

```
选 1-2 张核心宽表（而非全库同步）
  → 单机 Docker 跑 Arroyo，限内存 2-4GB 压测
    → 稳态内存可控？
      → 是：接 Debezium CDC，验证增量水位线
        → 落 Parquet/Iceberg 到 R2/S3
          → DuckDB 只读查询 + LLM Skill 暴露 schema
            → 端到端延迟 + 资源成本达标？
              → 是：逐步扩大宽表覆盖
              → 否：退回 Cron+Polars 或评估 CF Pipelines 托管
      → 否：换 DuckDB 定时刷新或 Timeplus 方案
```

**具体建议**：

1. **先单表后多表**：别一上来就同步全库。挑 1-2 张 LLM 最常查的宽表（比如订单+用户聚合），用 Arroyo 做一条管道，压测稳态内存。这一步就能验证"actor 爆"的问题在 Arroyo 上是否存在。
2. **限内存跑**：Docker 里给 Arroyo 容器限 2-4GB，故意制造压力，看它是优雅 backpressure 还是直接 OOM。这是区分"轻量"是宣传词还是事实的关键实验。
3. **DuckDB 作为兜底**：如果 Arroyo 在你的数据规模下也撑不住，退一步用 DuckDB 的 streaming patterns 做定时增量刷新——牺牲实时性换确定性，对 NL2SQL 场景往往够用。
4. **CF Pipelines 作为托管备选**：自托管验证通过但不想维护时，直接用托管版是顺理成章的升级路径。

> **一句话收束**：Arroyo 在架构基因上比 RisingWave 更适合 NL2SQL 同步场景，但"没试过"这个风险只能用一个小规模 POC 来消除，而不是靠架构推理来消除。先单机限内存压一条管道，是最便宜的去风险化动作。

---

## 7. 结语

在 AI 架构设计中，不要试图用复杂的应用层中间件去掩盖底层数据架构的缺失。

- 把适合在数据层解决的问题（宽表建模、数据清洗）交还给数据流水线
- 把适合在数据库解决的问题（语法校验、权限隔离）交还给数据库引擎
- 保持上层 NL2SQL 逻辑的极简，才是工程上的最优解

---

## 交叉引用

- **[MySQL 批判](mysql-critique.md)**：OLTP 数据库不适合分析查询的技术论证——本批判的数据层基础。
- **[Lakehouse 研究](lakehouse-research.md)**：湖仓架构的设计哲学——宽表直查的架构参照。
- **[SurrealDB 统一数据层](unified-data-layer.md)**：多模态数据库对 OLTP/OLAP 二分法的挑战——本批判的边界条件。
