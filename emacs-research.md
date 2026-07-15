# Emacs 调研：Neovim / Emacs / Helix 三者对比

> 从 Vim 方向键的不合理性出发，对比三大模态编辑器的架构、体积、演进和生态。

---

## 一、为什么讨论这个话题

Vim 的 hjkl 基于 1976 年 ADM-3A 终端键盘上的箭头标识，是历史妥协而非人体工学设计。现代替代方案：

- **Boon**：Emacs 原生模态编辑，空间优先键位分配（右手=移动，左手=操作），不依赖 hjkl
- **Flash/Leap**：Neovim 的视觉标签化跳转，直接取代 hjkl 的中长距离移动
- **Helix**：内置 Kakoune 风格的 selection-first 模式，不需要 hjkl 的 operator-motion 语法

→ 详细批判见 [Vim 移动逻辑批判与现代输入范式](vim-movement-critique.md)

---

## 二、三者定位

| 维度 | Neovim | Emacs | Helix |
|------|--------|-------|-------|
| 语言 | C + Lua | C + Elisp（45 年历史） | Rust |
| 定位 | Vim 的现代化分支 | 可扩展计算平台 | 现代无配置编辑器 |
| 哲学 | 折中：保留 Vim 兼容性 + 现代扩展 | 极端：一切可编程，Emacs Lisp 是操作系统 | 极端：开箱即用，拒绝插件系统 |
| 首次发布 | 2014（fork 自 Vim 7） | 1984（GNU 项目） | 2020 |
| 许可证 | Apache 2.0 | GPL v3 | MPL 2.0 |

---

## 三、安装体积对比

NixOS 实测（单二进制 / 含 runtime 的 store path / 含所有依赖的 store closure）：

| 编辑器 | 版本 | 二进制 | Store Path | Store Closure |
|--------|------|--------|-----------|---------------|
| **Helix** | 25.07.1 | ~16 KB（动态链接） | 28 MB | **259 MB** |
| **Neovim** | 0.12.3 | 6.8 MB | 38 MB | **250 MB** |
| **Emacs-nox** | 30.2 | 8.7 MB | 319 MB | **925 MB** |
| **Zed** | 1.6.3 | — | 346 MB | **1478 MB** |

分析：
- **Helix 和 Neovim 最轻量**：runtime 28-38MB，closure 主要是共享库（fontconfig、libvterm 等）
- **Helix 的 16KB 二进制**：纯动态链接，核心逻辑在 Rust shared lib 中，store path（28MB）才是真实安装体积
- **Emacs 中等**：319MB（内嵌 Elisp 解释器 + 大量内置功能），closure 925MB
- **Zed 最重**：346MB（GPU 渲染引擎 + Electron-like 架构），closure 1478MB——比 Emacs 多 60%
- Emacs 的体积换来的是**零外部依赖**——LSP、Git、终端、包管理全部内置

---

## 四、演进速度对比

GitHub 近一年数据（2025-2026）：

| 编辑器 | 年提交数 | 贡献者数 | 发布节奏 | 关键里程碑 |
|--------|---------|---------|---------|-----------|
| **Neovim** | ~5200 | ~439 | 每月 nightly | 0.10（内置 LSP/补全/Treesitter） |
| **Emacs** | ~3420 | ~329 | 每年大版本 | 29（内置 treesit/eglot）、30（原生编译） |
| **Helix** | ~920 | ~414 | 每季度 | 24.03（稳定版）、Steel 插件系统仍待合并 |

分析：
- **Neovim 活跃度最高**：年提交 5200+，社区驱动的快速迭代
- **Emacs 稳定演进**：年提交 3400+，但大量是 GNU 上游的基础设施变更
- **Helix 最年轻但贡献者密度高**：414 贡献者产出 920 提交，人均产出高但总量有限。同时存在显著的保守主义倾向——插件系统（Issue #122）和 Steel 绑定（PR #8675）延宕数年未合并，审核理念走向激进克制，宁可让生态停滞也不愿接受"不够完美"的方案
  → 详细分析见 [编辑器选型分析](editor-selection-2026.md) §一

### Emacs 23 以来的关键变化

| 版本 | 年份 | 关键变化 |
|------|------|---------|
| 23 | 2009 | 内置包管理器 `package.el`、Unicode 全面支持 |
| 24 | 2012 | 内置 CEDET（代码智能）、`lexical-binding` 成为默认 |
| 25 | 2016 | 内置 `xref`/`project`、JSON 解析器、子进程改进 |
| 26 | 2018 | 线程支持（`thread-first`/`thread-last`）、原生 JSON |
| 27 | 2020 | 原生 JSON 解析（C 层）、`libjansson` 集成、子进程异步改进 |
| 28 | 2021 | 移除对 XEmacs 兼容代码、`use-package` 未内置但文档推荐 |
| 29 | 2023 | **treesit**（内置 tree-sitter）、**eglot**（内置 LSP）、原生 Elisp 编译、Wayland 支持 |
| 30 | 2024 | 原生编译普及、`grep-buffer` 改进、Markdown 模式改进 |

**核心趋势**：Emacs 29 是分水岭——treesit + eglot 让 Emacs 终于有了现代编辑器的基础设施，不再需要 evil/eglot/lsp-mode 等第三方插件就能获得基础的编辑体验。

---

## 五、扩展能力对比

| 维度 | Neovim | Emacs | Helix |
|------|--------|-------|-------|
| 扩展语言 | Lua / Vimscript | **Elisp**（图灵完备） | 无（设计上拒绝） |
| 扩展加载 | lazy.nvim / packer | `use-package` + MELPA | N/A |
| LSP | 内置（0.10+） | 内置 eglot（29+） | 内置（开箱即用） |
| Treesitter | 内置（0.10+） | 内置 treesit（29+） | 内置（Rust 原生） |
| 补全 | 内置（0.8+）/ blink.cmp | 内置（30+）/ corfu / company | 内置 |
| Git 集成 | fugitive / neogit | magit（公认最强 Git 客户端） | 内置 gutter |
| 终端 | `:terminal` | `eshell` / `vterm` / `ansi-term` | 无内置终端 |
| 多光标 | vim-visual-multi / multicursor | `iedit` / `multiple-cursors` | 内置 selection 模式 |

关键差异：
- **Emacs 的扩展能力是维度级碾压**：Elisp 是图灵完备的编程语言，Emacs 本身就是一个 Lisp 运行时。你可以在 Emacs 里写邮件、看新闻、调试程序、管理文件系统——这不是"插件"，是"用 Elisp 重写了一切"
- **Neovim 的扩展是"编辑器增强"**：Lua 脚本主要用于增强编辑体验，不是构建独立应用
- **Helix 的无插件是刻意设计**：团队认为插件系统会导致"配置地狱"，选择内置一切

---

## 六、生态系统对比

| 维度 | Neovim | Emacs | Helix |
|------|--------|-------|-------|
| 包仓库 | MPA（mason）、Minitheon | **MELPA**（7000+ 包） | N/A |
| 社区规模 | 大（Vim 生态溢出） | 极大（40 年积累） | 小（2020 年起步） |
| 企业采用 | 高（Vim 兼容性） | 中（科研/运维/法律） | 低（新兴） |
| 插件质量 | 高（blink.cmp、fidget.nvim 等） | 极高（magit、org-mode、tramp） | N/A |

### Helix 的生态预测

Helix 拒绝内置插件系统，但社区一直在推动 Steel Scheme 作为扩展语言。截至 2026 年中，Steel 仍未合并到 main 分支。

**预测**：
- 短期（1-2 年）：Helix 保持"无插件"状态，靠内置功能覆盖 80% 场景
- 中期（3-5 年）：如果 Steel 合并，Helix 将获得类似 Emacs 的可编程能力，但生态从零开始需要 5-10 年积累
- 长期风险：如果 Steel 始终不合并，Helix 可能被 Neovim 的生态优势边缘化

→ Helix + Steel 的详细分析见 [编辑器选型分析](editor-selection-2026.md)

---

## 七、简易入门：曾经精通，多年未用

> 目标读者：用过 Emacs，但已经很多年没碰了。需要快速恢复生产力，不需要从零学起。

### 8.1 你离开后发生了什么

Emacs 29（2023）是分水岭。你离开时可能还在用 24-26：

| 你离开时 | 现在（30+） | 变化 |
|:---|:---|:---|
| 需要装 evil-mode 才能用 Vim 键位 | 内置 `viper-mode`（基础 Vim 兼容） | 不够好，evil-mode 仍然推荐 |
| 需要装 eglot/lsp-mode | **eglot 内置**（29+） | `M-x eglot` 直接连接 LSP server |
| 需要装 tree-sitter 包 | **treesit 内置**（29+） | 语法感知编辑，不再需要第三方 |
| 手动编译 C 扩展 | **原生编译**（29+） | Elisp 编译为原生代码，速度提升显著 |
| X11 | **Wayland 原生支持**（29+） | 不再需要 XWayland |
| 手动管理包 | `use-package` + MELPA | 包管理体验现代化 |

### 8.2 最小可用配置

一个恢复生产力的最小 `init.el`：

```elisp
;; 包管理
(require 'package)
(setq package-archives '(("melpa" . "https://melpa.org/packages/")
                         ("gnu" . "https://elpa.gnu.org/packages/")))
(package-initialize)

;; 基础体验
(tool-bar-mode -1)        ; 去掉工具栏
(menu-bar-mode -1)        ; 去掉菜单栏
(scroll-bar-mode -1)      ; 去掉滚动条
(global-display-line-numbers-mode 1) ; 行号
(setq inhibit-startup-screen t)      ; 跳过启动画面

;; 主题
(use-package gruvbox-theme
  :ensure t
  :config (load-theme 'gruvbox-dark-medium t))

;; 补全
(use-package corfu
  :ensure t
  :init (global-corfu-mode))

;; LSP（内置 eglot，不需要装 lsp-mode）
;; M-x eglot 即可连接到对应 server

;; Git
(use-package magit
  :ensure t
  :bind ("C-x g" . magit-status))
```

### 8.3 关键快捷键速查

你可能已经忘了但每天都需要的：

| 操作 | 快捷键 | 说明 |
|:---|:---|:---|
| 执行命令 | `M-x` | 万能入口，输命令名 |
| 保存 | `C-x C-s` | 肌肉记忆应该还在 |
| 搜索 | `C-s` / `C-r` | 增量搜索，向前/向后 |
| 跳转 | `M-g g` | 跳到指定行 |
| 窗口分割 | `C-x 2`（上下）/ `C-x 3`（左右） | 分屏 |
| 切换窗口 | `C-x o` | 在分屏间跳转 |
| 关闭缓冲区 | `C-x k` | 关闭当前 buffer |
| undo | `C-/` 或 `C-_` | 撤销（不是 `C-z`，那是挂起） |
| 重做 | `C-g` 后 `C-/` | Emacs 没有直接的 redo，需要 undo-tree 或 vundo |

### 8.4 和 Neovim 的心智切换

如果你同时用 Neovim，最大的心智冲突：

| 概念 | Neovim | Emacs |
|:---|:---|:---|
| 模式 | Normal/Insert/Visual（显式切换） | 区分不太严格（有 minor-mode 但不强制） |
| 命令 | `:command` | `M-x command` |
| 配置语言 | Lua | Elisp（学习成本更高，但能力也更强） |
| 插件管理 | lazy.nvim | `use-package` + MELPA |
| 退出 | `:q` / `ZZ` | `C-x C-c`（不是 `:q`） |
| 强制退出 | `:q!` | `C-x C-c`（未保存会提示） |

### 8.5 NixOS 配置

NixOS 上 Emacs 的正确姿势：

```nix
programs.emacs = {
  enable = true;
  package = pkgs.emacs-pgtk;  # Wayland 原生
  extraPackages = epkgs: [
    epkgs.use-package
    epkgs.corfu
    epkgs.magit
    epkgs.gruvbox-theme
  ];
};
```

或者用 developMode + 手动管理（更灵活）：

```nix
programs.emacs = {
  enable = true;
  package = pkgs.emacs-pgtk;
};
# developMode 时 init.el 指向 ~/Configuration/emacs/
```

---

## 八、决策矩阵

| 你的需求 | 推荐 | 理由 |
|---------|------|------|
| Vim 兼容 + 现代扩展 | **Neovim** | Vim 生态继承 + 0.10+ 内置 LSP/Treesitter |
| 极端可编程 + 一切内置 | **Emacs** | Elisp 图灵完备 + magit/org-mode/eglot 无出其右 |
| 零配置 + 现代默认 | **Helix** | 开箱即用，不需要任何配置 |
| 人体工学优先 | **Emacs + Boon** | 空间优先键位分配，彻底告别 hjkl |
| 新建配置（无历史包袱） | **Emacs** | 不需要迁移 Vim 键绑定，从零开始最自由 |

---

## 交叉引用

| 文档 | 关联论点 |
|------|---------|
| [Vim 移动逻辑批判](vim-movement-critique.md) | hjkl 历史缺陷、生物力学分析、Boon/Colemak 方案 |
| [编辑器选型分析](editor-selection-2026.md) | Neovim 0.12 架构、Helix + Steel 前景、Zed 审计冗余 |
| [LLM 基础认知与心智模型](llm-fundamentals.md) | AI 时代编辑器退化为"审计平台"的背景 |
