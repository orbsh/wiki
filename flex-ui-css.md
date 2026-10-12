# Flex UI 的 CSS 方案：变量即协议 + Trait 组合

> **语言：** [English](flex-ui-css-en.md)（主文档）· [中文](flex-ui-css.md)

## 定位

Flex UI 的样式层是一套**零 CSS 框架**的自有方案,从 fluxora 分离独立成仓库
(实现代号 `ydncf`,即 "You Don't Need a CSS Framework")。核心立场:
UI 结构由 Brick 数据流驱动,样式则**不依赖任何 CSS 框架**,用原生现代 CSS
(自定义属性 + 嵌套 + oklch)组织。

它和 Flex UI 的渲染架构同源:封闭的组件集 + 确定性的渲染,样式也是**确定性的、
可组合的,而非每组件一堆全局 class**。样式层面的哲学与组件层面一致——把
"创意决策固定成有限可组合的词汇表",而不是每次重新发明样式。

## 设计历史：从 fluxora 分离

这套 CSS 原先是 fluxora 仓库的一部分,git 历史记录了反复的样式重构
(COLOR/SIZE 派生类、统一颜色变量、删除多条 border/highlight/shadow 覆盖)。
后来把 CSS 独立成单独仓库(ydncf),fluxora 的 UI 引用它,让样式层可单独演进。

## 技术基础：现代 CSS 的三个能力

- **CSS 自定义属性(变量)作设计 token**:颜色、圆角、边框宽度/样式/颜色都在
  `:root` 用变量定义(`--fg`/`--bg`/`--round`/`--accent`/`--primary`/`--border-color`…),
  边框由组件变量组合(`--border: var(--border-width) var(--border-style) var(--border-color)`)。
- **oklch 颜色空间**:所有颜色用 oklch(可感知均匀),且支持**相对颜色语法**
  (`oklch(from var(--shadow-color) l c h / calc(alpha - .2))`)从单一基准色派生透明变体。
- **CSS 嵌套(nested selectors)**:用 `&.x` 组合类、`&>header` 选子元素,无需预处理器。

## 核心机制：Trait 组合

ydncf 的样式组织是**一组正交的 trait 类**,通过变量作为传递媒介,在 HTML 上
组合出具体形态。颜色类与视觉类**正交组合**:

```css
.border { border: var(--border); }

.accent {
  &.txt      { color: var(--accent); }
  &.border   { --border-color: var(--accent); }  /* 覆写变量 */
  &.shadow   { --shadow-color: var(--accent); }
  &.highlight { &:hover { background-color: var(--accent); } }
}
```

**组合原理**:`.accent.border` 不是两条独立规则叠加,而是 `.accent` 内部通过嵌套
把 `--border-color` 覆写成 accent 色,被 `.border` 的 `border: var(--border)`
消费。**变量是 trait 之间的传递媒介**——颜色类改写变量,视觉类读取变量。
于是颜色轴(accent/primary/secondary/warn/error) × 视觉轴(txt/border/shadow/highlight)
构成二维正交,组合任意。

**暗色自适应**:`@media (prefers-color-scheme: dark)` 只切换基础 `--fg`/`--bg`,
其余所有 token 都从这两个派生(相对色、透明派生),因此一套样式自动适配明暗。

## 与 Flex UI 的关系

- **两个层面都是"封闭词汇表 + 确定性"**:Brick 是组件的封闭集,ydncf 是样式的
  trait 封闭集;都刻意不允许无限扩展,靠有限组合覆盖大多数场景。
- **样式从生产者手里拿走**:与 Flex UI "AI/业务只管内容和结构,样式交给框架锁死"
  一致——ydncf 保证同样的结构 class 组合永远渲染成同样的样式,不漂移。
- **无需预处理器**:现代 CSS 原生的嵌套与自定义属性取代了 Sass/Less/PostCSS,
  减少一层构建复杂度。

## 交叉引用

- **[flex-ui.md](flex-ui.md)**：Flex UI 框架整体,本文是其样式层。
- **[projects/fluxora-architecture.md](projects/fluxora-architecture.md)**：CSS 方案的原出生地,「CSS 策略」一节(变量/嵌套/@layer、trait 组合、无预处理器)。
- 实现仓库:`~/world/ydncf`(独立,CSS 单独演进)。