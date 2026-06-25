# CSS & responsive design review reference

Deep checklist for layout, breakpoints, and responsive behavior. Each item pairs a
❌ bad pattern with a ✅ fix and the user-facing consequence. Default to
**mobile-first**, fluid, and content-driven layouts.

---

## Mobile-first breakpoints

❌ Desktop-first with `max-width` queries, and collapsing to one column far too early:
```css
.grid { grid-template-columns: repeat(4, 1fr); }
@media (max-width: 1200px) { .grid { grid-template-columns: 1fr; } } /* single column already at 1199px */
```
✅ Mobile-first `min-width`, adding columns as space allows:
```css
.grid { grid-template-columns: 1fr; }              /* phones */
@media (min-width: 600px)  { .grid { grid-template-columns: repeat(2, 1fr); } }
@media (min-width: 960px)  { .grid { grid-template-columns: repeat(3, 1fr); } }
@media (min-width: 1280px) { .grid { grid-template-columns: repeat(4, 1fr); } }
```
**Consequence:** collapsing to a single column at the first hint of narrowing wastes tablet/laptop space and makes the design look broken on the most common screens. Base styles target the smallest screen; layers add complexity for wider viewports.

### Let content choose breakpoints

🔵 Don't hard-code breakpoints to specific device widths (`768px = iPad`). Add a breakpoint **where the content starts to break** (line length, wrapping, overflow). Device sizes change; content needs don't.

---

## Fluid layouts (size between breakpoints, not just at them)

❌ Fixed widths and font sizes that only change at breakpoints (jumpy resizing).
✅ Intrinsically responsive values:
```css
.container { width: min(100% - 2rem, 1200px); margin-inline: auto; }
h1 { font-size: clamp(1.5rem, 1rem + 2.5vw, 3rem); }
.card { flex: 1 1 min(100%, 18rem); }                 /* wraps automatically */
.auto-grid { grid-template-columns: repeat(auto-fit, minmax(16rem, 1fr)); }
```
**Consequence:** `clamp()`, `min()`, `max()`, and `auto-fit/minmax` let the layout flex smoothly between breakpoints, so you need fewer media queries and avoid awkward in-between states.

### Container queries for reusable components

❌ A card that styles itself off the **viewport** width, so it looks wrong in a narrow sidebar on a wide screen.
✅
```css
.card-wrap { container-type: inline-size; }
@container (min-width: 24rem) { .card { display: grid; grid-template-columns: auto 1fr; } }
```
**Consequence:** components should respond to **their own** available space, not the page width — viewport media queries break reusable components placed in varying containers.

---

## Grid vs Flexbox

🔵 **Flexbox** for one-dimensional flow (a row or column, content-sized, wrapping toolbars/nav). **Grid** for two-dimensional layouts (rows *and* columns aligned).

❌ Nested flex hacks to fake a 2D grid (brittle, misaligned).
✅ Use `display: grid` with explicit tracks for 2D layouts.
**Consequence:** forcing the wrong tool produces fragile alignment that breaks when content length varies.

---

## Overflow & scroll traps

❌ Fixed widths or `100vw` causing horizontal scroll on mobile:
```css
.banner { width: 100vw; }   /* 100vw includes the scrollbar → overflow */
```
✅
```css
.banner { width: 100%; }
/* and guard wide content: */
img, video, pre, table { max-width: 100%; }
```
**Consequence:** unexpected horizontal scroll is a top mobile bug — content gets cut off and the page feels broken. Watch long unbroken strings (`overflow-wrap: anywhere`) and wide tables (wrap in a scroll container).

---

## Hover-only affordances on touch

❌ A menu or action that only appears on `:hover`:
```css
.actions { opacity: 0; }
.card:hover .actions { opacity: 1; }   /* invisible/unreachable on touch */
```
✅ Make it always visible/focusable, or gate hover-only enhancements behind a capability query:
```css
@media (hover: hover) and (pointer: fine) { .card:hover .actions { opacity: 1; } }
```
**Consequence:** touch users have no hover, so hover-only controls are undiscoverable or unreachable. Ensure the action is also reachable by tap and keyboard `:focus-within`.

---

## z-index & stacking

❌ Escalating magic numbers (`z-index: 9999`) scattered across files.
✅ Define a small named scale and use stacking contexts deliberately:
```css
:root { --z-dropdown: 10; --z-sticky: 20; --z-modal: 30; --z-toast: 40; }
```
**Consequence:** z-index wars cause modals to hide behind headers or tooltips to vanish. Remember stacking contexts are created by `transform`, `opacity < 1`, `filter`, etc. — a high `z-index` won't escape its parent context.

---

## Logical properties (RTL & writing modes)

❌ Physical properties that break in right-to-left locales:
```css
.card { margin-left: 1rem; padding-right: 2rem; text-align: left; }
```
✅
```css
.card { margin-inline-start: 1rem; padding-inline-end: 2rem; text-align: start; }
```
**Consequence:** physical left/right hard-code one direction, so RTL languages (Arabic, Hebrew) render mirrored/wrong. Logical properties adapt automatically.

---

## Dark mode & theming

✅ Drive colors through tokens and respect the user's preference:
```css
:root { color-scheme: light dark; --bg: #fff; --fg: #111; }
@media (prefers-color-scheme: dark) { :root { --bg: #111; --fg: #eee; } }
body { background: var(--bg); color: var(--fg); }
```
**Consequence:** hard-coded colors can't theme, and ignoring `prefers-color-scheme` forces a bright UI on users who chose dark. Set `color-scheme` so form controls/scrollbars theme too. Re-check contrast in **both** themes.

---

## Reduced motion

✅ Gate non-essential transitions/animations:
```css
@media (prefers-reduced-motion: reduce) {
  *, *::before, *::after { animation-duration: .01ms !important; transition-duration: .01ms !important; scroll-behavior: auto !important; }
}
```
**Consequence:** large motion can trigger nausea/vestibular issues; honor the user's setting (see accessibility reference).

---

## Hit-target sizes

✅ Interactive targets at least ~24×24px (44×44 comfortable on touch), with adequate spacing:
```css
.icon-btn { min-width: 44px; min-height: 44px; }
```
**Consequence:** cramped tap targets cause mis-taps, especially on mobile and for motor-impaired users.

---

## Fixed heights that clip content

❌
```css
.card { height: 220px; overflow: hidden; }   /* clips longer translations / large text */
```
✅
```css
.card { min-height: 220px; }                  /* grows with content */
```
**Consequence:** fixed `height` clips content when text is longer (other languages), the user zooms, or a larger font is set — text gets cut off. Prefer `min-height`; let content dictate height. Likewise avoid disabling zoom (`user-scalable=no`).

---

## Quick scan checklist

- [ ] Mobile-first `min-width` queries; doesn't collapse to one column too early.
- [ ] Breakpoints chosen by content, not specific device widths.
- [ ] Fluid sizing (`clamp`/`min`/`max`, `auto-fit minmax`) between breakpoints.
- [ ] Reusable components use container queries, not viewport queries.
- [ ] Grid for 2D, Flexbox for 1D — no nested-flex grid hacks.
- [ ] No horizontal overflow; media/tables/long strings constrained.
- [ ] No hover-only controls on touch; gated behind `@media (hover: hover)` + focus fallback.
- [ ] z-index uses a named scale; stacking contexts understood.
- [ ] Logical properties for spacing/alignment (RTL-safe).
- [ ] Theming via tokens; `prefers-color-scheme` + `color-scheme`; contrast checked in both.
- [ ] `prefers-reduced-motion` honored.
- [ ] Tap targets ≥ ~24px; zoom not disabled.
- [ ] `min-height` over fixed `height`; content not clipped.
