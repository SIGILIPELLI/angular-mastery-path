---
description: "Angular's built-in template control flow: @if with else and aliases, @for with track and loop variables, @empty, exhaustive @switch, and @let."
---

# 05 · Template Control Flow

Templates need to show things conditionally, repeat things for each item in a list, and
pick one of several branches. Angular's template syntax has built-in **blocks** for this,
written with an `@`: `@if`, `@for`, `@switch`, plus `@let` for local variables. They are
part of the template language itself, so there is nothing to import.

You will still see the older structural directives — `*ngIf`, `*ngFor`, `[ngSwitch]` — in
existing code. They were deprecated in Angular 20 in favour of the blocks, and
`ng generate @angular/core:control-flow` migrates them automatically (Level 4, lesson 05).

## A worked example

```ts title="src/app/order-view.ts"
import { Component, computed, signal } from '@angular/core';

type Status = 'pending' | 'shipped' | 'delivered' | 'cancelled';
interface Order { id: number; customer: string; total: number; status: Status; items: string[] }

@Component({
  selector: 'app-order-view',
  template: `
    @let open = openOrders();
    <h2>Orders ({{ open.length }} open)</h2>

    @if (loading()) {
      <p>Loading…</p>
    } @else if (orders().length === 0) {
      <p>No orders yet.</p>
    } @else {
      <ol>
        @for (order of orders(); track order.id; let i = $index, last = $last) {
          <li [class.last]="last">
            #{{ i + 1 }} {{ order.customer }} — {{ order.total }}
            @switch (order.status) {
              @case ('pending') { <em>awaiting payment</em> }
              @case ('shipped') { <strong>on its way</strong> }
              @case ('delivered') { <span>delivered</span> }
              @case ('cancelled') { <del>cancelled</del> }
              @default never;
            }
            @if (order.items[0]; as first) { (first item: {{ first }}) }
          </li>
        } @empty {
          <li>Nothing to show.</li>
        }
      </ol>
    }
  `,
})
export class OrderView {
  protected readonly loading = signal(false);
  protected readonly orders = signal<Order[]>([
    { id: 7, customer: 'Ada', total: 42, status: 'shipped', items: ['Keyboard'] },
    { id: 9, customer: 'Linus', total: 13, status: 'pending', items: [] },
  ]);
  protected readonly openOrders = computed(() =>
    this.orders().filter((o) => o.status === 'pending' || o.status === 'shipped'),
  );
}
```

## `@if`

```html
@if (condition) { ... } @else if (other) { ... } @else { ... }
```

The `as` form stores the condition's value in a local variable, which is handy when the
condition is an expression you don't want to repeat and TypeScript narrowing matters:

```html
@if (user(); as u) {
  <p>Signed in as {{ u.name }}</p>   <!-- u is non-null here -->
}
```

The template type checker narrows types inside the block just like TypeScript does after
an `if`, so `u.name` is not flagged as possibly `null`.

## `@for` and `track`

```html
@for (item of items(); track item.id) { ... } @empty { ... }
```

- The iterable can be any array or iterable (including a `Map`'s entries or a `Set`).
- **`track` is required.** Leave it out and the build stops:
  `NG5002: @for loop must have a "track" expression`.
- `@empty` renders when the collection has no items.

Inside the loop you can read these implicit variables, aliasing them with `let` when you
want a shorter name: `$index`, `$count`, `$first`, `$last`, `$even`, `$odd`.

### Choosing a track expression

`track` tells Angular how to recognise **the same item** between renders. Use a stable,
unique identity — usually a database ID. Use `track $index` only for static lists of
primitives that never reorder. Use `track item` (the object itself) only when you replace
the whole object on every change and don't mind all rows being recreated when the data is
refetched.

The difference is not cosmetic. We rendered the same three people twice, once tracked by
`p.id` and once by `$index`, each row containing an unbound `<input>`. After typing into
Ada's input in both lists and reversing the array, Chromium showed:

```text
#by-id     ['Linus | input=""', 'Grace | input=""', 'Ada | input="typed next to Ada"']
#by-index  ['Linus | input="typed next to Ada"', 'Grace | input=""', 'Ada | input=""']
```

With `track p.id`, Angular **moved** Ada's `<li>` (and the input inside it) to the end.
With `track $index`, row 0 is "the same row" before and after, so Angular kept the DOM
node in place and only changed its text to "Linus" — and the typed text stayed behind
with the wrong person. The same thing happens to focus, scroll position, animations and
child component state.

Duplicate track keys are a bug, and development builds warn about them in the console:

```text
NG0955: The provided track expression resulted in duplicated keys for a given collection.
Adjust the tracking expression such that it uniquely identifies all the items in the
collection. Duplicated keys were: key "1" at index "0" and "1".
```

## `@switch`

```html
@switch (order.status) {
  @case ('pending') { ... }
  @case ('shipped') { ... }
  @default { ... }
}
```

Cases compare with `===`. There is no fall-through, so no `break`.

When the switched value is a union type and you want the compiler to make sure every
member is handled, end with **`@default never;`**. Delete the `cancelled` case from the
example and the build fails:

```text
✘ [ERROR] TS2322: Type '"cancelled"' is not assignable to type 'never'.
      21 │             @switch (order.status) {
```

That turns "someone added a new status and forgot the UI" into a compile error.

Exhaustiveness relies on TypeScript **narrowing** the switched expression, and a
function call can't be narrowed. Switching directly on a signal — `@switch (status())`
with `@default never;` — fails even when every case is covered:

```text
✘ [ERROR] TS2322: Type 'IssueStatus' is not assignable to type 'never'.
  Type '"open"' is not assignable to type 'never'.
```

Read the signal into a `@let` first and switch on that:

```html
@let s = status();
@switch (s) {
  @case ('open') { Open }
  @case ('in-progress') { In progress }
  @case ('closed') { Closed }
  @default never;
}
```

## `@let`

`@let name = expression;` declares a read-only template variable. It is re-evaluated on
every refresh of the view, so it stays in sync with the signals it reads. Use it to avoid
repeating a long expression or calling a signal many times:

```html
@let user = session.user();
@if (user) { <p>{{ user.name }} ({{ user.email }})</p> }
```

`@let` variables are visible to the rest of the template after the declaration, including
inside nested blocks, but not before it. They are not a place for logic that belongs in a
`computed`.

## How It Actually Works

Each block compiles to an **embedded view** — a small, separately created chunk of DOM
with its own bindings — plus instructions that decide which embedded views exist.

- `@if` / `@switch` compile to a *conditional* instruction. On each update the condition
  is evaluated; if the selected branch index changed, the old embedded view is destroyed
  and the new one created. If it didn't change, the existing view is simply refreshed.
  That's why state inside an `@if` (an input's text, a child component) is lost when the
  condition flips off and on — the DOM is really removed.
- `@for` compiles to a *repeater*. On each update Angular computes the track key for every
  item and runs a reconciliation algorithm against the keys from the previous render:
  keys that disappeared have their views destroyed, new keys get new views, and keys that
  moved have their existing views **moved** in the DOM, not recreated. The algorithm is
  optimised for the common cases — appending, removing one, swapping two — so it avoids
  scanning the whole list when it can. Items whose key is unchanged but whose object is a
  new reference keep their view; only the bindings inside are updated.

This is also why `track` became mandatory with the new syntax. The old `*ngFor` defaulted
to object identity, which silently recreated every row whenever an HTTP call returned
fresh objects — one of the most common Angular performance problems. Requiring an
explicit key makes the choice visible.

## Common mistakes

- **Tracking by `$index` in a list that can reorder, insert or delete.** State ends up
  attached to the wrong row, as shown above.
- **Tracking by a property that isn't unique** (a name, a date). Expect NG0955 warnings
  and odd row reuse.
- **Heavy expressions in the iterable.** `@for (x of filterAndSort(items()); ...)` runs
  on every refresh; put it in a `computed`.
- **Mixing old and new syntax on the same element.** `*ngIf` still works (if you import
  `NgIf`), but mixing styles makes templates hard to read; migrate a file at a time.
- **Expecting `@if` to hide rather than remove.** Use `[hidden]` or a class if you need
  to keep the DOM (and its state) alive while invisible.

## Exercise

Build a `TaskBoard` component with a `tasks` signal of
`{ id: number; title: string; priority: 'low' | 'medium' | 'high'; done: boolean }`.

1. Show "All done 🎉" with `@if` when every task is done, otherwise the list.
2. Render tasks with `@for ... track task.id`, prefixing each with its 1-based position
   and striking through done tasks. Show "No tasks" with `@empty`.
3. Render a badge per priority with `@switch` and `@default never;`. Then add `'urgent'`
   to the union type and watch the build fail until you add a case.
4. Use `@let` to compute the number of remaining tasks once and show it in two places.
5. Add a "Shuffle" button, put an `<input>` in each row, and compare `track task.id`
   with `track $index` yourself.
