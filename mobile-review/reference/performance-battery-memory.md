# Performance, battery & memory review reference

Cross-platform deep-dive for the resource failures that don't show up in a
happy-path run on a flagship phone on wifi: jank, ANR, battery drain, OOM, slow
startup, and bloated app size. Each section pairs a ❌ anti-pattern with a ✅ fix
and names the user-visible symptom it prevents — they apply to RN, iOS, and
Android (platform specifics in the per-platform guides).

The mental model: **a frame is ~16ms (60fps) or ~8ms (120fps); the user feels
anything that misses it.** And every wakeup, every byte resident, and every
millisecond of cold start is paid by a real device with a real battery.

---

## 1. Keep work off the UI thread

❌ Network, disk, JSON parse, image decode, or crypto on the main/UI thread (the JS thread in RN):
```
// pseudocode — same mistake on every platform
onClick { val data = http.getSync(url); render(parse(data)) }  // blocks the frame
```
✅ Do the work off-thread, hop back only for the UI update:
```
onClick { launchBackground { val data = parse(http.get(url)); onMain { render(data) } } }
```
> Symptom: a frozen UI mid-interaction; on Android, > ~5s blocked → **ANR** dialog. The main thread's only job is to render and handle input — guard it.

### Chunk long main-thread work

❌ A 50ms+ synchronous loop (formatting a big list, computing a layout) on the UI thread.
✅ Move it off-thread, or yield/`InteractionManager.runAfterInteractions` (RN) / coroutine + dispatcher / GCD so the frame can land.
> Symptom: a visible hitch (dropped frames) exactly when the expensive thing runs — feels like the app "catches."

---

## 2. List virtualization

❌ Mounting every row (RN `ScrollView`+`map`, a manual `for` adding views, a non-recycling list).
✅ Use the platform's recycling/virtualizing list: `FlatList`/`FlashList` (RN), `LazyColumn`/`RecyclerView` (Android), `List`/`UICollectionView` with cell reuse (iOS). Provide stable keys and fixed-height hints (`getItemLayout`) where you can.
> Symptom: a long list takes seconds to appear, scrolls with stutter, and balloons memory because every item — including the 1,990 off-screen ones — is realized. Virtualization keeps only the visible window live.

---

## 3. Images: downsample & cache

❌ Loading a full-resolution remote/asset image into a thumbnail-sized view, no caching:
```
imageView.load("https://.../photo-4032x3024.jpg")   // decoded at full res into a 64dp view
```
✅ Request/decode at display size and cache decoded bitmaps (Glide/Coil on Android, Nuke/`UIImage` thumbnailing on iOS, `expo-image`/FastImage on RN):
```
imageView.load(url) { size(Size(128,128)); memoryCachePolicy(ENABLED) }
```
> Symptom: scrolling a photo grid spikes memory (each full-res bitmap is tens of MB) → OOM termination; with no cache the same image re-downloads and re-decodes on every scroll, draining data and battery.

---

## 4. Animate on the compositor, not the UI/JS thread

❌ Driving an animation by setting layout properties from JS/main every frame.
✅ Run it on the compositor/render thread: `useNativeDriver: true` / Reanimated (RN), Core Animation / `withAnimation` (iOS), `animate*AsState`/`Animatable` (Compose) animating transform/opacity.
> Symptom: the animation stutters the instant the UI/JS thread gets busy (a list re-render, a fetch resolving). Compositor-driven transform/opacity animations keep running independently of app-thread work.

---

## 5. Reduce wakeups — the battery section

### Batch network; don't poll tightly

❌ `setInterval(fetchUpdates, 3000)` or a 10s location poll kept on permanently.
✅ Batch and back off: push (FCM/APNs) instead of polling, coalesce requests, widen intervals, stop when backgrounded.
> Symptom: tight polling keeps the radio awake (the radio's tail energy is significant per wakeup) — measurable battery drain even when nothing's happening.

### Don't hold wakelocks / always-on location/sensors

❌ A `PARTIAL_WAKE_LOCK`, continuous high-accuracy GPS, or an always-listening sensor to "stay fresh."
✅ Use deferrable scheduled work (`WorkManager`/`BGTaskScheduler`), geofencing or significant-location-change over continuous GPS, and unregister sensors when off-screen.
> Symptom: the OS battery screen names your app as a top drainer; users uninstall. Continuous GPS especially is one of the heaviest costs on the device.

### Respect background execution limits

- Android **Doze / App Standby**: background network/jobs are deferred when the device is idle — design for "runs eventually," not "runs now."
- iOS suspends a backgrounded app within seconds; use background modes and `BGTaskScheduler` only for genuinely justified work.
> Symptom: work you assumed runs in the background silently doesn't (deferred or suspended) — features that depend on it appear broken. Fighting the limits with a foreground service or an unjustified background mode drains battery and risks store rejection.

---

## 6. Startup time / TTI

❌ Eager initialization of SDKs, heavy work in `Application.onCreate`/`AppDelegate`/RN root, large synchronous reads before first paint.
✅ Defer non-critical init (lazy/`App Startup` library/post-first-frame), parallelize what you can, and measure cold-start TTI on a low-end device.
> Symptom: a long white/splash screen on launch — the metric users and the stores notice. Every analytics/crash/ads SDK initialized eagerly adds to it.

---

## 7. App size

❌ Bundling uncompressed assets, multiple image scales you don't need, unused dependencies, or no resource shrinking.
✅ Ship App Bundles / app thinning (per-device assets), compress/convert images (WebP/HEIC), enable resource shrinking (R8 `shrinkResources`), and audit the dependency tree for dead weight.
> Symptom: a large download deters installs and gets dropped over cellular; oversized binaries also slow update adoption. Some markets cap or warn on size.

---

## 8. Memory profiling & leak signatures

- Profile with Android Studio Memory Profiler / Xcode Instruments (Allocations, Leaks) / Flipper (RN) — don't guess.
- Leak signatures to recognize: a retained `Activity`/ViewController after navigation (held by a static, listener, closure, or coroutine/task); a cache that only grows (no eviction/TTL); bitmaps/large buffers never released; listeners/observers/timers registered but never removed.
> Symptom: memory climbs over a session until the OS jettisons the app (a "random" crash users can't reproduce on demand). Fix the GC root / retain cause, not the symptom.

---

## 9. Jank & the frame budget

- Budget per frame: **~16ms at 60Hz, ~8ms at 120Hz** (ProMotion / high-refresh Android). Everything — layout, draw, your handlers — must fit.
- Use the platform jank tools: Android `Profile GPU Rendering` / `Perfetto` / `JankStats`, Xcode's Core Animation FPS / hitch ratio, RN's perf monitor.
- Common jank causes: overdraw / deep view hierarchies, layout thrash, main-thread IO, un-virtualized lists, JS-thread animations.
> Symptom: scrolling and transitions don't feel smooth even though "nothing crashes." Measure dropped frames; don't eyeball it on a flagship.

---

## 10. Network efficiency

- Batch and prefetch; compress (gzip/brotli) and prefer efficient formats; cache with sensible TTLs; paginate.
- Avoid chatty request fan-out on a screen (N round-trips where one batched call would do).
> Symptom: a screen that's slow on cellular and burns data/battery via many small, uncompressed, uncached requests. (Server-side API shape and pagination semantics → **backend-review**.)

---

## Quick scan checklist

- [ ] No network/disk/parse/decode/crypto on the main/UI/JS thread; long main-thread work chunked or moved off.
- [ ] Long lists virtualized/recycled with stable keys and layout hints.
- [ ] Images downsampled to display size and cached (decoded-bitmap cache).
- [ ] Animations run on the compositor (native driver / Core Animation / Compose animate), not per-frame on the app thread.
- [ ] No tight polling, wakelocks, or always-on GPS/sensors; network batched; push over poll.
- [ ] Background work uses deferrable scheduling and respects Doze/App Standby and iOS suspension.
- [ ] Cold-start TTI measured on a low-end device; non-critical init deferred.
- [ ] App size controlled: App Bundle/thinning, compressed assets, resource shrinking, no dead deps.
- [ ] Memory profiled; no retained Activities/VCs, unbounded caches, or unreleased bitmaps/listeners.
- [ ] Frame budget met (~16ms/8ms); jank measured with platform tools, not eyeballed on a flagship.
- [ ] Network batched, compressed, cached, and paginated; no chatty per-screen fan-out.
