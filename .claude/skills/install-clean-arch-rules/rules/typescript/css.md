---
paths:
  - "**/*.css"
  - "**/*.scss"
  - "**/*.module.css"
  - "**/*.module.scss"
  - "src/styles/**"
  - "styles/**"
---
# CSS Clean Architecture

> Framework-agnostic CSS architecture rules for maintainable SPA styling. Covers design tokens, ITCSS layer structure, naming conventions, component scoping, theming, and anti-patterns. See [react.md](react.md) and [angular.md](angular.md) for framework-specific integration.

---

## The Problem

Unstructured CSS in SPAs leads to specificity wars, global leakage, duplicated values, and styles that cannot be safely changed without side effects. A deliberate architecture makes every style rule predictable: you know where it lives, what it affects, and why it has the specificity it has.

---

## Methodology: ITCSS + BEM + Design Tokens

Three complementary tools, each solving a different problem:

| Tool | Solves | Scope |
|---|---|---|
| **ITCSS** | Where styles live and in what order | Macro — file/layer organization |
| **BEM** | What classes mean and how they relate | Micro — naming convention |
| **Design Tokens** | What values mean and where they come from | System — single source of truth |

### ITCSS Layer Hierarchy

Specificity increases from top to bottom. Each layer maps to a native CSS `@layer` — the browser enforces the ordering at runtime, not just by file load order.

| Layer | Native `@layer` | Enforced Rule | Example |
|---|---|---|---|
| **Abstracts** | *(no layer — zero CSS output)* | Variables, mixins, functions only | `_tokens.scss`, `_mixins.scss` |
| **Generic** | `generic` | Universal selectors (`*`) only. No IDs, no classes. | `_normalize.scss`, `_box-sizing.scss` |
| **Base** | `base` | Element selectors only. No classes, no nesting. | `body`, `h1`, `a`, `p` |
| **Objects** | `objects` | Layout properties only (`display`, `grid`, `flex`). No colors, no decorations. | `.o-grid`, `.o-stack` |
| **Components** | `components` | Fully decorated. No external margins. | `.c-button`, `.c-card` |
| **Utilities** | `utilities` | Immutable single-purpose. `!important` permitted. | `.u-text-center`, `.u-sr-only` |

---

## Layer 1: ITCSS — File Structure

Organize stylesheets into layers of increasing specificity. Styles in later layers may override earlier ones; the reverse is always a red flag.

```
styles/
  abstracts/          ← ZERO CSS output — variables, mixins, functions only
    _tokens.scss      ← CSS custom properties (design tokens)
    _mixins.scss      ← reusable SCSS mixins (breakpoints, container queries)
    _functions.scss   ← SCSS functions (e.g. rem(), px-to-em())
    _index.scss       ← barrel: @forward each partial
  generic/            ← Low-specificity global resets (universal selector, normalize)
    _box-sizing.scss  ← *, *::before, *::after { box-sizing: border-box }
    _normalize.scss   ← normalize.css or modern-normalize
    _reset.scss       ← opinionated resets (margin: 0, list-style: none, etc.)
  base/               ← Bare HTML element defaults — NO class selectors
    _typography.scss  ← body, h1–h6, p, a, code
    _elements.scss    ← img, button, input base styles
  objects/            ← Layout patterns, undecorated structural classes
    _layout.scss      ← grid, flex containers
    _container.scss   ← .o-container, .o-container-region (container-type)
    _stack.scss       ← .o-stack (vertical rhythm via gap)
  components/         ← (Angular only) cross-cutting shared component styles
    _button.scss
    _card.scss
  utilities/          ← Single-purpose helpers, highest specificity permitted
    _spacing.scss
    _visibility.scss
    _text.scss
  main.scss           ← Entry point — @use each layer in order
```

**Rules:**
- The `abstracts/` layer produces ZERO lines of CSS — only definitions
- `generic/` comes before `base/` — resets (universal `*`) have lower specificity than element selectors and must load first
- `utilities/` classes are the ONLY place where `!important` is permitted
- Never import the same partial more than once — use `@use` with namespaces
- Declare all `@layer` names at the top of `main.scss` before any rules are loaded — this fixes the order regardless of component load sequence

```scss
// styles/main.scss
// 1. Declare all layers upfront — order here is the final cascade order.
//    A rule in 'utilities' ALWAYS wins over 'components', regardless of selector specificity.
@layer generic, base, objects, components, utilities;

// 2. Abstracts produce no CSS — load outside any @layer
@use 'abstracts/tokens';
@use 'abstracts/mixins';

// 3. Assign each partial to its layer
@layer generic {
  @use 'generic/box-sizing';
  @use 'generic/normalize';
  @use 'generic/reset';
}

@layer base {
  @use 'base/typography';
  @use 'base/elements';
}

@layer objects {
  @use 'objects/container';
  @use 'objects/layout';
  @use 'objects/stack';
}

// Angular only — React components have co-located .module.css files
@layer components {
  @use 'components/button';
  @use 'components/card';
}

@layer utilities {
  @use 'utilities/spacing';
  @use 'utilities/visibility';
  @use 'utilities/text';
}
```

**Why `@layer` matters:** Without it, a single high-specificity element in `base/` (e.g., `input[type="text"]`) can silently override a component class. With `@layer`, the layer order is the absolute tiebreaker — selector specificity only competes within the same layer.

**React:** Each component has a co-located `.module.css` — no component styles in the global `styles/` tree.
**Angular:** Each component has a co-located `.component.scss` — use `@use '../../../styles/abstracts/mixins' as m` to pull in abstracts.

---

## Layer 2: Design Tokens — CSS Custom Properties

Design tokens are the single source of truth for all visual values. Every color, spacing step, radius, shadow, and font size must be a token — never a hardcoded value.

### Two-Tier Token Architecture

```scss
// styles/abstracts/_tokens.scss

// ── Tier 1: Primitives ──────────────────────────────────────────────────────
// Raw values — no UI meaning. These define the palette, not the usage.
:root {
  // Color palette
  --blue-100: #e0f0ff;
  --blue-500: #2563eb;
  --blue-700: #1d4ed8;
  --gray-50:  #f9fafb;
  --gray-200: #e5e7eb;
  --gray-700: #374151;
  --gray-900: #111827;
  --red-500:  #ef4444;
  --green-500: #22c55e;
  --white:    #ffffff;

  // Spacing scale (4px base)
  --space-1: 0.25rem;   // 4px
  --space-2: 0.5rem;    // 8px
  --space-3: 0.75rem;   // 12px
  --space-4: 1rem;      // 16px
  --space-6: 1.5rem;    // 24px
  --space-8: 2rem;      // 32px
  --space-12: 3rem;     // 48px
  --space-16: 4rem;     // 64px

  // Aspect ratios — use with aspect-ratio property
  --ratio-square:  1 / 1;
  --ratio-video:   16 / 9;
  --ratio-wide:    21 / 9;
  --ratio-golden:  1.618 / 1;
  --ratio-portrait: 3 / 4;

  // Layout widths — prevent "magic number" max-widths
  --width-readable:  65ch;    // optimal line length for body text
  --width-prose:     75ch;    // wider reading column
  --width-content:   960px;   // standard content container
  --width-wide:     1280px;   // wide layout container
  --width-full:     100%;

  // Typography
  --font-sans: 'Inter', system-ui, -apple-system, sans-serif;
  --font-mono: 'JetBrains Mono', 'Fira Code', monospace;
  --text-xs:   0.75rem;
  --text-sm:   0.875rem;
  --text-base: 1rem;
  --text-lg:   1.125rem;
  --text-xl:   1.25rem;
  --text-2xl:  1.5rem;
  --text-3xl:  1.875rem;

  // Radii
  --radius-sm: 0.25rem;
  --radius-md: 0.5rem;
  --radius-lg: 0.75rem;
  --radius-full: 9999px;

  // Shadows
  --shadow-sm: 0 1px 2px 0 rgb(0 0 0 / 0.05);
  --shadow-md: 0 4px 6px -1px rgb(0 0 0 / 0.1), 0 2px 4px -2px rgb(0 0 0 / 0.1);
  --shadow-lg: 0 10px 15px -3px rgb(0 0 0 / 0.1), 0 4px 6px -4px rgb(0 0 0 / 0.1);

  // Z-index scale — ALL z-index values in the app must use these tokens
  --z-negative:  -1;      // behind normal flow (e.g. background pseudo-elements)
  --z-base:       0;
  --z-elevated:   1;      // raised cards, floating labels
  --z-dropdown: 1000;     // dropdowns, popovers
  --z-sticky:   1100;     // sticky headers, sidebars
  --z-modal:    1200;     // dialogs, drawers
  --z-tooltip:  1300;     // tooltips (always on top)

  // Motion — Duration
  --duration-instant: 50ms;
  --duration-fast:   150ms;
  --duration-base:   300ms;
  --duration-slow:   500ms;

  // Motion — Easing
  --ease-linear:    linear;
  --ease-in:        cubic-bezier(0.4, 0, 1, 1);
  --ease-out:       cubic-bezier(0, 0, 0.2, 1);
  --ease-in-out:    cubic-bezier(0.4, 0, 0.2, 1);
  --ease-spring:    cubic-bezier(0.34, 1.56, 0.64, 1);  // slight overshoot
}

// ── Tier 2: Semantics ───────────────────────────────────────────────────────
// Map primitives to purpose. These are what components reference.
:root {
  // Surface
  --color-bg-page:      var(--gray-50);
  --color-bg-surface:   var(--white);
  --color-bg-elevated:  var(--white);
  --color-bg-subtle:    var(--gray-200);

  // Text
  --color-text-primary:   var(--gray-900);
  --color-text-secondary: var(--gray-700);
  --color-text-disabled:  var(--gray-200);
  --color-text-inverse:   var(--white);

  // Brand / interactive
  --color-brand:           var(--blue-500);
  --color-brand-hover:     var(--blue-700);
  --color-brand-subtle:    var(--blue-100);

  // Status
  --color-error:   var(--red-500);
  --color-success: var(--green-500);

  // Border
  --color-border:       var(--gray-200);
  --color-border-focus: var(--blue-500);

  // Accessibility — focus ring
  // Used by the global :focus-visible rule in base/_elements.scss
  --color-focus-ring:        var(--blue-500);
  --color-focus-ring-offset: var(--white);

  // Spacing aliases (layout-level semantics)
  --spacing-page-x: var(--space-6);
  --spacing-section: var(--space-12);
  --spacing-card-pad: var(--space-4);
}
```

**Rules:**
- Components MUST reference semantic tokens, never primitives directly
- `--button-bg: var(--color-brand)` — good. `--button-bg: #2563eb` — forbidden
- Adding a new color means adding a primitive AND a semantic entry — not an inline hex
- Semantic token names encode: `--color-{role}-{variant}`, `--space-{usage}`, `--radius-{size}`
- Every `z-index` declaration MUST use a `--z-*` token — `z-index: 9999` is forbidden
- Every `transition` MUST use `--duration-*` and `--ease-*` tokens
- All semantic token color pairings MUST pass WCAG AA contrast (4.5:1 for text, 3:1 for UI components). Use `stylelint-a11y` or `postcss-color-contrast` to automate this check. Document failing pairs as tech debt — never ship them silently.

```css
/* Correct */
.modal { z-index: var(--z-modal); }
.dropdown { z-index: var(--z-dropdown); }
.button { transition: background var(--duration-fast) var(--ease-out); }

/* Forbidden */
.modal { z-index: 1200; }
.button { transition: background 150ms ease-out; }
```

---

## Layer 3: Theming — Light / Dark / Brand

Override semantic tokens under a data attribute or class. Primitives never change — only semantic mappings do.

```scss
// styles/abstracts/_tokens.scss (continued)

[data-theme="dark"] {
  --color-bg-page:      var(--gray-900);
  --color-bg-surface:   #1f2937;       // gray-800
  --color-bg-elevated:  #374151;       // gray-700
  --color-bg-subtle:    #374151;
  --color-text-primary:   var(--white);
  --color-text-secondary: var(--gray-200);
  --color-text-disabled:  var(--gray-700);
  --color-border:         #4b5563;     // gray-600
}
```

```typescript
// Apply theme via data attribute — works with signals and Zustand
document.documentElement.setAttribute('data-theme', 'dark')
```

**Rules:**
- Never toggle themes by loading/unloading stylesheets — use attribute + token remapping
- `prefers-color-scheme` sets the initial default; user override is stored in `data-theme`
- All semantic tokens MUST have a dark-mode mapping if the app supports dark mode

```scss
// Respect system preference as default
@media (prefers-color-scheme: dark) {
  :root:not([data-theme="light"]) {
    --color-bg-page: var(--gray-900);
    // ... remaining dark mappings
  }
}
```

---

## Layer 4: BEM Naming Convention

### Naming Convention by Scope

Two separate naming conventions apply depending on where the styles live. **Never mix them in the same file.**

| Scope | Convention | Reason |
|---|---|---|
| Global CSS (`generic/`, `base/`, `objects/`, `components/`, `utilities/`) | **BEM + BEMIT namespaces** | Human-readable, no build tool dependency, works in plain HTML |
| Scoped CSS (React `.module.css`, Angular `.component.scss`) | **camelCase only** | Enables TypeScript dot-notation (`styles.cardFeatured`), compile-time safety |

BEM governs class naming only for global stylesheets.

```
.block {}                  ← standalone component
.block__element {}         ← part of block (double underscore)
.block--modifier {}        ← variant of block (double hyphen)
.block__element--modifier {} ← variant of an element
```

```scss
// Good
.user-card {}
.user-card__avatar {}
.user-card__name {}
.user-card--featured {}
.user-card__avatar--large {}

// Bad — element chain (forbidden)
.user-card__header__title {}   // never nest elements

// Bad — modifier without block
.--featured {}
```

### BEMIT Namespaces (global stylesheets only)

Use namespaces when writing styles in the global ITCSS layers to communicate intent at a glance:

| Prefix | Layer | Example |
|---|---|---|
| `c-` | Component | `.c-button`, `.c-modal` |
| `o-` | Object (layout) | `.o-grid`, `.o-container` |
| `u-` | Utility | `.u-sr-only`, `.u-mt-4` |
| `is-`, `has-` | State | `.is-loading`, `.has-error` |

**React/Angular components**: Namespaces are optional — CSS Modules / Angular encapsulation already provide scope. Use plain BEM inside `.module.css` or `.component.scss`.

### No External Margins Rule

**Components MUST NOT define their own external margins (`margin-top`, `margin-bottom`, `margin`, `gap` applied to self).**

A component with a hardcoded `margin-top: 24px` cannot be reused in a layout that requires no margin, or a different spacing. It couples the component to a specific context.

**Spacing between components is always the responsibility of the parent layout** — via Object classes or Utility classes.

```css
/* Forbidden — component owns its own external spacing */
.c-card {
  margin-top: var(--space-6);   /* breaks reuse */
}

/* Correct — parent controls spacing via layout object */
.o-stack > * + * {
  margin-top: var(--space-6);
}

/* Correct — parent controls spacing via utility */
<div class="u-mt-6">
  <UserCard />
</div>
```

This mirrors the same principle as the frontend architecture layer rule: presentational components receive all context via props/inputs — they do not reach out for their environment.

---

## Layer 5: Component Styles

### React — CSS Modules

Every presentational component has a co-located `.module.css`. **All class names MUST be camelCase.** This allows dot-notation access in TypeScript (`styles.cardFeatured`) instead of the error-prone string-index form (`styles['card--featured']`), and enables safe compile-time refactoring.

```
components/users/user-card/
  user-card.tsx
  user-card.module.css
  user-card.test.tsx
```

```css
/* user-card.module.css */
/* Class names: camelCase only — no kebab-case, no BEM double-hyphen */
.card {
  background: var(--color-bg-surface);
  border: 1px solid var(--color-border);
  border-radius: var(--radius-md);
  padding: var(--spacing-card-pad);
  box-shadow: var(--shadow-sm);
  transition: box-shadow var(--duration-fast) var(--ease-out);
}

.cardFeatured {                  /* camelCase modifier — not .card--featured */
  border-color: var(--color-brand);
  box-shadow: var(--shadow-md);
}

.avatar {
  width: var(--space-12);
  height: var(--space-12);
  border-radius: var(--radius-full);
}

.name {
  font-size: var(--text-sm);
  font-weight: 600;
  color: var(--color-text-primary);
}
```

```tsx
import clsx from 'clsx'
import styles from './user-card.module.css'

export function UserCard({ user, featured }: UserCardProps) {
  return (
    // Dot-notation access — type-safe, refactor-friendly
    <div className={clsx(styles.card, featured && styles.cardFeatured)}>
      <img className={styles.avatar} src={user.avatarUrl} alt={user.name} />
      <span className={styles.name}>{user.name}</span>
    </div>
  )
}
```

**Rules:**
- One `.module.css` per component file — no sharing between components
- **CSS Module class names MUST be camelCase** — `cardFeatured`, not `card--featured` or `card_featured`
- All values via CSS custom properties — no hardcoded hex, px magic numbers
- Use `clsx` (or `clsx` + `tailwind-merge` for Tailwind projects) for conditional class composition — never manual template literals
- No global selectors inside a module — **except** `:global` strictly for third-party library overrides (see below)
- Shared layout helpers live in `styles/objects/` — import them in `main.scss`, not in modules
- Use `composes` for extracting repeated patterns within the same module

#### `:global` — Third-Party Override Exception

When styling a third-party component that does not accept a `className` prop or custom class, use `:global` scoped under a local wrapper class. This prevents the override from leaking globally while still targeting the library's internal class.

```css
/* user-date-picker.module.css */
.wrapper :global(.rdp-day_selected) {
  background: var(--color-brand);
  color: var(--color-text-inverse);
}

.wrapper :global(.rdp-nav_button):hover {
  background: var(--color-bg-subtle);
}
```

```tsx
<div className={styles.wrapper}>
  <DayPicker ... />
</div>
```

**Rules for `:global`:**
- ONLY permitted for third-party library class overrides — never for application code
- ALWAYS scoped under a local wrapper class (`.wrapper :global(...)`) — never bare `:global(...)`
- Add a comment naming the library being targeted

### Angular — Scoped SCSS

Each component's `.component.scss` is automatically scoped by Angular's Emulated encapsulation.

```scss
// user-card.component.scss
@use '../../../styles/abstracts/tokens' as t;  // if re-exporting vars via SCSS
// Or simply reference CSS custom properties — they cascade through the DOM

.card {
  background: var(--color-bg-surface);
  border: 1px solid var(--color-border);
  border-radius: var(--radius-md);
  padding: var(--spacing-card-pad);
}

.card--featured {
  border-color: var(--color-brand);
}

.avatar {
  width: var(--space-12);
  height: var(--space-12);
  border-radius: var(--radius-full);
}
```

**Rules:**
- NEVER use `::ng-deep` — it breaks encapsulation globally and is deprecated
- NEVER use `ViewEncapsulation.None` on components that have broad selectors
- If a child component needs a style override, pass it via an `input()` CSS class or a CSS custom property scoped to `:host`
- Use `:host` and `:host-context()` for component-level containment

```scss
// Correct way to style based on host context
:host {
  display: block;
}

:host(.is-featured) .card {
  border-color: var(--color-brand);
}
```

### Component Styling API — Local CSS Variable Override

When a component needs to be customizable from the outside (e.g., a card placed on a colored hero background), **expose a local CSS variable as the public styling API**. The component maps a local variable to its default global token. The parent overrides only that local variable — no selector coupling, no `::ng-deep`, no string props.

```css
/* user-card.module.css */
/* Public CSS API — document exposed local variables here:
 *   --card-bg           override background (default: --color-bg-surface)
 *   --card-border-color override border    (default: --color-border)
 */
.card {
  background:   var(--card-bg,           var(--color-bg-surface));
  border-color: var(--card-border-color, var(--color-border));
  color:        var(--card-text-color,   var(--color-text-primary));
}
```

```tsx
{/* React — inline style sets the local variable scoped to this element */}
<UserCard
  user={user}
  style={{ '--card-bg': 'var(--color-brand-subtle)' } as React.CSSProperties}
/>
```

```html
<!-- Angular — bind the local variable via [style] -->
<app-user-card [style.--card-bg]="'var(--color-brand-subtle)'" />
```

**Rules:**
- Local variable names are prefixed with the component name (`--card-*`) — they form the component's public CSS API
- The fallback MUST be a global semantic token: `var(--local-var, var(--color-semantic-token))`
- Document all exposed local variables in a comment block at the top of the module/SCSS file
- This is the ONLY approved mechanism for a parent to change a child's internal appearance — never a descendant selector override

---

## Accessibility — Focus & Motion

### Focus Indicators

**Never remove the focus ring without replacing it.** `outline: none` without a `:focus-visible` replacement is an accessibility failure — keyboard users lose their location indicator.

The global focus style lives in `base/_elements.scss` and uses `--color-focus-ring` tokens:

```scss
// base/_elements.scss
// Global :focus-visible — applies to all interactive elements by default
:focus-visible {
  outline: 2px solid var(--color-focus-ring);
  outline-offset: 2px;
}

// Remove default outline only when :focus-visible is defined
// (browsers that don't support :focus-visible still show outlines)
:focus:not(:focus-visible) {
  outline: none;
}
```

Components that need a custom focus appearance MUST still use the token:

```css
/* user-card.module.css */
.card:focus-visible {
  outline: 2px solid var(--color-focus-ring);
  outline-offset: 3px;
  border-radius: var(--radius-md);
}
```

**Rules:**
- `outline: none` or `outline: 0` without a `:focus-visible` replacement is **forbidden**
- All custom focus styles MUST use `--color-focus-ring` and `--color-focus-ring-offset`
- Use `:focus-visible`, not `:focus` — `:focus` fires on mouse click too, `:focus-visible` only fires for keyboard/programmatic focus
- Dark mode MUST include a `--color-focus-ring` override that remains high-contrast on dark surfaces

### Reduced Motion

Animations and transitions MUST respect `prefers-reduced-motion`. Define a global mixin in `abstracts/` and apply it wherever motion is used.

```scss
// abstracts/_mixins.scss

// Wrap any transition or animation in this mixin
// to automatically disable it for users who prefer reduced motion.
@mixin motion-safe {
  @media (prefers-reduced-motion: no-preference) {
    @content;
  }
}

// For cases where a subtler motion is acceptable (opacity, color) even with reduced motion
@mixin motion-reduce {
  @media (prefers-reduced-motion: reduce) {
    @content;
  }
}
```

```css
/* user-card.module.css */
.card {
  background: var(--color-bg-surface);
}

/* Transition only plays if the user has not requested reduced motion */
@media (prefers-reduced-motion: no-preference) {
  .card {
    transition: box-shadow var(--duration-fast) var(--ease-out);
  }
}
```

```scss
// In Angular SCSS — using the mixin
@use '../../../styles/abstracts' as a;

.card {
  background: var(--color-bg-surface);

  @include a.motion-safe {
    transition: box-shadow var(--duration-fast) var(--ease-out);
  }
}
```

**Rule:** Every `transition` and `animation` declaration MUST be wrapped in `prefers-reduced-motion: no-preference` — either via the mixin or inline media query.

---

## Specificity Rules

1. **Never use IDs (`#id`) in CSS** — specificity is too high, cannot be overridden without `!important`
2. **Avoid chaining class selectors** (`.card.featured`) — increases specificity unnecessarily
3. **Keep selectors shallow** — max 2-3 levels deep (`.c-button .icon` is acceptable; `.page .section .card .header .title` is not)
4. **`!important` is only legal in `utilities/`** — never in component, object, or base styles
5. **Inline styles via JS are forbidden for layout/theme** — use CSS custom property updates instead

```typescript
// Forbidden — bypasses all CSS architecture
element.style.backgroundColor = '#2563eb'

// Correct — update a token, let CSS cascade
element.style.setProperty('--color-brand', '#2563eb')
```

---

## SCSS Module System

Always use `@use` and `@forward`. Never use `@import` (deprecated in Sass).

```scss
// abstracts/_index.scss — barrel file for abstracts
@forward 'tokens';
@forward 'mixins';
@forward 'functions';

// In a component SCSS file
@use '../../../styles/abstracts' as a;

.card {
  padding: a.$card-padding;   // SCSS variable from abstracts
  background: var(--color-bg-surface);  // CSS custom property
}
```

**Rules:**
- Only import `abstracts/` partials into component SCSS files — never import `base/`, `objects/`, or `utilities/` (those are global, loaded once in `main.scss`)
- Importing a full stylesheet into a component causes **style duplication** in the bundle — one copy per component that imports it
- Use SCSS variables (`$`) for build-time values (breakpoints, SCSS-only calculations), CSS custom properties (`--`) for runtime values (theming, dynamic overrides)
- Always use a namespace alias to prevent variable name collisions: `@use '...' as t` not `@use '...'`

### Barrel File vs. Specific Import

The `abstracts/_index.scss` barrel is convenient but imports everything (tokens + mixins + functions) into every component file. In projects with 200+ components this increases build memory and incremental rebuild time.

**Prefer specific partial imports** when a component only needs one thing:

```scss
// Component only needs mixins — import only mixins, not the full barrel
@use '../../../styles/abstracts/mixins' as m;

.card-grid {
  @include m.md { grid-template-columns: repeat(2, 1fr); }
}
```

```scss
// Component needs both tokens (SCSS vars) and mixins — barrel is acceptable here
@use '../../../styles/abstracts' as a;

.card {
  padding: a.$card-padding;
  @include a.motion-safe { transition: box-shadow var(--duration-fast) var(--ease-out); }
}
```

**Rule:** Only use the barrel (`abstracts/_index.scss`) when a component genuinely needs variables from multiple partials. For components that only call mixins, import `abstracts/mixins` directly. CSS custom properties (`var(--)`) require no import at all — they cascade through the DOM.

```scss
// Good — breakpoints are build-time, not runtime
$breakpoint-md: 768px;

@mixin respond-to-md {
  @media (min-width: $breakpoint-md) { @content; }
}

// Good — theme values are runtime
.card {
  background: var(--color-bg-surface);
}
```

---

## Logical Properties (RTL/LTR Support)

Writing `margin-left` or `padding-right` is a physical property — it is hard-coded to a visual direction. In Right-to-Left (RTL) languages (Arabic, Hebrew) or vertical writing modes, physical properties require a full second stylesheet to override. Logical properties adapt automatically.

**Rule: Use CSS Logical Properties for all directional spacing and sizing.**

| Physical (forbidden) | Logical (required) | Meaning |
|---|---|---|
| `margin-left` | `margin-inline-start` | Start of the inline axis |
| `margin-right` | `margin-inline-end` | End of the inline axis |
| `margin-top` / `margin-bottom` | `margin-block-start` / `margin-block-end` | Block axis edges |
| `margin-left` + `margin-right` | `margin-inline` | Both inline edges |
| `margin-top` + `margin-bottom` | `margin-block` | Both block edges |
| `padding-left` | `padding-inline-start` | |
| `padding-right` | `padding-inline-end` | |
| `padding-top` / `padding-bottom` | `padding-block-start` / `padding-block-end` | |
| `padding-left` + `padding-right` | `padding-inline` | |
| `padding-top` + `padding-bottom` | `padding-block` | |
| `width` (when meaning inline size) | `inline-size` | |
| `height` (when meaning block size) | `block-size` | |
| `text-align: left` | `text-align: start` | |
| `text-align: right` | `text-align: end` | |
| `border-left` | `border-inline-start` | |

```css
/* Forbidden — physical, breaks RTL */
.card {
  padding-left: var(--spacing-card-pad);
  padding-right: var(--spacing-card-pad);
  margin-top: var(--space-4);
  text-align: left;
}

/* Correct — logical, RTL-safe */
.card {
  padding-inline: var(--spacing-card-pad);
  margin-block-start: var(--space-4);
  text-align: start;
}
```

**Exceptions:** `border-radius` corners and `background-position` do not yet have full logical equivalents in all browsers — use physical values there until support improves.

**Activate RTL** by setting `dir="rtl"` on the `<html>` element. No CSS changes required when logical properties are used throughout.

---

## Responsive Design

Define breakpoints as SCSS variables in `abstracts/`. Use mixins, not raw `@media` literals.

```scss
// abstracts/_mixins.scss
$bp-sm: 640px;
$bp-md: 768px;
$bp-lg: 1024px;
$bp-xl: 1280px;

@mixin sm  { @media (min-width: $bp-sm)  { @content; } }
@mixin md  { @media (min-width: $bp-md)  { @content; } }
@mixin lg  { @media (min-width: $bp-lg)  { @content; } }
@mixin xl  { @media (min-width: $bp-xl)  { @content; } }
```

```scss
// Usage in component
@use '../../../styles/abstracts' as a;

.card-grid {
  display: grid;
  grid-template-columns: 1fr;

  @include a.md {
    grid-template-columns: repeat(2, 1fr);
  }

  @include a.lg {
    grid-template-columns: repeat(3, 1fr);
  }
}
```

**Rules:**
- Mobile-first: base styles target small screens, breakpoints expand upward
- No raw `px` media query literals in component files — always use the mixin
- Breakpoint variables live only in `abstracts/` — never duplicated in components

### Container Queries — Component-Intrinsic Responsiveness

In component-based architectures, a component should respond to **its container's width**, not the browser viewport. Using only `@media` forces components to know about the page layout, breaking encapsulation.

**Define containment in the `objects/` layer**, then use `@container` in component styles.

```scss
// styles/objects/_container.scss

// Generic page container
.o-container {
  max-width: 1280px;
  margin-inline: auto;
  padding-inline: var(--spacing-page-x);
}

// Containment region — establishes a named container context
.o-container-region {
  container-type: inline-size;
}

// Named container for targeted queries
.o-container-card {
  container-type: inline-size;
  container-name: card;
}
```

```css
/* user-card.module.css — responds to container, not viewport */
.card {
  display: flex;
  flex-direction: column;
  gap: var(--space-2);
}

/* When the card's container is >= 360px, switch to row layout */
@container (min-width: 360px) {
  .card {
    flex-direction: row;
    align-items: center;
  }

  .avatar {
    flex-shrink: 0;
  }
}
```

```tsx
{/* Wrap the card in a containment region so @container has a context */}
<div className="o-container-card">
  <UserCard user={user} />
</div>
```

**Rules:**
- Prefer `@container` over `@media` for component-level layout changes
- Use `@media` for page-level layout (sidebar visible/hidden, column count on the grid page)
- The containment wrapper (`.o-container-*`) lives in `objects/` — components do not add `container-type` to themselves
- Name containers when multiple nested container contexts exist to avoid ambiguous query targets
- `@container` queries follow the same mixin pattern — define named-container mixins in `abstracts/_mixins.scss`

```scss
// abstracts/_mixins.scss — container query mixins
@mixin container-sm { @container (min-width: 320px) { @content; } }
@mixin container-md { @container (min-width: 480px) { @content; } }
@mixin container-lg { @container (min-width: 640px) { @content; } }
```

---

## Anti-Patterns

| Anti-pattern | Problem | Fix |
|---|---|---|
| Hardcoded hex / px values | Values diverge across codebase, impossible to theme | Use design tokens |
| `!important` in component styles | Creates permanent specificity debt | Remove; fix selector structure |
| `::ng-deep` in Angular | Breaks encapsulation globally, deprecated | Use `:host`, `input()` class, or CSS custom property |
| Global class names without namespace | Name collisions across features | Use CSS Modules (React) or BEM + BEMIT (global) |
| Overriding child component styles from parent | Tight coupling between components | Pass style via `input()` class or CSS custom property |
| `@import` in SCSS | Deprecated, causes duplication | Use `@use` / `@forward` |
| Duplicated spacing/color constants | Updates require hunting all occurrences | Define once in `_tokens.scss` |
| Inline `style={{}}` for theme/layout | Cannot be themed or overridden by CSS | Use class + tokens |
| Deep selectors (4+ levels) | Brittle, breaks on HTML structure changes | Flatten with BEM elements |
| Empty or missing `_tokens.scss` | First thing removed under time pressure | Enforce in code review; generate at scaffold time |
| `z-index` magic numbers | Stacking context becomes unpredictable across features | Use `--z-*` tokens exclusively |
| Hardcoded transition durations/easing | Motion feels inconsistent across the app | Use `--duration-*` and `--ease-*` tokens |
| External margins on components | Component cannot be reused in different spacing contexts | Parent controls spacing via layout objects or utilities |
| `@media` for component-level layout | Component couples itself to viewport, breaks encapsulation | Use `@container` with a containment wrapper in `objects/` |
| Bare `:global()` in CSS Modules | Styles leak globally | Always scope under a local wrapper class |
| Physical properties (`margin-left`, `padding-right`) | Breaks RTL/LTR localization — requires a full override stylesheet | Use Logical Properties (`margin-inline`, `padding-block`) |
| `height: 100vh` | Broken on mobile — URL bar is included in `vh` | Use `100dvh` (Dynamic Viewport Height) |
| Font sizes in `px` | Breaks when the user increases browser default font size | Always use `rem` for typography; `px` only for borders and decorative hairlines |

---

## CSS Clean Architecture Checklist

Before committing styles:

- [ ] All colors, spacing, radii, and shadows reference a design token (`var(--...)`)
- [ ] No hardcoded hex, `px`, or `rem` values outside `abstracts/_tokens.scss`
- [ ] `!important` only appears in `styles/utilities/` — never in component styles
- [ ] React: one `.module.css` per component, no global selectors inside modules
- [ ] Angular: no `::ng-deep`, no `ViewEncapsulation.None` on broad-selector components
- [ ] Angular: `:host` used for component-level containment
- [ ] SCSS files use `@use` / `@forward` — no `@import`
- [ ] Only `abstracts/` partials are imported in component SCSS files
- [ ] Responsive styles use mixin breakpoints, not raw `@media` literals
- [ ] New theme variants override semantic tokens only, not primitives
- [ ] BEM class names: no element chains (`__x__y`), modifiers always paired with their block
- [ ] No inline `style` attributes for layout or theme values
- [ ] Dark mode semantic tokens defined for any new semantic token added
- [ ] Component has zero external margins — external spacing is parent-controlled via objects or utilities
- [ ] All `z-index` values use `--z-*` tokens — no raw integers
- [ ] All transitions use `--duration-*` and `--ease-*` tokens — no raw `ms` or keyword easing values
- [ ] Component-level layout changes use `@container` — `@media` reserved for page-level layout only
- [ ] SCSS: `@use` includes a namespace alias (`as t`, `as m`, `as a`) — no bare `@use` without alias
- [ ] SCSS: barrel file (`abstracts/_index.scss`) only used when multiple partials are needed — otherwise import the specific partial
- [ ] React: conditional classes use `clsx` — no manual template literal concatenation
- [ ] React: `:global()` only targets third-party library classes and is always scoped under a local wrapper
- [ ] React: CSS Module class names are camelCase — dot-notation access only, no string-index lookup
- [ ] Accessibility: interactive elements have a visible `:focus-visible` state using `--color-focus-ring`
- [ ] Accessibility: no `outline: none` / `outline: 0` without a replacement `:focus-visible` rule
- [ ] Accessibility: all `transition` and `animation` declarations are wrapped in `prefers-reduced-motion: no-preference`
- [ ] Component Styling API: if a parent needs to change a child's internal style, it is done via a local CSS variable (`--component-*`), not a descendant selector
- [ ] CSS Layers: `main.scss` declares `@layer generic, base, objects, components, utilities` before any rules
- [ ] Logical Properties: all directional spacing uses `margin-inline`, `padding-block`, etc. — no `margin-left`, `padding-right`
- [ ] Typography: all font sizes use `rem` — `px` only for borders and decorative hairlines
- [ ] Naming: BEM used only in global stylesheets — CSS Modules use camelCase only, no BEM modifiers
- [ ] Contrast: new semantic color token pairings pass WCAG AA (4.5:1 text, 3:1 UI) — verified via linter or manual check
