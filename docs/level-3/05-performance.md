---
description: "Make Angular apps fast: measure bundles with --stats-json, enforce budgets, split code, optimise images with NgOptimizedImage, and keep rendering cheap."
---

# 05 · Performance

Performance work has two halves: **loading** (how quickly the first useful screen
appears) and **runtime** (how smoothly the app responds afterwards). Angular gives you
good defaults for both, but the biggest wins still come from measuring your own app and
fixing what the measurements point at. This lesson is a checklist you can apply to any
Angular project, with each tool shown on a real build.

## 1. Measure the bundle

`ng build` prints sizes per chunk. To see **what is inside** a chunk, ask for the esbuild
metafile:

```bash
ng build --stats-json
```

With Angular 22 this wrote `dist/<project>/browser-stats.json`. You can drop that file
onto esbuild's online bundle analyzer (esbuild.github.io/analyze) for a treemap, or
summarise it yourself. For the Level 1 reading-list app, grouping each output file's
inputs by package gave:

```text
chunk-CFBDKLSF.js  160,347 bytes   @angular/core 116.9 KB · @angular/common 20.1 KB · rxjs 15.4 KB · tslib 1.9 KB · src 0.8 KB
chunk-VDKBJWIX.js   95,290 bytes   @angular/router 76.5 KB · @angular/platform-browser 11.9 KB · @angular/common 3.8 KB
main-YHP7CJVG.js     5,353 bytes   src 4.9 KB
```

(Sizes are minified, before compression.) This is the question to ask of every large
chunk: *which package is it, and does the first screen need it?* A date library, a chart
package or a full icon set showing up in the initial chunks is the usual culprit.

## 2. Enforce budgets

Budgets in `angular.json` turn size regressions into build warnings or errors:

```json title="angular.json (excerpt)"
"budgets": [
  { "type": "initial", "maximumWarning": "500kB", "maximumError": "1MB" },
  { "type": "anyComponentStyle", "maximumWarning": "4kB", "maximumError": "8kB" }
]
```

Those are the generated defaults. Tighten `initial` to a little above your current size
so that an accidental 100 kB import fails CI instead of shipping. Other budget types
include `bundle` (a named lazy chunk) and `anyScript`.

## 3. Load less up front

- **Lazy routes** (`loadComponent`, `loadChildren`) — each feature is its own chunk.
- **`@defer`** for heavy parts of a page (lesson 04).
- **Import precisely.** `import { debounce } from 'lodash-es'` tree-shakes; `import _ from 'lodash'`
  pulls in the whole CommonJS library, and the CLI warns about CommonJS dependencies
  because they defeat tree-shaking.
- **Check `canMatch` for role-specific areas** so their chunks aren't downloaded by
  everyone (Level 2, lesson 07).

## 4. Images: `NgOptimizedImage`

Images are often the largest bytes on a page and the element that decides Largest
Contentful Paint (LCP). The `NgOptimizedImage` directive (import it from
`@angular/common`, use `ngSrc` instead of `src`) applies best practices by default:

```html
<img ngSrc="/hero.jpg" width="1600" height="900" priority alt="Hero" />
<img ngSrc="/thumb.jpg" width="400" height="300" alt="Thumbnail" />
```

The rendered DOM in Chromium:

```html
<img ngsrc="/hero.jpg" width="1600" height="900" priority="" alt="Hero"
     loading="eager" fetchpriority="high" decoding="sync" src="/hero.jpg">
<img ngsrc="/thumb.jpg" width="400" height="300" alt="Thumbnail"
     loading="lazy" fetchpriority="auto" decoding="auto" src="/thumb.jpg">
```

- `width`/`height` are **required** — they reserve space and prevent layout shift.
- Images are **lazy** by default; with the thumbnail 6,000 px below the fold, only
  `hero.jpg` was requested on load.
- `priority` marks the LCP image: eager, high fetch priority. Put it on exactly the
  image that is your LCP element.
- In development it checks your work. An image whose real ratio (400×300) didn't match
  its attributes (400×400) produced:

```text
NG02952: The NgOptimizedImage directive (activated on an <img> element with the
`ngSrc="/thumb.jpg"`) has detected that the aspect ratio of the image does not match
the aspect ratio indicated by the width and height attributes.
```

With an image CDN, register a **loader** (`provideImgixLoader(...)`,
`provideCloudinaryLoader(...)`, or a custom `IMAGE_LOADER`) and add `ngSrcset`/`sizes`,
and the directive will generate responsive `srcset`s for you. Use `fill` for images that
should cover their parent instead of having fixed dimensions.

## 5. Keep rendering cheap

Angular 22's defaults — OnPush everywhere, zoneless scheduling, signals — already
minimise work per change (lesson 01). The remaining runtime costs you control:

| Symptom | Fix |
|---------|-----|
| A method call in a template doing real work (`{{ total(items) }}`) | `computed()` or a pure pipe — computed once per change, not per render |
| List rows re-created after every fetch | `track` by a stable id (Level 1, lesson 05) |
| Thousands of rows | paginate, or virtual scrolling (`@angular/cdk/scrolling`'s `cdk-virtual-scroll-viewport`) |
| An effect that writes signals and triggers more renders | derive with `computed`/`linkedSignal` instead |
| Hot `(scroll)`/`(mousemove)` handlers | throttle, or listen outside templates and only write a signal when a value changes |
| Heavy work on the main thread | `ng generate web-worker` and move it to a worker |

## 6. Profile before optimising

- **Angular DevTools** (a browser extension) has a **Profiler** tab that records
  change-detection passes and shows how long each component took, and an injector/
  component tree explorer. Record, interact, and look for components that run far more
  often or take longer than you expect.
- **Chrome's Performance panel** shows long tasks, layout shifts and LCP. Recent Angular versions can also add
  framework-specific tracks to that timeline in development builds (through Chrome's
  DevTools extensibility API), labelling change detection and component work; check the
  Angular DevTools guide on angular.dev for how to switch it on in your version.
- **Field data** matters more than lab data: measure Core Web Vitals (LCP, INP, CLS) for
  real users with the `web-vitals` library or your analytics provider.

We can't show profiler screenshots that would match your machine, and numbers from one
laptop don't transfer to another — measure your app on the hardware your users have.

## How It Actually Works

**Why the initial bundle is what it is.** The application builder hands esbuild one entry
(`main.ts`) plus every dynamic `import()` it finds (lazy routes, `@defer` dependencies).
esbuild builds a module graph and **splits** it: code reachable only from a dynamic import
goes into that import's chunk; code shared between several entry points goes into a
shared chunk (the unnamed `chunk-XXXX.js` files) so it's downloaded once. Tree-shaking
removes exports nothing imports, which works because Angular's packages are ES modules
marked side-effect free and because the Angular compiler emits static definitions
(`ɵcmp`, `ɵprov`) that bundlers can analyse. Services that are provided "in root" but
never injected, and components never referenced, disappear.

**Why `priority` helps LCP.** Browsers discover images while parsing HTML and assign them
a fetch priority; lazily-loaded or low-priority images can wait behind scripts and
stylesheets. `fetchpriority="high"` + `loading="eager"` tells the browser to fetch the
LCP image immediately. With server-side rendering (lesson 06), `NgOptimizedImage` can
also emit a `<link rel="preload">` for priority images so the fetch starts before any
JavaScript runs.

## Common mistakes

- **Optimising without measuring.** Guessing wastes time on things that don't matter.
- **One giant shared module/barrel file** that every feature imports, dragging unrelated
  code into the initial chunk.
- **`priority` on many images.** Only the LCP image should have it.
- **Images without dimensions**, causing layout shift.
- **Raising budgets whenever they fail** instead of asking why the bundle grew.

## Exercise

1. Run `ng build --stats-json` on one of your projects and list the five largest
   packages in the initial chunks. For each, decide: needed on first screen, lazy-loadable,
   or replaceable?
2. Set the `initial` budget to 10% above the current size. Add a heavy import to the root
   component and confirm the build fails.
3. Convert the images on one page to `NgOptimizedImage`. Mark the LCP image as
   `priority`, verify in the Network panel that below-the-fold images are not fetched
   on load, and fix any NG0295x warnings.
4. Find one template method call doing real work and replace it with a `computed`. Use
   the Angular DevTools profiler to compare before and after.
