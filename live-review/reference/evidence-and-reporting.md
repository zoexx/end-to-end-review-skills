# Evidence & reporting

A live-review finding lives or dies on whether the author can **see it and reproduce it**.
Code review points at `file:line`; live review points at a **screenshot + steps**. This guide
covers what to capture and how to write a finding that gets fixed.

## Capture as you go, not after

You can't re-screenshot a transient state once you've navigated away. So capture *at the
moment*:

- **Before/after pairs** for any action — the screen before the click, the screen after. The
  delta *is* the evidence (button clicked → nothing changed → that's the bug).
- **The broken state itself**, full-page when the problem is below the fold (`full_page=True`
  in qa-master, or `| full` in a `qa_request` capture point).
- **The console/network read-out** at the moment of failure (the `window.__qa` JSON, the
  failed-request list) — paste the actual error text, not a paraphrase.
- **Each viewport** when the finding is responsive — the mobile shot and the desktop shot side
  by side make "collapses on mobile" undeniable.

Label shots so they map to findings: `signup-mobile-dead-button.png`,
`dashboard-empty-state.png`, `checkout-horizontal-scroll-375.png`. In `qa_request`, use
`[screenshot: NAME]` capture points so one autonomous run returns per-step shots already named.

## Anatomy of a live finding

Five parts. Miss the repro and it's noise; miss the WHY and it gets deprioritized.

```
<severity> — <journey> · <viewport> · <url>
<what's wrong, in one or two sentences> + the user impact (WHY).
Repro: 1. … 2. … 3. …   (exact, so anyone can follow it)
Screenshot: <label>     (and the console/network text if relevant)
Fix: <concrete change>. (point at file:line if the source is available — bonus, not required)
```

Worked example:

> 🔴 **blocking** — Checkout · mobile (375) · `/cart`
> The "Pay now" button sits behind the sticky footer and can't be tapped — the footer overlaps
> the bottom 60px of the page and the button is in that zone. A mobile user literally cannot
> complete a purchase.
> **Repro:** open `/cart` at 375×812 → add an item → scroll to the bottom → try to tap "Pay now".
> **Screenshot:** `checkout-paynow-under-footer-375.png`.
> **Fix:** add bottom padding equal to the footer height to the cart container, or make the
> footer non-overlapping on small screens. (Likely `CartFooter.css` `position: fixed`.)

## Write repro steps a stranger could follow

The author may not have your context. Make steps mechanical:

- Start from a **URL and a viewport**, not "the page."
- Name controls by their **visible label** ("tap *Create account*"), not a selector.
- Include the **input** you used ("enter `a@a` as the email").
- End at the **observed result** ("button shows a spinner that never resolves").

Bad: "the signup is broken on mobile."
Good: "open `/get-started` at 375px → fill all fields → tap *Create account* → nothing happens,
console logs `TypeError: submit is not a function`."

## Severity is about the user, not the effort

Realize the impact from the running app, then label (full rubric in `live-checks.md`):

- Can a user **not complete a core journey**, or does the page **break**? → 🔴 **blocking**.
- **Real friction** at a common width/path with a workaround? → 🟠 **important**.
- **Cosmetic / rare / harmless**? → 🟡 **nit**. An idea → 🔵. Done well → 🌟.

A subtle visual nit you had to hunt for is still a nit. A dead button you found in two seconds
is still blocking. Don't let discovery effort distort severity.

## Correlate with source — optional bonus

If the codebase is checked out locally, you *may* trace a live bug to its likely origin and add
`file:line` to the Fix — it saves the author a search. But the **screenshot is the primary
evidence**; the code pointer is a convenience. Don't block the report on finding the exact line,
and don't turn a live review into a code review — that's a different skill.

## The report

Assemble findings into the SKILL's output format: grouped by severity (all 🔴 first), each with
location · WHY · repro · screenshot · fix, then the verdict block. Two extras that make a live
report trustworthy:

- **State what you walked and what you didn't.** "Walked: marketing, signup, dashboard.
  Did *not* walk: billing (stop line), SSO login (third-party)." A reviewer needs to know the
  coverage to trust the verdict.
- **State the environment.** URL, staging vs prod, viewports, which browser driver, test
  account used. A live result is only meaningful with its conditions attached — a bug on prod
  with a real account is a different claim than one on staging with a seeded user.

> ### Verdict: 🔁 request changes
> Marketing and onboarding are fast and clean. Blocking on the mobile "Pay now" button (no
> phone purchase possible) and the console error on `/app`. One important empty-state gap.
> 🔴 2 · 🟠 1 · 🟡 3 · 🔵 1 · 🌟 2
> Walked: marketing · signup · checkout (to pre-payment) · dashboard
> Not walked: real payment (stop line), SSO (third-party)
> Env: staging.example.com · mobile 375 + desktop 1280 · qa-master · test acct qa@…
