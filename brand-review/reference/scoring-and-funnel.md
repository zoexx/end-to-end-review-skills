# Scoring & the funnel — turning five numbers into one diagnosis

This is the model that ties the five dimensions together: how to score each one, how to roll
them into a single fit grade, and — most importantly — how to find which stage is *actually*
the constraint when several look weak. Pull this guide in Phase 3 (Score & diagnose) and
Phase 4 (Verdict).

## The four-level scale

Score each dimension against its **category benchmark**, not an absolute:

| Score | Meaning | Action posture |
| --- | --- | --- |
| 🟢 **Strong** | at/above benchmark; an asset and a moat | protect & exploit |
| 🟡 **Adequate** | functional, below potential | tune up once bigger stages are healthy |
| 🟠 **Weak** | materially underperforming; throttling downstream | fix soon |
| 🔴 **Critical** | broken or no signal; caps the whole funnel | fix *first*, before anything upstream |

Assign the score from the dimension's reference file (each lists its benchmarks). When a
metric can't be sourced, score from the available proxies and **lower confidence** on that
dimension — don't pretend precision you don't have.

## Weakest-link rollup (not an average)

The five dimensions are a funnel: value flows Awareness → Relevance → Resonance → Conversion
→ Advocacy, and Advocacy loops back to feed Awareness. **A funnel is only as strong as its
leakiest stage**, so the overall fit grade tracks the *worst* stage, not the mean — exactly
like the `end-to-end-review` orchestrator's "worst layer wins."

| Funnel pattern | Fit grade |
| --- | --- |
| all 🟢/🟡, no stage below 🟡 | 🟢 **Strong fit** |
| one stage 🟠, rest healthy | 🟡 **Emerging fit** |
| any stage 🔴, or two+ stages 🟠 | 🟠 **Weak fit** |
| two+ stages 🔴, or no signal across the funnel | 🔴 **No / pre-fit** |

Averaging hides the constraint: a 🟢🟢🟢🟢🔴 brand scores 80% on a mean and *fails* in
reality, because everything drains at the 🔴. Report the pattern, name the bottleneck, grade
on the weakest link.

## The bottleneck is the only thing that matters first

There is exactly **one** binding constraint at a time — the earliest/worst leaking stage.
Spending on any stage that isn't the constraint is wasted: more awareness into a funnel that
fails at Conversion just buys more people to lose. Theory of constraints, applied to brand.

So Phase 4 leads with: **which stage is the constraint, and the one or two levers on it.**
Everything else is secondary until the constraint moves.

## Cross-stage diagnosis tree — *why* a stage really leaks

A low number is frequently *caused by the stage before it*. Treat the cause, not the symptom.
Walk these before prescribing:

- **Conversion 🔴/🟠 — check Relevance first.**
  - Low CVR but on-target, high-intent traffic → genuine Conversion problem (funnel friction,
    weak trust signals, pricing). Fix here.
  - Low CVR with broad/cheap traffic → *Relevance* problem upstream; the traffic is irrelevant.
    Don't touch the checkout.
  - LTV:CAC bad but CVR fine → usually a **retention** leak (LTV), often an *Audience Match*
    failure (Relevance) — you converted the wrong people. Fix targeting/ICP and retention, not
    the landing page.

- **Advocacy 🔴/🟠 — split goodwill vs mechanism.**
  - High NPS, low referral → missing *mechanism*; build the referral loop. Real Advocacy fix.
  - Low NPS, low repeat → the *product/value* is the problem (Conversion's retention, or core
    fit). A referral program will only spread disappointment. Fix upstream.

- **Resonance 🟠 with strong Relevance** → the message is *correct* but *forgettable*. It
  informs without moving. Needs a point of view/identity, not new facts.

- **Relevance 🔴 with strong Awareness** → people know you and don't think you're for them.
  Positioning/ICP problem — the most expensive to ignore, because you're spending awareness
  budget to be rejected. Often shows up as high CTR, low CVR, or good CVR with bad LTV.

- **Awareness 🔴 with everything-else unknown** → you may simply lack the reach to have signal
  yet (pre-fit). Get enough reach to measure the next stage before concluding the brand fails.

- **Everything 🟢 except low growth** → no bottleneck *inside* the funnel; the constraint is
  *volume/fuel*. The brand works — invest to scale it. This is a 🟢 Strong fit, under-funded.

## Benchmarks — use the right comparison

Benchmarks are **category- and stage-specific**; a number is meaningless without the right
yardstick. In priority order:

1. **The brand's own historical baseline** — the most honest comparison. Is each dimension
   trending up or down? Always prefer this when you have ≥2 periods.
2. **Direct competitors** — for share of search, share of voice, mention share especially.
3. **Category benchmarks** — the rough defaults in each dimension's reference file (B2B SaaS,
   e-commerce, consumer, marketplace differ a lot). Use as a fallback, and *say* it's a generic
   benchmark, not their peer set.

When you only have one period and no peer data, score on *absolute* health thresholds
(LTV:CAC ≥ 3, NPS ≥ 30, Sean Ellis ≥ 40%) and flag lower confidence.

## Pre-fit vs scaling — which stages even matter

- **Pre-product-market-fit** (hunting for a market): Relevance and Resonance dominate — you're
  testing whether *anyone* deeply wants this. Don't over-invest in Awareness or optimize
  Conversion %; you're searching, not scaling. The Sean Ellis ≥40% and retention-curve-flattens
  signals are the ones that matter.
- **Scaling** (found a market): Awareness, Conversion economics (LTV:CAC), and Advocacy loops
  dominate — you're pouring fuel on something that works. Now CAC payback and referral rate are
  the constraints.

Scoring a scaling-stage brand by pre-fit metrics (or vice versa) misreads the constraint —
establish stage in Phase 1.

## Confidence & honesty

Attach a confidence level to the overall grade based on data coverage:

- **High** — most dimensions scored from the brand's own first-party metrics over ≥2 periods.
- **Medium** — mix of first-party metrics and public proxies; some single-period.
- **Low** — mostly proxies/qualitative; flag which dimensions are essentially estimates.

A 🔴 No-fit grade at *low* confidence means "no signal yet," which is different from "measured
and failing." Say which.
