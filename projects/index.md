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
| `fjall-openraft-design.md` | Fjall + Openraft 设计 | 原生分布式系统：嵌入式 KV + Raft 共识替代 Redis。交叉引用 redis-critique |
| `../unified-data-layer.md` | SurrealDB 统一数据层 | 现代多模型引擎、SurQL 人体工程学、Redis/PG 插件的统一替代（根目录） |
| `skillforge.md` | SkillForge | 多引擎 Agent 框架（Agno + Hermes）：上下文传播、技能加载、工具格式化 |
| `nixos-config.md` | NixOS 配置 | 基于 Flake 的 NixOS 主机（workstation、server、portable、qemu）、模块结构、便携更新工作流 |
