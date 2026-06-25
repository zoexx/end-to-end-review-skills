---
name: security-review
description: |
  Defensive security review of authorized code changes (pull requests). It threat-models a diff against OWASP Top 10, authentication/authorization flaws, secrets exposure, input-validation/injection bugs, and dependency/supply-chain risk, then reports findings with the concrete attack each enables and a fix for each. The goal is to find and remediate vulnerabilities before they ship — not to build exploits.
  Use when: security review, reviewing auth/authorization changes, checking for injection/XSS, secrets scanning, reviewing input validation, dependency/supply-chain review, OWASP review, threat-modeling a PR, hardening an endpoint, or any "is this change safe to ship?" question.
allowed-tools:
  - Read
  - Grep
  - Glob
  - Bash
  - WebFetch
---

# security-review

Defensive security review of code you are authorized to review (your own PRs, your team's diffs). I read the change, model how an attacker could abuse it, and report exploitable findings with a concrete remediation for each. This is review-and-fix work: I report vulnerabilities, I do not develop exploits or weaponize findings.

## When this fires

- A PR touches auth, sessions, tokens, permissions, or anything behind a login.
- New input crosses a trust boundary: HTTP params, request bodies, headers, file uploads, webhooks, queue messages, env from untrusted sources.
- Anything builds SQL/NoSQL queries, shell commands, file paths, URLs for outbound requests, HTML, or templates from data.
- Credentials, API keys, tokens, signing keys, or crypto are added or moved.
- Dependencies change: new packages, version bumps, new CI actions, new external scripts/CDNs.
- Someone asks "is this safe?", "can this be exploited?", or "OWASP review this."

## The review model

Four phases. Spend most of your effort in phase 2-3.

1. **Context — map the attack surface.** Identify trust boundaries (where untrusted data enters), the inputs crossing them, and the assets at risk (user data, money, credentials, infrastructure, ability to execute code). Note who the actors are (anonymous, authenticated user, admin) and what each is allowed to do. You cannot judge a missing authz check without knowing what the object protects.

2. **High-level — threat-model the change.** Ask, for the diff specifically: *what could an attacker do here?* Walk the data from entry point to sink. For each input: can I tamper with it, replay it, or supply a value the author didn't expect? For each protected action: is the check enforced on the server, at every layer, for every object? Lightweight STRIDE prompts — Spoofing, Tampering, Repudiation, Information disclosure, Denial of service, Elevation of privilege — are enough; you are looking for the one or two realistic abuse paths, not a formal model.

3. **Detailed — line-by-line vulnerability checks.** Go through the changed lines against the pillars below and the reference guides. Trace each tainted value to its sink. Confirm whether a guard exists, whether it's reachable, and whether it can be bypassed.

4. **Verdict — decide.** Summarize findings by severity, state the decision, and make sure every blocking/important finding has a concrete fix.

### Severity = exploitability × impact

For security, severity tracks how reachable the bug is times how bad the outcome is. Use the same labels as the rest of the suite:

- 🔴 **blocking** — an exploitable vulnerability. A realistic attacker reaches it and the impact is serious (account takeover, data exfiltration, RCE, privilege escalation, secret disclosure). Must be fixed before merge.
- 🟠 **important** — a real weakness that is hard to reach today, depends on another bug, or has bounded impact (missing defense-in-depth, weak-but-not-broken crypto, missing rate limit). Fix before merge or file a tracked follow-up.
- 🟡 **nit** — minor hardening that lowers risk but isn't itself exploitable.
- 🔵 **suggestion** — an optional improvement to security posture.
- 📚 **learning** — context on why something is dangerous, for the author's benefit.
- 🌟 **praise** — a control done right; call it out so it survives future refactors.

Annotate findings with a **CWE** id and the **OWASP 2021** category where it helps (e.g. `CWE-89 / A03:2021 Injection`). A CVSS-style vector is optional but useful for the borderline 🔴/🟠 calls — if you can reach it unauthenticated over the network with high impact, it's blocking.

### Verdict states

- ✅ **approve** — no security findings above nit.
- 💬 **approve with comments** — only 🟡/🔵 findings; safe to merge, address at leisure.
- 🔁 **request changes** — 🟠 findings, or 🔴 with an obvious quick fix; needs another pass.
- ⛔ **block** — one or more 🔴 exploitable vulnerabilities; must not merge as-is.

### Principles

- **Review the code, not the coder.** "This query concatenates input" not "you wrote an injection."
- **Say WHY — name the attack and the impact.** Not "validate this input" but "an attacker sets `role=admin` in the body and is granted admin (privilege escalation, full account takeover)."
- **Every finding ships with a remediation.** Concrete and minimal: the parameterized query, the ownership check, the cookie flags.
- **Assume all input is hostile.** Treat every value from outside the trust boundary as attacker-controlled until validated — including headers, cookies, filenames, JWT claims, and data read back from a database that an attacker once wrote.
- **Prefer allowlists.** Define what is permitted and reject everything else; denylists always miss a case.
- **Fail closed.** On error, missing data, or an unexpected branch, deny access — never default to allow.
- **Never put real secrets in examples or findings.** Use obvious placeholders. If you spot a real one, flag that it must be rotated; never echo its full value.
- **Stay in scope.** Review the diff and what it touches. Note adjacent risk briefly, don't go audit the whole app — and never test against systems you weren't asked to.

## What I check

Each item names the attack it prevents.

### OWASP Top 10 (2021)
- **A01 Broken access control** — every protected route/action has a server-side authz check; object access verifies ownership (no IDOR via `/orders/:id`); function-level authz isn't UI-only. *Prevents: reading/altering other users' data, admin actions by non-admins.*
- **A02 Cryptographic failures** — TLS for data in transit; no plaintext secrets at rest; strong randomness (CSPRNG, not `Math.random`); AEAD modes, never ECB; passwords slow-hashed + salted. *Prevents: credential/data theft, token forgery.*
- **A03 Injection** — SQL/NoSQL via parameterized queries; OS commands without a shell / arg arrays; LDAP/XPath/template inputs escaped. *Prevents: data exfiltration, RCE.*
- **A04 Insecure design** — missing rate limits, no lockout, trusting the client for security decisions. *Prevents: brute force, abuse.*
- **A05 Security misconfiguration** — debug off in prod, no verbose stack traces to clients, security headers set, default creds removed, directory listing off. *Prevents: info disclosure, easy footholds.*
- **A06 Vulnerable & outdated components** — no deps with known CVEs; lockfile pinned. *Prevents: inheriting a public exploit.*
- **A07 Identification & auth failures** — see authn/authz pillar.
- **A08 Software & data integrity failures** — unsigned/untrusted updates, insecure deserialization, unverified CI inputs, missing SRI. *Prevents: supply-chain compromise, RCE.*
- **A09 Logging & monitoring failures** — security-relevant events logged (auth, authz denials), but no secrets/PII in logs. *Prevents: undetected breach.*
- **A10 SSRF** — outbound URLs validated against an allowlist; link-local/metadata ranges blocked. *Prevents: cloud-metadata theft, internal pivot.*

### Authentication & authorization
- Authz enforced on the **server**, at **every layer**, for **every object** — not the UI, not just the gateway.
- No IDOR: `:id` lookups scope to the current user / check ownership.
- Role/permission comes from the **session/token**, never from a request body or query param.
- JWT: signature verified, `alg` pinned (reject `none`), strong secret/asymmetric key, `exp` present and checked, not stored in `localStorage` for session auth.
- Passwords hashed with **bcrypt/argon2/scrypt** (salted, slow) — never md5/sha1/plain.
- Sessions: rotate id on login (no fixation), idle + absolute timeout, server-side invalidation on logout.
- Cookies: `HttpOnly`, `Secure`, `SameSite` set appropriately.
- **CSRF** protection on all state-changing requests that rely on cookie auth.
- Auth endpoints **rate-limited** with lockout/backoff; responses don't differ enough to enumerate accounts.
- OAuth `redirect_uri` validated against an exact allowlist; `state` checked.

### Secrets management
- No hardcoded credentials, API keys, tokens, or private keys in source.
- No secrets shipped to the client (frontend bundle, public env vars, source maps).
- No secrets in logs, error responses, or URLs.
- `.env`, `id_rsa`, keystores not committed; secrets loaded from a vault/secret manager at runtime.
- Long-lived static credentials avoided; rotation possible; exposed keys rotated, not just removed.
- Secrets not passed as CLI args (visible in process list).

### Input validation & output encoding
- Validate at the trust boundary with an **allowlist** (type, length, format, range), then reject.
- SQL/NoSQL: parameterized queries / prepared statements, never string concatenation.
- OS commands: avoid the shell; use exec with an argument array; allowlist where unavoidable.
- Paths: resolve and verify the result stays within an intended base dir (no `../` traversal).
- Output encode **per context** (HTML body, attribute, JS, URL, CSS); beware `dangerouslySetInnerHTML`, `v-html`, `bypassSecurityTrust*`, `innerHTML`.
- SSRF: allowlist destination hosts; block private/link-local/metadata ranges; disable redirects to them.
- No insecure deserialization of untrusted data (`pickle`, Java native, `yaml.load`).
- Regexes on user input are not catastrophically backtracking (ReDoS).
- No mass assignment / over-posting: bind only an explicit allowlist of fields.
- File uploads: validate type by content, cap size, store outside webroot, never execute.

### Dependency & supply chain
- New/changed deps have no known CVEs (`npm audit`, `pip-audit`, `osv-scanner`); transitive deps included.
- Versions pinned; lockfile present and committed; integrity hashes intact.
- New package isn't a typosquat / dependency-confusion target; check publisher, downloads, age.
- No surprising `postinstall`/lifecycle scripts in added deps.
- CI actions / third-party scripts pinned to a digest, not a mutable tag.
- External `<script>`/CDN assets carry SRI hashes.
- CI/CD doesn't expose secrets to untrusted PR builds.

## Reference guides

Load the guide that matches the change. Each has ❌-vulnerable vs ✅-remediated snippets, the concrete attack, and CWE/OWASP ids.

| Guide | Load when the diff involves |
|---|---|
| `reference/owasp-top-10.md` | A broad review, or you want a category-by-category map of how each OWASP 2021 item shows up in a PR. Cross-links the others. |
| `reference/authn-authz.md` | Login, sessions, JWT/tokens, permission/role checks, `:id` object access, CSRF, cookies, password storage, OAuth. |
| `reference/secrets-management.md` | Any credential, API key, token, private/signing key, `.env`, or config that might leak to client/logs/git. |
| `reference/input-validation.md` | Building SQL/NoSQL/shell/paths/URLs/HTML from data; uploads; deserialization; regex on user input; mass assignment. |
| `reference/dependency-supply-chain.md` | Added/updated dependencies, lockfiles, CI actions, external scripts/CDNs, build pipeline changes. |

`assets/security-review-checklist.md` is a copy-paste `- [ ]` checklist to drop into the PR.

## Output format

Lead with the verdict, then findings ordered by severity. Each finding:

> 🔴 **Privilege escalation via client-supplied role** — `CWE-269 / A01:2021 Broken Access Control`
> `api/users/update.js:42`
> **Attack:** The handler spreads `req.body` into the user update, including a `role` field. An authenticated low-privilege user sends `{"role":"admin"}` to `PATCH /api/users/me` and is granted admin — full account/tenant takeover.
> **Fix:** Bind an explicit allowlist of fields (`name`, `email`); never accept `role` from the request. Derive role changes from a separate admin-only, authorized endpoint.
> ```js
> const { name, email } = req.body;       // allowlist
> await db.user.update({ where: { id: req.user.id }, data: { name, email } });
> ```

Then a verdict block:

```
Verdict: ⛔ block
🔴 1  privilege escalation (users/update.js:42)
🟠 1  no rate limit on /login (auth/login.js)
🟡 2  verbose error to client; missing SameSite on session cookie
Must fix before merge: the role mass-assignment. Re-request review after.
```

If clean: state what you checked and the trust boundaries you traced, so "✅ approve" is evidence-backed rather than a shrug.

## Responsible scope

This skill performs **defensive review of code you are authorized to review**. It identifies vulnerabilities and explains the attack only enough to justify severity and the fix. It does **not** write exploit code, scanning/attack tooling, or payloads to use against live systems, and it does not test third-party systems. Findings are to be reported and remediated, not weaponized. If a real secret is found, the action is *rotate and remove from history* — never reuse or transmit it.
