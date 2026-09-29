---
description: "Angular signals from first principles: signal(), set and update, computed() derived state, effect() side effects and untracked reads."
---

# 04 · Signals: signal, computed, effect

A **signal** is a value wrapped in a function that tells Angular when it is read and when it
changes. That second part is what makes signals useful: Angular does not have to guess what
might have changed after every click — the signal *tells* it, and it also knows exactly who
depends on it.

Signals are the core reactivity model in modern Angular. Component state, derived values,
inputs (lesson 06), queries (Level 3), forms (Level 2) and async data (Level 2) are all
built on them.

## Writable signals

```ts
import { signal } from '@angular/core';

const count = signal(0);        // WritableSignal<number>

count();                        // read: 0
count.set(5);                   // replace the value
count.update((n) => n + 1);     // compute the next value from the current one → 6
```

The type is inferred from the initial value. Give it explicitly when the initial value is
narrower than what the signal will hold:

```ts
const selectedId = signal<string | null>(null);
const books = signal<Book[]>([]);
```

To hand a signal to code that should read but not write it, expose a read-only view:

```ts
private readonly _books = signal<Book[]>([]);
readonly books = this._books.asReadonly();   // Signal<Book[]> — no set/update
```

### Signals hold values by reference — replace, don't mutate

This is the rule that catches everyone once. A signal notifies only when you call `set` or
`update`, and by default it compares old and new values with `Object.is`. Mutating an
array in place changes nothing as far as the signal is concerned. From a real test run:

```ts
const list = signal<string[]>(['a']);
const n = computed(() => list().length);

n();                                    // 1
list().push('b');                       // mutate in place: no notification
n();                                    // still 1 — stale!
list.update((l) => [...l, 'c']);        // new array: notification
n();                                    // 3
```

Always produce a new array or object: `[...list, item]`, `list.filter(...)`,
`{ ...book, status: 'done' }`.

## Computed signals

`computed()` derives a read-only signal from other signals:

```ts
import { computed, signal } from '@angular/core';

const price = signal(20);
const quantity = signal(3);
const total = computed(() => price() * quantity());   // Signal<number>

total();          // 60
quantity.set(4);
total();          // 80
```

You never list dependencies. Whatever signals the function **reads while it runs** become
its dependencies, and they can change from run to run — a `computed` that reads
`a()` only inside an `if` depends on `a` only when that branch executes.

Computed signals are **lazy** and **memoized**. We verified this with a counter inside
the function:

```ts
const a = signal(2);
let runs = 0;
const doubled = computed(() => { runs++; return a() * 2; });

// runs === 0   — nothing computed until someone reads it
doubled(); doubled();
// runs === 1   — second read used the cached value
a.set(5);
// runs === 1   — setting a dependency does not recompute yet
doubled();
// runs === 2   — recomputed on the next read
a.set(5); doubled();
// runs === 2   — same value (Object.is), nothing to recompute
```

That makes `computed` the right place for filtering, sorting, totals and any value you
would otherwise recalculate in a template method.

A `computed` must be a pure function of its inputs. Trying to write to a signal inside
one throws:

```text
NG0600: Writing to signals is not allowed in a `computed`.
```

## Effects

`effect()` runs a function for its **side effects** whenever the signals it read change:
writing to `localStorage`, logging, syncing with a non-Angular library, drawing on a
canvas.

```ts title="src/app/book-store.ts (excerpt)"
import { Service, effect, signal } from '@angular/core';

@Service()
export class BookStore {
  private readonly _books = signal<Book[]>(load());

  constructor() {
    effect(() => localStorage.setItem('reading-list', JSON.stringify(this._books())));
  }
}
```

Rules for effects:

- **They need an injection context.** Create them in a constructor or a field
  initializer of a component, directive or service. Anywhere else you get
  `NG0203: effect() can only be used within an injection context such as a constructor,
  a factory function, a field initializer, or a function used with runInInjectionContext`.
  They are destroyed automatically with the component or service that created them.
- **They run asynchronously and batch changes.** Setting a signal twice in a row runs the
  effect once, with the final value. In our test, `a.set(2); a.set(3)` followed by a
  flush logged `[3]`, not `[2, 3]`.
- **They always run at least once**, to discover their dependencies.
- **They are the last resort, not the first tool.** If you are computing a value from
  other signals, use `computed`. Copying one signal into another inside an effect creates
  extra renders and ordering bugs; Level 2 introduces `linkedSignal` for the legitimate
  "derived but also writable" case.

### Reading without tracking, and cleaning up

Sometimes an effect needs a signal's current value without re-running when it changes.
Wrap the read in `untracked`. And if an effect starts something that must be stopped
before the next run (a timer, a subscription), register a cleanup:

```ts
effect((onCleanup) => {
  const u = user();                 // tracked: re-run when user changes
  const t = untracked(theme);       // read, but not a dependency
  log.push(`run ${u} ${t}`);
  onCleanup(() => log.push(`cleanup ${u}`));
});
```

Starting with `user = 'ada'` and `theme = 'dark'`, then setting `theme` to `'light'`, then
`user` to `'grace'`, the log was:

```text
["run ada dark", "cleanup ada", "run grace light"]
```

Changing `theme` alone did not re-run the effect; changing `user` did, and the previous
run's cleanup executed first.

## Signals in a component

```ts title="src/app/cart-summary.ts"
import { Component, computed, signal } from '@angular/core';

interface Line { name: string; price: number; qty: number }

@Component({
  selector: 'app-cart-summary',
  template: `
    <ul>
      @for (line of lines(); track line.name) {
        <li>
          {{ line.name }} × {{ line.qty }}
          <button type="button" (click)="inc(line.name)">+</button>
        </li>
      }
    </ul>
    <p>Items: {{ itemCount() }} — Total: {{ total() }}</p>
  `,
})
export class CartSummary {
  protected readonly lines = signal<Line[]>([
    { name: 'Notebook', price: 4, qty: 1 },
    { name: 'Pen', price: 1.5, qty: 3 },
  ]);
  protected readonly itemCount = computed(() => this.lines().reduce((n, l) => n + l.qty, 0));
  protected readonly total = computed(() => this.lines().reduce((s, l) => s + l.price * l.qty, 0));

  protected inc(name: string) {
    this.lines.update((ls) => ls.map((l) => (l.name === name ? { ...l, qty: l.qty + 1 } : l)));
  }
}
```

(`@for` is covered in the next lesson.) Clicking **+** updates one signal; two computed
values and the list re-render; nothing else in the app is touched.

## How It Actually Works

Signals form a **dependency graph** with two kinds of edges.

1. **Tracking reads.** Angular keeps a pointer to the "currently running consumer" — a
   computed, an effect, or a component's template. When a signal's getter is called, it
   checks that pointer and records an edge: *this consumer depends on me*. That is why
   you never declare dependencies; reading is declaring.
2. **Push-pull updates.** When you call `set`, the signal bumps an internal **version**
   number and walks its consumers, marking them *dirty* — the push. It does not run them.
   Later, when someone reads a dirty computed, the computed asks each of its producers
   whether their version changed since it last ran, and recomputes only if one did — the
   pull. If a recomputed value is equal to the previous one, *its* version does not bump,
   so everything downstream stays clean. That is how `a.set(5)` twice cost only one
   recomputation above.

Templates are consumers too. Each component view has a reactive node that is the active
consumer while its template function runs in update mode, so every `{{ x() }}` in the
template creates an edge. When a signal read by a template changes, Angular marks that
view (and the path up to the root) for refresh and schedules change detection. In a
zoneless application — the default for new projects — that scheduling is the main way
the screen gets updated at all. Level 3, lesson 01 follows the whole path.

Effects are consumers that are **scheduled** instead of pulled. Marking an effect dirty
queues it; Angular flushes the queue as part of its change-detection cycle. Running
later is what gives batching for free: by the time the effect runs, all the synchronous
`set` calls have happened, so it sees only the final state.

## Common mistakes

- **Mutating instead of replacing** arrays or objects held in a signal. Nothing updates.
- **Forgetting the parentheses.** `if (this.loading)` is always truthy — it's a function.
  Use `if (this.loading())`.
- **Using `effect` to derive state.** `effect(() => this.total.set(a() + b()))` should
  be `total = computed(() => a() + b())`.
- **Creating effects in event handlers or `setTimeout`.** They need an injection context;
  create them once in the constructor and let them react.
- **Reading a signal once and storing the value.** `const n = this.count();` in the
  constructor captures a number, not a live value. Keep the signal and call it where you
  need it.

## Exercise

Create a `PasswordMeter` component with:

1. A `password` signal updated from an `<input>`'s `(input)` event
   (`password.set($any($event.target).value)` or a template reference variable).
2. Computed signals `length`, `hasDigit`, `hasUpper`, `hasSymbol`, and `score` (0–4,
   one point per rule, with length ≥ 12 as one rule).
3. A computed `label` that maps the score to "weak", "fair", "good" or "strong".
4. An `effect` that logs the label with `console.log` — then set the password three
   times quickly in one handler and confirm the effect logs once.
5. Explain in a comment why `label` must be a `computed` and not an `effect` that writes
   to a signal.
