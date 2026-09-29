---
description: "Level 2 of Angular: reactive and signal forms, DI in depth, RxJS and interop, resources, guards, resolvers, interceptors, pipes and directives."
---

# Level 2 · Intermediate <span class="level-badge">Real apps</span>

Goal: build a data-driven application the way production Angular teams do — forms with
real validation and server errors, data loaded from APIs with proper loading and error
states, URLs that capture what the user is looking at, and HTTP concerns handled once, in
one place.

Level 1 gave you the pieces. Level 2 is about the parts of Angular that make those pieces
scale: two form systems, the full dependency-injection model, enough RxJS to handle
events over time, the resource APIs that turn async data into signals, and the router and
HTTP features that most apps lean on every day.

Three ideas run through this level:

1. **Keep one source of truth.** Signal forms edit your model directly; filters live in
   the URL; resources derive their request from signals. When there's only one copy of
   the data, there's nothing to keep in sync.
2. **Signals for state, observables for events.** Use RxJS where time matters — debounce,
   cancel, retry — and convert to signals at the edge, where templates read values.
3. **Put cross-cutting concerns in one place.** Auth headers, retries, error mapping and
   busy indicators belong in interceptors; guards decide navigation; services own writes.

## Modules

1. [Reactive Forms](01-reactive-forms.md) — typed `FormGroup`s, built-in and custom validators, cross-field rules, `FormArray`
2. [Signal Forms](02-signal-forms.md) — `form()` over a model signal, schema rules, `[formField]`, `submit()` with server errors
3. [Dependency Injection in Depth](03-di-in-depth.md) — tokens, provider recipes, multi providers, the injector tree, `viewProviders`
4. [RxJS for Angular Developers](04-rxjs-essentials.md) — cold vs hot, Subjects, operators, and the four flattening strategies
5. [Signals and RxJS Interop](05-signals-rxjs-interop.md) — `toSignal`, `toObservable`, `takeUntilDestroyed`, `rxResource`
6. [Async Data with resource & httpResource](06-resource-httpresource.md) — status, cancellation, errors, reloads, local edits, testing
7. [Router in Depth](07-router-in-depth.md) — child routes, `canActivate`/`canMatch`/`canDeactivate`, resolvers, route providers
8. [HTTP Interceptors & Error Handling](08-interceptors-errors.md) — auth headers, `HttpContext`, busy counters, safe retries, error mapping
9. [Pipes & Directives](09-pipes-directives.md) — built-in pipes with real output, pure custom pipes, attribute directives, `hostDirectives`
10. [Project — An Issue Tracker](10-project-issue-tracker.md) — a CRUD app with URL filters, signal forms, resources and an in-memory API

## Before you start

- Finish Level 1, or be comfortable with standalone components, signals, `inject()`, the
  router basics and `HttpClient`.
- Everything still runs locally. The Level 2 project fakes its REST API with an
  interceptor, so no server or account is needed.
- If RxJS is completely new to you, read lesson 04 before lesson 01 — reactive forms
  expose observables and the examples use a few operators.

!!! note "Two form systems"
    This level teaches both reactive forms and signal forms on purpose. Signal forms
    became stable in Angular 22 and are the better default for new code, but reactive
    forms are in almost every existing Angular codebase and are not deprecated. You will
    need to read and maintain both.
