---
description: "Keep Angular apps current: ng update, the release and support cycle, and the automated migrations to control flow, inject(), signal inputs and outputs — with real results."
---

# 05 · Upgrading & Migrations

An Angular app that stays on an old major version slowly becomes expensive: it stops
getting security fixes, new libraries don't support it, and the eventual jump spans
several breaking changes at once. The good news is that Angular invests heavily in making
upgrades mechanical. This lesson covers the routine (`ng update`, one major at a time) and
the optional **modernisation migrations** that rewrite older code to current idioms —
run for real on a legacy-style component.

## The cadence you're keeping up with

Angular releases a major version about every six months; each gets roughly 6 months of
active support and 12 months of long-term support (Level 1, lesson 01). A practical
policy for most teams:

- Upgrade to each new major **within its active support window** — a few weeks after
  release, once your key third-party libraries support it.
- Apply minor and patch releases as they come (`ng update` handles them the same way).
- Never skip majors in one jump; go 20 → 21 → 22, running migrations at each step.

## `ng update`

Run it with no arguments to see what's out of date:

```bash
ng update
```

On a project already on the latest release, Angular CLI 22.2 printed:

```text
Using package manager: npm
Collecting installed dependencies...
Found 16 dependencies.
We analyzed your package.json and everything seems to be in order. Good work!
```

When updates exist, it lists each package with the command to run, for example:

```bash
ng update @angular/core@22 @angular/cli@22
```

That command does three things:

1. Updates the Angular packages (and their peer dependencies) in `package.json` and
   installs them.
2. Runs the **update migrations** each package ships for that version range — code and
   configuration changes the Angular team wrote for their own breaking changes.
3. Prints what it changed so you can review the diff.

Before running it, commit everything (so the diff is reviewable), and check the
interactive **update guide** at `angular.dev/update-guide`, which lists manual steps for
your exact from/to versions and app complexity. After it: `ng build`, `ng test`, click
through the app, then commit.

Third-party libraries (`@ngrx/*`, `@angular/material` is first-party but separate,
component libraries) usually publish a matching major shortly after Angular's. Update
them in the same step with `ng update <package>` so their own migrations run too.

## Modernisation migrations

Separately from version updates, `@angular/core` ships optional schematics that rewrite
code to newer APIs. In Angular 22.2 the collection includes:

| Command (`ng generate @angular/core:<name>`) | What it does |
|------------------|--------------|
| `standalone` | converts NgModule-based components to standalone, in stages |
| `control-flow` | `*ngIf`/`*ngFor`/`[ngSwitch]` → `@if`/`@for`/`@switch` |
| `inject` | constructor injection → `inject()` |
| `signal-input-migration` (alias `signal-inputs`) | `@Input()` → `input()` |
| `output-migration` (alias `outputs`) | `@Output()` + `EventEmitter` → `output()` |
| `signal-queries-migration` | `@ViewChild` & co. → `viewChild()` & co. |
| `signals` | runs the signal input, output and query migrations together |
| `route-lazy-loading` | eager `component:` routes → `loadComponent` |
| `ngclass-to-class`, `ngstyle-to-style` | `[ngClass]`/`[ngStyle]` → `[class.x]`/`[style.x]` bindings |
| `common-to-standalone` | `CommonModule` imports → the individual directives/pipes used |
| `self-closing-tag` | `<app-x></app-x>` → `<app-x />` |
| `cleanup-unused-imports` | removes unused entries from component `imports` |
| `service` | `@Injectable` → `@Service` where applicable |
| `router-testing-module-migration` | deprecated `RouterTestingModule` → `provideRouter()` |

Each accepts `--path` to limit it to part of the app, which is how you migrate a large
codebase gradually.

## A real run

We started from this component, written in the style of Angular 15–16:

```ts title="order-list.component.ts (before)"
import { Component, EventEmitter, Input, OnInit, Output } from '@angular/core';
import { NgFor, NgIf, NgClass } from '@angular/common';
import { HttpClient } from '@angular/common/http';

export interface Order { id: number; customer: string; total: number; urgent: boolean }

@Component({
  selector: 'app-order-list',
  standalone: true,
  imports: [NgIf, NgFor, NgClass],
  template: `
    <div *ngIf="orders.length; else empty">
      <div *ngFor="let order of orders; trackBy: trackById; let i = index" [ngClass]="{ urgent: order.urgent }">
        {{ i + 1 }}. {{ order.customer }} — {{ order.total }}
        <button (click)="select.emit(order)">Open</button>
      </div>
    </div>
    <ng-template #empty><p>No orders.</p></ng-template>
  `,
})
export class OrderListComponent implements OnInit {
  @Input() orders: Order[] = [];
  @Input() title = 'Orders';
  @Output() select = new EventEmitter<Order>();

  constructor(private http: HttpClient) {}

  ngOnInit() {
    console.log('loaded', this.title);
  }

  trackById(_: number, order: Order) {
    return order.id;
  }
}
```

and ran five migrations in a row:

```bash
ng g @angular/core:control-flow --path src/app/legacy
ng g @angular/core:inject --path src/app/legacy
ng g @angular/core:signal-input-migration --path src/app/legacy
ng g @angular/core:output-migration --path src/app/legacy
ng g @angular/core:ngclass-to-class --path src/app/legacy
```

Their reports:

```text
=== signal-input-migration
    Successfully migrated to signal inputs 🎉
      -> Migrated 1/2 inputs.
    To see why 1 inputs couldn't be migrated
    consider re-running with "--insert-todos" or "--best-effort-mode".
=== output-migration
      -> Migrated 1 out of 1 detected outputs (100.00 %).
=== ngclass-to-class
      -> Migrated 1 ngClass to class in 1 files.
```

The result, which built without errors:

```ts title="order-list.component.ts (after)"
import { Component, Input, OnInit, inject, input, output } from '@angular/core';

import { HttpClient } from '@angular/common/http';

export interface Order {
  id: number;
  customer: string;
  total: number;
  urgent: boolean;
}

@Component({
  selector: 'app-order-list',
  standalone: true,
  template: `
    @if (orders.length) {
      <div>
        @for (order of orders; track trackById(i, order); let i = $index) {
          <div [class.urgent]="order.urgent">
            {{ i + 1 }}. {{ order.customer }} — {{ order.total }}
            <button (click)="select.emit(order)">Open</button>
          </div>
        }
      </div>
    } @else {
      <p>No orders.</p>
    }
  `,
})
export class OrderListComponent implements OnInit {
  private http = inject(HttpClient);

  @Input() orders: Order[] = [];
  readonly title = input('Orders');
  readonly select = output<Order>();

  ngOnInit() {
    console.log('loaded', this.title());
  }

  trackById(_: number, order: Order) {
    return order.id;
  }
}
```

What to notice:

- **Templates:** `*ngIf … else` became `@if … @else` with the `<ng-template>` inlined;
  `*ngFor` with `trackBy` became `@for` with a `track` expression that calls the old
  function; `[ngClass]` became a `[class.urgent]` binding; the now-unused `NgIf`, `NgFor`
  and `NgClass` imports were removed.
- **Code:** constructor injection became `inject()`; `title` became `input()` and its use
  in `ngOnInit` was updated to `this.title()`; the output became `output<Order>()` —
  callers' `(select)="..."` bindings need no change.
- **Formatting:** the interface was reformatted. Run your formatter afterwards and review
  the diff as a whole.
- **What it didn't do:** `orders` stayed an `@Input`. Re-running with `--insert-todos`
  explained why, as a comment in the file:

```ts
// TODO: Skipped for migration because:
//  This input is used in a control flow expression (e.g. `@if` or `*ngIf`)
//  and migrating would break narrowing currently.
```

  The migration is conservative: it won't make a change that might alter type checking.
  You can finish by hand — or tidy the result further: `track order.id` instead of the
  helper function, and `standalone: true` can be removed because it's the default.

## How It Actually Works

Migrations are **schematics**: programs that read your project into a virtual file tree,
compute changes, and write them only if everything succeeds (which is also why
`--dry-run` works). The simple ones, like `control-flow`, parse templates and rewrite text
ranges. The signal migrations are more sophisticated: they build a TypeScript program for
each `tsconfig` in the project ("Preparing analysis for: tsconfig.app.json…"), find
**every reference** to each input across components, templates, host bindings and tests,
decide whether converting it is provably safe (it isn't if, for example, something writes
to the input, or narrowing would break), and then rewrite the declaration and all
references together. That global analysis is why they report partial results like
"Migrated 1/2 inputs" instead of guessing.

`ng update` works the same way: each Angular package declares, in its `package.json`,
a set of migrations keyed by version. When you update from 21 to 22, the CLI installs the
new version and runs every migration whose version falls in that range.

## Common mistakes

- **Skipping majors** and then facing years of breaking changes at once.
- **Running migrations on a dirty working tree** so you can't review or revert their
  changes cleanly.
- **Migrating everything in one giant PR.** Use `--path` per feature and ship in slices.
- **Ignoring skipped items.** Use `--insert-todos` and track the TODOs.
- **Upgrading Angular without its ecosystem** (Material, NgRx, component libraries),
  leading to peer-dependency conflicts.

## Exercise

1. Write a component in the "before" style above — `*ngIf`, `*ngFor`, `@Input`,
   `@Output`, constructor injection, `[ngClass]`.
2. Run the five migrations on its folder only, one at a time, committing between them so
   you can see each diff.
3. Run the signal input migration with `--insert-todos`, read the reason for anything
   skipped, and finish the migration by hand.
4. Run `ng update` in one of your projects and read what it proposes (don't apply it on
   a dirty tree).
