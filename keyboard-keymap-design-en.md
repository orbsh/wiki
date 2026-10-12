# Keyboard Keymap Remapping Design

> **Languages:** [English](keyboard-keymap-design-en.md) (primary) · [中文](keyboard-keymap-design.md)
> Complementary to [Vim Movement Logic Critique](vim-movement-critique-en.md) (movement keys/paradigm critique): this article focuses on the remapping implementation of **modifier keys**, recording the design decisions and the complete configuration for key remapping with kanata at the evdev layer. See the NixOS decision records at `docs/adr/019-keyboard-keymap-design.md` (evdev/XKB trade-offs, shift/parenthesis restoration) and `docs/adr/021-keyboard-ctrlcaps-swap.md` (caps=Ctrl pure swap, replacing 019's esc dual-mode).

---

## I. Positioning: evdev layer vs XKB keysym layer

Keyboard remapping has two layers, and they solve different problems:

| Layer | Target | Scope of effect | Limitation |
|------|---------|---------|------|
| **XKB** | keysym (logical key) | Regular applications that go through keysym | **Software such as Blender that uses keycode (physical key code) as its input source does not recognize it** — this is precisely the root cause of the initial discovery that caps↔ctrl did not take effect |
| **kanata (evdev)** | keycode (kernel input layer) | Before TTY, X11, Wayland and Blender; treats all applications alike, and the TTS console is also affected | Does not provide XKB's **per-window layout memory / dead keys / Compose and other stateful input composition logic** (kanata has its own deflayer multi-layer switching, but that is a remapping action layer, a different thing from XKB's layout layer) |

Reason for choosing kanata: the problem is that "some applications do not recognize keysym mappings"; only by descending to the evdev layer can it take effect uniformly for all applications (including Blender).

---

## II. Core Decisions

Only two mappings are kept; all other keys are restored to plain keys:

| Physical key | Behavior |
|--------|------|
| CapsLock | Ctrl (**pure single-key swap**, no tap-hold, no esc output) |
| Physical left Ctrl | Tap outputs caps; hold switches to the mouse layer (emergency mouse control) |

shift / alt / right Ctrl **are all restored to plain keys, with no dual-mode mapping of any kind**.

**Evolution**: ADR-019 once designed CapsLock as a tap-hold dual-mode key (tap=esc, hold=Ctrl). In practice, the esc dual-mode turned out to be a liability in terminal scenarios (see III.4), and ADR-021 converged on the simplest one-time swap.

---

## III. Key Trade-offs

### 1. Why use kanata (evdev layer) rather than XKB

XKB's `ctrl:swapcaps` acts on the keysym layer and only takes effect for regular applications that go through keysym. **Software such as Blender that uses keycode (physical key code) as its input source does not recognize XKB mappings** — this is precisely the root cause of the initial discovery of the problem (caps↔ctrl not taking effect in Blender). kanata acts on evdev (the kernel input layer, keycode), before all application layers, and takes effect uniformly for all software.

### 2. Why shift is fully restored, with no dual-mode

Initially, binding parentheses to a shift dual-mode was attempted (tap shift outputs `(`, hold keeps shift), but **occupying shift fully cascades into breaking functions that depend on shift's original semantics**:

- RIME input method: tapping shift toggles between Chinese and English
- idea: double-tapping shift to search
- Uppercase input

These functions are triggered by the key event of "the shift key itself being tapped". Once the shift dual-mode redirects tap into outputting a character, the application layer will never again receive the event of shift being tapped. **Sacrificing the key whose tap has the smallest side effects will break all applications that depend on that key's native semantics.**

### 3. Why parentheses have no shortcut

Moving parentheses to an Alt dual-mode was attempted (left Alt tap `(`, right Alt tap `)`), but:

- the right Alt position is hard to press
- the Alt key position is easily affected by the spacebar length
- `()` input is already a habit; the physical `shift+9` / `shift+0` suffices

**For programmers, proficiency with the symbol row (shift+number) is little different from the main key area**, so no extra shortcut is needed. Occupying Alt fully would also break key combinations such as Alt+Tab, which is not worth the cost.

### 4. Why the esc dual-mode on caps was abandoned (ADR-021)

caps no longer carries a tap-hold dual-mode and reverts to pure Ctrl. Reasons:

- **Low habit cost**: pressing the physical Esc with the ring finger requires no palm movement; the finger extension/contraction is itself a perceptible mode signal, and the mis-press cost is lower than for a substitute key
- **A ready-made deterministic alternative exists**: the esc semantics of the vim/terminal lineage can be achieved in one keystroke with `Ctrl+[` (0x1b, byte-identical to esc); what is wanted is "fast", not "close"
- **Dual-mode is a liability**: one key position carrying two semantics produces ambiguity at any boundary timing, and esc's hot scenarios (input method, vim, zellij mode exit) fall exactly on boundary timings

**Technical root cause (terminal byte compatibility)** — terminal raw mode is a single-byte channel, and Esc and Ctrl belong to two different encodings:

| Key | Byte |
|------|------|
| `Ctrl+letter` | `0x01`–`0x1a` (Ctrl+p=`0x10`)|
| `Esc` | `0x1b` single byte |
| `Alt+letter` / `Esc+letter` | `0x1b` + letter byte (two bytes)|

- The `Ctrl` key must be **physically in the pressed state** to synthesize `0x01-0x1a` when a letter is typed; whereas with tap-hold, "whether Ctrl holds is decided by the decision window" (`tap-hold-require-prior-idle 150` favors tap in the typing stream), pushing "a quick Ctrl tap" into a gray zone. The most common rhythm of terminal shortcuts is precisely "immediately Ctrl+letter while typing", where the first byte changes from Ctrl to `0x1b` and the Ctrl semantics are lost. This is the physical boundary of tap-hold, which QMK/keyd cannot eliminate either.
- Once tap mistakenly emits `0x1b`, the immediately following letter will be composed by the terminal within the **escape timeout** into an `Alt+letter` sequence. For programs that rely on Escape sequence decoding (zellij) the consequences are amplified — almost all zellij bindings in this repository are of the `Ctrl+Alt+`/`Alt+Shift+` form (a `0x1b`-prefixed sequence), and a `0x1b` prefix is a timing-sensitive decode; once it merges with an adjacent character, the entire key sequence is misaligned.

**Conclusion**: a modifier key (Ctrl) must be a key that is physically "unambiguously pressed". The 150–200ms decision window that tap-hold provides is, in terminal scenarios, an uncertainty period for "whether Ctrl holds", which any shortcut relying on immediate combination (zellij, nvim, shell) will stumble into. A pure swap is the only solution compatible with the terminal byte protocol.

### 5. Why the mouse layer on physical left Ctrl is kept

Physical left Ctrl carries the dual-mode of "tap caps, hold mouse layer". This tap-hold **only touches left-hand combinations not used in terminals** (caps tap, mouse layer emergency), and does not constitute the terminal's Ctrl modifier (Ctrl is already carried by the CapsLock single key), so it does not trigger the Esc/Ctrl byte ambiguity described above — this is fundamentally different from the esc dual-mode on caps. Incidentally, this keeps the added value of "emergency mouse control" on the left Ctrl key position.

---

## IV. Configuration Method

### 1. kanata.nix (evdev layer remapping)

```nix
services.kanata = {
  enable = true;

  keyboards.default = {
    # devices = [] auto-detects all keyboard devices
    # The remaining defcfg options go via extraDefCfg (the module auto-generates
    # the defcfg header, including linux-dev)
    #
    # Retain tap-hold-require-prior-idle and process-unmapped-keys: although the
    # pure swap itself does not need a decision window, the caps/mouse-layer
    # tap-hold of physical left Ctrl (lctl-mod) still relies on them, and keeping
    # them also leaves room for adding tap-hold keys in the future
    # (ADR-021 decided to keep them).
    extraDefCfg = ''
      process-unmapped-keys yes
      tap-hold-require-prior-idle 150
    '';

    config = ''
      (defalias
        ;; Physical left Ctrl: tap caps, hold switches to the mouse layer (emergency mouse control)
        lctl-mod (tap-hold-press 200 200 caps (layer-while-held mouse))
        ;; Mouse actions: uniform acceleration (10ms interval, linearly ramping from 2px to 50px within 3000ms)
        ;; Short tap = precise fine-tuning, long press = fast sweep, no extra keys needed
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

      ;; base: physical CapsLock=Ctrl (pure single key); physical left Ctrl tap caps, hold mouse layer; the rest restored to plain keys
      (deflayer base
        lctl @lctl-mod i j k l u o m p ;)

      ;; mouse layer: ijkl in a pyramid arrangement move the mouse (uniform acceleration), u/o/m three buttons, p/; scroll wheel
      (deflayer mouse
        _ _ @mup @mleft @mdown @mright @mlbtn @mrbtn @mmbtn @mwhu @mwhd)
    '';
  };
};
```

### 2. sys.nix (remove XKB swapcaps to avoid a two-layer conflict)

```nix
  # Note: the caps/ctrl swap is already handled by kanata (evdev layer), see ./kanata.nix.
  # No options are set here (to avoid a double swap with kanata).
  services.xserver.xkb = {
    # Do not set options = "ctrl:swapcaps"
  };
```

**Key point**: the XKB `options = "ctrl:swapcaps"` must be removed from `sys.nix`, otherwise it conflicts with kanata's two-layer swap.

---

## V. Consequences

- Physical CapsLock = Ctrl, physical left Ctrl = tap caps / hold mouse layer, no other behavior
- esc output is carried by the **physical Esc key** and the key combination `Ctrl+[` (no more esc dual-mode)
- The emergency mouse layer of physical left Ctrl (ijkl movement, u/o/m three buttons, p/; scroll wheel) is kept (tap caps / hold mouse layer)
- A pure swap is, for any application, "swapping two key stickers" — no tap mis-trigger, no residual layer state, no leftover key-position problems after a TTY/crash
- ADR-019's evdev vs XKB trade-off and the shift/parenthesis restoration decisions **remain in effect**; only the "caps dual-mode" part was superseded by ADR-021
- The capt layer is globally enabled (all hosts), and **SSH input is unaffected** (evdev layer, does not touch network input)

---

## VI. Correspondence with QMK

The user is a QMK user, and kanata's tap-hold concepts correspond one-to-one with QMK. caps has returned to a pure key (no tap-hold); currently only the mouse layer of physical left Ctrl retains tap-hold:

| kanata | QMK |
|--------|-----|
| `tap-hold-press` | `HOLD_ON_OTHER_KEY_PRESS` |
| `tap-hold-require-prior-idle 150` | `require-prior-idle` |
| mouse layer tap-hold | `LT` / tap-hold |

The physical keyboard layer (QMK firmware) and the system layer (kanata) can cooperate: QMK handles the hardware layer, kanata handles the system layer, and the two do not conflict.
