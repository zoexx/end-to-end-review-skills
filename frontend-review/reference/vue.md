# Vue review reference

Deep checklist of Vue 3 anti-patterns (Composition API, `<script setup>`, Pinia).
Each item pairs a ❌ bad snippet with a ✅ fix and the user-facing consequence.

---

## Reactivity fundamentals

### `ref` vs `reactive`

🔵 Prefer `ref` for primitives and `reactive` for objects you won't replace wholesale. The common trap is replacing a `reactive` object:

❌
```ts
let state = reactive({ count: 0 });
state = reactive({ count: 1 }); // reassigning drops the original reactivity link
```
✅
```ts
const state = reactive({ count: 0 });
state.count = 1;            // mutate in place
// or use a ref if you need to swap the whole value:
const data = ref({ count: 0 });
data.value = { count: 1 };
```
**Consequence:** templates stop updating because the binding still points at the old proxy.

### Losing reactivity by destructuring

❌
```ts
const { count } = reactive({ count: 0 });
count++; // a plain number now — nothing re-renders
```
✅
```ts
const state = reactive({ count: 0 });
const { count } = toRefs(state); // keeps reactivity
// or read state.count directly
```
**Consequence:** the UI silently goes stale — values change in JS but the DOM never updates. Same issue when destructuring a `defineProps()` result; use `toRefs(props)` or `toRef`.

### `computed` vs methods

❌ Calling a method in the template to derive a value:
```html
<span>{{ formatTotal() }}</span> <!-- re-runs on every render -->
```
✅
```ts
const total = computed(() => format(items.value.reduce(sum, 0)));
```
```html
<span>{{ total }}</span>
```
**Consequence:** methods re-execute on every re-render regardless of inputs; `computed` caches and only recomputes when a dependency changes — cheaper and predictable.

### `watch` vs `watchEffect`

🔵 Use `watch` when you need the old/new value or want lazy (not-immediate) behavior; `watchEffect` when you just want to run whenever any used dep changes.

❌ `watchEffect` doing a fetch that reads many refs — hard to see what triggers it, and it runs immediately:
```ts
watchEffect(() => { fetchUser(userId.value, includePosts.value); });
```
✅ Be explicit about the trigger:
```ts
watch(userId, (id) => fetchUser(id, includePosts.value));
```
**Consequence:** accidental extra fetches when an unrelated tracked ref changes — wasted requests and possible races.

### Deep watchers cost

❌
```ts
watch(bigTree, handler, { deep: true }); // walks the whole structure on every change
```
✅ Watch the specific field, or a getter:
```ts
watch(() => bigTree.selectedId, handler);
```
**Consequence:** deep watching large reactive structures is expensive and can cause jank. Use it only when you truly need to react to any nested change.

### Watcher cleanup / race

❌ Async watcher with no cancellation:
```ts
watch(query, async (q) => { results.value = await search(q); });
```
✅
```ts
watch(query, async (q, _old, onCleanup) => {
  const ctrl = new AbortController();
  onCleanup(() => ctrl.abort());
  results.value = await search(q, ctrl.signal);
});
```
**Consequence:** a slow earlier search resolves after a newer one and overwrites fresh results with stale data.

---

## Template directives

### `v-for` requires a stable `:key`

❌
```html
<li v-for="(item, i) in items" :key="i">{{ item.name }}</li>
```
✅
```html
<li v-for="item in items" :key="item.id">{{ item.name }}</li>
```
**Consequence:** index keys make Vue patch the wrong nodes on reorder/insert — input state, focus, and transitions attach to the wrong row.

### Never put `v-if` and `v-for` on the same element

❌
```html
<li v-for="u in users" v-if="u.active">{{ u.name }}</li>
```
✅
```html
<template v-for="u in users" :key="u.id">
  <li v-if="u.active">{{ u.name }}</li>
</template>
<!-- or filter in a computed: activeUsers -->
```
**Consequence:** precedence is ambiguous and the filter re-evaluates for every item each render — slower and error-prone. Prefer a `computed` filtered list.

### `v-if` vs `v-show`

🔵 `v-if` removes from the DOM (cheaper when rarely shown / heavy subtree); `v-show` toggles `display` (cheaper when toggled often). Using `v-if` on a frequently toggled panel forces repeated mount/unmount; using `v-show` on something rarely seen keeps it (and its cost) in the DOM.

---

## Component contract

### Don't mutate props

❌
```ts
const props = defineProps<{ modelValue: string }>();
props.modelValue = 'x'; // mutating a prop
```
✅ Emit and let the parent own it:
```ts
const emit = defineEmits<{ 'update:modelValue': [v: string] }>();
emit('update:modelValue', 'x');
```
**Consequence:** mutating props breaks one-way data flow; Vue warns, and the parent and child disagree about the value. For local editing, copy into a `ref` or use a `computed` with get/set over `v-model`.

### Declare an explicit `emits` contract

❌ Calling `$emit('save')` with no `defineEmits`.
✅
```ts
const emit = defineEmits<{ save: [payload: Form]; cancel: [] }>();
```
**Consequence:** undeclared emits are undocumented and can collide with native listeners; typed emits catch wrong payloads at compile time.

### `provide` / `inject` should be typed and keyed

✅
```ts
const ThemeKey: InjectionKey<Ref<Theme>> = Symbol('theme');
provide(ThemeKey, theme);
const theme = inject(ThemeKey)!; // typed
```
**Consequence:** string keys collide and lose types; injecting without a default and without `!` yields `undefined` at runtime — a crash deep in a child.

---

## Composition API & composables

✅ Extract reusable stateful logic into `useXxx()` composables; return refs, accept refs/getters as inputs:
```ts
function useFetch<T>(url: Ref<string>) { /* returns { data, error, loading } */ }
```
**Consequence:** business logic mixed into components can't be reused or tested in isolation. Keep composables side-effect-aware (register cleanup with `onScopeDispose`/lifecycle hooks so listeners don't leak).

### Lifecycle in `<script setup>`

❌ Registering `onMounted` inside a callback or after an `await` — it must run synchronously during setup.
✅ Call lifecycle hooks at the top level of `setup`.
**Consequence:** hooks registered after an `await` silently never fire.

---

## Performance

- `v-once` for static content that never updates; `v-memo` for expensive lists keyed on cheap deps.
- `shallowRef` / `shallowReactive` for large objects you replace wholesale (e.g. a big API payload) so Vue doesn't deeply track every nested field.

❌ `const data = ref(hugeApiPayload)` — deep reactivity over thousands of nodes.
✅ `const data = shallowRef(hugeApiPayload)` and replace `.value` wholesale.
**Consequence:** deep reactivity on big payloads adds measurable overhead on every access/update — slow renders.

---

## Pinia store shape

❌ Components reaching into and mutating store internals directly, or a store that holds derived data as plain state:
```ts
store.items.push(x); // ad-hoc mutation scattered across components
```
✅ Expose actions for mutations and `getters` for derived values; destructure with `storeToRefs`:
```ts
const counter = useCounter();
const { count, double } = storeToRefs(counter); // keeps reactivity
counter.increment();
```
**Consequence:** scattered direct mutations make state changes untraceable; destructuring a store without `storeToRefs` loses reactivity (same trap as `reactive`). Keep derived data in getters so it stays consistent.

---

## Quick scan checklist

- [ ] No reactivity lost via destructuring (`toRefs`/`storeToRefs` used).
- [ ] `reactive` objects mutated in place, not reassigned.
- [ ] Derived values are `computed`, not methods called in templates.
- [ ] `v-for` keyed on a stable id; never `v-if` + `v-for` on one element.
- [ ] `v-if` vs `v-show` chosen by toggle frequency.
- [ ] Props never mutated; `emits` declared and typed; `v-model` used for two-way.
- [ ] `provide`/`inject` use typed `InjectionKey`s.
- [ ] Async watchers cancel stale work (`onCleanup`).
- [ ] Lifecycle hooks registered synchronously in setup; composables clean up.
- [ ] `shallowRef`/`v-once`/`v-memo` used for large/static data where it pays.
