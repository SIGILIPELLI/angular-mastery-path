---
description: "Ship Angular components as standard custom elements with @angular/elements, use them from plain HTML or other frameworks, and weigh micro-frontend approaches honestly."
---

# 04 · Angular Elements & Micro-Frontends

Sometimes an Angular component needs to live somewhere that isn't an Angular app: a CMS
page, a server-rendered site, a React app owned by another team, a static marketing page.
**Angular Elements** (`@angular/elements`) packages a component as a **custom element** —
a browser-native HTML tag — so any page can use it with plain HTML and DOM APIs.

This lesson turns the reading list's `StarRating` component (Level 1, lesson 10) into a
`<star-rating>` element, drives it from vanilla JavaScript, and then discusses the larger
topic of micro-frontends.

## Building an element

```bash
npm install @angular/elements
```

Bootstrap an application **without a root component** and register the element:

```ts title="src/main.ts"
import { createApplication } from '@angular/platform-browser';
import { createCustomElement } from '@angular/elements';
import { provideBrowserGlobalErrorListeners } from '@angular/core';
import { StarRating } from './app/reading/star-rating';

createApplication({ providers: [provideBrowserGlobalErrorListeners()] })
  .then((app) => {
    const element = createCustomElement(StarRating, { injector: app.injector });
    customElements.define('star-rating', element);
  })
  .catch((err) => console.error(err));
```

`createApplication` gives you an environment injector with your providers (HTTP,
services, …) but renders nothing by itself. `createCustomElement` wraps the component in
a class the browser understands. The component itself is unchanged.

`ng build` produced a single initial bundle of 127.62 kB raw (38.14 kB estimated
transfer) — smaller than the full reading-list app because there's no router or other
pages.

## Using it from plain HTML

```html title="src/index.html"
<!doctype html>
<html lang="en">
<head>
  <meta charset="utf-8">
  <title>Plain HTML page</title>
  <base href="/">
</head>
<body>
  <h1>Not an Angular page</h1>
  <p>Rate this article: <star-rating id="r" value="2" label="Article rating"></star-rating></p>
  <p id="out">(no rating yet)</p>
  <script>
    // Plain DOM code — no Angular here.
    const r = document.getElementById('r');
    r.addEventListener('valueChange', (e) => {
      document.getElementById('out').textContent = 'You rated ' + e.detail;
    });
  </script>
</body>
</html>
```

We served the build and drove the element from JavaScript in Chromium:

```text
1 initial (value="2")        2 stars on | aria-label: Article rating
2 clicked 4th star           4 on | "You rated 4"
3 r.value = 1   (property)   1 on | property reads 1
4 r.setAttribute('value','5') 5 on | typeof r.value: string, "5"
5 created with createElement 5 buttons, label "Second"
console errors: none
```

The mapping from Angular to DOM:

| Angular | Custom element |
|---------|----------------|
| `input()` / `model()` named `label` | **attribute** `label` and **property** `label` (camelCase inputs map to dash-case attributes: `maxValue` → `max-value`) |
| `output()` / `model()` change named `valueChange` | **DOM `CustomEvent`** `valueChange`, payload in `event.detail` |
| component creation | when the element is **connected** to the document |
| component destruction | shortly after the element is **disconnected** |

### The type trap

Look at row 4. Attributes are always strings, so `setAttribute('value', '5')` put the
**string** `"5"` into a component that thinks `value` is a number. It happened to render
because `star <= value()` coerces, but `value() + 1` would give `"51"`. For elements,
add input transforms (`numberAttribute`, `booleanAttribute` — Level 1, lesson 06) to
every non-string input, or document that consumers must set properties, not attributes.

### Other things to know

- **Styles:** the element uses the component's view encapsulation. We checked
  `r.shadowRoot` — `false` with the default emulated encapsulation. Use
  `ViewEncapsulation.ShadowDom` if the host page's CSS must not leak in (and accept that
  your styles can't be themed from outside except through CSS custom properties).
- **Zoneless works:** inputs set from outside are applied through the same signal-input
  machinery as `setInput`, so the element re-renders without Zone.js.
- **Size:** each separately built element bundle includes its own copy of Angular.
  Putting several elements in **one** build (define them all in `main.ts`) shares the
  framework code between them.

## Micro-frontends

"Micro-frontends" means splitting one product's front end into parts that different teams
build and **deploy independently**, composed in the browser. The motivation is
organisational — team autonomy and release independence — not technical elegance. Common
approaches:

1. **Custom elements** (this lesson). Framework-agnostic, simple contract (attributes,
   properties, events). Each part ships its own framework copy unless you arrange sharing.
2. **Module / Native Federation.** A host app loads remote bundles at runtime and shares
   dependencies (such as `@angular/core`) between them via a manifest. For Angular's
   esbuild-based builder this is provided by community projects (Native Federation is the
   commonly used one); check its compatibility with your Angular version before adopting.
3. **Route-level composition** by the server or edge: different paths served by different
   apps, sharing only a design system and authentication.

Honest trade-offs:

- Independent deployment means **runtime integration**: version mismatches, duplicated
  dependencies, and contracts you can no longer check with the compiler.
- Shared state, routing and design consistency all become explicit problems.
- A well-structured single app (lesson 06) or a monorepo with libraries (lesson 02) gives
  most teams the separation they need with far less complexity. Reach for micro-frontends
  when independent release cycles are a hard requirement.

## How It Actually Works

`createCustomElement` generates a class extending `HTMLElement` and registers
`observedAttributes` derived from the component's inputs. The browser calls:

- `connectedCallback` → Angular creates the component with `createComponent` using your
  injector, attaches its host view to the application, and applies any attributes or
  properties set earlier (properties set before upgrade are captured and replayed).
- `attributeChangedCallback` → the attribute string is passed to the matching input
  (through its transform, if any) — hence the string in row 4.
- property setters → forwarded to the input directly, keeping the original type.
- `disconnectedCallback` → Angular schedules destruction; if the element is re-attached
  quickly (e.g. moved in the DOM), destruction is cancelled.

Outputs are subscribed once the component exists, and each emission is re-dispatched as a
`CustomEvent` on the host element. Because the component is attached to a real
`ApplicationRef`, change detection, DI and signals behave exactly as in an Angular app.

## Common mistakes

- **Number and boolean inputs without transforms** (strings from attributes).
- **Several independently built elements on one page**, each shipping Angular.
- **Relying on host-page CSS** to style an element with Shadow DOM encapsulation.
- **Choosing micro-frontends for technical reasons** rather than organisational ones.
- **Forgetting cleanup** in components that start timers or subscriptions — elements are
  destroyed when removed from the DOM, and leaks multiply on pages that add and remove
  them often.

## Exercise

1. Turn one of your components into a custom element and use it in a static HTML file
   with no build step on the page itself.
2. Add `numberAttribute` to its numeric inputs and repeat the `setAttribute` experiment:
   confirm the property is now a number.
3. Listen for one of its outputs from vanilla JavaScript and log `event.detail`.
4. Use the same element inside a small React or Vue page (or a plain `<script type="module">`)
   and note what integration work was needed.
5. Write a one-page decision record: for your organisation, would micro-frontends solve a
   real problem that a monorepo wouldn't?
