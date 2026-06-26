# Conversion (转化) — does it pay?

Where brand meets money. Awareness, Relevance, and Resonance are worthless if they don't
convert into economically sound customers. The question isn't just "do they buy?" — it's
**"do they buy for less than they're worth?"** The decisive metric here is a *ratio*
(LTV:CAC), not any single number.

## Metrics

### CAC (customer acquisition cost)
Fully-loaded cost to acquire one paying customer: `(marketing + sales spend) ÷ new customers`,
over the same period. Include ad spend, content, salaries, and tooling — not just media.

- **Source:** ad platforms + finance for spend; CRM/billing for new-customer count.
- **Watch:** *blended* CAC (all spend ÷ all new customers, incl. organic) flatters you;
  *paid* CAC (paid spend ÷ paid-attributed customers) is the truth for scaling decisions.
  Report both, decide on paid.

### LTV (lifetime value)
Total gross margin a customer generates before they churn:
`ARPU × gross margin % × average lifetime (= 1 ÷ churn rate)`.

- **Source:** billing/CRM for ARPU and churn; finance for margin.
- **Watch:** early-stage LTV is a *projection* — short-lived companies overestimate lifetime.
  State the assumption (churn rate, margin) behind the number.

### LTV:CAC ratio — the single most important fit number here
`LTV ÷ CAC`. This is whether the brand has a *business*, not just sales.

- **Benchmark:** **≥ 3:1 is healthy.** 1:1 means you lose money on every customer after costs;
  >5:1 often means you're *under*-investing in growth and leaving the market to competitors.
- **CAC payback:** months to recover CAC from gross margin — **< 12 months** for most SaaS,
  faster for transactional. Long payback strangles cash even at a fine LTV:CAC.

### ROAS (return on ad spend)
`revenue attributed to ads ÷ ad spend`. A channel-level efficiency check.

- **Benchmark:** breakeven ROAS depends entirely on margin; a 70%-margin product needs lower
  ROAS than a 20%-margin one. Compute *your* breakeven before judging a ROAS number.

### CVR & Revenue
Funnel-stage conversion rate(s) and absolute revenue/growth — the volume context for the
efficiency ratios above. A great LTV:CAC on tiny volume is a hobby, not fit; huge revenue at
1:1 is setting money on fire faster.

## Diagnosing low Conversion

- **CAC too high** → either the funnel converts poorly (fix the page/flow — but check
  Relevance first; high CAC is often *upstream* irrelevant traffic) or channels are
  saturated/mis-chosen.
- **LTV too low (the common one)** → acquisition is fine; *retention* is the leak. Customers
  don't stay long enough to be worth their cost. This is usually a product/onboarding problem
  surfacing as a brand-economics problem — see the cross-stage tree in `scoring-and-funnel.md`.
- **Good CVR, bad LTV:CAC** → you're converting the wrong people (Audience Match failure in
  Relevance). Treat upstream.
- **Strong LTV:CAC, low growth** → conversion is healthy and *under-fueled*; the bottleneck is
  Awareness/Relevance, not here. Don't optimize the checkout — buy more reach.

## Levers (only if Conversion is the bottleneck)

- **Lift LTV before scaling spend** — retention and expansion move LTV:CAC more durably than
  cutting CAC. A 10% churn improvement can outweigh any landing-page tweak.
- **Reduce funnel friction** — fewer steps, fewer form fields, no forced signup before value;
  every step drops a measurable % of users.
- **Trust at the decision point** — guarantees, social proof, transparent pricing, security
  badges reduce the perceived risk that kills conversion.
- **Right-channel CAC** — shift spend to channels with the best payback; kill the ones above
  breakeven ROAS.

## Common ways the number lies

- **Blended CAC hides paid reality** — organic subsidizes the average; you can't scale organic
  by spending more.
- **Projected LTV is optimistic** — early cohorts haven't churned yet; use cohort-based, not
  blended, LTV.
- **Attribution inflates ROAS** — last-click hands organic/branded conversions to paid;
  sanity-check against incrementality or holdout tests.
- **Revenue ≠ profit** — growing revenue at LTV:CAC < 1 accelerates losses; never read revenue
  without the ratio.
