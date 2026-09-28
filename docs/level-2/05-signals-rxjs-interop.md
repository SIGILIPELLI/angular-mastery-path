---
description: "Bridge signals and RxJS in Angular: toSignal with initialValue and requireSync, toObservable timing, takeUntilDestroyed, rxResource and output interop."
---

# 05 · Signals and RxJS Interop

Modern Angular code lives in two reactive worlds. Signals are best for **state you read**
(what's on screen right now); observables are best for **events and async workflows**
(debounce this, cancel that, retry the other). The package `@angular/core/rxjs-interop`
converts between them so you can use each where it fits.

| Function | Direction | Typical use |
|----------|-----------|-------------|
| `toSignal(obs$)` | Observable → Signal | Show an HTTP or router stream in a template |
| `toObservable(sig)` | Signal → Observable | Feed signal changes into `debounceTime`/`switchMap` |
| `takeUntilDestroyed()` | Cleanup | End a manual subscription with the component |
| `rxResource({ params, stream })` | Signal params → Observable loader → resource | Async data with status (next lesson covers resources in depth) |
| `outputFromObservable(obs$)` / `outputToObservable(ref)` | Outputs ↔ Observables | Expose or consume component outputs as streams |

## `toSignal`

```ts
private readonly route = inject(ActivatedRoute);
protected readonly tab = toSignal(this.route.queryParamMap.pipe(map((p) => p.get('tab'))));
```

`toSignal` subscribes immediately (it needs an injection context) and unsubscribes when
that context — usually the component — is destroyed. Because an observable might not have
emitted yet, the signal's value before the first emission is `undefined`, and the type
reflects it (`Signal<string | null | undefined>`). You have two ways to avoid the
`undefined`:

```ts
const b = toSignal(s$, { initialValue: 0 });                 // Signal<number>
const c = toSignal(behaviorSubject$, { requireSync: true }); // must emit synchronously
```

From our run with a plain `Subject`:

```text
before emit   a() = undefined   b() = 0
after emit(5) a() = 5           b() = 5
requireSync   c() = "ready"        (a BehaviorSubject emits on subscribe)
```

`requireSync` on an observable that doesn't emit synchronously throws immediately:

```text
NG0601: `toSignal()` called with `requireSync` but `Observable` did not emit synchronously.
```

**Errors:** if the observable errors, reading the signal *throws* that error. We saw
`read after error throws: boom`. Catch errors in the pipe (`catchError`) before calling
`toSignal` if the template must keep rendering.

## `toObservable`

```ts
protected readonly query = signal('');

private readonly results = toSignal(
  toObservable(this.query).pipe(
    debounceTime(300),
    distinctUntilChanged(),
    switchMap((q) => this.api.search(q)),
  ),
  { initialValue: [] },
);
```

This "signal in → RxJS in the middle → signal out" sandwich is the idiomatic way to use
RxJS operators with signal state.

`toObservable` emits **asynchronously**, through an effect. We set a signal from `'a'` to
`'b'` to `'c'` synchronously, flushed, then set `'c'` again:

```text
toObservable saw ["c"]
```

It never emitted `'a'` or `'b'`: the effect ran once, after all the synchronous sets,
and saw only the final value. The second `'c'` produced nothing because the signal's
value didn't change. That is usually what you want for UI state — but it means
`toObservable` is **not** an event bus. If every intermediate value matters (every
keystroke of a command log, every message), use a `Subject`.

## `takeUntilDestroyed`

```ts
constructor() {
  interval(10).pipe(takeUntilDestroyed()).subscribe(() => this.now.set(new Date()));
}
```

We let this tick for ~55 ms, destroyed the component, waited again, and checked the
counter: `ticks grew after destroy? false`. Without `takeUntilDestroyed`, the interval
would keep running (and holding the component in memory) forever.

## `rxResource`

When a request depends on signals and you want loading and error status for free, use a
resource. `rxResource` takes an observable-returning `stream` function:

```ts
import { rxResource } from '@angular/core/rxjs-interop';

protected readonly userId = input.required<number>();

protected readonly user = rxResource({
  params: () => ({ id: this.userId() }),
  stream: ({ params }) => this.http.get<User>(`/api/users/${params.id}`),
});
```

In the template you read `user.value()`, `user.isLoading()`, `user.error()` and
`user.status()`. Here's what the status did in our test, where the stream took 20 ms:

```text
status0  loading   undefined
status1  resolved  "user 1"
(set id to 2)
status2  loading   undefined   isLoading: true
status3  resolved  "user 2"
```

Note `status2`: when the params change, the previous value is **cleared** while the new
one loads. If you want to keep showing the old data during a reload, keep your own copy
(for example with `linkedSignal`, Level 3), or use `reload()`, which refetches with the same params and keeps the value — in a
separate test, `reload()` moved a resource from `resolved 1` to `reloading 1`
(`isLoading: true`) and then to `resolved 2`. When params change mid-flight, the
previous observable is unsubscribed — the same cancellation you get from `switchMap`.
Lesson 06 covers resources in depth.

## Outputs as observables

```ts
// child: expose a stream as an output
readonly scrolledToEnd = outputFromObservable(this.scroll$.pipe(filter(atEnd)));

// parent code: consume a child's output as an observable
outputToObservable(childRef.instance.scrolledToEnd).pipe(throttleTime(500)).subscribe(...)
```

## How It Actually Works

- **`toSignal`** creates a writable signal internally, subscribes to the observable, and
  calls `set` on each `next`. On `error`, it stores the error in the signal's state so the
  *getter* rethrows it. It registers `unsubscribe` with the current `DestroyRef` — which
  is why it needs an injection context. The returned signal is read-only.
- **`toObservable`** creates an `effect` that reads your signal and calls
  `subject.next(value)`, and returns that subject as an observable. In Angular 22's
  source the subject is a `ReplaySubject(1)`, so a late subscriber immediately receives
  the latest value. Since effects are scheduled and batched (Level 1,
  lesson 04), intermediate values are skipped and emissions happen after the current
  synchronous code finishes.
- **`takeUntilDestroyed`** is `takeUntil` wired to a `DestroyRef.onDestroy` callback.
- **`rxResource`** builds on the core `resource()` primitive: the `params` function is a
  computed; whenever it changes, the resource subscribes to the new stream (tearing down
  the old one) and maps notifications to its `status`, `value` and `error` signals.

Seen together, the two directions are asymmetric on purpose: observables → signals is
*lossless for the latest value*, signals → observables is *lossy for intermediate values*.
Signals model "what is true now"; they were never meant to carry every event.

## Common mistakes

- **Calling `toSignal` inside a method** (e.g. on click). It needs an injection context
  and creates a new subscription every call. Create it once, as a field.
- **Using `toSignal(http.get(...))` for data that depends on inputs.** It runs once, at
  construction, with whatever the inputs were then. Use `rxResource`/`httpResource`, or
  `toObservable(input) + switchMap`.
- **Relying on `toObservable` for every intermediate value.** Use a Subject for event streams.
- **Ignoring errors before `toSignal`.** An uncaught error makes every read of the signal
  throw, breaking the template.

## Exercise

1. Build a GitHub-style user search: `query = signal('')` bound to an input, then
   `toObservable(query)` → `debounceTime(300)` → `filter(q => q.length > 1)` →
   `switchMap` to a fake API (`of([...]).pipe(delay(200))`) → `toSignal` with
   `initialValue: []`.
2. Convert the same feature to `rxResource` with `params: () => this.debouncedQuery()`.
   (You'll need a debounced signal — build it with `toSignal(toObservable(query).pipe(debounceTime(300)))`.)
3. Show a spinner using the resource's `isLoading()` and keep the previous results
   visible while loading. Explain in a comment why the value disappears otherwise.
4. Write a test proving `toObservable` does not emit intermediate values set within one
   synchronous block.
