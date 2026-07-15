# `bx` 框架与 Vessel 分发

**项目路径：** `~/data/docker.io/xy`

## 1. 核心哲学

`bx`（Buildah eXtension）是基于 Nushell 的构建框架，用可编程的结构化逻辑替代 Dockerfile。它通过将构建视为代码而非静态配置来解决 CI/CD 中的"应试工程"。

## 2. `bx` 构建框架

### 架构

- **Plumbing**：将 `buildah`（mount/commit/push）封装为 Nushell 闭包。
- **Hub（`hub.yaml`）**：集中版本清单。构建自动查询 GitHub API 获取最新版本，消除手动版本升级。
- **逻辑**：用户提供 `acts` 闭包。框架处理容器生命周期。

### 关键特性

- **DRY**：`hub install [ rust nushell ]` 替代数百行 `curl | tar | mv`。
- **架构感知**：自动检测 `arch` 并选择正确的二进制文件。
- **CI 生成**：`x.nu action gen` 扫描脚本并动态生成 GitHub Actions 工作流。

## 3. Vessel：离线包分发

### 概念

自举的轻量级包分发系统，作为重型 Agent（Ansible）或复杂仓库工具（apt/yum）的替代方案。

### 工作流

1. **请求**：用户运行 `curl http://vessel/rust,nushell | sh`。
2. **引导**：脚本检查 Nushell。如果缺失，首先从 vessel 服务器下载安装。
3. **执行**：脚本回调 `vessel/install/...`，返回动态生成的 Nushell 脚本。
4. **安装**：脚本下载 `.tar.zst` 包并应用。

### 技术亮点

- **动态脚本**：服务器根据请求的包即时生成客户端脚本，注入必要函数（`info`、`tar-fs`、`install`）。
- **安装后钩子**：支持包内的 `setup.nu` 用于配置逻辑（systemd 设置、配置生成）。
- **零依赖**：客户端只需要 `curl` 和 `sh`（或 `zsh`/`bash`）。Nushell 被视为有效载荷，而非先决条件。

## 4. 评估

`bx` 代表构建工具中的**范式转移**：

- **从**：静态 Dockerfile、手动版本跟踪、重型 CI 配置。
- **到**：可编程管道、自动版本控制、自文档化构建。

它符合"第一性原理"方法：通过移除通常围绕容器构建积累的"胶水"代码来简化技术栈。
