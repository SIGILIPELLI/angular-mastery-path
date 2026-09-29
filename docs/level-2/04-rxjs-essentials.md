---
description: "RxJS for Angular: cold vs hot observables, Subjects, key operators, and switchMap vs mergeMap vs concatMap vs exhaustMap, verified."
---

# 04 · RxJS for Angular Developers

Signals handle *state* — values that exist right now. RxJS handles *events over time* —
keystrokes, HTTP responses, WebSocket messages, timers — and gives you operators for
combining, delaying, cancelling and retrying them. Angular's `HttpClient`, router events
and reactive-forms `valueChanges` are all RxJS observables, so you need a working
knowledge of it even in a signal-first app.

This lesson covers the subset you will use constantly. RxJS 7.8 is the version the
Angular 22 CLI installs.

## Observables are lazy — and usually cold

An `Observable` is a *recipe* for producing values. Nothing runs until someone
subscribes, and — for a **cold** observable — each subscription runs the recipe again:

```ts
let subscriptions = 0;
const cold = new Observable<number>((sub) => {
  subscriptions++;
  sub.next(Math.round(Math.random() * 1000));
  sub.complete();
});
cold.subscribe((v) => a.push(v));
cold.subscribe((v) => b.push(v));
// subscriptions === 2, and a[0] !== b[0] in our run: two independent executions
```

`HttpClient` observables are cold: two subscriptions, two requests (Level 1, lesson 09).

## Subjects: observables you push into

A `Subject` is both an observable and something you can call `.next()` on. It is **hot**:
every subscriber shares the same stream, and values emitted before you subscribe are gone.

```ts
const plain = new Subject<string>();
plain.next('lost');
plain.subscribe((v) => seen.push(v));
plain.next('kept');
// seen: ["kept"]

const state = new BehaviorSubject('initial');
state.next('one');
state.subscribe((v) => seen2.push(v));
state.next('two');
// seen2: ["one", "two"] — a BehaviorSubject replays its current value to new subscribers
// state.value === "two"
```

`BehaviorSubject` was the standard "observable store" before signals. In new code,
prefer a signal for state and keep Subjects for event streams.

## The operators you'll use weekly

Operators are functions passed to `.pipe(...)`; each returns a new observable.

| Operator | What it does |
|----------|--------------|
| `map(fn)` / `filter(fn)` | Transform / drop values |
| `tap(fn)` | Side effect (logging) without changing values |
| `debounceTime(ms)` | Emit only after the source is quiet for `ms` |
| `distinctUntilChanged()` | Drop a value equal to the previous one |
| `startWith(v)` | Emit `v` first |
| `combineLatest([a$, b$])` | Latest value of each, whenever any emits |
| `catchError(fn)` | Replace an error with another observable |
| `retry({ count, delay })` | Resubscribe after errors |
| `take(n)` / `takeUntil(x$)` | Complete after `n` values / when `x$` emits |
| `shareReplay({ bufferSize: 1, refCount: true })` | Share one execution between subscribers and replay the last value |

## Flattening: the four `*Map` operators

When each value triggers an async operation that is itself an observable (an HTTP call),
you need a *flattening* operator, and the choice decides what happens when a new value
arrives **while the previous operation is still running**. We sent two "save" events
back to back — save 1 takes 30 ms, save 2 takes 10 ms — through each operator:

```text
mergeMap    ["saved 2", "saved 1"]   run both concurrently; results in completion order
concatMap   ["saved 1", "saved 2"]   queue; run one at a time, in order
switchMap   ["saved 2"]              cancel the previous one; only the latest matters
exhaustMap  ["saved 1"]              ignore new values while one is running
```

Rules of thumb:

- **`switchMap`** for *reads* driven by user input: search-as-you-type, loading the
  detail for the selected item. Stale responses must not overwrite fresh ones.
- **`concatMap`** for *writes that must stay ordered*: saving successive edits.
- **`exhaustMap`** for *"submit" buttons*: ignore double-clicks while the request runs.
- **`mergeMap`** when operations are independent and order doesn't matter: uploading
  several files at once.

Using `switchMap` for writes is a classic bug — it can cancel a save the user expected to
happen.

## A type-ahead search, verified

```ts title="src/app/book-typeahead.ts (the stream)"
this.terms$.pipe(
  map((t) => t.trim()),
  filter((t) => t.length >= 2),
  debounceTime(300),
  distinctUntilChanged(),
  switchMap((q) =>
    this.http.get<string[]>('/api/search', { params: { q } }).pipe(
      catchError(() => of(['(error)'])),
    ),
  ),
)
```

We drove it with fake timers and the HTTP testing backend. Typing `a`, `an`, `ang`, then
`angu` 100 ms later produced exactly one request after the pause:

```text
requests after typing [ '/api/search?q=angu' ]
```

Typing on to `angular` before that response arrived **cancelled** the `angu` request
(`cancelled: true` on the test request) and sent `?q=angular`. Re-entering `angular ` with
a trailing space produced **no** request — `trim` plus `distinctUntilChanged` saw the same
term. Then a 500 error for `xx` became `(error)`, and the next term `yy` still worked:

```text
out ["Angular,AngularJS","(error)","still alive"]
```

That last point matters: `catchError` is placed **inside** the `switchMap`, on the inner
HTTP observable. Had it been on the outer pipe, the first error would have completed the
whole stream and the search box would silently stop working.

## Subscribing safely in components

Every `subscribe` needs an end. HTTP observables complete by themselves; `valueChanges`,
router events, `interval` and Subjects don't, and a subscription left running after the
component is destroyed is a memory leak that keeps doing work.

Your options, best first:

1. **Don't subscribe — convert to a signal** with `toSignal(obs$)` (next lesson), which
   unsubscribes automatically when the component is destroyed.
2. **Use the `async` pipe** in the template: `@if (user$ | async; as user) {...}`. It
   subscribes and unsubscribes with the view.
3. **`takeUntilDestroyed()`** from `@angular/core/rxjs-interop` when you really need a
   manual subscription:

```ts
import { takeUntilDestroyed } from '@angular/core/rxjs-interop';

export class Clock {
  protected readonly now = signal(new Date());
  constructor() {
    interval(1000)
      .pipe(takeUntilDestroyed())          // in an injection context: no argument needed
      .subscribe(() => this.now.set(new Date()));
  }
}
```

Outside an injection context, pass a `DestroyRef`: `takeUntilDestroyed(this.destroyRef)`.

## How It Actually Works

An `Observable` wraps a **subscribe function**. Calling `.subscribe(observer)` calls that
function with a `Subscriber`; the function calls `next`, `error` or `complete` on it and
returns a **teardown** (a cleanup function). `unsubscribe()` runs the teardown. That's
all "cold" means: each subscribe calls the function again.

An operator is a function from one observable to another. `map(fn)` returns an
observable whose subscribe function subscribes to the source and forwards `fn(value)`.
Chaining operators builds a chain of these; subscribing to the end of the chain
subscribes, link by link, all the way back to the source. Unsubscribing tears down the
chain in the same way.

`switchMap` keeps a reference to the current **inner subscription**. When a new outer
value arrives, it unsubscribes the inner one before subscribing to the next. For an HTTP
observable, that teardown aborts the in-flight request (the fetch backend passes an
`AbortSignal`), which is why the test request showed `cancelled: true`. `exhaustMap`
instead checks "is there an active inner subscription?" and drops the new value if so.

Errors terminate a stream by contract: after `error`, no more values may be emitted. So
an error that reaches the outer chain ends it for good. Catching on the inner observable
converts the error into a normal value before it can reach the outer chain.

## Common mistakes

- **Nested subscribes** (`a$.subscribe(a => b$.subscribe(...))`). Use a flattening
  operator instead; nested subscriptions can't be cancelled together.
- **`switchMap` for saves** — it can cancel writes.
- **`catchError` on the outer stream** of a long-lived pipeline, which kills it after the
  first error.
- **Forgetting to unsubscribe** from infinite streams. Prefer `toSignal`, `async` or
  `takeUntilDestroyed`.
- **Using a Subject as a state container** in new code when a signal would do.

## Exercise

1. Build a "username available?" checker: an input's value stream →
   `debounceTime(400)` → `distinctUntilChanged()` → `switchMap` to a fake API (use
   `of(...).pipe(delay(...))`) returning `true`/`false`. Show "checking…", "available"
   or "taken".
2. Add an `exhaustMap`-based "Save" button that ignores clicks while saving, and prove it
   by clicking three times quickly.
3. Replace the `exhaustMap` with `concatMap` and describe the difference in behaviour.
4. Write a unit test using `vi.useFakeTimers()` that proves only one API call happens when
   five characters are typed within 400 ms.
