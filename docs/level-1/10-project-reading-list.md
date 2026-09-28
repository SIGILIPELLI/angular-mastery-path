---
description: "Level 1 project: build a routed Angular reading-list app with a signal-based store, localStorage persistence, a model() rating control and Open Library search."
---

# 10 · Project — A Reading List App

This project puts every Level 1 lesson into one small, complete application. You will
build a reading list where you can add books, move them between *want to read*,
*reading* and *done*, rate finished books, open a detail page per book, and search the
public Open Library catalogue to add titles. Everything is kept in `localStorage`, so it
survives a reload.

Every file below was compiled with `ng build` on Angular 22.2, tested with `ng test`, and
exercised in headless Chromium. The results of that run are shown in the
[Checking it works](#checking-it-works) section.

| Lesson | Where it shows up |
|--------|-------------------|
| 02 CLI | `ng new`, `ng build`, `ng test` |
| 03 Templates | bindings and events in every component |
| 04 Signals | the store's `signal`, `computed` and `effect` |
| 05 Control flow | `@for` with `track`, `@if`/`@else`, `@empty` |
| 06 Inputs/outputs | `BookCard` inputs and outputs, `StarRating` with `model()` |
| 07 Services | `BookStore` and `OpenLibrary` with `@Service()` |
| 08 Routing | list, detail and search routes; `:id` bound to an input; lazy chunks |
| 09 HttpClient | the Open Library search |

## Step 1 — Create the project

```bash
npx @angular/cli@22 new reading-list --defaults --inline-template --inline-style
cd reading-list
```

Delete the generated `src/app/app.html`, `app.css` and `app.spec.ts` if they exist; the
root component in this project has an inline template. All app code lives under
`src/app/reading/`.

## Step 2 — The model

```ts title="src/app/reading/book.ts"
export type Status = 'want' | 'reading' | 'done';

export interface Book {
  id: string;
  title: string;
  author: string;
  status: Status;
  rating: number; // 0 = not rated
}
```

`rating` uses `0` for "not rated" rather than `undefined`, which keeps the JSON in
`localStorage` simple and avoids optional-chaining everywhere.

## Step 3 — The store

All state lives in one service. Components read signals and call methods; only the store
writes.

```ts title="src/app/reading/book-store.ts"
import { Service, computed, effect, signal } from '@angular/core';
import { Book, Status } from './book';

const KEY = 'reading-list';

@Service()
export class BookStore {
  private readonly _books = signal<Book[]>(load());

  readonly books = this._books.asReadonly();
  readonly counts = computed(() => {
    const c = { want: 0, reading: 0, done: 0 };
    for (const b of this._books()) c[b.status]++;
    return c;
  });

  constructor() {
    effect(() => localStorage.setItem(KEY, JSON.stringify(this._books())));
  }

  byId(id: string) {
    return this._books().find((b) => b.id === id);
  }

  add(title: string, author: string) {
    const book: Book = { id: crypto.randomUUID(), title, author, status: 'want', rating: 0 };
    this._books.update((list) => [...list, book]);
  }

  setStatus(id: string, status: Status) {
    this.patch(id, { status });
  }

  rate(id: string, rating: number) {
    this.patch(id, { rating });
  }

  remove(id: string) {
    this._books.update((list) => list.filter((b) => b.id !== id));
  }

  private patch(id: string, changes: Partial<Book>) {
    this._books.update((list) => list.map((b) => (b.id === id ? { ...b, ...changes } : b)));
  }
}

function load(): Book[] {
  try {
    return JSON.parse(localStorage.getItem(KEY) ?? '[]');
  } catch {
    return [];
  }
}
```

Design notes:

- `_books` is private and writable; `books` is a read-only view. No component can put
  the list into an invalid state by writing to it directly.
- `counts` is a `computed`, so the three tallies are recalculated only when the list
  changes, not on every render.
- The `effect` in the constructor persists every change. Effects batch, so three quick
  changes cause one write.
- Every update produces a **new array** and, for patched books, a **new object**
  (lesson 04). Mutating in place would leave the UI stale.
- `load()` tolerates corrupted storage by falling back to an empty list — the kind of
  small defensive choice that saves a user from a blank screen.

## Step 4 — A reusable rating control with `model()`

```ts title="src/app/reading/star-rating.ts"
import { Component, input, model } from '@angular/core';

@Component({
  selector: 'app-star-rating',
  template: `
    <span role="radiogroup" [attr.aria-label]="label()">
      @for (star of stars; track star) {
        <button
          type="button"
          role="radio"
          [attr.aria-checked]="star === value()"
          [attr.aria-label]="star + ' of 5'"
          [class.on]="star <= value()"
          (click)="value.set(star === value() ? 0 : star)">★</button>
      }
    </span>
  `,
  styles: `
    button { border: 0; background: none; font-size: 1.25rem; color: #bbb; cursor: pointer; }
    button.on { color: #e0a100; }
  `,
})
export class StarRating {
  readonly value = model(0);
  readonly label = input('Rating');
  protected readonly stars = [1, 2, 3, 4, 5];
}
```

`value` is a `model()`, so a parent can use `[(value)]` or, as below, bind the input and
the `valueChange` output separately. Clicking the current star again clears the rating.
The radio-group roles and labels give screen-reader users a meaningful control, not five
unlabeled "★" buttons.

## Step 5 — The book card

```ts title="src/app/reading/book-card.ts"
import { Component, input, output } from '@angular/core';
import { RouterLink } from '@angular/router';
import { Book, Status } from './book';
import { StarRating } from './star-rating';

@Component({
  selector: 'app-book-card',
  imports: [RouterLink, StarRating],
  template: `
    <article>
      <h3><a [routerLink]="['/books', book().id]">{{ book().title }}</a></h3>
      <p>{{ book().author }}</p>
      <select [value]="book().status" (change)="statusChange.emit(asStatus($event))" aria-label="Status">
        <option value="want">Want to read</option>
        <option value="reading">Reading</option>
        <option value="done">Done</option>
      </select>
      @if (book().status === 'done') {
        <app-star-rating [value]="book().rating" (valueChange)="rated.emit($event)" />
      }
      <button type="button" (click)="removed.emit()">Remove</button>
    </article>
  `,
})
export class BookCard {
  readonly book = input.required<Book>();
  readonly statusChange = output<Status>();
  readonly rated = output<number>();
  readonly removed = output<void>();

  protected asStatus(event: Event) {
    return (event.target as HTMLSelectElement).value as Status;
  }
}
```

The card is **presentational**: it receives a `book` and emits intentions
(`statusChange`, `rated`, `removed`). It doesn't know the store exists, which makes it
trivial to reuse and to test with a plain input.

## Step 6 — The list page

```ts title="src/app/reading/book-list.ts"
import { Component, computed, inject, signal } from '@angular/core';
import { BookCard } from './book-card';
import { BookStore } from './book-store';
import { Status } from './book';

type Filter = Status | 'all';

@Component({
  selector: 'app-book-list',
  imports: [BookCard],
  template: `
    <h2>My books</h2>
    <nav aria-label="Filter">
      @for (f of filters; track f) {
        <button type="button" [attr.aria-pressed]="filter() === f" (click)="filter.set(f)">
          {{ f }} @if (f !== 'all') { ({{ store.counts()[f] }}) }
        </button>
      }
    </nav>

    <form (submit)="add($event, title.value, author.value); title.value = ''; author.value = ''">
      <input #title placeholder="Title" required />
      <input #author placeholder="Author" required />
      <button>Add</button>
    </form>

    @for (book of visible(); track book.id) {
      <app-book-card
        [book]="book"
        (statusChange)="store.setStatus(book.id, $event)"
        (rated)="store.rate(book.id, $event)"
        (removed)="store.remove(book.id)" />
    } @empty {
      <p>No books here yet.</p>
    }
  `,
})
export class BookList {
  protected readonly store = inject(BookStore);
  protected readonly filters: Filter[] = ['all', 'want', 'reading', 'done'];
  protected readonly filter = signal<Filter>('all');
  protected readonly visible = computed(() => {
    const f = this.filter();
    const books = this.store.books();
    return f === 'all' ? books : books.filter((b) => b.status === f);
  });

  protected add(event: Event, title: string, author: string) {
    event.preventDefault();
    if (title.trim() && author.trim()) this.store.add(title.trim(), author.trim());
  }
}
```

`visible` is derived from two signals — the store's books and the local filter — so
changing either re-renders the list and nothing else. The filter buttons expose their
state with `aria-pressed`.

## Step 7 — The detail page

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
      <p>by {{ b.author }} — {{ b.status }}</p>
    } @else {
      <p>No book with id “{{ id() }}”.</p>
    }
    <a routerLink="/books">← Back</a>
  `,
})
export class BookDetail {
  readonly id = input.required<string>(); // filled from the :id route parameter
  private readonly store = inject(BookStore);
  protected readonly book = computed(() => this.store.byId(this.id()));
}
```

`id` is filled from the `:id` route parameter because the app enables
`withComponentInputBinding()` (next step). `book` is a `computed` over the input and the
store, so it stays correct if the router reuses the component for another id, and it
also updates if the book is edited elsewhere.

## Step 8 — Searching Open Library

The API client from lesson 09:

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

And the search page:

```ts title="src/app/reading/book-search.ts"
import { Component, inject, signal } from '@angular/core';
import { BookStore } from './book-store';
import { OpenLibrary, SearchHit } from './open-library';

@Component({
  selector: 'app-book-search',
  template: `
    <h2>Find a book</h2>
    <form (submit)="$event.preventDefault(); search(q.value)">
      <input #q type="search" placeholder="Search Open Library" />
      <button [disabled]="loading()">Search</button>
    </form>

    @if (loading()) {
      <p>Searching…</p>
    } @else if (error()) {
      <p role="alert">{{ error() }}</p>
    } @else {
      <ul>
        @for (hit of results(); track hit.key) {
          <li>
            {{ hit.title }} — {{ hit.author }} @if (hit.year) { ({{ hit.year }}) }
            <button type="button" (click)="store.add(hit.title, hit.author)">Add</button>
          </li>
        }
      </ul>
    }
  `,
})
export class BookSearch {
  private readonly api = inject(OpenLibrary);
  protected readonly store = inject(BookStore);
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
}
```

## Step 9 — Routes, config and the shell

```ts title="src/app/app.routes.ts"
import { Routes } from '@angular/router';
import { BookList } from './reading/book-list';

export const routes: Routes = [
  { path: '', redirectTo: 'books', pathMatch: 'full' },
  { path: 'books', component: BookList, title: 'My books' },
  { path: 'books/:id', loadComponent: () => import('./reading/book-detail').then((m) => m.BookDetail), title: 'Book' },
  { path: 'search', loadComponent: () => import('./reading/book-search').then((m) => m.BookSearch), title: 'Search' },
  { path: '**', redirectTo: 'books' },
];
```

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

```ts title="src/app/app.ts"
import { Component } from '@angular/core';
import { RouterLink, RouterLinkActive, RouterOutlet } from '@angular/router';

@Component({
  selector: 'app-root',
  imports: [RouterOutlet, RouterLink, RouterLinkActive],
  template: `
    <header>
      <h1>Reading List</h1>
      <nav>
        <a routerLink="/books" routerLinkActive="active">My books</a>
        <a routerLink="/search" routerLinkActive="active">Search</a>
      </nav>
    </header>
    <main><router-outlet /></main>
  `,
})
export class App {}
```

## Step 10 — Tests

```ts title="src/app/reading/book-store.spec.ts"
import { TestBed } from '@angular/core/testing';
import { BookStore } from './book-store';

describe('BookStore', () => {
  beforeEach(() => localStorage.clear());

  it('adds books and counts them by status', () => {
    const store = TestBed.inject(BookStore);
    store.add('Dune', 'Frank Herbert');
    store.add('Piranesi', 'Susanna Clarke');
    const [dune] = store.books();
    store.setStatus(dune.id, 'done');
    store.rate(dune.id, 4);

    expect(store.counts()).toEqual({ want: 1, reading: 0, done: 1 });
    expect(store.byId(dune.id)?.rating).toBe(4);
  });

  it('persists to localStorage', () => {
    const store = TestBed.inject(BookStore);
    store.add('Dune', 'Frank Herbert');
    TestBed.tick(); // flush the effect
    const saved = JSON.parse(localStorage.getItem('reading-list')!);
    expect(saved[0].title).toBe('Dune');
  });
});
```

Note `TestBed.tick()` in the second test: effects run asynchronously, so a test that
checks what an effect did must flush them first. Add the `OpenLibrary` test from
lesson 09 as `src/app/reading/open-library.spec.ts`.

## Checking it works

`ng test` passed all three tests (the two above plus the lesson 09 API test). `ng build`
produced an initial bundle of 260.99 kB raw (73.00 kB estimated transfer) with
`book-search` and `book-detail` split into lazy chunks of 2.25 kB and 878 bytes.

We then served the production build and drove it with Playwright in headless Chromium:
added *Dune* and *Piranesi*, set *Dune* to done, clicked the fourth star, reloaded,
opened the detail page, visited a bad id, and searched for "dune". Observed results:

```text
url http://127.0.0.1:4341/books title My books
empty: No books here yet.
cards 2
filters: ['all', 'want (1)', 'reading (0)', 'done (1)']
stored: [{"id":"0b1d56c3-…","title":"Dune","author":"Frank Herbert","status":"done","rating":4},
         {"id":"74352b50-…","title":"Piranesi","author":"Susanna Clarke","status":"want","rating":0}]
after reload cards 2
detail: Dune | by Frank Herbert — done | ← Back
missing: No book with id “nope”. title= Book
search first 3: ['Dune — Frank Herbert (1965) Add', 'Dune Messiah — Frank Herbert (1969) Add',
                 'Children of Dune — Frank Herbert (1976) Add']
console errors []
```

(The search results come from the live Open Library API and will change over time.)

One thing that run taught us: in the first attempt, the script counted the cards
immediately after clicking **Add** and got `1` instead of `2`. The app was fine — in a
zoneless app, rendering is **scheduled** after a signal changes rather than happening
synchronously inside the click, and the script checked too early. In tests, use
`await fixture.whenStable()` or assertions that wait for the DOM, never a check straight
after the event.

## How It Actually Works

Follow one click — setting *Dune* to "done" — through the app:

1. The `<select>`'s native `change` event fires. The listener compiled from
   `(change)="statusChange.emit(asStatus($event))"` runs inside `BookCard`.
2. `statusChange.emit('done')` synchronously calls the listener the parent `BookList`
   registered with `(statusChange)="store.setStatus(book.id, $event)"`.
3. `setStatus` calls `patch`, which calls `_books.update(...)` with a new array. The
   signal bumps its version and marks its consumers dirty: the `counts` and `visible`
   computeds, the persistence effect, and — through those — the templates of `BookList`
   and each `BookCard` that read them.
4. Because a template consumer was marked dirty, Angular schedules change detection. When
   it runs, it refreshes only views marked for refresh: `BookList` re-evaluates
   `visible()` and `store.counts()`; the `@for` repeater sees the same track keys, keeps
   every card, and gives the *Dune* card its new `book` object; that card's `@if` flips
   on and the `StarRating` is created.
5. The effect queue is flushed in the same cycle, and the effect writes the new list to
   `localStorage`.

Nothing re-renders "the whole app", and no code in the app asked for a render. The
signal graph carried the change from the store to exactly the views that depend on it.

## Extensions

Pick at least two:

1. **Sort options.** Add a sort signal (title, author, rating) and fold it into `visible`.
2. **Debounced search.** Search as the user types, waiting 300 ms after the last
   keystroke. You'll find a clean way to do it in Level 2, lesson 04 (`switchMap`), but
   try it first with `setTimeout` and see what goes wrong with overlapping responses.
3. **Duplicate protection.** Stop the same title/author pair from being added twice and
   show a message instead.
4. **Detail editing.** Let the detail page change status and rating using the same
   `StarRating` component.
5. **Tests for the card.** Render `BookCard` with `setInput('book', ...)`, click
   **Remove**, and assert the `removed` output fired.

## Exercise

Without looking back at the code, write down which lesson's concept solves each of these,
then check your answers against the table at the top:

1. The detail page must update when navigating from one book to another.
2. The list must not re-render every row when only one book's rating changes.
3. The rating control must be usable with two-way binding.
4. The book data must be shared by the list, the detail page and the search page.
5. The search page's code should not download until someone visits it.
