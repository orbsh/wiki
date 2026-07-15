# RisingWave CDC 集成（MySQL）

将 MySQL 同步到 RisingWave 的项目特定规则。

## 模式推断策略

- **版本**：需要 RisingWave v1.5+（支持通配符 `*`）。
- **语法**：`CREATE TABLE t (*, PRIMARY KEY (pk)) FROM source ...`
- **约束**：不要将显式列定义与 `*` 混合。对所有列使用 `*` 让 RW 从源自动推断类型（如 `bit(1)` -> `BOOLEAN`）。

## 源管理

- **源覆盖**：在配置中使用 `source_override` 将生成的源映射到现有源（如 `xmh_shop` -> `search._sources_v3`）以避免重新回填数据。
- **白名单**：按源配置 `debezium.table.include.list` 以减少性能开销。

## MySQL → RW 类型映射（手动调整）

从 `sql/search_prod/00_source.my.rw.sql` 手动覆盖派生。脚本 `my2rw.py` 应用这些规则：

| MySQL 类型 | 手动 / RW 类型 | 备注 |
|-----------|---------------|------|
| `BIT(1)` / `BIT` | `BOOLEAN` | info_schema 中的 `bit` 通常也匹配 `BIT(1)`。|
| `SMALLINT` | `INTEGER` | 原始是 `SMALLINT`，为安全升级。|
| `MEDIUMINT` | `BIGINT` | 原始是 `MEDIUMINT`，升级到 `BIGINT`（自增 ID）。|
| `INT` | `BIGINT` | 同 Mediumint，对齐到 64 位。|
| `BIGINT` | `BIGINT` | 无变更。|
| `FLOAT` | `REAL` | |
| `DOUBLE` | `DOUBLE PRECISION` | |
| `TINYINT` | `SMALLINT` | |
| `TINYTEXT` / `TEXT` | `TEXT` | |
| `BLOB` / `BINARY` | `BYTEA` | |
| `JSON` | `JSONB` | |
| `DATETIME` | `TIMESTAMP` | |
