---
description: "Angular SSR in v22: per-route render modes, prerendering, hydration, the HTTP transfer cache, 404 status, allowed hosts, browser-only code."
---

# 06 · SSR, Prerendering & Hydration

A client-rendered Angular app sends an almost empty HTML page and builds everything in the
browser. That's fine for an internal dashboard, but for public pages it means a blank
screen until JavaScript downloads and runs, and crawlers and link previews see very
little. **Server-side rendering (SSR)** runs your app on a server (or at build time) to
produce real HTML, and **hydration** lets the browser adopt that HTML instead of throwing
it away.

This lesson uses a small catalog app — prerendered product pages, a search page rendered
per request, and a client-only cart — and shows what each part actually returned when we
ran it on Angular 22.2 with Node.js 26.

## Setting it up

```bash
npx @angular/cli@22 new catalog --ssr        # new project
ng add @angular/ssr                          # or add to an existing one
```

That adds `@angular/ssr`, `@angular/platform-server` and Express, plus four files:

- `src/main.server.ts` — bootstraps the app on the server.
- `src/app/app.config.server.ts` — merges `provideServerRendering(withRoutes(serverRoutes))`
  into your app config.
- `src/app/app.routes.server.ts` — **how** each route is rendered (below).
- `src/server.ts` — an Express server that serves static files and hands every other
  request to `AngularNodeAppEngine`.

The browser config gains `provideClientHydration()`.

## Render modes per route

```ts title="src/app/app.routes.server.ts"
import { PrerenderFallback, RenderMode, ServerRoute } from '@angular/ssr';
import { PRODUCTS } from './catalog/products.data';

export const serverRoutes: ServerRoute[] = [
  // Static marketing page: rendered once at build time.
  { path: '', renderMode: RenderMode.Prerender },
  // One HTML file per known product at build time; unknown slugs are rendered on request.
  {
    path: 'products/:slug',
    renderMode: RenderMode.Prerender,
    getPrerenderParams: async () => PRODUCTS.map((p) => ({ slug: p.slug })),
    fallback: PrerenderFallback.Server,
  },
  // Depends on the query string: rendered on the server for every request.
  { path: 'search', renderMode: RenderMode.Server },
  // Depends on localStorage: nothing useful to render on the server.
  { path: 'cart', renderMode: RenderMode.Client },
  { path: '**', renderMode: RenderMode.Server },
];
```

| Mode | HTML produced | Good for |
|------|---------------|----------|
| `Prerender` (SSG) | once, at `ng build` | pages identical for every visitor: marketing, docs, product pages |
| `Server` (SSR) | on each request | pages that depend on the request: search, personalised or fast-changing data |
| `Client` (CSR) | none — the empty shell | pages that depend on browser-only state, or aren't worth rendering |

`ng build` reported `Prerendered 7 static routes.` — the home page plus six products —
and wrote them as real files (`dist/catalog/browser/products/arc-floor-lamp/index.html`
and so on) listed in `prerendered-routes.json`.

## What the server actually sent

We started the built server (`node dist/catalog/server/server.mjs`) and fetched each
route with `curl`:

```text
/                        200  title "Home · Desk & Lamp"            ng-server-context="ssg"
/products/arc-floor-lamp 200  title "Arc Floor Lamp · Desk & Lamp"  ng-server-context="ssg"
/products/no-such-thing  404  title "Not found · Desk & Lamp"       ng-server-context="ssr"
/search?q=lamp           200  "2 result(s) for “lamp” Arc Floor Lamp … Clamp Desk Lamp"   ng-server-context="ssr"
/cart                    200  empty <app-root>, no ng-state                             (client-rendered)
```

- Prerendered pages contain full content and per-page `<title>` and meta description, so
  crawlers and link previews see them without running JavaScript.
- The unknown product fell back to on-demand SSR and returned a **real 404 status**,
  which search engines need. The component sets it through the `RESPONSE_INIT` token,
  which is only provided during server rendering:

```ts title="src/app/catalog/product-page.ts (excerpt)"
private readonly responseInit = inject(RESPONSE_INIT, { optional: true });

constructor() {
  effect(() => {
    const p = this.product();
    if (p) {
      this.seo.set(p.name, p.summary);
    } else {
      this.seo.set('Not found', 'This product does not exist.');
      if (this.responseInit) this.responseInit.status = 404; // only set during SSR
    }
  });
}
```

(`REQUEST` and `REQUEST_CONTEXT` are the matching tokens for reading the incoming request
on the server.)

### Allowed hosts: the first thing that will break

Our very first request to the freshly built server failed:

```text
ERROR: Bad Request ("http://localhost:4391/").
Header "host" with value "localhost:4391" is not allowed.
```

Angular's server engine checks the `Host` header against an allow-list to prevent
**server-side request forgery** (a forged host could make the server fetch from or build
links to an attacker's domain). Configure the list in `angular.json` under
`projects.<name>.architect.build.options.security.allowedHosts`, pass `allowedHosts` to
`new AngularNodeAppEngine({...})`, or set the `NG_ALLOWED_HOSTS` environment variable —
which is what we did (`NG_ALLOWED_HOSTS=localhost`). In production, list your real
domain(s).

## Hydration

Look inside the server HTML and you'll find hydration annotations:

```html
<app-root ng-version="22.2.0" ngh="0" ng-server-context="ssg">…</app-root>
<script id="ng-state" type="application/json">{"__nghData__":[{},{"t":{"4":"t0"},"c":{"4":[{"i":"t0","r":1,"x":6}]}}, …]}</script>
```

When the browser bootstraps, `provideClientHydration()` makes Angular **walk the existing
DOM instead of creating new nodes**. `ngh` attributes point into `__nghData__`, which
records the structure the server produced — for example that a `@for` container rendered
6 items (`"x":6`) — so the client can match its views to existing elements without
guessing. Event listeners are attached to the existing nodes; nothing flickers or
re-renders.

Features you can turn on or off with `provideClientHydration(...)`:

- **HTTP transfer cache** (on by default) — see below. `withNoHttpTransferCache()`
  disables it; `withHttpTransferCacheOptions({...})` tunes it.
- **Incremental hydration** (on by default in 22) — `@defer (hydrate on viewport)`
  blocks stay dehydrated until their trigger. `withNoIncrementalHydration()` turns it off.
- **Event replay** — `withEventReplay()` records clicks that happen before the app has
  hydrated and replays them afterwards, so an early "Add to cart" isn't lost. The catalog
  enables it.

## The HTTP transfer cache

The search page loads results with `httpResource('/api/search?q=…')`, served by an
Express route in `server.ts`. We counted calls on the server and requests in the browser:

```text
first load of /search?q=lamp   server API calls: +1   browser /api requests: []
new search for "desk" (client) browser /api requests: ['/api/search?q=desk']
```

During SSR, `HttpClient` made the request, and the response was serialised into the page
(you can see it in the `ng-state` script, keyed by a hash of the request). When the
browser hydrated, the identical request was answered from that embedded cache — **no
second request**, and no flash of a loading state. Later requests go to the network as
usual. By default only `GET` and `HEAD` requests without auth headers are cached.

## Code that must only run in the browser

On the server there is no `window`, `document` layout, `localStorage` or
`IntersectionObserver`. Three tools:

- **`afterNextRender(() => ...)` / `afterEveryRender`** — run only in the browser, after
  rendering. The right place for DOM measurement, focus, and third-party widgets.
- **`isPlatformBrowser(inject(PLATFORM_ID))`** — branch in services. The catalog's cart
  only reads and writes `localStorage` in the browser:

```ts title="src/app/catalog/cart.ts (excerpt)"
private readonly isBrowser = isPlatformBrowser(inject(PLATFORM_ID));
private readonly _lines = signal<Line[]>(this.isBrowser ? readStorage() : []);

constructor() {
  if (this.isBrowser) {
    effect(() => localStorage.setItem(KEY, JSON.stringify(this._lines())));
  }
}
```

- **`RenderMode.Client`** — for a whole route that makes no sense on the server.

In our browser run, adding a lamp on a prerendered product page updated the header to
"Cart (1)", and the client-rendered `/cart` page showed it — also after a full reload.

## How It Actually Works

**Rendering.** For each request (or each prerendered path at build time) the engine
creates a fresh application instance with a server platform whose DOM is an in-memory
implementation (Angular's platform-server bundles the `domino` DOM library for this). It navigates the router to the URL and waits until the app
is **stable** — no pending tasks. Pending HTTP requests, resources and deferred loads all
register pending tasks (Level 2, lesson 06), so data loaded through Angular's APIs is in
the HTML; a raw `setTimeout` or a library that doesn't register tasks isn't waited for.
It then serialises the DOM, adds the `ngh` annotations and `ng-state`, applies response
status and headers, and destroys the instance. Nothing is shared between requests except
what you deliberately put in module-level variables (don't).

**Build.** `ng build` produces two bundles — `browser/` and `server/` (in our run
`server.mjs`, `main.server.mjs` and more) — and then runs the server bundle to prerender
the `Prerender` routes, calling `getPrerenderParams` to expand parameterised paths. The
app engine's manifest records which mode each route uses, so at runtime `server.ts`
serves prerendered files straight from disk (via `express.static`) and only renders
`Server`-mode routes; `Client` routes get the plain shell.

**Hydration** uses the annotations to pair each client-side view with existing DOM
nodes. If the DOM doesn't match what the client expects — usually because code rendered
differently on server and client (random values, `Date.now()`, browser-only branches in
templates) or because HTML was altered by a CDN or browser extension — Angular raises a
hydration mismatch error (`NG05xx`) that names the component and the expected versus
actual node. Fix the difference; as a last resort, add the `ngSkipHydration` attribute to
a component's host element so that component is re-rendered on the client instead of
hydrated.

## Common mistakes

- **Deploying without configuring allowed hosts** — every request returns 400.
- **Touching `window`/`localStorage` in constructors or field initialisers.** Use
  `afterNextRender`, `isPlatformBrowser`, or `RenderMode.Client`.
- **Rendering differently on server and client** (random IDs, current time), causing
  hydration mismatches.
- **Assuming SSR makes a page personalised-safe to cache.** A `Server`-mode page that
  shows user data must not be cached by a CDN for everyone.
- **Using `Prerender` for data that changes often.** Prerendered pages are only as fresh
  as your last build.

## Exercise

1. Create a project with `--ssr`. Add a `/blog/:slug` route prerendered from a list of
   slugs, and a `/now` route in `Server` mode that shows the server time.
2. Run `ng build` and find the prerendered files. Start the server — note the host error
   — and fix it with `NG_ALLOWED_HOSTS`.
3. Make an unknown blog slug return 404 using `RESPONSE_INIT`; check with `curl -I`.
4. Load data in `/now` with `httpResource` from an Express endpoint in `server.ts` and
   confirm in the Network panel that the first load makes no browser request.
5. Put `localStorage.getItem(...)` in a component constructor, run the build, and read
   the error. Fix it two different ways.
