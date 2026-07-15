# Neovim 0.12 + Neovide 极简代码审计平台架构规约（For AI）

> **用途**：本文档是专为 AI 深度优化的系统提示词与架构规约。可直接作为 Context 或 `.cursorrules` / `system_prompt` 喂给任何 AI 助手。  
> **最后更新**：2026-06-12

---

## 📌 用户画像与核心审美（User Profile & Aesthetics）

**技术背景**：用户是拥有 10 年 Emacs、7-8 年 Neovim、2-3 年 Helix 经验的顶级开发者。精通 Rust, Lisp (Scheme), Haskell, Scala, Go, Lua (OpenResty/Redis 级别) 等二十多种语言。

**工具定位**：编辑器在此场景下已完全退化/升级为"纯粹的代码审计平台"。不承担大量增删代码工作，核心关注：
- 项目文件导航
- Git 历史追溯/变基/Diff 审计
- 内置终端联动
- 高能文本检索

**极致审美与偏好**：

1. **极度反感网红 TUI 的低幼化（Juvenile）设计**
   - 坚决拒绝在字符终端里用 Box-drawing 字符去堆砌圆角、层级、虚假 GUI 边框、花哨 Nerd 图标
   - 这被视为浪费屏幕空间、分散注意力的低效设计

2. **重度集成度要求**
   - 要求 100% 的一体化（Integration）
   - 拒绝使用 Zellij、Tmux、Yazi、Lazygit 等外部独立工具拼接
   - 所有功能必须由 Neovim 0.12 核心（C Core PTY）分发
   - 确保在 Neovide GUI 下享用完全统一的亚像素字体渲染与物理微动画

3. **对 Lua 的态度**
   - 用户精通 Lua 但极度讨厌它
   - 在 Neovim 配置中，Lua 只能作为声明式（Declarative）的键值对配置工具（类似 JSON/TOML）
   - 严禁引入复杂的 Lua 闭包、异步回调地狱或架空内核的重型 Lua 插件生态

---

## 🛠️ Neovim 0.12 核心架构原则（Neovim 0.12 Core Rules）

在为用户生成或修改 `init.lua` 时，AI 必须严格遵守以下基于 Neovim 0.12 最新特性的技术约束：

### 1. 原生下沉原则（Sink to C Core）

**自动补全**：
- ❌ 禁止使用 `nvim-cmp`、`coq_nvim` 及其各种庞杂的 Lua 源（Sources）
- ✅ 必须且仅能使用 0.12 内置的纯 C 层异步补全：`vim.opt.autocomplete = true`

**包管理器**：
- ❌ 禁止使用 `lazy.nvim` 或 `packer.nvim`
- ✅ 必须直接使用 0.12 原生内置的 `vim.pack` 机制进行声明式配置，并维护 `nvim-lock.json`

### 2. 视觉纯净度最大化（Max Information Density）

- ❌ 禁止安装 `noice.nvim`、`fidget.nvim` 等伪 GUI 弹窗
- ✅ 0.12 原生 `ui2` 已消除 "Press ENTER" 中断，消息直接在命令行或原生 Buffer 流中呈现
- ✅ 必须将状态栏彻底关闭（`vim.opt.laststatus = 0`），释放 100% 屏幕空间给文本本身
- ✅ 必须抹平分屏线颜色（`WinSeparator` 设为 `none`），让视窗物理边界消融

### 3. 标准的现代正则搜索（Very Magic Regex）

- 用户极其讨厌 Vim 默认的反向转义正则（如 `\(...\)`、`\|`）
- ✅ 必须在所有键盘映射中，为 `/`、`?`、`:%s/` 等搜索/替换前置自动化注入 `\v`（Very Magic 模式）
- ✅ 使其行为 95% 等价于 PCRE/Perl/Python 标准正则（如 `/\v(error|panic).\+`）

---

## 📐 核心三大支柱配置规约（The Three Pillars）

### 1. Tab-local 级别项目隔离（One Project Per Tab）

**核心心智模型**：把每个 Tab 视为一个独立项目

**路径锁定**：
- ✅ 必须使用 2016 年内核引入的 `:tcd` (Tab-local Current Working Directory) 命令来硬核隔离每个标签页
- ❌ 禁止使用全局 `:cd` 或窗口级 `:lcd`

**顶部项目栏净化**：
- ❌ 禁止使用带彩色圆角的 `bufferline`
- ✅ 必须通过重写 `vim.opt.tabline`，用纯文本计算并提取当前 Tab 的 `:tcd` 路径最后一级文件夹名
- ✅ 将其渲染为最纯粹的项目名称列表（形如 `1:rust-service 2:go-api*`）

### 2. 脱水级 Neo-tree 配置（Dehydrated Neo-tree）

用户认可 Neo-tree 的操作逻辑，但极其讨厌其默认的视觉噪点。

**配置约束**：

1. **图标净化**：
   - ✅ 必须将所有 icon（包括文件夹、文件、Git 状态符号）设为空字符串 `""`
   - ✅ Git 状态仅通过纯文本的颜色高亮（Color Hint）暗示
   - ❌ 拒绝 `[M]`, `[U]` 等字符框

2. **缩进连线**：
   - ✅ 必须将树状缩进连线彻底关闭（`with_markers = false`）
   - ✅ 文件层次纯靠空格（Padding）缩进
   - ✅ 维持像 Python/Haskell 源码一样的纯净感

3. **Tab 作用域绑定**：
   - ✅ 必须通过 `cwd_target.sidebar = "tab"`，让 Neo-tree 实例完美属于当前的 Tab 作用域
   - ✅ 当用户在不同 Tab（项目）间切换时，侧边栏必须微秒级自动动态刷新为对应项目的根目录

### 3. C Core 内置终端接管（PTY Buffer Takeover）

**彻底废除 Zellij**：
- ✅ 所有多任务流转必须通过 Neovim 内置的 `:terminal` 实现

**进程目录承袭**：
- ✅ 在 0.12 架构下，因 Tab 锁定了 `:tcd`，通过 `:terminal` 唤起的新进程必须完美承袭当前项目的 CWD 路径

**终端 Buffer 化**：

1. **模式切换**：
   - ✅ 在终端模式下按 `Esc` 必须无感重置回 Normal 模式（利用映射 `[[<C-\><C-n>]]`）

2. **只读日志审计**：
   - ✅ 退回 Normal 模式后，终端将作为一个只读的 Vim Buffer 存在
   - ✅ AI 必须理解用户此时可以使用刚才配置的标准正则（`\v`）对终端输出的数万行 AI 审计日志、编译日志进行凌厉的文本检索
   - ✅ 直接进行 `y` (yank) 复制

3. **路径跳转**：
   - ✅ 光标移动到日志内的特定路径（如 `src/main.rs:42`）时
   - ✅ 支持通过原生 `gF` 键，让左侧代码视窗直接精准跳转

---

## 📋 NixOS 声明式环境架构（NixOS Pure Infrastructure）

**软硬分离原则**：
- ❌ 禁止使用 NixVim / NVF 这类将 Lua 配置混进 Nix 语言的反人类框架
- ✅ NixOS 的职责：负责环境的 Purity（纯净性）
- ✅ 通过 Nix Flake 或 Home-Manager 将独立的本地纯 Lua 配置目录（`~/.config/nvim`）以符号链接形式挂载（`xdg.configFile."nvim".source`）

**反 Mason 约束**：
- ❌ 由于 NixOS 缺乏标准 FHS 路径，通过 Mason 下载的二进制 LSP 会直接因找不到动态链接库而崩溃
- ❌ 严禁在配置中引入 Mason
- ✅ 所有编译器、调试器、LSP（如 `nil`, `gopls`, `rust-analyzer`, `pyright`）、`ripgrep`, `fd` 必须由 NixOS 系统层或 Home-Manager 在环境包（`environment.systemPackages`）中统一声明式注入

---

## 💡 AI 生成代码示例检查（Sanity Check For AI）

当你（AI）准备输出任何关于 Neovim 0.12 的配置时，请对照以下行为进行自检：

| 检查项 | 错误示例 | 正确做法 |
|--------|----------|----------|
| 补全插件 | `require('cmp')` | ❌ 违反 0.12 原生补全原则 |
| 图标装饰 | `nerd_font_icons` 或线条字符 | ❌ 违反去低幼化纯文本原则 |
| 目录切换 | `:cd` 或 `:lcd` | ❌ 违反项目隔离原则，必须用 `:tcd` |
| LSP 安装 | Mason 启动逻辑 | ❌ 在 NixOS 环境下无法运行 |
| 包管理器 | `lazy.nvim` / `packer.nvim` | ❌ 必须用 `vim.pack` |
| 状态栏 | `lualine` / `airline` | ❌ 必须 `laststatus = 0` |
| 弹窗美化 | `noice.nvim` / `fidget.nvim` | ❌ 使用原生 `ui2` |

---

## 🎯 使用建议

1. **保存位置**：将此文档保存在 NixOS 管理的项目配置目录下（如 `~/Configuration/nixos/docs/`）

2. **喂给 AI**：在每次与 AI 讨论 Neovim 升级和插件调整时，第一步就把这段"规约"直接塞给它

3. **插件迁移**：在这个架构标准下，如果你目前手头积攒的老插件里有需要进行复杂的跨 Tab 数据通信，或者你不太确定 0.12 原生能否平替的，可以把插件名字发出来，直接用这套"AI 规约"的严谨逻辑把它当场拆解并重构掉

---

## 📚 相关文档

- [Neovim 0.12 部署计划](../../Configuration/nixos/docs/memos/neovim-0.12-deployment-plan.md)
- [编辑器选型分析 2026](./editor-selection-2026.md)
