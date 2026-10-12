# Flex UI's CSS Approach: Variables as Protocol + Trait Composition

> **Languages:** [English](flex-ui-css-en.md) (primary) · [中文](flex-ui-css.md)

## Positioning

Flex UI's styling layer is its own **zero-CSS-framework** approach, split off from fluxora into its own repository
(implementation codename `ydncf`, i.e. "You Don't Need a CSS Framework"). Core position:
UI structure is driven by the Brick data flow, while styling **depends on no CSS framework**, organized with native modern CSS
(custom properties + nesting + oklch).

It shares the same origin as Flex UI's rendering architecture: a closed component set + deterministic rendering, and styling is likewise **deterministic
and composable, rather than a pile of global classes per component**. The philosophy at the styling level is consistent with that at the component level — fixing
"creative decisions into a finite, composable vocabulary", rather than reinventing styling every time.

## Design History: Splitting Off from fluxora

This CSS was originally part of the fluxora repository, and the git history records repeated styling refactors
(COLOR/SIZE derived classes, unified color variables, deletion of several border/highlight/shadow overrides).
Later the CSS was split into its own standalone repository (ydncf), which fluxora's UI references, letting the styling layer evolve independently.

## Technical Foundation: Three Capabilities of Modern CSS

- **CSS custom properties (variables) as design tokens**: colors, radii, border width/style/color are all defined as
  variables in `:root` (`--fg`/`--bg`/`--round`/`--accent`/`--primary`/`--border-color`…),
  and borders are composed from component variables (`--border: var(--border-width) var(--border-style) var(--border-color)`).
- **oklch color space**: all colors use oklch (perceptually uniform), and it supports **relative color syntax**
  (`oklch(from var(--shadow-color) l c h / calc(alpha - .2))`) to derive transparent variants from a single base color.
- **CSS nesting (nested selectors)**: use `&.x` to combine classes and `&>header` to select child elements, with no preprocessor needed.

## Core Mechanism: Trait Composition

ydncf's styling organization is **a set of orthogonal trait classes**, using variables as the transmission medium to
compose concrete forms in HTML. Color classes and visual classes **compose orthogonally**:

```css
.border { border: var(--border); }

.accent {
  &.txt      { color: var(--accent); }
  &.border   { --border-color: var(--accent); }  /* override the variable */
  &.shadow   { --shadow-color: var(--accent); }
  &.highlight { &:hover { background-color: var(--accent); } }
}
```

**Composition principle**: `.accent.border` is not two independent rules stacked together; instead, inside `.accent`, nesting
overrides `--border-color` to the accent color, which is then consumed by `.border`'s `border: var(--border)`.
**Variables are the transmission medium between traits** — color classes rewrite variables, visual classes read variables.
Thus the color axis (accent/primary/secondary/warn/error) × the visual axis (txt/border/shadow/highlight)
form a two-dimensional orthogonal space, freely composable.

**Dark-mode adaptation**: `@media (prefers-color-scheme: dark)` switches only the base `--fg`/`--bg`;
all remaining tokens are derived from these two (relative colors, transparency derivation), so a single set of styles adapts automatically to light and dark.

## Relationship to Flex UI

- **Both levels are "a closed vocabulary + determinism"**: Brick is the closed set of components, ydncf is the closed set of styling
  traits; both deliberately disallow unbounded extension, relying on finite composition to cover most scenarios.
- **Styling is taken out of the producer's hands**: consistent with Flex UI's "AI/business only handles content and structure; styling is locked down by the framework"
  — ydncf guarantees that the same structural class combination always renders to the same styling, without drift.
- **No preprocessor needed**: modern CSS's native nesting and custom properties replace Sass/Less/PostCSS,
  reducing a layer of build complexity.

## Cross-References

- **[flex-ui-en.md](flex-ui-en.md)**: Flex UI the framework as a whole; this document is its styling layer.
- **[projects/fluxora-architecture-en.md](projects/fluxora-architecture-en.md)**: the original birthplace of the CSS approach, in the "CSS Strategy" section (variables/nesting/@layer, trait composition, no preprocessor).
- Implementation repository: `~/world/ydncf` (standalone; the CSS evolves separately).