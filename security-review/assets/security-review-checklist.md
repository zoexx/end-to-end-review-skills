# Security Review Checklist

Defensive review of an authorized change. Copy into the PR and check each box, or mark `n/a`. Each item is phrased as a verifiable check — confirm it in the diff, not in intent. Severity tracks exploitability × impact; a 🔴 is an exploitable vuln (block).

## Quick scan — run first
- [ ] No secret-shaped strings in the diff (keys, tokens, passwords, private keys) — `grep` for `key`, `secret`, `token`, `password`, `BEGIN.*PRIVATE`.
- [ ] No `.env` / `id_rsa` / `*.pem` / keystore added.
- [ ] Dependency/lockfile changes audited (`npm audit` / `pip-audit` / `osv-scanner` clean).
- [ ] Trust boundaries identified: where does untrusted input enter, and what asset is at risk?

## Access control (A01)
- [ ] Every protected route/action has a **server-side** authz check (not UI-only); fails closed.
- [ ] Object reads/writes verify ownership — no IDOR via `:id`; returns 404 on miss.
- [ ] Role / permission / tenant comes from the session or token, never from the request body or query.
- [ ] Function-level authz enforced on admin/privileged endpoints when called directly.

## Authentication & session (A07)
- [ ] Passwords hashed with bcrypt / argon2 / scrypt (salted, slow) — no md5/sha1/sha256/plain.
- [ ] JWT: signature verified, `alg` pinned (reject `none`), strong secret/key, `exp` set & checked.
- [ ] Session id regenerated on login (no fixation); invalidated server-side on logout; timeouts set.
- [ ] Session JWT/token not stored in `localStorage` for auth.
- [ ] Cookies set `HttpOnly`, `Secure`, and `SameSite`.
- [ ] CSRF protection on all cookie-authenticated state-changing requests.
- [ ] Auth/reset/MFA/token endpoints are rate-limited with lockout/backoff.
- [ ] Login/signup/reset responses don't enumerate accounts (uniform message + timing).
- [ ] OAuth `redirect_uri` exact-matched against an allowlist; `state` verified.

## Injection & input validation (A03)
- [ ] SQL/NoSQL queries parameterized; identifiers allowlisted; input types coerced (no `$`-operator injection).
- [ ] No shell command built from input — `execFile`/spawn with an arg array; args allowlisted.
- [ ] File paths resolved and confirmed within the intended base dir; no `..` / absolute / NUL.
- [ ] No deserialization of untrusted data with code-capable formats (pickle, Java native, `yaml.load`).
- [ ] Regexes on user input are linear / length-bounded (no ReDoS).
- [ ] Only an explicit allowlist of fields bound from the request body (no mass assignment).
- [ ] Body size and content-type limits enforced.

## Output encoding / XSS (A03)
- [ ] User data rendered as text by default (framework auto-escaping intact).
- [ ] Any `dangerouslySetInnerHTML` / `v-html` / `bypassSecurityTrust*` / `innerHTML` is justified and sanitized (DOMPurify + allowlist).
- [ ] Output encoded for its context (HTML body / attribute / JS / URL); `href`/`src` schemes validated.
- [ ] Content-Security-Policy present (no `unsafe-inline`).

## SSRF & outbound requests (A10)
- [ ] Server-side fetches validate the URL against a host allowlist and https scheme.
- [ ] Private / link-local / loopback / metadata IP ranges blocked (validate the resolved IP).
- [ ] Redirects refused or re-validated.

## Secrets (A02 / A05)
- [ ] No hardcoded credentials/keys/tokens; secrets read from env / secret manager at runtime (fail fast if missing).
- [ ] No secret reachable from client code or a `NEXT_PUBLIC_/VITE_/REACT_APP_` var.
- [ ] No secrets in logs, error responses, URLs, or process args (logger redaction in place).
- [ ] Any exposed secret flagged for **rotation** (not just removal) + history scrub.
- [ ] Secrets are high-entropy and rotatable.

## Cryptography (A02)
- [ ] TLS enforced for data in transit; no plaintext sensitive data at rest.
- [ ] Randomness for tokens/keys uses a CSPRNG (`crypto.randomBytes`, not `Math.random`).
- [ ] Authenticated encryption (GCM/AEAD); no ECB; no home-rolled crypto.

## Dependencies & supply chain (A06 / A08)
- [ ] No dependency (direct or transitive) with a known CVE; lockfile committed and pinned.
- [ ] New package isn't a typosquat; internal names scoped + on a private registry (no confusion).
- [ ] Added deps' install/lifecycle scripts reviewed; none surprising.
- [ ] Third-party CI actions pinned to a commit SHA, not a mutable tag.
- [ ] External `<script>`/CDN assets carry SRI hashes (or are self-hosted).
- [ ] No secrets exposed to fork/untrusted CI runs.

## Configuration & logging (A05 / A09)
- [ ] Debug mode off in prod; no stack traces / internal detail in client responses.
- [ ] Security headers set (CSP, HSTS, `X-Content-Type-Options`, frame-ancestors); CORS not `*` with credentials.
- [ ] Security-relevant events logged (auth success/failure, authz denials, admin actions) without secrets/PII.

## Verdict
- [ ] Every 🔴/🟠 finding has a concrete remediation.
- [ ] Decision recorded: ✅ approve · 💬 approve with comments · 🔁 request changes · ⛔ block.
