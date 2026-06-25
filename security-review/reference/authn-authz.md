# Authentication & Authorization

Who you are (authN) vs. what you may do (authZ). Most serious PR findings are authZ: the code authenticates correctly, then fails to check permission for a specific action or object. Each pattern below has the ❌ vulnerable shape, the concrete attack, the ✅ fix, and CWE/OWASP ids.

Rules that override everything:
- **Enforce on the server, at every layer, for every object.** The UI hiding a button is not access control.
- **Authorization data comes from the session/token, never from client input.**
- **Fail closed.** Unknown role, missing record, thrown error → deny.

---

## 1. Missing server-side authorization (trusting the client) — CWE-602 / A01

```js
// ❌ Frontend decides; backend trusts
// client: if (user.isAdmin) showDeleteButton()
app.delete('/api/posts/:id', auth, (req, res) => deletePost(req.params.id));
```

**Attack:** The check exists only in JavaScript the attacker controls. They open devtools or curl the endpoint directly and delete anything. Client-side checks are UX, not security.

```js
// ✅ Authorize on the server before the action
app.delete('/api/posts/:id', auth, requireRole('admin'), (req, res) => deletePost(req.params.id));
```

---

## 2. IDOR — object-level authz missing — CWE-639 / A01

```js
// ❌ Any authenticated user can read any invoice
app.get('/api/invoices/:id', auth, async (req, res) => {
  res.json(await db.invoice.findById(req.params.id));
});
```

**Attack:** Increment `:id` to enumerate and read every user's invoices — data exfiltration. Authentication ≠ authorization for *this* object.

```js
// ✅ Scope every object read/write to the owner
const inv = await db.invoice.findOne({ id: req.params.id, ownerId: req.user.id });
if (!inv) return res.sendStatus(404);   // 404 avoids confirming the id exists
res.json(inv);
```

Apply the same ownership filter to UPDATE/DELETE, and to nested resources (`/users/:uid/cards/:cid` — verify `uid === req.user.id` AND the card belongs to that user). Don't rely on a hard-to-guess id; UUIDs leak via logs, referrers, and shares.

---

## 3. Privilege escalation via role in the request — CWE-269 / A01

```js
// ❌ Role taken from the request body / mass-assigned
await db.user.update({ where: { id: req.user.id }, data: req.body });   // body has role:"admin"
```

**Attack:** User sends `{"role":"admin"}` (or `isAdmin:true`) to their own profile-update endpoint and self-promotes — full takeover. Same bug enables setting `userId`/`tenantId` to act as someone else.

```js
// ✅ Allowlist editable fields; derive privilege server-side only
const { name, email } = req.body;
await db.user.update({ where: { id: req.user.id }, data: { name, email } });
// role changes go through a separate admin-only, authorized endpoint
```

---

## 4. Password storage — CWE-916 / A02 / A07

```js
// ❌ Fast or unsalted hashes — built for speed, terrible for passwords
crypto.createHash('md5').update(pw).digest('hex');     // also sha1, sha256, plain
```

**Attack:** When the DB leaks (and assume it will), md5/sha1/sha256 are cracked at billions of guesses/sec on a GPU; rainbow tables defeat unsalted hashes instantly. Reused passwords then unlock other sites.

```js
// ✅ Slow, salted, memory-hard KDF
const hash = await argon2.hash(pw);          // or bcrypt (cost ≥ 12) / scrypt
const ok   = await argon2.verify(hash, pw);
```

bcrypt, argon2id, scrypt, or PBKDF2 (high iterations) — each salts automatically and is deliberately slow. Never roll your own.

---

## 5. JWT pitfalls — CWE-347 / A02 / A07

```js
// ❌ Accepts whatever algorithm the token claims, incl. "none"
jwt.verify(token, secret);                     // no algorithms pin
jwt.sign({ sub: id }, secret);                 // no exp → never expires
// ❌ weak secret
const secret = 'secret';
```

**Attacks:**
- **`alg:none`** — attacker sends an unsigned token; a verifier that honors the header skips signature checks → forge any identity.
- **`alg` confusion (RS256→HS256)** — attacker signs with the *public* key as an HMAC secret; a lib that picks the alg from the header validates it → forgery.
- **No `exp`** — a token leaked once (XSS, log, proxy) works forever.
- **Weak secret** — brute-forced offline, then mint admin tokens.

```js
// ✅ Pin the algorithm, require expiry, use a strong secret/asymmetric key
jwt.sign({ sub: id }, secret, { algorithm: 'HS256', expiresIn: '15m' });
jwt.verify(token, secret, { algorithms: ['HS256'] });   // explicit allowlist
```

Also: don't put secrets/PII in the (base64, un-encrypted) payload. For session auth, prefer a short-lived token + httpOnly refresh cookie over storing JWTs in `localStorage` (any XSS reads them).

---

## 6. Session fixation & lifecycle — CWE-384 / A07

```js
// ❌ Keeps the pre-login session id after authenticating
req.session.userId = user.id;   // same session id the attacker may have set
```

**Attack:** Attacker plants a known session id (via a link or earlier injection), victim logs in, the id is now authenticated → attacker rides it. Also covers: no logout invalidation, no idle/absolute timeout.

```js
// ✅ Regenerate on privilege change; invalidate on logout
req.session.regenerate(() => { req.session.userId = user.id; });
// logout:
req.session.destroy();   // server-side, not just clearing the cookie
```

Set idle timeout (e.g. 30 min) and an absolute cap; invalidate server-side, not by trusting the client to discard the cookie.

---

## 7. Insecure cookie flags — CWE-1004/614/1275 / A05

```js
// ❌ Session cookie readable by JS and sent over http and cross-site
res.cookie('session', id);
```

**Attack:** No `HttpOnly` → any XSS steals the session via `document.cookie`. No `Secure` → it leaks over plaintext HTTP. No `SameSite` → it's sent on cross-site requests, enabling CSRF.

```js
// ✅
res.cookie('session', id, { httpOnly: true, secure: true, sameSite: 'lax', path: '/' });
```

Use `sameSite:'strict'` for sensitive apps; `'lax'` is the safe default. Scope `path`/`domain` tightly.

---

## 8. Missing CSRF protection — CWE-352 / A01

```html
<!-- ❌ State-changing POST authenticated only by an ambient cookie -->
<form action="/account/email" method="post"> ... </form>
```

**Attack:** With cookie-based auth and no CSRF defense, an attacker's page auto-submits a form to your endpoint; the browser attaches the victim's cookie and the action executes as them (change email → password reset → takeover).

```js
// ✅ Anti-CSRF token (synchronizer pattern) + SameSite cookies
app.post('/account/email', csrfProtection, updateEmail);   // verifies per-session token
```

Token-in-`Authorization`-header APIs (not cookie-auth) are largely immune. Add `SameSite` as defense-in-depth. Verify the protection covers *every* state-changing method, not just some.

---

## 9. No rate limiting / lockout on auth — CWE-307 / A07

```js
// ❌ Unlimited password attempts
app.post('/login', verifyPassword);
```

**Attack:** Credential stuffing and brute force at high speed; also enables MFA-code and reset-token guessing.

```js
// ✅ Per-account + per-IP limiter with exponential backoff and lockout
app.post('/login', loginLimiter, verifyPassword);
```

Apply to login, password reset, MFA verification, and token endpoints. Lock or delay after N failures; alert on spikes.

---

## 10. Account enumeration via differential responses — CWE-204 / A07

```js
// ❌ Different message/timing reveals which emails are registered
if (!user) return res.status(404).json({ error: 'No such user' });
if (!ok)   return res.status(401).json({ error: 'Wrong password' });
```

**Attack:** Attacker harvests valid accounts by comparing messages, status codes, or response timing, then targets them for stuffing/phishing. Same leak on signup ("email already in use") and password reset.

```js
// ✅ Identical response regardless of which factor failed
if (!user || !ok) return res.status(401).json({ error: 'Invalid credentials' });
// password reset: always "if that email exists, we sent a link"
```

Keep timing constant (do a dummy hash verify when the user is missing) to avoid timing-based enumeration.

---

## 11. MFA

Check, where present: MFA enforced *server-side* on the sensitive action (not skippable by calling a later endpoint directly); codes are single-use, short-lived, and rate-limited; backup codes hashed; "remember this device" tokens are bound and revocable. Missing MFA on admin/high-value accounts is at least 🟠.

---

## 12. OAuth / OIDC — CWE-601 / A01

```js
// ❌ redirect_uri not validated against a registered allowlist
res.redirect(req.query.redirect_uri + '?code=' + code);
```

**Attack:** Open-redirect on the OAuth callback lets the attacker register/inject a `redirect_uri` they control and capture the auth `code`/token → account takeover. Missing `state` enables CSRF on the login flow.

```js
// ✅ Exact-match the redirect against pre-registered URIs; require & verify state
if (!REGISTERED_URIS.has(req.query.redirect_uri)) return res.sendStatus(400);
if (req.query.state !== session.oauthState) return res.sendStatus(400);
```

Exact match (scheme + host + path), not prefix/substring; verify `state`; use PKCE for public clients.

---

## Review checklist (authn/authz)

- [ ] Every protected route has a server-side authz check; fails closed.
- [ ] Object reads/writes verify ownership (no IDOR); 404 on miss.
- [ ] Role/permission/tenant comes from session/token, never request body.
- [ ] Passwords hashed with bcrypt/argon2/scrypt (salted, slow).
- [ ] JWT: alg pinned, `exp` set & checked, strong secret, not in localStorage.
- [ ] Session id regenerated on login; invalidated on logout; timeouts set.
- [ ] Cookies: HttpOnly, Secure, SameSite.
- [ ] CSRF protection on all cookie-authenticated state changes.
- [ ] Auth endpoints rate-limited with lockout.
- [ ] Login/reset/signup responses don't enumerate accounts.
- [ ] OAuth redirect_uri allowlisted; state verified.
