# Accessibility (a11y) review reference

Deep checklist tied to **WCAG 2.2 AA**. Each item pairs a ❌ bad snippet with a
✅ fix and the user-facing consequence. Guiding rule: **no ARIA is better than
bad ARIA** — reach for a native element first.

---

## Semantic structure & landmarks

❌ `<div>` soup:
```html
<div class="nav">…</div>
<div class="main">…</div>
<div class="btn" onclick="save()">Save</div>
```
✅
```html
<nav>…</nav>
<main>…</main>
<button type="button" onclick="save()">Save</button>
```
**Consequence:** screen-reader users navigate by landmarks (`<nav>`, `<main>`, `<header>`, `<footer>`, `<aside>`) and rely on `<button>`/`<a>` for built-in keyboard, focus, and role behavior. A clickable `<div>` is invisible to assistive tech and isn't keyboard-operable. (WCAG 1.3.1, 4.1.2)

---

## Heading order

❌
```html
<h1>Dashboard</h1>
<h4>Recent orders</h4> <!-- skipped h2/h3 -->
```
✅ One `<h1>` per page/view, then descend without skipping levels (`h2`, `h3`…).
**Consequence:** screen-reader users jump between sections by heading level; skipped or misused levels (using headings for visual size) break the document outline so they can't tell how content is organized. (WCAG 1.3.1, 2.4.6)

---

## Accessible names & labels

❌ Icon-only button and an unlabeled input:
```html
<button><svg>…</svg></button>
<input type="email">
```
✅
```html
<button aria-label="Close dialog"><svg aria-hidden="true">…</svg></button>
<label for="email">Email</label>
<input id="email" type="email" autocomplete="email">
```
**Consequence:** without an accessible name, a screen reader announces "button" or "edit text" with no purpose — the control is unusable. Prefer a visible `<label>`; use `aria-label`/`aria-labelledby` only when no visible text exists. Placeholders are **not** labels (they vanish on input). (WCAG 1.3.1, 2.4.6, 3.3.2, 4.1.2)

---

## Focus management

### Trap focus in modals; return it on close

❌ Opening a dialog while focus stays on the page behind it; Tab leaks to the background.
✅ Use a native `<dialog>` or implement: move focus into the dialog on open, keep Tab cycling inside it, restore focus to the trigger on close, and close on Esc:
```html
<dialog ref={ref} aria-labelledby="title">
  <h2 id="title">Confirm</h2> …
</dialog>
```
**Consequence:** keyboard and screen-reader users get "lost behind" the modal — they Tab into hidden background content and can't operate the dialog. (WCAG 2.4.3, 2.1.2 no keyboard trap on the *page*)

### Don't kill the focus outline

❌
```css
*:focus { outline: none; }
```
✅
```css
:focus-visible { outline: 2px solid; outline-offset: 2px; }
```
**Consequence:** removing the outline leaves keyboard users with no idea where they are on the page. Use `:focus-visible` so the ring shows for keyboard but not mouse clicks. (WCAG 2.4.7 Focus Visible, and 2.4.11 Focus Not Obscured in 2.2)

### Manage focus on route changes / dynamic content

✅ On client-side navigation, move focus to the new view's heading (or a skip target) and update the document title.
**Consequence:** in SPAs the URL changes but focus stays put, so screen-reader users don't know the page changed.

---

## Keyboard operability

- Everything actionable must be reachable and operable by keyboard alone — Tab to reach, Enter/Space to activate (Space for buttons, Enter for links).
- Tab order follows visual/reading order; avoid positive `tabindex` (use `tabindex="0"`/`-1"` only).
- Provide a **skip link** to bypass repeated nav:
  ```html
  <a class="skip-link" href="#main">Skip to content</a>
  ```
- Esc closes overlays; Arrow keys move within composite widgets (menus, tabs, listboxes) per the ARIA Authoring Practices.

❌ `onClick` on a `<div>` with no `tabindex`/`role`/key handler.
✅ Use a `<button>`, or if truly custom: `role="button" tabindex="0"` plus Enter/Space handlers — but prefer the native element.
**Consequence:** mouse-only widgets exclude keyboard and switch users entirely. (WCAG 2.1.1 Keyboard, 2.4.1, 2.4.3)

---

## ARIA roles & states — used correctly or not at all

❌ Redundant/conflicting ARIA:
```html
<button role="button" aria-label="Save">Save</button> <!-- role redundant; label duplicates text -->
<div role="button" aria-pressed="true"> <!-- reinventing a toggle -->
```
✅ Native element, ARIA only to express state native HTML can't:
```html
<button aria-expanded="true" aria-controls="menu">Menu</button>
```
**Consequence:** wrong roles, fake roles, or stale `aria-*` states actively mislead screen-reader users — worse than no ARIA. Keep `aria-expanded`, `aria-selected`, `aria-checked`, `aria-current`, `aria-disabled` in sync with actual state. Hide decorative icons with `aria-hidden="true"`. (WCAG 4.1.2)

---

## Color & contrast

- Body text ≥ **4.5:1** against its background; large text (≥24px, or 18.66px bold) ≥ **3:1**. (WCAG 1.4.3)
- UI components and graphical objects (icons, input borders, focus rings) ≥ **3:1**. (WCAG 1.4.11)
- Don't convey meaning by color alone — pair color with text/icon (e.g. error state has an icon + message, not just red). (WCAG 1.4.1)

❌ Light-gray placeholder text as the only label; red border as the only error signal.
✅ Sufficient-contrast persistent label; error border **plus** an associated message.

---

## Motion

❌ Auto-playing parallax/animation with no opt-out.
✅
```css
@media (prefers-reduced-motion: reduce) {
  *, *::before, *::after { animation-duration: 0.01ms !important; transition-duration: 0.01ms !important; }
}
```
**Consequence:** large or vestibular-triggering motion can cause nausea/dizziness. Honor `prefers-reduced-motion`; avoid content that flashes more than 3×/sec. (WCAG 2.3.3, 2.3.1)

---

## Forms: errors announced & associated

❌
```html
<input type="text"><span class="error">Required</span> <!-- not linked -->
```
✅
```html
<label for="name">Name</label>
<input id="name" aria-describedby="name-err" aria-invalid="true">
<span id="name-err" role="alert">Name is required</span>
```
**Consequence:** an error not programmatically tied to its field (`aria-describedby`) and not announced (`role="alert"`/live region) is invisible to screen-reader users — they can't tell what failed or why. Show errors in text, near the field, after the relevant control. (WCAG 3.3.1, 3.3.3, 1.3.1)

---

## Live regions

✅ Announce async updates (toasts, "3 results found", cart count) with a polite live region:
```html
<div aria-live="polite" aria-atomic="true">{{ statusMessage }}</div>
```
**Consequence:** without a live region, dynamic changes (search results, save confirmations) happen silently — screen-reader users never learn the action worked. Use `assertive` only for urgent interruptions.

---

## Images & media

- Informative images need descriptive `alt`; decorative images use `alt=""` (empty, not missing) so they're skipped.
- Complex images (charts) need a longer text alternative nearby.
- Video needs captions; audio needs transcripts. (WCAG 1.1.1, 1.2.2)

❌ `<img src="chart.png">` (no alt) · `<img src="divider.png" alt="decorative divider">` (noise).
✅ `<img src="chart.png" alt="Revenue rose 20% from Q1 to Q2">` · `<img src="divider.png" alt="">`.

---

## Target size (WCAG 2.2)

✅ Interactive targets at least **24×24 CSS px** (2.5.8 AA), with spacing so adjacent targets don't overlap; 44×44 is the comfortable mobile baseline.
**Consequence:** tiny tap targets cause mis-taps for motor-impaired and touch users.

---

## SEO crossover

- Meaningful `<title>` and meta description per page/route; one `<h1>`.
- Semantic structure and `alt` text double as SEO signals.
- Add structured data (JSON-LD) for rich results where relevant (products, articles, breadcrumbs).

---

## Quick scan checklist

- [ ] Native elements for buttons/links/inputs; landmarks (`<nav>/<main>/...`) present.
- [ ] One `<h1>`, no skipped heading levels.
- [ ] Every control has an accessible name; placeholders aren't the only label.
- [ ] Modals trap focus, close on Esc, and return focus to the trigger.
- [ ] Visible `:focus-visible` ring; no blanket `outline: none`.
- [ ] Fully keyboard operable; skip link present; no positive `tabindex`.
- [ ] ARIA only where native won't do, and states kept in sync; decorative icons `aria-hidden`.
- [ ] Text contrast ≥ 4.5:1, UI/graphics ≥ 3:1; meaning never color-only.
- [ ] `prefers-reduced-motion` honored.
- [ ] Form errors associated (`aria-describedby`) and announced (`role="alert"`).
- [ ] Live region for async status updates.
- [ ] Meaningful `alt`; decorative images `alt=""`.
- [ ] Touch targets ≥ 24×24 px.
