# 平铺 WM + launcher 全键盘下的浏览器使用模式

> **状态**: 已落地——tag 森林与控制台（2026-09-05）、Rust 全栈重写（2026-09-26 定案 / 09-28–29 落地，Python 树已删）、扩展面（事件日志 ns 8 + stdio 常驻宿主，2026-10-01 / 10-10）；未落地部分见 §九 与 §十一
> **触发**: qutebrowser 的 modal 输入法痛点 → 一种"非模态、最小 UI、全键盘、外部可控"的浏览器使用方式
> **定位**: 从全键盘 UX 工具升级为**信息流精炼**——浏览器是信息的主要流入途径
> **上游参考**: qutebrowser/qutebrowser#3444（OPEN）
> **通用内容组织模型**: 见 [标签森林](tag-forest.md)

---

## 一、定位与动机

**信息流精炼**：对多数场景而言，浏览器是信息的主要流入途径。本模式不止是"全键盘操作浏览器"的 UX 改进，更是把浏览器入口变成一条**信息流管道**：**捕获 → 属性化 → 精炼 → 消费**。

- **捕获**：每个流入的页面被外部可控地接收、归档（mudra 维护每页记录）。
- **属性化**：给每个页面打 **tag**（归属）与**评分**（importance / urgency）。
- **精炼**：按重要/紧急筛选、排序、沉淀，决定看什么、先看什么、归档什么。
- **消费**：跳转、打开、导出（后续可扩展：LLM 总结、转 Markdown）。

浏览器从"一个看网页的工具"变成"一个替你先分拣信息的入口"。这正建立在**外部可控**之上——只有完全可编程地列出、控制、标记每一页，才可能在它之上做分类与精炼。

要的是**一种「浏览器使用模式」，不是某一款浏览器**：在 Niri 下，复刻 qutebrowser"最大化 webview、最小化 UI"的观感，同时满足非模态、全键盘、外部可控，并与 launcher（walker）工作流自洽。

**硬约束 · 几乎 niri 独享**：整套模式依赖两个**结构性**条件——① WM 提供**接口/CLI 级精细控制**（窗口移动 / 聚焦 / 设列宽 / workspace 路由等 IPC 动作）；② 该 WM 的 **workspace 模型**适合「一实例一 workspace / 隔离维度路由」。

- **IPC 层面**：niri、hyprland 满足；cosmic-de 目前似乎还不满足（窗口层控制能力不足）。
- **但仅 IPC 不够，workspace 模型才是分水岭**：
  - **平铺类（如 hyprland）不合适**：workspace 有**上限**，单个 workspace 内能排布的窗口数也有限——实际限制了「隔离维度任意增长 + 每维度多窗口平铺」的形态。
  - **堆叠类（如 cosmic-de）**：即便 IPC 强到足以支撑，理论上可行，但**体验不佳**（无平铺布局优势）。
  - 结论：这套机制**几乎是 niri 独享**——niri 新建 workspace 廉价、可增长；且**单列内可上下叠放窗口**，实现灵活布局，常见场景如**开发者工具叠加摆放**（编辑器 + 终端 + 调试器同列上下）。

> 平铺式 WM 与现代软件预设布局的冲突（缩小即触发响应式错位）、终端场景为何交给终端复用器（Zellij，附带后台运行、进程管理）——完整论证见 [Linux 桌面工作流](linux-desktop-workflow.md)。

触发痛点有二：

- **qutebrowser 的 modal 输入法无解**：qutebrowser 有插入/normal 模式切换，但没有模式切换 hook（上游 #3444，autocmd/插件系统未实现），输入法在 normal 模式、文本框聚焦时吃键。无法干净地"只在插入模式开启输入法"。
- **扩展层控制力度不足**：Vimium / SurfingKeys 这类扩展进不了错误页（4xx/5xx 显示的 `chrome-error` 页）和内置页，控制不完整。

## 二、选型结论

- qutebrowser modal：输入法无解（上游 #3444），排除。
- 纯扩展层：错误页/内置页注入不了，不能完全控制，排除。
- Nyxt / EVE / uzbl：全可编程但系于 WebKit / Emacs / Lisp，门槛与兼容不划算。
- **chromium `--app` + CDP + 自研 mudra-keys 扩展 + okm(KV) 的 mudra**：折中，各层各司其职。容器不复刻 qutebrowser"嵌入引擎画标签条"，而是**完整 Chromium + 外部控制器**（自研扩展 + 错误页可控）。组织与精炼由 mudra 的 **tag 森林**承担。

## 三、架构（各层职责）

- **`--app` 每页一窗口**：无 omnibox、无标签条 → 最大化 webview。Niri 默认无标题栏 → 打开就是一张干净网页。"标签"（tab）不在浏览器里，由外部控制器（mudra）按 **tag 森林**管理。**CDP 操纵不了 Chromium 的 UI chrome**（标签条/omnibox 是浏览器自身 UI，非 CDP 能力范围）——最小 UI 只能靠 `--app`，不能用 CDP 去缩/藏 chrome。
- **CDP 脊梁**：控制浏览器里所有 target（含错误页、内置页）+ `Target.targetCreated/Destroyed` 事件实时同步到 KV 存储（okm/fjall，满足"关闭时更新列表"）+ 恢复。
- **mudra-keys 扩展（自研，2026-09-05 落地）**：普通页面的键盘驾驶——四模式状态机（normal/hint/insert/command）、qutebrowser 式底部状态栏（ctx·mode·tag 胶囊｜title·url·滚动%）、link hint、滚动命令 + `<N>%` 百分比跳转、`:set` 运行时改配置。MV3 **零构建**（源码即产物，`--load-extension=<repo>/frontend`，扩展根 = `frontend/`：`extension/` 放 content/SW，`shared/` 放与控制台面板共用的库——solid 运行时（全局 IIFE bundle）、bar、tag 胶囊组件）。决策记录在 mudra 仓库 `docs/ADR-self-maintained-extension.md`。取代 SurfingKeys 的动机：SurfingKeys MV2 构建在 chromium 139+ 被静默拒载，且交互面（tag 胶囊、状态栏数据源）需要与 mudrad 深度耦合，自研才能闭环。
- **KV 存储 + web 控制台**：**tag 森林**组织与精炼（多 tag 维度 × 页面）、isolated 实例隔离、窗口↔进程映射、url 记录与过滤、网站列宽记忆；控制台（`mudra ui`）是唯一管理面（早期 launcher 路线已废弃，见 §七）。
- **扩展面（2026-10 落地）**：页面生命周期事件落只追加的事件日志（`POST /events` 按 cursor 回放），常驻子进程扩展经 stdio 被宿主托管——线契约采用 **Induction**（与 aura 的 effector 同一份契约，mudra 零 aura crate 依赖）。决策见 §十一 与 mudra 仓 `docs/ADR-extension-protocol.md`。

  **引擎控制面（核实 2026-08）**：chromium 是唯一具备成熟 **CDP** 的引擎（`/json` 端点 + WebSocket，`Target.*`/`Page.*`/`Runtime.*` 完备控制面——目标/窗口/导航/注入，mudra 全依赖它）。替代引擎 Ladybird、Servo **都有**远程控制，但都走 **Firefox 远程调试协议（RDP）**——actor + TCP JSON 包，且都标着不完整（Servo `protocol.rs` 自注 "currently only supports JSON packets"；Ladybird `LibDevTools` 含 `FirefoxClient.*`/`Actor.*`，即 RDP actor 架构）——**不是 CDP**，也非成熟的窗口/会话管理表面，能力面远不够当 daily-driver 受控引擎。→ 引擎抽象方向：mudra 控制层抽象成 `BrowserEngine` 接口（chromium=CDP backend），等哪个引擎协议长成熟再补，而非现在改包。

## 四、资源占用（多窗口 vs 多实例）

Chromium 是 **process-per-page**（每页面一个 renderer，无论一窗多标签还是一个标签一窗）。因此：

- **同一实例内**开多个 `--app` 窗口 ≈ 一个窗口开同样多的标签：1 个 browser 进程 + 每页 1 个 renderer + 共享 cache/GPU。
- **多个独立实例**（不同 profile）= 各自 browser 进程 + 各自 cache，明显更重。

结论：**按 isolated 实例隔离**——每个 `isolated` 标签的维度一个独立实例（profile），并发实例数 = **隔离维度数**，而非页面/主题数。代价是多跑几个 browser 进程 + cache，换取 **cookie / 登录态 / 隐私**的硬隔离。非隔离标签不新增实例，在共享实例内多页。

## 五、地址输入、历史与命令（热键）

- "地址栏"由**扩展的命令模式 `:open`** 承担：候选 = 当前活动页（仅 console 角色，选中→focus）排在**历史行**之前（History collection，ns 7，`sim_score`/RRF 排名），选中历史即开该 URL，无候选 Enter 即按输入开 URL。
- 历史由 `/open`、`/add` 经 `history_bump` 写入，**不 bump epoch**（面板不可见 collection 不占 epoch 纪律）。
- 前进 / 后退 / 刷新 = CDP history 命令（`Page.getNavigationHistory` / `navigateToHistoryEntry` / reload），经统一动词面 `/nav` 暴露。
- 早期"launcher（walker）承担地址输入"方案随 launcher 层一起废弃（2026-09）。

## 六、内容组织：tag 森林（取代 session）

页面不再按单归属的 session 组织，而是落进**标签森林**（多棵树、树间不互斥、树内单选、无单一根；退化情形覆盖普通多标签与传统分类两端，完整模型见 [标签森林](tag-forest.md)）。

- **situation 树（必选、单选、默认 inbox）**：`inbox / work / personal / privacy`。决定**当前上下文**与隔离。新信息默认落 `inbox`，处理后移到对应维度值——**inbox 分流**。
- **importance / urgency / quality 树（两级、rank 排序）**：叶节点 `☆..☆☆☆☆☆`，`quality` 是第三条评分轴（价值之外的"完成度/可信度"维度）。评分即树，不是独立字段——域名/子域名规则表给叶 tag 打默认分（如 infoq 高重要、其用户贡献子域名低重要），手动覆盖。
- **state 流水树**：`to-read / reading / distilled / archived`——精炼流程的状态轴（与 situation 的"归属"、评分的"权重"正交）。
- **topic 树（普通、多选）**：`项目:x / 新闻 / 技术` 等可多选归属。

**隔离实例**：`isolated=true` 的 tag 命中 → 该内容运行在独立实例（独立 profile/cookie/进程）。目前 situation 四个子节点都隔离，互为互斥不会冲突；多命中取第一个是未定义行为，不特殊处理。

**pin 常驻（PWA）**：IM / RSS 等固定在某实例长期常驻、独立窗口，对应渐进式 Web 应用（**PWA**）——像原生 app 一样常驻的"应用槽"。

**术语**：**tab → page**（标签条里的"标签"改名 page，避免与 tag 混淆）。

**页面树（`parent_id`）**：一个页面里打开的另一个页面是**子页**（CDP `openerId`）——页与打开它的页形成**树**。既有标签森林（内容组织轴）之外，这是页面的**打开关系**轴：用于**链接挖掘**（从"高价值父页引出的子页"反向发现）与**分拣**。

## 七、窗口管理与热键

窗口管理目标：**统一切换**——mudra 的每个 page 窗口就是普通 niri 窗口，直接出现在 Alt+Tab（`recent-windows`）里，不需要在浏览器内部再维护一套自己的切换方式（像多标签浏览器那样的双层切换）。窗口过多只是副作用，需要时在 launcher 中过滤窗口即可；**最常用的依然是 Alt+Tab**。

- **不加 title 前缀、不做排除**：早期方案曾打算给 `--app` 窗口加 title 前缀以便从通用切换中过滤/排除——已废弃。单独 page 窗口能直接 Alt+Tab 切换、能在 launcher 中过滤，本身就是优势；专门的 mudra 窗口菜单不再必要。
- **mudra 专属视图交给 web 控制台**：控制台（`mudra ui`，chromium `--app` 载入本地面板）里显示的只有 mudra 的 page；实例启动时即记录地址、pid、窗口 id 等映射，不依赖窗口 title。**Mod+Space（Win+Space）绑定打开该控制台**（NixOS niri 配置，替代原 cwdhist 直达）。
- **无需 launcher 改造**：早期方案保留 launcher `p` 模式作为 page 热路径——已废弃。mudra 的全部管理面收敛到 web 控制台：**搜索过滤、操作（聚焦/关闭/移动等）、tag 森林树形展示**都在控制台完成。launcher 侧零改动（仅 Mod+Space 绑定）。
- **关闭语义**：mudra 主动关闭 → 从归档**删除**该页（不留痕）；通过 niri 关窗 / 意外（崩溃）关闭 → **保留**（仅标 closed，不删）——主动关不留痕，外部/意外关不丢。

**分拣：页面树 → workspace**：把某个页面连同它的**整棵子树**移到独立 workspace，用于归类（WM 层操作）。

**交互收敛（2026-09 定稿）**：launcher（walker elephant menus 的 `p`/`t`/`a`/`s` 模式及后续 `p` 保留方案）整体废弃；交互两层——
1. **Alt+Tab / windowsmru**：统一切换，窗口过多时系统级过滤；
2. **web 控制台**：搜索过滤、全部管理操作、tag 森林树形展示（tag 多选批量、评分轴、排序也在此）。
CLI 保留为脚本化事实源，不承担交互。

## 八、实现设计（Rust 全栈，2026-09 落地；本节记当前形态）

选型：**B（每 isolated 实例一实例/工作区）并发版 + Rust + 命令名 `mudra`**。

### 交互分层（设计主线）
交互分两层 + 一个脚本化事实源（2026-09 收敛，launcher 层已废弃）：
1. **接口 / CLI（核心操作）**：`mudra` 命令——数据与页面操作的事实源（open / ls / focus / tag / star / col），无 UI 假设，供脚本与其他层回调。
2. **Web 控制台（实际的管理面）**：搜索过滤、全部页面操作、tag 森林树形展示；统一切换则留在 Alt+Tab / windowsmru。
3. **WM（展示相关）**：niri——workspace 布局、列宽、窗口映射；**页面树 → workspace** 移动（分拣）。

### 组件
- **`mudra`**（CLI，Rust）：纯 8899 转发——解析参数 + POST + 打印，零 DB 写、零进程操作。
- **`mudrad`**（daemon，Rust 单二进制）：持各运行实例的 CDP WebSocket，订阅 Target 事件实时写 KV、转发控制命令、拉起/停掉实例；作为 systemd user 服务常驻（`graphical-session.target`，启动脚本补 wayland env 闸门）。
- **每 isolated 实例一个 Chromium 实例**：默认不跑，`mudra open` 才拉起；**可并发多个**（隔离维度各一个）；spawn 带 `PDEATHSIG`，实例死则窗口随之。
- **存储**：okm `VirtualStorage`（fjall 引擎）KV，`~/.local/share/mudra/store`（developMode 下 `~/.local/share/mudra` 是仓库 symlink，wrapper 解析仓内 `target/{release,debug}` 产物）。面板 = leptos wasm（trunk 构建），数据读 8899 的 shaped reads（`/forest`、`/ctx_pages`），WS 只是 epoch 失效总线。

### 数据模型（KV collection 布局，规格见 mudra 仓 `docs/SCHEMA.md`）
- **Tag**（ns 1）：邻接表（`parent_id`，根哨兵 −1）；`by_name` 函数索引做裸名寻址。种子树（`POST /tag_seed`，幂等）：`situation`（inbox/work/personal/privacy，四叶 isolated、inbox required）、`importance`/`urgency`/`quality`（各 ☆..☆☆☆☆☆ 五级、`rank` 排星序）、`state` 流水（to-read/reading/distilled/archived）、`topic`。
- **Page**（ns 2）：`url` 由 CDP `infoChanged` **实时更新** → 可地址过滤；`parent_id` = 页面树（子页 = 由它打开的页，CDP `openerId`）；`by_target` / `by_instance` 函数索引，`page_by_target` 语义 = 活 CDP target 查找。
- **page_tag**：junction（两端分别落在 Page ns 2 与 Tag ns 1，自身不占 ns）。
- **Instance**（ns 4）：`profile` = situation 叶名（1 isolated tag ↔ 1 instance）；`proxy` / `extensions` 按实例持久化，`mudra conf` 预建 `running=0` 行。
- **SiteWidth**（ns 5）：域名 → 列宽比例（0~1）。
- **State**（ns 6）：定长槽位——`current_context`（situation，默认 inbox）、sort、各表自增计数器、dev_mode 等。
- **History**（ns 7）：地址栏访问积累（`by_url` 函数索引），`/open`、`/add` 经 `history_bump` 写入，**不 bump epoch**（面板不可见 collection 不占 epoch 纪律）。
- **Event**（ns 8）：只追加事件日志（page_open / page_close / tag_set），`POST /events {cursor,limit}` 回放，checkpoint 归消费方；同样不 bump epoch。
- **迁移策略（原型阶段）**：schema 变更直接删库重建（停 daemon → `rm -rf ~/.local/share/mudra/store` → 起 → `POST /tag_seed`），不做迁移。

### 隔离实例 = workspace（并发）
- 每个 isolated tag 绑定一个 niri **workspace**（如 `web:<tag>`），**可同时开多个实例，一个实例一个 workspace**。
- `mudra open` = 切到该 workspace + 拉起该实例、重建其 pages。
- 实例窗口进对应 workspace：`mudra open` 时 `niri msg action focus-workspace <name>` + 窗口↔实例映射（启动即记录 pid/窗口 id，按映射而非 title 定位）。
- 切/混上下文 = 关掉或移动某实例窗口，另起目标；**不同实例窗口可混合摆在同一 workspace，也可移动 workspace 分开**。

### 窗口↔进程映射 与 批量移动
- 记录每个窗口属于哪个实例：实例拉起时即记录**地址、pid、窗口 id**（CDP target 归属 + pid 对齐到 niri 窗口），不依赖窗口 title 前缀。
- **一条命令把某实例的全部窗口移到别的 workspace**：`mudra move <instance|tag> <workspace>` → 遍历该实例的窗口 → `niri msg action move-window-to-workspace <ws>`。
- **可混合**：不同实例窗口可摆同一 workspace，再移动来分开/分组——**workspace 移动即"分开"手段**。

### 链接地址记录与过滤
- CDP `infoChanged` 实时更新 `pages.url`。
- 可按地址过滤窗口：`mudra ls` / walker 按 url 子串过滤。

### 代理 与 扩展插件列表
- **代理**：每独立实例可设 `proxy`，拉起时 `chromium --proxy-server=<url>`（可加 `--proxy-bypass-list`）。
- **插件列表（可配置）**：实例允许设置预装的扩展清单（如 mudra-keys、Bitwarden）。拉起时 `chromium --load-extension=<dir1>,<dir2>,...` 加载 unpacked 扩展，清单随实例配置插入。默认加载 mudra 仓库内 `frontend/`（自研扩展 + 面板共享库）。**已实测（2026-08-28）**：`--app` + `--load-extension` 能把 unpacked 扩展载入 `--app` 窗口（当时用 SurfingKeys 构建产物验证；2026-09-05 起日常即 mudra-keys 自身）。
- 两个均为标准 Chromium 命令行开关（NixOS 包装脚本透传）。

### 网站列宽记忆（Win+R）
- **niri 列宽 = 输出宽度比例**：`preset-column-widths { default-column-width { proportion 0.5; } }`；`switch-preset-column-width`（Mod+R）循环档位 1/3 1/2 2/3 1（= 0.33/0.5/0.67/1.0）。
- **读取（已核实 2026-08）**：niri `msg -j windows` 顶层无宽度字段，但聚焦窗嵌套 `layout.tile_size[0]` 即列宽；除以 `focused-output.logical.width` 得比例。
- **设置（已核实）**：`set-column-width <N%>`(百分比) 或 `-N`(像素减)；分数形式如 `1/2` **报错**。恢复时把比例 snap 到档位后按整数百分比发射。
- **捕获**：调整列宽时按 Win+R → `mudra` 读聚焦窗列宽比例 + 当前聚焦 url 域名 → snap 到最近档位写 `site_widths`。
- **恢复**：打开该域名 → 聚焦其列 → 按记录比例整数百分比 `set-column-width`。
- 比率方案本就契合 niri 模型（宽度即比例）。

### CLI 命令（当前面，见 mudra 仓 README）
```
mudra open <url> [--ctx LEAF]       打开页面（实例不在则拉起）
mudra add <url> [--ctx LEAF]        往当前实例再加一页
mudra close [query] [--ctx LEAF]    关整实例，或按 url 过滤关单页
mudra ctx [LEAF]                    显示 / 切换当前上下文（situation 叶）
mudra ls [LEAF] [-f FILTER]         各叶页数概览 / 某叶页列表（url·title 过滤）
mudra targets <ctx>                 列活动 page target
mudra focus <query> [--ctx LEAF]    按 url/title 找页并带到前台
mudra goto|back|forward|reload [--ctx]   导航
mudra page move-here|swap|close <url>    页操作
mudra move <ctx> <workspace>        把该上下文的窗口移到 workspace（niri）
mudra conf <leaf> [--proxy] [--ext] 预建/改实例行
mudra col remember|show             列宽记忆
mudra tag <action> [tag_id] [page_id]    tag 森林种子 / 页指派
mudra sort mru|mtime|rating         排序偏好（state 槽）
mudra dev [on|off]                  扩展开发模式（spawn 前清 5 个缓存点）
mudra ui                            打开 web 控制台
```
daemon 不是 CLI 子命令：`mudrad run` 常驻归 systemd user 服务。

### CDP 映射
- 拉起：`chromium --app=<url> --remote-debugging-port=<动态端口> --user-data-dir=<该实例 profile> --no-first-run`（`--load-extension` 载入 mudra-keys）
- 开页 `Target.createTarget` / 关页 `Page.close` → 事件同步
- 导航 `Page.navigate` / `reload` / `getNavigationHistory` / `navigateToHistoryEntry` / 聚焦 `Target.activateTarget`
- **新窗口拦截（级联）**：`--app` 里 `_blank`/`window.open` 默认开成 chrome 窗口；mudrad 维护注入脚本（拦 `window.open` 与 `a[target=_blank]` → Image beacon → 本地 `/open` → 新 `--app`）。**必须在每个新 page target 出现时注入**（`Target.targetCreated` → `Page.addScriptToEvaluateOnNewDocument`），否则 mudrad 自己开出的新 `--app` 窗口没带脚本，里面再点新窗口又退回 chrome 默认（非 `--app`）。
- 按地址精确控页：`Target.getTargets` 列所有 target（url/title/type）→ 按地址/标题搜 → 命中 → `Target.activateTarget` 激活；错误页亦可。

### 事件同步（核心）
`mudrad` 收到 `targetCreated/infoChanged` → upsert page（url/title 实时）；`targetDestroyed` → 置 `closed_at`。实时，无需轮询。

- **epoch 失效信号**：写操作返回"是否有实际变化"决定是否 bump epoch；面板不轮询，只听 WS 上的 epoch 数字帧，之后重拉 shaped reads。推送只作失效信号，数据面保持纯 KV 读。
- **事件日志**：同一批生命周期写点同时落 Event collection（ns 8）——baseline 重放不算页面事件（daemon 重启不是页面打开），只有活事件流才发；Refresh 分支只在 url 实际变化时发 `page_open`（仅 title 变化是元数据，不是事件）；close 的三个出口（`close_target` / `close_page` / `mark_down`）都发 `page_close`（实例尸体没有 destroyed 事件，teardown 必须补发）。

### 实现备注
- **依赖**：okm（git 源——schema 派生 + fjall 引擎）、induction（git 源——扩展帧与宿主半）、tokio / tungstenite（CDP WS）、kdl（config 解析）；面板侧 leptos + web-sys，trunk 构建（`wasm-bindgen-cli` 与 workspace 版本钉死对齐）。
- **不引 aura crate**：与 aura 的共用只到线契约（induction 两个 crate），不引 aura 运行时——可移植性靠"同一份契约"而非依赖。
- CDP 即 JSON 消息：单 reader 按信封 id 路由 + 超时自摘 waiter，手写即可，不需 jsonrpc 封装。
- 剪贴板走 Wayland `wl-copy`：用 CDP 取聚焦窗口 URL → 管道到 `wl-copy`。

## 九、已验证 / 待验证

- ✅ `--app` 模式无 omnibox / 标签条，Niri 下纯页面（截图实测）。
- ✅ CDP 能列出并 attach `--app` 窗口（它就是普通 page target）。
- ✅ niri `move-window-to-workspace` / `move-column-to-workspace` / `set-column-width` 存在；列宽读取（`layout.tile_size[0]`/`logical.width`）与设置（百分比）已核实。
- ✅ walker **支持多字符前缀**（`src/data.rs` `starts_with`）+ `argument_delimiter`（2026-08 核实；launcher 层 2026-09 整体废弃，此条仅作记录）。
- ✅ 自研 mudra-keys 扩展（`--load-extension=frontend/`）在 `--app` 窗口全功能运行（2026-09-05：四模式、hint、滚动、`:set`、状态栏均 CDP 实测）。
- ✅ tag 森林落地（2026-09-05）：situation 切换 / isolated 实例 / 三条评分轴与 state 流水树种子 / 面板指派；History（ns 7）与地址栏补全 2026-10-01 落地。
- ✅ Rust 全栈落地（2026-09-28–29）：store / spawn / CDP / 生命周期 / 控制动词 / 面板 wasm / CLI；`cargo test --workspace` 138 测全过 + 活体冒烟，Python 树同批删除。
- ✅ 扩展面落地（2026-10-01 / 10-10）：事件日志 ns 8 + `/events` 回放；stdio 常驻宿主（hello 携带 schema、epoch→日志窗口扇出、`host:invoke` 落回动词面）；常驻 e2e 测试压住三个面。
- ⏳ CDP 对错误页（`chrome-error`）reload / navigate 未实测（按机制应当可控）。
- ⏳ 同实例多窗口 ≈ 多标签的资源等价逻辑成立，未实测；走 B 并发版，多实例资源为预期代价。
- ✅ 扩展拦截面落地（2026-10-13）：`mudra:before_navigate` 判决（allow/deny/redirect）、调用 id 由宿主分配且与事件日志 id 不相交、宿主侧 200ms 预算、一切失败面放行；闸门挂 `/open` 与 `/nav` 的 `goto`；真机常驻测试压住 deny/allow/redirect 三态。
- ✅ Hook collection 落地（2026-10-13，ns 9）：`name` / `matches`（逗号分隔 URL 子串，空=所有页）/ `code` / `enabled` / `updated_at`，按 name 幂等 upsert（缺省字段保留原值），写不 bump epoch；治理面 `/hook_set` `/hook_list` `/hook_delete` + CLI `mudra hook set|ls|rm`，执行落点=既有注入桥（一钩子一份带守卫源码）。
- ⏳ 精炼管线（§十 P8–P12：正文转 MD + 全文检索、NB 评分、RSS 捕获、LLM 汇总）未开工。

## 十、开放问题

- **CDP error-page 路由**：错误页/内置页 extension 注入不了（`chrome-error`），按键控制只能走 CDP；普通页由 mudra-keys 自治。如何按 target 类型判定切换（以及错误页上还有哪些操作必要）待实测定义。
- **引擎 UI 缺口**：下载 / 打印等原本由浏览器 chrome 提供的功能，需另行接（CDP / 省略）。
- **tag 森林的深化**：树内单选的 app 约束、isolated 多命中（现为未定义行为）、域名词典的规则迭代与 ML 辅助；inbox 分流的 UX 细节。
- **ML 打标**：朴素贝叶斯 / 逻辑回归从手动评分学"该选哪个值"，生成规则供审（非 LLM 黑箱）。
- **钩子的生效边界**：disable / delete 收不回已经注入到活页面的代码——reload 才按当前集合重新武装。是否要一个"重载匹配页"的动词（把当前页按新集合重新武装）留待需求出现。
- **扩展状态归属**：`store` 臂明确不实现（扩展状态归扩展自己，宿主拥有的事件日志只是投递设施）；若将来出现真正需要共享状态的扩展，这条边界要重新审。

## 十一、扩展面（常驻子进程，2026-10 落地）

控制面之外的第二条机器面。**形**：扩展 = 宿主管托的**常驻子进程**（stdio 管道），**不是**网络服务联邦。动机：一次页面事件常有多个消费者（分类、拦广告、改样式），若每个消费者都是一个服务，用户就在维护服务注册表；托管把"多消费者"变成进程生命周期问题。

- **线契约 = Induction**（crates 住 `~/world/induction`；词汇出自 aura BGI 的抽取——来处在 aura ADR-0035 §3，家在这两个 crate；帧形状与 11 个臂名都由库定义，改名前先读库）。mudra 零 aura crate 依赖——契约即依赖，正如 `aura_alloc` 之于 wasm guest。自造帧词汇与 LSP 类比均已撤回：扩展是**插件血统**（能力单向提供、下游自用不用），词汇静态、双方天然知道，协商面是伪需求（LSP 的 capability 协商解决的是两个独立演化对等体的对齐）。
- **声明**：`config.kdl` 的 `[extensions]` 只管 name → 可执行路径（路由元数据）；事件订阅声明在**脚本的 interface_schema**，由 `hello` 帧携带（进程外 binary 宿主无法内省）——不进配置（改订阅不该改配置），也不是协商（静态声明）。投递只按声明的 kind 扇出，无广播、无隐藏分发。
- **投递 = 声明的 `Tier`**：`Hot` + deadline = **拦截语义**（导航前判决，超时是 failure value，mudra 侧映射 fail-open——导航 UX 优先）；`Cold` = **观察语义**（dispatch 返回 Pending，不泊产生方；处理器可经 host 帧后续作答，也可以不答）。一个处理器只有其中一种形状。
- **事件日志是投递基础设施**：事件同时落只追加的 Event collection（ns 8），消费者自持 cursor 回放（checkpoint 归消费方）；扩展重启从自己的 cursor 续读。
- **效应留在 mudra**：扩展是**决策面**（LLM 调用、规则、状态都在它那边），效应经 `host:invoke` 帧落回动词面（映射既有 8899 动词目录）；扩展不碰 CDP、不开第二个浏览器。`store` 臂**不实现**——扩展状态归扩展自己（每份数据一个家；宿主拥有的事件日志是投递设施，不是扩展存储）。
- **扩展之间无直连**：只共享事件日志；要串联就各自读写各自的家（扩展目录 / mudra 存储 / k10r 的 HTTP 面），而不是长出 N 条点对点链路。
- **信任模型不变**：本机 = 信任边界，扩展面与 8899 一样不设身份准入；跨节点认证归联邦（well-known）层，不适用于本机面。
- **可移植性**：同一个 per-language guest 在 mudrad 与 aura effector 两边通吃（k10r 同步是预期的第一个双侧消费者）；k10r 集成只是"一个扩展示例"，mudra 不认识 k10r。
- **已落地的实现形状（拦截面，2026-10-13）**：调用帧 = `mudra:before_navigate`，args = `{url, ctx}`；应答 = `{"verdict":"allow"|"deny"|"redirect","url"?}`；调用 id 由宿主分配，取自与事件日志行 id 不相交的空间；等待有宿主侧预算（200 ms——`Tier`/deadline 在 hello schema 里是预留位，库与 guest 尚未携带，故先是宿主常量）；多扩展按注册序（名字序）取**第一个非 allow**；`deny` 以 `{"blocked":true,"by":扩展名,"url":…}` 回复调用方且不执行动词，`redirect` 改写请求 url 后执行；闸门挂 8899 的两个导航**入口**——`/open` 与 `/nav` 的 `goto`，刻意**不挂** `run_verb`（guest 自己的 `host:invoke` 不能被它正在执行的那个判决拦住）；所有失败面（无声明会话、预算到期、通道断、队列满、应答不可解析）一律放行。
- **钩子面（轻量规则路径，2026-10-13 落地）**：页面侧 JS 作为数据——Hook collection（ns 9：`name` / `matches` / `code` / `enabled` / `updated_at`，按 name 幂等 upsert、缺省字段保留原值、写不 bump epoch），治理走 8899 动词（`/hook_set` `/hook_list` `/hook_delete`）与 CLI（`mudra hook set|ls|rm`）。执行落点=既有 `INJECT_JS` 注入桥：**一钩子一份**带字面子串守卫的源码（语法错只杀自己），code 逐字内联（不用 `eval`，严格站点 CSP），写钩子后对运行中实例的活页做一次 best-effort 重注入；disable/delete 只影响未来文档（活页代码收不回，reload 重新武装）。
- **未落地**：`iterate` 各臂空载（触及即显式错误值拒答，等消费方报名）、`/events` 与 `/hook_*` 尚无面板入口（只走 CLI/curl）。

决策全文见 mudra 仓 `docs/ADR-extension-protocol.md`（双语）；被它取代的 aura+probe 联邦草案见 `docs/ADR-aura-probe-federation.md`（Superseded）。