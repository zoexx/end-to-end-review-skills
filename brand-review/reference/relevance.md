# Relevance (相关性) — is it *for them*?

Awareness gets you considered; Relevance gets you chosen. The question: when your target
buyer meets your message, do they think **"this is for me, for my problem"**? Relevance is
measured *against the ICP* — a high CTR from the wrong audience is negative relevance,
because it spends money and pollutes every downstream metric.

## Metrics

### CTR (click-through rate)
Of people who saw the message (ad, search result, email), the share who clicked.

- **Source:** ad platforms (Meta/Google), Search Console (organic CTR by query), ESP (email).
- **Benchmark (rough, varies wildly by channel):** search ads 3–5%; display 0.1–1%; Meta
  feed ~0.9–1.5%; cold email 1–3%; good email newsletter 20–30% open / 2–5% click.
- **Read:** CTR measures whether the *promise* is relevant. Low CTR with high impressions =
  the hook doesn't match what the audience wants. But high CTR + low downstream CVR = the
  ad over-promises (a relevance *mismatch* between ad and landing page).

### Landing-page CVR (conversion rate)
Of people who land, the share who take the intended next step (signup, lead, add-to-cart).

- **Source:** GA4 / analytics, segmented by *source* (so you separate message relevance from
  traffic quality).
- **Benchmark:** B2B SaaS lead/signup ~2–5%; e-commerce ~1.5–3%; high-intent search landing
  can be 10%+. Compare like-for-like traffic.
- **Read:** the single clearest "is the message landing?" number. Low CVR from on-target,
  high-intent traffic = the page doesn't convince *this* audience it's for them.

### Survey / message testing
Direct: "How disappointed would you be if you could no longer use this?" (the Sean Ellis
PMF question), or message-resonance polls ("which of these describes the value to you?").

- **Source:** in-product survey, panel, customer interviews.
- **Benchmark:** ≥40% "very disappointed" on the Sean Ellis test is the classic PMF
  threshold. Below ~25% = the value isn't relevant enough to the people you're reaching.

### Audience Match
How well *actual converters* match the *intended ICP* (firmographics/demographics, use case).

- **Source:** CRM / analytics segmentation — compare who converts to who you targeted.
- **Read:** if converters skew away from the ICP, your message is relevant — to the wrong
  people. Either the targeting or the ICP definition is wrong. This is the most-missed
  relevance failure: the funnel looks fine but is filling with users who won't retain.

## Diagnosing low Relevance

- **High CTR, low CVR** → ad/landing mismatch. The hook promises one thing, the page delivers
  another. Align the message; don't add more traffic.
- **Low CTR across the board** → the core promise isn't resonant with this audience, or the
  audience is wrong. Test new angles *and* re-examine the ICP.
- **Good CVR, bad retention/LTV downstream** → audience-match failure. You're converting the
  wrong people relevantly. Fix targeting/ICP — this is the cause of many "Conversion looks
  fine but unit economics are bad" cases (see `scoring-and-funnel.md`).

## Levers (only if Relevance is the bottleneck)

- **Sharpen the value proposition** — pass the 5-second test: a stranger should know *what it
  is, who it's for, and why it's better* before scrolling. Lead with the job-to-be-done, not
  adjectives.
- **Message-market match by segment** — different ICPs need different landing pages/hooks;
  one generic message is relevant to no one.
- **Cut the wrong traffic** — narrowing targeting to the ICP usually lifts CVR *and* LTV more
  than any copy change.

## Common ways the number lies

- **CTR optimizes for clicks, not customers** — clickbait wins CTR and loses CVR/LTV.
- **Blended CVR hides everything** — always segment by source; a great organic CVR can mask a
  terrible paid one.
- **Survey selection bias** — surveying only happy active users inflates the PMF score;
  include churned and bounced users.
