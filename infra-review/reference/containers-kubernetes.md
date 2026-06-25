# Containers & Kubernetes

Deep-dive for reviewing Dockerfiles and Kubernetes manifests. A container image is the unit you ship to prod, and a K8s manifest decides how it runs, what it can reach, and what happens when it misbehaves. The failures here are concrete: a root container that becomes a host breakout, a pod with no limits that OOM-kills its neighbors, a missing probe that routes traffic to a dead process, a secret baked into a layer that's now in your registry forever. Each section pairs a ❌ anti-pattern with a ✅ fix and names the incident it causes.

Split into **Dockerfile** (§1–5) and **Kubernetes** (§6–12).

---

## 1. Running as root

By default a container runs as root (UID 0). If anything in it is exploited, the attacker is root inside the container — and with a kernel or runtime escape, root on the node.

❌ No user → runs as root:
```dockerfile
FROM node:20
COPY . /app
CMD ["node", "server.js"]        # ❌ process is root (UID 0)
```
✅ Create and drop to a non-root user:
```dockerfile
FROM node:20.11.1-slim
RUN useradd -r -u 10001 app
WORKDIR /app
COPY --chown=app:app . .
USER 10001                       # ✅ runs as non-root
CMD ["node", "server.js"]
```
> Incident: an RCE in your app gives the attacker root inside the container; from there a container-escape CVE (they appear regularly) makes them root on the node and then the cluster. A non-root user with no escalation path turns the same RCE into a much smaller foothold. Run as a fixed non-root UID and enforce it again in the pod `securityContext` (§8).

---

## 2. Unpinned / `:latest` base image

`FROM image:latest` (or a floating major) means your build isn't reproducible and you can't tell which base — with which CVEs — you actually shipped.

❌ Floating base:
```dockerfile
FROM python:latest               # ❌ which Python? which patch level? changes under you
```
✅ Pin to an exact version + digest:
```dockerfile
FROM python:3.12.4-slim@sha256:abc123…   # exact version, immutable digest
```
> Incident: `:latest` rolls to a new base with a regression (or a new CVE) and your next build ships it without anyone choosing to; and when a vuln scanner flags "the python base," you can't tell which image is affected because the tag has moved many times. Pin to a digest so the base is reproducible and auditable, and bump it deliberately through review.

---

## 3. Secrets baked into image layers

Every `RUN`, `COPY`, and `ARG` is a layer in the image history. A secret used at build time stays in those layers even if a later step "removes" it — anyone who pulls the image can extract it.

❌ Secret in a layer:
```dockerfile
ARG NPM_TOKEN
RUN npm config set //registry.npmjs.org/:_authToken=$NPM_TOKEN   # ❌ token in this layer forever
COPY .env /app/.env                                              # ❌ secrets shipped in the image
```
✅ Build-time secret mount (not persisted); never COPY secret files:
```dockerfile
RUN --mount=type=secret,id=npm_token \
    NPM_TOKEN=$(cat /run/secrets/npm_token) npm ci      # ✅ mount exists only during this RUN
# runtime config/secrets are injected by the orchestrator, never copied in (see §11)
```
> Incident: a build does `ARG GITHUB_TOKEN` / `COPY .env` to install private deps; the token is now in a layer of every image pushed to your registry, readable by anyone with pull access via `docker history` / layer extraction — a leak that survives "deleting" the line because it lives in an earlier layer. Use BuildKit secret mounts for build-time secrets and inject runtime secrets from the orchestrator; add a `.dockerignore` (see §5) so `.env` can't be copied in by accident.

---

## 4. Single-stage bloat — no multi-stage build

Shipping the full build toolchain (compilers, dev deps, source) in the runtime image bloats it and hugely widens the attack surface — every extra binary is a potential exploit primitive.

❌ One stage, everything shipped:
```dockerfile
FROM golang:1.22
COPY . .
RUN go build -o app .            # ❌ runtime image has the whole Go toolchain + source
CMD ["./app"]
```
✅ Multi-stage: build fat, ship thin:
```dockerfile
FROM golang:1.22 AS build
COPY . .
RUN CGO_ENABLED=0 go build -o /app .
FROM gcr.io/distroless/static:nonroot   # ✅ minimal runtime, no shell, non-root
COPY --from=build /app /app
USER nonroot
ENTRYPOINT ["/app"]
```
> Incident: a 900 MB image carrying `gcc`, `git`, package managers, and your source ships to prod; when one dependency has an RCE, the attacker has a full toolchain to pivot with — and your pull times and registry bill are larger for it. A multi-stage build with a distroless/minimal final stage cuts the image to the binary plus its runtime, removing the shell and tools an attacker would use.

---

## 5. Missing `.dockerignore`, `ADD` instead of `COPY`, no `HEALTHCHECK`

Three smaller Dockerfile hazards that bite in production.

❌ Build context and instruction hazards:
```dockerfile
# no .dockerignore → COPY . . pulls in .git, .env, node_modules, secrets  ❌
ADD https://example.com/installer.sh /tmp/   # ❌ ADD fetches remote URLs / auto-extracts tarballs
# no HEALTHCHECK → orchestrator can't tell a hung process from a healthy one
```
✅ Scope the context, use `COPY`, declare health:
```dockerfile
# .dockerignore: .git, .env*, node_modules, *.pem, dist, coverage
COPY package*.json ./           # COPY is explicit and local-only
RUN npm ci
COPY . .
HEALTHCHECK --interval=30s --timeout=3s CMD curl -fsS http://localhost:8080/healthz || exit 1
```
> Incident: with no `.dockerignore`, `COPY . .` sweeps `.git` (full history, possibly old secrets) and a local `.env` into the image; meanwhile `ADD <url>` silently re-downloads a remote script whose contents can change, and with no `HEALTHCHECK` a wedged process keeps receiving traffic. A `.dockerignore` keeps junk and secrets out, `COPY` keeps the instruction predictable, and a `HEALTHCHECK` lets the platform detect a hung container. (In K8s, prefer pod probes — §7.)

---

## 6. No resource requests and limits

Without `requests` and `limits`, a pod is scheduled blindly and can consume an entire node's CPU/memory, starving or OOM-killing every other pod on it — the classic noisy-neighbor outage.

❌ Unbounded container:
```yaml
containers:
  - name: api
    image: api@sha256:…
    # ❌ no resources → can eat the whole node; scheduler can't bin-pack safely
```
✅ Set requests (scheduling) and limits (cap):
```yaml
resources:
  requests: { cpu: "250m", memory: "256Mi" }   # guaranteed floor → correct scheduling
  limits:   { cpu: "1",    memory: "512Mi" }    # hard ceiling → OOM-kills only THIS pod
```
> Incident: a memory leak (or a big request) makes one pod balloon; with no `limits` it grows until the **node** runs out of memory and the kernel OOM-killer reaps *other* tenants' pods — one bad deploy takes down unrelated services. `requests` let the scheduler place pods correctly; `limits` contain a misbehaving pod to itself. Set both for cpu and memory; watch that the limit is realistic so you don't throttle/OOM healthy traffic.

---

## 7. Missing liveness / readiness / startup probes

Without probes, Kubernetes assumes "running = healthy." A hung process keeps getting traffic; a slow-starting one gets traffic before it's ready; a deadlocked one never restarts.

❌ No probes:
```yaml
containers:
  - name: api
    image: api@sha256:…
    # ❌ no readiness → traffic routed before ready; no liveness → hung pod never restarts
```
✅ Readiness (traffic gate) + liveness (restart) + startup (slow boot):
```yaml
readinessProbe: { httpGet: { path: /readyz, port: 8080 }, periodSeconds: 5 }   # gate traffic
livenessProbe:  { httpGet: { path: /livez,  port: 8080 }, periodSeconds: 10, failureThreshold: 3 }
startupProbe:   { httpGet: { path: /livez,  port: 8080 }, failureThreshold: 30, periodSeconds: 5 }  # slow boot
```
> Incident: a deploy rolls out; pods report `Running` but the app is still warming caches, so the Service sends traffic and users get 500s — a readiness probe would have held traffic until `/readyz` passed. Separately, a pod deadlocks and serves errors forever because nothing restarts it — a liveness probe would have. Keep liveness cheap and dependency-free (don't fail liveness because a *downstream* is down, or you'll crash-loop the whole fleet); put dependency checks in readiness; use a startup probe for slow boots so liveness doesn't kill them mid-start.

---

## 8. No `securityContext` / `runAsNonRoot`

Even a non-root image should be locked down at the pod level: prevent privilege escalation, drop Linux capabilities, make the root filesystem read-only.

❌ Default-permissive container:
```yaml
containers:
  - name: api
    image: api@sha256:…
    # ❌ may run as root, can escalate, has default caps, writable rootfs
```
✅ Hardened securityContext:
```yaml
securityContext:
  runAsNonRoot: true
  runAsUser: 10001
  allowPrivilegeEscalation: false
  readOnlyRootFilesystem: true        # writable paths via emptyDir volumes
  capabilities: { drop: ["ALL"] }     # add back only what's truly needed
  seccompProfile: { type: RuntimeDefault }
```
> Incident: an exploited pod runs as root with the default capability set and a writable filesystem, so the attacker installs tools, writes a cron persistence hook, and probes the node — all things `runAsNonRoot`, `drop: ["ALL"]`, and `readOnlyRootFilesystem` would have blocked or hugely slowed. Set a restrictive `securityContext` on every workload; enforce it cluster-wide with Pod Security Admission (`restricted`).

---

## 9. `privileged`, `hostPath`, `hostNetwork`, `hostPID`

These break the container boundary: a privileged pod is effectively root on the node; `hostPath` mounts the node's filesystem; `hostNetwork`/`hostPID` expose the node's network/process namespace.

❌ Container with host-level access:
```yaml
securityContext: { privileged: true }     # ❌ ≈ root on the node
volumeMounts: [{ name: host, mountPath: /host }]
volumes: [{ name: host, hostPath: { path: / } }]   # ❌ mounts the node's whole filesystem
hostNetwork: true                          # ❌ shares the node's network namespace
```
✅ Stay inside the sandbox; if you truly need host access, scope it tightly:
```yaml
# normal workloads: none of the above. For a genuine node agent (CSI/CNI/monitoring),
# scope hostPath to a single read-only path and justify it explicitly.
volumes: [{ name: sock, hostPath: { path: /var/run/agent.sock, type: Socket } }]
```
> Incident: an app pod is given `privileged: true` "to make a sysctl work," and a later compromise of that pod is a compromise of the node and every pod scheduled on it — full cluster takeover from one app bug. `hostPath: /` mounted read-write lets a pod rewrite the kubelet config or read other tenants' data. Reserve host access for genuine infrastructure agents, scope it to the minimum path, and block it for app workloads via policy.

---

## 10. No rolling-update strategy / PodDisruptionBudget

Without a controlled update strategy and a disruption budget, a deploy or a node drain can take down all replicas of a service at once.

❌ No strategy / no PDB:
```yaml
# Deployment with default-but-unbounded surge, and NO PodDisruptionBudget
# → a node drain can evict every replica simultaneously  ❌
```
✅ Bounded rolling update + a PDB:
```yaml
spec:
  strategy:
    type: RollingUpdate
    rollingUpdate: { maxUnavailable: 0, maxSurge: 1 }   # never drop below desired during rollout
---
apiVersion: policy/v1
kind: PodDisruptionBudget
spec: { minAvailable: 2, selector: { matchLabels: { app: api } } }   # drains respect this
```
> Incident: a cluster autoscaler scales down (or an admin drains a node for patching) and, with no PDB, evicts all three replicas of a service that happened to land on that node — instant outage during a routine, planned operation. A PDB makes voluntary disruptions (drains, autoscaler) keep `minAvailable` pods up; a `maxUnavailable: 0` rolling update keeps capacity during deploys. Set both for anything that must stay available.

---

## 11. Secrets as plaintext env in the manifest

Putting secret values directly in a manifest (or a `ConfigMap`) commits them to git and prints them in `kubectl describe`. A K8s `Secret` is base64, not encryption — but at least it's the right object to wire to a real secret store.

❌ Secret value in the manifest:
```yaml
env:
  - name: DB_PASSWORD
    value: "Pa55w0rd!"        # ❌ committed to git, visible in `describe`, in pod spec
```
✅ Reference a Secret sourced from a real store:
```yaml
env:
  - name: DB_PASSWORD
    valueFrom: { secretKeyRef: { name: db-creds, key: password } }   # not in the manifest
# db-creds is populated by External Secrets / Sealed Secrets / CSI from Vault/SSM,
# and etcd encryption-at-rest is enabled. (Secret-store depth → security-review.)
```
> Incident: a `DB_PASSWORD: "..."` literal is committed in a manifest; now it's in git history forever and printed by `kubectl describe pod`, so anyone with repo or namespace read has the prod DB password. Reference a `Secret` object instead, populate that Secret from a real manager (External Secrets Operator, Sealed Secrets, Secrets Store CSI), and enable etcd encryption-at-rest so the base64 in etcd isn't the only barrier. (Cross-ref **security-review**.)

---

## 12. RBAC too broad / no NetworkPolicy

A workload's ServiceAccount should grant the minimum API access it needs; the default network model is flat (any pod can reach any pod), so without NetworkPolicies a compromised pod can talk to everything.

❌ Cluster-admin to a workload, open network:
```yaml
kind: ClusterRoleBinding
roleRef: { kind: ClusterRole, name: cluster-admin }   # ❌ the app can do anything in the cluster
subjects: [{ kind: ServiceAccount, name: api }]
# and no NetworkPolicy → every pod can reach every other pod and all egress
```
✅ Least-privilege Role + default-deny network:
```yaml
kind: Role            # namespaced, minimal verbs/resources
rules: [{ apiGroups: [""], resources: ["configmaps"], verbs: ["get","list"] }]
---
kind: NetworkPolicy   # default-deny, then allow only needed ingress/egress
spec: { podSelector: { matchLabels: { app: api } }, policyTypes: [Ingress, Egress],
        ingress: [{ from: [{ podSelector: { matchLabels: { app: gateway } } }] }] }
```
> Incident: an app pod is bound to `cluster-admin` so it can read one ConfigMap; when that pod is compromised, the attacker uses its token to create pods, read every Secret, and pivot across namespaces — full cluster compromise from one app bug. And with no NetworkPolicy, a breached front-end pod can reach the database and every internal service directly. Grant a namespaced `Role` with exactly the verbs needed, and apply a default-deny `NetworkPolicy` so lateral movement is blocked by default.

---

## Quick scan checklist

**Dockerfile**
- [ ] Runs as a **non-root** `USER`; not UID 0.
- [ ] Base image pinned to an exact version + **digest** — no `:latest`.
- [ ] No secrets in `ARG`/`COPY`/layers; build secrets via BuildKit `--mount=type=secret`.
- [ ] **Multi-stage** build; runtime stage is minimal/distroless, no build toolchain.
- [ ] `.dockerignore` excludes `.git`/`.env`/secrets; `COPY` over `ADD`; a `HEALTHCHECK` (or rely on pod probes).
- [ ] Image scanning runs in CI and fails on high-severity CVEs.

**Kubernetes**
- [ ] Both `resources.requests` and `resources.limits` set for cpu and memory.
- [ ] Liveness + readiness (+ startup for slow boot) probes; liveness has no downstream dependency.
- [ ] `securityContext`: `runAsNonRoot`, `allowPrivilegeEscalation: false`, `readOnlyRootFilesystem`, `drop: ["ALL"]`.
- [ ] No `privileged`/`hostPath`/`hostNetwork`/`hostPID` on app workloads (scoped + justified for true node agents).
- [ ] Image referenced by **digest**, not `:latest`.
- [ ] Rolling-update strategy bounds unavailability; a `PodDisruptionBudget` protects voluntary disruptions.
- [ ] Secrets referenced via `secretKeyRef` from a real store — no plaintext values in manifests; etcd encryption on.
- [ ] ServiceAccount RBAC is a namespaced least-privilege `Role`, not `cluster-admin`.
- [ ] A default-deny `NetworkPolicy` exists; ingress/egress opened only as needed.
