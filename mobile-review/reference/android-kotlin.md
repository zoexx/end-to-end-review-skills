# Android / Kotlin review reference

Deep checklist of Android and Kotlin anti-patterns. Each item pairs a ❌ bad
snippet with a ✅ fix and the user-facing consequence. Covers lifecycle-aware
components and leaks, coroutines, Jetpack Compose recomposition, list keys, ANR,
`WorkManager`, runtime permissions, StrictMode, and R8.

The mental model: **the system can destroy and recreate your Activity at any time
(rotation, low memory, process death), and the main thread renders every frame.**
Most Android bugs are: leaked a `Context`, blocked the main thread, or recomposed
too much.

---

## Context / Activity leaks

❌ Holding an `Activity`/`View`/`Context` in a static, a singleton, or a long-lived object:
```kotlin
object Cache { var ctx: Context? = null }      // ❌ pins the Activity forever
Cache.ctx = this                               // in an Activity
```
✅ Use `applicationContext` for anything long-lived; never store an `Activity` statically:
```kotlin
Cache.ctx = applicationContext                 // outlives any single Activity safely
```
> Consequence: a static reference to an `Activity` keeps it (and its whole view tree, ~MBs) alive after it's destroyed — on every rotation you leak another copy. LeakCanary flags this as a retained `Activity`. Eventually `OutOfMemoryError`.

### Inner classes, handlers, and listeners

❌ A non-static inner `Handler`/`Runnable`/anonymous listener posting delayed work — it holds an implicit reference to the outer `Activity`.
✅ Remove callbacks in `onDestroy` (`handler.removeCallbacksAndMessages(null)`), or use a `WeakReference`/lifecycle-scoped coroutine.
> Consequence: a delayed `Runnable` keeps the destroyed `Activity` alive until it fires — a leak window the length of the delay, multiplied by every rotation.

### Lifecycle-aware components

✅ Tie observers/sensors/location to the lifecycle so they auto-stop:
```kotlin
lifecycle.addObserver(myLocationObserver)      // started on START, stopped on STOP
// or collect flows with repeatOnLifecycle(Lifecycle.State.STARTED)
```
> Consequence: a manually-registered listener you forget to unregister in `onStop` keeps running (and leaking) while the screen is gone — battery and memory.

---

## Coroutines: scope, dispatcher, cancellation

### Scope to the lifecycle, not `GlobalScope`

❌
```kotlin
GlobalScope.launch { repo.sync() }   // ❌ outlives the screen; never cancelled
```
✅ Use `viewModelScope` / `lifecycleScope`:
```kotlin
viewModelScope.launch { repo.sync() } // cancelled when the ViewModel clears
```
> Consequence: `GlobalScope` work keeps running after the user leaves — it may touch a dead view, leak the scope, and waste battery/network. Lifecycle scopes cancel automatically.

### Right dispatcher: `IO` for blocking, `Main` for UI

❌ Network/disk on `Dispatchers.Main` (or no dispatcher):
```kotlin
viewModelScope.launch { val data = api.fetch() } // blocks main if fetch is blocking
```
✅
```kotlin
viewModelScope.launch {
  val data = withContext(Dispatchers.IO) { api.fetch() } // off main
  state.value = data                                     // back on main
}
```
> Consequence: a blocking call on the main thread freezes rendering and triggers an **ANR** ("App Not Responding") after ~5s — the user gets a "close the app?" dialog.

### Structured concurrency & cancellation

- Use `coroutineScope`/`supervisorScope` so a child failure cancels siblings (or doesn't, deliberately).
- Make long loops cooperatively cancellable: check `isActive` / call `ensureActive()`; don't swallow `CancellationException`.
```kotlin
try { work() } catch (e: CancellationException) { throw e } // rethrow, never swallow
catch (e: Exception) { handle(e) }
```
> Consequence: swallowing `CancellationException` breaks cancellation — the coroutine keeps running after its scope is cancelled, leaking work and confusing state.

---

## Jetpack Compose recomposition

### Unstable parameters force recomposition

❌ Passing a `List`/lambda/unstable type that Compose can't prove stable:
```kotlin
@Composable fun Items(rows: List<Row>) { ... }   // List is unstable → recomposes
```
✅ Use stable/immutable types (`ImmutableList` from kotlinx.collections.immutable, or `@Immutable`/`@Stable` annotations), and hoist lambdas:
```kotlin
@Composable fun Items(rows: ImmutableList<Row>) { ... }
```
> Consequence: an unstable param makes Compose conservatively recompose the composable (and its children) on every parent recomposition — jank and wasted work on a busy screen.

### `remember` to keep, `derivedStateOf` to compute

❌ Recreating objects or recomputing derived values on every recomposition:
```kotlin
val sorted = items.sortedBy { it.name }          // ❌ re-sorts every recomposition
```
✅
```kotlin
val sorted = remember(items) { items.sortedBy { it.name } }
// derive state that depends on other state without extra recompositions:
val showFab by remember { derivedStateOf { listState.firstVisibleItemIndex > 0 } }
```
> Consequence: re-sorting/re-allocating every frame burns the main thread; reading a fast-changing state directly (scroll offset) recomposes constantly — `derivedStateOf` recomposes only when the *derived* result changes.

### Don't read state too high

❌ Reading a frequently-changing state at the top of a screen so the whole screen recomposes.
✅ Push the read down to the smallest composable that needs it (defer the read with a lambda/`State`).
> Consequence: a high read invalidates the entire subtree on every change — scroll offset or text input lags the whole screen.

### Side effects in the right effect handler

❌ Launching coroutines or registering callbacks directly in the composable body.
✅ Use `LaunchedEffect(key)`, `DisposableEffect` (with `onDispose` cleanup), `rememberCoroutineScope`.
> Consequence: work in the body re-runs on every recomposition — duplicate network calls, leaked listeners.

---

## List keys: `RecyclerView` & `LazyColumn`

❌ `LazyColumn { items(rows) { Row(it) } }` with no key, or `RecyclerView` without stable IDs.
✅
```kotlin
LazyColumn { items(rows, key = { it.id }) { Row(it) } }
// RecyclerView: setHasStableIds(true) + getItemId returning a stable id; use DiffUtil
```
> Consequence: without stable keys, Compose/RecyclerView reuse the wrong slot on reorder/insert/delete — animations jump, scroll position drifts, and item state (selection, expanded) lands on the wrong row.

---

## ANR from main-thread IO

❌ File/DB/SharedPreferences/network on the main thread:
```kotlin
prefs.edit().putString("k", v).commit()   // ❌ commit() is synchronous disk IO
```
✅ Use async APIs and background dispatchers:
```kotlin
prefs.edit().putString("k", v).apply()    // async; or DataStore (coroutine-based)
```
> Consequence: synchronous IO on the main thread stalls frames and risks an ANR. Enable **StrictMode** in debug to catch disk/network on the main thread:
```kotlin
StrictMode.setThreadPolicy(StrictMode.ThreadPolicy.Builder().detectAll().penaltyLog().build())
```

---

## `WorkManager` for deferrable / guaranteed work

❌ A foreground `Service` or a coroutine to upload/sync that must survive process death.
✅ Enqueue `WorkManager` with constraints:
```kotlin
val req = OneTimeWorkRequestBuilder<SyncWorker>()
  .setConstraints(Constraints.Builder().setRequiredNetworkType(NetworkType.CONNECTED).build())
  .setBackoffCriteria(BackoffPolicy.EXPONENTIAL, 10, TimeUnit.SECONDS)
  .build()
WorkManager.getInstance(ctx).enqueue(req)
```
> Consequence: ad-hoc background work is killed when the process dies and ignores Doze/App Standby; `WorkManager` persists the request, respects battery constraints, and retries with backoff. Holding a wakelock to keep work alive instead drains the battery.

---

## Runtime permissions

❌ Calling a guarded API assuming the permission is granted, or crashing when it isn't:
```kotlin
camera.open()   // ❌ SecurityException if CAMERA not granted
```
✅ Request just-in-time with the Activity Result API, show rationale, and handle denial:
```kotlin
val launcher = registerForActivityResult(RequestPermission()) { granted ->
  if (granted) openCamera() else showDegradedUiAndSettingsLink()
}
launcher.launch(Manifest.permission.CAMERA)
```
> Consequence: an ungranted permission throws `SecurityException` (crash) or leaves a dead screen. On Android 11+, repeated denial becomes permanent — you must route the user to Settings, not re-prompt.

---

## R8 / ProGuard & crash mapping

- Enable `minifyEnabled true` for release; keep rules for reflection-based libs (Gson/Moshi models, anything using reflection).
- **Upload the `mapping.txt`** to your crash reporter so release stack traces deobfuscate.
> Consequence: missing keep rules crash only in the minified release build (works in debug); a missing mapping file makes every production crash an unreadable wall of `a.b.c`.

---

## Memory leaks (LeakCanary patterns)

Common retained-object signatures LeakCanary surfaces:
- `Activity`/`Fragment` retained after destroy → static ref, leaked listener, or a coroutine/handler outliving the scope.
- A `Fragment`'s `View` retained after `onDestroyView` → holding a binding/view reference past `onDestroyView` (null it out).
- A bitmap or large structure on the leak path → cache without eviction.
> Consequence: each leaked `Activity` is megabytes; a few rotations or screen revisits and you hit `OutOfMemoryError`. Fix the GC root LeakCanary names, don't just `System.gc()`.

---

## Quick scan checklist

- [ ] No `Activity`/`View`/`Context` held in statics/singletons; `applicationContext` for long-lived needs.
- [ ] Delayed `Handler`/`Runnable`/listeners removed in `onDestroy`/`onStop`; lifecycle-aware where possible.
- [ ] Coroutines use `viewModelScope`/`lifecycleScope`, not `GlobalScope`; cancelled with the lifecycle.
- [ ] Blocking work on `Dispatchers.IO`, UI on `Main`; `CancellationException` re-thrown, never swallowed.
- [ ] Compose params are stable/immutable; `remember`/`derivedStateOf` used; state read at the lowest level.
- [ ] Side effects in `LaunchedEffect`/`DisposableEffect`, not the composable body.
- [ ] `LazyColumn`/`RecyclerView` have stable keys/IDs + `DiffUtil`.
- [ ] No disk/DB/network on the main thread; StrictMode enabled in debug; `apply()`/DataStore over `commit()`.
- [ ] Deferrable/guaranteed background work uses `WorkManager` with constraints + backoff — no wakelock hacks.
- [ ] Permissions requested just-in-time with rationale; denial degrades gracefully and routes to Settings.
- [ ] R8 enabled for release with correct keep rules; `mapping.txt` uploaded for symbolication.
- [ ] Fragment view references nulled in `onDestroyView`; LeakCanary clean.
