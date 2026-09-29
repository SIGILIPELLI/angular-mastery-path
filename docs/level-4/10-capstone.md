---
description: "Level 4 capstone: the SSR catalog taken to production with error reporting, @boundary, chunk-reload safety, release tags and a passing audit."
---

# 10 · Capstone — A Production Angular App

The capstone takes the **Level 3 catalog** (a server-rendered shop with prerendered
product pages, per-request search, a client-only cart and incremental hydration) and
applies Level 4 to it: build configuration, error handling, deployment concerns and a
production-readiness review. The goal isn't more features — it's an app you could run
for real users and sleep at night.

Start from the finished Level 3 project ([lesson 10](../level-3/10-project-catalog.md)).
Everything below was built, tested, served and exercised on Angular 22.2; results are at
the end.

## Requirements

1. **Release tagging** — every build carries a version; every error report includes it.
2. **Error reporting** — one `ErrorHandler` for all errors, deduplicated, with route and
   release.
3. **Containment** — a failure in a secondary section doesn't break the product page.
4. **Deploy safety** — users on an old version recover from missing chunks without a
   reload loop.
5. **Budgets and source maps** — a budget close to the real size; maps generated but not
   public.
6. **Security headers** — sent by the server for every response.
7. **Readiness** — the lesson 09 audit passes, and tests run in CI.

## 1. Release tagging and build configuration

Three settings changed in `angular.json`, shown here keyed by where each lives inside
`projects.catalog.architect.build` (this is a summary, not a file to paste):

```json
{
  "options.define": {
    "BUILD_VERSION": "'dev'"
  },
  "configurations.production.budgets[0]": {
    "type": "initial",
    "maximumWarning": "340kB",
    "maximumError": "400kB"
  },
  "configurations.production.sourceMap": {
    "scripts": true,
    "hidden": true
  }
}
```

`BUILD_VERSION` defaults to `'dev'` and CI overrides it:
`ng build --define "BUILD_VERSION='1.0.0'"`. Defining a default in `angular.json` avoids
the `ReferenceError` we hit in lesson 01 when a constant was only defined on the command
line. The budget was tightened from the generated 500 kB/1 MB to 340 kB/400 kB, just
above the app's actual initial size.

## 2. Error reporting

```ts title="src/app/core/error-reporter.ts"
import { ErrorDetails, ErrorHandler, Service, inject } from '@angular/core';
import { Router } from '@angular/router';

declare const BUILD_VERSION: string;

export interface ErrorReport {
  message: string;
  stack?: string;
  url: string;
  release: string;
  count: number;
}

/**
 * Collects errors, deduplicates them and (in a real deployment) ships them to a
 * monitoring endpoint. Here the transport is console.error so the behaviour is visible.
 */
@Service({ autoProvided: false })
export class ErrorReporter implements ErrorHandler {
  private readonly router = inject(Router);
  private readonly seen = new Map<string, ErrorReport>();

  handleError(error: unknown): void {
    this.record(error);
  }

  onViewError(error: Error, details: ErrorDetails): void {
    this.record(error, details.declarationType.name);
  }

  reports(): ErrorReport[] {
    return [...this.seen.values()];
  }

  private record(error: unknown, where?: string) {
    const err = error instanceof Error ? error : new Error(String(error));
    const key = `${err.message}|${err.stack?.split('\n')[1] ?? ''}`;
    const existing = this.seen.get(key);
    if (existing) {
      existing.count++;
      return; // already reported: don't flood the monitoring service
    }
    const report: ErrorReport = {
      message: where ? `[${where}] ${err.message}` : err.message,
      stack: err.stack,
      url: this.router.url,
      release: BUILD_VERSION,
      count: 1,
    };
    this.seen.set(key, report);
    console.error('[error report]', JSON.stringify({ message: report.message, url: report.url, release: report.release }));
  }
}
```

It deduplicates by message and first stack frame (lesson 08 showed rendering errors
repeat on every refresh), records the route and release, and implements `onViewError` so
errors caught by `@boundary` blocks arrive with the component type. In production,
replace `console.error` with a batched `fetch`/`navigator.sendBeacon` to your monitoring
endpoint.

## 3. Deploy safety

```ts title="src/app/core/chunk-reload.ts"
import { NavigationError } from '@angular/router';

const KEY = 'reloaded-for-release';
declare const BUILD_VERSION: string;

/** After a deploy, old chunk names 404. Reload once per release to pick up the new ones. */
export function reloadOnMissingChunk(e: NavigationError): void {
  const missingChunk = String(e.error).includes('Failed to fetch dynamically imported module');
  if (!missingChunk || typeof window === 'undefined') return;
  if (sessionStorage.getItem(KEY) === BUILD_VERSION) return; // already tried: avoid a loop
  sessionStorage.setItem(KEY, BUILD_VERSION);
  window.location.assign(e.url);
}
```

## 4. Wiring

```ts title="src/app/app.config.ts"
import { ApplicationConfig, ErrorHandler, provideBrowserGlobalErrorListeners } from '@angular/core';
import { provideHttpClient } from '@angular/common/http';
import { provideClientHydration, withEventReplay } from '@angular/platform-browser';
import { provideRouter, withComponentInputBinding, withNavigationErrorHandler } from '@angular/router';
import { reloadOnMissingChunk } from './core/chunk-reload';
import { ErrorReporter } from './core/error-reporter';
import { routes } from './app.routes';

export const appConfig: ApplicationConfig = {
  providers: [
    provideBrowserGlobalErrorListeners(),
    provideRouter(routes, withComponentInputBinding(), withNavigationErrorHandler(reloadOnMissingChunk)),
    ErrorReporter,
    { provide: ErrorHandler, useExisting: ErrorReporter },
    provideHttpClient(),
    provideClientHydration(withEventReplay()),
  ],
};
```

## 5. Containment with `@boundary`

The related-products section is now wrapped in a boundary inside its deferred block:

```ts title="src/app/catalog/product-page.ts (template)"
  template: `
    @if (product(); as p) {
      <h2>{{ p.name }}</h2>
      <p>{{ p.summary }}</p>
      <p class="price">{{ p.price | currency: 'USD' }}</p>
      <button type="button" (click)="cart.add(p.slug)">Add to cart</button>

      @defer (hydrate on viewport) {
        @boundary {
          <app-related-products [slug]="p.slug" />
        } @error {
          <p class="muted">Related products are unavailable right now.</p>
        }
      }
    } @else {
      <h2>Product not found</h2>
      <p><a routerLink="/">Back to all products</a></p>
    }
  `,
```

If related products fail to render, the product, price and "Add to cart" button still
work, and the error is reported through `onViewError`.

## 6. Security headers

Our first attempt enabled `"security": { "autoCsp": true }`, and the build refused:

```text
✖ Building... [FAILED: Cannot set both SSR and auto-CSP at the same time.]
```

Automatic hash-based CSP only works for client-rendered builds. A strict `script-src`
policy for an SSR app needs a per-request nonce, which is a bigger change; for the
capstone we set the headers that don't depend on scripts, in `server.ts`:

```ts title="src/server.ts (excerpt)"
/**
 * Security headers for every response. A strict script-src CSP for an SSR app needs a
 * per-request nonce (Angular's autoCsp option can't be combined with SSR), so this policy
 * restricts framing, plugins and <base> but not scripts.
 */
app.use((_req, res, next) => {
  res.setHeader('Content-Security-Policy', "frame-ancestors 'none'; object-src 'none'; base-uri 'self'");
  res.setHeader('X-Content-Type-Options', 'nosniff');
  res.setHeader('Referrer-Policy', 'strict-origin-when-cross-origin');
  next();
});
const angularApp = new AngularNodeAppEngine();
```

## 7. Tests

The Level 3 tests (product page and cart) still pass, plus one for the reporter:

```ts title="src/app/core/error-reporter.spec.ts"
import { TestBed } from '@angular/core/testing';
import { provideRouter } from '@angular/router';
import { ErrorReporter } from './error-reporter';

describe('ErrorReporter', () => {
  it('reports each distinct error once and counts repeats', () => {
    TestBed.configureTestingModule({ providers: [provideRouter([]), ErrorReporter] });
    const reporter = TestBed.inject(ErrorReporter);
    const log = vi.spyOn(console, 'error').mockImplementation(() => {});

    const boom = new Error('boom');
    reporter.handleError(boom);
    reporter.handleError(boom);
    reporter.handleError(boom);
    reporter.handleError(new Error('other'));

    expect(log).toHaveBeenCalledTimes(2);
    expect(reporter.reports().map((r) => [r.message, r.count])).toEqual([['boom', 3], ['other', 1]]);
    expect(reporter.reports()[0].release).toBe('dev'); // BUILD_VERSION default from angular.json
  });
});
```

## Verification

**Unit tests:** `ng test --watch=false` → `Tests  5 passed (5)`.

**Production build** (`ng build --define "BUILD_VERSION='1.0.0'"`):

```text
                     | Initial total    | 309.60 kB |                86.07 kB
Prerendered 7 static routes.
```

**Readiness audit** (lesson 09's script):

```text
✔ production is the default build configuration
✔ initial bundle budget has an error limit — 340kB / 400kB
✔ optimization enabled in production
✔ output hashing = all
✔ a Content Security Policy is configured — header set in src/server.ts
✔ all JS file names are content-hashed — 8 files
✔ bundles do not link source maps publicly
✔ index.html declares a language
✔ index.html has a title
✔ index.html has a viewport meta tag
✔ total JavaScript on disk — 309.9 kB across 8 files
✔ no bypassSecurityTrust* calls
✔ no console.log left in src/app
✔ custom ErrorHandler registered
✔ unit tests exist — 3 spec files

15/15 checks passed
```

**Server** (`NG_ALLOWED_HOSTS=localhost node dist/catalog/server/server.mjs`), status and
headers per request:

```text
/                            200  CSP: frame-ancestors 'none'; object-src 'none'; base-uri 'self'  nosniff
/products/oak-standing-desk  200  (same headers)
/products/nope               404  (same headers)
/search?q=desk               200  (same headers)
/main-GIXUEJH3.js            200  (same headers)  Cache-Control: public, max-age=31536000
```

**Deploy-safety drill.** In Chromium we made the server return 404 for the search page's
chunk — what happens to a user holding an old `index.html` after a deploy — and clicked
**Search**:

```text
full page navigations: ['/', '/search', '/']
sessionStorage 'reloaded-for-release': 1.0.0
error reports: 2 × "Failed to fetch dynamically imported module: …/chunk-VVlfEql5.js" (release 1.0.0)
```

The app reloaded **once** to `/search`. Because our simulated server still returned 404
for the chunk, the second attempt failed too — and the release flag stopped a reload loop,
leaving the user on the home page with the error reported. On a real deploy, the reload
fetches the new `index.html` whose chunk names exist, and the navigation succeeds.

## What's left (and would be next)

Honest gaps a production review would still list for this app:

- **Prerendered HTML has no `Cache-Control` header** (the audit above shows `None` for
  `/`); add `no-cache` for HTML in `server.ts` so new releases are picked up promptly.
- **No strict `script-src` CSP** — needs nonce support end to end.
- **Errors go to the console**, not a monitoring service; source maps aren't uploaded
  anywhere.
- **No end-to-end tests** in CI (Playwright would cover the flows we drove by hand).
- **Accessibility** hasn't had a screen-reader pass.
- **The Docker image** from lesson 07 wasn't built on the machine used for this course.

Writing these down is part of the job: a readiness review that ends with "no known
issues" usually means nobody looked.

## How It Actually Works

Follow a request for `/products/oak-standing-desk` in the finished system:

1. **Express** runs the security-header middleware, then `express.static` finds the
   prerendered `products/oak-standing-desk/index.html` and returns it — no Angular
   rendering on the server for this request.
2. **The browser** paints it, downloads the hashed `main-*.js` (cached for a year after
   the first visit) and the product-page chunk, and hydrates. The related-products block
   stays dehydrated until scrolled into view (Level 3).
3. **If anything throws** — in an event handler, a promise, a timer, change detection or
   the related-products boundary — it lands in `ErrorReporter`, deduplicated and tagged
   with `/products/oak-standing-desk` and release `1.0.0`.
4. **After a deploy**, a user still on 1.0.0 who navigates to a lazy route whose chunk
   was removed triggers `reloadOnMissingChunk`, which reloads once per release.
5. **In CI**, tests, the production build (with budgets) and the readiness audit gate
   every change.

## Exercise

Complete at least three of the "What's left" items:

1. Add `Cache-Control: no-cache` for HTML responses and confirm with `curl -I`.
2. Replace `console.error` in `ErrorReporter` with a batched POST to a small Express
   endpoint in `server.ts` that writes reports to a log file.
3. Write Playwright tests for: browsing to a product, adding to cart, searching, and the
   404 page.
4. Do a keyboard-only and screen-reader pass of the whole app and fix what you find.
5. Build and run the Docker image from lesson 07 for this app, with `NG_ALLOWED_HOSTS`
   set, and repeat the verification steps against the container.
6. Add a GitHub Actions workflow that runs tests, the production build with
   `BUILD_VERSION` set to the commit SHA, and the readiness audit.
