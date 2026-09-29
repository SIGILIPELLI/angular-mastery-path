---
description: "Level 3 project: a server-rendered Angular catalog with prerendered pages, per-request search, a client cart, hydration and tests."
---

# 10 · Project — A Server-Rendered Product Catalog

This project builds a small shop front, "Desk & Lamp", that uses the right rendering
strategy for each page and pulls together most of Level 3:

| Lesson | Where it shows up |
|--------|-------------------|
| 01 Change detection | signals everywhere; nothing relies on zone-triggered updates |
| 02 Composition | a reusable `ProductCard` |
| 03 State | a `Cart` service store with computed totals |
| 04 `@defer` | related products with `hydrate on viewport` |
| 05 Performance | per-route lazy chunks; budgets from the CLI defaults |
| 06 SSR | three render modes, a real 404, the HTTP transfer cache, browser-only storage |
| 07 Testing | component and service tests, including server-only behaviour |
| 09 Security | allowed hosts for the server engine |

Every file below was built with `ng build`, served with the generated Node server,
fetched with `curl`, driven in headless Chromium and unit-tested on Angular 22.2.

## Setup

```bash
npx @angular/cli@22 new catalog --ssr --defaults --inline-template --inline-style
cd catalog
```

Feature code lives in `src/app/catalog/`.

## Data and model

```ts title="src/app/catalog/product.ts"
export interface Product {
  slug: string;
  name: string;
  category: 'desk' | 'lighting' | 'storage';
  price: number;
  summary: string;
}
```

```ts title="src/app/catalog/products.data.ts"
import { Product } from './product';

// Shared by the Angular app (for prerendering) and the Express API in server.ts.
export const PRODUCTS: readonly Product[] = [
  { slug: 'oak-standing-desk', name: 'Oak Standing Desk', category: 'desk', price: 649, summary: 'Height-adjustable desk with a solid oak top and quiet dual motors.' },
  { slug: 'walnut-writing-desk', name: 'Walnut Writing Desk', category: 'desk', price: 420, summary: 'Compact desk with one drawer, sized for small rooms.' },
  { slug: 'arc-floor-lamp', name: 'Arc Floor Lamp', category: 'lighting', price: 189, summary: 'Overhead light for reading corners, with a dimmable warm LED.' },
  { slug: 'clamp-desk-lamp', name: 'Clamp Desk Lamp', category: 'lighting', price: 79, summary: 'Clamps to any desk edge and frees up surface space.' },
  { slug: 'cable-tray', name: 'Under-desk Cable Tray', category: 'storage', price: 35, summary: 'Keeps power strips and cables off the floor.' },
  { slug: 'stacking-shelves', name: 'Stacking Shelves', category: 'storage', price: 129, summary: 'Modular birch shelves that stack two or three high.' },
];

export function findProduct(slug: string): Product | undefined {
  return PRODUCTS.find((p) => p.slug === slug);
}
```

The product list is a plain module so it can be imported by the Angular app (for
prerendering) **and** by the Express server (for the search API) without duplication.

## Services

```ts title="src/app/catalog/cart.ts"
import { PLATFORM_ID, Service, computed, effect, inject, signal } from '@angular/core';
import { isPlatformBrowser } from '@angular/common';
import { findProduct } from './products.data';

interface Line { slug: string; qty: number }
const KEY = 'catalog-cart';

@Service()
export class Cart {
  private readonly isBrowser = isPlatformBrowser(inject(PLATFORM_ID));
  private readonly _lines = signal<Line[]>(this.isBrowser ? readStorage() : []);

  readonly lines = computed(() =>
    this._lines().flatMap((l) => {
      const product = findProduct(l.slug);
      return product ? [{ product, qty: l.qty }] : [];
    }),
  );
  readonly count = computed(() => this._lines().reduce((n, l) => n + l.qty, 0));
  readonly total = computed(() => this.lines().reduce((s, l) => s + l.product.price * l.qty, 0));

  constructor() {
    if (this.isBrowser) {
      effect(() => localStorage.setItem(KEY, JSON.stringify(this._lines())));
    }
  }

  add(slug: string) {
    this._lines.update((ls) =>
      ls.some((l) => l.slug === slug)
        ? ls.map((l) => (l.slug === slug ? { ...l, qty: l.qty + 1 } : l))
        : [...ls, { slug, qty: 1 }],
    );
  }

  remove(slug: string) {
    this._lines.update((ls) => ls.filter((l) => l.slug !== slug));
  }
}

function readStorage(): Line[] {
  try {
    return JSON.parse(localStorage.getItem(KEY) ?? '[]');
  } catch {
    return [];
  }
}
```

The cart is browser-only state: on the server it starts empty and never touches
`localStorage` (a unit test below proves it).

```ts title="src/app/catalog/seo.ts"
import { Service, inject } from '@angular/core';
import { Meta, Title } from '@angular/platform-browser';

@Service()
export class Seo {
  private readonly title = inject(Title);
  private readonly meta = inject(Meta);

  set(title: string, description: string) {
    this.title.setTitle(`${title} · Desk & Lamp`);
    this.meta.updateTag({ name: 'description', content: description });
    this.meta.updateTag({ property: 'og:title', content: title });
    this.meta.updateTag({ property: 'og:description', content: description });
  }
}
```

## Components

```ts title="src/app/catalog/product-card.ts"
import { Component, input } from '@angular/core';
import { CurrencyPipe } from '@angular/common';
import { RouterLink } from '@angular/router';
import { Product } from './product';

@Component({
  selector: 'app-product-card',
  imports: [RouterLink, CurrencyPipe],
  template: `
    <a [routerLink]="['/products', product().slug]">
      <h3>{{ product().name }}</h3>
      <p>{{ product().summary }}</p>
      <strong>{{ product().price | currency: 'USD' }}</strong>
    </a>
  `,
})
export class ProductCard {
  readonly product = input.required<Product>();
}
```

```ts title="src/app/catalog/home.ts"
import { Component, inject } from '@angular/core';
import { ProductCard } from './product-card';
import { PRODUCTS } from './products.data';
import { Seo } from './seo';

@Component({
  selector: 'app-home',
  imports: [ProductCard],
  template: `
    <h2>Furniture for focused work</h2>
    <div class="grid">
      @for (p of products; track p.slug) {
        <app-product-card [product]="p" />
      }
    </div>
  `,
})
export class Home {
  protected readonly products = PRODUCTS;

  constructor() {
    inject(Seo).set('Home', 'Desks, lamps and storage for a calm home office.');
  }
}
```

```ts title="src/app/catalog/product-page.ts"
import { Component, RESPONSE_INIT, computed, effect, inject, input } from '@angular/core';
import { CurrencyPipe } from '@angular/common';
import { RouterLink } from '@angular/router';
import { Cart } from './cart';
import { RelatedProducts } from './related-products';
import { findProduct } from './products.data';
import { Seo } from './seo';

@Component({
  selector: 'app-product-page',
  imports: [CurrencyPipe, RouterLink, RelatedProducts],
  template: `
    @if (product(); as p) {
      <h2>{{ p.name }}</h2>
      <p>{{ p.summary }}</p>
      <p class="price">{{ p.price | currency: 'USD' }}</p>
      <button type="button" (click)="cart.add(p.slug)">Add to cart</button>

      @defer (hydrate on viewport) {
        <app-related-products [slug]="p.slug" />
      }
    } @else {
      <h2>Product not found</h2>
      <p><a routerLink="/">Back to all products</a></p>
    }
  `,
})
export class ProductPage {
  readonly slug = input.required<string>();
  protected readonly cart = inject(Cart);
  protected readonly product = computed(() => findProduct(this.slug()));
  private readonly responseInit = inject(RESPONSE_INIT, { optional: true });
  private readonly seo = inject(Seo);

  constructor() {
    effect(() => {
      const p = this.product();
      if (p) {
        this.seo.set(p.name, p.summary);
      } else {
        this.seo.set('Not found', 'This product does not exist.');
        if (this.responseInit) this.responseInit.status = 404; // only set during SSR
      }
    });
  }
}
```

`@defer (hydrate on viewport)` is **incremental hydration**: the related products are
rendered into the server HTML (good for users and crawlers), but their code isn't
downloaded or hydrated until the section scrolls into view. Incremental hydration is on
by default with `provideClientHydration()` in Angular 22.

```ts title="src/app/catalog/related-products.ts"
import { Component, computed, input } from '@angular/core';
import { ProductCard } from './product-card';
import { PRODUCTS } from './products.data';

@Component({
  selector: 'app-related-products',
  imports: [ProductCard],
  template: `
    <h3>You might also like</h3>
    <div class="grid">
      @for (p of related(); track p.slug) {
        <app-product-card [product]="p" />
      }
    </div>
  `,
})
export class RelatedProducts {
  readonly slug = input.required<string>();
  protected readonly related = computed(() => {
    const current = PRODUCTS.find((p) => p.slug === this.slug());
    return PRODUCTS.filter((p) => p.category === current?.category && p.slug !== this.slug());
  });
}
```

```ts title="src/app/catalog/search.ts"
import { Component, inject, input } from '@angular/core';
import { httpResource } from '@angular/common/http';
import { Router } from '@angular/router';
import { Product } from './product';
import { ProductCard } from './product-card';
import { Seo } from './seo';

@Component({
  selector: 'app-search',
  imports: [ProductCard],
  template: `
    <h2>Search</h2>
    <form role="search" (submit)="$event.preventDefault(); go(box.value)">
      <input #box name="q" type="search" [value]="q() ?? ''" aria-label="Search products" />
      <button>Search</button>
    </form>
    @if (results.error()) {
      <p role="alert">Search is unavailable right now.</p>
    } @else if (results.hasValue()) {
      <p class="count">{{ results.value().length }} result(s) for “{{ q() }}”</p>
      <div class="grid">
        @for (p of results.value(); track p.slug) { <app-product-card [product]="p" /> }
      </div>
    }
  `,
})
export class Search {
  readonly q = input<string>();
  private readonly router = inject(Router);

  protected readonly results = httpResource<Product[]>(() =>
    this.q() ? { url: '/api/search', params: { q: this.q()! } } : undefined,
  );

  constructor() {
    inject(Seo).set('Search', 'Search the Desk & Lamp catalog.');
  }

  protected go(q: string) {
    this.router.navigate([], { queryParams: { q: q.trim() || null } });
  }
}
```

```ts title="src/app/catalog/cart-page.ts"
import { Component, inject } from '@angular/core';
import { CurrencyPipe } from '@angular/common';
import { Cart } from './cart';

@Component({
  selector: 'app-cart-page',
  imports: [CurrencyPipe],
  template: `
    <h2>Your cart</h2>
    @for (line of cart.lines(); track line.product.slug) {
      <p>
        {{ line.product.name }} × {{ line.qty }} — {{ line.product.price * line.qty | currency: 'USD' }}
        <button type="button" (click)="cart.remove(line.product.slug)">Remove</button>
      </p>
    } @empty {
      <p>Your cart is empty.</p>
    }
    <p class="total">Total: {{ cart.total() | currency: 'USD' }}</p>
  `,
})
export class CartPage {
  protected readonly cart = inject(Cart);
}
```

## Routing, rendering and configuration

```ts title="src/app/app.routes.ts"
import { Routes } from '@angular/router';

export const routes: Routes = [
  { path: '', loadComponent: () => import('./catalog/home').then((m) => m.Home) },
  { path: 'products/:slug', loadComponent: () => import('./catalog/product-page').then((m) => m.ProductPage) },
  { path: 'search', loadComponent: () => import('./catalog/search').then((m) => m.Search) },
  { path: 'cart', loadComponent: () => import('./catalog/cart-page').then((m) => m.CartPage) },
  { path: '**', redirectTo: '' },
];
```

```ts title="src/app/app.routes.server.ts"
import { PrerenderFallback, RenderMode, ServerRoute } from '@angular/ssr';
import { PRODUCTS } from './catalog/products.data';

export const serverRoutes: ServerRoute[] = [
  // Static marketing page: rendered once at build time.
  { path: '', renderMode: RenderMode.Prerender },
  // One HTML file per known product at build time; unknown slugs are rendered on request.
  {
    path: 'products/:slug',
    renderMode: RenderMode.Prerender,
    getPrerenderParams: async () => PRODUCTS.map((p) => ({ slug: p.slug })),
    fallback: PrerenderFallback.Server,
  },
  // Depends on the query string: rendered on the server for every request.
  { path: 'search', renderMode: RenderMode.Server },
  // Depends on localStorage: nothing useful to render on the server.
  { path: 'cart', renderMode: RenderMode.Client },
  { path: '**', renderMode: RenderMode.Server },
];
```

```ts title="src/app/app.config.ts"
import { ApplicationConfig, provideBrowserGlobalErrorListeners } from '@angular/core';
import { provideHttpClient } from '@angular/common/http';
import { provideClientHydration, withEventReplay } from '@angular/platform-browser';
import { provideRouter, withComponentInputBinding } from '@angular/router';
import { routes } from './app.routes';

export const appConfig: ApplicationConfig = {
  providers: [
    provideBrowserGlobalErrorListeners(),
    provideRouter(routes, withComponentInputBinding()),
    provideHttpClient(),
    provideClientHydration(withEventReplay()),
  ],
};
```

```ts title="src/app/app.ts"
import { Component, inject } from '@angular/core';
import { RouterLink, RouterOutlet } from '@angular/router';
import { Cart } from './catalog/cart';

@Component({
  selector: 'app-root',
  imports: [RouterOutlet, RouterLink],
  template: `
    <header>
      <a routerLink="/"><strong>Desk &amp; Lamp</strong></a>
      <nav>
        <a routerLink="/search">Search</a>
        <a routerLink="/cart">Cart ({{ cart.count() }})</a>
      </nav>
    </header>
    <main><router-outlet /></main>
  `,
})
export class App {
  protected readonly cart = inject(Cart);
}
```

And the API added to the generated `src/server.ts` (plus
`import { PRODUCTS } from './app/catalog/products.data';` and a `let apiCalls = 0;`
counter at the top):

```ts title="src/server.ts (excerpt)"
/**
 * A tiny JSON API. The Angular app calls it with HttpClient during server rendering
 * and in the browser.
 */
app.get('/api/search', (req, res) => {
  const q = String(req.query['q'] ?? '').toLowerCase();
  apiCalls++;
  res.json(PRODUCTS.filter((p) => p.name.toLowerCase().includes(q) || p.category === q));
});
app.get('/api/stats', (_req, res) => {
  res.json({ apiCalls });
});
```

## Tests

```ts title="src/app/catalog/product-page.spec.ts"
import { RESPONSE_INIT } from '@angular/core';
import { TestBed } from '@angular/core/testing';
import { Title } from '@angular/platform-browser';
import { provideRouter } from '@angular/router';
import { ProductPage } from './product-page';

describe('ProductPage', () => {
  async function render(slug: string, responseInit: ResponseInit | null = null) {
    TestBed.configureTestingModule({
      providers: [provideRouter([]), { provide: RESPONSE_INIT, useValue: responseInit }],
    });
    const fixture = TestBed.createComponent(ProductPage);
    fixture.componentRef.setInput('slug', slug);
    await fixture.whenStable();
    return fixture.nativeElement as HTMLElement;
  }

  it('renders a known product and sets the page title', async () => {
    const el = await render('arc-floor-lamp');
    expect(el.querySelector('h2')?.textContent).toBe('Arc Floor Lamp');
    expect(TestBed.inject(Title).getTitle()).toBe('Arc Floor Lamp · Desk & Lamp');
  });

  it('sets a 404 status for an unknown product during SSR', async () => {
    const responseInit: ResponseInit = {};
    const el = await render('no-such-thing', responseInit);
    expect(el.querySelector('h2')?.textContent).toBe('Product not found');
    expect(responseInit.status).toBe(404);
  });
});
```

```ts title="src/app/catalog/cart.spec.ts"
import { PLATFORM_ID } from '@angular/core';
import { TestBed } from '@angular/core/testing';
import { Cart } from './cart';

describe('Cart', () => {
  beforeEach(() => localStorage.clear());

  it('adds, merges quantities and totals prices', () => {
    const cart = TestBed.inject(Cart);
    cart.add('clamp-desk-lamp');
    cart.add('clamp-desk-lamp');
    cart.add('cable-tray');
    expect(cart.count()).toBe(3);
    expect(cart.total()).toBe(79 * 2 + 35);
  });

  it('does not touch localStorage on the server', () => {
    TestBed.configureTestingModule({ providers: [{ provide: PLATFORM_ID, useValue: 'server' }] });
    const spy = vi.spyOn(Storage.prototype, 'getItem');
    const cart = TestBed.inject(Cart);
    cart.add('cable-tray');
    TestBed.tick();
    expect(spy).not.toHaveBeenCalled();
    expect(localStorage.getItem('catalog-cart')).toBeNull();
  });
});
```

Providing `RESPONSE_INIT` and `PLATFORM_ID: 'server'` in the test is how you check
server-only behaviour without running a server.

## Checking it works

`ng test`: 4 of 4 passed. `ng build` reported `Prerendered 7 static routes.` and these
browser bundles:

```text
Initial chunk files  | Names            |  Raw size | Estimated transfer size
main-RZ4V5SJA.js     | main             | 307.36 kB |                85.27 kB
styles-5INURTSO.css  | styles           |   0 bytes |                 0 bytes

                     | Initial total    | 307.36 kB |                85.27 kB

Lazy chunk files     | Names            |  Raw size | Estimated transfer size
chunk-De_yyEdm.js    | product-page     |   1.66 kB |               859 bytes
chunk-DkYbdOwa.js    | search           |   1.65 kB |               861 bytes
chunk-BC0yod_6.js    | cart-page        |   1.06 kB |               609 bytes
chunk-D8ECwJES.js    | related-products | 841 bytes |               841 bytes
chunk-nR_sToOh.js    | home             | 783 bytes |               783 bytes
chunk-DCBpEu1A.js    | -                | 700 bytes |               700 bytes
chunk-DcMscjxV.js    | -                | 421 bytes |               421 bytes
```

We started the server with `NG_ALLOWED_HOSTS=localhost node dist/catalog/server/server.mjs`
(without the allow-list every request got `400 Bad Request` — lesson 06) and checked each
route with `curl`:

```text
/                        200  prerendered (ng-server-context="ssg")  title "Home · Desk & Lamp"
/products/arc-floor-lamp 200  prerendered                            title "Arc Floor Lamp · Desk & Lamp"
/products/no-such-thing  404  rendered on request (ssr)              "Product not found"
/search?q=lamp           200  rendered on request (ssr)              "2 result(s) for “lamp”"
/cart                    200  empty shell, rendered in the browser
```

Then in headless Chromium:

```text
/search?q=lamp first load       server API calls +1, browser /api requests: []  (transfer cache)
new search "desk"               browser /api requests: ['/api/search?q=desk']     "4 result(s)"
add a lamp on a product page    header "Cart (1)"; /cart shows it, also after reload
product page, 120 px viewport   related products present in HTML before scrolling;
                                chunk-D8ECwJES.js (related-products) requested only after scrolling
click a related product         navigated to /products/clamp-desk-lamp
console errors                  none
```

In an earlier run with a 300 px-tall viewport, the related section was already on screen,
so its chunk loaded immediately — `hydrate on viewport` means *in* the viewport, and on a
short product page that may be "right away". Choose triggers with your real layouts in mind.

## How It Actually Works

Follow a first visit to `/products/arc-floor-lamp`:

1. **At build time**, `getPrerenderParams` returned six slugs; the builder rendered each
   product page on the server platform, waited for stability, and wrote the HTML — including
   the related products and hydration annotations — to
   `dist/catalog/browser/products/arc-floor-lamp/index.html`.
2. **The request** hits Express. `express.static` finds that file and sends it; Angular
   code doesn't run on the server at all for this request. (`/products/no-such-thing` has no
   file, so it falls through to `AngularNodeAppEngine`, which renders it on demand and
   applies the 404 status the component set through `RESPONSE_INIT`.)
3. **The browser** paints the HTML immediately, then loads `main.js` and the
   `product-page` chunk and **hydrates**: it attaches to the existing DOM using the `ngh`
   annotations rather than re-creating it. The `@defer (hydrate on viewport)` block is left
   **dehydrated** — visible, but with no Angular code attached yet.
4. **On scroll**, an `IntersectionObserver` fires, Angular downloads the
   `related-products` chunk and hydrates just that block, making its links live.
5. **Clicking "Add to cart"** works on the hydrated part as normal: the `Cart` signal
   updates, and the header's `count()` re-renders.

Each page does the least work that still gives the user something useful as early as
possible — static HTML where the content is the same for everyone, server rendering where
it depends on the request, client rendering where only the browser knows the answer, and
JavaScript only for the parts the user actually reaches.

## Extensions

1. **Images.** Add product photos with `NgOptimizedImage`, marking the product image as
   `priority`, and check that SSR emits a preload link for it (lesson 05).
2. **Structured data.** Add a JSON-LD `Product` script to each product page and validate
   it with a structured-data testing tool.
3. **Sitemap.** Generate `sitemap.xml` from `PRODUCTS` in `server.ts` or at build time.
4. **Cache headers.** Set `Cache-Control` for prerendered HTML and `no-store` for any
   personalised server-rendered route.
5. **Test the server.** Write a Node test that imports the built server's handler and
   asserts the status codes above.

## Exercise

Answer briefly:

1. Why is `/search` rendered per request instead of prerendered, and why is `/cart` client-only?
2. What would go wrong if `Cart` read `localStorage` in its constructor unconditionally?
3. How can you tell from the page source whether a response was prerendered or rendered on
   request?
4. Why did the first `/search` load make no browser API request, and when would you
   want to disable that behaviour?
