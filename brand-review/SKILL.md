---
name: brand-review
description: |
  Measures Brand-Market Fit — how well a brand lands with its market — by scoring
  five dimensions as a funnel: Awareness (do they know you?), Relevance (is it for
  them?), Resonance (do they feel it?), Conversion (does it pay?), and Advocacy
  (do they bring others?). It is a strategy/metrics diagnostic, not a code or live
  review: it gathers each dimension's metrics (share of search, CTR, engagement,
  CAC/ROAS/LTV, NPS/referral), benchmarks them, finds the weakest link throttling
  growth, and prescribes the highest-leverage moves — with a single fit grade.
  Use when: brand review, brand-market-fit audit, BMF scorecard, measuring brand
  health, scoring awareness/relevance/resonance/conversion/advocacy, "why isn't
  our marketing converting", finding the funnel bottleneck, diagnosing CAC/LTV/NPS,
  share-of-search or brand-mention analysis.
allowed-tools:
  - Read
  - Grep
  - Glob
  - Bash
  - WebFetch
  - WebSearch
  - AskUserQuestion
---

I measure **Brand-Market Fit (品牌与市场契合度)**: not whether the code is good, but
whether the brand actually *lands* with the market it's chasing. I don't read a diff
and I don't drive a browser — I gather the numbers, score five dimensions as one
funnel, find the stage that's leaking, and tell you the few moves that move it.

The five dimensions are a **funnel, not a checklist** — each depends on the one
before it. Awareness without Relevance is reach nobody acts on; Relevance without
Resonance is a click with no memory; Resonance without Conversion is applause that
doesn't pay; Conversion without Advocacy is a bucket that never compounds. A brand is
only as strong as its **weakest stage** — that stage is the constraint, and fixing
anything else first is wasted motion.

## brand-review vs the rest of the suite

| | code-review skills | `live-review` | **`brand-review`** (this skill) |
| --- | --- | --- | --- |
| Input | a diff / source | a **URL** | a **brand + its metrics** |
| Asks | "is it built well?" | "does it work for a user?" | "**does the market want it?**" |
| Method | reads code | drives a browser | gathers metrics, benchmarks, diagnoses |
| Evidence | `file:line` | screenshot + repro | **a metric vs its benchmark** |
| Output | merge verdict | merge verdict | **a brand-market-fit grade** |

It's a third mode. The others judge the artifact; this judges the *fit*. Run it on a
launch, a quarterly review, a repositioning, or when growth has stalled and nobody can
say at which stage. It pairs naturally with `live-review` (the live walk often explains
*why* a stage leaks) but answers a different question.

## When this fires

- "Run a brand-market-fit audit / brand review / BMF scorecard."
- Growth has plateaued and you need to find the bottleneck dimension, not guess.
- A launch or repositioning needs a baseline + targets across the five dimensions.
- A specific symptom: "great traffic, no signups" (Relevance/Conversion), "people love
  us but never refer" (Advocacy), "nobody's heard of us" (Awareness).

## The diagnostic

Four phases, same spine as the rest of the suite. Don't score before you've scoped.

### 1. Context — what brand, which market, what stage

You can't benchmark a number without knowing the brand. Establish, briefly:

- **Brand & category.** What is it, what category does it compete in (the category sets
  every benchmark — a 2% CVR is great for enterprise SaaS, dismal for impulse e‑comm).
- **ICP / target market.** Who is this *for*? Relevance and Audience Match are measured
  against this, not against "everyone."
- **Stage & goal.** Pre-product-market-fit (hunting for a market) vs scaling (pouring
  fuel on a found one) changes which dimensions matter. What does success look like?
- **Time window & data access.** What period are we scoring, and what can you give me —
  analytics (GA), ad platforms, search console, billing/CRM, survey/NPS, social? If a
  metric isn't available, I'll say so and fall back to a documented qualitative proxy
  rather than invent a number. **Never fabricate a metric.** A gap is a finding too.

If the ICP, category, or goal is unclear, I ask before scoring — a BMF score against the
wrong market is worse than no score.

### 2. Gather — one column of metrics per dimension

For each of the five dimensions, collect the metrics in the table below. Pull them from
the data the user provides, from analytics they grant access to, or — for the *public*
signals (share of search, organic/brand mention, UGC) — from web and social sources
(`WebSearch`/`WebFetch`, or the `last30days` skill for fresh cross-platform mentions).
Record **the number, the source, and the window** for every metric; a metric with no
provenance is a guess.

| Dimension | Core question | Metrics to gather |
| --- | --- | --- |
| **Awareness** 认知 | How many know you? | Brand Awareness, Share of Search, Direct Traffic, Organic Mention |
| **Relevance** 相关性 | Is it for them? | CTR, Landing-page CVR, Survey, Audience Match |
| **Resonance** 共鸣 | Do they feel it? | Engagement Rate, Comments, Share Rate, UGC count |
| **Conversion** 转化 | Does it pay? | CAC, ROAS, CVR, Revenue, LTV |
| **Advocacy** 传播 | Do they bring others? | NPS, Referral Rate, Repeat Purchase, Brand Mention |

Each `reference/*.md` defines its metrics precisely (formula, how to source it, common
ways the number lies) and gives category benchmarks. Pull the guide for the dimension
you're scoring.

### 3. Score & diagnose — rate each stage, find the constraint

For each dimension, rate it against its benchmark on a four-level scale, then find the
funnel's weakest stage — **that's the diagnosis.**

- 🟢 **Strong** — at or above benchmark; this stage is an asset.
- 🟡 **Adequate** — functional but below where it should be; a tune-up, not a fire.
- 🟠 **Weak** — materially underperforming; meaningfully throttling the stages after it.
- 🔴 **Critical** — broken or no signal; growth is capped here until it's fixed.

The **overall fit is weakest-link, not an average** (mirroring the orchestrator's
"worst layer wins"): a 🟢🟢🟢🟢🔴 brand is a 🔴 brand — four healthy stages can't
compensate for one that's broken, because the funnel drains there. Name the bottleneck
explicitly and diagnose *why* it leaks (is Conversion low because traffic is irrelevant
upstream, or because the funnel itself has friction? `reference/scoring-and-funnel.md`
has the cross-stage diagnosis tree).

### 4. Verdict & levers — the grade and the few moves that matter

Give the overall fit grade (below), then the **one or two highest-leverage moves**,
concentrated on the bottleneck. A list of twenty tactics is noise; the funnel says
where the one that matters is. Each recommendation carries a severity label and names
the metric it's meant to move and by roughly how much.

## The shared review model (adapted)

Same severity vocabulary as the rest of the suite, so findings compose. Because this is
a **strategy diagnostic, not a merge gate**, the verdict is a *fit grade*, not
approve/block.

**Severity labels** (one per finding/recommendation):

- 🔴 **blocking** — a broken stage capping the whole funnel; growth can't scale until fixed.
- 🟠 **important** — a real drag on fit; significant value leaking, not yet fatal.
- 🟡 **nit** — minor underperformance or polish; worth it once the big stages are healthy.
- 🔵 **suggestion** — an experiment or alternative to test, take it or leave it.
- 📚 **learning** — context/benchmark, no action required.
- 🌟 **praise** — a dimension that's genuinely strong and worth defending/doubling down on.

**Fit grades** (the overall verdict):

- 🟢 **Strong fit** — every stage healthy; awareness compounds into advocacy that feeds
  awareness. Pour fuel on it.
- 🟡 **Emerging fit** — the core works; one or two stages leak. Fixable; know which.
- 🟠 **Weak fit** — a stage is critically broken; growth is capped there. Fix the
  constraint before spending on anything upstream.
- 🔴 **No / pre-fit** — multiple stages broken or no signal. The brand isn't landing;
  this is a positioning/product question, not a marketing-spend one.

**Principles:**

- **Measure, don't assert.** Every score is a number vs a benchmark vs a window. No data → say so, use a labeled proxy, never invent.
- **Find the constraint.** The funnel has exactly one weakest stage; lead with it. Don't bury it under a flat list of every dimension.
- **Say WHY in market terms.** "Share of search is 4% vs the category leader's 30% — three of four buyers never consider you" beats "low awareness."
- **Diagnose across stages.** A low number is often caused by the stage *before* it. Treat the cause, not the symptom.
- **Praise is signal.** A 🟢 dimension is a moat — name it so the team protects and exploits it.

## Reference guides

Load on demand — pull the guide for the dimension you're scoring. Don't load all of them.

| Reference file | Load when scoring… |
| --- | --- |
| `reference/awareness.md` | Awareness — share of search, direct traffic, brand recall, organic mention; how to source each |
| `reference/relevance.md` | Relevance — CTR, landing CVR, message-test surveys, audience/ICP match |
| `reference/resonance.md` | Resonance — engagement rate, comments, share rate, UGC volume, sentiment |
| `reference/conversion.md` | Conversion — CAC, ROAS, CVR, revenue, LTV, and the LTV:CAC ratio |
| `reference/advocacy.md` | Advocacy — NPS, referral rate, repeat purchase, brand mention, the loop |
| `reference/scoring-and-funnel.md` | the overall model — benchmarks, the four-level scale, weakest-link rollup, the cross-stage diagnosis tree |

A copy-paste scorecard to fill in lives in [`assets/brand-review-scorecard.md`](assets/brand-review-scorecard.md).

## Output format

Lead with the scorecard and the bottleneck, then the prioritized moves, then the grade.

```markdown
# Brand-market-fit review — <brand> · <category> · <window>

**Fit grade:** 🟠 Weak fit   **Bottleneck:** Conversion 🔴
**Summary:** <2–3 sentences — what the funnel looks like and where it leaks.>

## Scorecard
| Dimension | Score | Key metric (vs benchmark) | Read |
| --- | --- | --- | --- |
| Awareness  | 🟢 Strong   | Share of search 22% (cat. avg 15%) | known in-category |
| Relevance  | 🟢 Strong   | Landing CVR 6.1% (B2B SaaS ~5%)    | message lands |
| Resonance  | 🟡 Adequate | Share rate 0.8% (target ~2%)       | liked, not shared |
| Conversion | 🔴 Critical | LTV:CAC 1.4:1 (healthy ≥3:1)       | unit economics underwater |
| Advocacy   | 🟠 Weak     | NPS 18 (good ≥30)                  | few promoters |
```

Then the findings, grouped by severity, each with the metric, the WHY, and a lever:

> 🔴 **blocking** — Conversion · LTV:CAC = 1.4:1 (healthy ≥ 3:1)
> You pay ~$420 to acquire a customer worth ~$590 over their life — every paid
> customer barely breaks even, so spending more to grow loses money faster. The leak
> isn't top-of-funnel (Awareness/Relevance are 🟢); it's that acquired users don't
> retain. **Lever:** lift LTV before scaling spend — attack the post-purchase drop-off
> (onboarding completion, month-2 retention). Target LTV:CAC ≥ 3:1 before reopening paid.

> 🌟 **praise** — Awareness · Share of search 22% vs 15% category average
> In your category you're the brand buyers search for by name — a durable moat. Protect
> it; don't let a repositioning dilute the name recognition you've earned.

End with the verdict block:

```
### Fit grade: 🟠 Weak fit — bottleneck: Conversion

Top of funnel is genuinely strong: you're known and relevant. The brand fails at the
cash register — unit economics are underwater (LTV:CAC 1.4:1) and weak advocacy (NPS 18)
means no organic compounding to offset it. Do not scale spend. Fix retention/LTV first,
then advocacy; awareness needs no investment right now.

🔴 1 · 🟠 1 · 🟡 1 · 🌟 1
Scored: Awareness 🟢 · Relevance 🟢 · Resonance 🟡 · Conversion 🔴 · Advocacy 🟠
Window: 2026-Q1 · Sources: GA4, Stripe, Delighted (NPS), Google Trends
```

Lead with the constraint. A reader who stops after the bottleneck and its one lever has
handled the thing actually capping growth.

## Data & tooling notes

- **Bring-your-own-metrics first.** The most accurate scores come from the user's own
  analytics/CRM/billing/survey exports — ask for them in Phase 1. Read pasted numbers or
  CSV/exports with `Read`.
- **Public signals** (share of search, organic/brand mention, UGC volume, sentiment) can
  be sourced live: `WebSearch`/`WebFetch`, Google Trends for brand-vs-category search, and
  the **`last30days`** skill for fresh cross-platform mention volume and sentiment.
- **Benchmarks are category-specific.** A score is meaningless without the right comparison
  — `reference/scoring-and-funnel.md` lists default benchmarks by category and flags when
  to insist on the user's own historical baseline instead.
- **No fabrication.** If a metric can't be sourced, mark it `n/a — proxy: <what you used>`
  in the scorecard and lower confidence on that dimension. Missing data is itself a finding.
