# Knowledge Base (SurrealDB RAG)

## Architecture Decisions

### ADR-005: 使用 SurrealDB 内置 HTTP 函数调用外部 API

- **状态**: 已采纳
- **背景**: 需要在向量入库流程中调用 Ollama 等外部嵌入 API
- **决策**: 在 SurrealDB 中通过 `http::post()` / `http::get()` 内置函数直接发起 HTTP 请求，而非由后端应用中转
- **理由**:
  - **减少数据中转**：向量数据量大（如 bge-m3 单条 ~4KB），直连避免后端无意义转发，节省带宽和内存
  - **接口隐藏**：应用只需调用 `fn::ollama::embed()` 函数，无需知道 Ollama 的存在
  - **配置集中**：Ollama 地址、超时、模型等配置在数据库 `config` 表中统一管理
  - **降低多应用维护成本**：多个应用只需对接数据库一个接口，无需各自管理 HTTP 调用
  - **解耦**：更换嵌入模型或服务时，只需修改数据库中的函数定义，应用零改动
  - **事务一致性**：HTTP 请求可在事务中执行，确保"嵌入+存储"原子操作
  - **权限控制**：通过 SurrealDB 权限系统控制谁能调用嵌入功能
- **执行模型**:
  - **异步非阻塞**：SurrealDB 基于 tokio 异步运行时，HTTP 请求为异步执行
  - **不阻塞其他查询**：一个查询中的 HTTP 请求不会阻塞其他并发查询
  - **单查询内串行**：同一查询中的多个 HTTP 调用依次执行，暂无原生并行语法
  - **当前场景无影响**：`fn::ollama::embed()` 每次仅发起一次 HTTP 请求，不存在队头阻塞问题
- **相关文件**:
  - `communicate-skills/skills/surrealdb-query/assets/ollama.surql`（函数实现）
  - `SPECIFICATION3.md` — ADR-004（统一存储架构，同属减少外部依赖的设计原则）
