---
description: "Lazy-load parts of an Angular template with @defer: triggers, prefetching, placeholder, loading and error blocks, and what gets split."
---

# 04 · Deferrable Views

Route-level lazy loading (Level 1, lesson 08) splits your app by **page**. But pages
themselves often contain heavy pieces the user may never look at: a chart below the fold,
a comments section, a rich-text editor behind an "Edit" button, a map in a tab. `@defer`
splits by **part of a template**: the component code inside the block is moved into a
separate chunk and downloaded only when a trigger fires.

## A worked example

```ts title="src/app/report.ts"
import { Component, signal } from '@angular/core';
import { HeavyChart } from './heavy-chart';
import { Comments } from './comments';

@Component({
  selector: 'app-report',
  imports: [HeavyChart, Comments],
  template: `
    <h1>Report</h1>

    @defer (on interaction(showBtn); prefetch on hover(showBtn)) {
      <app-heavy-chart [points]="points()" />
    } @placeholder {
      <p>Chart appears when you click “Show chart”.</p>
    } @loading (after 100ms; minimum 300ms) {
      <p>Loading chart…</p>
    } @error {
      <p role="alert">Couldn't load the chart.</p>
    }
    <button #showBtn type="button">Show chart</button>

    <div style="height: 2000px">Long article…</div>

    @defer (on viewport) {
      <app-comments />
    } @placeholder (minimum 200ms) {
      <p>Comments</p>
    }
  `,
})
export class Report {
  protected readonly points = signal([3, 1, 4, 1, 5, 9]);
}
```

`ng build` listed the deferred components as their own lazy chunks:

```text
chunk-BreFoQpF.js   | defer-demo    |   1.43 kB |               713 bytes
chunk-TRR4wwOy.js   | heavy-chart   | 444 bytes |               444 bytes
chunk-C2Z0ns_5.js   | comments      | 332 bytes |               332 bytes
```

(Our stand-in components are tiny; in a real app this is where a 200 kB chart library
would go.) We then loaded the page in Chromium, recording every JavaScript request:

```text
loaded                | js requests: 3 | chart: 0 placeholder: 1 | comments: 0 placeholder: 1
hovered the button    | js requests: 4 | new: chunk-TRR4wwOy.js   ← prefetch on hover
clicked (50 ms later) | js requests: 4 | chart: 1 placeholder: 0  ← no new request needed
scrolled to bottom    | js requests: 5 | new: chunk-C2Z0ns_5.js   | comments: 1
```

Hovering fetched the chart's code *before* the click, so clicking rendered it
immediately — and because the code was already there, the `@loading` block never had to
appear. The comments chunk wasn't requested until its placeholder scrolled into view.

## Triggers

| Trigger | Fires when |
|---------|-----------|
| `on idle` (the default) | the browser is idle (`requestIdleCallback`) |
| `on viewport` | the placeholder (or a referenced element) enters the viewport |
| `on interaction` | the user clicks or presses a key on the placeholder or referenced element |
| `on hover` | pointer enters / focus moves to it |
| `on immediate` | right after the surrounding view renders |
| `on timer(2s)` | after a delay |
| `when condition` | an expression becomes truthy (checked with the template) |

`interaction(showBtn)`, `hover(showBtn)` and `viewport(el)` accept a **template reference**
to use another element as the trigger. Without one, they watch the `@placeholder`
content, so that block must contain exactly one root element.

Multiple triggers are OR-ed: `@defer (on viewport; on timer(5s))`. Once a block has
loaded, it stays loaded — `when` going falsy again doesn't unload it.

**Prefetching** separates *downloading* from *rendering*: `prefetch on idle`,
`prefetch on hover(...)`, `prefetch when ...`. A good default for below-the-fold content
is `@defer (on viewport; prefetch on idle)`.

## The sub-blocks

- **`@placeholder`** — shown before the trigger. Keep it cheap and the same size as the
  final content to avoid layout shift. `minimum 200ms` keeps it on screen at least that
  long to avoid flicker.
- **`@loading`** — shown while the chunk downloads. `after 100ms` waits before showing it
  (fast loads never flash a spinner); `minimum 300ms` keeps it visible long enough to read
  once shown.
- **`@error`** — shown if loading the chunk fails (e.g. offline).

## What actually gets deferred

A component, directive or pipe is moved into the deferred chunk only if:

1. it is **standalone**, and
2. it is **not referenced anywhere else in the same file** outside `@defer` blocks —
   not in the template, and not in class code.

Break either rule and Angular silently loads it eagerly. We tried it: adding one
`<app-comments />` above the `@defer` removed the `comments` chunk from the build output
— its code was folded into `defer-demo` (which grew from 1.43 kB to 1.68 kB) — and the
compiler printed no warning. After adding a `@defer`, **check the build output** for the
chunk you expect.

Everything else inside the block — plain HTML, bindings to the parent's signals — is
still part of the parent; only the dependencies' code moves.

## `@defer` and server-side rendering

On the server, `@defer` blocks render their **`@placeholder`** (triggers like viewport
and hover make no sense without a browser). With hydration enabled (lesson 06) you can go
further with *incremental hydration*: `@defer (hydrate on viewport)` renders the real
content on the server but delays making it interactive in the browser until the trigger.
In Angular 22 incremental hydration is on by default when you use `provideClientHydration()`
— the `withIncrementalHydration()` helper is deprecated for that reason.

## How It Actually Works

At compile time, the Angular compiler rewrites each `@defer` block:

- The block's dependencies (the components, directives and pipes used inside it that
  pass the two rules) are removed from the component's static `imports` metadata and
  replaced with a **dependency function**: `() => [import('./heavy-chart').then(m => m.HeavyChart)]`.
  The bundler sees those dynamic `import()`s and emits separate chunks — exactly like
  `loadComponent` in routes.
- The block itself becomes a small state machine in the template with states
  *placeholder → loading → complete* (or *error*), each backed by an embedded view.

At runtime, triggers are set up when the placeholder renders: an `IntersectionObserver`
for `viewport`, event listeners for `interaction`/`hover`, `requestIdleCallback` for
`idle`, a timer for `timer`, and a reactive check for `when`. Prefetch triggers call the
dependency function early and cache the promise. When the main trigger fires, Angular
awaits the (possibly already resolved) promise, then swaps the placeholder view for the
real content view, honouring the `after`/`minimum` timings.

## Common mistakes

- **Referencing the deferred component elsewhere** in the file, silently cancelling the
  split.
- **Deferring content above the fold**, making the most important part of the page
  appear last and shifting layout.
- **Placeholders with a different size** from the final content, causing layout shift
  (a Core Web Vitals problem — lesson 05).
- **Multiple root elements in a placeholder** used as an implicit trigger.
- **Deferring tiny components.** Every chunk is an extra request; defer things that are
  genuinely heavy or rarely seen.

## Exercise

1. Take a page with a heavy component (use a real library such as a charting package, or
   simulate one with a large static data file) and wrap it in
   `@defer (on viewport; prefetch on idle)`.
2. Compare the initial bundle size in `ng build` before and after.
3. In DevTools' Network panel, confirm when the chunk is requested with and without the
   prefetch trigger.
4. Add a `@placeholder` with the same height as the final content and verify in the
   Performance panel that the layout doesn't shift when it swaps.
5. Deliberately reference the deferred component outside the block and confirm the chunk
   disappears from the build output.
