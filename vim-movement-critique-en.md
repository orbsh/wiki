# Vim Movement Logic: A Critique and the Modern Input Paradigm

> **Languages:** [English](vim-movement-critique-en.md) (primary) · [中文](vim-movement-critique.md)
> A demystification of the traditional Vim movement dogma along three dimensions — historical baggage, biomechanics, and the modern code context — and the new paradigm formed by Colemak + Xiaohe double-pinyin + modern Neovim plugins.

---

## 1. hjkl's Historical Genetic Defect

hjkl is not an ergonomic design; it exists purely because the ADM-3A terminal keyboard Bill Joy used in 1976 had these four arrows printed on those keys. It is a **historical compromise**, not an optimal solution.

```
ADM-3A keyboard layout (1976)
┌───┬───┬───┬───┬───┬───┬───┬───┬───┬───┬───┬───┬───┐
│ ` │ 1 │ 2 │ 3 │ 4 │ 5 │ 6 │ 7 │ 8 │ 9 │ 0 │ - │ = │
├───┴─┬─┴─┬─┴─┬─┴─┬─┴─┬─┴─┬─┴─┬─┴─┬─┴─┬─┴─┬─┴─┬─┴─┬─┴───┤
│ Tab │ Q │ W │ E │ R │ T │ Y │ U │ I │ O │ P │ [ │ ] │
├─────┴┬──┴┬──┴┬──┴┬──┴┬──┴┬──┴┬──┴┬──┴┬──┴┬──┴┬──┴─────┤
│ Ctrl │ A │ S │ D │ F │ G │ H │ J │ K │ L │ ; │ ' │    │
├──────┴─┬─┴─┬─┴─┬─┴─┬─┴─┬─┴─┬─┴─┬─┴─┬─┴─┬─┴─┬─┴──────┤
│  Ctrl  │ Z │ X │ C │ V │ B │ N │ M │ , │ . │ / │ Ctrl │
└────────┴───┴───┴───┴───┴───┴───┴───┴───┴───┴───┴──────┘
                                 ↑↑↑↑
                   Arrows printed directly on HJKL
```

Back then terminals had no dedicated arrow keys, and the arrow markings were printed directly on the HJKL keycaps. Bill Joy wrote it into Vi, and it has been part of Unix culture ever since. **The keyboard layout determined the editing paradigm, rather than the editing paradigm optimizing the keyboard layout** — the causality is inverted.

---

## 2. The Brutal Truth from a Biomechanical Perspective

### 2.1 The Outward-Splayed Hand Effect

In the standard touch-typing posture the two hands naturally splay outward (pinky outward), with the result that:
- **The pinky is relatively longer**: the outward splay stretches the pinky's reach
- **The index finger is relatively shorter and at a disadvantage**: its range of motion in the inward direction is compressed

In hjkl, h and j are handled by the **left index finger** — precisely the weakest finger in the outward-splayed posture.

### 2.2 Joint Stress from Horizontal Pressing

| Finger | Pressing direction | Joint load |
|------|---------|---------|
| Middle/ring finger | Can press vertically downward (natural force) | Low |
| Index finger (h, j) | Must pull sideways inward, an almost purely horizontal shear force | **High** |

The h and j handled by the index finger require lateral inward presses that run against the physiological structure. At high frequency this very easily causes RSI (repetitive strain injury).

### 2.3 The Hidden Cost of the b Key

The traditional dogma recommends using `b` in place of pressing `h` repeatedly (back one word). But anatomically:

- `b` requires the **left index finger to stretch from f all the way to b**, an extreme inward stretch
- This is "trading one violent joint twist for a few mild ones" — the cost outweighs the benefit
- `<C-b>` (page back) is double torture: holding Ctrl down + pulling the index finger sideways

### 2.4 Why Vim Does Not Acknowledge These Problems

The Vim community has an extremely strong "dogmatic inertia". People who question hjkl are usually classified as "not knowing how to use Vim" or "not being patient enough". But biomechanical facts do not change because of a community consensus.

---

## 3. How w/b Fails in Modern Programming

Vim's word movement (`w`/`b`/`e`) is based on simple lexical rules — letters, digits and punctuation form word boundaries. Those rules worked well for 1970s C and English prose, but in modern code they collapse completely.

### 3.1 Shattered Word Boundaries

```
Example of modern code:
<div className="flex items-center">

The jump path of w/b:
<div → className= → "flex → items-center" → ">
     ↑              ↑         ↑
 boundary1      boundary2 boundary3
```

Every `w` lands somewhere unpredictable. The brain has to compute in real time where the next stop will be, which is a very high cognitive load.

### 3.2 The Nightmare of Mixed Delimiters

```
JSON:    {"key": "value", "nested": [1, 2, 3]}
XML:     <tag attr="val" />
CSS:     margin: 10px 20px 30px 40px;
```

Commas, colons, semicolons, quotation marks, angle brackets — each symbol has a different meaning for `w`. Forcing w/b into use brings an extremely high cognitive load, and memorizing a separate command set for every symbol is even less realistic.

### 3.3 Conclusion

w/b is **a lexical tool designed for English prose**, not a structural navigation tool designed for code. In modern programming its cognitive cost far outweighs its benefit.

---

## 4. The Real Modern Vim Movement Strategy

Abandon rigid lexical rules and adopt a **hybrid strategy**:

### 4.1 Medium and Long Distances: Visual Label Jumping

Use plugins such as flash.nvim or leap.nvim to achieve "two keystrokes to anywhere the eye can reach":

```lua
-- flash.nvim: anywhere the eye can reach, two keystrokes to get there
require("flash").jump()  -- labels every visible character; hit the label key to jump straight there
```

This kind of jumping, grounded in **visual intuition**, completely replaces w/b at medium and long distances:
- It ignores the ambiguities of lexical parsing
- It requires no prediction of the landing point
- Jump distance is decoupled from cognitive cost

### 4.2 Structural Movement: Treesitter Semantic Objects

Through nvim-treesitter-textobjects, jump and select directly by AST syntax node:

```lua
-- move by syntax nodes such as functions, classes and arguments
-- instead of hopping between symbol fragments
```

This avoids fragmented jumping between symbols and operates directly on **semantic boundaries**.

### 4.3 Tiny Ranges: Back to Mindless Fine-Tuning

For a 2-3 character misalignment there is no need to think about semantics. Just use an ergonomic arrow-key mapping:

```lua
-- inverted-T arrow-key mapping (system level or Neovim level)
vim.keymap.set('', '<M-i>', '<Up>')
vim.keymap.set('', '<M-j>', '<Left>')
vim.keymap.set('', '<M-k>', '<Down>')
vim.keymap.set('', '<M-l>', '<Right>')
```

The vertical downward press of the inverted-T layout (IJKL or similar) matches the force direction that modern ergonomics advocates, replacing the anti-human single-row hjkl.

Why the inverted-T is more reasonable (these advantages are layout-independent and apply to any keyboard):

- **Up and down on two separate rows**: space naturally distinguishes up from down, so you do not have to rely on muscle memory to remember "which one is up and which one is down"
- **Up (forward) alone on the upper row**: in games such as FPS/RPG where up is the forward direction, most of the time is spent on up (forward) plus left/right and very little on rapid forward/back switching; putting "up" on the upper row is therefore very natural. And when the hand is placed, having the middle finger on the upper row is also natural — for rapid up/down switching, moving the middle finger up and down is far lighter than moving the index finger for h/j
- **Down (in the middle of the home row)**: downward movement is the highest-frequency direction in coding, so it goes on a strong home-row key (the middle finger)
- **Left and right at the two ends**: matches the habit of physical arrow keys, and left and right are symmetric

### 4.4 Summary of the Hybrid Strategy

| Distance | Strategy | Tool |
|------|------|------|
| Far (>10 characters) | Visual label jumping | flash.nvim / leap.nvim |
| Medium (3-10 characters) | Semantic object movement | treesitter-textobjects |
| Near (1-3 characters) | Mindless arrow-key fine-tuning | inverted-T mapping |

---

## 5. The Colemak Keyboard Layout

If patching inside Vim still cannot relieve finger fatigue, the problem has to be fixed at the system layout layer.

### 5.1 A Statistical Victory

| Metric | QWERTY | Colemak |
|------|--------|---------|
| High-frequency letters | Scattered across the rows | Concentrated on the home row (R S T N E I O A) |
| Home-row usage | ~30% | ~70% |
| Finger travel distance | Baseline | More than halved |
| Design idea | Deliberately splits high-frequency letters apart (to prevent mechanical key jams) | Maximizes home-row utilization |

### 5.2 The Colemak-DH Variant

For matrix keyboards and split keyboards, Colemak-DH moves D and H down one row:
- Fixes excessive middle-finger stretching
- Cures the index-finger lateral-press pain point at the layout level
- Is the ultimate optimized version of Colemak

### 5.3 Why Not Dvorak

Dvorak places the high-frequency letters on the outer edges of two rows; theoretically optimal, but in practice the fingers frequently jump back and forth between the home row and the outer rows. Colemak keeps most of the QWERTY positions (cutting the relearning cost) while concentrating the highest-frequency letters on the home row — **the pragmatic optimum**.

---

## 6. The Native Emacs Alternative: Boon Modal Editing

For Emacs users there is also an option that bypasses the entire Vim system: [Boon](https://github.com/jyp/boon) — a modal editing scheme designed from scratch and built for Emacs rather than for Vim.

### 6.1 A Fundamental Difference in Design Philosophy

| Dimension | Vim | Boon |
|------|-----|------|
| Design starting point | Efficiency compromises of the terminal era | Modern ergonomics |
| Key assignment logic | Mnemonic | Spatial position |
| Division of labor between the hands | No explicit division | Right hand = movement, left hand = operations (like a gamepad) |
| Relationship with the editor | Vim is a standalone editor; Emacs emulates it via Evil | Boon is a native modal layer for Emacs |
| Emacs compatibility | Evil must maintain its own independent keybinding map | Natively compatible with C-x/C-c shortcuts, Dired and Magit |

### 6.2 Space-First Key Assignment

Boon's core principle: **key assignment is based on where the fingers are on the keyboard, not on the letters printed on the keycaps**.

- The most frequent commands go to the strong home-row fingers (index, middle)
- Common combinations alternate between the hands or roll within one hand (finger roll)
- Hard-to-reach keys are left for the user's own low-frequency commands
- Movement keys on the right hand, operation keys on the left — a natural division of labor between the hands

This directly solves Vim's fundamental problem: hjkl's assignment is based on the arrow markings printed on a 1976 terminal keyboard, a **mnemonic logic** ("h = left" because that is where the arrow is); Boon's assignment is based on **the fingers' natural direction of movement**.

### 6.3 Modular Design

Boon does not have to be adopted wholesale. You can use only part of it:
- `boon COMMAND`: command mode (movement + operations)
- `boon SPECIAL`: special keybindings (C-x, C-c prefixes)
- `boon INITIALS`: initials input mode
- boon-motions or boon-operators can also be used on their own

### 6.4 Where It Fits

Boon is especially suited to:
- **Heavy Emacs users**: those who do not want to bring in the full Evil/Vim emulation layer
- **New configurations**: starting from scratch, unconstrained by the historical baggage of Vim keybindings
- **Those who value ergonomics**: space-first assignment plus a split between the hands is more natural than hjkl's positional mapping

Boon is not suited to:
- Users who have already deeply customized an Evil configuration (high migration cost)
- Situations that require unified keybindings across editors (Boon is Emacs-only)

---

## 7. A Match Made in Heaven: Colemak + Xiaohe double-pinyin + Vim

Switching to Colemak is not without cost — it completely displaces the physical positions of Vim's hjkl, and the Wubi radicals fail as well. Yet the arrival of Xiaohe double-pinyin closes the loop perfectly.

### 7.1 The Accidental Synergy Between Xiaohe and Colemak

Wubi radicals depend strictly on the QWERTY layout and are void the moment the layout changes. Xiaohe double-pinyin, by contrast, is based on initial/final logic and is decoupled from physical key positions.

**An unexpected synergy**: the most frequent Chinese finals in Xiaohe double-pinyin (an, ang, eng, ing) all fall, under the Colemak layout, on the home keys of the index and middle fingers of both hands. With Colemak + the standard QWERTY version of Xiaohe double-pinyin mounted at the system level, both Chinese and English input can enjoy touch-typing on the home row.

### 7.2 Vim's Positional Mapping Method: The Inverted-T Layout

hjkl squeezes all four directions onto a single row (h = left, j = down, k = up, l = right); up and down are two adjacent keys and left and right are at the two ends — **there is no spatial logic, only muscle memory**. This stems from the 1976 ADM-3A terminal printing the arrows on the single HJKL row of keycaps — a historical coincidence, not a design.

The modern arrow-key habit is the **inverted-T** (the physical arrow-key layout): up alone on one row, and left, down and right on the next. Mapped onto a group of keys near the Colemak home row:

```vim
" inverted-T arrow keys (up alone on one row, left/right/down on the next)
noremap u k    " up (row above)
noremap n h    " left (home row, left)
noremap e j    " down (home row, middle)
noremap i l    " right (home row, right)
```

```
       u (Up)       
   n      e      i  
 Left   Down   Right
```

The general advantages of the inverted-T layout (up and down on two rows, the middle finger that can compete for up/down beating the index finger, left/right symmetry) are covered in [Section 4.3](#43-tiny-ranges-back-to-mindless-fine-tuning); here only the concrete keys under Colemak are given: `u` lands on the upper row as "up", and `n e i` sit on the home row as "left/down/right".

### 7.3 Unified Cognition Across Devices

| Device | Layout | Input method | Cognitive bridge |
|------|------|--------|---------|
| Computer | Colemak | Xiaohe double-pinyin (QWERTY version) | Initial/final logic |
| Phone | QWERTY (touchscreen constraint) | Xiaohe double-pinyin (built in) | The same set of double-pinyin rules |

Phones cannot use Colemak because of their touchscreen mechanism, but the Xiaohe double-pinyin built into phone input methods serves perfectly as the bridge. The brain's cognitive cost across devices is unified by double-pinyin logic, while the physical differences are automatically isolated by muscle memory.

---

## Conclusion

The evolution of tools is, in essence, people re-adapting to their own physiological limits and to changes in their working environment.

| Traditional dogma | Modern replacement | Essential change |
|---------|---------|---------|
| hjkl arrow keys | flash.nvim visual jumping + inverted-T arrow keys | From historical compromise to ergonomics |
| w/b lexical movement | treesitter semantic objects | From lexical fragments to structural awareness |
| QWERTY | Colemak-DH | From mechanical anti-jam to home-row maximization |
| Wubi (bound to QWERTY) | Xiaohe double-pinyin (layout-independent) | From physical binding to logical binding |

**Core idea**: break free from the hostage-taking of historical dogma; let tools adapt to people, rather than people adapting to tools.

---

> **Further reading**: the remapping implementation for **modifier keys** (CapsLock→esc/Ctrl, shift restored to its original function, no shortcut key for brackets) is in [Keyboard Keymap Design](keyboard-keymap-design-en.md). This document focuses on movement keys and paradigms and does not repeat the modifier-key content.
