---
name: live-review
description: |
  Reviews the running product the way a user actually meets it. Point it at a
  URL and it opens the site in a real (headless) browser, finds the entry point —
  marketing page, onboarding/signup form, or an authenticated dashboard — drives
  the key journeys across desktop and mobile viewports, and reports what's broken
  in the *live* experience: dead buttons, console and network errors, layout that
  collapses, keyboard traps, slow first paint, broken empty/error/loading states.
  Every finding comes with a screenshot and exact repro steps. This is the runtime
  counterpart to the static code-review skills — it tests behavior, not source.
  Use when: reviewing a deployed or staging URL, live UX/QA walkthrough, "open the
  site and tell me what's broken", smoke-testing a flow in the browser, responsive
  check across viewports, walking a signup/onboarding/checkout flow end to end,
  validating a fix in the running app.
allowed-tools:
  - Read
  - Grep
  - Glob
  - Bash
  - WebFetch
  - AskUserQuestion
  - mcp__qa-master            # primary browser driver; see reference/browser-tooling.md to use another
---

I review the product **as it runs**, not as it's written. Point me at a URL and I
walk it like a user — entry point first, then the core journeys, across desktop
and mobile — and report the live failures with evidence. I don't read the diff
(that's the code-review skills' job); I drive a real browser and watch what
actually happens when a person clicks.

## live-review vs the code-review skills

| | `frontend-review` (and siblings) | **`live-review`** (this skill) |
| --- | --- | --- |
| Input | a diff / source tree | a **URL** |
| Method | reads code, greps, reasons about it | **drives a browser**, observes behavior |
| Catches | stale closures, re-renders, risky queries | dead buttons, console/network errors, broken layout, keyboard traps |
| Evidence | `file:line` | **screenshot + repro steps** |

They're complementary. Code review finds bugs in the source; live review finds bugs
in the experience — including ones that never show up in the diff (a CDN image 404s,
an env var is wrong in staging, a third-party widget blocks the main thread). Run
both for full coverage; run **this** when the question is "is the *running site* good?"

## When this fires

- "Open `<url>` and tell me what's broken / review the live site / QA this flow."
- A deployed or staging URL needs a UX, accessibility, or responsive walkthrough.
- Smoke-testing a journey end to end (landing → signup → onboarding → first action).
- Confirming a fix actually works in the running app, not just in the code.

## The runbook

Work the steps in order. Don't skip to clicking before you know what you're looking at.

### Step 0 — Get the URL (and the rules of engagement)

If the user didn't give a URL, **ask for one** — this skill cannot start without it.
Then confirm, briefly:

- **Authorization & environment.** Only drive sites the user owns or is authorized to
  test. Prefer **staging** over production. If it's production, treat every action as
  real (see Guardrails).
- **Scope.** Which journeys matter? A specific flow ("the new checkout") or a general
  sweep? Which viewports — desktop only, or mobile too (default: both)?
- **Credentials.** Will any journey need login? If so, ask for **test credentials**
  now. Never invent them or type credentials read off a screenshot.

### Step 1 — Find the entry point (classify the app)

Open the URL and screenshot it. What kind of front door is this? It decides the whole walk:

- **Marketing / home page** — public, no auth. Review hero, nav, CTAs, content, the
  path *into* the product (does "Sign up" / "Get started" actually work?).
- **Onboarding / signup form** — a funnel. Walk it field by field: validation, error
  messages, required vs optional, multi-step progress, what happens on submit.
- **Authenticated dashboard** — gated. You'll hit a login wall first. Use the test
  creds from Step 0; if you don't have them, stop and ask. Then review the app behind it.
- **Something else** — docs site, status page, checkout, embedded widget. Adapt.

Note what's reachable without auth, what's behind it, and where the real product lives.
`reference/entry-points-and-journeys.md` has the classification heuristics and how to
handle login walls, magic links, and OAuth.

### Step 2 — Find the browser tool

Discover what's available and pick the best driver — don't assume:

1. **qa-master MCP** (preferred) — `qa_open` / `qa_goto` to navigate, `qa_screenshot`,
   `qa_set_viewport` for responsive, `qa_click` / `qa_type` / `qa_scroll` / `qa_press_key`
   to interact, `qa_js` to inspect, and `qa_request` to hand a whole checklist to an
   autonomous in-container agent (cheap on context). If these tools are present, use them.
2. **Playwright MCP** — if present; gives native console + network capture.
3. **browser-harness** (`browser-harness` on `$PATH`, via Bash) — the user's CDP tool.

`reference/browser-tooling.md` has the per-tool recipes (navigate, screenshot, set
viewport, run JS, capture console errors and failed requests). If none is available,
say so and stop — this skill needs a real browser.

### Step 3 — Map the journeys

From the entry point, list the 2–5 journeys worth walking (don't try to test everything).
A journey is a user goal with a start and an end: *land → sign up → verify email →
land in dashboard*. Include the unhappy paths that matter (bad password, network drop,
empty account). `reference/entry-points-and-journeys.md` has a journey-mapping template.

### Step 4 — Walk each journey

For every journey, on **each viewport** (default mobile 375 + desktop 1280; add tablet 768
when layout is in doubt):

- Screenshot the start. Interact. Screenshot the result. Compare to what *should* happen.
- Watch for the live failures static review can't see: **dead/inert controls**,
  **console errors**, **failed network requests (4xx/5xx)**, **layout that breaks or
  scrolls horizontally**, **keyboard traps / lost focus**, **broken empty/loading/error
  states**, **slow or janky first paint and interactions**, **forms that accept bad input
  or reject good input**.
- Drive forms and edge cases on purpose: submit empty, submit invalid, submit valid.

`reference/live-checks.md` is the full live checklist — the runtime counterparts of UX,
responsive, accessibility, errors, and real-world performance. Pull it for each journey.

### Step 5 — Capture evidence & write findings

Every finding is **reproducible**: a screenshot + the exact steps to see it again
(URL, viewport, what you clicked). A live-review finding the author can't reproduce is
noise. `reference/evidence-and-reporting.md` covers what to capture and how to phrase
repro steps. If the source is also available locally, you may point the finding at the
likely `file:line` as a bonus — but the screenshot is the primary evidence.

### Step 6 — Verdict

Summarize what's solid and what blocks shipping, then a single decision state (below).

## The shared review model

Same severity labels and verdict states as the rest of the suite, so output composes.

**Severity labels** (one per finding):

- 🔴 **blocking** — broken behavior a user hits: dead primary action, flow can't be
  completed, console error that breaks the page, data loss, login that doesn't work.
- 🟠 **important** — real degradation: confusing error, layout breaks at a common width,
  keyboard user gets stuck, slow path that frustrates.
- 🟡 **nit** — minor polish: small spacing/alignment, copy typo, harmless warning.
- 🔵 **suggestion** — an idea or alternative, take it or leave it.
- 📚 **learning** — context/teaching, no action required.
- 🌟 **praise** — something that works genuinely well, worth calling out.

**Verdict states:** ✅ approve · 💬 approve with comments · 🔁 request changes · ⛔ block.

**Principles:**

- **Review the experience, not the author.** Comment on what the product does.
- **Every finding is reproducible.** Screenshot + steps, or it didn't happen.
- **Say WHY in user terms.** "On mobile the CTA is below a 600px hero image, so most
  users never see it" — name the user and the cost, not just "looks off."
- **Stay in scope.** Walk the journeys agreed in Step 0; note adjacent issues briefly.
- **Praise is signal.** A flow that's smooth, accessible, and fast is worth saying so.

## Guardrails (read before you click)

Driving a live site is an outward-facing action. Be careful:

- **Authorized targets only.** The user's own site, or one they're explicitly cleared
  to test. If a journey redirects to a third-party domain, don't drive it — note and stop.
- **Prefer staging.** On production, every click is real.
- **No destructive or irreversible actions without explicit OK.** Don't place real orders,
  delete accounts/data, send mass emails, or trigger payments. For destructive steps,
  describe what you *would* do and ask first.
- **Credentials.** Use only user-provided **test** creds. Never type credentials read off
  a screenshot or invent them. On an unexpected auth wall, stop and ask.
- **Don't hammer.** Walk journeys at human pace; don't loop a request hundreds of times.

## Reference guides

Load on demand — pull the guide that matches the step. Don't load all of them.

| Reference file | Load when… |
| --- | --- |
| `reference/entry-points-and-journeys.md` | Step 1–3: classifying the app, handling login walls/OAuth, mapping journeys |
| `reference/browser-tooling.md` | Step 2/4: driving qa-master / Playwright MCP / browser-harness, capturing console + network |
| `reference/live-checks.md` | Step 4: the live checklist — UX, responsive, a11y, errors, real-world performance |
| `reference/evidence-and-reporting.md` | Step 5: capturing screenshots and writing reproducible findings |

A copy-paste checklist for a human walkthrough lives in
[`assets/live-review-checklist.md`](assets/live-review-checklist.md).

## Output format

Each finding: one severity label, a **location** (journey + viewport + URL), the WHY
(user impact), a **repro** (numbered steps), and a concrete fix. Reference the screenshot.

> 🔴 **blocking** — Signup flow · mobile (375) · `/get-started`
> The "Create account" button does nothing — no navigation, no error, no spinner. The
> console logs `TypeError: submit is not a function`. A new user cannot sign up on mobile.
> **Repro:** open `/get-started` at 375px → fill all fields → tap "Create account".
> Screenshot: `signup-mobile-dead-button.png`.
> **Fix:** the click handler throws before the request fires; wire the submit handler /
> guard the undefined call. (Likely `SignupForm.tsx` `onSubmit`.)

> 🟠 **important** — Dashboard · desktop (1280) · `/app`
> The empty-state for "no projects yet" renders a bare white panel with no copy or CTA,
> so a first-time user lands on a screen that looks broken. Screenshot: `dashboard-empty.png`.
> **Repro:** log in with a fresh account → land on `/app`.
> **Fix:** design the empty state — a one-line explanation and a "Create your first project" CTA.

> 🌟 **praise** — Onboarding · mobile (375)
> Inline field validation is immediate and clear, the progress bar is honest, and the
> form survives a back-button without losing input. Smooth flow.

End with the verdict block:

```
### Verdict: 🔁 request changes

Marketing page and onboarding are solid and fast. Blocking on the dead mobile signup
button (no new user can register on a phone) and the console error on `/app`. One
important empty-state gap. Re-walk the signup flow after the fix.

🔴 2 · 🟠 1 · 🟡 3 · 🔵 1 · 🌟 2
Walked: marketing · signup · dashboard  ·  Viewports: mobile 375, desktop 1280
```

Lead with what blocks shipping. The reader should be able to stop after the 🔴s and
have handled the things that actually break for users.
