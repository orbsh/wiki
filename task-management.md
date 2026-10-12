# 任务管理架构：scratch 改进计划 与 agent 集成

> **语言：** [English](task-management-en.md)（主文档）· [中文](task-management.md)
> **状态**: 改进计划 / 设计分析（2026-08-28）

---

## 〇、scratch 现状（来源）

scratch 是既有工具：

- 本地：`~/Configuration/nushell/scripts/scratch`
- 线上：<https://github.com/orbsh/scratch.nu>

本文档有两层意图：**① scratch 的改进计划**；**② agent 集成模式**——最终可能与 [graph-memory.md](graph-memory.md) 一起集成到 agent 里。

## 一、核心形态（scratch 模型）

scratch **存数据库**：

- 每条是一个**记录**（record）。
- **树形组织**。
- 另有**标签体系**，标签也可以组织成**树**。

## 二、scratch vs neorg 对比

### scratch：结构化 == 树形 == 脑图

可**拓展为网络结构**：

- Rust 重写
- 使用 KV 存储

### neorg：文本化 == 可编辑

- 不能容纳很多元数据
- 元数据要考虑视觉呈现
- 要设计语法
- **视觉呈现与语法设计是耦合的** → 扩展性差（改呈现往往牵动语法）

### scratch 的界面短板

- 只能在终端里用，没有能实时修改的界面。
- 发展方向：
  - 做 web 版
  - 和 launcher 结合
  - cli 接口

## 三、改进计划

- **结构化拓展为网络结构**：Rust 重写；使用 KV 存储。
- **界面短板**：web 版 / 结合 launcher / cli 接口（实时可改的界面 + 键盘工作流接入）。
- **视觉呈现**：结构化与文本化**都需要**设计视觉呈现（不是某一方的独有代价）。区别在耦合度——**文本化（neorg）的视觉呈现与语法设计耦合，扩展性差**；结构化（scratch）在此基础上更易解耦。

## 四、agent 集成模式

- 考虑 scratch 与 agent 的集成模式：任务树 + 标签树可作为 agent 的任务/记忆结构。
- **协调定位**：task 图不止是个人任务清单，还是**多 agent 协调介质**（共享任务视图替代消息互传）与**组织账本**（工作走图则追踪零成本）——论据见 [graph-memory.md](graph-memory.md) §任务网状化。
- **终局**：与 [graph-memory.md](graph-memory.md) 一起集成到 agent（任务结构并入 agent 的记忆/推理编排）。