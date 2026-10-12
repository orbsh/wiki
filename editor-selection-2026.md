# 编辑器选型分析：2026 深度技术报告

> **语言：** [English](editor-selection-2026-en.md)（主文档）· [中文](editor-selection-2026.md)
> **相关文档**：嵌入式脚本语言（Rune vs Steel）的技术深度对比见 [嵌入式脚本语言选型](embedded-script-languages.md)

## 一、Helix 的保守主义危机

### 1.1 完美的底层架构设计蓝图

- Helix 官方在规划其延宕数年的插件系统时，选定的核心内嵌扩展语言正是 Steel Scheme（[讨论](https://github.com/helix-editor/helix/discussions/3806)）。
- 纯 Rust 开发的高性能现代 TUI 编辑器（Helix），内嵌同样由纯 Rust 编写、天生线程安全的不可变 Lisp 虚拟机（Steel），其架构的严谨性、启动速度和理论上限完全可以对传统的 Neovim + Lua 实施降维打击。

### 1.2 质疑：比 C 语言项目还保守的 Rust 开源奇观

- **核心矛盾点**：Rust 语言的核心红利在于其极强的编译器安全兜底和宏系统，它天生支持且鼓励开发者进行大刀阔斧、无畏的"快节奏演进与大范围重构"。
- **事实指控**：然而 Helix 团队的审核理念走到了激进的对立面。在插件系统（Issue #122）和 Steel 绑定的合并（PR #8675）上，官方展现出了近乎苛刻的精英克制主义。为了追求绝对的零配置（Batteries-included）和 100% 稳定性，宁可让全球数万极客在原地干等三年，[生态至今必须依赖拉取特定开发分支编译才能艰难体验](https://sraj.me/notes/helix-steel/)。
- **对比推理**：反观使用 C 语言编写的古老 Neovim 社区，为了打破传统 Vim 的僵化，反而在引入 Lua 脚本层、推进多线程异步和迭代高频特性上表现得极其疯狂和务实。

---

## 二、Neovim 0.12 + Neovide：全功能集成帝国

### 架构设计（解耦下沉）

Neovim 0.12 正在经历一场"去 Lua 膨胀化"的革命。它将过去必须依赖第三方 Lua 插件的动态功能（如补全、包管理、Treesitter 增量选择）下沉到了纯 C 语言内核中。对于精通多种语言的开发者来说，Lua 在这里退化成了类似 JSON/TOML 的声明式配置，不需要再面对臃肿的 Lua 回调地狱。

### 信息密度与审美

天花板级别。配合 Neovide 的亚像素级渲染、无边框悬浮和物理微动画，所有视觉元素全部统一在同一个 Grid 像素块中。通过关闭状态栏（`laststatus=0`），可以实现 100% 没有任何圆角、伪 GUI 边框、花哨图标污染的纯文本矩阵。

### 集成方式：编辑器吞噬一切

Neovim 的集成方向是**向内**：`:terminal` 内嵌 PTY，Fugitive 内嵌 Git，Telescope 内嵌文件搜索。编辑器本身是集成平台，外部工具被拉进 buffer 里运行。RPC（`nvim --remote`）也能被外部控制，但社区重心在"内部扩展"。

代价是配置复杂度——Lua 回调地狱、插件冲突、启动优化，都需要用户自己扛。

---

## 三、Kakoune：Unix 哲学的纯编辑组件

### 架构设计（client-server）

Kakoune 的设计起点和 Neovim 完全相反：编辑器不是平台，是**被集成的组件**。

- **client-server 架构**：`kak` 命令启动 server 进程管理 buffer 和 undo 历史，`kak -c <session>` 连接已有实例，`echo 'buffer README.md' | kak -p <session>` 往运行中的实例注入命令
- **不做终端**：没有 `:terminal`，终端由 zellij/tmux 管理
- **不做文件管理**：没有内置文件树，yazi/lf 在另一个 pane 跑
- **不做 Git 集成**：没有 fugitive，lazygit 在另一个 pane 跑

### 外部交互：命令接口 > 内嵌

Kakoune 暴露了一个干净的命令接口，外部工具可以**主动操控**编辑器：

```
# 从 shell 脚本控制 kak 实例
echo 'select All; exec d' | kak -p my_session    # 清空当前 buffer
echo 'buffer src/main.rs' | kak -p my_session     # 切换文件

# zellij 中：kak 占一个 pane，nushell 在另一个 pane 跑命令
# 两者通过终端 PTY 粘合，零耦合
```

### 理念：Unix 组合 > 编辑器大一统

Kakoune 的正确用法是**三件套组合**：

```
┌─────────────────────┬──────────────────────┐
│   zellij            │   kak                │
│   （窗口/session/   │   （纯文本编辑）      │
│    终端管理）        │                      │
├─────────────────────┼──────────────────────┤
│   nushell           │   lazygit            │
│   （任务/脚本）      │   （Git 操作）        │
└─────────────────────┴──────────────────────┘
```

三者各自独立，通过终端 PTY 粘合。这是 Unix 哲学的组合方式——每个工具做好一件事，通过管道和终端协作，不是编辑器的大一统。

**AI 时代的优势**：编辑器退化成审计界面后，Kakoune 的"我只管编辑"反而更诚实。不需要在编辑器里嵌入 Git/终端/AI 工具链，这些全部在 zellij 的其他 pane 里运行，各自保持独立的进程生命周期。

### 局限

- 配置语言是 Kakscript（类 Vimscript），生态小
- 没有 LSP/Treesitter 内置支持（需要外部 wrapper 如 kak-lsp）
- 插件生态远小于 Neovim
- 对于需要深度集成 AI 补全、LSP 诊断面板的场景，不如 Neovim

---

## 四、Helix：现代纯粹的无服务器编辑刺客

### 架构设计（无状态/无服务器）

基于 Rust 编写，开箱即用，拒绝任何插件。**没有 Central Server 进程**——同一个目录下重复敲 `hx` 会新开独立进程，Buffer 和 Undo 历史完全割裂。

### 集成方式：拒绝集成

官方既没有内置终端，也没有 IPC 机制。多任务、Git、目录树能力全部外包给外部工具（如 Zellij）。但与 Kakoune 不同的是，Helix 没有命令接口供外部控制——它是一个**封闭的纯编辑器**。

### 定位

适合短命的一次性编辑任务（SSH 上去改个配置文件）。不适合需要 session 持久化、多文件导航、外部工具协作的日常工作流。

---

## 五、Emacs：可编程操作系统的缩减存在

### 定位

Elisp 图灵完备、Emacs 本身就是 Lisp 运行时，理论上能力上限最高。但体积代价（NixOS closure 925MB，Neovim 的 3.7 倍）和启动速度（2-5s）在 AI 审计时代成为硬伤——编辑器已经退化成审查界面，不需要一个操作系统级别的运行时。

magit（Git）和 org-mode（文档）仍然是同类最强，但这两个能力在 Zellij + Neovim/Kakoune 的组合中可以被替代（lazygit + markdown 渲染）。对于不需要 org-mode 深度工作流的场景，Emacs 的额外体积不产生等量价值。

---

## 六、Zed / 传统现代 GUI 流派：审计冗余

基于 Rust、GPU 渲染，追求全自动协作开发环境。但 AI 时代编辑器退化成"审计平台"，Zed 沉重的项目管理和复杂 UI 交互反而变成视觉和思维噪点。

---

## 七、四者横向对比

### 定位

| 维度 | Neovim | Kakoune | Helix | Emacs |
|------|--------|---------|-------|-------|
| 语言 | C + Lua | C（原生） | Rust | C + Elisp |
| 定位 | 全功能集成平台 | 纯编辑组件 | 无配置编辑器 | 可扩展计算平台 |
| 哲学 | 编辑器吞噬一切 | Unix 组合 | 只做编辑 | 一切可编程 |
| 首次发布 | 2014（fork 自 Vim 7） | 2004（fork 自 Vim） | 2020 | 1984（GNU 项目） |

### 集成方式

| 维度 | Neovim | Kakoune | Helix | Emacs |
|------|--------|---------|-------|-------|
| 终端 | `:terminal`（内嵌） | 外部 zellij/tmux | 无 | `eshell`/`vterm` |
| Git | fugitive/neogit（内嵌） | 外部 lazygit | 内置 gutter | magit（内嵌） |
| 文件管理 | telescope/nvim-tree | 外部 yazi/lf | 无 | dired |
| 外部→编辑器 | RPC（可被控制） | 命令接口（强） | 无 | ELisp RPC |
| 编辑器→外部 | 内嵌一切 | 不做（纯组件） | 无 | 内嵌一切 |

### 扩展能力

| 维度 | Neovim | Kakoune | Helix | Emacs |
|------|--------|---------|-------|-------|
| 扩展语言 | Lua / Vimscript | Kakscript（小） | 无 | **Elisp**（图灵完备） |
| LSP | 内置（0.10+） | 外部 kak-lsp | 内置 | 内置 eglot |
| Treesitter | 内置（0.10+） | 外部 | 内置（Rust 原生） | 内置 treesit |

### 安装体积（NixOS 实测）

| 编辑器 | 二进制 | Store Closure |
|--------|--------|---------------|
| Helix | ~16 KB | **259 MB** |
| Neovim | 6.8 MB | **250 MB** |
| Kakoune | ~2 MB | **~180 MB** |
| Emacs-nox | 8.7 MB | **925 MB** |

### 演进速度（GitHub 近一年）

| 编辑器 | 年提交数 | 贡献者数 | 发布节奏 |
|--------|---------|---------|---------|
| Neovim | ~5200 | ~439 | 每月 nightly |
| Kakoune | ~800 | ~150 | 按需发布 |
| Helix | ~920 | ~414 | 每季度 |
| Emacs | ~3420 | ~329 | 每年大版本 |

---

## 八、决策链

```
[编辑器选型决策]
│
┌──────────────────┼──────────────────┼──────────────────┐
│                  │                  │                  │
[全功能集成]        [Unix 组合]        [极简一次性]        [深度定制]
│                  │                  │                  │
Neovim + Neovide    Kakoune + Zellij   Helix              Emacs
│                  │                  │                  │
LSP/Git/AI 全内嵌   编辑器只管编辑      零配置 SSH 快编     Elisp 无限可能
配置复杂但上限高     启动快、职责清晰    无 session 持久化   体积大、启动慢
```

> 嵌入式脚本语言的选型（Rune vs Steel）见 [嵌入式脚本语言选型](embedded-script-languages.md)

### 落地执行结论

1. **编辑器侧**：放弃在现阶段继续死守 Helix + Steel 的空中楼阁，果断全面切回 Neovim 生产力阵营。在 Neovim 生态下，成熟的 Fennel (Lisp-to-Lua) 编译器或 conjure 插件能在享受现代编辑器最顶尖、最庞大的生态基础设施（LSP 补全、Git 协同、AI 结对编程）的同时，继续用 Lisp 小括号和宏来编写所有配置。

2. **轻量场景**：Kakoune + Zellij 组合覆盖需要快速启动、干净分屏、外部工具协作的场景。Kakoune 不试图替代 Neovim 的全功能集成，而是在 Unix 组合哲学下提供纯粹的编辑体验。

3. **长线观望**：保持对 Helix main 分支的增量追踪。鉴于 Helix 社区极端的严苛审核，未来他们一旦将 Steel 稳定合并发布，其产出的必然是一个工艺品级别的高性能编辑器。到了那一天，Lisp 本能可以支撑在一分钟内重返战场、降维接管。

---

## 九、核心洞察

### 设计哲学：反"TUI 模仿 GUI"

大量 TUI 工具（yazi、lazygit、各种现代 CLI 文件管理器）正在犯同一个错误：**在字符终端里堆砌伪 GUI 元素**——圆角边框、多层级面板、花哨图标、阴影效果。

- **浪费空间**：Box-drawing 字符占据宝贵的屏幕面积，压缩实际内容密度
- **操作低效**：为了"好看"增加大量无意义的视觉层级，反而干扰快速扫描
- **低幼化审美**：把字符终端当成"简陋的 GUI"来模仿，既没有 GUI 的真正交互能力，又丧失了 TUI 的纯粹性
- **风格割裂**：多个 TUI 工具各自为政，圆角样式、配色方案、边框粗细难以统一

**Neovide 的降维打击**：Neovide 不是"TUI"，而是**真正的 GUI**——亚像素级渲染、无边框悬浮、物理微动画、100% 视觉统一。所有元素在同一个 Grid 像素块中渲染，风格完全一致。

### 编辑器的本质矛盾

- **集成 vs 纯净**：Neovim 全功能但配置复杂；Kakoune 干净但依赖外部工具链；Helix 极简但拒绝集成
- **平台 vs 组件**：Neovim 是平台（吞噬外部工具），Kakoune 是组件（被外部工具组合），Helix 是孤岛
- **现代化 vs 实用性**：Zed/VS Code 功能强大但在审计场景下变成冗余负担
- **稳定 vs 前沿**：Helix + Steel 是理想方案但尚未成熟

### 真实需求本质

不是"写代码"，而是"审计 AI 生成的代码"。这意味着：
- 需要快速浏览大量文件（目录树 + 分屏）
- 需要实时运行命令（内置终端或外部 zellij）
- 需要查看 Git 历史（Fugitive/lazygit 或原生 Git）
- **需要视觉纯净**（拒绝 TUI 圆角污染，追求 100% 风格统一）

---

## 交叉引用

| 文档 | 关联论点 |
|------|---------|
| [Vim 移动逻辑批判](vim-movement-critique.md) | hjkl 历史缺陷、生物力学分析、Boon 模态编辑方案 |
| [嵌入式脚本语言选型](embedded-script-languages.md) | Helix Steel vs Neovim Rune 技术对比 |
| [编程语言选型](language-selection.md) | Rust/Lisp/Python/Nushell 四象限定位与淘汰语言批判 |
