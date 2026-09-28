---
description: "Angular HTTP interceptors with HttpInterceptorFn: auth headers, HttpContext opt-outs, loading counters, safe retries with backoff, error mapping and ordering."
---

# 08 · HTTP Interceptors & Error Handling

Some logic applies to *every* request: attach the auth token, count requests in flight
for a global progress bar, retry transient failures, turn raw HTTP errors into
something the UI can show. Writing that in every service is repetitive and easy to get
wrong. **Interceptors** are functions that sit between `HttpClient` and the network and
see every request and response.

## Registering interceptors

```ts title="src/app/app.config.ts (excerpt)"
provideHttpClient(
  withInterceptors([busyInterceptor, authInterceptor, errorInterceptor, retryInterceptor]),
),
```

An interceptor is a plain function of type `HttpInterceptorFn`:

```ts
export const noopInterceptor: HttpInterceptorFn = (req, next) => next(req);
```

It receives the request and a `next` function that passes the request down the chain,
and it returns the observable of events coming back. It runs in an injection context, so
it can `inject()` services. `ng generate interceptor auth` scaffolds one.

## 1. Attaching an auth token — with an opt-out

```ts title="src/app/http/auth.interceptor.ts"
import { HttpContextToken, HttpInterceptorFn } from '@angular/common/http';
import { inject } from '@angular/core';

export const SKIP_AUTH = new HttpContextToken<boolean>(() => false);

export const authInterceptor: HttpInterceptorFn = (req, next) => {
  const token = inject(Session).token();
  if (!token || req.context.get(SKIP_AUTH)) return next(req);
  return next(req.clone({ setHeaders: { Authorization: `Bearer ${token}` } }));
};
```

Requests are **immutable** — `req.clone({...})` makes a modified copy. The header
arrived as `Bearer abc123` in our test.

`HttpContext` carries per-request metadata that interceptors can read but that is never
sent to the server. A call that must not carry the token opts out like this:

```ts
this.http.get('/api/public', { context: new HttpContext().set(SKIP_AUTH, true) });
```

— and in our test that request had no `Authorization` header.

Only attach tokens to **your own API origin**. An interceptor that adds a bearer token to
every request will happily send it to a third-party URL too. Check
`req.url.startsWith(environment.apiUrl)` (or a relative URL) before adding it.

## 2. A global "busy" counter

```ts title="src/app/http/busy.interceptor.ts"
@Service()
export class Busy {
  readonly count = signal(0);
}

export const busyInterceptor: HttpInterceptorFn = (req, next) => {
  const busy = inject(Busy);
  busy.count.update((n) => n + 1);
  return next(req).pipe(finalize(() => busy.count.update((n) => n - 1)));
};
```

A header component shows a progress bar when `busy.count() > 0`. We saw `busy 1` while
a request was outstanding and `0` after it finished.

Use `finalize`, not `tap({ complete })`, for cleanup. `finalize` runs on completion,
error **and unsubscription**. In our test the caller used `firstValueFrom`, which
unsubscribes as soon as the first value arrives — a `tap({ complete })` in the auth
interceptor never logged, while `finalize` in the busy interceptor did. With
`tap`, the counter would have drifted upward forever.

## 3. Retrying safely

```ts title="src/app/http/retry.interceptor.ts"
export const retryInterceptor: HttpInterceptorFn = (req, next) =>
  req.method !== 'GET'
    ? next(req)
    : next(req).pipe(
        retry({
          count: 2,
          delay: (err: HttpErrorResponse, attempt) =>
            err.status >= 500 || err.status === 0
              ? timer(250 * 2 ** attempt)          // exponential backoff
              : throwError(() => err),             // 4xx: don't retry
        }),
      );
```

Three rules are baked in:

- **Only retry idempotent requests.** Retrying a POST can create two orders. Our test
  flushed a 503 for a POST and it failed immediately with no second request.
- **Only retry errors that might go away**: 5xx and network errors (status 0). A 404
  won't fix itself; in our test the 404 was not retried (zero follow-up requests).
- **Back off.** Immediate retries hammer a struggling server. Here, a GET that failed with
  503 twice succeeded on the third attempt and the caller only ever saw `{"ok":true}`.

## 4. Mapping errors to something the UI understands

```ts title="src/app/http/error.interceptor.ts"
export class ApiError extends Error {
  constructor(readonly status: number, message: string) {
    super(message);
  }
}

export const errorInterceptor: HttpInterceptorFn = (req, next) =>
  next(req).pipe(
    catchError((err: HttpErrorResponse) =>
      throwError(() =>
        new ApiError(
          err.status,
          err.status === 0 ? 'Network error' : err.error?.message ?? err.statusText,
        ),
      ),
    ),
  );
```

A 404 with body `{ "message": "No such book" }` reached the caller as
`ApiError 404 No such book`. Components can now show `error.message` without knowing the
server's error format. Don't swallow errors here — rethrow, so callers (and
`httpResource`'s `error()` signal) still see them.

A 401 is the classic special case: redirect to login, or refresh the token and replay
the request. Refreshing correctly under concurrency (several requests failing at once
should trigger *one* refresh) needs a shared in-flight refresh observable; keep that
logic in the auth service and let the interceptor call it.

## Ordering

Interceptors run **in array order on the way out, and reverse order on the way back**.
Our log for one request with `[busy, auth, error, retry]` began
`busy -> auth -> ...` and ended with `busy <-`. Consequences for the order above:

- `busy` is outermost, so it counts one "busy" period per call, including all retries.
- `error` sits *outside* `retry`, so it maps the error only after retries are exhausted.
  Swap them and `retry` would see `ApiError` objects instead of `HttpErrorResponse`, and
  its `err.status >= 500` check would break silently.
- `auth` is outside `retry`, so all attempts carry the same token.

## How It Actually Works

`provideHttpClient(withInterceptors(fns))` builds a **handler chain** once per injector.
Each interceptor is wrapped as a handler whose `handle(req)` calls
`interceptorFn(req, nextHandler.handle)` inside `runInInjectionContext(environmentInjector, ...)`
— hence `inject()` works. The innermost handler is the backend (`FetchBackend` by
default, or the testing backend).

Because every piece returns a **cold** observable, nothing is sent until the caller
subscribes; then subscription flows inward (outermost interceptor first) until the
backend issues the request, and events flow outward through each interceptor's operators.
`retry` works by *resubscribing* to `next(req)` — which re-runs everything inside it
(including the backend), but not the interceptors outside it. That's why its position in
the array defines which interceptors see each attempt.

Class-based interceptors (`HttpInterceptor` with `intercept(req, next)`, registered via
the `HTTP_INTERCEPTORS` multi-provider and `withInterceptorsFromDi()`) are the older
style. They behave the same way; you'll meet them in existing code.

## Common mistakes

- **Mutating the request** — it's immutable. Use `clone`.
- **Retrying POST/PATCH** — duplicate writes.
- **Sending tokens to every URL**, including third-party ones.
- **Catching and returning `EMPTY`** in an error interceptor. Callers then complete
  without a value or an error, and loading spinners never stop.
- **Cleanup in `tap({ complete })`** that never runs when callers unsubscribe early.

## Exercise

1. Write a `timingInterceptor` that logs `METHOD url — 123 ms` for every request using
   `performance.now()` and `finalize`.
2. Write a `CACHE` `HttpContextToken<boolean>` and a caching interceptor that stores GET
   responses in a `Map` for 30 seconds when the token is set.
3. Add a 401 handler that navigates to `/login?returnUrl=...` using `inject(Router)`.
4. Put all interceptors in an order you can justify in a comment, then write tests with
   `provideHttpClientTesting()` proving: a cached request is served without hitting the
   backend, and a 401 triggers navigation.
