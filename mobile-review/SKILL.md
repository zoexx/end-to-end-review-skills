---
name: mobile-review
description: |
  Reviews mobile changes — React Native, native iOS (Swift/SwiftUI) and Android (Kotlin/Compose) — for lifecycle correctness, navigation and state, offline/sync safety, performance/battery/memory, platform accessibility and UX conventions, the native boundary, permissions/privacy, and store readiness. It catches the failures that don't show up in a happy-path simulator run: crashes on process death, ANR/jank, battery drain, data loss when connectivity drops mid-write, and changes that get rejected from the App Store or Play Store.
  Use when: reviewing mobile changes, iOS/Android review, reviewing a React Native PR, Swift/SwiftUI review, Kotlin/Compose review, checking offline/sync, mobile performance/battery review, app-store release review.
allowed-tools:
  - Read
  - Grep
  - Glob
  - Bash
  - WebFetch
---

# mobile-review

A senior-level review pass for mobile code — React Native and native iOS/Android. It reads the change against the device and connectivity reality it ships into (not the simulator on wifi), then works down to the line level using the reference guides — favoring concrete failure modes (the crash, the ANR, the dead battery, the lost write, the store rejection) over taste.

## When this fires

- A PR touches `.tsx`/`.jsx` in a React Native app, `.swift`/SwiftUI, or `.kt`/Jetpack Compose.
- A change touches Activity/Fragment/ViewController lifecycle, navigation/back-stack, or state restoration.
- Local persistence, offline behavior, sync, a write queue, or cache invalidation changes.
- Something touches the RN bridge / TurboModules, a runtime permission request, secure storage, or background work (location, polling, wakelocks, `BGTaskScheduler`, `WorkManager`).
- A release-affecting change: versioning, crash reporting, feature flags, forced update, OTA (CodePush/EAS), privacy manifest, or background-mode justification.
- Someone asks for a "mobile review," "iOS/Android review," or "is this safe offline / on a low battery / on the App Store?"
- The `end-to-end-review` orchestrator routes mobile files here.

Out of scope (route elsewhere): server API contracts, status codes, and request semantics → **backend-review**; SQL schema/index/migration depth on a server DB → **database-review**; credential handling, crypto, injection, and OWASP depth → **security-review**. This skill cross-references those rather than duplicating them — but secure *on-device* storage (Keychain/Keystore vs plaintext) and just-in-time permissions are in scope here.

## The review model

Four phases, run in order. Don't nitpick lines before you understand what device and connectivity reality this ships into.

1. **Context** — What devices, OS versions, and form factors are in scope (the device/OS matrix)? Does this run online, offline, or both — and what happens on the flaky middle ground? What is the change *supposed* to do, and from which entry point (cold launch, deep link, push notification, background wake)? Read the PR description, linked issue, and any design/HIG/Material reference. Correctness is relative to intent **and** to the worst device on the matrix.
2. **High-level** — Is this the right *shape*? Look at navigation and state architecture (back-stack, deep links, restoration after process death) and the native/JS boundary fit (what belongs in native vs JS, what crosses the bridge and how often). A wrong abstraction here — state that can't survive a kill, work on the wrong thread, a chatty bridge — caught now saves a hundred line comments.
3. **Detailed** — Line by line, using the reference guides below. Retain cycles, main/UI-thread work, recomposition, list recycling, lifecycle leaks, permission denial handling, offline write loss. This is where most findings land.
4. **Verdict** — Summarize, group findings by severity, and give one explicit decision.

### Severity labels

Every finding carries exactly one:

| Label | Meaning |
| --- | --- |
| 🔴 **blocking** | Must fix before merge — crash, data loss, ANR, store rejection, security hole |
| 🟠 **important** | A real problem; should fix, not strictly a blocker |
| 🟡 **nit** | Minor style/polish, non-blocking |
| 🔵 **suggestion** | Optional improvement or alternative |
| 📚 **learning** | Context/teaching, no action required |
| 🌟 **praise** | Genuinely good work, worth calling out |

### Verdict states

`✅ approve` · `💬 approve with comments` · `🔁 request changes` · `⛔ block`

### Principles

- **Review the code, not the coder.** Comment on the change; prefer "this closure captures `self` strongly" over "you leaked memory."
- **Say why.** Every blocking/important finding names the concrete user impact: the crash on rotation, the ANR on a slow phone, the battery the wakelock drains, the write lost when the OS kills the app, the rejection from store review. "Network on the main thread → ANR on a 3G mid-range Android" beats "move this off the UI thread."
- **Stay in scope.** Flag pre-existing problems separately; don't hold the PR hostage to unrelated cleanup. Defer server and crypto depth to the sibling skills.
- **Praise is signal.** Calling out a clean `[weak self]`, a correct write queue, or a just-in-time permission prompt teaches the pattern.

## What I check

Six pillars. Each links to a reference guide for the dense version.

### 1. Lifecycle & navigation
- Activity/Fragment and ViewController lifecycle methods are paired and correct — no work started in `onResume`/`viewWillAppear` that isn't stopped in the matching teardown (camera, sensors, observers leak otherwise).
- **State survives process death.** Android can kill and recreate the process at any time; iOS can too. State is restored (`savedInstanceState`, `SceneStorage`/state restoration, RN's persisted nav state), not lost — otherwise the user returns to a blank or wrong screen.
- Navigation stack is correct: back button / swipe-back / hardware back does the expected thing, no leaked screens piling up on the stack, deep links land on a valid, restorable screen.
- Config changes (rotation, dark mode, font scale, locale, split-screen) don't crash or lose in-progress input.
- Backgrounding/foregrounding is handled: pause expensive work, persist drafts, refresh stale data on return — `AppState` (RN), `onStop`/`onStart`, scene phase.

### 2. State, data & offline
- Local persistence is correct and **migrated** — a schema change ships a migration; opening an old DB after upgrade doesn't crash or wipe data.
- Offline-first where the product needs it: reads work from cache, writes queue instead of failing, the UI doesn't dead-end on "no connection."
- **No data loss when connectivity drops mid-write** — in-flight mutations are persisted to a durable queue before the network call, retried with backoff, and survive an app kill.
- Sync conflict resolution is explicit (last-write-wins vs server-authoritative vs merge), not accidental; optimistic UI updates roll back correctly on failure.
- Cache has a TTL/invalidation story; stale data is detectable. Sensitive cached data is encrypted at rest.

### 3. Performance, battery & memory
- No heavy work on the **main/UI thread** — network, disk, JSON, image decode, crypto run off-thread (off the JS thread in RN); the frame budget is ~16ms (jank) and Android shows an ANR after ~5s.
- Lists are virtualized/recycled (`FlatList`/`LazyColumn`/`RecyclerView`), not a `map`/`ScrollView`/manual loop that mounts everything; stable keys and item layout hints are set.
- Images are downsampled to display size and cached; no full-resolution bitmaps held in memory.
- No memory leaks — retain cycles (iOS closures capturing `self`), `Context`/`Activity` leaks and static references (Android), listeners/timers/subscriptions not cleaned up (RN/all).
- Battery: no needless wakelocks, tight polling, or always-on location; background work is batched and deferrable (`WorkManager`/`BGTaskScheduler`), respecting Doze/App Standby and iOS background limits.
- Startup time / TTI and app size aren't regressed by the change (large assets, eager init, a heavy dependency).

### 4. Platform a11y & UX conventions
- Screen-reader labels exist: `accessibilityLabel`/`contentDescription`; decorative elements are hidden from the reader; custom controls expose role/state.
- **Dynamic type / font scaling** is respected — text doesn't clip or truncate at large accessibility font sizes; layouts use scalable units.
- Touch targets meet platform minimums (~44pt iOS / ~48dp Android); hit areas aren't smaller than the visual.
- Platform navigation and gestures are respected — iOS back-swipe, Android system back / predictive back, no fighting the platform's own gestures.
- Safe areas / notches / Dynamic Island insets handled; content isn't under the status bar, home indicator, or camera cutout.
- Dark mode supported; haptics used appropriately; the change reads as iOS HIG on iOS and Material on Android, not a single cross-platform compromise that feels alien on both.

### 5. Native boundary & permissions/privacy
- RN bridge / TurboModule use is sparing — no chatty per-frame bridge calls or large serialized payloads crossing each tick (serialization cost stalls the JS thread); heavy native work doesn't block the UI thread.
- Threading across the boundary is explicit: native callbacks marshal back to the right thread; nothing assumes a thread it isn't on.
- Runtime permissions are **just-in-time** — requested when the feature is used, with a rationale, and **denial is handled gracefully** (degrade, explain, route to Settings) — never a crash or a dead screen on "Don't allow."
- Privacy: ATT prompt before tracking (iOS), scoped storage / `READ_MEDIA_*` (Android), no sensitive data logged or left in plaintext.
- Secure storage uses Keychain (iOS) / Keystore-backed `EncryptedSharedPreferences` (Android) — **never** `UserDefaults`/plain `SharedPreferences`/`AsyncStorage` for tokens or secrets. (Crypto/credential depth → **security-review**.)

### 6. Release & store readiness
- Version/build numbers bumped; crash reporting wired and **symbolicated** (dSYM/ProGuard mapping uploaded) so production crashes are readable.
- Feature flags default safe; there's a forced-update / kill-switch path for a bad release.
- No store-review traps: private/undocumented APIs, missing iOS privacy manifest (`PrivacyInfo.xcprivacy`) or `NS*UsageDescription` strings, an unjustified background mode, or a permission requested without a user-facing reason.
- OTA updates (CodePush/EAS) are safe: they don't push native changes JS-only updates can't satisfy, they're staged/rollback-able, and they respect store rules on what may be updated over the air.

## Reference guides

Load the guide that matches the change. Don't load all of them — pull the one the diff touches.

| Load this | When the change involves |
| --- | --- |
| `reference/react-native.md` | `.tsx`/`.jsx` in an RN app, the bridge/TurboModules, React Navigation, `FlatList`, animations, `AppState` |
| `reference/ios-swift.md` | Swift/SwiftUI, closures/retain cycles, `DispatchQueue`/async-await, Combine, Core Data, Keychain, `BGTaskScheduler` |
| `reference/android-kotlin.md` | Kotlin/Compose, lifecycle/leaks, coroutines, recomposition, `RecyclerView`/`LazyColumn`, `WorkManager`, permissions |
| `reference/performance-battery-memory.md` | UI-thread work, list rendering, images, animations, background/polling/location, startup, app size, memory profiling |
| `reference/offline-data-sync.md` | local DB & migrations, offline-first, write queue/retry, conflict resolution, optimistic updates, connectivity handling |
| `assets/mobile-review-checklist.md` | a copy-pasteable PR checklist for a human reviewer |

## Output format

Write findings as inline comments. Each has: a **severity label**, a **`file:line`** anchor, a one-line **WHY** (the concrete user impact), and a **concrete fix** — show the corrected code when the fix isn't obvious.

> 🔴 **blocking** — `app/screens/Checkout.tsx:88`
> The "Place order" handler `await`s the network call before persisting anything, so if the OS kills the app while it's in flight (low memory, user swipes it away) the order is **silently lost** — the user thinks they paid and didn't. Write the mutation to a durable queue first, then drain the queue with retry/backoff so it survives a kill.
> ```tsx
> // before: lost if the app dies mid-request
> await api.placeOrder(cart);
> // after: persist intent, then sync — survives process death
> await queue.enqueue({ type: 'placeOrder', cart, idempotencyKey });
> drainQueue(); // retries with backoff; dedupes on idempotencyKey
> ```

> 🟠 **important** — `ios/ProfileViewController.swift:54`
> This `URLSession` completion closure captures `self` strongly and never breaks the cycle, so every visit to this screen **leaks the view controller** (and its image cache) — memory climbs until the app is jettisoned. Capture `self` weakly.
> ```swift
> task = session.dataTask(with: url) { [weak self] data, _, _ in
>   guard let self else { return }
>   DispatchQueue.main.async { self.render(data) } // UI on main, no cycle
> }
> ```

> 🌟 **praise** — `android/SyncWorker.kt:31`
> Nice — deferring this upload to `WorkManager` with a network constraint means it survives process death and respects Doze instead of holding a wakelock. Exactly the right place for background work.

Close with a verdict block:

```
## Verdict: 🔁 request changes

🔴 blocking (1)
- Checkout.tsx:88 — order lost if app killed mid-request (persist before network)

🟠 important (2)
- ProfileViewController.swift:54 — retain cycle leaks the VC every visit
- SettingsScreen.kt:120 — camera permission denial crashes instead of degrading

🟡 nit (2) · 🔵 suggestion (1) · 🌟 praise (1)

Summary: Solid navigation and clean WorkManager use, but a write can be lost on
a process kill and a denied permission crashes the screen. Fix the blocker and
the two important items and this is good to merge.
```

Pick the verdict honestly: `✅` clean, `💬` only nits/suggestions, `🔁` important issues to address, `⛔` a blocker that risks a crash, data loss, or store rejection.
