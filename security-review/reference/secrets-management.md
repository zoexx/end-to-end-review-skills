# Secrets Management

A secret is anything that grants access if leaked: passwords, API keys, OAuth client secrets, DB connection strings, private/signing keys, session/JWT secrets, encryption keys, webhook signing secrets. The review question is always: **could an attacker — or an unauthorized insider — read this?** Source, client bundle, logs, error responses, URLs, process args, and git history are all "readable."

Never echo a real secret's full value in a finding. If you find one, the remediation is **rotate it and scrub history** — removing the line is not enough; it's already public.

CWE refs: CWE-798 (hardcoded), CWE-312 (cleartext storage), CWE-532 (info in logs), CWE-200 (exposure), CWE-321 (hardcoded crypto key). OWASP A02 (cryptographic failures) / A05 (misconfiguration).

---

## 1. Hardcoded credentials in source — CWE-798

```js
// ❌
const stripe = new Stripe('sk_live_4eC39H...REDACTED');
const db = mysql.createConnection({ host, user: 'root', password: 'Pa55w0rd!' });
const JWT_SECRET = 'supersecret';
```

**Attack:** Anyone with repo read access — every employee, every contractor, every fork, every CI log, and anyone if the repo ever goes public — has the live key. Hardcoded keys also can't be rotated without a code change/deploy.

```js
// ✅ Load from the environment at runtime; fail fast if absent
const stripe = new Stripe(requireEnv('STRIPE_SECRET_KEY'));
function requireEnv(k){ const v = process.env[k]; if(!v) throw new Error(`missing ${k}`); return v; }
```

Even a "test"/"dev" key is a finding if it's live; even an internal-only key is a finding (defense in depth, insider risk).

---

## 2. Secrets shipped to the client — CWE-200

```js
// ❌ Anything bundled into frontend code is public
const KEY = process.env.SECRET_API_KEY;          // webpack inlines it
// ❌ Build-tool "public" prefixes are PUBLIC by design
NEXT_PUBLIC_STRIPE_SECRET=sk_live_...
VITE_API_SECRET=...
REACT_APP_PRIVATE_KEY=...
```

**Attack:** The frontend bundle is downloaded by every visitor; "View Source" / devtools reveals the key. `NEXT_PUBLIC_`, `VITE_`, `REACT_APP_` (and similar) are explicitly exposed to the browser — a secret behind such a prefix is leaked to every user.

```js
// ✅ Keep the secret server-side; the client calls your backend, which calls the API
// browser → POST /api/charge → server uses STRIPE_SECRET_KEY (never sent to browser)
```

Only *publishable*/public keys (designed to be exposed, e.g. `pk_live_…`, a Maps client key with referrer restrictions) belong in client code. Check source maps don't ship server bundles.

---

## 3. Secrets in logs and error responses — CWE-532 / CWE-209

```js
// ❌ Logs the full request including auth header & body password
log.info('incoming', { headers: req.headers, body: req.body });
// ❌ Returns the connection string / token in an error to the client
res.status(500).json({ error: err.message });   // "ECONNREFUSED postgres://user:pw@..."
```

**Attack:** Logs flow to shared dashboards, third-party log SaaS, and long-term storage; a single log breach (or a curious teammate) yields live credentials. Error messages echoed to clients leak connection strings, keys, and tokens directly to the attacker.

```js
// ✅ Redact before logging; generic errors to clients
log.info('incoming', { method: req.method, path: req.path, userId: req.user?.id });
res.status(500).json({ error: 'Internal error' });   // detail stays in server logs (redacted)
```

Add a logger redactor for `authorization`, `cookie`, `password`, `token`, `secret`, `api_key`, `set-cookie`.

---

## 4. Committed secret files — CWE-312 / CWE-540

```
# ❌ committed to the repo
.env
config/credentials.yml.enc.key
id_rsa            private SSH key
gcp-service-account.json
*.pem / *.p12 keystores
```

**Attack:** These hold live secrets and live forever in git history even after deletion. A leaked `.env` or `id_rsa` is immediate, full access to whatever it unlocks.

**Fix:**
- Add to `.gitignore`; commit `.env.example` with **placeholder** values only.
- Load real values from a secret manager / CI secret store at runtime.
- If already committed: **rotate the secret** (it's compromised), then scrub history (`git filter-repo` / BFG) and force-push, coordinating with the team. Removing the file in a new commit does **not** remove it from history.

---

## 5. Long-lived static credentials & no rotation

```js
// ❌ A permanent API token with full scope, no expiry, used everywhere
const TOKEN = 'permanent-admin-token';
```

**Attack:** A static long-lived credential maximizes blast radius — one leak compromises everything until someone notices, and there's no automatic cutoff.

**Fix:** Prefer short-lived, automatically-rotated credentials (OIDC workload identity, STS/assumed roles, time-boxed tokens). Scope each credential to least privilege. Where static keys are unavoidable, document and automate rotation, and ensure the code reads the *current* value at runtime (not a baked-in copy) so rotation doesn't require a redeploy.

---

## 6. Weak / predictable secrets — CWE-330 / CWE-321

```js
// ❌ Guessable or hardcoded crypto material
const JWT_SECRET = 'changeme';
const key = 'aaaaaaaaaaaaaaaa';                  // hardcoded AES key
```

**Attack:** Short/dictionary secrets are brute-forced offline; an attacker then forges valid JWTs or decrypts data. A hardcoded encryption key means every install shares it.

```js
// ✅ High-entropy, generated, externally managed
// JWT_SECRET = output of: openssl rand -hex 32   (stored in secret manager)
```

Signing/JWT secrets ≥ 256 bits of entropy from a CSPRNG. Encryption keys generated and stored in a KMS, never in code.

---

## 7. Secrets on the command line / in process args — CWE-214

```bash
# ❌ Visible to any local user via `ps`, and often captured in shell history & CI logs
curl -H "Authorization: Bearer $(cat token)" ...   # ok
mytool --api-key=sk_live_abc123 ...                 # ❌ leaks in `ps aux` and CI output
```

**Attack:** Process arguments are world-readable on most systems (`ps`, `/proc`); CI consoles and shell history capture them.

```bash
# ✅ Pass via environment variable or a file read at runtime
API_KEY="$API_KEY" mytool          # tool reads from env, not argv
```

---

## Remediation toolkit

- **Storage:** a secret manager / vault — HashiCorp Vault, AWS/GCP/Azure Secret Manager, Doppler, 1Password; or the platform's CI secret store. App reads them as env at runtime.
- **Detection in the pipeline:** pre-commit secret scanning (`gitleaks`, `trufflehog`, `detect-secrets`) and a CI secret-scan gate so the next one is caught automatically.
- **On exposure:** rotate first (assume compromised), then remove from history (`git filter-repo`/BFG + force push), then investigate access logs for misuse.
- **Hygiene:** `.env` git-ignored; `.env.example` with placeholders; least-privilege scoping; documented/automated rotation; logger redaction for secret-shaped keys.

## Review checklist (secrets)

- [ ] No hardcoded keys/passwords/tokens/private keys in the diff.
- [ ] No secret reachable from client code or a `*_PUBLIC_/VITE_/REACT_APP_` var.
- [ ] No secrets in logs, error responses, URLs, or process args.
- [ ] No `.env`/`id_rsa`/`*.pem`/keystore added; `.env.example` is placeholders only.
- [ ] Secrets loaded from env/secret manager at runtime; missing → fail fast.
- [ ] Secrets are high-entropy and rotatable; any exposed one is flagged for rotation.
