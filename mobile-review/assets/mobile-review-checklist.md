# Mobile review checklist

Copy-paste into a PR and tick what applies. Grouped by the six review pillars
(with a store-readiness sub-list under pillar 6). Severity: 🔴 blocking ·
🟠 important · 🟡 nit · 🔵 suggestion · 📚 learning · 🌟 praise.

## 1. Lifecycle & navigation

- [ ] Lifecycle methods paired: work started on resume/appear is stopped on the matching teardown (no leaked camera/sensor/observer).
- [ ] State survives process death (`savedInstanceState` / state restoration / persisted nav state) — no blank/wrong screen on return.
- [ ] Back button / swipe-back / hardware (and predictive) back do the expected thing; no leaked screens stacking up.
- [ ] Deep links land on a valid, restorable screen and `reset` the stack rather than appending.
- [ ] Config changes (rotation, dark mode, font scale, locale, split-screen) don't crash or lose in-progress input.
- [ ] Background/foreground handled: pause expensive work, persist drafts, refresh stale data on return.

## 2. State, data & offline

- [ ] Every schema change ships a tested, forward-only migration; no destructive fallback in production.
- [ ] Store choice fits the data; sensitive data encrypted at rest (not plaintext KV/blob).
- [ ] Reads come from a local source of truth; writes go local + queued — UI never dead-ends offline.
- [ ] In-flight mutations persist to a durable queue *before* the network call; survive an app kill.
- [ ] Retry uses exponential backoff + jitter; the queue resumes on next launch / connectivity.
- [ ] Replayable mutations carry a stable idempotency key (no double charge/order on retry).
- [ ] Conflict resolution is explicit (server-authoritative / LWW / merge); conflicts detected via version/`updatedAt`.
- [ ] Optimistic updates roll back (or reconcile) on failure — UI never shows an unaccepted state.
- [ ] Connectivity events trigger a drain *attempt*; success decided by the response, not a boolean.
- [ ] Caches have a TTL/staleness story; sensitive cached values encrypted.

## 3. Performance, battery & memory

- [ ] No network/disk/parse/decode/crypto on the main/UI/JS thread; long main-thread work chunked or moved off.
- [ ] Long lists virtualized/recycled (`FlatList`/`FlashList`/`LazyColumn`/`RecyclerView`/reuse) with stable keys + layout hints.
- [ ] Images downsampled to display size and cached; no full-res bitmaps held in memory.
- [ ] Animations run on the compositor (native driver / Core Animation / Compose animate), not per-frame on the app/JS thread.
- [ ] No memory leaks: retain cycles (`[weak self]`), `Context`/`Activity` leaks/static refs, listeners/timers/subscriptions cleaned up.
- [ ] No tight polling, wakelocks, or always-on GPS/sensors; network batched; push over poll.
- [ ] Background work uses deferrable scheduling (`WorkManager`/`BGTaskScheduler`) respecting Doze/App Standby & iOS suspension.
- [ ] Cold-start TTI measured on a low-end device; non-critical init deferred.
- [ ] App size not regressed (App Bundle/thinning, compressed assets, resource shrinking, no dead deps).
- [ ] Frame budget met (~16ms/8ms); jank measured with platform tools, not eyeballed on a flagship.

## 4. Platform a11y & UX conventions

- [ ] Screen-reader labels present (`accessibilityLabel`/`contentDescription`); decorative elements hidden; custom controls expose role/state.
- [ ] Dynamic Type / font scaling respected — no clipping/truncation at large accessibility sizes.
- [ ] Touch targets meet platform minimums (~44pt iOS / ~48dp Android); hit area not smaller than the visual.
- [ ] Platform navigation/gestures respected (iOS back-swipe, Android system/predictive back); no fighting native gestures.
- [ ] Safe areas / notches / Dynamic Island / camera cutouts handled; content not under system bars.
- [ ] Dark mode supported; haptics appropriate; reads as HIG on iOS and Material on Android, not an alien compromise.

## 5. Native boundary & permissions/privacy

- [ ] RN bridge/TurboModule use is sparing — no per-frame crossings or large serialized payloads; heavy native work off the UI thread.
- [ ] Threading across the boundary explicit: native callbacks marshal to the right thread.
- [ ] Permissions requested just-in-time with a rationale; denial degrades gracefully (no crash/dead screen) and routes to Settings.
- [ ] Privacy honored: ATT prompt before tracking (iOS), scoped storage / `READ_MEDIA_*` (Android); no sensitive data logged.
- [ ] Secrets/tokens in Keychain (iOS) / Keystore-backed encrypted storage (Android) — never `UserDefaults`/plain prefs/`AsyncStorage`.

## 6. Release & store readiness

- [ ] Version/build numbers bumped.
- [ ] Crash reporting wired and symbolicated (dSYM / ProGuard `mapping.txt` uploaded) — production crashes are readable.
- [ ] Feature flags default safe; a forced-update / kill-switch path exists for a bad release.
- [ ] OTA updates (CodePush/EAS) are staged, rollback-able, and don't push native changes a JS-only update can't satisfy.

### Store-review traps

- [ ] No private/undocumented APIs.
- [ ] iOS privacy manifest (`PrivacyInfo.xcprivacy`) present with required-reason API declarations.
- [ ] All referenced `NS*UsageDescription` strings provided with a clear user-facing reason.
- [ ] Every requested permission maps to a visible feature; nothing requested speculatively.
- [ ] Each declared background mode is genuinely used and justified.
- [ ] No blanket ATS exception (`NSAllowsArbitraryLoads`) shipping plaintext HTTP.

---

### Verdict

✅ approve · 💬 approve with comments · 🔁 request changes · ⛔ block

Counts: 🔴 _ · 🟠 _ · 🟡 _ · 🔵 _ · 🌟 _
