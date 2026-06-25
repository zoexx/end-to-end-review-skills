# OWASP Top 10 (2021) — how each shows up in a PR

A category-by-category map. For each: the **shape** it takes in a diff, a **❌ vulnerable** snippet, the **attack**, the **✅ fix**, and where to go deeper. CWE ids are the canonical mappings.

Cross-references: deep dives live in `authn-authz.md` (A01, A07), `secrets-management.md` (A02 secrets), `input-validation.md` (A03, A10, deserialization), `dependency-supply-chain.md` (A06, A08).

---

## A01:2021 — Broken Access Control

The most common serious finding. The code knows *who* the user is but doesn't check *what they're allowed to do* — or checks it only in the UI.

### IDOR — object access without ownership check (CWE-639)

```js
// ❌ Returns any order by id — no check that it belongs to the caller
app.get('/api/orders/:id', auth, async (req, res) => {
  const order = await db.order.findById(req.params.id);
  res.json(order);
});
```

**Attack:** Authenticated user requests `/api/orders/1001`, `1002`, ... and reads every customer's order — bulk data exfiltration. Being logged in is not the same as being authorized for *this* object.

```js
// ✅ Scope the lookup to the caller
app.get('/api/orders/:id', auth, async (req, res) => {
  const order = await db.order.findOne({ id: req.params.id, userId: req.user.id });
  if (!order) return res.sendStatus(404);   // 404, not 403 — don't confirm existence
  res.json(order);
});
```

### Missing function-level authz (CWE-862)

```js
// ❌ Admin route protected only by hiding the button in the frontend
app.post('/api/admin/users/:id/delete', auth, deleteUser);
```

**Attack:** Any logged-in user calls the endpoint directly (curl) and deletes accounts. UI hiding is not access control.

```js
// ✅ Enforce server-side, fail closed
app.post('/api/admin/users/:id/delete', auth, requireRole('admin'), deleteUser);
```

→ Full treatment, including privilege escalation via request body and CSRF, in `authn-authz.md`.

---

## A02:2021 — Cryptographic Failures

Sensitive data exposed because it isn't encrypted, is weakly encrypted, or uses bad randomness.

```js
// ❌ Predictable token from a non-cryptographic RNG (CWE-338)
const resetToken = Math.random().toString(36).slice(2);
// ❌ ECB leaks plaintext structure (CWE-327)
const c = crypto.createCipheriv('aes-256-ecb', key, null);
```

**Attack:** `Math.random()` is seeded from predictable state; an attacker predicts the next password-reset token and takes over accounts. ECB encrypts identical blocks identically, leaking structure.

```js
// ✅ CSPRNG + authenticated encryption (GCM)
const resetToken = crypto.randomBytes(32).toString('hex');
const iv = crypto.randomBytes(12);
const c = crypto.createCipheriv('aes-256-gcm', key, iv);   // store iv + auth tag
```

Also check: TLS enforced (no plaintext transport, HSTS), passwords slow-hashed (see `authn-authz.md`), no secrets at rest in plaintext (see `secrets-management.md`), no home-rolled crypto. CWE-326/327/328/759/916.

---

## A03:2021 — Injection

Untrusted data is interpreted as code/query structure. SQL, NoSQL, OS command, LDAP, XPath, template.

```js
// ❌ SQL injection via string concatenation (CWE-89)
db.query(`SELECT * FROM users WHERE email = '${req.query.email}'`);
```

**Attack:** `email = ' OR '1'='1` dumps the table; `'; DROP TABLE users;--` destroys it; UNION-based payloads exfiltrate other tables.

```js
// ✅ Parameterized — data never becomes query structure
db.query('SELECT * FROM users WHERE email = $1', [req.query.email]);
```

→ SQL/NoSQL/command/LDAP/template injection and XSS (a form of injection into the browser) are covered in depth in `input-validation.md`.

---

## A04:2021 — Insecure Design

A flaw in the security design, not a coding bug — no amount of input sanitizing fixes it.

```js
// ❌ No rate limit or lockout on login → unlimited credential guessing
app.post('/login', async (req, res) => { /* check password */ });
```

**Attack:** Credential stuffing / brute force at thousands of req/s; weak passwords fall. Also watch for: business logic that trusts the client (price sent from the browser), missing workflow limits (unlimited coupon redemption).

```js
// ✅ Per-account + per-IP rate limit with backoff, plus secondary controls
app.post('/login', loginRateLimiter, async (req, res) => { /* ... */ });
```

→ Auth-specific design controls (lockout, MFA, enumeration) in `authn-authz.md`.

---

## A05:2021 — Security Misconfiguration

Defaults left on, debug enabled, verbose errors, missing hardening.

```js
// ❌ Stack trace and SQL leaked to the client (CWE-209)
app.use((err, req, res, next) => res.status(500).json({ error: err.stack }));
// ❌ Debug mode in production reveals config, enables debugger consoles
app.set('env', 'development');
```

**Attack:** Error responses reveal file paths, library versions, and query structure that map the app for the next attack; debug consoles (e.g. Werkzeug, Spring) can become RCE.

```js
// ✅ Generic message to client, detail to server-side logs only
app.use((err, req, res, next) => {
  log.error(err);                              // full detail server-side
  res.status(500).json({ error: 'Internal error' });
});
```

Also verify: security headers present (`Content-Security-Policy`, `X-Content-Type-Options: nosniff`, `Strict-Transport-Security`, `X-Frame-Options`/frame-ancestors), CORS not `*` with credentials, default/sample accounts removed, directory listing off, cloud storage buckets not public. CWE-16/209/548.

---

## A06:2021 — Vulnerable and Outdated Components

A dependency (direct or transitive) with a known CVE, or an unpinned/abandoned package.

```json
// ❌ Pulls a version range with a known RCE; lockfile may drift
"lodash": "^4.17.4",
"log4j-core": "2.14.1"
```

**Attack:** The version resolves to one with a public exploit (e.g. prototype pollution, Log4Shell-class RCE). The attacker doesn't need to find a bug — it's already published with a PoC.

```
// ✅ Audit, pin, and update
npm audit --omit=dev        # or: pip-audit / osv-scanner
# bump to a patched version, commit the lockfile
```

→ Detection, pinning, transitive risk, typosquatting, and SBOM in `dependency-supply-chain.md`. CWE-1104/1035.

---

## A07:2021 — Identification and Authentication Failures

Broken login, session, or credential handling.

```js
// ❌ Fast hash for passwords; JWT with no expiry
const hash = crypto.createHash('sha256').update(password).digest('hex');  // CWE-916
jwt.sign({ sub: user.id }, secret);   // no exp → token valid forever
```

**Attack:** A leaked DB of sha256 hashes is cracked at billions/sec on a GPU. A stolen JWT with no `exp` never stops working, so a single XSS or log leak is permanent account access.

```js
// ✅ Slow, salted hash + short-lived tokens
const hash = await argon2.hash(password);
jwt.sign({ sub: user.id }, secret, { expiresIn: '15m', algorithm: 'HS256' });
```

→ JWT pitfalls, session fixation, MFA, account enumeration, rate limiting in `authn-authz.md`. CWE-287/297/384/620.

---

## A08:2021 — Software and Data Integrity Failures

Code or data accepted without verifying its integrity/origin. Insecure deserialization, unsigned updates, untrusted CI inputs, missing SRI.

```html
<!-- ❌ Third-party script with no integrity check (CWE-353) -->
<script src="https://cdn.example.com/widget.js"></script>
```

```python
# ❌ Deserializing untrusted input → arbitrary code execution (CWE-502)
data = pickle.loads(request.body)
```

**Attack:** If the CDN (or its account) is compromised, the unverified script runs in every user's browser. `pickle.loads` on attacker bytes executes arbitrary Python — direct RCE.

```html
<!-- ✅ Pin with Subresource Integrity -->
<script src="https://cdn.example.com/widget.js"
        integrity="sha384-<hash>" crossorigin="anonymous"></script>
```
```python
# ✅ Use a data-only format
data = json.loads(request.body)
```

→ Deserialization detail in `input-validation.md`; SRI, signed builds, CI integrity in `dependency-supply-chain.md`. CWE-345/353/502/494/829.

---

## A09:2021 — Security Logging and Monitoring Failures

Security-relevant events aren't logged — or logs leak sensitive data.

```js
// ❌ No record of authz denial; AND secrets/PII written to logs (CWE-532)
if (!authorized) return res.sendStatus(403);
log.info(`login ${email} pw=${password} token=${req.headers.authorization}`);
```

**Attack:** Without logging auth failures, a breach goes undetected for months. Conversely, passwords/tokens in logs turn a log-store compromise (or a shared log viewer) into a credential leak.

```js
// ✅ Log the event, redact the secrets
if (!authorized) {
  log.warn('authz_denied', { userId: req.user?.id, route: req.path });
  return res.sendStatus(403);
}
log.info('login_success', { userId: user.id });   // no password, no token
```

Log: auth success/failure, authz denials, input-validation failures, admin actions — with enough context to investigate, none of the secrets. CWE-778/532/779.

---

## A10:2021 — Server-Side Request Forgery (SSRF)

The server makes an outbound request to a URL the attacker controls.

```js
// ❌ Fetches whatever URL the user supplies (CWE-918)
const r = await fetch(req.query.url);
res.send(await r.text());
```

**Attack:** Attacker passes `url=http://169.254.169.254/latest/meta-data/iam/...` and the server returns cloud credentials; or `http://localhost:6379` to reach internal services behind the firewall.

```js
// ✅ Allowlist destinations; block private/link-local ranges; no redirects
const allowed = new Set(['api.partner.com']);
const u = new URL(req.query.url);
if (u.protocol !== 'https:' || !allowed.has(u.hostname)) return res.sendStatus(400);
const r = await fetch(u, { redirect: 'error' });
```

→ Full SSRF guidance (DNS rebinding, redirect chasing, IP-range blocking) in `input-validation.md`. CWE-918.

---

## Using this in a review

1. Walk the diff once per relevant category above — most PRs touch 2-4.
2. For each tainted input, trace to its sink and ask which category the sink belongs to (query → A03, outbound fetch → A10, object lookup → A01).
3. Map every finding to a CWE + OWASP id for the report, and pull the remediation from the linked reference guide.
