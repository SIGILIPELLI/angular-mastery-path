---
description: "Test Angular 22 apps with Vitest and TestBed: component tests with setInput and whenStable, outputs, services, HTTP, router, fake timers, coverage."
---

# 07 · Testing with Vitest & TestBed

New Angular 22 projects run unit tests with **Vitest** (version 5 in our projects) in a
`jsdom` environment, through the CLI's `@angular/build:unit-test` builder. `ng test`
compiles your specs with the same Angular compiler as your app, so templates are
type-checked in tests too. Karma is still available (`ng new --test-runner=karma`) for
older projects.

You have already written a few tests in this course. This lesson turns that into a
method: what to test at each level, how to drive components the way users do, and how to
read failures and coverage. Every test below ran against the Level 1 reading-list app.

## What to test where

| Level | Tool | Tests |
|-------|------|-------|
| Pure logic | plain Vitest, no TestBed | pipes, validators, pure functions |
| Services | `TestBed.inject` | stores, API clients (with `HttpTestingController`) |
| Components | `TestBed.createComponent` | rendering, inputs, outputs, user interaction |
| Routing | `RouterTestingHarness` | guards, resolvers, input binding (Level 2, lesson 07) |
| Whole app in a browser | Playwright / Cypress | critical user journeys, end to end |

Aim most of your tests at services and components; keep a smaller number of end-to-end
tests for the flows that make money or would be embarrassing to break. For end-to-end
testing in depth, see [Playwright Mastery Path](https://sigilipelli.github.io/playwright-mastery-path/).

## Component tests: drive the DOM, assert on the DOM

```ts title="src/app/reading/book-card.spec.ts"
import { TestBed } from '@angular/core/testing';
import { provideRouter } from '@angular/router';
import { BookCard } from './book-card';
import { Book } from './book';

const dune: Book = { id: 'b1', title: 'Dune', author: 'Frank Herbert', status: 'want', rating: 0 };

describe('BookCard', () => {
  async function setup(book: Book) {
    TestBed.configureTestingModule({ providers: [provideRouter([])] });
    const fixture = TestBed.createComponent(BookCard);
    fixture.componentRef.setInput('book', book);
    await fixture.whenStable();
    return { fixture, el: fixture.nativeElement as HTMLElement };
  }

  it('renders the title as a link to the detail page', async () => {
    const { el } = await setup(dune);
    const link = el.querySelector('h3 a')!;
    expect(link.textContent).toBe('Dune');
    expect(link.getAttribute('href')).toBe('/books/b1');
  });

  it('only shows the rating control for finished books', async () => {
    const { fixture, el } = await setup(dune);
    expect(el.querySelector('app-star-rating')).toBeNull();

    fixture.componentRef.setInput('book', { ...dune, status: 'done' });
    await fixture.whenStable();
    expect(el.querySelector('app-star-rating')).not.toBeNull();
  });

  it('emits statusChange when a new status is selected', async () => {
    const { fixture, el } = await setup(dune);
    const emitted: string[] = [];
    fixture.componentInstance.statusChange.subscribe((s) => emitted.push(s));

    const select = el.querySelector('select')!;
    select.value = 'reading';
    select.dispatchEvent(new Event('change'));

    expect(emitted).toEqual(['reading']);
  });

  it('emits removed when Remove is clicked', async () => {
    const { fixture, el } = await setup(dune);
    const removed = vi.fn();
    fixture.componentInstance.removed.subscribe(removed);

    el.querySelector<HTMLButtonElement>('button[type=button]:last-of-type')!.click();

    expect(removed).toHaveBeenCalledTimes(1);
  });
});
```

The pattern, in four moves:

1. **Configure** only what the component needs (`provideRouter([])` because the card
   uses `routerLink`).
2. **Create** it and set inputs with `fixture.componentRef.setInput(...)` — the same path
   a parent template uses, including transforms and required-input checks.
3. **Wait** with `await fixture.whenStable()`. In a zoneless, OnPush app, rendering is
   scheduled, not synchronous (lesson 01). `whenStable` waits for pending renders and
   pending tasks. (`fixture.detectChanges()` still exists and forces a synchronous pass,
   but `whenStable` tests the scheduling your users actually get.)
4. **Assert** on what a user would see: text, attributes (`href`, `aria-pressed`),
   presence of elements — and on outputs, by subscribing to the `OutputEmitterRef`.

Avoid asserting on private fields or calling protected methods directly. If a test only
passes because it reaches into internals, it will break on every refactor that doesn't
change behaviour.

## Testing a component with its real store

```ts title="src/app/reading/book-list.spec.ts"
import { TestBed } from '@angular/core/testing';
import { provideRouter } from '@angular/router';
import { BookList } from './book-list';
import { BookStore } from './book-store';

describe('BookList', () => {
  beforeEach(() => {
    localStorage.clear();
    TestBed.configureTestingModule({ providers: [provideRouter([])] });
  });

  it('filters by status using the store', async () => {
    const store = TestBed.inject(BookStore);
    store.add('Dune', 'Frank Herbert');
    store.add('Piranesi', 'Susanna Clarke');
    store.setStatus(store.books()[0].id, 'done');

    const fixture = TestBed.createComponent(BookList);
    await fixture.whenStable();
    const el: HTMLElement = fixture.nativeElement;
    const titles = () => [...el.querySelectorAll('app-book-card h3')].map((h) => h.textContent);

    expect(titles()).toEqual(['Dune', 'Piranesi']);

    const doneButton = [...el.querySelectorAll('nav button')].find((b) => b.textContent!.includes('done'))!;
    (doneButton as HTMLButtonElement).click();
    await fixture.whenStable();

    expect(titles()).toEqual(['Dune']);
    expect(doneButton.getAttribute('aria-pressed')).toBe('true');
  });
});
```

This one uses the **real** `BookStore` — it's in-memory and fast, so a fake would add
nothing. Fake the things that are slow, non-deterministic or external: HTTP (with
`provideHttpClientTesting()`, Level 1 lesson 09), time, and browser APIs jsdom lacks.

To replace a dependency, provide an alternative:

```ts
TestBed.configureTestingModule({
  providers: [
    { provide: OpenLibrary, useValue: { search: () => of([{ key: 'k', title: 'Dune', author: 'F. H.' }]) } },
  ],
});
```

## Reading a failure

We changed one expected value on purpose. Vitest's output:

```text
 FAIL   reading-list  src/app/reading/book-list.spec.ts > BookList > filters by status using the store
AssertionError: expected [ 'Dune' ] to deeply equal [ 'Piranesi' ]

- Expected
+ Received

  [
-   "Piranesi",
+   "Dune",
  ]

 ❯ src/app/reading/book-list.spec.ts:29:22
     27|     await fixture.whenStable();
     28|
     29|     expect(titles()).toEqual(['Piranesi']);
       |                      ^
```

The full test name path, a diff and the exact line — that's why small, well-named tests
with one behaviour each are worth the effort.

## Time, timers and async

- **Fake timers:** `vi.useFakeTimers()` then `vi.advanceTimersByTime(300)` for
  debounces, retries and polling (Level 2, lesson 04 used this for a type-ahead). Call
  `vi.useRealTimers()` afterwards.
- **Effects:** flush them with `TestBed.tick()` before asserting on their side effects
  (the `BookStore` persistence test in Level 1's project does this).
- **Resources and HTTP:** flush requests *before* awaiting `whenStable()`; a pending
  request keeps the app unstable (Level 2, lesson 06).

## Running and coverage

```bash
ng test                        # watch mode in a terminal
ng test --watch=false          # single run, e.g. in CI
ng test --include src/app/reading/book-card.spec.ts
ng test --coverage             # we installed @vitest/coverage-v8 for this
```

With all eight reading-list tests passing, `--coverage` printed:

```text
File             | % Stmts | % Branch | % Funcs | % Lines | Uncovered Line #s
-----------------|---------|----------|---------|---------|-------------------
All files        |   83.67 |    87.93 |   71.42 |    92.1 |
 book-card.ts    |      95 |      100 |      80 |     100 |
 book-list.ts    |   69.81 |    77.77 |   44.44 |   84.61 | 31-34,49-50
 book-store.ts   |    86.2 |      100 |   81.25 |   89.47 | 39,51
 open-library.ts |     100 |    85.71 |     100 |     100 | 33
 star-rating.ts  |   88.23 |    83.33 |      50 |     100 | 14
```

Two things to notice. First, `book-detail.ts` and `book-search.ts` are **missing** from
the table: coverage only included files some test loaded, so an untested file didn't
drag the percentage down — it silently disappeared. Second, `book-list.ts` lines 49–50
(the `add` method) are uncovered: we never tested adding a book through the form.
Coverage is a map of what you haven't looked at, not a grade.

## Component harnesses

For components from `@angular/material` or your own design system, the CDK's
**component harnesses** (`@angular/cdk/testing`) give tests a stable API — for example
`await loader.getHarness(MatButtonHarness.with({ text: 'Save' }))` then
`await button.click()` — instead of depending on internal DOM structure that a library
upgrade may change.

## How It Actually Works

`ng test` builds your spec files (and everything they import) with the Angular
application builder in test mode, then hands the bundles to Vitest. Each spec file runs
in a worker with a `jsdom` window; Angular's testing environment is initialised once per
file, and **`TestBed` is reset before every test**, so each test gets fresh injectors and
fresh root services. That's why `BookStore` state didn't leak from one test to the next
(only `localStorage`, which jsdom keeps for the whole file, needed clearing).

`TestBed.createComponent` bootstraps a tiny application around one component: an
environment injector from your `configureTestingModule` providers, a host element, and a
`ComponentFixture`. In zoneless mode (the default), the fixture uses the same
change-detection scheduler as a real app, so `whenStable()` resolves when the scheduler
has nothing left to do and there are no pending tasks.

`HttpTestingController` swaps the backend at the end of the interceptor chain (Level 2,
lesson 08), which means interceptors *do* run in your tests if you register them —
useful when you want to test them, surprising when you don't.

## Common mistakes

- **Asserting immediately after an event** without `await fixture.whenStable()`.
- **Mocking everything**, so tests only check that mocks were called. Use real
  collaborators when they are fast and deterministic.
- **Testing implementation details** (private fields, call counts) instead of behaviour.
- **Chasing a coverage number.** Untested files may not even appear in the report.
- **Shared mutable fixtures** between tests; build what each test needs in the test.

## Exercise

1. Write tests for `StarRating`: clicking the third star sets the value to 3; clicking it
   again clears to 0; `aria-checked` follows the value.
2. Write a test for adding a book through `BookList`'s form (fill both inputs, submit,
   assert the new card appears). Re-run coverage and confirm lines 49–50 are covered.
3. Add a test file for `BookDetail` using `RouterTestingHarness` and check that the file
   now appears in the coverage table.
4. Make one test fail on purpose and practise reading the diff.
