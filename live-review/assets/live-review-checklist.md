# Live review checklist

For a hands-on walkthrough of a **running** site (a deployed/staging URL), not a code diff.
Copy into an issue/PR and tick what applies. Severity: 🔴 blocking · 🟠 important · 🟡 nit ·
🔵 suggestion · 📚 learning · 🌟 praise.

**Target:** `<url>`  ·  **Env:** staging / prod  ·  **Driver:** qa-master / Playwright / browser-harness
**Viewports:** mobile 375 · tablet 768 · desktop 1280  ·  **Test account:** `<…>`

## 0. Setup

- [ ] URL confirmed; authorized to test; staging preferred over prod.
- [ ] Scope & journeys agreed; stop lines noted (no real payment/delete/mass email).
- [ ] Test credentials in hand for any gated journey.
- [ ] Browser tool selected and working (open + screenshot + set viewport).

## 1. Entry point

- [ ] Classified: marketing / onboarding / authed app / dashboard / other.
- [ ] Reachable-without-auth vs behind-the-wall mapped.
- [ ] Login wall handled with test creds (or stopped & asked); OAuth/SSO not driven.

## 2. Behavior — does it work?

- [ ] Every primary CTA fires (nav / spinner / result — never silent nothing).
- [ ] Each journey can be completed start → expected end.
- [ ] Nav & footer links resolve (no 404 / dead anchors).
- [ ] State persists on reload; input survives the back button.
- [ ] Failures are visible (no spinner-that-never-stops; no silent 500).

## 3. Console & network

- [ ] No uncaught console errors / `unhandledrejection` during the walk.
- [ ] No failed requests (4xx/5xx) for JS, CSS, images, fonts, or APIs.
- [ ] No CORS / mixed-content / CSP violations (common with bad staging env vars).
- [ ] Warnings noted (usually 🟡, but flag — they sit near real bugs).

## 4. Responsive (each viewport)

- [ ] No horizontal scroll at 375 / 768 / 1280.
- [ ] Nothing overlaps, clips, or overflows; long strings/tables/numbers constrained.
- [ ] Layout stacks at sensible breakpoints (not too early, not too late).
- [ ] Tap targets reachable & ~44px on mobile; nothing hidden under fixed bars/cookie banner.
- [ ] Images crisp and correctly cropped per breakpoint.
- [ ] Sticky/fixed elements don't cover the content they act on.

## 5. States beyond happy path

- [ ] Empty state designed (copy + CTA, not a bare panel).
- [ ] Loading state shown (skeleton/spinner) — throttle to actually see it.
- [ ] Error state human & actionable (no raw stack trace / silent fail).
- [ ] "No results" designed.
- [ ] Long/overflow content paginates or virtualizes (doesn't freeze the tab).

## 6. Forms (drive adversarially)

- [ ] Submit empty → required fields caught with clear, field-level errors.
- [ ] Submit invalid → caught before a pointless round-trip; error near the field.
- [ ] Submit valid → actually succeeds and advances.
- [ ] Double/slow submit → button disables; no duplicate create / double-charge.
- [ ] Autofill / paste / password managers work.

## 7. Accessibility (live)

- [ ] Keyboard-only: everything reachable & operable; visible focus ring; logical order.
- [ ] Modals/menus trap focus, close on Esc, return focus to trigger.
- [ ] Focus not lost to `<body>` after async updates.
- [ ] Contrast adequate; meaning not conveyed by color alone.
- [ ] `prefers-reduced-motion` honored; content reflows at 200% zoom.

## 8. Real-world performance (measured)

- [ ] TTFB / DOMContentLoaded / LCP read from the page (number, not vibe).
- [ ] No visible layout shift (CLS) as images/fonts/ads settle.
- [ ] Interactions smooth (no jank / long main-thread tasks).
- [ ] No oversized images or unsplit bundles; third-party scripts don't delay interactivity.
- [ ] Each perf finding has a measured number + user-facing cost.

## Evidence

- [ ] Every finding has a screenshot + exact repro steps (URL, viewport, actions).
- [ ] Console/network error text pasted verbatim where relevant.
- [ ] Coverage stated: what was walked and what was skipped (and why).

---

### Verdict

✅ approve · 💬 approve with comments · 🔁 request changes · ⛔ block

Counts: 🔴 _ · 🟠 _ · 🟡 _ · 🔵 _ · 🌟 _
Walked: ____________  ·  Not walked: ____________
Env: ____________ · viewports ____________ · driver ____________
