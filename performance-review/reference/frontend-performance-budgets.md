# Frontend performance budgets

Deep-dive for the *budget* lens on frontend performance: how many KB ship, how many round trips before first paint, and whether a CI gate stops the next regression. Each section pairs a ❌ anti-pattern with a ✅ fix and names the user-facing cost.

**Scope boundary:** the *mechanics* of Core Web Vitals — how LCP/CLS/INP actually work, the render-path fixes, layout-thrash, font-swap detail — live in **frontend-review** (`web-vitals-performance.md`). This guide is the budget-and-prioritization layer: it sets ceilings (bundle size, request count, Vitals targets), enforces them in CI, and ranks frontend cost against backend cost in one perf review. Route render-internals fixes to frontend-review; own the *budget* here.

The core idea: **every shipped KB is parse + compile + execute time on the user's CPU** — and the user's CPU is a mid-range Android, not your laptop. A budget makes that cost a number you can gate on.

---

## 1. Bundle budgets & code splitting

A bundle with no budget grows forever, one "small" dependency at a time, until the app is slow on the devices most of your users have.

❌ Everything in one bundle, every route's code shipped to load the homepage:
```ts
import Dashboard from './Dashboard';   // ❌ eagerly bundled even on routes that never use it
import Editor from './Editor';         // ❌ a heavy rich-text editor in the initial chunk
```
✅ Route-level code splitting — download only the current view:
```ts
const Dashboard = lazy(() => import('./Dashboard'));   // React.lazy / dynamic import
const Editor = lazy(() => import('./Editor'));         // loaded on demand
// Angular: loadComponent / loadChildren · Vue: defineAsyncComponent · SvelteKit: per-route by default
```

> Cost: a 1.2MB initial bundle is ~1.2MB to download, then *parse + compile + execute* on the main thread — easily 2–4s of CPU on a budget phone before the page is interactive. Splitting so the first route ships ~150KB cuts that to a fraction.

Set a budget (e.g. "initial JS ≤ 170KB gzipped") and enforce it (§8). The number is the point — "keep it small" isn't gateable.

---

## 2. Dynamic import of heavy/optional features

Code for a feature most users never trigger shouldn't be in the bundle everyone downloads.

❌ A heavy library in the initial bundle for a rarely-used feature:
```ts
import { Chart } from 'heavy-charting-lib';   // ❌ 200KB shipped to everyone for a chart 5% open
```
✅ Dynamic-import it when the feature is actually used:
```ts
async function openChart() {
  const { Chart } = await import('heavy-charting-lib');   // fetched on click, not on load
  renderChart(Chart);
}
```

> Cost: shipping a 200KB chart/editor/PDF/map library in the initial load taxes *every* visitor for a feature a small fraction reach. Lazy-load on interaction and the other 95% never pay for it.

---

## 3. Tree-shaking & barrel-file bloat

Imports that defeat tree-shaking quietly pull in code you never use.

❌ Whole-library import + barrel file:
```ts
import _ from 'lodash';                       // ❌ pulls the whole lib for one function
import { Button } from '@/components';         // ❌ barrel index re-exports everything → whole graph
```
✅ Import the exact thing; bypass barrels on hot paths:
```ts
import debounce from 'lodash/debounce';        // just the one function
import { Button } from '@/components/Button';   // direct path, tree-shakeable
```
- Ensure deps are ESM and side-effect-free (`"sideEffects": false` in `package.json`) so the bundler can drop unused exports.
- **Barrel files** (`index.ts` re-exporting a whole directory) often defeat tree-shaking and pull the entire module graph — import from the leaf module on perf-sensitive paths.

> Cost: `import _ from 'lodash'` adds ~70KB for a one-line helper; a barrel import can drag dozens of unused components into the chunk. Per-function/leaf imports ship only what's used.

---

## 4. Third-party script cost

Third-party scripts (analytics, chat, ads, tag managers) are often the biggest performance offenders, and they're easy to miss because they're not in *your* bundle.

❌ Render-blocking, eagerly-loaded third-party tags:
```html
<script src="https://tag-manager.example/loader.js"></script>   <!-- ❌ blocks parsing -->
<script src="https://chat-widget.example/embed.js"></script>     <!-- ❌ heavy, main-thread -->
```
✅ Defer, lazy-load on interaction/idle, and budget them:
```html
<script src="https://analytics.example/a.js" defer></script>     <!-- doesn't block -->
<!-- load chat only when the user is likely to use it -->
<script> requestIdleCallback(() => loadChatWidget()); </script>
```

> Cost: a single chat or tag-manager script can add hundreds of KB and a long main-thread task → worse INP and a delayed interactive page. They're frequent long-task offenders (frontend-review owns INP mechanics). Audit their real-world cost (WebPageTest "block 3rd party") and set a budget for them.

---

## 5. Image & font budgets

Images are usually the single biggest *bytes* on a page; fonts are a common render-blocker.

❌ Oversized originals and unbounded fonts:
```html
<img src="hero-4000px.jpg">                    <!-- ❌ 4000px file into a 400px slot, no format -->
<link href="...?family=Roboto:100,300,400,500,700,900&display=block" rel="stylesheet">  <!-- ❌ 6 weights, blocking -->
```
✅ Budget bytes: modern format, right-sized, subsetted, self-hosted:
```html
<img src="hero-800.avif" srcset="hero-400.avif 400w, hero-800.avif 800w"
     sizes="(max-width:600px) 100vw, 50vw" width="800" height="500" alt="…">
<!-- 1–2 weights, woff2, subset, font-display:swap (mechanics → frontend-review) -->
```
- **Images:** AVIF/WebP with fallback, responsive `srcset`/`sizes`, compress (don't ship 4000px into a 400px slot). Set an image-weight budget per page.
- **Fonts:** ship only the weights/styles used, `woff2`, subset to the glyphs needed, self-host or `preconnect`. Each extra weight is another file.

> Cost: oversized/uncompressed images are usually the #1 LCP and bandwidth cost; a 6-weight unsubsetted font family is hundreds of KB of render-blocking download. Both are pure budget — fixable without touching logic.

---

## 6. Request waterfalls & preloading

Latency before first paint is dominated by *serial* round trips. A waterfall is requests that could be parallel happening one after another.

❌ Sequential, discovered-late requests:
```js
const cfg = await fetch('/config').then(r => r.json());   // 1
const me  = await fetch('/me').then(r => r.json());        // 2 (doesn't need cfg) — serial
const data = await fetch(`/data?u=${me.id}`);              // 3 (needs me) — unavoidable here
// the browser also can't preconnect to the API/font origin until it parses the script that uses it
```
✅ Parallelize independent fetches; hint the browser early:
```js
const [cfg, me] = await Promise.all([                      // fire the independent two together
  fetch('/config').then(r => r.json()),
  fetch('/me').then(r => r.json()),
]);
```
```html
<link rel="preconnect" href="https://api.example.com">     <!-- warm the connection early -->
<link rel="preload" as="font" href="/inter.woff2" crossorigin>  <!-- discover the critical font now -->
```
- **Parallelize** independent client fetches (the frontend twin of backend-latency §1); fetch on the server / in the route loader where possible to avoid client-side waterfalls for above-the-fold data.
- **`preconnect`/`dns-prefetch`** to required third-party origins early; **`preload`** the critical, late-discovered resources (LCP image, key font).

> Cost: three serial 100ms fetches = 300ms of blank screen before content; parallelizing the independent ones and preconnecting cuts it toward 100ms. Waterfalls are latency you can delete without changing logic.

---

## 7. Over-fetching & over-rendering

Shipping data or DOM the user doesn't need is wasted bytes and wasted CPU.

❌ Fetch the world, render all of it:
```ts
const all = await fetch('/api/items').then(r => r.json());  // ❌ 10k items, every field
return all.map(renderRow);                                  // ❌ 10k DOM rows
```
✅ Fetch only what's shown; paginate/virtualize:
```ts
const page = await fetch('/api/items?limit=50&cursor=' + cursor).then(r => r.json());
// long lists: virtualize (windowing) — mechanics → frontend-review
```
- **Over-fetching:** request only the fields and rows rendered (GraphQL field selection, `?fields=`, pagination). Over-fetching costs network + serialization + client memory.
- **Over-rendering:** virtualize long lists; don't mount 10k nodes. (The INP/render mechanics are frontend-review's; the *budget* — "don't ship 10k rows" — is here.)

> Cost: fetching and rendering 10k rows when 50 are visible is megabytes of transfer and a huge DOM that's slow to render, scroll, and update — memory and interaction latency both suffer.

---

## 8. Enforcing budgets in CI

A budget that isn't gated is a suggestion. The frontend regresses one PR at a time unless the build fails when a ceiling is crossed.

✅ Gate the budget in CI:
```jsonc
// size-limit / bundlesize — fail the build if a chunk exceeds its budget
"size-limit": [{ "path": "dist/main.*.js", "limit": "170 KB" }]
```
```jsonc
// Lighthouse CI — assert Web Vitals against a budget (lhci autorun)
"assertions": {
  "categories:performance": ["error", { "minScore": 0.9 }],
  "largest-contentful-paint": ["error", { "maxNumericValue": 2500 }],
  "cumulative-layout-shift":  ["error", { "maxNumericValue": 0.1 }],
  "total-blocking-time":      ["error", { "maxNumericValue": 200 }]
}
```
- **Bundle size:** `size-limit`, `bundlesize`, or the framework's built-in budget (Angular `budgets`, Next bundle analyzer) failing the build over the ceiling.
- **Web Vitals:** **Lighthouse CI** with assertions on LCP/CLS/TBT (lab TBT proxies field INP). Confirm against field/RUM, not just one lab run (measuring-profiling §7).

> Cost: without a CI budget, "we'll keep an eye on the bundle" means a year of +5KB PRs compounding into a +250KB regression nobody can bisect. The gate makes the offending PR fail *its own* build.

---

## 9. Hydration & framework runtime cost

For SSR/isomorphic apps, hydration (attaching JS to server-rendered HTML) is a frequently-missed cost: the page *looks* ready but isn't interactive until hydration finishes.

❌ Ship and hydrate the entire page's JS up front (big bundle → long hydration → high INP/TBT).
✅ Reduce what hydrates: islands / partial hydration (Astro, Qwik resumability), `next/dynamic` with `ssr:false` for client-only widgets, React Server Components to keep non-interactive parts off the client bundle.

> Cost: a heavily-hydrated page is visually complete but unresponsive to clicks for seconds (the "looks loaded, feels broken" gap) — high TBT/INP. Shipping less client JS is the lever; the render-internals are frontend-review's.

---

## Quick scan checklist

- [ ] Initial **bundle** has a budget and stays within it; routes are code-split.
- [ ] Heavy/optional features (charts, editors, maps) are **dynamic-imported** on demand, not in the initial chunk.
- [ ] No whole-library or barrel-file imports defeating **tree-shaking**; per-function/leaf imports used.
- [ ] **Third-party scripts** are deferred/lazy-loaded and budgeted; their real cost was audited.
- [ ] **Images** modern-format + responsive + compressed; **fonts** subsetted, woff2, weights minimized.
- [ ] No client-side **waterfalls** — independent fetches parallelized; needed origins `preconnect`ed, critical resources `preload`ed.
- [ ] No **over-fetching** (only fields/rows shown) or **over-rendering** (long lists virtualized).
- [ ] A **CI budget** gate (size-limit / Lighthouse CI) fails the build on regression.
- [ ] **Hydration** cost considered for SSR apps; client JS minimized.
- [ ] Web Vitals tracked at a **budget** level; render-internals fixes routed to **frontend-review**.
