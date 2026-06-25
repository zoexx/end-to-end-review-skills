# end-to-end-review-skills

A suite of **[Agent Skills](https://docs.claude.com/en/docs/agents-and-tools/agent-skills)** that give Claude Code a structured, senior-level review process for every layer of a web product — from the pixels in the browser down to the rows in the database.

Each layer is its own installable skill. An orchestrator skill stitches them into a single end-to-end pass.

```
end-to-end-review-skills/
├── frontend-review/      🎨  UI/UX, components, state, a11y, perf, SEO
├── backend-review/       ⚙️   APIs, services, business logic, errors, observability
├── database-review/      🗄️   schema, queries, indexes, migrations, integrity
├── security-review/      🔒  authn/z, secrets, input validation, OWASP, supply chain
├── mobile-review/        📱  iOS/Android/RN — lifecycle, offline/sync, battery, native bridge
├── infra-review/         🚇  CI/CD, IaC, containers/K8s, deploy & rollback safety
├── performance-review/   ⚡  cross-cutting perf — profiling, latency, caching, budgets, scale
└── end-to-end-review/    🧭  orchestrator — routes to the engaged layers and merges the report
```

> Inspired by [awesome-skills/code-review-skill](https://github.com/awesome-skills/code-review-skill). Where that skill is one skill spanning many languages, this suite is many skills spanning the layers of one product.

---

## Why a suite, not one skill

A real review fails differently at each layer. A frontend reviewer is thinking about layout shift and keyboard focus; a database reviewer is thinking about missing indexes and lock contention. Bundling them into one prompt produces shallow, generic feedback. Splitting them lets each skill carry a **deep, layer-specific checklist** and load only the reference material that matters — and lets you run just the one you need.

You can install a single skill (e.g. only `frontend-review`), or install all five and let the `end-to-end-review` orchestrator coordinate a full sweep.

---

## Install

Skills are plain folders. Copy the ones you want into a skills directory Claude Code reads:

| Scope | Path | Available in |
| --- | --- | --- |
| **Personal** | `~/.claude/skills/` | every project on your machine |
| **Project** | `<repo>/.claude/skills/` | that repo, shared via git |

```bash
# Personal install — all eight skills
git clone https://github.com/zoexx/end-to-end-review-skills
cp -R end-to-end-review-skills/{frontend,backend,database,security,mobile,infra,performance,end-to-end}-review ~/.claude/skills/

# Or just one skill, project-scoped
cp -R end-to-end-review-skills/frontend-review <repo>/.claude/skills/
```

Restart Claude Code (or run `/skills` to refresh). Each skill activates automatically when your request matches its `description`, or you can invoke it explicitly:

```
> review the frontend changes on this branch
> /frontend-review
> run an end-to-end review of this PR
```

---

## The shared review model

Every skill in this suite speaks the same language so their output composes cleanly.

### Four-phase process

1. **Context** — understand *intent* before code. Read the PR/diff, the linked issue, the blast radius. A change is only correct relative to what it was meant to do.
2. **High-level** — assess architecture and design fit before nitpicking lines. Is this the right shape? Catching a wrong abstraction here saves a hundred line comments later.
3. **Detailed** — line-by-line, layer-specific analysis using the skill's reference guides.
4. **Verdict** — a summary and an explicit decision, with findings grouped by severity.

### Severity labels

Every finding carries one label, so authors can triage at a glance:

| Label | Meaning |
| --- | --- |
| 🔴 **blocking** | Must fix before merge — correctness, security, data loss |
| 🟠 **important** | Should fix — a real problem, not a blocker |
| 🟡 **nit** | Minor style/polish, non-blocking |
| 🔵 **suggestion** | Optional improvement or alternative to consider |
| 📚 **learning** | Educational context, no action required |
| 🌟 **praise** | Genuinely good work, worth calling out |

### Verdict states

`✅ approve` · `💬 approve with comments` · `🔁 request changes` · `⛔ block`

### Review principles

- **Review the code, not the coder.** Comment on the change; ask questions instead of issuing verdicts.
- **Say why.** Every blocking/important finding names the concrete risk — the bug it causes, the attack it enables, the user it hurts.
- **Stay in scope.** Flag pre-existing issues separately from the change under review; don't hold a PR hostage to unrelated cleanup.
- **Praise is signal too.** Calling out a good pattern teaches as much as flagging a bad one.

---

## What each skill covers

| Skill | Focus | Sample findings it catches |
| --- | --- | --- |
| **frontend-review** | architecture & maintainability, UX & design fidelity, Core Web Vitals, accessibility & SEO | un-memoized callbacks causing re-renders, layout shift from un-sized images, missing focus states, `<div>` soup instead of semantic HTML |
| **backend-review** | API contracts, business logic, error handling, concurrency, observability | unbounded queries, swallowed errors, non-idempotent retries, missing pagination, N+1 across a service boundary |
| **database-review** | schema design, query performance, migrations, transactions & integrity | missing indexes on FKs, non-sargable predicates, blocking migrations on large tables, missing `NOT NULL`/unique constraints |
| **security-review** | authn/z, secrets, input validation, OWASP Top 10, dependency & supply-chain risk | IDOR, SQL/command injection, hardcoded secrets, missing authorization checks, vulnerable transitive deps |
| **mobile-review** | lifecycle & navigation, offline/sync, battery/memory, platform a11y, native bridge & store readiness | retain cycles, main-thread jank/ANR, lost writes when offline, missing VoiceOver/TalkBack labels, secrets in plaintext storage |
| **infra-review** | CI/CD, IaC, containers/K8s, deploy & release safety, config/secrets/observability | fork pwn-request secret exfil, unpinned actions/images, missing resource limits & probes, deploys with no rollback path |
| **performance-review** | profiling, algorithmic cost, latency/throughput, caching, frontend budgets, load & scalability | O(n²) on the hot path, sequential awaits, cache stampede, bundle bloat, memory growth under load |
| **end-to-end-review** | orchestration | routes a diff to the right domain skills, merges findings, de-dupes cross-layer issues, produces one prioritized report |

Each skill folder contains a `SKILL.md` (the entry point), a `reference/` directory of deep-dive guides loaded on demand, and an `assets/` checklist you can hand to a human reviewer.

---

## Contributing

New layers (data/ML, AI/LLM, design-system) and new language guides under any skill's `reference/` are welcome. See [CONTRIBUTING.md](CONTRIBUTING.md).

## License

[MIT](LICENSE) © 2026 Zoe Ying
