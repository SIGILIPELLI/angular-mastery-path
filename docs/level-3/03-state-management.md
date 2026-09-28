---
description: "Manage Angular state with signals: where state should live, signal-based service stores, linkedSignal for resettable derived state, and NgRx SignalStore."
---

# 03 · State Management with Signals

"State management" sounds like a library decision, but most of it is a *placement*
decision: for each piece of state, who owns it, who may change it, and how long it lives.
Get that right and signals, `computed` and a few services cover the majority of apps.
This lesson gives you a way to decide, one new primitive (`linkedSignal`), and a look at
NgRx SignalStore for when you want more structure.

## Where should this state live?

| Kind of state | Example | Put it in |
|---------------|---------|-----------|
| Local UI | is the menu open, current tab | a `signal` in the component |
| Derived | filtered list, totals, "can submit" | `computed` — never stored separately |
| URL | search terms, filters, selected id, page | route/query params (read as inputs) |
| Server data | issues, products, the current user | a resource (`httpResource`) or a service that loads it |
| Form draft | fields being edited | signal forms' model (Level 2) |
| Shared client state | cart, preferences, auth session | a root service with private writable signals |
| Feature-scoped shared state | a wizard's steps | a service provided on the feature's route or component |

Two rules prevent most state bugs:

1. **One owner per piece of state.** If the same fact is stored in two places, they will
   disagree eventually.
2. **Derive, don't copy.** If a value can be computed from other state, make it a
   `computed`. Copying it into another signal with an `effect` is where "why is this
   stale?" bugs come from.

## The service store pattern (recap)

You built this in Level 1:

```ts
@Service()
export class Cart {
  private readonly _items = signal<CartItem[]>([]);
  readonly items = this._items.asReadonly();
  readonly total = computed(() => this._items().reduce((s, i) => s + i.price * i.qty, 0));

  add(item: CartItem) { this._items.update((xs) => [...xs, item]); }
}
```

Private writable signal, public read-only views and computeds, and **methods as the
only way to change state**. That last point is what makes it a store: every change goes
through code you can test, log or validate. For many apps this is all you need.

## `linkedSignal`: derived, but also writable

Sometimes a value *starts* as something derived but the user can then change it — and it
should **reset** when its source changes. The classic example is "selected option":

```ts
const options = signal(['S', 'M', 'L']);
const selected = linkedSignal(() => options()[0]);

selected.set('L');           // user picks L        → selected() === 'L'
options.set(['XS', 'S']);    // options replaced    → selected() === 'XS' (reset)
```

That's what we observed: `L`, then `XS`. A `computed` can't be set; a plain `signal`
wouldn't reset. Before `linkedSignal`, people wrote an `effect` that copied the source
into a signal — with the ordering bugs that implies.

The long form gives the computation access to the **previous** value, so you can keep
the user's choice when it's still valid:

```ts
const size = linkedSignal<string[], string>({
  source: options,
  computation: (opts, previous) =>
    previous && opts.includes(previous.value) ? previous.value : opts[0],
});

size.set('S');
options.set(['S', 'M', 'L', 'XL']);   // 'S' still available → size() === 'S'
options.set(['XL']);                  // 'S' gone            → size() === 'XL'
```

Observed: `S`, then `XL`.

### Keeping stale data visible while a resource reloads

Resources clear their value while loading new params (Level 2, lesson 06), which makes
lists flash empty. `linkedSignal` fixes it neatly:

```ts
protected readonly results = resource({ params: () => this.query(), loader: ... });

protected readonly shown = linkedSignal<{ value: string[] | undefined }, string[]>({
  source: () => ({ value: this.results.hasValue() ? this.results.value() : undefined }),
  computation: (src, prev) => src.value ?? prev?.value ?? [],
});
```

Our run:

```text
loaded a    status resolved  value ["a1","a2"]   shown ["a1","a2"]
loading b   status loading   value undefined     shown ["a1","a2"]   ← old data kept
loaded b    status resolved  value ["b1","b2"]   shown ["b1","b2"]
```

Render `shown()` and dim it while `results.isLoading()` — no flash of an empty list.

## NgRx SignalStore

For larger apps, some teams want a consistent *shape* for every store: state, derived
values and methods declared the same way everywhere, plus plugins (entity collections,
devtools, persistence). **NgRx SignalStore** (`@ngrx/signals`) provides that on top of
Angular signals. It's a separate community library, not part of Angular; version 22.0.1
was current when this was written. We installed it and ran this store:

```ts title="src/app/todo.store.ts"
import { computed } from '@angular/core';
import { patchState, signalStore, withComputed, withMethods, withState } from '@ngrx/signals';

type Todo = { id: number; title: string; done: boolean };

export const TodoStore = signalStore(
  { providedIn: 'root' },
  withState({ todos: [] as Todo[], filter: 'all' as 'all' | 'open' | 'done' }),
  withComputed(({ todos, filter }) => ({
    visible: computed(() =>
      todos().filter((t) => filter() === 'all' || (filter() === 'done') === t.done),
    ),
    remaining: computed(() => todos().filter((t) => !t.done).length),
  })),
  withMethods((store) => ({
    add(title: string) {
      patchState(store, (s) => ({
        todos: [...s.todos, { id: s.todos.length + 1, title, done: false }],
      }));
    },
    toggle(id: number) {
      patchState(store, (s) => ({
        todos: s.todos.map((t) => (t.id === id ? { ...t, done: !t.done } : t)),
      }));
    },
    setFilter(filter: 'all' | 'open' | 'done') {
      patchState(store, { filter });
    },
  })),
);
```

`inject(TodoStore)` gives an object with signals for each state key, the computeds and
the methods (`add, filter, remaining, setFilter, todos, toggle, visible` in our run).
After adding two todos and toggling the first, `remaining()` was `1`; after
`setFilter('open')`, `visible()` held only "Run tests".

State is **protected** by default: calling `patchState(store, ...)` from a component
failed to compile —

```text
TS2345: Argument of type '{ todos: Signal<Todo[]>; ... }' is not assignable to parameter
of type 'WritableStateSource<{ todos: Todo[]; filter: ...; }>'.
```

— so, as in the hand-written service, changes must go through the store's methods.

**Should you use it?** It's the same pattern as the service store with less boilerplate
and a plugin ecosystem. Adopt it if your team values that uniformity across many
features; skip it for small apps. The older Redux-style `@ngrx/store` (actions, reducers,
effects) is still maintained and common in large existing codebases.

## How It Actually Works

`linkedSignal` is a writable signal with a computation attached. Internally it behaves
like a `computed` that tracks its source, plus a slot for a manually set value:

- Reading it checks whether the **source** changed since the last read. If it did, the
  computation runs (receiving `{ source, value }` of the previous state) and its result
  replaces whatever was set manually.
- `set`/`update` store a value without touching the source; reads return it until the
  source changes again.

So it's lazy and memoised like `computed` (Level 1, lesson 04), and "reset on source
change" falls out of ordinary version tracking.

`signalStore(...)` builds a class at call time. Each `with*` feature adds members:
`withState` keeps each top-level state key in its own internal writable signal — the
compile error above refers to `[STATE_SOURCE].todos` as a `WritableSignal` — and exposes
a read-only signal per key, so a template reading `todos()` isn't disturbed when only
`filter` changes; `withComputed` and `withMethods` receive the members defined so far. The result
is an injectable class, so `providedIn: 'root'` or listing it in a component's
`providers` works exactly like any service (Level 2, lesson 03). `patchState` merges the partial state you pass into those
signals immutably.

## Common mistakes

- **Duplicated state** — the same list in a service and in a component signal.
- **`effect` to sync two signals.** Use `computed` or `linkedSignal`.
- **Putting URL state in services**, making views impossible to bookmark or share.
- **Global stores for local UI state**, making components harder to reuse and test.
- **Reaching for a library before you have a problem it solves.**

## Exercise

1. Build a product page with a `variants` input (sizes with stock counts) and a
   `selectedSize` `linkedSignal` that keeps the user's choice when the variants update but
   falls back to the first in-stock size when the chosen one sells out.
2. Build a search page whose results resource keeps previous results visible (dimmed)
   while loading, using the `linkedSignal` pattern above.
3. Implement the same todo feature twice — once as a `@Service()` store with signals and
   once with NgRx SignalStore — and write a short comparison: lines of code, testability,
   and how each prevents writes from outside.
