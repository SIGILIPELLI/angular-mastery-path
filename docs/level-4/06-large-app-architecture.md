---
description: "Structure large Angular apps: feature-first folders, core, shared and feature layers, public APIs, enforced dependency rules and scoping."
---

# 06 · Large-App Architecture

Small apps can be organised any way you like. Large ones — dozens of features, several
teams, years of changes — succeed or fail on a few structural decisions: how code is
grouped, which parts may depend on which, and how those rules are enforced once nobody
remembers them. Angular doesn't impose a structure, so this lesson gives you a proven
default and a way to keep it honest.

## Organise by feature, not by type

A folder per *kind* of file (`components/`, `services/`, `models/`) scatters every
feature across the tree; changing "orders" touches five folders. Group by **feature**
instead, and inside a feature by **role**:

```text
src/app/
├── core/                  app-wide singletons: auth, error handling, HTTP interceptors, layout shell
├── shared/
│   └── ui/                reusable, dumb presentational pieces: buttons, cards, pipes
└── features/
    ├── orders/
    │   ├── index.ts       ← the feature's public API
    │   ├── feature/       routed pages ("smart" components) + orders.routes.ts
    │   ├── ui/            components specific to orders
    │   └── data-access/   API clients, stores, models
    └── billing/
        ├── index.ts
        ├── feature/
        └── data-access/
```

- **`feature/`** components are *containers*: they inject stores, read route inputs and
  coordinate.
- **`ui/`** components are *presentational*: inputs in, outputs out, no injected data
  services (Level 1's `BookCard` was one). They're easy to test and reuse.
- **`data-access/`** holds everything about getting and storing data: `httpResource`s,
  API services, signal stores.

This is the same split Nx popularised with "library types"; it works equally well as
folders in a single CLI app.

## Dependency rules

Structure only helps if dependencies flow one way:

```text
features/*  ──►  shared/, core/
features/A  ──►  features/B   only through B's index.ts
shared/, core/  ──X  features/   (never)
ui/  ──X  data-access/          (presentational components don't fetch data)
```

Each feature exposes a **public API** — typically its routes and the few types others
need — and keeps everything else private:

```ts title="src/app/features/orders/index.ts"
// Public API of the orders feature: only this may be imported by other features.
export { ORDERS_ROUTES } from './feature/orders.routes';
export type { OrderSummary } from './data-access/order-summary';
```

## Enforcing the rules

Rules in a README erode; rules in CI don't. Teams usually enforce them with ESLint —
`no-restricted-imports` patterns, `eslint-plugin-boundaries`, or Nx's
`@nx/enforce-module-boundaries`. To show the idea without extra dependencies, here is a
small Node script that checks the two most important rules:

```js title="tools/check-boundaries.mjs"
// Minimal architecture check: run with `node tools/check-boundaries.mjs`.
// Rules:
//  1. shared/ and core/ must not import from features/.
//  2. A feature may import another feature only through its index.ts (public API).
import { readFileSync, readdirSync, statSync } from 'node:fs';
import { dirname, join, relative, resolve, sep } from 'node:path';

const APP = resolve('src/app');
const files = [];
(function walk(dir) {
  for (const name of readdirSync(dir)) {
    const p = join(dir, name);
    if (statSync(p).isDirectory()) walk(p);
    else if (p.endsWith('.ts') && !p.endsWith('.spec.ts')) files.push(p);
  }
})(APP);

const area = (p) => relative(APP, p).split(sep); // e.g. ['features', 'orders', 'ui', 'x.ts']
const violations = [];

for (const file of files) {
  const src = readFileSync(file, 'utf8');
  for (const [, spec] of src.matchAll(/from\s+['"](\.{1,2}\/[^'"]+)['"]/g)) {
    const target = resolve(dirname(file), spec);
    const [fromTop, fromFeature] = area(file);
    const [toTop, toFeature, ...rest] = area(target);

    if ((fromTop === 'shared' || fromTop === 'core') && toTop === 'features') {
      violations.push(`${relative(APP, file)}: ${fromTop}/ must not import features/ (${spec})`);
    }
    const crossFeature = fromTop === 'features' && toTop === 'features' && fromFeature !== toFeature;
    if (crossFeature && rest.length > 0 && rest.join('/') !== 'index') {
      violations.push(`${relative(APP, file)}: deep import into features/${toFeature} (${spec}); use its index.ts`);
    }
  }
}

console.log(`checked ${files.length} files`);
for (const v of violations) console.log('✘ ' + v);
process.exit(violations.length ? 1 : 0);
```

We ran it against a sample tree containing one legitimate cross-feature import
(`import type { OrderSummary } from '../../orders'`) and two planted violations:

```text
checked 8 files
✘ features/billing/feature/billing-page.ts: deep import into features/orders (../../orders/data-access/orders-api); use its index.ts
✘ shared/ui/price.ts: shared/ must not import features/ (../../features/orders/data-access/orders-api)
```

It exited with code 1, which fails a CI step. The import through `index.ts` passed. (A
real project would also handle path aliases and `export … from` statements; a lint
plugin does that for you.)

## Scope state and services to features

- **Lazy-load each feature** with `loadChildren: () => import('./features/orders').then(m => m.ORDERS_ROUTES)`
  so features are separate chunks (Level 1, lesson 08).
- **Provide feature services on the feature's route** (`providers: [OrdersStore]` in the
  parent route) rather than `providedIn: 'root'`, so they live in the lazy chunk and are
  created only when the feature is used (Level 2, lesson 07). Mark such services
  `@Service({ autoProvided: false })` so they can't be accidentally used globally.
- **Keep `core/` small.** It's for things there must be exactly one of. If "core" grows
  business logic, it has become a dumping ground.
- **Treat `shared/` as a library** with its own public API — or make it one (lesson 02).

## Communication between features

Features should know as little about each other as possible:

- **Via the URL** — navigating to `/billing?order=42` couples less than injecting another
  feature's store.
- **Via a small shared contract** in `core/` (an event bus service or a shared signal) when
  several features must react to the same thing (e.g. "user signed out").
- **Via the public API** for types and routes — never deep imports.

## Conventions that scale

- One component, directive, pipe or service per file, named by what it *is*
  (`order-list.ts`, `OrderList`) — the 2025 style guide (Level 1, lesson 02).
- `ng generate` everywhere, so new code starts consistent.
- A formatter (`prettier`, already configured by `ng new`) and a linter
  (`ng add @angular-eslint/schematics`) in CI.
- An `index.ts` only at feature/library boundaries, not in every folder — barrel files
  everywhere make circular imports and accidental coupling easy.

## How It Actually Works

Why do these rules matter so much in Angular specifically?

- **Bundling follows imports.** A chunk contains everything reachable from its entry. One
  deep import from `billing` into `orders/data-access` drags orders code into the billing
  chunk (or into a shared chunk loaded by both). Import direction is bundle size.
- **DI follows the injector tree.** A service provided on a lazy route lives in that
  route's environment injector; code outside the route can't inject it (`NG0201`), which
  is a useful *runtime* boundary. A service provided in root is visible everywhere — the
  opposite of encapsulation.
- **Compilation is per file.** The Angular compiler resolves each component's `imports`
  from its own file's imports, so a clean dependency graph also means faster incremental
  rebuilds: changing a leaf `ui/` component doesn't invalidate unrelated features.

## Common mistakes

- **A giant `shared` folder** that everything imports and that imports everything.
- **Barrel files everywhere**, hiding circular dependencies.
- **Root-provided feature services**, so feature code loads eagerly and state leaks across
  features.
- **Presentational components injecting stores**, making them impossible to reuse.
- **Rules without enforcement.**

## Exercise

1. Restructure the Level 2 issue tracker into `core/`, `shared/ui/` and
   `features/issues/{feature,ui,data-access}` with an `index.ts` exporting only the routes.
2. Move `IssueApi` and `Busy` to the right layers; provide feature services on the
   `issues` route.
3. Add `tools/check-boundaries.mjs` (above) and an npm script `"lint:boundaries"`; plant a
   violation and confirm it fails.
4. Replace the script with an ESLint rule (for example `no-restricted-imports` patterns or
   `eslint-plugin-boundaries`) and compare what each catches.
