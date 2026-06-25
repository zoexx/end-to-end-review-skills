# Frontend review checklist

Copy-paste into a PR and tick what applies. Grouped by the four review pillars
plus tooling. Severity: 🔴 blocking · 🟠 important · 🟡 nit · 🔵 suggestion ·
📚 learning · 🌟 praise.

## 1. Code architecture & maintainability

- [ ] State has a single, clear owner; no imperative DOM clobbering fighting the framework.
- [ ] Reusable components take props/slots and own no app-specific business logic.
- [ ] Hook/effect dependency arrays are complete and honest (no suppressed exhaustive-deps).
- [ ] No stale-closure bug: handlers/effects don't capture outdated state; functional updates used.
- [ ] No needless re-renders: stable callback/object identity; memoization only where it pays.
- [ ] Derived values computed in render / `computed` / signals — not synced via effects.
- [ ] Props not copied into local state (or remounted via `key` when reset is intended).
- [ ] List keys / `track` use stable ids, never the array index for reorderable lists.
- [ ] Business logic separated from presentation (hooks/composables/services vs views).
- [ ] TypeScript is meaningful — no `any` on props/state; variants modeled as unions.
- [ ] Naming and structure match the surrounding codebase / style guide.

## 2. User experience & design fidelity

- [ ] Responsive: handles tablet/mobile edge cases; doesn't collapse to one column too early.
- [ ] No horizontal scroll at common widths; long strings/tables/media constrained.
- [ ] Micro-interactions present: hover, active, AND focus states; smooth transitions.
- [ ] Empty state designed.
- [ ] Loading state designed (skeleton/spinner).
- [ ] Error / "no results" state designed.
- [ ] Matches design spec (spacing, type scale, color); deviations intentional and noted.
- [ ] Touch targets large enough; hover-only affordances have touch/keyboard equivalents.

## 3. Performance & Core Web Vitals

- [ ] LCP element eager + prioritized (`fetchpriority`/preload); not lazy-loaded.
- [ ] No needless render-blocking CSS/JS; non-critical scripts `defer`/`async`.
- [ ] CLS: every image/video/iframe/ad has dimensions or `aspect-ratio`.
- [ ] Fonts use `font-display: swap` + metric-matched fallback (no swap reflow).
- [ ] Injected content (banners/cookie bars) reserves space or overlays — doesn't push content.
- [ ] INP: search/filter/scroll/resize handlers debounced/throttled (passive scroll).
- [ ] No long tasks > 50ms on interaction; heavy work chunked/Worker/deferred.
- [ ] Long lists virtualized.
- [ ] Routes code-split; heavy optional features dynamically imported.
- [ ] No barrel-file imports defeating tree-shaking; per-function lib imports.
- [ ] Images: modern format (AVIF/WebP), responsive `srcset`/`sizes`, below-fold `lazy`.
- [ ] Regression confirmed against field/RUM, not just one lab run.

## 4. Accessibility (a11y) & SEO

- [ ] Native elements for buttons/links/inputs; landmarks (`<nav>/<main>/...`) used.
- [ ] One `<h1>`; no skipped heading levels.
- [ ] Every interactive control has an accessible name; placeholders aren't the only label.
- [ ] Fully keyboard operable; logical tab order; skip link; no positive `tabindex`.
- [ ] Visible `:focus-visible` ring; no blanket `outline: none`.
- [ ] Modals trap focus, close on Esc, and return focus to the trigger.
- [ ] ARIA only where native won't do, states kept in sync; decorative icons `aria-hidden`.
- [ ] Color contrast ≥ 4.5:1 text / ≥ 3:1 UI; meaning never conveyed by color alone.
- [ ] `prefers-reduced-motion` honored.
- [ ] Form errors associated (`aria-describedby`) and announced (`role="alert"`/live region).
- [ ] Live region for async status updates (toasts, result counts).
- [ ] Meaningful `alt`; decorative images `alt=""`.
- [ ] Touch/hit targets ≥ ~24×24px.
- [ ] SEO: `<title>`/meta description per route; structured data where relevant.

## Review tooling

- [ ] Chromatic — visual/snapshot regression run; intended diffs reviewed.
- [ ] Ruttl — design-fidelity feedback gathered on staging where relevant.
- [ ] Lighthouse — LCP/CLS/INP, a11y, SEO scores checked.
- [ ] axe DevTools — automated a11y scan clean (ARIA, contrast, names, landmarks).
- [ ] WebPageTest — waterfall/filmstrip reviewed for suspected perf regressions.

---

### Verdict

✅ approve · 💬 approve with comments · 🔁 request changes · ⛔ block

Counts: 🔴 _ · 🟠 _ · 🟡 _ · 🔵 _ · 🌟 _
