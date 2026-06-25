# React review reference

Deep checklist of React-specific anti-patterns. Each item pairs a ❌ bad snippet
with a ✅ fix and the user-facing consequence. Covers React 18/19 (hooks, Server
Components, `use client`).

---

## Re-render hygiene

### Stale closure: effect/callback missing a dependency

❌
```tsx
function Counter({ step }: { step: number }) {
  const [count, setCount] = useState(0);
  useEffect(() => {
    const id = setInterval(() => setCount(count + step), 1000);
    return () => clearInterval(id);
  }, []); // count & step captured once, forever stale
}
```
✅
```tsx
useEffect(() => {
  const id = setInterval(() => setCount(c => c + step), 1000);
  return () => clearInterval(id);
}, [step]); // functional update avoids needing `count`; `step` is declared
```
**Consequence:** the counter freezes or uses an old `step` — a silent correctness bug. Trust the exhaustive-deps lint rule; suppressing it hides real bugs.

### Memoizing the wrong thing (or nothing)

❌ Passing a fresh object/array literal to a memoized child every render:
```tsx
<ExpensiveList style={{ marginTop: 8 }} items={data.filter(Boolean)} />
```
✅
```tsx
const style = useMemo(() => ({ marginTop: 8 }), []);
const items = useMemo(() => data.filter(Boolean), [data]);
<ExpensiveList style={style} items={items} />
```
**Consequence:** `React.memo` on the child is defeated — it re-renders every time, so the optimization is dead weight. Conversely, do not `useMemo`/`useCallback` cheap values; the bookkeeping costs more than it saves.

### `useCallback` for event-handler identity

❌
```tsx
const Row = React.memo(RowImpl);
function List({ rows }) {
  return rows.map(r => <Row key={r.id} onSelect={() => select(r.id)} />);
}
```
✅
```tsx
const handleSelect = useCallback((id: string) => select(id), []);
return rows.map(r => <Row key={r.id} id={r.id} onSelect={handleSelect} />);
```
**Consequence:** a new closure per row breaks `React.memo`, re-rendering the whole list on any state change — janky scrolling on big lists.

### Stable, meaningful list keys

❌
```tsx
{items.map((item, i) => <Item key={i} {...item} />)}
```
✅
```tsx
{items.map(item => <Item key={item.id} {...item} />)}
```
**Consequence:** index keys cause React to reuse the wrong DOM node when the list reorders/filters — checkbox state, focus, and inputs jump to the wrong row. Never use the array index as a key for a list that can reorder, insert, or delete.

### Derived state instead of an effect

❌
```tsx
const [fullName, setFullName] = useState('');
useEffect(() => { setFullName(`${first} ${last}`); }, [first, last]);
```
✅
```tsx
const fullName = `${first} ${last}`; // just compute it during render
```
**Consequence:** the effect triggers a second render every change (flicker, wasted work) and can desync. Effects are for synchronizing with *external* systems, not for computing values from props/state. If it's expensive, `useMemo`.

### Don't copy props into state

❌
```tsx
function Profile({ user }) {
  const [name, setName] = useState(user.name); // frozen at mount
}
```
✅
```tsx
function Profile({ user }) {
  const name = user.name; // or lift edit state and seed via `key={user.id}`
}
```
**Consequence:** when the prop updates, the UI shows stale data. If you genuinely need to reset local edit state when the identity changes, remount with a `key`.

### Context splitting

❌ One mega-context holding both rarely-changing config and fast-changing values:
```tsx
<AppContext.Provider value={{ theme, user, cursorX, cursorY }}>
```
✅
```tsx
<ThemeContext.Provider value={theme}>
  <PointerContext.Provider value={pointer}>
```
**Consequence:** every consumer of the context re-renders whenever *any* field changes — high-frequency values like pointer/scroll position thrash the whole tree. Split by update frequency, and memoize the provider `value`.

---

## Effects & data fetching

### Cleanup to avoid leaks

❌
```tsx
useEffect(() => {
  window.addEventListener('resize', onResize);
}, []); // never removed
```
✅
```tsx
useEffect(() => {
  window.addEventListener('resize', onResize);
  return () => window.removeEventListener('resize', onResize);
}, [onResize]);
```
**Consequence:** listeners/timers/subscriptions accumulate across mounts — memory growth and duplicate handlers firing.

### Race conditions in fetch-on-change

❌
```tsx
useEffect(() => {
  fetch(`/api/items?q=${query}`).then(r => r.json()).then(setItems);
}, [query]);
```
✅
```tsx
useEffect(() => {
  let active = true;
  const ctrl = new AbortController();
  fetch(`/api/items?q=${query}`, { signal: ctrl.signal })
    .then(r => r.json())
    .then(d => { if (active) setItems(d); })
    .catch(e => { if (e.name !== 'AbortError') throw e; });
  return () => { active = false; ctrl.abort(); };
}, [query]);
```
**Consequence:** a slow earlier request can resolve *after* a faster later one and overwrite fresh results with stale data — users see the wrong list. Prefer a data library (TanStack Query, SWR, RTK Query) that handles this for you.

### Suspense & error boundaries for async UI

✅ Wrap data-loading subtrees so loading and failure are first-class:
```tsx
<ErrorBoundary fallback={<RetryCard />}>
  <Suspense fallback={<Skeleton />}>
    <Comments postId={id} />
  </Suspense>
</ErrorBoundary>
```
**Consequence:** without an error boundary a thrown render error unmounts the whole app to a blank screen; without Suspense/loading state users stare at empty space.

---

## State colocation

❌ Lifting state to the top when only one subtree uses it:
```tsx
// App holds `isMenuOpen` even though only <Header> needs it
```
✅ Keep state in the lowest common owner that needs it.
**Consequence:** over-lifted state re-renders huge trees and couples unrelated components. Colocate; lift only when two siblings must share.

---

## Controlled vs uncontrolled inputs

❌ `value` without `onChange` (or flipping between `undefined` and a string):
```tsx
<input value={name} /> // read-only; React warns; switching to/from undefined remounts
```
✅
```tsx
<input value={name ?? ''} onChange={e => setName(e.target.value)} />
```
**Consequence:** the field appears frozen, or React warns about switching controlled/uncontrolled and loses cursor position. Pick one mode and keep `value` a defined string.

---

## Error boundaries

✅ At least one boundary around each independently-failing region (route, widget):
```tsx
<ErrorBoundary fallback={<ErrorState />}>
  <Dashboard />
</ErrorBoundary>
```
**Consequence:** one component's render crash shouldn't white-screen the entire app. Note: error boundaries don't catch errors in event handlers or async code — handle those with try/catch and state.

---

## Server Components & `use client` boundaries (React 19 / Next App Router)

### Don't ship server-only code to the client

❌
```tsx
'use client';
import { db } from '@/server/db'; // secrets + heavy deps bundled to the browser
```
✅ Keep data access in a Server Component; pass plain serializable data to a Client Component:
```tsx
// page.tsx (server) — no 'use client'
const items = await db.items.findMany();
return <ClientGrid items={items} />;
```
**Consequence:** marking a data/secret module `'use client'` leaks server code and credentials into the bundle and bloats it. Push `'use client'` to the leaves that actually need interactivity.

### Props across the boundary must be serializable

❌ Passing a function or class instance from a Server Component to a Client Component (other than a Server Action).
✅ Pass strings/numbers/plain objects, or a Server Action reference.
**Consequence:** runtime serialization error or silent loss of behavior.

---

## `forwardRef` for reusable components

❌ A custom `<Input>` that drops the ref:
```tsx
function Input(props) { return <input {...props} />; } // parent can't focus it
```
✅
```tsx
const Input = forwardRef<HTMLInputElement, Props>((props, ref) =>
  <input ref={ref} {...props} />);
```
**Consequence:** parents can't focus, scroll-to, or measure the control — breaks focus management and a11y. (In React 19 `ref` can be a regular prop; either way, forward it.)

---

## Event handler identity & inline handlers

🟡 Inline arrow handlers are fine on cheap DOM nodes; only hoist/`useCallback` when the child is memoized or the handler is expensive. Don't reflexively wrap everything — premature memoization adds noise.

---

## Quick scan checklist

- [ ] Every `useEffect`/`useCallback`/`useMemo` has a complete, honest dependency array.
- [ ] Functional updates (`setX(x => ...)`) used where the new value depends on the old.
- [ ] List keys are stable IDs, never the array index for reorderable lists.
- [ ] Derived values computed in render (or `useMemo`), not synced via effects.
- [ ] Fetch effects cancel/guard against races (or use a data library).
- [ ] All subscriptions/listeners/timers are cleaned up.
- [ ] Context split by update frequency; provider `value` memoized.
- [ ] `'use client'` sits at the interactive leaves, not over data/secret modules.
- [ ] Reusable inputs forward refs; controlled inputs keep a defined `value`.
- [ ] No `any` on props/state; variant props modeled as discriminated unions.
