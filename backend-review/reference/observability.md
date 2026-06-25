# Observability

Deep-dive for reviewing whether a service can be *understood in production*. Observability isn't a feature you add later — it's how the on-call engineer figures out what's broken at 3am. Code that can't be observed is code that can't be operated. Each section pairs a ❌ anti-pattern with a ✅ fix and names the incident it prevents (or fails to detect).

The three pillars: **logs** (discrete events), **metrics** (aggregates over time), **traces** (one request across services). A mature path emits all three.

---

## 1. Unstructured / print logging

A log you can't query is a log you can't use during an incident.

❌ String-concatenated, unstructured:
```python
print("user " + user_id + " did " + action + " at " + str(time.time()))   # ❌
log.info(f"processing order {order_id} failed: {err}")                      # ❌ unparseable
```
✅ Structured key-value (machine-queryable):
```python
log.info("order.process.failed",
         order_id=order_id, user_id=user_id, error=str(err), duration_ms=dur)
```
> Incident: during an outage you need "all failed orders for customer X in the last 10 min." With `print` you're grepping freeform strings and writing fragile regexes; with structured logs it's a one-line query. The unstructured choice costs you minutes you don't have mid-incident.

Use the platform's structured logger (pino/winston, structlog, zap/zerolog, slog) and log JSON in production.

---

## 2. Logging secrets / PII

A log line is forever and widely readable. A secret or PII in it is a breach, a compliance violation, and often un-deletable once shipped to a log aggregator.

❌ Leaking sensitive data:
```ts
log.info("login attempt", { email, password: req.body.password });   // ❌ password in logs
log.debug("charging card", { cardNumber, cvv });                     // ❌ PCI violation
log.info("request", { headers: req.headers });                       // ❌ Authorization/Cookie leak
```
✅ Redact / omit / hash:
```ts
log.info("login attempt", { email: maskEmail(email) });              // u***@d***.com
log.debug("charging card", { last4: cardNumber.slice(-4) });          // never full PAN/CVV
log.info("request", { headers: redact(req.headers, ["authorization", "cookie"]) });
```
> Incident: passwords/tokens/card numbers land in a log index that contractors, analytics pipelines, and third-party log SaaS can read. Now it's a reportable breach and the token must be rotated everywhere — caused by one careless `log({ ...req.body })`.

Establish a redaction allow/deny list at the logger level so it can't be forgotten per-call site. (Auth-token handling depth → **security-review**.)

---

## 3. No correlation / trace IDs

Without an id that ties a request's logs together across services, you can't follow one failure through the system.

❌ Logs with no correlating id:
```ts
log.info("payment failed");   // ❌ which request? which user? which downstream call?
```
✅ Propagate a correlation/trace id end to end:
```ts
// at the edge: adopt incoming W3C traceparent or mint one
const traceId = req.header("traceparent") ?? newTraceId();
req.context = { traceId };
log.info("payment.failed", { traceId, orderId });            // every log carries it
await downstream.post("/charge", body, { headers: { traceparent: traceId } }); // pass it on
```
> Incident: a payment fails somewhere across api-gateway → orders → payments → bank-adapter. Without a shared id you can't stitch the four services' logs together; you're guessing which log lines belong to the same request and the investigation takes hours instead of one query: `traceId = "abc"`.

Surface the trace id in error responses too (see api-design §4) so support can jump straight to the logs.

---

## 4. Log level misuse

Levels are how you separate signal from noise. Misused, they either page you for nothing or hide the real failure.

❌ Wrong levels:
```ts
log.error("user not found");          // ❌ a normal 404 — not an error, will trip alerts
log.info("DB connection pool exhausted"); // ❌ a real incident logged as info — invisible
for (const row of millionRows) log.debug("row", row);  // ❌ floods logs, may even be on in prod
```
✅ Level matches operational meaning:
```ts
log.info("user.lookup.miss", { id });           // expected outcome
log.error("db.pool.exhausted", { active, max }); // page-worthy: needs human action
// hot-loop detail at debug, off in prod; sample if you must keep some
```
> Incident: `error` on every 404 makes the error-rate dashboard useless and the team mutes the alert — so when a *real* error spike happens, nobody notices. Meanwhile the actual pool-exhaustion incident was logged at `info` and scrolled past unseen.

Guide: `error` = needs action / broke a request · `warn` = degraded but handled · `info` = notable lifecycle events · `debug` = developer detail, off in prod.

---

## 5. Missing metrics on key paths

You can't alert on, or capacity-plan for, what you don't measure. Logs answer "what happened to this request"; metrics answer "how is the system doing right now."

❌ Critical path emits nothing measurable:
```ts
app.post("/checkout", async (req, res) => {
  const r = await checkout(req.body);   // ❌ no count, no latency, no error rate
  res.json(r);
});
```
✅ Emit RED on every endpoint, USE on every resource:
```ts
app.post("/checkout", async (req, res) => {
  const end = checkoutLatency.startTimer({ route: "checkout" });
  try {
    const r = await checkout(req.body);
    checkoutTotal.inc({ route: "checkout", status: "ok" });
    res.json(r);
  } catch (e) {
    checkoutTotal.inc({ route: "checkout", status: "error" });  // error rate
    throw e;
  } finally { end(); }                                          // duration
});
```
> Incident: checkout latency creeps from 200ms to 4s over a week as data grows. With no latency metric there's no graph, no alert, no early warning — you find out when customers complain that checkout times out.

- **RED** (request-driven services): **R**ate, **E**rrors, **D**uration per endpoint.
- **USE** (resources: pools, queues, CPU): **U**tilization, **S**aturation, **E**rrors. E.g. connection-pool in-use vs max, queue depth, consumer lag.

---

## 6. Cardinality explosions in metric labels

Metric labels with unbounded values multiply time series and can take down your metrics backend (and your bill).

❌ High-cardinality label:
```ts
httpRequests.inc({ route: "/orders", userId: req.user.id });   // ❌ one series PER USER
requestLatency.observe({ url: req.originalUrl }, ms);          // ❌ unique per id/query string
```
✅ Bounded labels only:
```ts
httpRequests.inc({ route: "/orders/:id", method: "GET", status: "200" });  // template, not raw path
// put userId in a log line or a trace span attribute, NOT a metric label
```
> Incident: a `userId` label creates a new time series per user; with a million users the metrics store ingests millions of series, runs out of memory, and your monitoring goes dark — right when you need it. High-cardinality dimensions belong in logs/traces, never in metric labels.

Keep labels low-cardinality and bounded: method, templated route, status class, region. Never raw ids, emails, URLs with query strings, or error messages.

---

## 7. Alerting on symptoms vs causes (and on the wrong thing)

Alerts should fire on user-visible pain or imminent failure, point to a cause, and be actionable. Otherwise they train the team to ignore them.

❌ Alert on a raw, non-actionable internal metric:
```yaml
- alert: HighCPU
  expr: cpu_usage > 80%        # ❌ 80% CPU may be totally fine — not user-visible, not actionable
```
✅ Alert on SLO-violating symptoms, backed by cause metrics for diagnosis:
```yaml
- alert: CheckoutErrorBudgetBurn
  expr: rate(checkout_total{status="error"}[5m]) / rate(checkout_total[5m]) > 0.05
  for: 5m
  annotations:
    summary: "Checkout error rate > 5% for 5m"
    runbook: "https://.../runbooks/checkout"     # actionable: tells on-call what to do
```
> Incident: a CPU alert pages the on-call at 2am for a CPU blip that didn't affect a single user; after a few false pages the team mutes it, and the next page — a real checkout outage — is also ignored. Alert on what users feel (error rate, latency SLO, queue lag), and attach a runbook so the page is actionable.

Every alert needs: a clear symptom condition, a `for:` to avoid flapping, and a runbook link. No alert without an action.

---

## 8. No tracing across services

In a distributed system, a single user action touches many services. Without distributed tracing you can see each service's logs but not the *shape* of the request across them — where the latency or error actually originated.

❌ No spans / context not propagated:
```python
def handler(req):
    a = service_a.call()      # ❌ no span; which downstream call is the slow one?
    b = service_b.call()
    return combine(a, b)
```
✅ Instrument spans and propagate context:
```python
with tracer.start_as_current_span("handler") as span:
    span.set_attribute("user.id", uid)        # span attributes (fine here, unlike metric labels)
    with tracer.start_as_current_span("service_a.call"):
        a = service_a.call(headers=inject_trace_context())   # propagate W3C traceparent
    with tracer.start_as_current_span("service_b.call"):
        b = service_b.call(headers=inject_trace_context())
    return combine(a, b)
```
> Incident: a request takes 5s and nobody knows why. With tracing, the waterfall instantly shows `service_b.call` is the 4.8s span. Without it, you bisect by adding log timestamps across four repos and redeploying — hours of work for what a trace shows in one view.

Use OpenTelemetry; propagate the W3C `traceparent` header on every outbound call so spans link into one trace.

---

## 9. Swallowing errors so they never surface

The observability angle on the swallowed-error bug (see error-handling §1): an error caught and dropped emits *nothing* — no log, no metric, no span error — so it's invisible to every monitoring tool. The system is wrong and your dashboards are green.

❌ Failure with zero observability:
```go
if err := publish(event); err != nil {
    // ❌ ignored — no log, no metric; the event is lost and nothing records it
}
```
✅ Make every failure observable, even when you continue:
```go
if err := publish(event); err != nil {
    log.Error("event.publish.failed", "event_id", event.ID, "err", err)
    publishFailures.Inc()        // a metric you can alert on
    deadLetter(event)            // and don't lose the event
}
```
> Incident: events silently fail to publish; downstream consumers go quiet; because the failure was swallowed there's no log, no metric, no alert — you discover it days later when someone asks why a report is empty. Observability turns a silent multi-day data gap into an alert within minutes.

---

## 10. Missing audit trail for sensitive actions

Some actions need a durable, tamper-evident record independent of debug logs: who did what, to what, when. Regular logs rotate and aren't a compliance record.

❌ Sensitive change with no audit:
```ts
await db.users.update(targetId, { role: "admin" });   // ❌ who granted admin? when? no record
```
✅ Write an explicit, durable audit event:
```ts
await db.transaction(async (tx) => {
  await tx.users.update(targetId, { role: "admin" });
  await tx.auditLog.insert({                  // immutable, retained, queryable
    actor_id: req.user.id, action: "role.grant",
    target_type: "user", target_id: targetId,
    before: { role: prevRole }, after: { role: "admin" },
    at: new Date(), trace_id: req.traceId,
  });
});
```
> Incident: a security review (or a regulator) asks "who granted this account admin and when?" and there's no answer — debug logs already rotated away. Sensitive actions (permission/role changes, data exports, deletions, financial moves, config changes) need a dedicated, retained audit trail, written transactionally with the change so it can't drift.

---

## Quick scan checklist

- [ ] Logs are structured key-value, not `print`/string concat.
- [ ] No secrets/PII in logs; logger-level redaction in place.
- [ ] A correlation/trace id is minted at the edge and propagated to every downstream call + error response.
- [ ] Log levels match operational meaning (404 isn't `error`; pool exhaustion isn't `info`).
- [ ] Key paths emit RED metrics; key resources emit USE metrics.
- [ ] Metric labels are low-cardinality (no ids/emails/raw URLs as labels).
- [ ] Alerts fire on user-visible symptoms with a `for:` and a runbook, not raw internal gauges.
- [ ] Distributed traces span service hops; `traceparent` propagated.
- [ ] No swallowed error is invisible — every failure logs + increments a metric.
- [ ] Sensitive actions write a durable audit trail, transactionally with the change.
