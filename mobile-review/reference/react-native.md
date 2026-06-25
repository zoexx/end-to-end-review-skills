# React Native review reference

Deep checklist of React Native-specific anti-patterns. Each item pairs a ❌ bad
snippet with a ✅ fix and the user-facing consequence. Covers the bridge and the
New Architecture (Fabric/TurboModules), Hermes, lists, animations, navigation,
and the leaks that survive a JS reload but not a profiler.

The mental model: **two threads matter — the JS thread and the UI (main) thread.**
Anything that stalls JS stalls touch handling and JS-driven animation; anything
that crosses the bridge every frame stalls both.

---

## Lists: virtualize, don't `map`

❌ Rendering a large array with `.map` inside a `ScrollView`:
```tsx
<ScrollView>
  {items.map(item => <Row key={item.id} {...item} />)}
</ScrollView>
```
✅ Virtualize with `FlatList` (or `FlashList`):
```tsx
<FlatList
  data={items}
  keyExtractor={item => item.id}
  renderItem={({ item }) => <Row {...item} />}
  getItemLayout={(_, i) => ({ length: ROW_H, offset: ROW_H * i, index: i })}
/>
```
> Consequence: `ScrollView`/`map` mounts **every** row up front. On a 2,000-item list that's seconds of jank on mount, huge memory, and dropped frames on scroll. `FlatList` only mounts what's near the viewport.

### Stable `keyExtractor`, never the index

❌ `keyExtractor={(_, i) => String(i)}` — or no `keyExtractor` with non-`id` data.
✅ `keyExtractor={item => item.id}`.
> Consequence: index keys make RN reuse the wrong row component when the list reorders/filters — input focus, selection, and swipe state jump to the wrong item.

### `getItemLayout` for fixed-height rows

✅ Provide `getItemLayout` when row height is constant.
> Consequence: without it, `scrollToIndex` and initial layout have to measure rows lazily — janky scroll-to and blank flashes. With it, RN skips measurement.

### Cheap `renderItem`

❌ Allocating inline objects/closures and heavy work per row in `renderItem`.
✅ Hoist a memoized row component and stable callbacks; pass `item.id` not a fresh closure.
```tsx
const Row = React.memo(RowImpl);
const onPress = useCallback((id: string) => select(id), []);
// renderItem={({ item }) => <Row item={item} onPress={onPress} />}
```
> Consequence: a new closure/object per row defeats `React.memo`, re-rendering the whole visible window on any state change — the list stutters while scrolling.

---

## Animations: drive them natively

❌ Animating with `useNativeDriver: false` (the default for some props), or driving layout with JS on every frame:
```tsx
Animated.timing(opacity, { toValue: 1, duration: 300 }).start(); // JS thread per frame
```
✅ Use the native driver (or Reanimated, which runs on the UI thread):
```tsx
Animated.timing(opacity, {
  toValue: 1, duration: 300, useNativeDriver: true,
}).start();
```
> Consequence: a JS-driven animation drops frames the moment the JS thread is busy (a fetch resolving, a list re-rendering). The native driver runs the animation on the UI thread, so it stays smooth even while JS is blocked. Note `useNativeDriver` only animates non-layout props (opacity, transform) — layout animation needs Reanimated.

---

## The bridge & New Architecture

### Don't cross the bridge every frame

❌ Reading native values or calling a native module on each scroll/gesture tick over the old bridge.
✅ Keep high-frequency work on the native side (Reanimated worklets, `Animated.event` with `useNativeDriver`), or batch.
> Consequence: the old bridge is **async and serialized** — every crossing is a JSON round-trip. Per-frame crossings flood it, lag the gesture, and stall JS. On the New Architecture (Fabric + JSI), synchronous calls exist, but a chatty native call still blocks the caller's thread.

### Large payloads across the boundary

❌ Passing a multi-megabyte object/base64 image through a native module call.
✅ Pass a handle/URI; let native read the bytes. On the New Architecture, prefer JSI/`HostObject` over serialized props for big data.
> Consequence: serializing/deserializing a big payload blocks the JS thread for the duration — visible as a frozen UI right when the data arrives.

### TurboModules / Fabric migration notes

- Spec files (`Native*` TS specs) drive codegen; a mismatch between the spec and the native impl fails at build/runtime — review both sides together.
- A library that isn't New-Architecture-ready will break under Fabric; check the dependency's interop status before enabling it.
> Consequence: half-migrated modules crash on launch or silently no-op; the failure surfaces only with the New Architecture flag on.

---

## Re-render hygiene

❌ Fresh object/array/function literals passed to memoized children every render:
```tsx
<Header style={{ padding: 8 }} actions={data.filter(Boolean)} onTap={() => go()} />
```
✅
```tsx
const style = useMemo(() => ({ padding: 8 }), []);
const actions = useMemo(() => data.filter(Boolean), [data]);
const onTap = useCallback(() => go(), []);
```
> Consequence: new identities defeat `React.memo`/`PureComponent`, so children re-render every time — on a busy screen this is the difference between 60fps and a stutter.

### Derive, don't sync via effect

❌
```tsx
const [label, setLabel] = useState('');
useEffect(() => setLabel(`${first} ${last}`), [first, last]);
```
✅ `const label = \`${first} ${last}\`;`
> Consequence: the effect causes a second render every change (flicker, wasted work) and can desync. Compute during render; `useMemo` only if it's expensive.

---

## Memory leaks: clean up everything you start

### Listeners, timers, subscriptions

❌
```tsx
useEffect(() => {
  const sub = DeviceEventEmitter.addListener('evt', onEvt);
  const id = setInterval(poll, 1000);
}, []); // never removed
```
✅
```tsx
useEffect(() => {
  const sub = DeviceEventEmitter.addListener('evt', onEvt);
  const id = setInterval(poll, 1000);
  return () => { sub.remove(); clearInterval(id); };
}, []);
```
> Consequence: listeners and intervals accumulate across mounts — duplicate handlers fire, a `setInterval` keeps polling (and waking the radio) after the screen is gone, and `setState` on an unmounted component warns or leaks.

### `AppState` subscriptions

❌ Subscribing to `AppState` without removing the subscription.
✅ Keep the returned subscription and call `.remove()` in cleanup.
> Consequence: every screen that mounts adds another `AppState` handler; on the next background/foreground they all fire, multiplying work.

---

## `AppState`: handle background/foreground

✅ Pause expensive work and refresh stale data on transition:
```tsx
useEffect(() => {
  const sub = AppState.addEventListener('change', state => {
    if (state === 'active') refetchIfStale();
    if (state === 'background') persistDraft();
  });
  return () => sub.remove();
}, []);
```
> Consequence: without this, a screen keeps a camera/socket/timer alive in the background (battery, possible crash), and shows stale data when the user returns. iOS may suspend JS in the background — long-running JS work won't finish there.

---

## Navigation (React Navigation)

### Don't pile screens on the stack

❌ `navigation.navigate('Detail', { id })` from a notification handler each time, or `push` in a loop.
✅ Use `navigate` (which dedupes by route+params where appropriate) or reset the stack for deep links; `popToTop`/`reset` when re-entering a flow.
> Consequence: repeated `push` stacks dozens of identical screens — memory grows and the back button takes forever to escape. Deep links especially should `reset` to a known stack, not append.

### Leaks from listeners tied to focus

❌ Starting a subscription in a screen and never tearing it down because the screen is only *blurred*, not unmounted (it stays mounted in the stack).
✅ Use `useFocusEffect` so work starts on focus and stops on blur:
```tsx
useFocusEffect(useCallback(() => {
  const sub = subscribe();
  return () => sub.unsubscribe();
}, []));
```
> Consequence: stacked-but-unfocused screens keep their timers/sockets running — N background screens all polling drains battery and bandwidth.

### Heavy params

❌ Passing large objects as navigation params.
✅ Pass an id; read from a store/cache on the destination.
> Consequence: nav params are serialized into persisted nav state — big params bloat state restoration and can break it.

---

## Hermes

- Ship Hermes for faster startup and lower memory; confirm it's enabled (`global.HermesInternal != null`).
- Hermes parses bytecode, not source — keep source maps for symbolicated crash stacks.
> Consequence: without Hermes (or without source maps), startup is slower and production JS crash traces are unreadable.

---

## Images: cache and size them

❌ `<Image source={{ uri }} />` for a thumbnail that points at a full-resolution remote image, with no caching strategy.
✅ Request a sized variant, set explicit dimensions, and use a caching image lib (`expo-image`, `react-native-fast-image`) for lists.
> Consequence: decoding full-res images into thumbnails blows memory and causes scroll jank; no cache re-downloads on every scroll, wasting battery and data.

---

## Platform-specific code

✅ Branch deliberately with `Platform.OS` / `.ios.tsx`/`.android.tsx`, and keep platform conventions (back handling, haptics, share sheet) on each side.
```tsx
if (Platform.OS === 'android') BackHandler.addEventListener('hardwareBackPress', onBack);
```
> Consequence: assuming one platform's behavior (e.g. ignoring Android hardware/predictive back, or iOS safe-area insets) produces a screen that's broken on the other platform — and reviewers only test one.

---

## Quick scan checklist

- [ ] Long lists use `FlatList`/`FlashList` with stable `keyExtractor` (never index) and `getItemLayout` when fixed-height.
- [ ] `renderItem` is cheap: memoized row + stable callbacks, no inline object/array literals.
- [ ] Animations use `useNativeDriver: true` or Reanimated — not JS-per-frame.
- [ ] No per-frame bridge/native-module calls; large payloads passed by handle, not serialized.
- [ ] Every listener/timer/subscription (`AppState`, `DeviceEventEmitter`, sockets) is cleaned up.
- [ ] `useFocusEffect` for work that should stop when a stacked screen blurs.
- [ ] Background/foreground handled via `AppState` (pause work, persist drafts, refresh stale).
- [ ] Navigation doesn't pile duplicate screens; deep links `reset` to a known stack; params are small.
- [ ] Images are sized to display and cached; no full-res bitmaps in lists.
- [ ] Hermes enabled and source maps retained for symbolication.
- [ ] Platform-specific behavior (back, safe areas, haptics) handled on both OSes.
- [ ] New Architecture: spec and native impl reviewed together; deps are Fabric-ready.
