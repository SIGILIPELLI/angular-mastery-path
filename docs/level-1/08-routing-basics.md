---
description: "Angular Router basics: routes, routerLink and routerLinkActive, params bound to inputs, redirects, wildcards, titles and lazy loading."
---

# 08 · Routing Basics

A single-page application still needs URLs: people bookmark pages, share links, press
Back and reload. The Angular Router maps the browser's URL to components, updates the URL
when the user navigates in the app, and loads code for a route only when it is first
needed.

## Configuring routes

Routes are a plain array of objects:

```ts title="src/app/app.routes.ts"
import { Routes } from '@angular/router';
import { BookList } from './reading/book-list';

export const routes: Routes = [
  { path: '', redirectTo: 'books', pathMatch: 'full' },
  { path: 'books', component: BookList, title: 'My books' },
  {
    path: 'books/:id',
    loadComponent: () => import('./reading/book-detail').then((m) => m.BookDetail),
    title: 'Book',
  },
  {
    path: 'search',
    loadComponent: () => import('./reading/book-search').then((m) => m.BookSearch),
    title: 'Search',
  },
  { path: '**', redirectTo: 'books' },
];
```

And they are registered once, in the application config:

```ts title="src/app/app.config.ts"
import { ApplicationConfig, provideBrowserGlobalErrorListeners } from '@angular/core';
import { provideRouter, withComponentInputBinding } from '@angular/router';
import { routes } from './app.routes';

export const appConfig: ApplicationConfig = {
  providers: [
    provideBrowserGlobalErrorListeners(),
    provideRouter(routes, withComponentInputBinding()),
  ],
};
```

What each piece means:

- **`path`** — a URL segment pattern, *without* a leading slash. `:id` is a parameter.
- **`component`** — the component to show, loaded eagerly (part of the main bundle).
- **`loadComponent`** — a function returning a dynamic `import()`. The component's code
  goes into its own chunk and is downloaded the first time someone visits. If the file
  uses `export default`, you can write `loadComponent: () => import('./about')`.
- **`redirectTo`** — send this URL somewhere else. With `path: ''`, always add
  `pathMatch: 'full'`, otherwise *every* URL starts with the empty string and matches.
- **`'**'`** — the wildcard. It matches anything, so it must be **last**; the router
  checks routes in order and takes the first match.
- **`title`** — sets `document.title` after navigation.

## Where routed components appear: `<router-outlet>`

```ts title="src/app/app.ts"
import { Component } from '@angular/core';
import { RouterLink, RouterLinkActive, RouterOutlet } from '@angular/router';

@Component({
  selector: 'app-root',
  imports: [RouterOutlet, RouterLink, RouterLinkActive],
  template: `
    <header>
      <nav>
        <a routerLink="/books" routerLinkActive="active">My books</a>
        <a routerLink="/search" routerLinkActive="active" ariaCurrentWhenActive="page">Search</a>
      </nav>
    </header>
    <main><router-outlet /></main>
  `,
})
export class App {}
```

The header stays; whatever the current route matches is rendered right after
`<router-outlet />`.

## Linking and navigating

- **`routerLink="/books"`** — navigate without a full page reload. It renders a real
  `href`, so middle-click, "open in new tab" and crawlers still work.
- **Array form for dynamic links:** `[routerLink]="['/books', book.id]"` produces
  `/books/42`, encoding each segment safely.
- **Query parameters:** `[queryParams]="{ tab: 'notes' }"` → `/books/42?tab=notes`.
- **`routerLinkActive="active"`** adds a class when the link's route is active. Add
  `[routerLinkActiveOptions]="{ exact: true }"` for links like `/` that would otherwise be
  active everywhere, and `ariaCurrentWhenActive="page"` for screen readers.

From code, inject the `Router`:

```ts
import { Router } from '@angular/router';

export class BookForm {
  private readonly router = inject(Router);

  protected async saved(id: string) {
    await this.router.navigate(['/books', id]);        // array of segments
    // or: this.router.navigateByUrl(`/books/${id}`);  // full URL string
  }
}
```

Both return a promise that resolves to `true` when navigation succeeded.

## Route parameters as component inputs

With `withComponentInputBinding()` enabled, the router sets **inputs** on the routed
component from path parameters, query parameters and route `data`, matched by name:

```ts title="src/app/reading/book-detail.ts"
import { Component, computed, inject, input } from '@angular/core';
import { RouterLink } from '@angular/router';
import { BookStore } from './book-store';

@Component({
  selector: 'app-book-detail',
  imports: [RouterLink],
  template: `
    @if (book(); as b) {
      <h2>{{ b.title }}</h2>
      <p>by {{ b.author }}</p>
    } @else {
      <p>No book with id “{{ id() }}”.</p>
    }
    <a routerLink="/books">← Back</a>
  `,
})
export class BookDetail {
  readonly id = input.required<string>();     // from :id
  readonly tab = input<string>();             // from ?tab=...
  private readonly store = inject(BookStore);
  protected readonly book = computed(() => this.store.byId(this.id()));
}
```

In a test that navigated with `RouterTestingHarness`, a component with those two inputs
rendered:

```text
/books/42?tab=notes   →  id=42 tab=notes    (document.title: "Book details")
/books/7              →  id=7  tab=none
```

Parameters are always **strings**; convert with `Number()` or an input transform if you
need a number. Because `id` is a signal, `book` recomputes when you navigate from
`/books/1` to `/books/2` — the router **reuses** the component instance when only
parameters change, so code that read the id once in the constructor would show the wrong
book.

Without input binding, you read the same data from `ActivatedRoute`, which exposes
observables:

```ts
private readonly route = inject(ActivatedRoute);
protected readonly id = toSignal(this.route.paramMap.pipe(map((p) => p.get('id'))));
```

That style is everywhere in existing code, and it still works.

## How It Actually Works

When the URL changes — a `routerLink` click, `router.navigate`, or the browser's Back
button (a `popstate` event) — the router runs a **navigation pipeline**:

1. **Parse** the URL string into a `UrlTree`: segments, parameters, query params and
   fragment.
2. **Apply redirects** and **match** the tree against your config, depth-first and in
   order, producing a tree of matched routes. For lazy routes, this is where the
   `loadComponent` import happens (only once — the result is cached).
3. **Run guards and resolvers** (Level 2, lesson 07). Any guard can cancel or redirect.
4. **Create a `RouterState`** — the tree of `ActivatedRoute` objects with params and data.
5. **Activate**: compare the new state with the old one. Outlets whose route config did
   not change keep their component instance and just receive new params; others destroy
   the old component and create the new one next to `<router-outlet>`.
6. **Update the browser URL** with `history.pushState` (or `replaceState`), and set the
   title.

`routerLink` renders an ordinary `<a href="/books/42">`. Its click listener cancels the
browser's default navigation only for plain left-clicks, which is why Ctrl/Cmd-click
still opens a new tab.

Because the app uses real paths (`/books/42`) instead of `#/books/42`, the **web server**
must return `index.html` for every app URL, or a reload on `/books/42` gives a 404. The
dev server does this for you; production hosting needs a "SPA fallback" rule (Level 4,
lesson 07).

## Common mistakes

- **Leading slash in `path`.** `{ path: '/books' }` is an error; paths are relative
  segments.
- **Wildcard not last.** Everything after `'**'` is unreachable.
- **Forgetting `pathMatch: 'full'` on an empty-path redirect** — every URL matches the
  empty prefix, so everything gets redirected.
- **Reading params once** (`this.route.snapshot.paramMap.get('id')` in the constructor)
  and then navigating between two detail pages. The component is reused and shows stale
  data. Use an input signal or the observable.
- **Using `href` instead of `routerLink`** for in-app links, which reloads the whole app.
- **Forgetting to import `RouterLink`** into a component that uses it. The template
  compiles (`routerLink` looks like a plain attribute) but the link does nothing — check
  the component's `imports` first when a link "doesn't work".

## Exercise

1. Add three routes to a new app: `/` (home), `/products` (a list of five hard-coded
   products) and `/products/:slug` (a detail page), plus a wildcard that shows a
   "Page not found" component with its own `title`.
2. Load the detail and not-found components lazily and confirm with `ng build` that they
   appear as lazy chunks.
3. Enable `withComponentInputBinding()` and read `slug` as an input. Add "Next product"
   links on the detail page and verify the page updates correctly when clicking from one
   product to the next.
4. Support `?sort=price` on `/products` through an input, and sort the list with a
   `computed`.
5. Highlight the current nav link with `routerLinkActive`, using `exact: true` for Home.
