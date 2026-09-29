---
description: "Level 2 project: an Angular issue tracker with httpResource, URL filters, signal forms with server errors, interceptors and a guard."
---

# 10 · Project — An Issue Tracker

This project brings Level 2 together in a small but realistic CRUD app: list issues
with search and status filters kept in the URL, open an issue, change its status or
delete it, and create new issues through a validated form that also shows errors coming
back from the server.

To keep it runnable with no backend, the "server" is an **interceptor** that implements a
small REST API in memory, with 300 ms of latency so loading states are visible. Swap it
for a real API by removing one line from `app.config.ts`.

| Lesson | Where it shows up |
|--------|-------------------|
| 02 Signal forms | `NewIssuePage`: schema validation, `submit()`, server error on `title` |
| 03 DI | `Busy` service shared by an interceptor and the header |
| 04 RxJS | the fake backend (`of`, `delay`, `timer`, `switchMap`), `finalize` |
| 06 Resources | `httpResource` for the list and the detail page |
| 07 Router | lazy routes, query params as inputs, `canDeactivate` |
| 08 Interceptors | busy counter, error mapping, the fake backend |
| 09 Pipes & directives | `DatePipe` with explicit time zone, host bindings in `StatusBadge` |

Every file below was built with `ng build`, tested with `ng test`, and driven end-to-end
in headless Chromium on Angular 22.2. The results are in
[Checking it works](#checking-it-works).

## Setup

```bash
npx @angular/cli@22 new issue-tracker --defaults --inline-template --inline-style
cd issue-tracker
```

Files live under `src/app/issues/` (feature code) and `src/app/http/` (HTTP plumbing).

## The model

```ts title="src/app/issues/issue.ts"
export type IssueStatus = 'open' | 'in-progress' | 'closed';
export type Priority = 'low' | 'medium' | 'high';

export interface Issue {
  id: number;
  title: string;
  description: string;
  status: IssueStatus;
  priority: Priority;
  createdAt: string; // ISO timestamp
}

export type NewIssue = Pick<Issue, 'title' | 'description' | 'priority'>;
```

## The in-memory API

```ts title="src/app/http/fake-backend.interceptor.ts"
import { HttpErrorResponse, HttpInterceptorFn, HttpRequest, HttpResponse } from '@angular/common/http';
import { Observable, delay, of, switchMap, throwError, timer } from 'rxjs';
import { Issue, NewIssue } from '../issues/issue';

// An in-memory REST API so the project runs with no server.
// Only requests under /api/ are handled; everything else passes through.
let issues: Issue[] = [
  { id: 1, title: 'Login button misaligned on mobile', description: 'Overlaps the logo below 360px.', status: 'open', priority: 'medium', createdAt: '2026-09-20T09:00:00Z' },
  { id: 2, title: 'Export to CSV drops the last row', description: 'Reproducible with any report over 100 rows.', status: 'in-progress', priority: 'high', createdAt: '2026-09-22T14:30:00Z' },
  { id: 3, title: 'Typo in welcome email', description: '"Welcom" in the subject line.', status: 'closed', priority: 'low', createdAt: '2026-09-25T08:15:00Z' },
];
let nextId = 4;
const LATENCY_MS = 300;

export function resetFakeDb(data: Issue[]) {
  issues = structuredClone(data);
  nextId = Math.max(0, ...issues.map((i) => i.id)) + 1;
}

export const fakeBackendInterceptor: HttpInterceptorFn = (req, next) => {
  const url = new URL(req.urlWithParams, 'http://local');
  if (!url.pathname.startsWith('/api/issues')) return next(req);
  const id = Number(url.pathname.split('/')[3]) || null;

  switch (true) {
    case req.method === 'GET' && id === null: {
      const q = (url.searchParams.get('q') ?? '').toLowerCase();
      const status = url.searchParams.get('status');
      const result = issues
        .filter((i) => !q || i.title.toLowerCase().includes(q))
        .filter((i) => !status || i.status === status)
        .sort((a, b) => b.createdAt.localeCompare(a.createdAt));
      return ok(result);
    }
    case req.method === 'GET': {
      const issue = issues.find((i) => i.id === id);
      return issue ? ok(issue) : fail(404, `Issue ${id} not found`);
    }
    case req.method === 'POST': {
      const body = (req as HttpRequest<NewIssue>).body!;
      if (issues.some((i) => i.title.toLowerCase() === body.title.trim().toLowerCase())) {
        return fail(409, 'An issue with this title already exists.');
      }
      const created: Issue = { ...body, title: body.title.trim(), id: nextId++, status: 'open', createdAt: new Date().toISOString() };
      issues = [...issues, created];
      return ok(created, 201);
    }
    case req.method === 'PATCH': {
      const existing = issues.find((i) => i.id === id);
      if (!existing) return fail(404, `Issue ${id} not found`);
      const updated = { ...existing, ...(req.body as Partial<Issue>), id: existing.id };
      issues = issues.map((i) => (i.id === id ? updated : i));
      return ok(updated);
    }
    case req.method === 'DELETE':
      issues = issues.filter((i) => i.id !== id);
      return ok(null, 204);
    default:
      return fail(405, 'Method not allowed');
  }

  function ok<T>(body: T, status = 200): Observable<HttpResponse<T>> {
    return of(new HttpResponse({ status, body: structuredClone(body), url: req.url })).pipe(delay(LATENCY_MS));
  }
  function fail(status: number, message: string): Observable<never> {
    // delay() only delays values, not errors — so wait first, then fail.
    return timer(LATENCY_MS).pipe(
      switchMap(() => throwError(() => new HttpErrorResponse({ status, error: { message }, url: req.url }))),
    );
  }
};
```

It implements `GET /api/issues?q=&status=`, `GET /api/issues/:id`, `POST` (with a 409 on a
duplicate title), `PATCH` and `DELETE`, and returns clones so the app can't accidentally
mutate the "database".

One bug worth knowing about: the first version used `throwError(...).pipe(delay(LATENCY_MS))`
for failures. A unit test for the 409 path finished in 43 ms instead of 300+ — RxJS
`delay` delays **values**, not error notifications, so errors arrived instantly. The
`timer(...).pipe(switchMap(() => throwError(...)))` version waits first; afterwards the
same test took 350 ms. Fake backends that fail faster than they succeed hide real
loading-state bugs.

## HTTP plumbing

```ts title="src/app/http/api-error.ts"
import { HttpErrorResponse, HttpInterceptorFn } from '@angular/common/http';
import { catchError, throwError } from 'rxjs';

export class ApiError extends Error {
  constructor(readonly status: number, message: string) {
    super(message);
  }
}

export const errorInterceptor: HttpInterceptorFn = (req, next) =>
  next(req).pipe(
    catchError((err: HttpErrorResponse) =>
      throwError(() => new ApiError(err.status, err.status === 0 ? 'Network error — check your connection.' : err.error?.message ?? err.statusText)),
    ),
  );
```

```ts title="src/app/http/busy.ts"
import { HttpInterceptorFn } from '@angular/common/http';
import { Service, computed, inject, signal } from '@angular/core';
import { finalize } from 'rxjs';

@Service()
export class Busy {
  private readonly count = signal(0);
  readonly active = computed(() => this.count() > 0);
  start() { this.count.update((n) => n + 1); }
  stop() { this.count.update((n) => n - 1); }
}

export const busyInterceptor: HttpInterceptorFn = (req, next) => {
  const busy = inject(Busy);
  busy.start();
  return next(req).pipe(finalize(() => busy.stop()));
};
```

`Busy` exposes a `computed` boolean rather than the raw counter, so the header's template
can't depend on implementation details.

## The API service for writes

```ts title="src/app/issues/issue-api.ts"
import { HttpClient } from '@angular/common/http';
import { Service, inject } from '@angular/core';
import { firstValueFrom } from 'rxjs';
import { Issue, IssueStatus, NewIssue } from './issue';

@Service()
export class IssueApi {
  private readonly http = inject(HttpClient);

  create(input: NewIssue) {
    return firstValueFrom(this.http.post<Issue>('/api/issues', input));
  }

  setStatus(id: number, status: IssueStatus) {
    return firstValueFrom(this.http.patch<Issue>(`/api/issues/${id}`, { status }));
  }

  remove(id: number) {
    return firstValueFrom(this.http.delete<void>(`/api/issues/${id}`));
  }
}
```

Reads use `httpResource` in the components; writes go through this service. The methods
return Promises (`firstValueFrom`) because they're awaited from event handlers and
`submit()` actions.

## A status badge

```ts title="src/app/issues/status-badge.ts"
import { Component, input } from '@angular/core';
import { IssueStatus } from './issue';

@Component({
  selector: 'app-status-badge',
  host: { '[class]': 'status()', role: 'status' },
  template: `
    @let s = status();
    @switch (s) {
      @case ('open') { Open }
      @case ('in-progress') { In progress }
      @case ('closed') { Closed }
      @default never;
    }
  `,
  styles: `
    :host { display: inline-block; padding: 0 0.5rem; border-radius: 999px; font-size: 0.8rem; }
    :host(.open) { background: #dff5e1; }
    :host(.in-progress) { background: #fff1c7; }
    :host(.closed) { background: #e6e6e6; }
  `,
})
export class StatusBadge {
  readonly status = input.required<IssueStatus>();
}
```

The `@let` before `@switch` is deliberate: switching directly on `status()` with
`@default never;` failed to compile, because TypeScript can't narrow a function call
(Level 1, lesson 05). The `host` binding puts the status on the element as a class, and
`:host(.open)` styles it.

## The list page: filters in the URL

```ts title="src/app/issues/issue-list.ts"
import { Component, inject, input } from '@angular/core';
import { httpResource } from '@angular/common/http';
import { DatePipe } from '@angular/common';
import { Router, RouterLink } from '@angular/router';
import { Issue, IssueStatus } from './issue';
import { StatusBadge } from './status-badge';

@Component({
  selector: 'app-issue-list',
  imports: [RouterLink, DatePipe, StatusBadge],
  template: `
    <h2>Issues</h2>
    <form role="search" (submit)="$event.preventDefault(); search(q.value, st.value)">
      <input #q type="search" name="q" aria-label="Search titles" [value]="query() ?? ''" />
      <select #st name="status" aria-label="Status" [value]="status() ?? ''">
        <option value="">Any status</option>
        <option value="open">Open</option>
        <option value="in-progress">In progress</option>
        <option value="closed">Closed</option>
      </select>
      <button>Filter</button>
      <a routerLink="/issues/new">New issue</a>
    </form>

    @if (issues.error(); as err) {
      <p role="alert">{{ err.message }} <button type="button" (click)="issues.reload()">Retry</button></p>
    } @else if (issues.hasValue()) {
      <ul class="issues" [attr.aria-busy]="issues.isLoading()">
        @for (issue of issues.value(); track issue.id) {
          <li>
            <a [routerLink]="['/issues', issue.id]">#{{ issue.id }} {{ issue.title }}</a>
            <app-status-badge [status]="issue.status" />
            <small>{{ issue.priority }} · {{ issue.createdAt | date: 'mediumDate' : 'UTC' }}</small>
          </li>
        } @empty {
          <li>No issues match.</li>
        }
      </ul>
    } @else {
      <p>Loading…</p>
    }
  `,
})
export class IssueList {
  // Bound from query params (?q=...&status=...) by withComponentInputBinding().
  readonly query = input<string>(undefined, { alias: 'q' });
  readonly status = input<IssueStatus>();
  private readonly router = inject(Router);

  protected readonly issues = httpResource<Issue[]>(() => ({
    url: '/api/issues',
    params: { q: this.query() ?? '', status: this.status() ?? '' },
  }));

  protected search(q: string, status: string) {
    this.router.navigate([], { queryParams: { q: q || null, status: status || null } });
  }
}
```

The filters are **query-parameter inputs** (`?q=export&status=open`), so filtered views
are bookmarkable and survive reloads, and the Back button undoes a filter change.
Submitting the form only navigates; the `httpResource` notices the new inputs and
refetches. `query` uses `alias: 'q'` so the URL stays short while the class property stays
readable.

## The detail page

```ts title="src/app/issues/issue-detail.ts"
import { Component, inject, input, numberAttribute, signal } from '@angular/core';
import { httpResource } from '@angular/common/http';
import { DatePipe } from '@angular/common';
import { Router, RouterLink } from '@angular/router';
import { Issue, IssueStatus } from './issue';
import { IssueApi } from './issue-api';
import { StatusBadge } from './status-badge';

@Component({
  selector: 'app-issue-detail',
  imports: [RouterLink, DatePipe, StatusBadge],
  template: `
    @if (issue.error(); as err) {
      <p role="alert">{{ err.message }}</p>
    } @else if (issue.hasValue()) {
      @let i = issue.value();
      <h2>#{{ i.id }} {{ i.title }} <app-status-badge [status]="i.status" /></h2>
      <p>{{ i.description || 'No description.' }}</p>
      <p><small>Priority {{ i.priority }} · opened {{ i.createdAt | date: 'medium' : 'UTC' }}</small></p>

      <div role="group" aria-label="Change status">
        @for (s of statuses; track s) {
          <button type="button" [disabled]="s === i.status || saving()" (click)="setStatus(s)">{{ s }}</button>
        }
      </div>
      @if (actionError()) { <p role="alert">{{ actionError() }}</p> }
      <button type="button" class="danger" [disabled]="saving()" (click)="remove()">Delete</button>
    } @else {
      <p>Loading…</p>
    }
    <p><a routerLink="/issues">← All issues</a></p>
  `,
})
export class IssueDetail {
  readonly id = input.required({ transform: numberAttribute });
  private readonly api = inject(IssueApi);
  private readonly router = inject(Router);

  protected readonly statuses: IssueStatus[] = ['open', 'in-progress', 'closed'];
  protected readonly saving = signal(false);
  protected readonly actionError = signal('');
  protected readonly issue = httpResource<Issue>(() => `/api/issues/${this.id()}`);

  protected async setStatus(status: IssueStatus) {
    await this.run(async () => {
      const updated = await this.api.setStatus(this.id(), status);
      this.issue.value.set(updated); // show the server's version immediately
    });
  }

  protected async remove() {
    if (!confirm('Delete this issue?')) return;
    await this.run(async () => {
      await this.api.remove(this.id());
      await this.router.navigate(['/issues']);
    });
  }

  private async run(action: () => Promise<void>) {
    this.saving.set(true);
    this.actionError.set('');
    try {
      await action();
    } catch (e) {
      this.actionError.set((e as Error).message);
    } finally {
      this.saving.set(false);
    }
  }
}
```

After a successful PATCH, `issue.value.set(updated)` shows the server's copy immediately
(the resource's status becomes `local`) instead of refetching. The `id` input uses
`numberAttribute` because route parameters are always strings.

## The form

```ts title="src/app/issues/new-issue.ts"
import { Component, inject, signal } from '@angular/core';
import { Router, RouterLink } from '@angular/router';
import { FormField, form, maxLength, minLength, required, submit } from '@angular/forms/signals';
import { ApiError } from '../http/api-error';
import { NewIssue } from './issue';
import { IssueApi } from './issue-api';

const blank: NewIssue = { title: '', description: '', priority: 'medium' };

@Component({
  selector: 'app-new-issue',
  imports: [FormField, RouterLink],
  template: `
    <h2>New issue</h2>
    <form (submit)="save($event)" novalidate>
      <label>Title <input [formField]="f.title" /></label>
      @if (f.title().touched()) {
        @for (e of f.title().errors(); track e.kind) { <p class="err" role="alert">{{ e.message }}</p> }
      }
      <label>Description <textarea rows="4" [formField]="f.description"></textarea></label>
      @for (e of f.description().errors(); track e.kind) { <p class="err" role="alert">{{ e.message }}</p> }
      <label>Priority
        <select [formField]="f.priority">
          <option value="low">Low</option>
          <option value="medium">Medium</option>
          <option value="high">High</option>
        </select>
      </label>
      <button [disabled]="f().submitting()">{{ f().submitting() ? 'Saving…' : 'Create issue' }}</button>
      <a routerLink="/issues">Cancel</a>
    </form>
  `,
})
export class NewIssuePage {
  private readonly api = inject(IssueApi);
  private readonly router = inject(Router);
  private saved = false;

  protected readonly model = signal<NewIssue>({ ...blank });
  protected readonly f = form(this.model, (p) => {
    required(p.title, { message: 'Give the issue a title.' });
    minLength(p.title, 5, { message: 'Titles need at least 5 characters.' });
    maxLength(p.title, 120, { message: 'Keep titles under 120 characters.' });
    maxLength(p.description, 2000, { message: 'Descriptions are limited to 2000 characters.' });
  });

  /** Used by the canDeactivate guard. */
  hasUnsavedChanges() {
    return !this.saved && this.f().dirty();
  }

  protected async save(event: Event) {
    event.preventDefault();
    await submit(this.f, {
      action: async () => {
        try {
          const created = await this.api.create(this.model());
          this.saved = true;
          await this.router.navigate(['/issues', created.id]);
          return undefined;
        } catch (e) {
          if (e instanceof ApiError && e.status === 409) {
            return [{ fieldTree: this.f.title, kind: 'server', message: e.message }];
          }
          throw e;
        }
      },
    });
  }
}
```

```ts title="src/app/issues/unsaved-changes.guard.ts"
import { CanDeactivateFn } from '@angular/router';

export interface HasUnsavedChanges {
  hasUnsavedChanges(): boolean;
}

export const unsavedChangesGuard: CanDeactivateFn<HasUnsavedChanges> = (component) =>
  !component.hasUnsavedChanges() || confirm('Discard this new issue?');
```

The submit action translates a `409` into a **field error** on `title`, and rethrows
anything else so it isn't silently swallowed. `hasUnsavedChanges` returns `false` after a
successful save — otherwise the guard would ask "Discard this new issue?" while navigating
to the issue you just created.

## Wiring

```ts title="src/app/app.routes.ts"
import { Routes } from '@angular/router';
import { unsavedChangesGuard } from './issues/unsaved-changes.guard';

export const routes: Routes = [
  { path: '', pathMatch: 'full', redirectTo: 'issues' },
  { path: 'issues', title: 'Issues', loadComponent: () => import('./issues/issue-list').then((m) => m.IssueList) },
  {
    path: 'issues/new',
    title: 'New issue',
    canDeactivate: [unsavedChangesGuard],
    loadComponent: () => import('./issues/new-issue').then((m) => m.NewIssuePage),
  },
  { path: 'issues/:id', title: 'Issue', loadComponent: () => import('./issues/issue-detail').then((m) => m.IssueDetail) },
  { path: '**', redirectTo: 'issues' },
];
```

```ts title="src/app/app.config.ts"
import { ApplicationConfig, provideBrowserGlobalErrorListeners } from '@angular/core';
import { provideHttpClient, withInterceptors } from '@angular/common/http';
import { provideRouter, withComponentInputBinding } from '@angular/router';
import { routes } from './app.routes';
import { busyInterceptor } from './http/busy';
import { errorInterceptor } from './http/api-error';
import { fakeBackendInterceptor } from './http/fake-backend.interceptor';

export const appConfig: ApplicationConfig = {
  providers: [
    provideBrowserGlobalErrorListeners(),
    provideRouter(routes, withComponentInputBinding()),
    // Order: busy (outermost) → error mapping → fake backend (innermost, replaces the network).
    provideHttpClient(withInterceptors([busyInterceptor, errorInterceptor, fakeBackendInterceptor])),
  ],
};
```

```ts title="src/app/app.ts"
import { Component, inject } from '@angular/core';
import { RouterLink, RouterOutlet } from '@angular/router';
import { Busy } from './http/busy';

@Component({
  selector: 'app-root',
  imports: [RouterOutlet, RouterLink],
  template: `
    <header>
      <a routerLink="/issues"><strong>Issue Tracker</strong></a>
      @if (busy.active()) { <span class="busy" aria-live="polite">Working…</span> }
    </header>
    <main><router-outlet /></main>
  `,
})
export class App {
  protected readonly busy = inject(Busy);
}
```

## Tests

```ts title="src/app/issues/new-issue.spec.ts"
import { TestBed } from '@angular/core/testing';
import { provideHttpClient, withInterceptors } from '@angular/common/http';
import { provideRouter, Router } from '@angular/router';
import { errorInterceptor } from '../http/api-error';
import { fakeBackendInterceptor, resetFakeDb } from '../http/fake-backend.interceptor';
import { NewIssuePage } from './new-issue';

describe('NewIssuePage', () => {
  beforeEach(() => {
    resetFakeDb([
      { id: 1, title: 'Existing issue', description: '', status: 'open', priority: 'low', createdAt: '2026-09-01T00:00:00Z' },
    ]);
    TestBed.configureTestingModule({
      providers: [
        provideRouter([]),
        provideHttpClient(withInterceptors([errorInterceptor, fakeBackendInterceptor])),
      ],
    });
  });

  function fillTitle(el: HTMLElement, value: string) {
    const input = el.querySelector('input')!;
    input.value = value;
    input.dispatchEvent(new Event('input'));
  }

  it('shows the server error when the title is taken', async () => {
    const fixture = TestBed.createComponent(NewIssuePage);
    const el: HTMLElement = fixture.nativeElement;
    await fixture.whenStable();

    fillTitle(el, 'existing ISSUE');
    el.querySelector('form')!.dispatchEvent(new Event('submit'));
    await fixture.whenStable(); // waits for the (fake) HTTP call — no request is left pending

    expect(el.querySelector('.err')?.textContent).toContain('already exists');
    expect(fixture.componentInstance.hasUnsavedChanges()).toBe(true);
  });

  it('creates the issue and navigates to it', async () => {
    const router = TestBed.inject(Router);
    const navigate = vi.spyOn(router, 'navigate').mockResolvedValue(true);
    const fixture = TestBed.createComponent(NewIssuePage);
    const el: HTMLElement = fixture.nativeElement;
    await fixture.whenStable();

    fillTitle(el, 'A brand new bug');
    el.querySelector('form')!.dispatchEvent(new Event('submit'));
    await fixture.whenStable();

    expect(navigate).toHaveBeenCalledWith(['/issues', 2]);
    expect(fixture.componentInstance.hasUnsavedChanges()).toBe(false);
  });
});
```

Here `await fixture.whenStable()` *does* wait for the HTTP call, because the request is
handled by the fake backend inside the test rather than left pending in
`HttpTestingController` (compare lesson 06).

## Checking it works

`ng test`: both tests passed. `ng build`:

```text
Initial chunk files | Names         |  Raw size | Estimated transfer size
main-EBRFL3LB.js    | main          | 286.50 kB |                79.54 kB
styles-5INURTSO.css | styles        |   0 bytes |                 0 bytes
                    | Initial total | 286.50 kB |                79.54 kB
Lazy chunk files    | Names         |  Raw size | Estimated transfer size
chunk-TCP2IgGb.js   | new-issue     |  62.52 kB |                15.88 kB
chunk-BJ5wj82Y.js   | issue-detail  |   2.69 kB |                 1.23 kB
chunk-CeHieItW.js   | issue-list    |   2.64 kB |                 1.23 kB
```

The signal forms code is only needed by the "New issue" page, so it lands in that lazy
chunk rather than the initial bundle.

We then served the production build and drove it with Playwright in headless Chromium
(dialogs: "Delete this issue?" accepted, "Discard this new issue?" dismissed):

```text
1  /issues  title "Issues"   #3 Typo in welcome email (Closed), #2 Export to CSV… (In progress), #1 Login button… (Open)
2  /issues?q=export          #2 Export to CSV drops the last row
3  /issues?status=closed     #3 Typo in welcome email     (select shows "closed")
4  empty submit              ["Give the issue a title."]
5  title "Typo"              ["Titles need at least 5 characters."]
6  Cancel with dirty form    dialog "Discard this new issue?" → stayed on /issues/new
7  duplicate title           ["An issue with this title already exists."]  still on /issues/new
8  while saving              button "Saving…", header busy indicator shown
9  created                   /issues/4 "#4 Search ignores accents Open"
10 status → in-progress      badge "In progress"
11 delete                    back on /issues with the original three issues
12 /issues/99                "Issue 99 not found"
console errors: none
```

## How It Actually Works

Trace step 7 — creating an issue whose title already exists:

1. `submit(this.f, ...)` marks every field touched; the form is valid, so it sets
   `submitting()` to `true` and calls the action. The button re-renders as "Saving…".
2. `api.create()` calls `HttpClient.post`. The request passes through `busyInterceptor`
   (count → 1, header shows "Working…"), then `errorInterceptor`, then
   `fakeBackendInterceptor`, which never calls `next` — it *is* the backend.
3. After 300 ms the backend emits an `HttpErrorResponse` with status 409.
   `errorInterceptor` converts it to `ApiError(409, 'An issue with this title already exists.')`;
   `finalize` in `busyInterceptor` decrements the count.
4. `firstValueFrom` rejects with the `ApiError`; the action catches it and returns a
   field-targeted error. Signal forms attach it to `title`, `submitting()` goes back to
   `false`, and the template's `@for` over `f.title().errors()` renders it.
5. When the user edits the title, the server error is cleared automatically (lesson 02).

No component subscribed to anything, and nothing had to be unsubscribed. The data
flowed through interceptors as observables and reached the UI as signals.

## Extensions

1. **Debounced search.** Update `?q=` as the user types, 300 ms after the last keystroke,
   using `replaceUrl: true` so every keystroke doesn't add a history entry.
2. **Keep the list visible while filtering.** The list flashes "Loading…" on each filter
   change because a resource clears its value when params change. Keep the previous
   value with `linkedSignal` (Level 3, lesson 03).
3. **Edit page.** Reuse the form for editing: load the issue with a resolver, initialise
   the model from it, and PATCH on submit.
4. **Optimistic delete.** Remove the issue from the list immediately and restore it if
   the DELETE fails.
5. **Real backend.** Remove `fakeBackendInterceptor` and point the app at a small JSON
   server of your choice; nothing else should need to change.

## Exercise

Answer in a few sentences each:

1. Why do the list filters live in query parameters rather than in signals inside
   `IssueList`?
2. Why is `errorInterceptor` placed before (outside) `fakeBackendInterceptor` in the
   array? What would the form see if you swapped them?
3. Why does `NewIssuePage` set `saved = true` before navigating, and what bug would
   appear without it?
4. The detail page sets `issue.value` after a PATCH instead of calling `reload()`. Give
   one advantage and one risk of that choice.
