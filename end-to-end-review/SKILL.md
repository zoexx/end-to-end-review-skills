---
name: end-to-end-review
description: |
  Orchestrates a full-stack code review by routing a change to the right layer-specific
  review skills (frontend, backend, database, security), running each, then merging the
  results into one prioritized report with a single overall verdict. De-duplicates findings
  that surface at multiple layers and surfaces cross-layer issues a single-layer review misses.
  Use when: reviewing a whole PR/branch end to end, full-stack review, review everything,
  reviewing a change that spans UI + API + database, "do a complete review", pre-merge review,
  reviewing a feature that touches multiple layers.
allowed-tools:
  - Read
  - Grep
  - Glob
  - Bash
  - WebFetch
  - Task
---

This skill is the conductor. It doesn't re-implement layer expertise — it decides **which** layers a change touches, runs each layer's review with that layer's depth, and folds everything into one report an author can act on top-down.

## When this fires

Use it for any review that spans more than one layer: a feature PR, a release branch, "review this whole thing." For a change that's obviously single-layer (a CSS tweak, one migration), invoke that layer's skill directly instead — this orchestrator adds overhead only worth paying when the diff crosses boundaries.

It coordinates four sibling skills (install whichever you have; the orchestrator engages the ones present):

| Skill | Engaged when the diff touches… |
| --- | --- |
| **frontend-review** | `*.{tsx,jsx,ts,js,vue,svelte,astro}` in `components/`, `pages/`, `app/`, `src/`, `*.{css,scss,less,html}`, design/markup, client state |
| **backend-review** | `routes/`, `api/`, `controllers/`, `services/`, `handlers/`, `middleware/`, `*.go`, server-side `*.py`/`*.rb`/`*.java`/`*.ts`, API contracts, business logic |
| **database-review** | `migrations/`, `*.sql`, `schema.prisma`, `models/`, `entities/`, `repositories/`, ORM query code, anything that changes the data model or a query |
| **security-review** | auth/session/crypto code, input handling, file uploads, `package.json`/lockfiles, `.env`/config, anything reachable by untrusted input — **plus a light pass on every review** |

> Security gets a light pass on **every** review even if no obviously-security file changed: a new endpoint, a new query, or a new form field all open attack surface.

## The review model

Every sibling skill — and this orchestrator — shares one model so output composes:

**Four phases.** (1) **Context** — read the PR description, the diff, the linked issue, the blast radius. (2) **High-level** — does the change have the right shape *across* layers? (3) **Detailed** — each engaged layer runs its line-by-line checks. (4) **Verdict** — one merged summary + decision.

**Severity labels.** 🔴 **blocking** · 🟠 **important** · 🟡 **nit** · 🔵 **suggestion** · 📚 **learning** · 🌟 **praise**

**Verdict states.** ✅ approve · 💬 approve with comments · 🔁 request changes · ⛔ block

**Principles.** Review the code, not the coder. Every blocking/important finding names the concrete impact. Stay in scope. Praise is signal.

## How to run it

### 1. Scope the diff

```bash
git diff --name-only main...HEAD      # or origin/main...HEAD, or the PR's base
git diff --stat main...HEAD
```

Map the changed files to layers using the routing table above. Note files that hit **multiple** layers (e.g. a Prisma model used by an API handler rendered in a React table) — those are the cross-layer hotspots.

### 2. Run each engaged layer

Two modes — pick by size:

- **Inline (small/medium diff):** load each engaged skill's `SKILL.md`, then work its phases over the relevant files, pulling in that skill's `reference/*.md` as needed.
- **Fan-out (large diff, recommended):** spawn one subagent per engaged layer with the `Task` tool, each running its layer's skill over its slice of the diff and returning findings in the shared format. Run them in parallel; collect when all return. This keeps each layer's deep context out of the others' way.

Each layer returns findings as: `<severity> [layer] file:line — finding. Why: <impact>. Fix: <concrete change>.`

### 3. Merge

- **De-duplicate.** The same root cause often shows up at two layers — money stored as `float` is flagged by database (wrong type), backend (rounding errors), and frontend (display drift). Keep **one** finding, attribute it to the layer that owns the fix, and note the other lenses ("also surfaces as a display bug in the UI").
- **Promote cross-layer findings.** If a change is only wrong *because* of how layers interact (an API returns an unbounded list → DB does a full scan → the frontend renders 10k rows and janks), call it out as a single **cross-layer** finding — no individual layer fully owns it.
- **Sort.** Group by severity first (all 🔴 together), then by layer within a severity.

### 4. Decide

The overall verdict is the **worst** layer verdict — any 🔴 anywhere makes the whole review ⛔ block or 🔁 request changes. A clean full-stack pass is ✅. Don't average; a perfect frontend doesn't offset a SQL injection.

## Output format

Use the template in [`assets/end-to-end-review-report-template.md`](assets/end-to-end-review-report-template.md). Shape:

```markdown
# End-to-end review — <PR title / branch>

**Verdict:** 🔁 request changes
**Layers engaged:** frontend · backend · database · security
**Summary:** <2-3 sentences: what the change does and the headline risks.>

## 🔴 Blocking
- 🔴 [security] `api/orders/[id].ts:42` — Order fetched by `id` with no ownership check (IDOR).
  Why: any logged-in user can read any order by guessing the id — data exposure.
  Fix: scope the query to `where: { id, userId: session.userId }` and 404 on miss.

## 🟠 Important
- 🟠 [cross-layer] `total` stored and sent as a JS `number`.
  Why: float rounding corrupts money end to end (DB → API → UI).
  Fix: store `numeric`, transport as integer cents or string, format in the UI.

## 🟡 Nits / 🔵 Suggestions
- ...

## 🌟 Praise
- 🌟 Clean error envelope and consistent status codes across the new endpoints.

## By layer
| Layer | Verdict | 🔴 | 🟠 | 🟡 | 🔵 |
| --- | --- | --- | --- | --- | --- |
| frontend | 💬 | 0 | 1 | 3 | 2 |
| backend  | 🔁 | 0 | 2 | 1 | 0 |
| database | 💬 | 0 | 1 | 0 | 1 |
| security | ⛔ | 1 | 0 | 0 | 0 |
```

Lead with what blocks the merge. An author should be able to read top-to-bottom and stop when they run out of time, having handled the most important things first.

## Reference guides

This skill carries no `reference/` of its own — depth lives in each sibling skill's `reference/` directory, loaded on demand by that layer's review. The orchestrator's job is routing, merging, and the single verdict.
