---
description: "Call APIs from Angular with HttpClient: provideHttpClient, typed GET and POST, HttpParams, mapping responses, HttpErrorResponse and testing without a server."
---

# 09 · HTTP with HttpClient

Almost every real app talks to a server. Angular's `HttpClient` wraps the browser's
networking APIs with typed methods, JSON handling, a pipeline for cross-cutting concerns
(interceptors, Level 2 lesson 08), and a testing backend that lets you test data code
without a server.

## Enabling it

Add `provideHttpClient()` to the application providers:

```ts title="src/app/app.config.ts"
import { ApplicationConfig, provideBrowserGlobalErrorListeners } from '@angular/core';
import { provideHttpClient } from '@angular/common/http';
import { provideRouter, withComponentInputBinding } from '@angular/router';
import { routes } from './app.routes';

export const appConfig: ApplicationConfig = {
  providers: [
    provideBrowserGlobalErrorListeners(),
    provideRouter(routes, withComponentInputBinding()),
    provideHttpClient(),
  ],
};
```

In Angular 22, `HttpClient` uses the **`fetch` API** by default. You will see
`provideHttpClient(withFetch())` in many tutorials and projects from Angular 17–21; in 22
`withFetch()` is deprecated because it no longer does anything — the type definitions say
"`withFetch` is not required anymore. `FetchBackend` is the default `HttpBackend`." If you
need upload progress events, which `fetch` cannot report, opt back into
`XMLHttpRequest` with `provideHttpClient(withXhr())`.

## Wrap endpoints in a service

Components should not build URLs. Put each API behind a service that returns data in the
shape your app wants. Here is a client for the public Open Library search API, used in the
Level 1 project:

```ts title="src/app/reading/open-library.ts"
import { HttpClient, HttpParams } from '@angular/common/http';
import { Service, inject } from '@angular/core';
import { Observable, map } from 'rxjs';

export interface SearchHit {
  key: string;
  title: string;
  author: string;
  year?: number;
}

// The subset of the API response we use.
interface SearchResponse {
  numFound: number;
  docs: { key: string; title: string; author_name?: string[]; first_publish_year?: number }[];
}

@Service()
export class OpenLibrary {
  private readonly http = inject(HttpClient);

  search(query: string): Observable<SearchHit[]> {
    const params = new HttpParams()
      .set('q', query)
      .set('fields', 'key,title,author_name,first_publish_year')
      .set('limit', 10);
    return this.http
      .get<SearchResponse>('https://openlibrary.org/search.json', { params })
      .pipe(
        map((res) =>
          res.docs.map((d) => ({
            key: d.key,
            title: d.title,
            author: d.author_name?.[0] ?? 'Unknown',
            year: d.first_publish_year,
          })),
        ),
      );
  }
}
```

Points to notice:

- **`get<SearchResponse>(...)`** tells TypeScript what the JSON will look like. It is a
  *promise to the compiler*, not a runtime check — if the server sends something else,
  your code will find out at runtime. Validate untrusted responses (for example with a
  schema library) when correctness matters.
- **`HttpParams`** encodes query parameters for you. It is immutable: each `.set()`
  returns a new instance, so chain the calls or reassign. A plain object also works:
  `{ params: { q: query, limit: 10 } }`.
- **Map at the edge.** The API's `author_name` array becomes a single `author` string
  here, so the rest of the app never sees the API's naming.

When we ran this service under a test backend, the request it produced was:

```text
GET https://openlibrary.org/search.json?q=dune&fields=key,title,author_name,first_publish_year&limit=10
```

and a response containing one full document and one with no author or year mapped to:

```json
[{"key":"/works/OL893414W","title":"Dune","author":"Frank Herbert","year":1965},
 {"key":"/works/x","title":"Anon","author":"Unknown"}]
```

## Using it from a component

`HttpClient` methods return RxJS **Observables**. For now you only need one fact about
them: nothing happens until something calls `subscribe`. The simplest pattern is to
subscribe and copy the result into signals:

```ts title="src/app/reading/book-search.ts (excerpt)"
protected readonly results = signal<SearchHit[]>([]);
protected readonly loading = signal(false);
protected readonly error = signal('');

protected search(query: string) {
  if (!query.trim()) return;
  this.loading.set(true);
  this.error.set('');
  this.api.search(query).subscribe({
    next: (hits) => {
      this.results.set(hits);
      this.loading.set(false);
    },
    error: () => {
      this.error.set('Search failed. Check your connection and try again.');
      this.loading.set(false);
    },
  });
}
```

HTTP observables complete after one response, so this subscription cleans itself up.
Level 2 shows better tools — `toSignal`, `rxResource` and `httpResource` (lessons 05
and 06) — that handle loading state, errors and cancellation for you.

## Writing data

```ts
createBook(input: { title: string; author: string }) {
  return this.http.post<Book>('/api/books', input);
}
updateBook(id: string, changes: Partial<Book>) {
  return this.http.patch<Book>(`/api/books/${id}`, changes);
}
deleteBook(id: string) {
  return this.http.delete<void>(`/api/books/${id}`);
}
```

Objects are serialised as JSON. In our test, the POST body arrived as `{"title":"X"}`
and Angular detected the content type as `application/json` (it adds the header when the
request is sent; you don't need to set it yourself).

## Handling errors

Any response with a non-2xx status, and any network failure, arrives on the observable's
error channel as an **`HttpErrorResponse`**. We answered a POST with `409 Conflict` and a
JSON body; the error object had:

```text
instanceof HttpErrorResponse: true
status:     409
statusText: Conflict
error:      {"message":"title taken"}          ← the parsed response body
message:    Http failure response for /api/books: 409 Conflict
```

So `err.error` is where the server's own error message lives. A status of `0` means the
request never got a response (offline, DNS failure, or blocked by CORS). Handle expected
errors (validation, conflicts, 404) where you call the service and show a useful message;
leave unexpected ones to a global handler (Level 4, lesson 08).

## Testing without a server

`provideHttpClientTesting()` replaces the real backend with a controller you drive from
the test:

```ts title="src/app/reading/open-library.spec.ts"
import { TestBed } from '@angular/core/testing';
import { provideHttpClient } from '@angular/common/http';
import { HttpTestingController, provideHttpClientTesting } from '@angular/common/http/testing';
import { firstValueFrom } from 'rxjs';
import { OpenLibrary } from './open-library';

describe('OpenLibrary', () => {
  beforeEach(() =>
    TestBed.configureTestingModule({ providers: [provideHttpClient(), provideHttpClientTesting()] }),
  );

  it('maps search results', async () => {
    const api = TestBed.inject(OpenLibrary);
    const http = TestBed.inject(HttpTestingController);

    const result = firstValueFrom(api.search('dune'));
    const req = http.expectOne((r) => r.url === 'https://openlibrary.org/search.json');
    expect(req.request.params.get('q')).toBe('dune');
    req.flush({ numFound: 1, docs: [{ key: 'k', title: 'Dune', author_name: ['Frank Herbert'] }] });

    expect(await result).toEqual([{ key: 'k', title: 'Dune', author: 'Frank Herbert', year: undefined }]);
    http.verify(); // fails if any unexpected request was made
  });
});
```

Order matters: `provideHttpClient()` first, then `provideHttpClientTesting()`.

## How It Actually Works

`http.get(...)` does not send anything. It builds an `HttpRequest` object and returns a
**cold** observable describing "send this request when subscribed." We confirmed it: with
the testing backend, creating the observable produced **zero** requests, and subscribing
to it twice produced **two** separate requests. That is why an unsubscribed call silently
does nothing, and why accidentally subscribing twice (say, once in code and once with the
`async` pipe) doubles your traffic.

On subscription, the request travels through a chain of **handlers**: each interceptor
you registered (Level 2) can inspect or replace the request and the resulting stream, and
at the end of the chain sits the **backend** — `FetchBackend` by default, `HttpXhrBackend`
with `withXhr()`, or the testing backend in tests. The fetch backend calls `fetch()`,
passes an `AbortSignal`, reads the body according to `responseType` (JSON by default),
and emits an `HttpResponse` whose `body` the `get` method unwraps for you. Unsubscribing
before the response arrives aborts the underlying request — the basis of cancel-on-new-
search patterns with `switchMap` in Level 2.

`HttpClient` itself is a root-provided service that depends on the handler chain
configured by `provideHttpClient(...features)`. That is why the features — interceptors,
XSRF protection, the backend choice — are passed to one provider function in one place.

## Common mistakes

- **Forgetting to subscribe** to a POST/DELETE, then wondering why the server never saw
  it. (`await firstValueFrom(obs)` is fine too.)
- **Forgetting `provideHttpClient()`** — you get
  `NG0201: No provider found for HttpClient`.
- **Trusting the generic type.** `get<User>()` does not validate anything.
- **Building query strings by hand** (`'?q=' + query`) and breaking on `&` or spaces.
  Use `HttpParams` or the `params` object.
- **Catching an error and returning nothing useful**, leaving the UI spinning. Always
  reset loading state in the error path.
- **Calling a third-party API that doesn't send CORS headers** from the browser. The
  browser blocks it (status 0); you need a server-side proxy. Open Library does send
  them, which is why the Level 1 project can call it directly.

## Exercise

1. Write an `Authors` service that calls Open Library's author search,
   `https://openlibrary.org/search/authors.json?q={name}`, and maps each document to
   `{ key: string; name: string; birthDate?: string; workCount: number }` (the API's
   fields are `key`, `name`, `birth_date` and `work_count`).
2. Add a `work(id)` method that loads `https://openlibrary.org/works/{id}.json`. Build a
   component with a search box, a loading message and an error message, then request a
   work ID that doesn't exist (for example `OL0000000000W`, which returned 404 when we
   tried it) and check how your component handles the error.
3. Write a unit test with `provideHttpClientTesting()` that asserts the URL, flushes a
   fake response and checks the mapped result. Add a second test that flushes a 404 and
   checks the error path.
4. Call `http.get(...)` without subscribing in the test and use
   `HttpTestingController.expectNone` to prove no request was made.
