---
description: "Level 3 of Angular: zoneless change detection, composition, state, @defer, performance, SSR and hydration, testing, accessibility, security."
---

# Level 3 · Advanced <span class="level-badge">Under the hood</span>

Goal: understand what Angular is doing well enough to make an app fast, robust,
accessible and safe — and to explain *why* a fix works rather than trying things until
the bug goes away.

Levels 1 and 2 were about building features. Level 3 is about the qualities that
separate a demo from a product: rendering only what changed, loading only what's needed,
serving real HTML to users and crawlers, testing behaviour rather than implementation,
working for keyboard and screen-reader users, and keeping attacker-controlled data from
ever becoming code.

Three ideas run through this level:

1. **Measure, then change.** Build output, the stats file, network traces, coverage
   reports and the browser's accessibility tree tell you what's really happening. Every
   lesson shows those outputs from real runs.
2. **Angular 22's defaults are good — know what they are.** Zoneless scheduling, OnPush
   everywhere, automatic sanitisation, hydration with an HTTP transfer cache. Most
   problems come from code that works against a default without realising it.
3. **Prefer declarations over imperative fixes.** `computed` over effects, `@defer` over
   hand-written lazy loading, render modes over `if (isBrowser)` scattered everywhere,
   headless ARIA directives over hand-rolled keyboard handling.

## Modules

1. [Change Detection & Zoneless](01-change-detection-zoneless.md) — when Angular renders, which views it checks, OnPush vs Eager, and what no longer triggers an update
2. [Composition: Projection & Queries](02-content-projection-queries.md) — `ng-content` slots and fallbacks, template outlets, and signal queries
3. [State Management with Signals](03-state-management.md) — where state should live, `linkedSignal`, and NgRx SignalStore
4. [Deferrable Views](04-deferrable-views.md) — `@defer` triggers, prefetching, and the rules that decide what really gets split
5. [Performance](05-performance.md) — bundle analysis, budgets, `NgOptimizedImage` and cheap rendering
6. [SSR, Prerendering & Hydration](06-ssr-hydration.md) — render modes per route, hydration, the transfer cache, response status and allowed hosts
7. [Testing with Vitest & TestBed](07-testing.md) — component and service tests, reading failures, and what coverage does and doesn't tell you
8. [Accessibility & the CDK](08-accessibility-cdk.md) — semantic HTML, `@angular/aria` widgets, focus trapping and live announcements
9. [Security](09-security.md) — sanitisation, `bypassSecurityTrust*`, XSRF, strict CSP and secrets
10. [Project — A Server-Rendered Product Catalog](10-project-catalog.md) — prerendered and server-rendered routes, hydration, deferred sections, and tests

## Before you start

- Finish Level 2, or be comfortable with signal forms, resources, interceptors and the
  router's guards.
- Lessons 06 and 10 use server-side rendering, which needs Node.js on your machine (the
  same one the CLI uses). Everything still runs locally; no hosting account is needed.
- Lesson 03 installs one third-party library (`@ngrx/signals`) and lesson 08 installs
  `@angular/cdk` and `@angular/aria`. Everything else uses packages the CLI already added.

!!! note "Numbers are from one machine"
    Bundle sizes, request counts and coverage percentages in this level were captured from
    real builds and runs on Angular 22.2. Your numbers will differ with versions, code and
    hardware; the point is how to produce and read them, not the specific values.
