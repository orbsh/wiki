# 项目文档索引

项目特定文档存放在此。每个文件涵盖特定项目的结构、约定、决策和运维知识。

> 全 wiki 索引见 [../index.md](../index.md)。

| 文件 | 项目 | 内容 |
|------|------|------|
| `deployment-ci.md` | deployment + ci-data | Helm 部署结构、约定、CI 数据配置、入职流程 |
| `fluxora-architecture.md` | Fluxora | 完整架构：项目结构、编解码策略、流式合并、陷阱 |
| `fluxora-decisions.md` | Fluxora | 架构决策记录（ADR）|
| `knowledge-base.md` | 知识库 | SurrealDB RAG 架构、通过内置 HTTP 函数嵌入（ADR-005）|
| `architecture-choices.md` | 架构选择 | 网关选择、仓库策略（Zot+WG）、Redis/数据库批判 |
| `kv-storage-engine.md` | KV 存储引擎架构 | 嵌入式 KV 架构设计、复合键编码与用例。交叉引用 redis-critique |
| `https://github.com/orbsh/okm` | OKM：Object-Keyspace Mapping（项目，原 wiki 文档已并入） | 对标 ORM 的 KV 键空间映射过程宏。交叉引用 kv-storage-engine |
| `../unified-data-layer.md` | SurrealDB 统一数据层 | 现代多模型引擎、SurrealQL 人体工程学、Redis/PG 插件的统一替代（根目录） |
| `skillforge.md` | SkillForge | 多引擎 Agent 框架（Agno + Hermes）：上下文传播、技能加载、工具格式化 |
| `psi.md` | ψ | wasm 原生的方法链语言（假想设计）：单原语调用、统一访问、标签模型、Json 与 Path、无 LLVM 后端、三宿主形态 |
| `nixos-config.md` | NixOS 配置 | 基于 Flake 的 NixOS 主机（workstation、server、portable、qemu）、模块结构、便携更新工作流 |
