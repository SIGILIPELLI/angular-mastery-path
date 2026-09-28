---
description: "Load async data with Angular's resource() and httpResource(): params, status signals, idle requests, cancellation, errors, reload, local edits and testing."
---

# 06 · Async Data with resource & httpResource

Loading data in a component used to mean the same boilerplate every time: a `loading`
signal, an `error` signal, a `data` signal, a subscription, cancelling stale requests
when inputs change, and remembering to reset everything in each branch. The **resource**
APIs package that pattern. You describe *what* to load as a function of signals, and get
back an object with `value`, `status`, `error` and `isLoading` signals that stay correct
automatically.

Three flavours, all stable (`@publicApi 22.0`) in Angular 22:

| API | Loader returns | Package |
|-----|----------------|---------|
| `resource({ params, loader })` | a `Promise` | `@angular/core` |
| `rxResource({ params, stream })` | an `Observable` | `@angular/core/rxjs-interop` |
| `httpResource(() => url or request)` | — it makes an `HttpClient` GET for you | `@angular/common/http` |

## `httpResource`: the common case

```ts title="src/app/user-card.ts"
import { Component, input } from '@angular/core';
import { httpResource } from '@angular/common/http';

interface User { id: number; name: string; email: string }

@Component({
  selector: 'app-user-card',
  template: `
    @if (user.error(); as err) {
      <p role="alert">Could not load user {{ id() }}.</p>
      <button type="button" (click)="user.reload()">Try again</button>
    } @else if (user.hasValue()) {
      <h2>{{ user.value().name }}</h2>
      <p>{{ user.value().email }}</p>
    }
    @if (user.isLoading()) {
      <p aria-live="polite">Loading…</p>
    }
  `,
})
export class UserCard {
  readonly id = input.required<number>();
  protected readonly user = httpResource<User>(() => `/api/users/${this.id()}`);
}
```

That's the whole data layer for this component. When `id` changes, the resource
computes the new URL and fetches it; if a request is still in flight for the previous
id, it's cancelled. We rendered it under the HTTP testing backend and recorded the text
at each step:

```text
T1  Loading…                                     (request for /api/users/1 outstanding)
T2  Ada ada@example.com                          (flushed 200)
T3  Could not load user 2. Try again             (id → 2, flushed 500)
T4  Could not load user 2. Try again Loading…    (clicked Try again → reload())
T5  Grace grace@example.com                      (flushed 200)
```

The request function can return a **URL string** or a **request object** —
`{ url, method, params, headers, body, ... }` — for anything beyond a simple GET. Returning
`undefined` means "nothing to load right now" and leaves the resource **idle**:

```ts
protected readonly query = signal('');
protected readonly results = httpResource<User[]>(
  () => (this.query() ? { url: '/api/users', params: { q: this.query() } } : undefined),
  { defaultValue: [] },
);
```

With an empty query this stayed `idle` with value `[]`; setting `query` to `gr` sent
`GET /api/users?q=gr` and resolved to the flushed array.

`httpResource` also exposes the response's `headers()` and `statusCode()` as signals, and
has variants for non-JSON bodies: `httpResource.text()`, `httpResource.blob()`,
`httpResource.arrayBuffer()`. Pass `parse: (raw) => ...` to validate or transform the
JSON (a natural place for a schema library).

!!! note "Reads, not writes"
    `httpResource` is designed for **fetching** data that depends on signals. For POST,
    PUT and DELETE triggered by a user action, call `HttpClient` directly (or use
    `submit()` in signal forms). A resource re-runs whenever its params change, which
    is exactly wrong for "create an order".

## `resource`: any Promise-based loader

```ts
const r = resource({
  params: () => id(),                           // undefined → idle
  loader: async ({ params, abortSignal }) => {
    const res = await fetch(`/api/users/${params}`, { signal: abortSignal });
    if (!res.ok) throw new Error(`User ${params} not found`);
    return (await res.json()) as User;
  },
});
```

The loader receives `params` (already non-undefined — the type excludes `undefined`),
an `abortSignal` that fires when the result is no longer wanted, and `previous` (the
status before this load). Pass the signal to `fetch` or any API that accepts one.

We ran a resource whose loader took 20 ms, and logged its state along the way:

```text
A  idle      value: undefined   hasValue: false    params() returned undefined
B  loading                                          id set to 1
C  resolved  {"id":2,"name":"User 2"}   aborted: [1]
D  error     "User 13 not found"        hasValue: false
E  local     {"id":2,"name":"Renamed locally"}
```

Line C is the interesting one: we set the id to 1 and then to 2 before the first load
finished. The first load's `abortSignal` fired (`aborted: [1]`) and only user 2's result
was ever shown — no race condition to handle yourself.

## Status and value, precisely

| `status()` | Meaning | `value()` |
|------------|---------|-----------|
| `idle` | params returned `undefined` | `defaultValue` (or `undefined`) |
| `loading` | loading after params changed | `defaultValue` (or `undefined`) — the old value is cleared |
| `reloading` | loading after `reload()` | the previous value is kept |
| `resolved` | loaded successfully | the loaded value |
| `error` | loader threw / request failed | **reading it throws** |
| `local` | you called `value.set(...)` / `update(...)` | your value |

That `error` row is easy to trip over. Reading `value()` in the error state threw:

```text
Resource is currently in an error state (see Error.cause for details): User 13 not found
```

So in templates, check `error()` first or guard with `hasValue()` — as `UserCard` does —
rather than reading `value()` unconditionally. `hasValue()` is also a type guard: inside
`@else if (user.hasValue())`, `user.value()` is typed without `undefined`.

**Local edits.** A resource's `value` is writable. Setting it switches the status to
`local`; the next params change or `reload()` replaces it with server data again. It's a
convenient place for optimistic updates.

**Defaults.** `defaultValue: []` makes the value type `User[]` instead of
`User[] | undefined`. Note it's used while *idle or loading*, so an empty list and "not
loaded yet" look the same — show `isLoading()` if the difference matters.

## Chaining resources

When one request needs the result of another, the `params` function receives a context
with `chain`:

```ts
protected readonly org = httpResource<Org>(() => `/api/orgs/${this.orgId()}`);

protected readonly members = resource({
  params: ({ chain }) => chain(this.org)?.membersUrl,
  loader: ({ params }) => fetch(params).then((r) => r.json() as Promise<Member[]>),
});
```

Per the API docs, `chain(other)` returns the other resource's value only when it is
`resolved` or `local`, and otherwise propagates its status — so `members` shows
`loading` while `org` loads, and `error` if `org` fails, without extra code.

The `?.` is needed: an `httpResource` without a `defaultValue` is typed `Org | undefined`,
and without it the build failed with `TS2532: Object is possibly 'undefined'`. Returning
`undefined` from `params` simply leaves `members` idle.

## Testing components that use resources

Use `provideHttpClient()` + `provideHttpClientTesting()` and flush requests from the test.
One trap we hit: `await fixture.whenStable()` **does not resolve while a resource
request is outstanding**. In a test that awaited it before flushing, it was still pending
after 500 ms; after flushing, it resolved immediately. Resources register a *pending task*
with Angular while loading, and "stable" means "no pending tasks". So flush first, then
await stability — or advance with `TestBed.tick()`.

## How It Actually Works

A resource is a small state machine built out of signals:

1. `params` is wrapped in a `computed`. An internal effect watches it.
2. When the computed produces a new value, the resource aborts the previous load's
   `AbortController`, sets status to `loading` (or `idle` for `undefined`), and calls the
   loader with the new params and a fresh `abortSignal`.
3. When the promise settles, the resource checks that the result still belongs to the
   **current** request (a stale result is dropped), then sets `value`/`error` and
   `status`.
4. While a load is pending, it holds a `PendingTasks` entry, which keeps the application
   "unstable" — this is what server-side rendering waits for before serialising HTML
   (Level 3, lesson 06), and what `whenStable()` waits for in tests.

`httpResource` builds the `params` computed from your request function and uses a loader
that makes the request through `HttpClient` — so interceptors (lesson 08), the test
backend and the default `fetch` backend all apply. `rxResource` uses a streaming loader
that subscribes to your observable and unsubscribes on the next change.

Because `status`, `value` and `error` are signals, templates and computeds that read
them participate in normal change detection. There's nothing to subscribe to and nothing
to unsubscribe from.

## Common mistakes

- **Reading `value()` in the error state.** It throws. Guard with `error()`/`hasValue()`.
- **Using `httpResource` for mutations.** It re-fetches when params change; that's not
  what a POST should do.
- **Forgetting that `loading` clears the value.** Lists flash empty on every parameter
  change unless you keep the old value (Level 3 shows `linkedSignal` for this) or show
  a skeleton.
- **Ignoring `abortSignal` in `resource` loaders.** The resource ignores stale results
  anyway, but without passing the signal to `fetch`, the network request still runs.
- **Awaiting `whenStable()` before flushing** test requests.

## Exercise

1. Build a `RepoList` component with `username = input.required<string>()` and an
   `httpResource` for `https://api.github.com/users/{username}/repos` (a real public
   endpoint — note GitHub rate-limits unauthenticated requests, so expect occasional 403s
   and handle them).
2. Show loading, error-with-retry and list states. Make sure nothing reads `value()` in
   the error state.
3. Add a `sort = signal<'stars' | 'name'>('stars')` and derive the sorted list with
   `computed`. Explain why sorting belongs in a `computed`, not in the request.
4. Write a test with `provideHttpClientTesting()` that flushes a 404 and asserts the
   error message renders.
