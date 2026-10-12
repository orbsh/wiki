# 键盘键位重映射设计

> **语言：** [English](keyboard-keymap-design-en.md)（主文档）· [中文](keyboard-keymap-design.md)
> 与 [Vim 移动逻辑批判](vim-movement-critique.md)（移动键/范式批判）互补：本文聚焦**修饰键**的重映射实现，记录在 evdev 层用 kanata 做键位重映射的设计决策与完整配置。决策档案见 NixOS `docs/adr/019-keyboard-keymap-design.md`（evdev/XKB 取舍、shift/括号还原）与 `docs/adr/021-keyboard-ctrlcaps-swap.md`（caps=Ctrl 纯交换，取代 019 的 esc 双模）。

---

## 一、定位：evdev 层 vs XKB keysym 层

键盘重映射有两个层次，解决的问题不同：

| 层次 | 作用对象 | 生效范围 | 局限 |
|------|---------|---------|------|
| **XKB** | keysym（逻辑键） | 走 keysym 的常规应用 | **Blender 等以 keycode（物理键码）为输入源的软件不认**——这正是最初发现 caps↔ctrl 不生效的根因 |
| **kanata（evdev）** | keycode（内核输入层） | 先于 TTY、X11、Wayland、Blender，对所有应用一视同仁，TTS 控制台也生效 | 不提供 XKB 的 **per-window 布局记忆 / 死键 / Compose 等有状态的输入组合逻辑**（kanata 自带 deflayer 多层切换，但那是重映射动作层，与 XKB 的布局层是两回事） |

选择 kanata 的理由：问题出在"某些应用不认 keysym 映射"，只有下沉到 evdev 层才能对所有应用（含 Blender）一致生效。

---

## 二、核心决策

只保留两条映射，其余键全部还原为普通键：

| 物理键 | 行为 |
|--------|------|
| CapsLock | Ctrl（**纯单键交换**，无 tap-hold、无 esc 输出） |
| 物理左 Ctrl | 点击输出 caps；按住切到鼠标层（应急鼠标控制） |

shift / alt / 右 Ctrl **全部还原为普通键，不做任何双模映射**。

**演进**：ADR-019 曾把 CapsLock 设计成 tap-hold 双模键（点击=esc、按住=Ctrl）。实践后发现 esc 双模在终端场景是负资产（见三·4），ADR-021 收敛为最简的一次性交换。

---

## 三、关键取舍

### 1. 为什么用 kanata（evdev 层）而非 XKB

XKB 的 `ctrl:swapcaps` 作用在 keysym 层，只对走 keysym 的常规应用生效。**Blender 等以 keycode（物理键码）为输入源的软件不认 XKB 映射**——这正是最初发现问题（Blender 中 caps↔ctrl 不生效）的根因。kanata 作用在 evdev（内核输入层，keycode），先于所有应用层，对所有软件一致生效。

### 2. 为什么 shift 完全还原，不做双模

最初尝试过把括号绑定到 shift 双模（点击 shift 输出 `(`, 按住保持 shift），但**占满 shift 会连锁破坏依赖 shift 原始语义的功能**：

- RIME 输入法单击 shift 切换中英文
- idea 双击 shift 搜索
- 大写输入

这些功能靠"shift 键本身被点击"的键事件触发。一旦 shift 的双模把 tap 重定向成输出字符，应用层就永远收不到 shift 被点击的事件。**牺牲点按副作用最小的键，会破坏所有依赖该键原生语义的应用。**

### 3. 为什么括号不设快捷方式

尝试过把括号挪到 Alt 双模（左 Alt 点击 `(`, 右 Alt 点击 `)`），但：

- 右 Alt 位置难按
- Alt 键位容易受空格键长度影响
- `()` 输入本身已习惯，用物理 `shift+9` / `shift+0` 即可

**程序员对符号区（shift+数字）的熟练度和主键区区别不大**，不需要额外快捷键。占满 Alt 还会破坏 Alt+Tab 等组合键，得不偿失。

### 4. 为什么放弃 caps 的 esc 双模（ADR-021）

caps 不再承载 tap-hold 双模，改回纯 Ctrl。理由：

- **习惯成本低**：无名指按物理 Esc 不需移动手掌，手指伸缩本身就是可感知的模式信号，误触成本比平替键更低
- **有现成的确定性替代**：vim/终端系的 esc 语义可用 `Ctrl+[`（0x1b，与 esc 字节完全相同）一键达成；要的是「快」，不是「近」
- **双模是负资产**：一个键位承载两种语义，任何边界时序都产生歧义，而 esc 的热点场景（输入法、vim、zellij 模式退出）恰好在边界时序上

**技术根因（终端字节兼容）** — 终端 raw mode 是单字节通道，Esc 与 Ctrl 分属两种编码：

| 按键 | 字节 |
|------|------|
| `Ctrl+字母` | `0x01`–`0x1a`（Ctrl+p=`0x10`）|
| `Esc` | `0x1b` 单字节 |
| `Alt+字母` / `Esc+字母` | `0x1b` + 字母字节（两字节）|

- `Ctrl` 键必须**物理处于按下状态**才能在键入字母时合成出 `0x01-0x1a`；而 tap-hold 的「Ctrl 是否成立由判定窗口决定」（`tap-hold-require-prior-idle 150` 在打字流中优先判 tap），把「快敲 Ctrl」推入灰色地带。终端快捷键最常见的节奏恰是「打字过程中立刻 Ctrl+字母」，首字节从 Ctrl 变成 `0x1b`，Ctrl 语义丢失。这是 tap-hold 的物理边界，QMK/keyd 也无法消除。
- 一旦 tap 误发 `0x1b`，紧接着的字母会被终端在 **escape timeout** 内合成为 `Alt+字母` 序列。对依赖 Escape 序列解码的程序（zellij）后果放大——本仓库 zellij 绑定几乎全是 `Ctrl+Alt+`/`Alt+Shift+` 形式（`0x1b` 前缀序列），`0x1b` 前缀是时序敏感解码，一旦与相邻字符合流，整段按键序列错位。

**结论**：修饰键（Ctrl）必须是一个物理上「无歧义按下」的键。tap-hold 提供的 150–200ms 判定窗口在终端场景就是「Ctrl 是否成立」的不确定期，任何依赖即时组合的快捷键（zellij、nvim、shell）都会踩中。纯交换才是与终端字节协议相容的解。

### 5. 为什么保留物理左 Ctrl 的鼠标层

物理左 Ctrl 承担「点按 caps、按住鼠标层」的双模。这个 tap-hold **只碰非终端用途的左手组合**（caps 点按、鼠标层应急），不构成终端的 Ctrl 修饰键（Ctrl 已由 CapsLock 单键承担），因此不触发上文所述的 Esc/Ctrl 字节歧义——这与 caps 上 esc 双模有本质区别。顺带这让左 Ctrl 键位仍有「应急鼠标控制」的附加值。

---

## 四、配置方法

### 1. kanata.nix（evdev 层重映射）

```nix
services.kanata = {
  enable = true;

  keyboards.default = {
    # devices = [] 自动检测所有键盘设备
    # defcfg 的其余选项走 extraDefCfg（模块自动生成 defcfg 头部，含 linux-dev）
    #
    # 保留 tap-hold-require-prior-idle 与 process-unmapped-keys：纯交换本身虽
    # 不需要判定窗口，但物理左 Ctrl 的 caps/鼠标层 tap-hold（lctl-mod）仍依赖，
    # 保留亦为日后添加 tap-hold 键留余地（ADR-021 决定保留）。
    extraDefCfg = ''
      process-unmapped-keys yes
      tap-hold-require-prior-idle 150
    '';

    config = ''
      (defalias
        ;; 物理左 Ctrl：点击 caps，按住切到鼠标层（应急鼠标控制）
        lctl-mod (tap-hold-press 200 200 caps (layer-while-held mouse))
        ;; 鼠标动作：匀加速（10ms 间隔，3000ms 内从 2px 线性爬到 50px）
        ;; 短点按=精确微调，长按=快速扫过，无需额外按键
        mup    (movemouse-accel-up 10 3000 2 50)
        mdown  (movemouse-accel-down 10 3000 2 50)
        mleft  (movemouse-accel-left 10 3000 2 50)
        mright (movemouse-accel-right 10 3000 2 50)
        mlbtn  mltp
        mrbtn  mrtp
        mmbtn  mmtp
        mwhu   (mwheel-up 50 120)
        mwhd   (mwheel-down 50 120))

      (defsrc
        caps lctl i j k l u o m p ;)

      ;; base：物理 CapsLock=Ctrl（纯单键）；物理左 Ctrl 点击 caps，按住鼠标层；其余还原普通键
      (deflayer base
        lctl @lctl-mod i j k l u o m p ;)

      ;; mouse 层：ijkl 品字形移动鼠标（匀加速），u/o/m 三键，p/; 滚轮
      (deflayer mouse
        _ _ @mup @mleft @mdown @mright @mlbtn @mrbtn @mmbtn @mwhu @mwhd)
    '';
  };
};
```

### 2. sys.nix（移除 XKB swapcaps，避免双层冲突）

```nix
  # 注意：caps/ctrl 交换已由 kanata（evdev 层）接管，见 ./kanata.nix。
  # 这里不设 options（避免与 kanata 双重交换）。
  services.xserver.xkb = {
    # 不设置 options = "ctrl:swapcaps"
  };
```

**关键**：必须从 `sys.nix` 移除 XKB 的 `options = "ctrl:swapcaps"`，否则与 kanata 双层交换冲突。

---

## 五、后果

- 物理 CapsLock = Ctrl，物理左 Ctrl = 点击 caps / 按住鼠标层，无其它行为
- esc 输出由**物理 Esc 键**与组合键 `Ctrl+[` 承担（不再有 esc 双模）
- 物理左 Ctrl 的应急鼠标层（ijkl 移动、u/o/m 三键、p/; 滚轮）保留（tap caps / hold 鼠标层）
- 纯交换对任何应用都是「两张键贴纸互换」——无 tap 误触、无 layer 残留状态、无 TTY/崩溃后键位残留问题
- ADR-019 的 evdev vs XKB 取舍、shift/括号还原决策**仍然有效**，仅「caps 双模」部分被 ADR-021 取代
- capt 层全局启用（所有 host），**SSH 输入不受影响**（evdev 层，不碰网络输入）

---

## 六、与 QMK 的对应

用户是 QMK 用户，kanata 的 tap-hold 概念与 QMK 一一对应。caps 已回归纯键（无 tap-hold），当前仅物理左 Ctrl 的鼠标层保留 tap-hold：

| kanata | QMK |
|--------|-----|
| `tap-hold-press` | `HOLD_ON_OTHER_KEY_PRESS` |
| `tap-hold-require-prior-idle 150` | `require-prior-idle` |
| 鼠标层 tap-hold | `LT` / tap-hold |

物理键盘层（QMK 固件）与系统层（kanata）可协同：QMK 处理硬件层，kanata 处理系统层，二者互不冲突。