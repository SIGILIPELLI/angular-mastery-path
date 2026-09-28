---
description: "Angular Router in depth: child routes and layouts, functional canActivate/canMatch/canDeactivate guards, resolvers, RedirectCommand, redirect functions and route providers."
---

# 07 · Router in Depth: Guards, Resolvers, Children

Level 1 mapped URLs to components. Real apps also need to decide **whether** a route may
be entered or left, **which** of two configurations applies to the same URL, **what data**
must be ready before a page renders, and **how** to group routes under shared layouts and
services. This lesson covers the router features for each, using functional APIs — plain
functions that call `inject()` — which replaced the older class-based guards.

Every behaviour quoted below was observed with `RouterTestingHarness` on Angular 22.2.

## The route table we'll build

```ts title="src/app/app.routes.ts"
export const routes: Routes = [
  { path: 'login', component: Login },
  { path: 'not-found', component: NotFound },
  {
    path: 'shop',
    component: ShopLayout,
    children: [
      { path: 'p/:id', component: ProductPage, resolve: { product: productResolver } },
    ],
  },
  { path: 'dashboard', canMatch: [adminMatch], providers: [AdminApi], component: AdminHome },
  { path: 'dashboard', canActivate: [authGuard], component: UserArea },
  { path: 'edit', component: Editor, canDeactivate: [unsavedChangesGuard] },
  { path: 'old-product/:id', redirectTo: ({ params }) => `/shop/p/${params['id']}` },
];
```

## Child routes and layouts

`children` nests routes under a parent. The parent component renders a
`<router-outlet />` of its own, and the child's component appears inside it:

```ts
@Component({
  selector: 'app-shop-layout',
  imports: [RouterOutlet],
  template: `[shop <router-outlet />]`,
})
export class ShopLayout {}
```

Navigating to `/shop/p/42` rendered `[shop product Product 42]` — the layout wrapped the
product page. Use this for sections that share navigation, a sidebar or a toolbar. A
parent route with no `component` (a "componentless" route) groups children for shared
guards, resolvers or providers without adding a layout.

For big features, `loadChildren: () => import('./admin/admin.routes').then((m) => m.ADMIN_ROUTES)`
lazily loads a whole child route array — one chunk per feature.

## `canActivate`: may we enter?

```ts title="src/app/auth.guard.ts"
import { inject } from '@angular/core';
import { CanActivateFn, Router } from '@angular/router';

export const authGuard: CanActivateFn = (_route, state) => {
  const auth = inject(Auth);
  const router = inject(Router);
  return auth.user()
    ? true
    : router.createUrlTree(['/login'], { queryParams: { returnUrl: state.url } });
};
```

A guard returns `true` (continue), `false` (cancel), or a **`UrlTree`** / **`RedirectCommand`**
(cancel and go there instead). It may also return a Promise or Observable of those. Logged
out, `/dashboard` ended at:

```text
/login?returnUrl=%2Fdashboard
```

Return a `UrlTree` rather than calling `router.navigate()` inside the guard — the router
then treats it as one redirect, not a cancelled navigation followed by a new one.

!!! warning "Guards are UX, not security"
    Guards run in the user's browser and the user can change that code. They decide what
    the UI shows; the **server** must enforce who may read or change data.

## `canMatch`: which route applies?

`canActivate` runs *after* a route has been matched. `canMatch` runs *during* matching:
returning `false` makes the router **skip that route and try the next one**. That lets two
routes share a path:

```ts
const adminMatch: CanMatchFn = () => inject(Auth).user()?.role === 'admin';

{ path: 'dashboard', canMatch: [adminMatch], providers: [AdminApi], component: AdminHome },
{ path: 'dashboard', canActivate: [authGuard], component: UserArea },
```

A regular user got `user area`; an admin got `admin`. Because a failed `canMatch` means
the route (and a lazy route's code) is never loaded, it's also the right way to keep
admin-only chunks from being downloaded by everyone.

We also hit a subtle behaviour: after changing the signed-in user to an admin *while
already on* `/dashboard`, navigating to `/dashboard` again still showed `user area`. The
router **ignores navigation to the current URL by default**, so guards didn't re-run.
Navigating away and back showed `admin`. If your app changes identity in place, navigate
explicitly (for example to `/`) or configure
`withRouterConfig({ onSameUrlNavigation: 'reload' })` and make your guards
`runGuardsAndResolvers: 'always'` where it matters.

## Route-level providers

`providers` on a route create an **environment injector** for that route and its
children. `AdminApi` above was injected by `AdminHome`, but injecting it from the root
failed:

```text
NG0201: No provider found for `_AdminApi`
```

Use this to scope feature services to a feature — especially lazy ones, whose services
then live in the lazy chunk.

## Resolvers: data before render

A resolver loads data the page needs and makes navigation **wait** for it:

```ts title="src/app/product.resolver.ts"
export const productResolver: ResolveFn<Product> = (route) => {
  const id = route.paramMap.get('id')!;
  if (id === 'missing') {
    return new RedirectCommand(inject(Router).parseUrl('/not-found'));
  }
  return inject(ProductApi).get(id); // Observable or Promise
};
```

With `withComponentInputBinding()`, the resolved value arrives as an **input** named after
its key:

```ts
export class ProductPage {
  readonly product = input.required<Product>();
}
```

`/shop/p/missing` ended at `/not-found`: returning a `RedirectCommand` from a resolver
redirects the navigation.

**Resolver or resource?** A resolver blocks navigation — the old page stays until the data
arrives, which feels slow on a slow network unless you show a global progress bar.
Loading in the component (with `httpResource`, lesson 06) navigates instantly and shows a
skeleton. Prefer resources for most pages; reach for resolvers when rendering without the
data makes no sense, or when a missing record should redirect before anything renders.

## `canDeactivate`: may we leave?

```ts
export interface HasUnsavedChanges { hasUnsavedChanges(): boolean }

export const unsavedChangesGuard: CanDeactivateFn<HasUnsavedChanges> = (component) =>
  !component.hasUnsavedChanges() || confirm('Discard unsaved changes?');
```

With the editor dirty and the answer "no", `router.navigateByUrl('/login')` resolved to
`false` and the URL stayed `/edit`; with "yes", it resolved `true` and moved to `/login`.
It doesn't protect against closing the tab — add a `beforeunload` listener for that.

## Redirect functions

`redirectTo` can be a function that receives the matched route's params and query params
and returns a string or `UrlTree`, which makes renaming URLs painless:

```ts
{ path: 'old-product/:id', redirectTo: ({ params }) => `/shop/p/${params['id']}` }
```

`/old-product/7` became `/shop/p/7`. The function runs in an injection context, so it can
`inject()` services too.

## How It Actually Works

The navigation pipeline from Level 1 lesson 08, with the new pieces slotted in:

1. **Match** — for each candidate route, depth-first and in order: evaluate `redirectTo`
   (strings or functions), run **`canMatch`** guards, and if they pass, load lazy
   `loadChildren`/`loadComponent` code. A failing `canMatch` backtracks to the next
   candidate.
2. **Guards** — compute which routes are being deactivated and activated. Run
   **`canDeactivate`** for each route being left (innermost first), then **`canActivate`**
   and `canActivateChild` for each being entered (outermost first). The first
   `false`/`UrlTree` wins.
3. **Resolve** — run resolvers for activated routes (in parallel within a level) and
   put results into route `data`.
4. **Activate** — create/reuse components; with input binding, copy params, query params
   and data into inputs.

Guards and resolvers are called with `runInInjectionContext` using the **route's**
environment injector — which is why `inject()` works inside them and why route-level
`providers` are visible to that route's guards. Returning a `UrlTree` or
`RedirectCommand` from any guard or resolver cancels the current navigation and schedules
a new one to that URL, preserving "this was a redirect" semantics for the browser history.

## Common mistakes

- **Treating guards as security.** Enforce authorisation on the server.
- **Using `canActivate` to hide lazy code.** The chunk is already downloaded by then; use
  `canMatch`.
- **Calling `router.navigate` inside a guard** instead of returning a `UrlTree`.
- **Heavy resolvers** that make every click feel stuck. Prefer component-level loading.
- **Expecting a same-URL navigation to re-run guards.** It's ignored by default.

## Exercise

1. Add an `/account` section with a layout component and three child routes (`profile`,
   `security`, `billing`), all protected by one `canActivate` on the parent.
2. Make `billing` available only to users with `plan === 'pro'` using `canMatch`, with a
   fallback `billing` route that shows an upgrade page.
3. Add a resolver for `profile` that redirects to `/not-found` when the user record is
   missing, and receive the data via a component input.
4. Add `canDeactivate` to `security` when a password field is dirty.
5. Write a `RouterTestingHarness` test for each rule.
