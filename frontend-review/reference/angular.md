# Angular review reference

Deep checklist of Angular anti-patterns (Angular 17+, standalone, signals, RxJS).
Each item pairs a ❌ bad snippet with a ✅ fix and the user-facing consequence.

---

## Change detection

### Default vs `OnPush`

❌ Default change detection on a large, frequently-rendered component tree:
```ts
@Component({ /* default strategy */ })
```
✅
```ts
@Component({ changeDetection: ChangeDetectionStrategy.OnPush })
```
**Consequence:** default strategy re-checks the whole component on every async event anywhere in the app — wasted work and jank on large lists. `OnPush` only re-checks when an `@Input` reference changes, an event fires in the component, or an observable bound via `async` emits. Feed it immutable inputs and signals.

### Zone pollution from third-party callbacks

❌ A library callback (map SDK, websocket) triggering change detection on every tick inside Angular's zone.
✅ Run hot, non-UI work outside Angular and re-enter only to update UI:
```ts
this.zone.runOutsideAngular(() => {
  socket.on('tick', d => { this.buffer.push(d); });
});
// later, batched: this.zone.run(() => this.render());
```
**Consequence:** high-frequency events inside the zone fire change detection hundreds of times/second — the app stutters. (With zoneless / signals this matters less, but watch for it on zone-based apps.)

### `async` pipe vs manual subscribe

❌
```ts
ngOnInit() { this.svc.user$.subscribe(u => this.user = u); } // manual, no unsubscribe shown
```
✅
```html
<div *ngIf="user$ | async as user">{{ user.name }}</div>
```
**Consequence:** the `async` pipe subscribes and unsubscribes automatically and plays well with `OnPush`. Manual subscriptions in components leak unless explicitly torn down (see RxJS below) and often forget to mark for check under `OnPush`.

---

## RxJS

### Always tear down subscriptions

❌
```ts
ngOnInit() { interval(1000).subscribe(() => this.tick()); } // runs forever after destroy
```
✅
```ts
private destroyRef = inject(DestroyRef);
ngOnInit() {
  interval(1000).pipe(takeUntilDestroyed(this.destroyRef)).subscribe(() => this.tick());
}
```
**Consequence:** a leaked subscription keeps the component alive and its callback firing after navigation — memory growth and "ghost" updates. Prefer `async` pipe; when you must subscribe, use `takeUntilDestroyed()` (Angular 16+) or a `takeUntil(destroy$)` pattern.

### `shareReplay` to avoid duplicate work — but bound it

❌
```ts
data$ = this.http.get('/api/config').pipe(shareReplay()); // can leak the source subscription
```
✅
```ts
data$ = this.http.get('/api/config').pipe(shareReplay({ bufferSize: 1, refCount: true }));
```
**Consequence:** multiple `async` pipes on the same stream would otherwise fire multiple HTTP requests; without `refCount: true`, `shareReplay` keeps the upstream subscribed forever (a leak).

### `switchMap` vs `mergeMap` for request-per-input

❌
```ts
this.query$.pipe(mergeMap(q => this.search(q))).subscribe(setResults);
```
✅
```ts
this.query$.pipe(
  debounceTime(300),
  distinctUntilChanged(),
  switchMap(q => this.search(q)),
).subscribe(setResults);
```
**Consequence:** `mergeMap` keeps every in-flight request, so a slow earlier search can overwrite a newer one (stale results) and you flood the API. `switchMap` cancels the previous request when a new query arrives. (Use `concatMap` when order matters and you must not drop any, e.g. sequential saves.)

---

## Signals (Angular 17+)

✅ Prefer signals for component state; derive with `computed`, react with `effect`:
```ts
count = signal(0);
double = computed(() => this.count() * 2);
increment() { this.count.update(c => c + 1); }
```
❌ Reading a signal inside a template via a method call wrapper, or mutating signal state by reference instead of `.set`/`.update`.
**Consequence:** signals give fine-grained, glitch-free updates and play naturally with `OnPush`/zoneless. Mutating the underlying object without `.set`/`.update` won't notify consumers — stale UI. Keep `effect()` for side effects only, not for deriving state (use `computed`).

---

## Standalone components & lazy routes

✅ Use `standalone: true` components and lazy-load feature routes:
```ts
{ path: 'admin', loadComponent: () => import('./admin/admin.component').then(m => m.AdminComponent) }
```
**Consequence:** eager-loading every feature module bloats the initial bundle and slows first load (LCP). Lazy `loadComponent`/`loadChildren` splits code per route so users only download what they navigate to.

---

## Template performance

### `trackBy` / `track` in lists

❌
```html
<div *ngFor="let row of rows">{{ row.name }}</div>
```
✅
```html
<!-- new control flow (Angular 17+) -->
@for (row of rows; track row.id) { <div>{{ row.name }}</div> }
<!-- or legacy: *ngFor="let row of rows; trackBy: trackById" -->
```
**Consequence:** without tracking, Angular destroys and rebuilds every DOM node when the array reference changes — lost focus/scroll, flicker, and slow updates on large lists.

### Keep template expressions cheap

❌
```html
<span>{{ heavyCompute(item) }}</span> <!-- runs every change-detection cycle -->
```
✅ Precompute in the component, a `computed` signal, or a pure pipe:
```ts
@Pipe({ name: 'fmt', pure: true }) // memoized by input
```
**Consequence:** method calls and impure pipes in templates execute on every change-detection pass — a hidden, repeated cost that drags interaction latency (INP).

---

## Dependency injection

🔵 Prefer the `inject()` function and provide services at the right scope. Providing a service in `root` when it should be route- or component-scoped keeps unnecessary state alive; over-scoping creates duplicate instances. Use `providedIn: 'root'` for app-wide singletons and route/component providers for scoped state.

---

## Forms: reactive vs template-driven

🔵 Prefer **reactive forms** for anything non-trivial (validation, dynamic controls, testability):
```ts
form = this.fb.group({ email: ['', [Validators.required, Validators.email]] });
```
❌ Template-driven `[(ngModel)]` for complex multi-step/validated forms — validation logic ends up scattered in the template and is hard to test.
**Consequence:** template-driven forms get unwieldy fast; reactive forms keep validation and state in typed, testable code. Wire validation errors to accessible, announced messages (see accessibility reference).

---

## Quick scan checklist

- [ ] `OnPush` on non-trivial components fed immutable inputs/signals.
- [ ] Hot third-party callbacks run via `runOutsideAngular`.
- [ ] `async` pipe preferred; every manual subscribe uses `takeUntilDestroyed`/`takeUntil`.
- [ ] `shareReplay({ refCount: true, bufferSize: 1 })` for shared streams.
- [ ] Typeahead/search uses `debounceTime` + `distinctUntilChanged` + `switchMap`.
- [ ] Signal state mutated via `.set`/`.update`; `computed` for derived, `effect` for side effects only.
- [ ] Feature routes lazy-loaded; components standalone.
- [ ] Every list uses `track`/`trackBy` with a stable id.
- [ ] No method calls / impure pipes doing real work in hot templates.
- [ ] Complex forms are reactive with accessible validation messages.
