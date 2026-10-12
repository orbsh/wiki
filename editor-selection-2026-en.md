# Editor Selection Analysis: 2026 Technical Deep Dive

> **Languages:** [English](editor-selection-2026-en.md) (primary) · [中文](editor-selection-2026.md)
> **Related documents**: for a technical deep dive comparing embedded scripting languages (Rune vs Steel), see [Embedded Script Language Selection](embedded-script-languages-en.md)

## 1. Helix's Crisis of Conservatism

### 1.1 A Flawless Blueprint for the Low-Level Architecture

- When Helix officially planned its plugin system — delayed for several years — the core embedded extension language it settled on was precisely Steel Scheme ([discussion](https://github.com/helix-editor/helix/discussions/3806)).
- A high-performance modern TUI editor developed in pure Rust (Helix), embedding an immutable Lisp virtual machine also written in pure Rust and natively thread-safe (Steel): the rigor of its architecture, its startup speed, and its theoretical ceiling are fully capable of delivering a dimensional-reduction strike against the traditional Neovim + Lua.

### 1.2 Doubt: A Rust Open-Source Spectacle More Conservative Than C Projects

- **The core contradiction**: Rust's core dividend lies in its extremely strong compiler safety net and macro system; it natively supports and encourages developers to carry out bold, fearless "fast-paced evolution and large-scale refactoring".
- **The factual charge**: Yet the Helix team's review philosophy has walked to the radical opposite. On the plugin system (Issue #122) and the merge of the Steel binding (PR #8675), the project has shown an almost harsh, elitist restraint. In pursuit of absolute zero-configuration (batteries-included) and 100% stability, it would rather let tens of thousands of geeks worldwide sit and wait in place for three years; [to this day the ecosystem still requires pulling a specific development branch and compiling it to get a hard-won taste](https://sraj.me/notes/helix-steel/).
- **The comparative inference**: By contrast, the ancient Neovim community, written in C, in order to break the rigidity of traditional Vim, instead behaves with utter abandon and pragmatism in introducing a Lua scripting layer, advancing multithreaded async, and iterating on high-frequency features.

---

## 2. Neovim 0.12 + Neovide: The Full-Featured Integration Empire

### Architecture (Decoupled and Pushed Down)

Neovim 0.12 is undergoing a revolution of "de-bloating Lua". It has pushed down into the pure C core the dynamic features that previously had to rely on third-party Lua plugins (completion, package management, Treesitter incremental selection, and the like). For developers fluent in multiple languages, Lua here degenerates into declarative configuration akin to JSON/TOML — no longer must one face a bloated Lua callback hell.

### Information Density and Aesthetics

Ceiling-level. Combined with Neovide's subpixel-level rendering, borderless floating, and physical micro-animations, every visual element is unified inside one and the same Grid pixel block. By turning off the status line (`laststatus=0`), you get a 100% pure-text matrix with no rounded corners, no pseudo-GUI borders, and no gaudy icon pollution.

### Integration Style: The Editor Devours Everything

Neovim's integration direction is **inward**: `:terminal` embeds a PTY, Fugitive embeds Git, Telescope embeds file search. The editor itself is an integration platform, and external tools are pulled in to run inside the buffer. RPC (`nvim --remote`) also lets the editor be controlled from outside, but the community's center of gravity is "internal extension".

The price is configuration complexity — Lua callback hell, plugin conflicts, startup optimization — and the user has to shoulder all of it.

---

## 3. Kakoune: A Pure Editing Component in the Unix Philosophy

### Architecture (client-server)

Kakoune's design starting point is the exact opposite of Neovim's: the editor is not a platform, it is **a component to be integrated**.

- **client-server architecture**: the `kak` command starts a server process that manages buffers and undo history, `kak -c <session>` connects to an existing instance, and `echo 'buffer README.md' | kak -p <session>` injects commands into a running instance
- **No terminal**: there is no `:terminal`; the terminal is managed by zellij/tmux
- **No file management**: there is no built-in file tree; yazi/lf runs in another pane
- **No Git integration**: there is no fugitive; lazygit runs in another pane

### External Interaction: Command Interface > Embedding

Kakoune exposes a clean command interface, and external tools can **actively drive** the editor:

```
# Drive a kak instance from a shell script
echo 'select All; exec d' | kak -p my_session    # clear the current buffer
echo 'buffer src/main.rs' | kak -p my_session     # switch files

# In zellij: kak takes one pane, nushell runs commands in another pane
# The two are glued by the terminal PTY, with zero coupling
```

### Philosophy: Unix Composition > Editor Monolith

The correct way to use Kakoune is as a **three-piece combination**:

```
┌─────────────────────┬──────────────────────┐
│   zellij            │   kak                │
│   (window/session/  │   (pure text edit)   │
│    terminal mgmt)   │                      │
├─────────────────────┼──────────────────────┤
│   nushell           │   lazygit            │
│   (tasks/scripts)   │   (Git operations)   │
└─────────────────────┴──────────────────────┘
```

The three are each independent, glued together by the terminal PTY. This is the Unix-philosophy way of composing things — each tool does one thing well and cooperates through pipes and the terminal, not an editor monolith.

**The advantage in the AI era**: once the editor degenerates into an audit interface, Kakoune's "I only mind the editing" is instead more honest. There is no need to embed a Git/terminal/AI toolchain in the editor; all of that runs in other panes of zellij, each keeping its own independent process lifecycle.

### Limitations

- The configuration language is Kakscript (Vimscript-like), with a small ecosystem
- No built-in LSP/Treesitter support (an external wrapper such as kak-lsp is needed)
- The plugin ecosystem is far smaller than Neovim's
- For scenarios that need deeply integrated AI completion and LSP diagnostic panels, it falls short of Neovim

---

## 4. Helix: The Modern, Pure, Serverless Editing Assassin

### Architecture (Stateless / Serverless)

Written in Rust, ready out of the box, refusing any plugins. **There is no central server process** — typing `hx` repeatedly in the same directory opens a new, independent process each time, and buffers and undo history are completely severed from one another.

### Integration Style: Refusing Integration

Officially there is neither a built-in terminal nor any IPC mechanism. Multitasking, Git, and directory-tree capabilities are all outsourced to external tools (such as Zellij). But unlike Kakoune, Helix offers no command interface for external control — it is a **closed, pure editor**.

### Positioning

Suited to short-lived, one-off editing tasks (SSH in and change a config file). Not suited to daily workflows that need session persistence, multi-file navigation, and cooperation with external tools.

---

## 5. Emacs: The Reduced Existence of a Programmable Operating System

### Positioning

Elisp is Turing-complete and Emacs is itself a Lisp runtime, so in theory its capability ceiling is the highest. But its size cost (a NixOS closure of 925 MB, 3.7× that of Neovim) and startup speed (2–5 s) become crippling weaknesses in the AI audit era — the editor has already degenerated into a review interface, and it does not need an operating-system-level runtime.

magit (Git) and org-mode (documents) remain the strongest in their class, but both capabilities can be replaced within a Zellij + Neovim/Kakoune combination (lazygit + markdown rendering). For scenarios that do not need org-mode's deep workflow, Emacs's extra size does not produce equivalent value.

---

## 6. Zed / the Traditional Modern GUI School: Audit Redundancy

Built on Rust with GPU rendering, pursuing a fully automated collaborative development environment. But in the AI era, as the editor degenerates into an "audit platform", Zed's heavy project management and complex UI interaction instead become visual and mental noise.

---

## 7. Four-Way Comparison

### Positioning

| Dimension | Neovim | Kakoune | Helix | Emacs |
|------|--------|---------|-------|-------|
| Language | C + Lua | C (native) | Rust | C + Elisp |
| Positioning | Full-featured integration platform | Pure editing component | Zero-config editor | Extensible computing platform |
| Philosophy | The editor devours everything | Unix composition | Editing only | Everything is programmable |
| First release | 2014 (fork of Vim 7) | 2004 (fork of Vim) | 2020 | 1984 (GNU project) |

### Integration Style

| Dimension | Neovim | Kakoune | Helix | Emacs |
|------|--------|---------|-------|-------|
| Terminal | `:terminal` (embedded) | external zellij/tmux | none | `eshell`/`vterm` |
| Git | fugitive/neogit (embedded) | external lazygit | built-in gutter | magit (embedded) |
| File management | telescope/nvim-tree | external yazi/lf | none | dired |
| External → editor | RPC (can be controlled) | command interface (strong) | none | ELisp RPC |
| Editor → external | embeds everything | does not do it (pure component) | none | embeds everything |

### Extensibility

| Dimension | Neovim | Kakoune | Helix | Emacs |
|------|--------|---------|-------|-------|
| Extension language | Lua / Vimscript | Kakscript (small) | none | **Elisp** (Turing-complete) |
| LSP | built-in (0.10+) | external kak-lsp | built-in | built-in eglot |
| Treesitter | built-in (0.10+) | external | built-in (Rust-native) | built-in treesit |

### Install Size (Measured on NixOS)

| Editor | Binary | Store Closure |
|--------|--------|---------------|
| Helix | ~16 KB | **259 MB** |
| Neovim | 6.8 MB | **250 MB** |
| Kakoune | ~2 MB | **~180 MB** |
| Emacs-nox | 8.7 MB | **925 MB** |

### Evolution Speed (GitHub, Past Year)

| Editor | Commits/year | Contributors | Release cadence |
|--------|---------|---------|---------|
| Neovim | ~5200 | ~439 | monthly nightly |
| Kakoune | ~800 | ~150 | released as needed |
| Helix | ~920 | ~414 | quarterly |
| Emacs | ~3420 | ~329 | one major version per year |

---

## 8. The Decision Chain

```
[Editor Selection Decision]
│
┌──────────────────────┼──────────────────────┼──────────────────────┐
│                      │                      │                      │
[Full integration]      [Unix composition]      [Minimal one-shot]      [Deep customization]
│                      │                      │                      │
Neovim + Neovide        Kakoune + Zellij        Helix                   Emacs
│                      │                      │                      │
LSP/Git/AI embedded     Editor edits only       Zero-config SSH edit    Elisp: unlimited potential
Complex config, high ceiling    Fast startup, clear duties    No session persistence    Large size, slow startup
```

> For the choice of embedded scripting language (Rune vs Steel), see [Embedded Script Language Selection](embedded-script-languages-en.md)

### Execution Conclusions

1. **On the editor side**: give up defending the castle in the air of Helix + Steel any longer at this stage, and decisively switch back wholesale to the Neovim productivity camp. Within the Neovim ecosystem, the mature Fennel (Lisp-to-Lua) compiler or the conjure plugin lets you keep writing all configuration with Lisp parentheses and macros while enjoying the most advanced and largest ecosystem infrastructure of any modern editor (LSP completion, Git collaboration, AI pair programming).

2. **Lightweight scenarios**: the Kakoune + Zellij combination covers scenarios that need fast startup, clean splits, and cooperation with external tools. Kakoune does not try to replace Neovim's full-featured integration; under the Unix composition philosophy it provides a pure editing experience.

3. **Long-term watching**: keep incrementally tracking Helix's main branch. Given the Helix community's extremely harsh review standards, once they stably merge and release Steel in the future, what they produce will inevitably be a craft-grade high-performance editor. On that day, Lisp instinct can carry a return to the battlefield within one minute for a dimensional-reduction takeover.

---

## 9. Core Insights

### Design Philosophy: Against "TUI Mimicking GUI"

A large number of TUI tools (yazi, lazygit, and various modern CLI file managers) are making the same mistake: **piling up pseudo-GUI elements inside a character terminal** — rounded borders, multi-level panels, gaudy icons, shadow effects.

- **Wasted space**: box-drawing characters occupy precious screen area and compress actual content density
- **Inefficient operation**: adding large amounts of meaningless visual hierarchy for the sake of "looking good" instead interferes with fast scanning
- **Infantilized aesthetics**: treating the character terminal as a "crude GUI" to imitate — it has neither the real interaction capability of a GUI nor the purity of a TUI
- **Fragmented style**: multiple TUI tools each go their own way; rounded-corner styles, color schemes, and border thicknesses are hard to unify

**Neovide's dimensional-reduction strike**: Neovide is not a "TUI" but a **true GUI** — subpixel-level rendering, borderless floating, physical micro-animations, 100% visual unity. All elements are rendered inside the same Grid pixel block, with a completely consistent style.

### The Editor's Essential Contradictions

- **Integration vs purity**: Neovim is full-featured but complex to configure; Kakoune is clean but depends on an external toolchain; Helix is minimal but refuses integration
- **Platform vs component**: Neovim is a platform (devours external tools), Kakoune is a component (composed by external tools), Helix is an island
- **Modernity vs practicality**: Zed/VS Code are powerful but become a redundant burden in audit scenarios
- **Stability vs the frontier**: Helix + Steel is the ideal solution but is not yet mature

### The Nature of the Real Need

It is not "writing code" but "auditing AI-generated code". This means:
- The need to quickly browse large numbers of files (directory tree + splits)
- The need to run commands in real time (a built-in terminal or an external zellij)
- The need to view Git history (Fugitive/lazygit or native Git)
- **The need for visual purity** (rejecting TUI rounded-corner pollution, pursuing 100% stylistic unity)

---

## Cross References

| Document | Related Arguments |
|------|---------|
| [Vim Movement Logic Critique](vim-movement-critique-en.md) | hjkl's historical flaws, biomechanical analysis, the Boon modal editing proposal |
| [Embedded Script Language Selection](embedded-script-languages-en.md) | Helix Steel vs Neovim Rune technical comparison |
| [Programming Language Selection](language-selection-en.md) | Rust/Lisp/Python/Nushell four-quadrant positioning and critique of eliminated languages |
