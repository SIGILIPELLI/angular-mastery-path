---
description: "Level 1 of Angular: the CLI, standalone components, templates, signals, control flow, inputs and outputs, services, routing and HttpClient."
---

# Level 1 · Entry <span class="level-badge">Foundations</span>

Goal: build a small multi-page Angular application from an empty folder and be able to
explain what the framework is doing at each step — which class became a component, how a
value in the class reached the DOM, why the screen updated when a signal changed, where a
service instance came from, and how a URL was turned into a component on screen.

Angular is a complete framework rather than a view library, so even Level 1 touches a lot
of surface: the CLI, components, templates, reactivity, dependency injection, the router and
the HTTP client. The trick is that each of these is small on its own. By the end of the level
you will have used all of them together in one app.

Three ideas run through the whole level:

1. **Everything is a standalone component with explicit imports.** A component lists the
   other components, directives and pipes its template uses in its `imports` array. There
   is no hidden global registry — if a tag is not imported, the compiler tells you.
2. **State lives in signals.** A `signal()` holds a value, `computed()` derives from it and
   the template reads it by calling it: `{{ count() }}`. When a signal changes, Angular
   knows exactly which views read it and refreshes only those.
3. **Classes ask for what they need with `inject()`.** Services, the router, the HTTP
   client — you do not construct them yourself. Angular's injector creates them once and
   hands the same instance to everyone who asks.

## Modules

1. [What Angular Is](01-what-angular-is.md) — what the framework includes, the compiler, the release cycle and when Angular is a good fit
2. [The CLI & Project Structure](02-cli-project-structure.md) — `ng new`, what every generated file does, and `ng serve` / `build` / `test` / `generate`
3. [Components & Templates](03-components-templates.md) — the `@Component` decorator, interpolation, property, attribute, class and event bindings, template reference variables and style encapsulation
4. [Signals: signal, computed, effect](04-signals-basics.md) — writable signals, derived state, side effects and the rules that keep them predictable
5. [Template Control Flow](05-control-flow.md) — `@if`, `@for` with `track`, `@switch` and `@let`
6. [Inputs, Outputs & model()](06-inputs-outputs-model.md) — passing data into components, emitting events out, and two-way binding
7. [Services & Dependency Injection](07-services-di.md) — `@Service`, `@Injectable`, `inject()` and how the root injector hands out singletons
8. [Routing Basics](08-routing-basics.md) — `provideRouter`, `routerLink`, route parameters as inputs and lazy-loaded components
9. [HTTP with HttpClient](09-httpclient.md) — typed GET and POST requests, query parameters, error handling and testing without a server
10. [Project — A Reading List App](10-project-reading-list.md) — a routed app with a signal-based store, persistence, a reusable rating control and live search

## What you need before starting

- **Node.js** in one of the ranges Angular 22 supports. The CLI's `package.json` declares
  `^22.22.3 || ^24.15.0 || >=26.0.0`; check yours with `node -v`.
- **TypeScript basics**: classes, interfaces, union types and generics. Angular code is
  TypeScript everywhere. If you need a refresher, work through
  [TypeScript Mastery Path](https://sigilipelli.github.io/typescript-mastery-path/) first.
- **HTML, CSS and a little JavaScript in the browser** — events, the DOM, `fetch`.
  [JavaScript Mastery Path](https://sigilipelli.github.io/javascript-mastery-path/) covers
  this if you are new to it.
- A terminal and an editor. No accounts or cloud services are needed anywhere in Levels 1–3.

## How the code is organized

Code blocks are titled with a path relative to the project root, for example
`src/app/book-store.ts`. The CLI in Angular 20 and later generates files without the old
`.component.ts` / `.service.ts` suffixes (`book-card.ts`, class `BookCard`), and this course
follows that convention. Blocks labelled `text` show terminal or browser output captured
from a real run on Angular 22.2.0 with Node.js 26.

!!! note "About versions"
    This course was written against **Angular 22.2** (CLI 22.2.0, TypeScript 6.0,
    Vitest 5). Angular ships a major version roughly every six months, and recent majors
    changed several defaults — new projects are zoneless, components use `OnPush` change
    detection unless told otherwise, and the CLI now generates services with the new
    `@Service()` decorator. Where a lesson depends on a version-specific default it says
    so. `ng version` tells you exactly what you have installed, and the documentation for
    each major lives at `vNN.angular.dev`.
