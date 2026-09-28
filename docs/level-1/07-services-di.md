---
description: "Share state and logic in Angular with services: the new @Service decorator, @Injectable, inject(), root singletons and component-scoped providers."
---

# 07 · Services & Dependency Injection

Components should be about presentation: what to show and how to react to the user. Code
that fetches data, holds state shared by several screens, talks to `localStorage`, formats
money or logs errors belongs in **services** — plain classes that Angular creates for you
and hands to whoever asks. The mechanism that does the handing out is **dependency
injection** (DI).

## A first service

```ts title="src/app/logger.ts"
import { Service } from '@angular/core';

@Service()
export class Logger {
  readonly lines: string[] = [];

  log(msg: string) {
    this.lines.push(msg);
    console.log(msg);
  }
}
```

Use it from any component, directive, pipe or other service with `inject()`:

```ts title="src/app/save-button.ts"
import { Component, inject } from '@angular/core';
import { Logger } from './logger';

@Component({
  selector: 'app-save-button',
  template: `<button type="button" (click)="save()">Save</button>`,
})
export class SaveButton {
  private readonly logger = inject(Logger);

  protected save() {
    this.logger.log('saved');
  }
}
```

That's the whole pattern. You never write `new Logger()`. The first time anything injects
`Logger`, Angular creates one instance; everyone after that gets the same instance for
the lifetime of the app.

## `@Service` and `@Injectable`

Angular 22 introduced `@Service()`, and it is what `ng generate service` now produces.
It is a streamlined version of the decorator you will see in almost all existing Angular
code:

```ts
// Angular 22 CLI default
@Service()
export class Logger {}

// Equivalent, and still fully supported
@Injectable({ providedIn: 'root' })
export class Logger {}
```

Both make the class **automatically available application-wide** as a singleton, and both
are tree-shakable — if nothing injects the service, it is dropped from the bundle.
Differences worth knowing:

- `@Service()` requires `inject()`; it **does not support constructor parameter
  injection**. Trying it fails the build:

  ```text
  ✘ [ERROR] NG2028: @Service class cannot use constructor dependency injection.
  Use the `inject` function instead.
  ```

  `@Injectable` supports both styles:

  ```ts
  @Injectable({ providedIn: 'root' })
  export class Legacy {
    constructor(private http: HttpClient) {}   // works
  }
  ```

- `@Service({ autoProvided: false })` creates a service that is **not** registered
  anywhere automatically; you must list it in some `providers` array (below). This is the
  equivalent of a bare `@Injectable()`.
- `@Service({ factory: () => ... })` lets you supply the value from a function.

The core team ships an `ng generate @angular/core:service` migration that converts
`@Injectable` to `@Service` where applicable, but there's no urgency: `@Injectable` is not
deprecated in 22. This course uses `@Service()` for new code and `@Injectable` when
showing patterns you will meet in existing apps. Everything else in this lesson applies
to both.

## `inject()` rules

`inject()` works only in an **injection context**: a field initialiser or constructor of
a class Angular is creating (components, directives, pipes, services), a provider
factory, or a function run with `runInInjectionContext`. Calling it later — inside a
click handler or a `setTimeout` — throws `NG0203`. So inject at the top of the class and
keep the reference:

```ts
export class BookList {
  private readonly store = inject(BookStore);   // ✅ field initialiser
  protected onClick() {
    inject(BookStore);                           // ❌ NG0203 at runtime
  }
}
```

Constructor injection (`constructor(private store: BookStore)`) is the older style.
`ng generate @angular/core:inject` migrates it to `inject()`.

## Scoping a service to a component

Sometimes you want **one instance per component**, not one per app — a wizard's state, a
form's draft, an editor's undo history. Register the service in the component's
`providers`:

```ts title="src/app/counter-panel.ts"
import { Component, Service, inject, signal } from '@angular/core';
import { Logger } from './logger';

@Service({ autoProvided: false })
export class Counter {
  private readonly logger = inject(Logger);
  readonly value = signal(0);

  inc() {
    this.value.update((v) => v + 1);
    this.logger.log(`inc -> ${this.value()}`);
  }
}

@Component({
  selector: 'app-counter-panel',
  providers: [Counter],
  template: `<button type="button" (click)="counter.inc()">{{ counter.value() }}</button>`,
})
export class CounterPanel {
  protected readonly counter = inject(Counter);
}
```

We rendered two `<app-counter-panel />` elements side by side, clicked the first button
twice and the second once. Result: the buttons read `2` and `1` — two separate `Counter`
instances — while the shared root `Logger` recorded all three calls:

```text
buttons 2 1 log ["inc -> 1","inc -> 2","inc -> 1"]
```

A component-scoped service is created with the component and destroyed with it, and it is
visible to that component's children too (Level 2, lesson 03 covers the injector
hierarchy in full).

## Services that hold state

Services are the natural home for state shared across components. Combine them with
signals: keep a private writable signal and expose read-only views and methods.

```ts title="src/app/cart.ts"
import { Service, computed, signal } from '@angular/core';

export interface CartItem { sku: string; name: string; price: number; qty: number }

@Service()
export class Cart {
  private readonly _items = signal<CartItem[]>([]);

  readonly items = this._items.asReadonly();
  readonly count = computed(() => this._items().reduce((n, i) => n + i.qty, 0));
  readonly total = computed(() => this._items().reduce((s, i) => s + i.price * i.qty, 0));

  add(item: Omit<CartItem, 'qty'>) {
    this._items.update((items) => {
      const existing = items.find((i) => i.sku === item.sku);
      return existing
        ? items.map((i) => (i.sku === item.sku ? { ...i, qty: i.qty + 1 } : i))
        : [...items, { ...item, qty: 1 }];
    });
  }

  clear() {
    this._items.set([]);
  }
}
```

A header component can show `cart.count()` and a checkout page `cart.total()`; when a
product page calls `cart.add(...)`, both update, because both templates read signals owned
by the same instance. Only `Cart` can write, so every change goes through a method you can
test, log or validate.

## How It Actually Works

An **injector** is a map from *tokens* (usually a class) to *providers* (recipes for making
a value), plus a cache of values already made. Angular builds a tree of them:

- The **root environment injector**, created by `bootstrapApplication` from the
  `providers` in `app.config.ts`.
- **Element injectors** for components and directives that declare `providers`.
- Route-level environment injectors for routes with `providers` (Level 2).

When `inject(Counter)` runs inside `CounterPanel`, Angular first asks the element injector
of that `CounterPanel` instance. It has a provider (from `providers: [Counter]`), so it
creates a `Counter` — running `Counter`'s field initialisers inside that same injection
context, which is how `Counter` itself can `inject(Logger)` — and caches it on that
element. For `Logger`, the element injectors up the tree have no provider, so the lookup
reaches the root injector.

What does "automatically provided" mean for a class that isn't in any providers array?
The compiler attaches an **injectable definition** to the class (a static `ɵprov`
property) that says "if nobody else provides me, the root injector may create me with
this factory." The root injector checks that definition on a cache miss. Because the
registration lives on the class instead of in a central list, a service nobody imports is
never referenced and the bundler can drop it.

`inject()` works only during creation because Angular sets a module-level "current
injector" pointer just before running a class's factory and clears it afterwards. Inside
a click handler that pointer is empty — hence `NG0203`.

If no injector in the chain can supply the token, you get:

```text
NG0201: No provider found for `Counter`.
```

— which is exactly what happened in our tests when we injected an
`@Service({ autoProvided: false })` class without listing it anywhere.

## Common mistakes

- **Instantiating services with `new`.** You get an instance outside DI: its own
  `inject()` calls throw, and it's not the instance everyone else shares.
- **Adding a root service to a component's `providers` "to be safe."** That creates a
  second, private instance for that component tree — a classic source of "my data isn't
  updating" bugs.
- **Constructor injection with `@Service`.** Use `inject()` (NG2028), or stay on
  `@Injectable` for that class.
- **Storing state in plain mutable fields and expecting templates to update.** With
  `OnPush` and zoneless defaults, templates react to signals. Keep shared state in signals.
- **Circular dependencies** (`A` injects `B`, `B` injects `A`) cause `NG0200`. Extract
  the shared part into a third service.

## Exercise

1. Create a `Notifications` service with `@Service()` that holds a signal of
   `{ id: number; text: string }[]`, and methods `push(text)` and `dismiss(id)`. Make
   `push` auto-dismiss after 4 seconds.
2. Create a `NotificationList` component that shows all notifications with a dismiss
   button, and a `NotifyButton` component that pushes "Hello!". Put both on the page.
3. Add `providers: [Notifications]` to `NotifyButton` (using an `autoProvided: false`
   variant of the service) and observe that clicking no longer shows anything in the
   list. Explain why in a comment, then remove it.
4. Try converting `Notifications` to constructor injection of some other service and
   read the NG2028 error.
