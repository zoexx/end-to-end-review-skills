# Web Vitals & performance review reference

Deep checklist for the three Core Web Vitals — **LCP**, **CLS**, **INP** — plus
bundle, images, fonts, and network. Each item pairs a ❌ bad pattern with a ✅ fix
and the user-facing consequence. Targets (field/p75): LCP ≤ 2.5s, CLS ≤ 0.1,
INP ≤ 200ms.

---

## LCP — Largest Contentful Paint (loading)

### Prioritize the LCP element

❌ The hero image lazy-loaded and undiscovered until late:
```html
<img src="hero.jpg" loading="lazy">
```
✅ Eagerly load and prioritize the above-the-fold hero:
```html
<img src="hero.jpg" fetchpriority="high" decoding="async" width="1200" height="600">
<!-- or preload it -->
<link rel="preload" as="image" href="hero.jpg" fetchpriority="high">
```
**Consequence:** lazy-loading or de-prioritizing the LCP image delays the main paint — users stare at a blank/partial page longer. Never lazy-load above-the-fold imagery.

### Remove render-blocking resources

❌ Large synchronous CSS/JS in `<head>` blocking first paint; a blocking third-party script.
✅ Inline critical CSS, defer the rest; `defer`/`async` non-critical scripts; split CSS so only above-the-fold styles block.
```html
<script src="analytics.js" defer></script>
```
**Consequence:** every render-blocking byte pushes back the moment content appears. Audit the critical request chain in Lighthouse.

### Server/data latency

🔵 Slow TTFB (server, edge cache misses, waterfalled data fetches) directly inflates LCP. Stream HTML, cache at the edge, and avoid client-side fetch waterfalls for above-the-fold content (fetch in parallel / on the server).

---

## CLS — Cumulative Layout Shift (visual stability)

### Always reserve space for media

❌
```html
<img src="thumb.jpg"> <!-- no dimensions; reflows when it loads -->
```
✅
```html
<img src="thumb.jpg" width="320" height="180"> <!-- or aspect-ratio in CSS -->
```
```css
.thumb { aspect-ratio: 16 / 9; width: 100%; height: auto; }
```
**Consequence:** an unsized image pushes content down when it arrives — users misclick or lose their place. Set `width`/`height` (or `aspect-ratio`) on **every** image, video, iframe, and ad slot.

### Font swap reflow

❌ A web font that loads late and reflows text (FOUT/FOIT with metric mismatch).
✅
```css
@font-face { font-family: Inter; src: url(inter.woff2) format('woff2'); font-display: swap; }
```
Pair with `size-adjust`/`ascent-override` or a well-matched fallback in `font-family` to minimize the swap shift; `preload` the primary font.
**Consequence:** a poorly matched fallback → web font swap visibly jumps the text layout.

### Dynamically injected content

❌ Banners, cookie bars, or async-loaded sections inserted **above** existing content.
✅ Reserve a fixed-height slot, insert below the fold, or overlay (position over, don't push) content.
**Consequence:** injecting content above what the user is reading shoves everything down mid-interaction.

---

## INP — Interaction to Next Paint (responsiveness)

### Debounce / throttle high-frequency input

❌
```js
input.addEventListener('input', e => fetchResults(e.target.value)); // fires every keystroke
window.addEventListener('scroll', () => recalcLayout());           // fires constantly
```
✅
```js
input.addEventListener('input', debounce(e => fetchResults(e.target.value), 300));
window.addEventListener('scroll', throttle(recalcLayout, 100), { passive: true });
```
**Consequence:** un-throttled handlers run expensive work on every event, blocking the main thread so the UI lags behind the user's input (high INP, janky scroll). Use `{ passive: true }` for scroll/touch.

### Break up long tasks

❌ A single synchronous loop processing thousands of items on click.
✅ Chunk the work, yield to the main thread (`await scheduler.yield()` / `setTimeout` / `requestIdleCallback`), or move it to a Web Worker.
**Consequence:** any task > 50ms blocks input handling; the page feels frozen. Defer non-urgent work (React `useTransition`/`useDeferredValue`, Angular signals/`afterNextRender`).

### Virtualize long lists

❌ Rendering 5,000 DOM rows.
✅ Windowing (`react-window`/`@tanstack/virtual`, CDK `*cdkVirtualFor`, Vue `vue-virtual-scroller`).
**Consequence:** huge DOM trees are slow to render, scroll, and update — memory and interaction latency both suffer.

### Keep event handlers cheap

🔵 Avoid forced synchronous layout ("layout thrashing") — don't read `offsetHeight`/`getBoundingClientRect` and then write styles in a loop. Batch reads then writes.

---

## Bundle & code-splitting

- **Route-level splitting:** lazy-load each route (`React.lazy`/dynamic `import()`, Angular `loadComponent`, Vue async components) so users download only the current view.
- **Dynamic import** heavy/optional code (charts, editors, modals) on demand:
  ```ts
  const Chart = lazy(() => import('./Chart'));
  ```
- **Barrel-file bloat:** importing from a re-exporting `index.ts` can pull the whole module graph and defeat tree-shaking.
  ❌ `import { Button } from '@/components';`
  ✅ `import { Button } from '@/components/Button';`
- **Tree-shaking:** ensure deps are ESM and side-effect-free (`"sideEffects": false`); avoid importing an entire utility lib for one function (`import debounce from 'lodash/debounce'`, not `import _ from 'lodash'`).
- **Analyze:** check the bundle (`vite-bundle-visualizer`, `webpack-bundle-analyzer`, `source-map-explorer`) for surprise large deps and duplicates.

**Consequence:** every shipped KB is parse/compile/execute time on the user's CPU — bloated bundles hurt LCP and INP, especially on mid-range mobile.

---

## Images

- Modern formats: **AVIF/WebP** with a fallback; compress to a sane quality (don't ship 4000px originals into a 400px slot).
- Responsive `srcset` + `sizes` so each device gets an appropriately sized file:
  ```html
  <img src="p-800.jpg" srcset="p-400.jpg 400w, p-800.jpg 800w, p-1600.jpg 1600w"
       sizes="(max-width: 600px) 100vw, 50vw" width="800" height="600" alt="…">
  ```
- `loading="lazy"` for below-the-fold images (never the LCP/above-the-fold one).
- `decoding="async"` to avoid blocking on decode.

**Consequence:** oversized/uncompressed images are usually the single biggest LCP and bandwidth cost.

---

## Fonts

- Self-host or `preconnect` to the font origin; `preload` the critical font file.
- `font-display: swap` (or `optional`) so text is visible during load.
- Subset to the glyphs/weights you use; ship `woff2`.

**Consequence:** unsubsetted, render-blocking fonts delay text paint and cause swap shifts (CLS).

---

## Network

- Cache static assets aggressively with content hashes (`Cache-Control: immutable, max-age=31536000`); never cache HTML the same way.
- `preconnect`/`dns-prefetch` to required third-party origins (API, CDN, fonts) early.
- Enable HTTP/2/3 and compression (Brotli/gzip).
- Defer/lazy-load non-critical third-party scripts (analytics, chat, ads) — they're frequent INP and main-thread offenders.

---

## How to measure

- **Lighthouse** (DevTools / `npx lighthouse <url> --view`) — lab scores for LCP/CLS/TBT plus diagnostics. Lab TBT approximates field INP.
- **WebPageTest** — waterfall, filmstrip, and repeat-view data; great for pinpointing render-blocking and CLS sources.
- **Chrome DevTools Performance panel** — record interactions to find long tasks and layout thrashing.
- **Field / RUM** — `web-vitals` library, Chrome UX Report (CrUX), or your analytics. **Field data is the source of truth**; lab numbers are a proxy. Always confirm a suspected regression against field/RUM where available.

---

## Quick scan checklist

- [ ] LCP element eager + `fetchpriority="high"` (or preloaded); not lazy.
- [ ] No needless render-blocking CSS/JS; non-critical scripts `defer`/`async`.
- [ ] Every image/video/iframe/ad has dimensions or `aspect-ratio` (CLS).
- [ ] Web fonts use `font-display: swap` and a metric-matched fallback.
- [ ] Injected content reserves space or overlays — never pushes content down.
- [ ] Search/filter/scroll/resize handlers debounced/throttled (passive scroll).
- [ ] No long tasks > 50ms on interaction; heavy work chunked/Worker/deferred.
- [ ] Long lists virtualized.
- [ ] Routes code-split; heavy optional features dynamically imported.
- [ ] No barrel-file imports defeating tree-shaking; per-function lib imports.
- [ ] Images: modern format, responsive `srcset`/`sizes`, below-fold `lazy`.
- [ ] Third-party scripts deferred; needed origins `preconnect`ed.
- [ ] Regression confirmed against field/RUM, not just one lab run.
