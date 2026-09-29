---
description: "Share Angular code across apps: workspaces, ng generate library, public APIs, ng-packagr builds, partial compilation and Nx."
---

# 02 · Libraries & Monorepos

Once an organisation has more than one Angular app, the same things get written twice: a
button, a date formatter, an auth interceptor, API models. The fix is a **library** —
code packaged with a deliberate public API that several apps consume. The Angular CLI
supports this out of the box with multi-project workspaces and a library builder
(`ng-packagr`). Everything below was generated and built with Angular CLI 22.2.

## A workspace with no default app

```bash
npx @angular/cli@22 new platform --no-create-application
cd platform
ng generate library ui-kit
ng generate application shop
```

`ng generate library` reported:

```text
CREATE projects/ui-kit/README.md (1431 bytes)
CREATE projects/ui-kit/ng-package.json (155 bytes)
CREATE projects/ui-kit/package.json (210 bytes)
CREATE projects/ui-kit/tsconfig.lib.json (486 bytes)
CREATE projects/ui-kit/tsconfig.lib.prod.json (401 bytes)
CREATE projects/ui-kit/tsconfig.spec.json (449 bytes)
CREATE projects/ui-kit/src/public-api.ts (70 bytes)
CREATE projects/ui-kit/src/lib/ui-kit.spec.ts (526 bytes)
CREATE projects/ui-kit/src/lib/ui-kit.ts (194 bytes)
UPDATE angular.json (923 bytes)
UPDATE package.json (813 bytes)
UPDATE tsconfig.json (1012 bytes)
```

Both projects live under `projects/`, share one `node_modules` and one `angular.json`, and
are built with `ng build ui-kit` / `ng build shop`. The library's targets in
`angular.json`:

```json
"build": { "builder": "@angular/build:ng-packagr", "defaultConfiguration": "production", ... },
"test":  { "builder": "@angular/build:unit-test", ... }
```

## The library's public API

Write the library like any Angular code:

```ts title="projects/ui-kit/src/lib/button.ts"
import { Component, input } from '@angular/core';

export type ButtonVariant = 'primary' | 'secondary' | 'danger';

@Component({
  selector: 'ui-button',
  host: { '[class]': '"ui-button ui-button--" + variant()' },
  template: `<ng-content />`,
  styles: `
    :host { display: inline-block; padding: 0.5rem 1rem; border-radius: 6px; cursor: pointer; }
    :host(.ui-button--primary) { background: #c3002f; color: white; }
    :host(.ui-button--secondary) { background: #eee; color: #222; }
    :host(.ui-button--danger) { background: #8b0000; color: white; }
  `,
})
export class UiButton {
  readonly variant = input<ButtonVariant>('primary');
}
```

```ts title="projects/ui-kit/src/lib/format-price.ts"
/** Formats an amount in minor units (cents) as a currency string. */
export function formatPrice(cents: number, currency = 'USD', locale = 'en-US'): string {
  return new Intl.NumberFormat(locale, { style: 'currency', currency }).format(cents / 100);
}
```

Then decide what consumers may import. **Only what `public-api.ts` exports is part of the
library**; everything else is private implementation:

```ts title="projects/ui-kit/src/public-api.ts"
/*
 * Public API Surface of ui-kit
 */

export * from './lib/button';
export * from './lib/format-price';
```

Treat that file like a contract. Adding an export is easy; removing or changing one is a
breaking change for every app that uses it.

## Building it

```bash
ng build ui-kit
```

```text
Building entry point 'ui-kit'
------------------------------------------------------------------------------
✔ Compiling with Angular sources in partial compilation mode.
✔ Generating FESM and DTS bundles
✔ Copying assets
✔ Writing package manifest
✔ Built ui-kit
```

The output in `dist/ui-kit/`:

```text
dist/ui-kit/README.md
dist/ui-kit/fesm2022/ui-kit.mjs
dist/ui-kit/fesm2022/ui-kit.mjs.map
dist/ui-kit/package.json
dist/ui-kit/types/ui-kit.d.ts
```

and ng-packagr filled in the package manifest:

```json title="dist/ui-kit/package.json"
{
  "name": "ui-kit",
  "version": "0.0.1",
  "peerDependencies": { "@angular/common": "^22.2.0", "@angular/core": "^22.2.0" },
  "dependencies": { "tslib": "^2.3.0" },
  "sideEffects": false,
  "module": "fesm2022/ui-kit.mjs",
  "typings": "types/ui-kit.d.ts",
  "exports": {
    "./package.json": { "default": "./package.json" },
    ".": { "types": "./types/ui-kit.d.ts", "default": "./fesm2022/ui-kit.mjs" }
  },
  "type": "module"
}
```

That's the **Angular Package Format**: one flat ES module bundle per entry point,
bundled type declarations, an `exports` map, `sideEffects: false` for tree-shaking, and
Angular itself as a **peer** dependency so apps don't end up with two copies.

## Consuming it in the same workspace

`ng generate library` added a path mapping to the root `tsconfig.json`:

```json
"paths": { "ui-kit": ["./dist/ui-kit"] }
```

So the app imports it by package name, exactly as it would from npm:

```ts title="projects/shop/src/app/app.ts"
import { Component, signal } from '@angular/core';
import { UiButton, formatPrice } from 'ui-kit';

@Component({
  selector: 'app-root',
  imports: [UiButton],
  template: `
    <h1>Shop</h1>
    <p>Total: {{ total() }}</p>
    <ui-button (click)="checkout()">Checkout</ui-button>
    <ui-button variant="secondary">Keep shopping</ui-button>
  `,
})
export class App {
  protected readonly total = signal(formatPrice(4999));
  protected checkout() {
    console.log('checkout');
  }
}
```

`ng build shop` then succeeded (initial total 202.66 kB). Note the mapping points at
`dist/`: **build the library before the app**, and rebuild it (or run
`ng build ui-kit --watch`) when you change it.

## Testing the library

`ng test ui-kit` runs the library's specs with Vitest, like an app's:

```ts title="projects/ui-kit/src/lib/format-price.spec.ts"
import { formatPrice } from './format-price';

describe('formatPrice', () => {
  it('formats minor units', () => {
    expect(formatPrice(123456)).toBe('$1,234.56');
    // de-DE puts a no-break space (U+00A0) between the amount and the symbol.
    expect(formatPrice(999, 'EUR', 'de-DE')).toBe('9,99\u00a0€');
  });
});
```

Our first version expected `'9,99 €'` with an ordinary space and failed with a message
that looks absurd at first glance:

```text
AssertionError: expected '9,99 €' to be '9,99 €' // Object.is equality
```

`Intl.NumberFormat` uses a non-breaking space in many locales. Shared formatting code is
exactly where such details bite several apps at once — one more reason to put it in a
tested library.

## Publishing

For use outside the workspace, publish the built folder:

```bash
ng build ui-kit
cd dist/ui-kit
npm publish          # to npm, or to a private registry configured in .npmrc
```

Version it with semantic versioning, keep a changelog, and consider secondary entry
points (`ui-kit/forms`, `ui-kit/charts` — each a folder with its own `ng-package.json`)
so apps can import one area without pulling in the others.

## Monorepos beyond the CLI

A CLI workspace is already a small monorepo. As the number of apps and libraries grows,
teams often adopt **Nx**, a third-party build system with Angular support, for:

- a dependency graph of projects and "affected" commands (test/build only what a change
  touches),
- local and remote computation caching,
- lint rules that enforce boundaries between libraries (lesson 06).

It's a significant tool with its own conventions; adopt it when CI times or dependency
tangles are real problems, not preemptively.

## How It Actually Works

The difference between building an **app** and a **library** is the compilation mode.
An application is compiled with full AOT: templates become instruction functions
specific to the exact Angular version in the workspace (Level 1, lesson 01). A library
is compiled in **partial compilation mode**: the compiler emits a *declaration* instead —
calls such as `ɵɵngDeclareComponent({ ... template: "...", ... })` that preserve the
template and metadata in a stable, version-tolerant format. When an application later
imports the library, the application's build runs the **Angular linker**, which turns
those declarations into final instructions for the Angular version the app uses. That's
what lets one published library work across a range of Angular versions (as its
`peerDependencies` range promises), and why you should never publish a library compiled
in full AOT mode.

ng-packagr orchestrates the rest: compile with `tsconfig.lib.prod.json`, bundle each entry
point into a flat ES module with Rollup, bundle the `.d.ts` files, copy assets, and write
the `package.json` with the `exports` map.

## Common mistakes

- **Deep imports** (`ui-kit/src/lib/button`) that bypass the public API and break on the
  next release.
- **Listing Angular as a regular dependency** of a library instead of a peer dependency.
- **Forgetting to rebuild the library** before the app, then debugging stale code.
- **One giant "shared" library** everything depends on — changes to it rebuild and retest
  everything. Split by domain.
- **Putting app-specific code in the library** (routes, environment values).

## Exercise

1. Create a workspace with `--no-create-application`, a `ui-kit` library and an app.
2. Add `UiButton` and a `UiCard` component to the library, export only the components
   from `public-api.ts`, and use them in the app.
3. Add a secondary entry point `ui-kit/format` containing `formatPrice`, build, and
   import it as `import { formatPrice } from 'ui-kit/format'`.
4. Open `dist/ui-kit/fesm2022/ui-kit.mjs` and find the `ɵɵngDeclareComponent` call for
   your button (it was there in ours) — that's the partial-compilation output the linker
   will finish.
5. Write a unit test for `formatPrice` and run it with `ng test ui-kit`.
