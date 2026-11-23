---
title: "Angular Cheat Sheet: Quick Reference, Beginner to Advanced"
description: "A free Angular cheat sheet: 24 quick-reference tables from every level of the Angular Mastery Path, each linked to its full lesson."
---

# Angular Cheat Sheet

Quick-reference tables from **24 lessons** of the free Angular Mastery Path, in course order from beginner to advanced. Each section links to the full lesson with explanations and runnable examples.

## Level 1 · Entry

### [01 · What Angular Is](level-1/01-what-angular-is.md)

*What is in the box*

| Concern | What Angular provides | Package |
|---------|----------------------|---------|
| UI | Components, templates, directives, pipes | `@angular/core`, `@angular/common` |
| Reactivity | Signals (`signal`, `computed`, `effect`, `linkedSignal`, `resource`) | `@angular/core` |
| Wiring | Dependency injection (`inject()`, providers, hierarchical injectors) | `@angular/core` |
| Navigation | Router with lazy loading, guards and resolvers | `@angular/router` |
| Forms | Reactive forms, template-driven forms, and signal forms | `@angular/forms` |
| Server calls | `HttpClient` with interceptors, plus `httpResource` | `@angular/common/http` |
| Server rendering | SSR, prerendering and hydration | `@angular/ssr` |
| Tooling | CLI, dev server, production build (esbuild + Vite), test runner integration | `@angular/cli`, `@angular/build` |
| Extras | Component Dev Kit, Material components, accessibility primitives, i18n | `@angular/cdk`, `@angular/material`, `@angular/aria`, `@angular/localize` |

### [02 · The CLI &amp; Project Structure](level-1/02-cli-project-structure.md)

*Creating a project*

| Flag | What it does | Default in 22 |
|------|--------------|---------------|
| `--style` | `css`, `scss`, `sass`, `less` or `tailwind` | `css` |
| `--ssr` | Adds server-side rendering and prerendering (Level 3, lesson 06) | asked |
| `--routing` | Generates `app.routes.ts` and wires the router | yes |
| `--test-runner` | `vitest` or `karma` | `vitest` |
| `--zoneless` | App does not load `zone.js` | yes for new apps |
| `--inline-template` / `--inline-style` | Root component uses `template:` / `styles:` instead of separate files | no |
| `--skip-git`, `--skip-install` | Skip `git init` / `npm install` | no |

### [03 · Components &amp; Templates](level-1/03-components-templates.md)

*Anatomy of a component*

| Property | Purpose |
|----------|---------|
| `selector` | The element name that creates this component (`<app-profile-card />`). Prefix with your app's prefix (`app-` by default) to avoid clashing with real or future HTML elements. |
| `template` / `templateUrl` | Inline template string, or path to an `.html` file. Use one or the other. |
| `styles` / `styleUrl` | Inline CSS string (or array), or path to a stylesheet. |
| `imports` | Components, directives and pipes this template uses. |
| `host` | Bindings and listeners on the component's own element (the `<app-profile-card>` tag). |
| `changeDetection` | Change detection strategy — `OnPush` by default in Angular 22 (Level 3, lesson 01). |

### [10 · Project — A Reading List App](level-1/10-project-reading-list.md)

| Lesson | Where it shows up |
|--------|-------------------|
| 02 CLI | `ng new`, `ng build`, `ng test` |
| 03 Templates | bindings and events in every component |
| 04 Signals | the store's `signal`, `computed` and `effect` |
| 05 Control flow | `@for` with `track`, `@if`/`@else`, `@empty` |
| 06 Inputs/outputs | `BookCard` inputs and outputs, `StarRating` with `model()` |
| 07 Services | `BookStore` and `OpenLibrary` with `@Service()` |
| 08 Routing | list, detail and search routes; `:id` bound to an input; lazy chunks |
| 09 HttpClient | the Open Library search |

## Level 2 · Intermediate

### [01 · Reactive Forms](level-2/01-reactive-forms.md)

| System | Model lives in | Best for |
|--------|----------------|----------|
| **Template-driven** (`FormsModule`, `ngModel`) | The template | Very small forms; older code |
| **Reactive** (`ReactiveFormsModule`) | A tree of `FormControl` / `FormGroup` objects in the class | Most existing production code; complex dynamic forms |
| **Signal forms** (`@angular/forms/signals`) | A signal holding your data model | New code in Angular 22+ (next lesson) |

### [02 · Signal Forms](level-2/02-signal-forms.md)

*The model is just a signal*

| Signal | Meaning |
|--------|---------|
| `value` | The field's value (writable — `f.username().value.set('ada')` updates the model) |
| `errors` | Array of `{ kind, message? }` objects for this field |
| `valid` / `invalid` / `pending` | Validation status, `pending` while async rules run |
| `touched` / `dirty` | Interaction state |
| `disabled` / `readonly` / `hidden` | Computed from your schema rules |
| `required`, `min`, `max`, `minLength`, `maxLength` | Metadata derived from validators, useful for hints and native attributes |
| `submitting` | True while a `submit()` action runs |

### [03 · Dependency Injection in Depth](level-2/03-di-in-depth.md)

*Resolution options*

| Option | Effect |
|--------|--------|
| `{ optional: true }` | Return `null` instead of throwing `NG0201` when nothing provides it |
| `{ self: true }` | Look only at the current element's injector |
| `{ skipSelf: true }` | Start at the parent — how a nested component finds "the enclosing one of me" (a nested menu finding its parent menu) |
| `{ host: true }` | Stop the search at the host component's boundary |

### [04 · RxJS for Angular Developers](level-2/04-rxjs-essentials.md)

*The operators you'll use weekly*

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

### [05 · Signals and RxJS Interop](level-2/05-signals-rxjs-interop.md)

| Function | Direction | Typical use |
|----------|-----------|-------------|
| `toSignal(obs$)` | Observable → Signal | Show an HTTP or router stream in a template |
| `toObservable(sig)` | Signal → Observable | Feed signal changes into `debounceTime`/`switchMap` |
| `takeUntilDestroyed()` | Cleanup | End a manual subscription with the component |
| `rxResource({ params, stream })` | Signal params → Observable loader → resource | Async data with status (next lesson covers resources in depth) |
| `outputFromObservable(obs$)` / `outputToObservable(ref)` | Outputs ↔ Observables | Expose or consume component outputs as streams |

### [06 · Async Data with resource &amp; httpResource](level-2/06-resource-httpresource.md)

| API | Loader returns | Package |
|-----|----------------|---------|
| `resource({ params, loader })` | a `Promise` | `@angular/core` |
| `rxResource({ params, stream })` | an `Observable` | `@angular/core/rxjs-interop` |
| `httpResource(() => url or request)` | — it makes an `HttpClient` GET for you | `@angular/common/http` |

### [09 · Pipes &amp; Directives](level-2/09-pipes-directives.md)

*Built-in pipes, with real output*

| Template | Output |
|----------|--------|
| `{{ when \| date }}` | `Sep 28, 2026` |
| `{{ when \| date: 'yyyy-MM-dd HH:mm' : 'UTC' }}` | `2026-09-28 14:05` |
| `{{ when \| date: 'fullDate' : 'UTC' }}` | `Monday, September 28, 2026` |
| `{{ price \| currency }}` | `$1,234.50` |
| `{{ price \| currency: 'EUR' }}` | `€1,234.50` |
| `{{ price \| currency: 'INR' : 'symbol' : '1.0-0' }}` | `₹1,235` |
| `{{ ratio \| percent: '1.1-1' }}` | `45.7%` |
| `{{ big \| number }}` | `9,876,543.21` |
| `{{ pi \| number: '1.2-2' }}` | `3.14` |
| `{{ 'the lord of the rings' \| titlecase }}` | `The Lord Of The Rings` |
| `{{ 'Angular' \| uppercase }}` | `ANGULAR` |
| `{{ ['a','b','c','d'] \| slice: 1 : 3 }}` | `b,c` |
| `{{ { a: 1 } \| json }}` | `{ "a": 1 }` |
| `@for (e of { zoe: 3, adam: 5 } \| keyvalue; ...)` | `adam=5 zoe=3` (sorted by key) |

### [10 · Project — An Issue Tracker](level-2/10-project-issue-tracker.md)

| Lesson | Where it shows up |
|--------|-------------------|
| 02 Signal forms | `NewIssuePage`: schema validation, `submit()`, server error on `title` |
| 03 DI | `Busy` service shared by an interceptor and the header |
| 04 RxJS | the fake backend (`of`, `delay`, `timer`, `switchMap`), `finalize` |
| 06 Resources | `httpResource` for the list and the detail page |
| 07 Router | lazy routes, query params as inputs, `canDeactivate` |
| 08 Interceptors | busy counter, error mapping, the fake backend |
| 09 Pipes & directives | `DatePipe` with explicit time zone, host bindings in `StatusBadge` |

## Level 3 · Advanced

### [02 · Composition: Projection &amp; Queries](level-3/02-content-projection-queries.md)

*Queries as signals*

| Query | Looks in | Returns |
|-------|----------|---------|
| `viewChild(x)` / `viewChild.required(x)` | this component's own template | `Signal<T \| undefined>` / `Signal<T>` |
| `viewChildren(x)` | own template | `Signal<readonly T[]>` |
| `contentChild(x)` / `.required` | projected content | `Signal<T \| undefined>` / `Signal<T>` |
| `contentChildren(x)` | projected content | `Signal<readonly T[]>` |

### [03 · State Management with Signals](level-3/03-state-management.md)

*Where should this state live?*

| Kind of state | Example | Put it in |
|---------------|---------|-----------|
| Local UI | is the menu open, current tab | a `signal` in the component |
| Derived | filtered list, totals, "can submit" | `computed` — never stored separately |
| URL | search terms, filters, selected id, page | route/query params (read as inputs) |
| Server data | issues, products, the current user | a resource (`httpResource`) or a service that loads it |
| Form draft | fields being edited | signal forms' model (Level 2) |
| Shared client state | cart, preferences, auth session | a root service with private writable signals |
| Feature-scoped shared state | a wizard's steps | a service provided on the feature's route or component |

### [04 · Deferrable Views](level-3/04-deferrable-views.md)

*Triggers*

| Trigger | Fires when |
|---------|-----------|
| `on idle` (the default) | the browser is idle (`requestIdleCallback`) |
| `on viewport` | the placeholder (or a referenced element) enters the viewport |
| `on interaction` | the user clicks or presses a key on the placeholder or referenced element |
| `on hover` | pointer enters / focus moves to it |
| `on immediate` | right after the surrounding view renders |
| `on timer(2s)` | after a delay |
| `when condition` | an expression becomes truthy (checked with the template) |

### [05 · Performance](level-3/05-performance.md)

*Keep rendering cheap*

| Symptom | Fix |
|---------|-----|
| A method call in a template doing real work (`{{ total(items) }}`) | `computed()` or a pure pipe — computed once per change, not per render |
| List rows re-created after every fetch | `track` by a stable id (Level 1, lesson 05) |
| Thousands of rows | paginate, or virtual scrolling (`@angular/cdk/scrolling`'s `cdk-virtual-scroll-viewport`) |
| An effect that writes signals and triggers more renders | derive with `computed`/`linkedSignal` instead |
| Hot `(scroll)`/`(mousemove)` handlers | throttle, or listen outside templates and only write a signal when a value changes |
| Heavy work on the main thread | `ng generate web-worker` and move it to a worker |

### [06 · SSR, Prerendering &amp; Hydration](level-3/06-ssr-hydration.md)

*Render modes per route*

| Mode | HTML produced | Good for |
|------|---------------|----------|
| `Prerender` (SSG) | once, at `ng build` | pages identical for every visitor: marketing, docs, product pages |
| `Server` (SSR) | on each request | pages that depend on the request: search, personalised or fast-changing data |
| `Client` (CSR) | none — the empty shell | pages that depend on browser-only state, or aren't worth rendering |

### [07 · Testing with Vitest &amp; TestBed](level-3/07-testing.md)

*What to test where*

| Level | Tool | Tests |
|-------|------|-------|
| Pure logic | plain Vitest, no TestBed | pipes, validators, pure functions |
| Services | `TestBed.inject` | stores, API clients (with `HttpTestingController`) |
| Components | `TestBed.createComponent` | rendering, inputs, outputs, user interaction |
| Routing | `RouterTestingHarness` | guards, resolvers, input binding (Level 2, lesson 07) |
| Whole app in a browser | Playwright / Cypress | critical user journeys, end to end |

### [10 · Project — A Server-Rendered Product Catalog](level-3/10-project-catalog.md)

| Lesson | Where it shows up |
|--------|-------------------|
| 01 Change detection | signals everywhere; nothing relies on zone-triggered updates |
| 02 Composition | a reusable `ProductCard` |
| 03 State | a `Cart` service store with computed totals |
| 04 `@defer` | related products with `hydrate on viewport` |
| 05 Performance | per-route lazy chunks; budgets from the CLI defaults |
| 06 SSR | three render modes, a real 404, the HTTP transfer cache, browser-only storage |
| 07 Testing | component and service tests, including server-only behaviour |
| 09 Security | allowed hosts for the server engine |

## Level 4 · Master

### [01 · The Build System &amp; Configurations](level-4/01-build-system.md)

*Other options you will reach for*

| Option | Use it for |
|--------|-----------|
| `baseHref` | deploying under a sub-path (`/app/`) — Level 4, lesson 07 |
| `outputHashing` | `all` (default in production) for cache-busting file names |
| `sourceMap` | `{ "scripts": true, "hidden": true }` to generate maps for error tracking without linking them from the bundle |
| `subresourceIntegrity` | add `integrity` hashes to script tags |
| `allowedCommonJsDependencies` | silence the CommonJS warning for a dependency you've accepted |
| `externalDependencies` | leave an import unbundled (import maps, micro-frontends) |
| `security.autoCsp` | generated strict CSP — Level 3, lesson 09 |

### [04 · Angular Elements &amp; Micro-Frontends](level-4/04-elements-microfrontends.md)

*Using it from plain HTML*

| Angular | Custom element |
|---------|----------------|
| `input()` / `model()` named `label` | **attribute** `label` and **property** `label` (camelCase inputs map to dash-case attributes: `maxValue` → `max-value`) |
| `output()` / `model()` change named `valueChange` | **DOM `CustomEvent`** `valueChange`, payload in `event.detail` |
| component creation | when the element is **connected** to the document |
| component destruction | shortly after the element is **disconnected** |

### [05 · Upgrading &amp; Migrations](level-4/05-upgrading-migrations.md)

*Modernisation migrations*

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

### [07 · Deployment](level-4/07-deployment.md)

| App type | Output | Host on |
|----------|--------|---------|
| Client-rendered (CSR) | `dist/<app>/browser/` — static files | any static host or CDN: object storage + CDN, GitHub Pages, Netlify, Vercel, Firebase Hosting, Nginx |
| Prerendered only | static files, one `index.html` per route | same as above |
| Server-rendered routes | `browser/` + `server/server.mjs` | a Node runtime: a VM, a container platform, or a platform with an Angular SSR adapter |

### [09 · Production Readiness Review](level-4/09-production-readiness.md)

*Build &amp; delivery*

| Check | Evidence |
|-------|----------|
| Production build is the default and is what gets deployed | `defaultConfiguration: "production"`; CI builds without `-c development` (L4-01) |
| Budgets fail the build on regressions | `initial` budget with `maximumError`, set just above current size (L3-05) |
| Environments hold no secrets | review of `src/environments/*` (L4-01, L3-09) |
| Deep links work on the real host | reload `/some/inner/route` in production returns the app with 200 (L4-07) |
| Cache headers: hashed files immutable, HTML revalidated | `curl -I` output (L4-07) |
| Old chunks survive a deploy / chunk-load errors reload once | deploy procedure, `withNavigationErrorHandler` (L4-07, L4-08) |

---

Go deeper with the full [Angular Mastery Path](index.md) curriculum or test yourself with the [interview questions](interview-questions.md).
