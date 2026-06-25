# iOS / Swift review reference

Deep checklist of Swift and SwiftUI anti-patterns. Each item pairs a ❌ bad
snippet with a ✅ fix and the user-facing consequence. Covers retain cycles,
threading, SwiftUI state, Combine/async-await cancellation, Core Data, Keychain,
background tasks, and the HIG/ATS basics that bite at store review.

The mental model: **UIKit/SwiftUI is main-thread-only, and ARC won't free a
cycle.** Most iOS bugs are one of: touched the UI off the main thread, or kept a
strong reference you meant to be weak.

---

## Retain cycles: `[weak self]` in escaping closures

❌ An escaping closure (network completion, Combine sink, timer, stored handler) capturing `self` strongly:
```swift
service.fetch { result in
  self.items = result   // closure retains self; service retains closure → cycle
}
```
✅ Capture weakly and bail if gone:
```swift
service.fetch { [weak self] result in
  guard let self else { return }
  self.items = result
}
```
> Consequence: the view controller / view model is never deallocated — its memory, its image caches, and its observers leak. Revisit the screen a few times and the app is jettisoned for memory. Use Instruments → Leaks or the memory graph debugger to confirm.

### Delegates must be `weak`

❌ `var delegate: FooDelegate?` (strong).
✅ `weak var delegate: (any FooDelegate)?`.
> Consequence: a strong delegate back-reference is the classic two-object cycle — parent owns child, child strongly owns parent back.

### `[unowned self]` only when self truly outlives the closure

❌ `[unowned self]` on a closure that can fire after `self` is gone.
✅ Prefer `[weak self]` + `guard let self`; use `unowned` only for closures whose lifetime is strictly bounded by `self`.
> Consequence: `unowned` after dealloc is a **crash** (use-after-free), not a leak.

---

## Main-thread UI updates

❌ Updating UI from a background queue (URLSession callbacks, Core Data background context, GCD work):
```swift
URLSession.shared.dataTask(with: url) { data, _, _ in
  self.label.text = decode(data)   // ❌ not on main — undefined behavior / crash
}.resume()
```
✅ Hop to main for the UI write:
```swift
URLSession.shared.dataTask(with: url) { [weak self] data, _, _ in
  let text = decode(data)
  DispatchQueue.main.async { self?.label.text = text }
}.resume()
```
With async-await, annotate UI-touching code `@MainActor`:
```swift
@MainActor func render(_ text: String) { label.text = text }
```
> Consequence: touching UIKit/SwiftUI off the main thread is undefined — flicker, garbled layout, or a hard crash that reproduces only under load. The Main Thread Checker flags it in debug.

### Keep the main thread free

❌ Synchronous disk/JSON/crypto/image-decode on the main thread.
✅ Do the work on a background queue / `Task.detached` / `await`, then hop back for the UI.
> Consequence: heavy main-thread work drops frames (jank) and, if long enough, makes the app appear frozen and gets it killed by the watchdog.

---

## SwiftUI state ownership

### `@StateObject` to own, `@ObservedObject` to observe

❌ Creating an observable model with `@ObservedObject` inside a view:
```swift
struct CartView: View {
  @ObservedObject var model = CartModel()  // ❌ recreated on every parent re-render
}
```
✅ Own it with `@StateObject` (created once), pass it down as `@ObservedObject`:
```swift
struct CartView: View {
  @StateObject private var model = CartModel()  // lifetime tied to the view
}
```
> Consequence: `@ObservedObject = SomeModel()` builds a **new** model every time the view is reconstructed — in-progress state (loading flags, edits, network tasks) resets unexpectedly and work restarts.

### `@State` is for value types owned by the view

❌ `@State` holding a reference type you expect to observe.
✅ `@State` for `struct`/value local UI state; `@StateObject`/`@Observable` for reference models.
> Consequence: mutating a class through `@State` won't trigger updates the way you expect; the view won't re-render on internal changes.

### Don't do heavy work in `body`

❌ Sorting/filtering/formatting a large collection inline in `body`.
✅ Compute in the model / a cached property; `body` should be cheap and pure.
> Consequence: `body` recomputes constantly (any observed change, parent update). Expensive work there runs on the main thread on every recomputation — the prime cause of SwiftUI jank.

### `@Observable` (iOS 17+) narrows invalidation

✅ Prefer the `@Observable` macro over `ObservableObject`; views re-render only when the fields they read change.
> Consequence: `@Published` invalidates every observer of the object on any change; `@Observable` tracks per-field reads, cutting needless re-renders.

---

## View lifecycle

- `onAppear`/`onDisappear` can fire more than once and aren't symmetric with allocation — don't treat `onAppear` as "init once."
- Start subscriptions/timers in `onAppear` and **tear them down in `onDisappear`**.
- UIKit: pair `viewWillAppear`/`viewWillDisappear`; don't start a location/sensor session you never stop.
> Consequence: work started on appear and never stopped keeps running off-screen — battery, network, and possible crashes when the view is gone.

---

## Combine / async-await cancellation

### Store and cancel Combine subscriptions

❌ A `sink` whose `AnyCancellable` is discarded:
```swift
publisher.sink { self.update($0) }   // ❌ cancelled immediately or leaked
```
✅ Store it; the set's deinit cancels:
```swift
publisher
  .receive(on: DispatchQueue.main)
  .sink { [weak self] in self?.update($0) }
  .store(in: &cancellables)
```
> Consequence: an unstored cancellable is torn down at once (the pipeline never runs) or, if retained by a cycle, leaks and keeps firing.

### Cancel `Task`s tied to view lifetime

❌ A long `Task {}` started in `onAppear` and never cancelled.
✅ Use `.task {}` (auto-cancels on disappear) or hold the `Task` and `cancel()` in `onDisappear`; check `Task.isCancelled` / `try Task.checkCancellation()` in loops.
> Consequence: an orphaned task keeps fetching after the user left — wasted battery/network, and a `@MainActor` write to a gone view.

---

## Core Data threading & contexts

❌ Passing an `NSManagedObject` between threads, or using the view context on a background queue:
```swift
DispatchQueue.global().async {
  obj.title = "x"   // ❌ obj belongs to another context/thread
}
```
✅ Use `perform`/`performBackgroundTask` and pass `NSManagedObjectID`s across threads:
```swift
container.performBackgroundTask { ctx in
  let obj = try? ctx.existingObject(with: objectID)
  obj?.title = "x"
  try? ctx.save()
}
```
> Consequence: Core Data is **not thread-safe** — cross-thread access corrupts the store or crashes intermittently. Enable `-com.apple.CoreData.ConcurrencyDebug 1` to catch it. Heavy fetches/saves on the view context also jank the UI.

---

## Keychain for secrets — never `UserDefaults`

❌
```swift
UserDefaults.standard.set(authToken, forKey: "token")   // ❌ plaintext, unencrypted
```
✅ Store credentials in the Keychain (with an appropriate accessibility class):
```swift
// kSecAttrAccessibleAfterFirstUnlockThisDeviceOnly for a token used in background
SecItemAdd([
  kSecClass: kSecClassGenericPassword,
  kSecAttrAccount: "authToken",
  kSecValueData: Data(token.utf8),
  kSecAttrAccessible: kSecAttrAccessibleAfterFirstUnlockThisDeviceOnly,
] as CFDictionary, nil)
```
> Consequence: `UserDefaults` is an unencrypted plist — tokens/secrets there are trivially readable from a backup or a jailbroken device. (Crypto/credential-handling depth → **security-review**.)

---

## Background tasks: `BGTaskScheduler`, not a timer

❌ Expecting a `Timer` or a long `Task` to keep running after the app backgrounds.
✅ Register and schedule a `BGAppRefreshTask`/`BGProcessingTask`, do bounded work, and set an expiration handler:
```swift
BGTaskScheduler.shared.register(forTaskWithIdentifier: id, using: nil) { task in
  task.expirationHandler = { /* save & stop */ }
  doWork { task.setTaskCompleted(success: $0) }
}
```
> Consequence: iOS suspends a backgrounded app within seconds — a plain timer won't fire and unfinished work is lost. Declaring a background mode you don't genuinely use is a store-review rejection.

---

## Memory: images and autorelease

- Downsample images to the display size (`UIGraphicsImageRenderer` / `CGImageSourceCreateThumbnailAtIndex`) before showing; don't hold full-res `UIImage`s in a list.
- Wrap tight loops that create temporary objects in `autoreleasepool` to cap peak memory.
> Consequence: full-resolution images in cells spike memory and cause termination; without an autorelease pool a big loop's temporaries accumulate until the loop ends.

---

## HIG & App Transport Security basics

- Respect Dynamic Type (use text styles, not fixed point sizes), safe-area insets, and the Dynamic Island / notch.
- Honor the platform back-swipe and standard gestures; don't reinvent navigation.
- ATS: don't add blanket `NSAllowsArbitraryLoads = true` to ship plaintext HTTP — justify any exception narrowly.
- Provide every `NS*UsageDescription` you reference and the privacy manifest (`PrivacyInfo.xcprivacy`) for required-reason APIs.
> Consequence: ignoring Dynamic Type clips text for low-vision users; a blanket ATS exception, a missing usage string, or a missing privacy manifest is an automatic App Store rejection.

---

## Quick scan checklist

- [ ] Escaping closures capture `[weak self]`; delegates are `weak`; no `unowned` that can outlive `self`.
- [ ] All UIKit/SwiftUI writes happen on the main thread / `@MainActor`.
- [ ] No synchronous disk/JSON/crypto/decode on the main thread.
- [ ] `@StateObject` owns models; `@ObservedObject` only observes; `@State` for value types.
- [ ] `body` is cheap and pure — no heavy sort/filter/format inline.
- [ ] Combine cancellables stored; `Task`s cancelled on disappear (`.task {}` or explicit cancel).
- [ ] Core Data accessed via `perform`/`performBackgroundTask`; object IDs crossed between threads, not objects.
- [ ] Secrets/tokens in Keychain with a sensible accessibility class — never `UserDefaults`.
- [ ] Background work uses `BGTaskScheduler` with an expiration handler; background modes are justified.
- [ ] Images downsampled before display; tight allocation loops use `autoreleasepool`.
- [ ] Dynamic Type, safe areas, and back-swipe respected; ATS exceptions narrow; usage strings + privacy manifest present.
