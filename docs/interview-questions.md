---
title: "Angular Interview Questions and Answers (179 Questions)"
description: "179 Angular interview questions with answers and explanations, from beginner to advanced, organised by level. Free, from the Angular Mastery Path."
---

# Angular Interview Questions and Answers

**179 questions** with the correct answer and a short explanation, organised from beginner (Level 1) to advanced (Level 4). Every question comes from a lesson in the free Angular Mastery Path, so you can open the lesson to go deeper.

!!! tip "Practise, don't just read"
    Each lesson also ends with an interactive quiz, and the [final quiz](quiz.md) tests all four levels. Try to answer each question below before reading the answer.

## Level 1 · Entry

### [01 · What Angular Is](level-1/01-what-angular-is.md)

**Q1.** What does 'framework' mean for Angular versus a library like React?

**Answer:** Angular ships routing, forms, HTTP, testing and tooling as one coordinated, versioned set

Fewer decisions and a consistent structure, at the cost of learning Angular's way.

**Q2.** How does Angular relate to AngularJS?

**Answer:** Angular (2+) was a rewrite, not an upgrade; AngularJS 1.x reached end of support at the end of 2021

$scope and ng-controller are AngularJS.

**Q3.** What does ng update do?

**Answer:** Upgrades all @angular/* packages together and runs code migrations

All @angular/* packages should be on the same version.

**Q4.** Which package provides the router?

**Answer:** @angular/router

Forms are in @angular/forms and HttpClient in @angular/common/http.

### [02 · The CLI &amp; Project Structure](level-1/02-cli-project-structure.md)

**Q5.** Why does almost nobody build Angular apps without the CLI?

**Answer:** The CLI owns build config so ng update can later migrate it

It generates, builds, tests and upgrades.

**Q6.** How do you create a project without a global install?

**Answer:** npx @angular/cli@22 new reading-list

Add --defaults to skip prompts.

**Q7.** Which test runner and zone behaviour are defaults in v22 new apps?

**Answer:** vitest, and zoneless (no zone.js)

Per the flag table.

**Q8.** Inside a project, which CLI version runs?

**Answer:** The project's local copy in node_modules

Each project uses the version it was built with.

### [03 · Components &amp; Templates](level-1/03-components-templates.md)

**Q9.** What is an Angular component?

**Answer:** A class holding state and behaviour, a template describing the DOM, optional styles and a selector

It's the unit apps are built from.

**Q10.** What does the selector in @Component do?

**Answer:** Says where the component appears, e.g. &lt;app-profile-card&gt;

Used as an element in other templates.

**Q11.** Which template features does this lesson cover?

**Answer:** Interpolation and property, attribute, class, style and event bindings (but not control flow or inputs/outputs)

Control flow is lesson 5 and inputs/outputs lesson 6.

**Q12.** What holds a component's reactive state in modern Angular?

**Answer:** Signals

signal(), computed(), effect().

### [04 · Signals: signal, computed, effect](level-1/04-signals-basics.md)

**Q13.** What is a signal?

**Answer:** A value wrapped in a function that tells Angular when it's read and when it changes

Angular then knows exactly who depends on it.

**Q14.** After `list().push('b')` on a signal of an array, what does a computed length show?

**Answer:** Stale: still the old value, since mutation notifies nothing

Signals compare with Object.is. Replace instead: list.update(l =&gt; [...l, 'c']).

**Q15.** How do you expose a read-only view of a writable signal?

**Answer:** signal.asReadonly()

It has no set or update.

**Q16.** What do you pass computed() as dependencies?

**Answer:** Nothing: it tracks the signals read while running

price() * quantity() derives total.

### [05 · Template Control Flow](level-1/05-control-flow.md)

**Q17.** Which blocks does Angular's template control flow provide?

**Answer:** @if, @for, @switch and @let

They need no imports.

**Q18.** What happened to *ngIf and *ngFor?

**Answer:** Deprecated in Angular 20 in favour of the blocks; a CLI schematic migrates them

ng generate @angular/core:control-flow.

**Q19.** What does `track order.id` in @for do?

**Answer:** Identifies items so Angular can reuse DOM across changes

It's required in @for.

**Q20.** What is @empty for?

**Answer:** Content shown when the @for collection has no items

Also $index and $last are available.

### [06 · Inputs, Outputs &amp; model()](level-1/06-inputs-outputs-model.md)

**Q21.** What do inputs, outputs and models do?

**Answer:** Inputs carry data down, outputs carry events up, models are both (two-way binding)

Declared with input(), output() and model().

**Q22.** What does input() return?

**Answer:** A read-only signal you read with this.max()

It can be used inside computed().

**Q23.** What does input.required do?

**Answer:** The parent must supply it, and there's no default

Optional input(default) infers the type from the default.

**Q24.** How do you bind a model two-way from a parent?

**Answer:** [(value)]="qty" with a writable signal

Transforms like numberAttribute convert string attributes.

### [07 · Services &amp; Dependency Injection](level-1/07-services-di.md)

**Q25.** What belongs in services?

**Answer:** Fetching data, shared state, localStorage access, formatting, logging

Components should be about presentation.

**Q26.** How do you get a service into a component?

**Answer:** inject(Logger)

Angular creates one instance and shares it.

**Q27.** What did Angular 22 introduce?

**Answer:** The @Service() decorator, which ng generate service now produces

@Injectable still works.

**Q28.** How many instances do consumers get by default?

**Answer:** The same single instance for the app's lifetime

DI hands out one shared instance.

### [08 · Routing Basics](level-1/08-routing-basics.md)

**Q29.** Why does `path: ''` with redirectTo need pathMatch: 'full'?

**Answer:** Otherwise every URL starts with the empty string and matches

Redirect only the exact empty path.

**Q30.** What does loadComponent do?

**Answer:** Lazy-loads the component's code as its own chunk, downloaded on first visit

`component` is part of the main bundle.

**Q31.** Where must the '**' wildcard route go?

**Answer:** Last: the router takes the first match in order

It matches anything.

**Q32.** What does withComponentInputBinding() enable?

**Answer:** Route params bound directly to component inputs

Registered with provideRouter.

### [09 · HTTP with HttpClient](level-1/09-httpclient.md)

**Q33.** How do you enable HttpClient?

**Answer:** provideHttpClient() in the application providers

In Angular 22 it uses the fetch API by default.

**Q34.** What is true about withFetch() in Angular 22?

**Answer:** Deprecated: fetch is already the default backend; withXhr() opts back into XHR for upload progress

fetch can't report upload progress.

**Q35.** Where should API URLs be built?

**Answer:** In a service behind typed methods, not in components

The service returns data in the app's shape.

**Q36.** What do you use to add query params safely?

**Answer:** HttpParams

It handles encoding.

### [10 · Project — A Reading List App](level-1/10-project-reading-list.md)

**Q37.** Where does the Reading List keep its data?

**Answer:** localStorage, written by an effect in the BookStore

It survives a reload.

**Q38.** Why does rating use 0 for 'not rated' instead of undefined?

**Answer:** It keeps the localStorage JSON simple and avoids optional chaining everywhere

A model design choice.

**Q39.** What is the store's rule?

**Answer:** Components read signals and call methods; only the store writes

State sits in one service with an asReadonly view.

**Q40.** Which pieces does the project exercise?

**Answer:** CLI, templates, signals, control flow, inputs/outputs/model, services, routing and HttpClient (Open Library search)

Each lesson maps to a part.

## Level 2 · Intermediate

### [01 · Reactive Forms](level-2/01-reactive-forms.md)

**Q41.** What are Angular's three form systems?

**Answer:** Template-driven, reactive and signal forms

Signal forms became stable in Angular 22.

**Q42.** Where does the model live in reactive forms?

**Answer:** In a tree of FormControl/FormGroup objects in the class

Template-driven keeps it in the template via ngModel.

**Q43.** What does a validator return when valid?

**Answer:** null

Otherwise an error object like { forbidden: { word } }.

**Q44.** What is a cross-field validator for?

**Answer:** A group validator such as passwordsMatch comparing password and confirm

It sets { passwordsMismatch: true }.

### [02 · Signal Forms](level-2/02-signal-forms.md)

**Q45.** What problem do signal forms address?

**Answer:** Reactive forms keep a second copy of your data inside a control tree; signal forms use a signal model as the source of truth

They're in @angular/forms/signals.

**Q46.** How do you create one?

**Answer:** form() over a model signal, with schema validators

Stable in v22.

**Q47.** Should new code use signal forms?

**Answer:** Yes, they're the recommended approach for new Angular code, while reactive forms remain common in existing code

Both are worth knowing.

**Q48.** What do validators attach to in signal forms?

**Answer:** A schema over the model's fields

Validation errors are reactive signals.

### [03 · Dependency Injection in Depth](level-2/03-di-in-depth.md)

**Q49.** When do you need an InjectionToken?

**Answer:** For anything that isn't a class: a config object, function or string

Identity is the object; the string is just a description.

**Q50.** What are the four provider recipes?

**Answer:** useValue, useClass, useFactory, useExisting

useExisting is an alias to the same instance.

**Q51.** What does a token with a `factory` default give you?

**Answer:** A tree-shakable token that works without any provider

FEATURE_FLAGS applied without a provider.

**Q52.** Why are abstract classes good tokens?

**Answer:** They work as a TypeScript type and a runtime value for 'some implementation of this contract'

Interfaces vanish at runtime.

### [04 · RxJS for Angular Developers](level-2/04-rxjs-essentials.md)

**Q53.** What does a cold observable do on each subscription?

**Answer:** Runs its recipe again: two subscriptions mean two executions (e.g. two HTTP requests)

HttpClient observables are cold.

**Q54.** What does a plain Subject do with values emitted before you subscribe?

**Answer:** They're gone; only later values arrive

A BehaviorSubject replays its current value.

**Q55.** What do debounceTime and distinctUntilChanged do?

**Answer:** Emit only after the source is quiet; drop a value equal to the previous one

Typical for search boxes.

**Q56.** What do signals versus RxJS handle?

**Answer:** Signals: state that exists now; RxJS: events over time with operators

Prefer signals for state, Subjects for event streams.

### [05 · Signals and RxJS Interop](level-2/05-signals-rxjs-interop.md)

**Q57.** What does toSignal(obs$) do?

**Answer:** Turns an observable into a signal, e.g. to show an HTTP or router stream in a template

Provide initialValue or requireSync.

**Q58.** What does toObservable(sig) do?

**Answer:** Feeds signal changes into operators like debounceTime or switchMap

Signal → Observable.

**Q59.** What does takeUntilDestroyed() do?

**Answer:** Ends a manual subscription when the component is destroyed

It avoids leaks.

**Q60.** Why use both worlds?

**Answer:** Signals for state you read; observables for events and async workflows (debounce, cancel, retry)

The interop package bridges them.

### [06 · Async Data with resource &amp; httpResource](level-2/06-resource-httpresource.md)

**Q61.** What do the resource APIs package?

**Answer:** Loading/error/data signals plus cancelling stale requests when inputs change

You describe what to load as a function of signals.

**Q62.** What does httpResource(() =&gt; `/api/users/${this.id()}`) do when id changes?

**Answer:** Computes the new URL and fetches it, cancelling an in-flight request for the previous id

Status signals stay correct automatically.

**Q63.** What do resource(), rxResource() and httpResource() load from?

**Answer:** A Promise, an Observable, and an HttpClient GET respectively

All are stable in Angular 22.

**Q64.** How do you retry a failed resource?

**Answer:** resource.reload()

The lesson's 'Try again' button.

### [07 · Router in Depth: Guards, Resolvers, Children](level-2/07-router-in-depth.md)

**Q65.** What can a canActivate guard return?

**Answer:** true, false, or a UrlTree to redirect (e.g. to /login with returnUrl)

Guards are functional and use inject().

**Q66.** What is the difference between canActivate and canMatch?

**Answer:** canActivate decides whether you may enter a matched route; canMatch decides which of two configurations for the same URL applies

Two 'dashboard' routes differ by canMatch.

**Q67.** What does canDeactivate protect against?

**Answer:** Leaving a route with unsaved changes

e.g. unsavedChangesGuard.

**Q68.** What is a componentless parent route for?

**Answer:** Grouping children for shared guards, resolvers or providers without a layout

loadChildren lazily loads a whole child route array.

### [08 · HTTP Interceptors &amp; Error Handling](level-2/08-interceptors-errors.md)

**Q69.** What is an interceptor?

**Answer:** A function between HttpClient and the network that sees every request and response

HttpInterceptorFn(req, next).

**Q70.** Why use req.clone()?

**Answer:** Requests are immutable; clone makes a modified copy (e.g. setHeaders)

Used to add the Authorization header.

**Q71.** Why check the URL before adding a bearer token?

**Answer:** Otherwise the token would also be sent to third-party URLs

Only attach tokens to your own API origin.

**Q72.** What is HttpContext for?

**Answer:** Per-request metadata interceptors read but never sent to the server (e.g. SKIP_AUTH)

It enables opt-outs.

### [09 · Pipes &amp; Directives](level-2/09-pipes-directives.md)

**Q73.** What do pipes and attribute directives do?

**Answer:** Pipes transform a value for display; attribute directives add behaviour or styling to an existing element

Both are standalone and imported into a component.

**Q74.** What does `{{ price | currency: 'EUR' }}` give for 1234.5 in en-US?

**Answer:** €1,234.50

Shown in the lesson's table.

**Q75.** What does `{{ 0.4567 | percent: '1.1-1' }}` give?

**Answer:** 45.7%

Digit info is minInt.minFrac-maxFrac.

**Q76.** What does `['a','b','c','d'] | slice: 1 : 3` give?

**Answer:** b,c

Start 1, end 3 exclusive.

### [10 · Project — An Issue Tracker](level-2/10-project-issue-tracker.md)

**Q77.** How does the project run without a backend?

**Answer:** An interceptor implements a small in-memory REST API with 300 ms latency, removable with one line in app.config.ts

Latency makes loading states visible.

**Q78.** Where are the search and status filters kept?

**Answer:** In the URL (query params bound as inputs)

URL state is shareable.

**Q79.** How is the Busy service used?

**Answer:** Shared by an interceptor that counts requests in flight and the header showing progress

finalize decrements the count.

**Q80.** What protects unsaved issue edits when navigating away?

**Answer:** A canDeactivate guard

From the router lesson.

## Level 3 · Advanced

### [01 · Change Detection &amp; Zoneless](level-3/01-change-detection-zoneless.md)

**Q81.** What are the Angular 22 change-detection defaults?

**Answer:** Zoneless (no Zone.js) and OnPush by default; the old always-check mode is ChangeDetectionStrategy.Eager

`Default` remains a deprecated alias for Eager.

**Q82.** When does zoneless Angular schedule a pass?

**Answer:** When told something changed: a signal read in a template updates, a bound listener fires, markForCheck, setInput, view attach/remove

Plain field assignments aren't on the list.

**Q83.** When is an OnPush component refreshed?

**Answer:** When marked dirty: a signal it reads changes, an input gets a new value (Object.is), a listener fires, or markForCheck is called

markForCheck also marks ancestors.

**Q84.** Does mutating a field in place update a zoneless OnPush view?

**Answer:** No: plain assignments don't notify; use a signal

Observed in the lesson's lab.

### [02 · Composition: Projection &amp; Queries](level-3/02-content-projection-queries.md)

**Q85.** What are the three composition tools?

**Answer:** Content projection (&lt;ng-content&gt;), template outlets and queries

Inputs pass data; these pass markup.

**Q86.** What do viewChild and contentChild differ by?

**Answer:** View queries look in the component's own template; content queries look in projected content

Both return signals.

**Q87.** What does a template outlet let a caller do?

**Answer:** Pass an &lt;ng-template&gt; the component stamps out with data as often as it likes

For example custom row layouts.

**Q88.** What is a fallback in &lt;ng-content&gt;?

**Answer:** Default content shown if no content is projected into the slot

Slots can be named by selector.

### [03 · State Management with Signals](level-3/03-state-management.md)

**Q89.** What are the two rules that prevent most state bugs?

**Answer:** One owner per piece of state, and derive don't copy (use computed)

Copying into another signal via an effect creates stale-value bugs.

**Q90.** Where should derived state like a filtered list live?

**Answer:** In a computed(), never stored separately

Derived values stay in sync automatically.

**Q91.** What is the service store pattern?

**Answer:** Private writable signal, public read-only views and computeds, and methods as the only way to change state

Every change goes through testable code.

**Q92.** What is linkedSignal for?

**Answer:** A value that starts derived but is writable, and resets when its source changes (like a selected option)

For example selected resets when options changes.

### [04 · Deferrable Views](level-3/04-deferrable-views.md)

**Q93.** What does @defer split by?

**Answer:** Part of a template: the block's component code becomes a separate chunk downloaded when a trigger fires

Route lazy loading splits by page.

**Q94.** In the lesson, what did `prefetch on hover(showBtn)` do?

**Answer:** Fetched the chart's chunk on hover, before the click, so clicking rendered immediately

The @loading block never had to appear.

**Q95.** When was the comments chunk requested for `@defer (on viewport)`?

**Answer:** When its placeholder scrolled into view

Observed by recording JS requests.

**Q96.** What are the companion blocks of @defer?

**Answer:** @placeholder, @loading and @error

@loading supports after and minimum timings.

### [05 · Performance](level-3/05-performance.md)

**Q97.** What are the two halves of performance work?

**Answer:** Loading and runtime

Measure your own app first.

**Q98.** How do you see what's inside a bundle chunk?

**Answer:** ng build --stats-json, then analyse the esbuild metafile

It writes browser-stats.json.

**Q99.** What do budgets do?

**Answer:** Enforce size limits so builds fail or warn when bundles grow

Part of the checklist.

**Q100.** Where do the biggest wins come from?

**Answer:** Measuring your own app and fixing what the measurements point at

Angular's defaults are already good.

### [06 · SSR, Prerendering &amp; Hydration](level-3/06-ssr-hydration.md)

**Q101.** What does SSR solve for public pages?

**Answer:** A blank screen until JS runs, and crawlers/link previews seeing little

Hydration lets the browser adopt the server HTML instead of discarding it.

**Q102.** Which render mode fits a page identical for every visitor?

**Answer:** Prerender (SSG), built once at ng build

Server renders per request for query-dependent pages; Client for localStorage-dependent ones.

**Q103.** What does `fallback: PrerenderFallback.Server` do?

**Answer:** Unknown slugs not prerendered are rendered on request

getPrerenderParams lists the known slugs.

**Q104.** How do you add SSR to an existing project?

**Answer:** ng add @angular/ssr

It adds main.server.ts, server routes and provideClientHydration().

### [07 · Testing with Vitest &amp; TestBed](level-3/07-testing.md)

**Q105.** What runs unit tests in new Angular 22 projects?

**Answer:** Vitest in jsdom through the CLI's unit-test builder (Karma still available)

Templates are type-checked in tests too.

**Q106.** Where should most tests be aimed?

**Answer:** Services and components, with fewer end-to-end tests for critical flows

E2E for flows that make money or would be embarrassing.

**Q107.** How do you drive a component test with inputs?

**Answer:** fixture.componentRef.setInput('book', book) then await fixture.whenStable()

Assert on the DOM.

**Q108.** Which tool tests guards and resolvers?

**Answer:** RouterTestingHarness

HttpTestingController is for API clients.

### [08 · Accessibility &amp; the CDK](level-3/08-accessibility-cdk.md)

**Q109.** Where does most accessibility come from?

**Answer:** Plain semantic HTML and correct labels

No library fixes a &lt;div (click)&gt; pretending to be a button.

**Q110.** What is @angular/aria?

**Answer:** Headless directives implementing WAI-ARIA patterns (tabs, menu, listbox, tree…) without styles

Separate entry points like @angular/aria/tabs.

**Q111.** What does @angular/cdk/a11y offer?

**Answer:** Lower-level tools: focus trapping, live announcements, focus monitoring and keyboard list navigation

Three layers: templates, aria, CDK.

**Q112.** Which element should you use for an action?

**Answer:** &lt;button&gt;, and &lt;a href&gt;/routerLink for navigation

Both are keyboard-operable for free.

### [09 · Security](level-3/09-security.md)

**Q113.** What is the core front-end security principle?

**Answer:** Never let data become code

Most attacks are XSS: attacker text executed as JavaScript.

**Q114.** What does Angular do with a hostile comment in `{{ comment }}`?

**Answer:** Shows it as text; interpolation never interprets HTML

The lesson's &lt;p&gt; contained escaped &amp;lt;b&amp;gt;...

**Q115.** What happens to `[href]` bound to `javascript:...`?

**Answer:** Angular sanitises it to unsafe:javascript:..., so it doesn't run

pwned3 stayed undefined.

**Q116.** What does bypassSecurityTrustHtml with user input risk?

**Answer:** It disables sanitisation, so the onerror payload ran (pwned4 was true)

Never use it on user input.

### [10 · Project — A Server-Rendered Product Catalog](level-3/10-project-catalog.md)

**Q117.** Which rendering strategy fits each page in the 'Desk &amp; Lamp' catalog?

**Answer:** Prerender for product pages, per-request server render for search, client-only for the cart

The right strategy per page.

**Q118.** Why is the cart client-only?

**Answer:** It depends on browser-only storage, so nothing useful renders on the server

Browser-only storage must be guarded for SSR.

**Q119.** What does `hydrate on viewport` do for related products?

**Answer:** Defers hydrating that part until it scrolls into view

@defer with hydration triggers.

**Q120.** What does the Cart service store provide?

**Answer:** Computed totals over a private signal with methods as the only mutation path

From the state lesson.

## Level 4 · Master

### [01 · The Build System &amp; Configurations](level-4/01-build-system.md)

**Q121.** Which builders compile and serve Angular apps?

**Answer:** @angular/build:application (esbuild + Angular compiler) and @angular/build:dev-server (Vite-based)

Most production concerns are options on these.

**Q122.** Where do build options live?

**Answer:** angular.json under projects → architect targets, with options and per-configuration overrides

production is the default configuration for ng build.

**Q123.** What do configurations like production do?

**Answer:** Override options for a specific build mode on top of shared options

Environments, budgets, deployment paths and i18n are build options.

**Q124.** Why learn where options live?

**Answer:** It saves hours when handling environments, deployment paths, i18n, SSR, CSP and budgets

They're ultimately builder options.

### [02 · Libraries &amp; Monorepos](level-4/02-libraries-monorepos.md)

**Q125.** What does a library solve?

**Answer:** The same code written twice across apps: packaged with a deliberate public API that several apps consume

The CLI supports multi-project workspaces and ng-packagr.

**Q126.** How do you create a workspace with a library and an app?

**Answer:** ng new platform --no-create-application, then ng generate library ui-kit and ng generate application shop

Both live under projects/ and share node_modules and angular.json.

**Q127.** What builder builds libraries?

**Answer:** @angular/build:ng-packagr

Built with ng build ui-kit.

**Q128.** What defines what a library exposes?

**Answer:** Its public-api.ts

Keep the public API deliberate.

### [03 · Internationalization](level-4/03-i18n.md)

**Q129.** How does @angular/localize translate?

**Answer:** At build time: mark text, extract to translation files, and the build produces one translated app per locale

No runtime lookup and no flash of untranslated text.

**Q130.** Why add a custom ID like @@readingListHeading?

**Answer:** It keeps the ID stable when English text changes; otherwise the hash changes and the message must be re-translated

The description helps translators.

**Q131.** How do you handle plurals?

**Answer:** ICU expressions such as {count(), plural, =0 {…} =1 {…} other {…}}

Also $localize for code strings.

**Q132.** What does `ng add @angular/localize` add?

**Answer:** Types, a polyfill and the tiny runtime $localize function

It updates main.ts, tsconfigs and angular.json.

### [04 · Angular Elements &amp; Micro-Frontends](level-4/04-elements-microfrontends.md)

**Q133.** What does Angular Elements do?

**Answer:** Packages a component as a browser-native custom element usable from plain HTML and DOM APIs

Useful in a CMS, static page or another team's React app.

**Q134.** How do you bootstrap for an element?

**Answer:** An application without a root component, registering the element

The lesson turns StarRating into &lt;star-rating&gt;.

**Q135.** Which package provides it?

**Answer:** @angular/elements

npm install @angular/elements.

**Q136.** What is the larger topic discussed afterwards?

**Answer:** Micro-frontends

Custom elements are one integration approach.

### [05 · Upgrading &amp; Migrations](level-4/05-upgrading-migrations.md)

**Q137.** What upgrade policy does the lesson recommend?

**Answer:** Upgrade each major within its active support window, apply minors/patches as they come, and never skip majors (20 → 21 → 22)

Majors ship about every six months.

**Q138.** What does `ng update @angular/core@22 @angular/cli@22` do?

**Answer:** Updates packages, runs the update migrations they ship, and prints what changed

Commit first so the diff is reviewable.

**Q139.** What should you check before running it?

**Answer:** Commit everything and consult angular.dev/update-guide for manual steps

Afterwards run ng build, ng test and click through.

**Q140.** How should third-party libraries be updated?

**Answer:** With ng update &lt;package&gt; in the same step so their migrations run too

They usually publish a matching major shortly after.

### [06 · Large-App Architecture](level-4/06-large-app-architecture.md)

**Q141.** How should a large app be organised?

**Answer:** By feature, and inside a feature by role (feature, ui, data-access), not by file type

Type folders scatter each feature across the tree.

**Q142.** What are `ui/` components?

**Answer:** Presentational: inputs in, outputs out, no injected data services

feature/ components are containers; data-access/ holds data code.

**Q143.** Which dependency rule holds?

**Answer:** Features depend on shared/core; feature to feature only through the other's index.ts; shared/core never depend on features

Dependencies flow one way.

**Q144.** How do you keep the rules honest?

**Answer:** Enforce them in CI (e.g. ESLint boundaries), not a README

Rules in a README erode.

### [07 · Deployment](level-4/07-deployment.md)

**Q145.** Where do client-rendered Angular builds go?

**Answer:** Any static host or CDN (dist/&lt;app&gt;/browser)

SSR routes need a Node runtime for server/server.mjs.

**Q146.** What are the key static-hosting items?

**Answer:** The base href, SPA fallbacks for deep links, and cache headers (hashed files immutable, HTML revalidated)

Reloading an inner route must return the app.

**Q147.** How should hashed files be cached?

**Answer:** Immutable, long-lived; HTML should be revalidated

Filenames change with content.

**Q148.** What does a prerendered-only app produce?

**Answer:** Static files, one index.html per route

Host it like static files.

### [08 · Error Handling &amp; Observability](level-4/08-errors-observability.md)

**Q149.** What are the two goals of error handling?

**Answer:** Find out about errors with enough context, and keep one failure from taking down the page

Wire both up.

**Q150.** What does a custom ErrorHandler do?

**Answer:** Receives errors Angular catches so you can send them to logging/monitoring

onViewError gives the component name.

**Q151.** What does a missing-chunk navigation error usually mean?

**Answer:** A new version was deployed, so reload to get it

Handled in withNavigationErrorHandler.

**Q152.** What does @boundary do?

**Answer:** Contains errors from a risky widget so the rest of the page keeps working

Containment limits blast radius.

### [09 · Production Readiness Review](level-4/09-production-readiness.md)

**Q153.** What question does a production-readiness review ask?

**Answer:** 'What will hurt us at 2 a.m.?' rather than 'does the feature work?'

It happens before users depend on the app.

**Q154.** What does 'evidence' mean in the checklist?

**Answer:** What you can show, not just claim: e.g. curl -I headers, build output, CI pipeline

Automate what a script can check.

**Q155.** Which build checks appear?

**Answer:** Production default, failing budgets, no secrets in environments, working deep links, correct cache headers, chunk-load recovery

Each links to a lesson.

**Q156.** What should every data load have?

**Answer:** Loading, error and empty states

Resources should never read value() in an error state.

### [10 · Capstone — A Production Angular App](level-4/10-capstone.md)

**Q157.** What is the capstone's goal?

**Answer:** An app you could run for real users and sleep at night, not more features

It takes the Level 3 SSR catalog to production.

**Q158.** How is a release version carried into error reports?

**Answer:** BUILD_VERSION defined in angular.json (default 'dev') and overridden by CI with --define

A default avoids a ReferenceError when the constant is only on the command line.

**Q159.** What does the budget change do?

**Answer:** Tightens the generated 500 kB/1 MB to 340 kB/400 kB, just above the real initial size

Budgets catch regressions.

**Q160.** What about source maps?

**Answer:** Generated but hidden: not public

sourceMap.scripts true with hidden true.

## Mixed questions across all levels

**Q161.** What does this component's `count` display after two clicks?

```typescript
readonly count = signal(0);
protected inc() { this.count.update((n) => n + 1); }
```

**Answer:** 2, since update computes the next value from the current one

signal.update(fn) passes the current value.

**Q162.** A signal holds `['a']`, and `list().push('b')` mutates it in place. What does `computed(() => list().length)` return afterwards?

**Answer:** Still 1: in-place mutation doesn't notify; replace with list.update(l =&gt; [...l, 'b'])

Signals compare with Object.is. Verified with a minimal signal/computed implementation: 1, 1, then 3 after replacement.

**Q163.** price = signal(20), quantity = signal(3), total = computed(() =&gt; price() * quantity()). After quantity.set(4), what is total()?

**Answer:** 80

computed tracks the signals it reads. Verified by running: 60 then 80.

**Q164.** A computed's function reads signal `s`. You call `s.set(1)` when s is already 1. Does the computed recompute?

**Answer:** No: Object.is says nothing changed, so no notification

Verified: the function ran once across redundant sets.

**Q165.** Which template renders the list correctly?

**Answer:** @for (b of books(); track b.id) { &lt;li&gt;{{ b.title }}&lt;/li&gt; } @empty { &lt;li&gt;None&lt;/li&gt; }

@for needs track; @empty handles no items. *ngFor is deprecated since v20.

**Q166.** With routes [ {path:'', redirectTo:'books', pathMatch:'full'}, {path:'books'}, {path:'books/:id'}, {path:'**', redirectTo:'books'} ], where does /books/42 go, and what are its params?

**Answer:** The books/:id route with { id: '42' }

The router takes the first match in order. Verified with a small matcher.

**Q167.** Where must the '**' route go, and why?

**Answer:** Last: routes are checked in order and the first match wins, so it would otherwise swallow everything

The same reason `path: ''` redirects need pathMatch: 'full'.

**Q168.** A component with OnPush (default in v22) mutates a plain field inside a setTimeout. Does the view update?

**Answer:** No: zoneless Angular only updates after a notification such as a signal read in the template changing; use a signal

Plain assignments aren't on the notification list.

**Q169.** Where should a filtered list derived from `items` and `query` signals live?

**Answer:** In a computed(), never stored separately

Derive, don't copy.

**Q170.** A child needs a required numeric input. Which declaration is right?

**Answer:** readonly max = input.required({ transform: numberAttribute })

input.required has no default and the parent must supply it.

**Q171.** Which provider recipe gives the same instance a second name?

**Answer:** useExisting

An alias to another token.

**Q172.** A service needs configuration that isn't a class. What do you inject?

**Answer:** An InjectionToken&lt;AppConfig&gt; provided with useValue

Tokens are the lookup keys for non-class values.

**Q173.** A search box should wait until typing pauses and cancel stale requests. Which operators?

**Answer:** debounceTime then distinctUntilChanged and switchMap

switchMap cancels the previous inner observable.

**Q174.** Which API loads data for a component when an `id` input changes and cancels the in-flight request?

**Answer:** httpResource(() =&gt; `/api/users/${this.id()}`)

Resources keep value, error and isLoading signals correct.

**Q175.** Which guard stops a user leaving a form with unsaved changes?

**Answer:** canDeactivate

canActivate controls entry; canMatch chooses between configs.

**Q176.** User text includes &lt;script&gt;. How does Angular render it in `{{ comment }}`?

**Answer:** As escaped text: interpolation never interprets HTML

bypassSecurityTrustHtml on user input would defeat this.

**Q177.** Which render mode fits a page that depends on localStorage?

**Answer:** Client

The server has no localStorage.

**Q178.** Which command upgrades Angular packages and runs their migrations?

**Answer:** ng update @angular/core@22 @angular/cli@22 (one major at a time)

Commit first, then build and test.

**Q179.** Which structure scales for large apps?

**Answer:** Feature-first folders with feature/ui/data-access roles, dependency rules enforced in CI

Features expose a public API via index.ts.

## Practical coding questions

### A tiny signal

Implement signal(initial) with a read function plus set(v) and update(fn). It must only notify subscribers when the new value differs by Object.is. The provided computed() tracks reads. Run the checks to see why mutating an array in place leaves a computed stale.

Starter code:

```
let active = null;
export function signal(initial) {
  // return a read function with .set(v) and .update(fn); notify only when !Object.is(new, old)
}
export function computed(fn) {
  let cache, dirty = true;
  const mark = () => { dirty = true; };
  const read = () => {
    if (dirty) { const prev = active; active = mark; try { cache = fn(); } finally { active = prev; } dirty = false; }
    return cache;
  };
  return read;
}
const list = signal(['a']);
const n = computed(() => list().length);
console.log(n());
list().push('b');
console.log(n());
list.update((l) => [...l, 'c']);
console.log(n());
const price = signal(20), qty = signal(3);
const total = computed(() => price() * qty());
console.log(total()); qty.set(4); console.log(total());
const s = signal(1); let runs = 0; const d = computed(() => { runs++; return s() * 2; });
d(); d(); s.set(1); d(); console.log(runs); s.set(2); console.log(d(), runs);
```

Expected output:

```text
1
1
3
60
80
1
4 2
```

??? success "Show a solution"

    ```
    let active = null;
    export function signal(initial) {
      let value = initial; const subs = new Set();
      const read = () => { if (active) subs.add(active); return value; };
      read.set = (v) => { if (Object.is(v, value)) return; value = v; subs.forEach((s) => s()); };
      read.update = (fn) => read.set(fn(value));
      return read;
    }
    export function computed(fn) {
      let cache, dirty = true;
      const mark = () => { dirty = true; };
      const read = () => {
        if (dirty) { const prev = active; active = mark; try { cache = fn(); } finally { active = prev; } dirty = false; }
        return cache;
      };
      return read;
    }
    const list = signal(['a']);
    const n = computed(() => list().length);
    console.log(n());
    list().push('b');
    console.log(n());
    list.update((l) => [...l, 'c']);
    console.log(n());
    const price = signal(20), qty = signal(3);
    const total = computed(() => price() * qty());
    console.log(total()); qty.set(4); console.log(total());
    const s = signal(1); let runs = 0; const d = computed(() => { runs++; return s() * 2; });
    d(); d(); s.set(1); d(); console.log(runs); s.set(2); console.log(d(), runs);
    ```

    A signal notifies only through set/update and compares with Object.is, so list().push('b') is invisible while list.update(l =&gt; [...l, 'c']) is seen. computed tracks what it reads and recomputes only when notified. Verified by running the solution in Node.

### First match wins

Write match(routes, url) returning the first route that matches: '' matches only the empty URL when pathMatch is 'full', ':name' segments capture params, '**' matches anything. Print where each URL ends up.

Starter code:

```
function match(routes, url) {
  // your code
}
const routes = [
  { path: '', redirectTo: 'books', pathMatch: 'full' },
  { path: 'books' }, { path: 'books/:id' }, { path: '**', redirectTo: 'books' },
];
for (const u of ['/', '/books', '/books/42', '/nope/x']) { const m = match(routes, u); console.log(u, '->', m.redirectTo ?? m.path, JSON.stringify(m.params ?? {})); }
```

Expected output:

```text
/ -> books {}
/books -> books {}
/books/42 -> books/:id {"id":"42"}
/nope/x -> books {}
```

??? success "Show a solution"

    ```
    function match(routes, url) {
      const segs = url.split('/').filter(Boolean);
      for (const r of routes) {
        if (r.path === '**') return r;
        const p = r.path.split('/').filter(Boolean);
        if (r.path === '') { if (r.pathMatch === 'full' ? segs.length === 0 : true) return r; continue; }
        if (p.length !== segs.length) continue;
        if (p.every((x, i) => x.startsWith(':') || x === segs[i])) return { ...r, params: Object.fromEntries(p.map((x, i) => x.startsWith(':') ? [x.slice(1), segs[i]] : null).filter(Boolean)) };
      }
      return null;
    }
    const routes = [
      { path: '', redirectTo: 'books', pathMatch: 'full' },
      { path: 'books' }, { path: 'books/:id' }, { path: '**', redirectTo: 'books' },
    ];
    for (const u of ['/', '/books', '/books/42', '/nope/x']) { const m = match(routes, u); console.log(u, '->', m.redirectTo ?? m.path, JSON.stringify(m.params ?? {})); }
    ```

    The router checks routes in order and takes the first match, so the wildcard must be last and an empty-path redirect needs pathMatch: 'full'. Verified by running the solution in Node.

---

More free practice: the [Angular Mastery Path](index.md) curriculum, the [Interview Prep Pack](https://learnmasterypath.com/interview-pack/) and the [free resume ATS check](https://learnmasterypath.com/resume/).
