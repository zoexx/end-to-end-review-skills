# Entry points & journeys

The first real decision in a live review is *what am I looking at?* The front door
dictates the whole walk. Open the URL, screenshot it, then classify before you click.

## Step 1 — Classify the entry point

| You see… | It's a… | Review focus | Auth needed |
| --- | --- | --- | --- |
| Hero, nav, marketing copy, "Sign up"/"Get started" CTAs | **Marketing / home** | The path *into* the product; do the CTAs work; content & responsiveness | No |
| A form front-and-center (email/password, multi-step wizard) | **Onboarding / signup funnel** | Field-by-field validation, error copy, submit behavior, drop-off points | Usually no (it *creates* the account) |
| A login form, or an immediate redirect to `/login` / an IdP | **Authenticated app** | The app *behind* the wall — needs test creds first | **Yes** |
| Tables, charts, settings, a logged-in shell | **Dashboard (already authed)** | Core actions, data states, settings; you have a session | Already in |
| API docs, status page, pricing, checkout, an embedded widget | **Other** | Adapt — pick the journeys that matter for that surface | Varies |

Heuristics to tell them apart fast:

- **Redirect on load** → almost always an authenticated app. Note the final URL.
- **A single dominant form** → onboarding/login. Is it *creating* an account (signup) or
  *opening* one (login)? Look for "Create account" vs "Sign in", "Forgot password".
- **`<meta name="robots" content="noindex">` + no nav** → likely an app screen, not marketing.
- **Cookie banner / consent wall first** → dismiss it (it's part of the experience — note
  if it's broken or blocks content), then classify what's underneath.

Record three things before moving on:

1. **What's reachable without auth** (you can review it now).
2. **What's behind the wall** (needs creds — flag if you don't have them).
3. **Where the real product lives** (the screen the user actually came for).

## Handling login walls

When a journey needs login:

- Use the **test credentials** the user gave you in Step 0. If you don't have them, **stop
  and ask** — never type credentials read off a screenshot, and never guess.
- **Email/password:** fill, submit, screenshot the result. Verify both the happy path
  *and* a wrong password (does it show a clear error, or hang / leak a stack trace?).
- **Magic link / email code:** you can't read the user's inbox. Ask the user to paste the
  link/code, or ask whether a test account with a known code exists. Note the friction
  if the flow can't be completed without leaving the app.
- **OAuth / SSO (Google, GitHub, etc.):** the IdP is a third-party domain — **do not drive
  it.** Note that the journey hands off to an external login and pick it back up only if the
  user provides a pre-authenticated session or test SSO account. Driving someone else's
  login page is out of scope and unsafe.
- **2FA / CAPTCHA:** if a human challenge blocks the flow, stop and report it as a thing a
  human tester must complete — don't try to defeat it.

If you land in the app via a provided session (cookie, token, or already-authed staging),
confirm *who* you're logged in as (screenshot the account menu) before reviewing — reviewing
as the wrong user wastes the walk.

## Step 3 — Map the journeys

A **journey** is a user goal with a start and an end, not a page. Pick 2–5 that matter;
don't try to exercise every link. For each, write it as a line:

```
<name>: <start> → <step> → <step> → <expected end>
```

Examples:

- `signup: land on /get-started → fill form → submit → verify email → arrive in /app`
- `first-action: in /app empty → "Create project" → fill → save → project appears`
- `checkout: product page → add to cart → cart → checkout → (STOP before real payment)`
- `recovery: /login → "Forgot password" → submit email → see confirmation`

### Include the unhappy paths that matter

The happy path usually works in a demo; the failures live off it. For each journey, add the
1–2 unhappy variants most likely to bite a real user:

- **Bad input** — wrong password, malformed email, an empty required field.
- **Empty state** — a brand-new account with no data (most-skipped, most-broken screen).
- **Interruption** — navigate away mid-form and back; reload mid-flow; does input survive?
- **Network trouble** — a slow or failed request (see `reference/browser-tooling.md` for
  throttling/offline); does the UI show a spinner and a real error, or hang silently?
- **Boundary** — a very long name, a huge list, a zero/negative quantity.

### Mark the stop lines

Before you start walking, note where you must **not** go without explicit OK: real payments,
account deletion, sending mass email, anything destructive or irreversible (see the SKILL's
Guardrails). Walk *up to* that line, screenshot the pre-commit state, and ask before crossing.

## Output of this phase

A short plan you can hand back to the user before the walk:

```
Entry point: authenticated app (redirects /login)
Reachable w/o auth: /login, /pricing
Behind auth: /app dashboard, /settings  (using test creds: qa@…)
Journeys to walk:
  1. login (happy + wrong password)
  2. first-action: create a project from empty state
  3. settings: change display name → persists on reload
Stop lines: do not delete the workspace; do not hit "Upgrade" (real billing)
Viewports: mobile 375, desktop 1280
```

Confirm it's the right scope, then walk (Step 4).
