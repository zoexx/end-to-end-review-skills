# Dependency & Supply Chain

The code you didn't write is still code you ship. A PR that adds or bumps a dependency, changes a lockfile, edits CI, or pulls an external script is changing your attack surface. Two threats: **known-vulnerable components** (a dep with a public CVE — A06) and **integrity/provenance failures** (you ran code that wasn't what you thought — A08). Each below: the shape in a diff, the attack, the fix, CWE/OWASP id.

CWE refs: CWE-1104/1035 (vulnerable third-party components), CWE-494 (download without integrity check), CWE-829 (untrusted functionality), CWE-345/353 (insufficient integrity verification), CWE-506 (embedded malicious code). OWASP A06 / A08.

---

## 1. Known-vulnerable dependency — CWE-1104 / A06

```json
// ❌ package.json — version with a published CVE / wide range that resolves to one
"lodash": "^4.17.4",
"axios": "0.21.0",
"log4j-core": "2.14.1"
```

**Attack:** The resolved version has a public advisory and often a working PoC — prototype pollution, SSRF, Log4Shell-class RCE. The attacker doesn't research your code; they search your `package-lock.json` for known-bad versions.

**Detect & fix:**
```bash
npm audit --omit=dev          # Node
pip-audit                     # Python
osv-scanner -r .              # multi-ecosystem, queries OSV
# go: govulncheck ./...   |  rust: cargo audit  |  ruby: bundler-audit
```
Bump to the patched version, run the audit clean, **commit the updated lockfile**. Check **transitive** deps too — the vuln is usually several levels down (`npm ls <pkg>` to see who pulls it).

---

## 2. Unpinned versions & lockfile integrity — CWE-494 / A08

```json
// ❌ Floating range, no committed lockfile
"some-lib": "latest"
"other":    "*"
```

**Attack:** "It worked yesterday" — a new release (or a compromised one) is pulled on the next install/build, in CI or on a teammate's machine, with no review. Without a committed lockfile + integrity hashes, builds aren't reproducible and a swapped package goes unnoticed.

**Fix:**
- Commit the lockfile (`package-lock.json`/`yarn.lock`/`pnpm-lock.yaml`/`poetry.lock`/`Cargo.lock`). It pins exact versions **and** integrity hashes.
- Install with the frozen/CI mode so the lockfile is authoritative: `npm ci`, `pnpm i --frozen-lockfile`, `pip install --require-hashes`.
- Avoid `latest`/`*`; pin or use tight ranges and let the lockfile fix the exact version.

---

## 3. Typosquatting & dependency confusion — CWE-829 / A08

```json
// ❌ A name that looks right but isn't
"crossenv": "1.0.0",        // real one is "cross-env"
"reqeusts": "...",          // typo of "requests"
"@yourco/internal-utils"    // also published on the PUBLIC registry by an attacker
```

**Attack:**
- **Typosquatting** — a malicious package with a near-identical name runs hostile code (often in `postinstall`) the moment it's installed.
- **Dependency confusion** — your private package name also exists on the public registry at a higher version; the resolver prefers public and pulls the attacker's copy.

**Fix:** Verify a new dependency before adding it — exact name, publisher/maintainer, download counts, repo, release age, open issues. For internal packages, scope them (`@yourco/…`), host on a private registry, and configure the resolver to never fetch those scopes from public (e.g. scoped registry config / `.npmrc`). Pin and lock.

---

## 4. Malicious / surprising lifecycle scripts — CWE-506 / A08

```json
// ❌ A dependency that runs code on install
"scripts": { "postinstall": "node ./collect.js" }
```

**Attack:** `preinstall`/`install`/`postinstall` scripts run automatically with your user's privileges during `npm install` — in dev and in CI, where secrets and tokens are present. This is the classic exfiltration vector for a compromised package.

**Fix:** Review the install scripts of any added/updated dependency (`npm pack` and read, or inspect on the registry). In CI, install with scripts disabled where feasible (`npm ci --ignore-scripts`) and explicitly allowlist the few that genuinely need them. Be most suspicious of a *minor/patch* bump that newly introduces a lifecycle script.

---

## 5. Unpinned CI actions & third-party CI — CWE-829 / A08

```yaml
# ❌ Mutable tag — the code behind it can change under you
- uses: some-org/some-action@v3
- uses: some-org/deploy@main
```

**Attack:** A tag/branch is mutable; if the action's repo (or its maintainer's account) is compromised, `@v3` silently starts running attacker code **inside your CI**, which holds deploy keys, registry tokens, and cloud credentials — supply-chain compromise of your whole pipeline.

```yaml
# ✅ Pin to an immutable commit SHA (comment the version for humans)
- uses: some-org/some-action@a1b2c3d4e5f6...   # v3.1.0
```

Pin third-party actions to a full commit SHA; keep first-party/official actions reasonably current. Treat `pull_request_target` and self-hosted runners on public repos as high-risk (they expose secrets to fork PRs).

---

## 6. Missing Subresource Integrity on external scripts — CWE-353 / A08

```html
<!-- ❌ Browser runs whatever the CDN serves, unverified -->
<script src="https://cdn.example.com/lib@1.2.3/lib.min.js"></script>
```

**Attack:** If the CDN, its DNS, or the maintainer account is compromised (a real, recurring event), the swapped script runs in every user's browser with full page access — credential/session theft, formjacking.

```html
<!-- ✅ Pin the exact bytes with SRI; fail if they don't match -->
<script src="https://cdn.example.com/lib@1.2.3/lib.min.js"
        integrity="sha384-<base64-hash>" crossorigin="anonymous"></script>
```

Add `integrity` to every external `<script>`/`<link rel=stylesheet>`. Better: self-host critical third-party scripts so the bytes are in your build and reviewed.

---

## 7. CI/CD secret exposure to untrusted builds — A08

**Attack:** A workflow that runs on fork PRs (`pull_request_target`, or a self-hosted runner) with repo secrets in the environment lets an attacker's PR code read those secrets (print them, exfiltrate them). New CI steps that `echo` env, upload artifacts containing logs, or run untrusted PR code with secrets present are the finding.

**Fix:** Don't expose secrets to workflows triggered by untrusted PRs; gate deploy/secret steps to trusted branches or require maintainer approval; use OIDC short-lived credentials over long-lived stored secrets; never echo secrets.

---

## 8. Provenance, signing, and SBOM — CWE-345 / A08

Defense-in-depth to verify *what* you shipped:
- **SBOM** — generate a software bill of materials (CycloneDX/SPDX via `syft`, `cdxgen`, or `npm sbom`) so a future CVE can be matched to your releases.
- **Provenance / signing** — prefer dependencies and build artifacts with verifiable provenance (npm provenance attestations, Sigstore/cosign-signed images, SLSA). Verify signatures of base images and binaries you pull.
- **Reproducible installs** — frozen-lockfile installs everywhere so dev, CI, and prod resolve identically.

---

## 9. Review what a new dependency actually does

Adding a dependency is a trust decision. For a genuinely new package in the diff, sanity-check:
- **Need** — could this be a few lines of first-party code instead? (Small, single-purpose deps with a large transitive tree are a poor trade.)
- **Health** — maintained (recent commits), reasonable adoption, an issue tracker, a real repo matching the published package.
- **Footprint** — its own dependency tree (`npm ls`/the lockfile diff), permissions, lifecycle scripts, and whether it makes network calls or reads the filesystem unexpectedly.
- **License** — compatible with your project.

A PR that adds one direct dep but 40 transitive ones deserves a 🟠 note, not a silent approve.

---

## Review checklist (dependency / supply chain)

- [ ] `npm audit` / `pip-audit` / `osv-scanner` clean for added/changed deps (incl. transitive).
- [ ] Versions pinned; lockfile present, committed, and consistent with the manifest.
- [ ] New package isn't a typosquat; internal names scoped + private (no confusion).
- [ ] Added deps' install/lifecycle scripts reviewed; none surprising.
- [ ] Third-party CI actions pinned to a commit SHA, not a mutable tag.
- [ ] External `<script>`/CDN assets carry SRI hashes (or are self-hosted).
- [ ] No secrets exposed to fork/untrusted CI runs; no secrets echoed.
- [ ] New dependency justified, healthy, and its footprint understood; SBOM/provenance where applicable.
