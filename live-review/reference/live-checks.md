# Live checks

The runtime checklist. These are the failures you can only see by *running* the product —
the live counterparts of what the code-review skills reason about statically. Pull this for
each journey in Step 4. You won't run every check on every screen; pick what the screen invites.

Each check below names **what to do**, **what's broken**, and **why it matters to a user**.

---

## 1. Does it actually work? (behavior)

The first job of a live review: confirm the thing does the thing.

- **Primary actions fire.** Click every primary CTA and submit. A button that does nothing
  — no nav, no spinner, no error — is the #1 live bug and invisible to code review if the
  handler silently throws. *Why: the user is stuck with no feedback.*
- **Journeys complete.** You can get from start to expected end (Step 3). If a flow can't be
  finished, that's blocking regardless of how nice the screens look.
- **Links go somewhere real.** Spot-check nav and footer links for 404s / dead anchors.
- **State persists where it should.** Reload after an action — did the change stick? Hit the
  back button mid-form — is input preserved or wiped? *Why: lost work is rage-quit material.*
- **No silent failures.** An action that fails should *say so*. A request that 500s behind a
  spinner-that-never-stops is worse than an honest error.

## 2. Console & network (the errors users don't see but feel)

Install the collectors from `reference/browser-tooling.md`, then walk the journey.

- **Console errors.** Uncaught `TypeError`s, failed dynamic imports, `unhandledrejection`.
  Even when the UI *looks* fine, a console error often means a broken feature one click away.
- **Failed requests (4xx/5xx).** A 404 on a JS chunk, image, or font; a 500 from an API. Map
  each failure to what the user loses (a missing avatar vs a broken save are different severities).
- **CORS / mixed content / CSP violations.** Common in staging with wrong env vars — a request
  blocked by CORS means a feature that works locally is dead in the deployed environment.
- **Noisy warnings.** Deprecations and React key warnings are usually 🟡, but flag them — they
  often sit next to a real bug.

## 3. Responsive layout (across viewports)

Run each screen at **mobile 375**, **desktop 1280**, and **tablet 768** when in doubt.

- **No horizontal scroll** at any width — the classic mobile bug. A 400px-wide element on a
  375px screen makes the whole page slide. *Why: feels broken on every phone.*
- **Nothing overlaps, clips, or overflows its container.** Long names, long emails, big numbers.
- **Layout doesn't collapse too early or too late.** A two-column layout that stacks at 1100px
  wastes a desktop; a table that stays 4-wide at 375px is unusable.
- **Tap targets are reachable and big enough** (~44px) on mobile; nothing hides under a fixed
  header/footer or a cookie bar.
- **Images aren't stretched or pixelated**; the right crop/source loads per breakpoint.
- **Sticky/fixed elements behave** — a sticky CTA that covers the form it submits is a trap.

## 4. The states beyond happy path

The screens demos skip and users hit first.

- **Empty state.** Fresh account, no data. Is there copy and a CTA, or a bare broken-looking
  panel? *Why: a first-time user's first screen is usually the empty one.*
- **Loading state.** Is there a skeleton/spinner, or does the screen sit blank then pop? Throttle
  the network (where the driver allows) to actually see it.
- **Error state.** Force a failure (bad input, dropped request). Is the message human and
  actionable, or a raw stack trace / silent nothing?
- **"No results."** Search/filter for something that doesn't exist — designed, or a blank void?
- **Long / overflow content.** A 200-item list, a 5000-char note — does it paginate/virtualize
  or freeze the tab?

## 5. Forms (drive them adversarially)

For every form in a journey:

- **Submit empty** → are required fields caught with clear, associated errors?
- **Submit invalid** → bad email, mismatched passwords, out-of-range number — caught *before*
  a pointless round-trip? Is the error next to the field, not just a top banner?
- **Submit valid** → does it actually succeed and move on?
- **Double-submit / slow submit** → is the button disabled after click, or can you fire it
  twice and create duplicates? *Why: double-charges, duplicate signups.*
- **Inline vs on-submit validation** → does it nag on every keystroke, or validate sensibly?
- **Autofill / paste / password managers** work; input isn't blocked by aggressive masking.

## 6. Accessibility, live

Code review checks the markup; here you check the *interaction*.

- **Keyboard-only walk.** Tab through the journey. Can you reach and operate everything? Is the
  focus ring visible? Is the tab order logical? *Why: keyboard and screen-reader users.*
- **Focus management.** Open a modal/menu — does focus move into it, trap there, and return to
  the trigger on close (Esc)? A modal you can't escape with a keyboard is a trap.
- **No focus loss.** After an async update, focus shouldn't jump to `<body>` and strand the user.
- **Contrast & "color-only" meaning.** Spot low-contrast text and states conveyed by color alone
  (a red border with no text). Confirm with the page's computed styles via JS if unsure.
- **Motion.** With `prefers-reduced-motion`, are big animations actually reduced?
- **Zoom.** At 200% browser zoom, does content reflow or get cut off?

## 7. Real-world performance (measured, not guessed)

Numbers from the running page beat lab guesses. Pull them with JS (`reference/browser-tooling.md`).

- **First paint feels slow?** Read it: `performance.getEntriesByType('navigation')[0]` for TTFB
  and DOMContentLoaded; the LCP via a `PerformanceObserver`. Name the number, not a vibe.
- **Layout shift (CLS) you can see** — content jumping as images/fonts/ads load. Screenshot
  mid-load vs settled.
- **Janky interactions** — a click/scroll that stutters. Long tasks blocking the main thread,
  often a heavy third-party script.
- **Heavy payloads** — sum `resource` transfer sizes; flag a multi-MB image or an unsplit bundle.
- **Third-party drag** — analytics/chat/ad scripts that delay interactivity. *Why: the user's
  first impression is "this is slow," even if your code is fine.*

Always tie a perf finding to a **measured number and a user-facing cost** ("LCP 4.1s on mobile —
the hero is a 2.3MB unoptimized PNG; users stare at blank space for 4s"). No number, no finding.

---

## Triage reminder

Map each observation to severity by **user impact**, not how easy it was to spot:

- Flow can't complete / dead primary action / page-breaking error → 🔴 **blocking**
- Real friction or breakage at a common width/path, but a workaround exists → 🟠 **important**
- Cosmetic, rare, or harmless → 🟡 **nit**
- A better way → 🔵 **suggestion**; a thing done well → 🌟 **praise**
