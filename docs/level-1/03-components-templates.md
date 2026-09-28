---
description: "Angular components and templates: @Component metadata, interpolation, property, attribute, class, style and event bindings, template variables and scoped styles."
---

# 03 · Components & Templates

A component is the unit you build Angular apps from: a class that holds state and
behaviour, a template that describes the DOM, optional styles, and a selector that says
where it appears. This lesson covers everything you can put in a template *except* control
flow (lesson 05) and inputs/outputs (lesson 06).

## Anatomy of a component

```ts title="src/app/profile-card.ts"
import { Component, signal } from '@angular/core';

@Component({
  selector: 'app-profile-card',
  template: `
    <article [class.featured]="featured()" [style.border-color]="accent()">
      <img [src]="avatarUrl()" [alt]="name() + ' avatar'" width="64" height="64" />
      <h2>{{ name() }}</h2>
      <p [attr.aria-label]="'Followers: ' + followers()">{{ followers() }} followers</p>
      <input #nameBox [value]="name()" (keyup.enter)="name.set(nameBox.value)" />
      <button type="button" (click)="follow()">Follow</button>
      <button type="button" (click)="featured.set(!featured())">Toggle featured</button>
    </article>
  `,
  styles: `
    article { border: 2px solid; padding: 1rem; border-radius: 8px; }
    .featured { background: #fff7e0; }
    h2 { margin: 0.25rem 0; }
  `,
  host: { class: 'card', '[attr.data-followers]': 'followers()' },
})
export class ProfileCard {
  protected readonly name = signal('Ada Lovelace');
  protected readonly avatarUrl = signal('/favicon.ico');
  protected readonly followers = signal(10);
  protected readonly featured = signal(false);
  protected readonly accent = signal('#c3002f');

  protected follow() {
    this.followers.update((n) => n + 1);
  }
}
```

The `@Component` metadata you will use most:

| Property | Purpose |
|----------|---------|
| `selector` | The element name that creates this component (`<app-profile-card />`). Prefix with your app's prefix (`app-` by default) to avoid clashing with real or future HTML elements. |
| `template` / `templateUrl` | Inline template string, or path to an `.html` file. Use one or the other. |
| `styles` / `styleUrl` | Inline CSS string (or array), or path to a stylesheet. |
| `imports` | Components, directives and pipes this template uses. |
| `host` | Bindings and listeners on the component's own element (the `<app-profile-card>` tag). |
| `changeDetection` | Change detection strategy — `OnPush` by default in Angular 22 (Level 3, lesson 01). |

Members used in the template are marked `protected`: the template can see them, but other
TypeScript code that holds a reference to the component cannot. Use `private` for things
the template should not touch — the compiler will report an error if the template tries.

## Template syntax reference

### Interpolation — `{{ expression }}`

Writes the expression's value as **text**. `{{ followers() }} followers` becomes a text
node. HTML inside the value is escaped, not interpreted, so interpolating
`'<b>hi</b>'` shows the literal tags. Template expressions are a restricted subset of
JavaScript: you can call methods, read properties, use operators and the ternary, but you
cannot assign with `=` (except through a signal's `set`), use `new`, or reference globals
such as `window` or `console`.

### Property binding — `[property]="expression"`

Sets a **DOM property** on an element (or an input on a component — lesson 06).
`[src]="avatarUrl()"` sets `img.src`. The right-hand side is an expression, not a string:
`[disabled]="true"` passes the boolean `true`, while `disabled="true"` would be a plain
attribute.

### Attribute binding — `[attr.name]="expression"`

Some things are attributes with no matching DOM property of the same name — `data-*`
attributes, or `colspan` (the property is spelled `colSpan`). Use the `attr.` prefix:
`[attr.data-id]="id()"`, `[attr.colspan]="span()"`. Binding `null` or `undefined`
**removes** the attribute, which is how you conditionally omit one.

ARIA attributes are a special case: recent versions of Angular (including 22) accept
`[aria-label]="label()"` directly and set the attribute for you. `[attr.aria-label]`
still works and is what you will see in older code. Writing `[colspan]="2"`, on the other
hand, fails the build with
`NG8002: Can't bind to 'colspan' since it isn't a known property of 'td'`.

### Class and style bindings

```html
<article [class.featured]="featured()">            <!-- toggle one class -->
<article [class]="{ featured: featured(), compact: compact() }">  <!-- object form -->
<article [style.border-color]="accent()">          <!-- one style property -->
<div [style.width.px]="width()">                   <!-- with a unit suffix -->
```

Angular merges these with any static `class="..."` on the same element rather than
replacing it.

### Event binding — `(event)="statement"`

Listens to a DOM event (or a component output). `$event` is the native event object.
Key events accept filters: `(keyup.enter)`, `(keydown.escape)`, `(keydown.shift.tab)`.

### Template reference variables — `#name`

`#nameBox` gives the `<input>` element a name you can use elsewhere in the same template:
`nameBox.value`. On a component element, the variable refers to the component instance.
They are handy for small, local interactions; for anything the class needs to know about,
use a signal.

### Self-closing tags

Components can be written `<app-profile-card />`. Native void elements (`<img>`,
`<input>`) behave as in HTML.

## What the compiler catches

Templates are type-checked against the class. Here are three real errors from
`ng build` on Angular 22.2 for a deliberately broken component:

```text
✘ [ERROR] TS2339: Property 'nmae' does not exist on type '{ name: string; }'.
      4 │ ...<app-greeting /> <p>{{ user().nmae }}</p> <input [value]="count(...

✘ [ERROR] TS2345: Argument of type 'string' is not assignable to parameter of type 'number'.
      4 │ ...put [value]="count()" (input)="count.set($event.target.value)" />`,

✘ [ERROR] NG8001: 'app-greeting' is not a known element:
1. If 'app-greeting' is an Angular component, then verify that it is included in the
   '@Component.imports' of this component.
```

The first is a typo. The second is a genuine bug: `count` is a `signal<number>` but an
input's value is always a string — you would need `Number(...)`. The third is the
"forgot to import" error you will hit a lot at first: using another component in a
template requires adding its class to `imports`.

## How It Actually Works

**Rendering.** When Angular creates `ProfileCard`, it runs the compiled template function
in creation mode (build the elements, attach listeners) and then in update mode (evaluate
each binding and write it to the DOM). Afterwards, each binding remembers its last value.
On later refreshes, Angular re-evaluates the bindings in this view and writes to the DOM
only where a value actually changed. Here is the component's real DOM in Chromium after
clicking **Follow** twice, toggling featured and renaming:

```html
<app-profile-card class="card" _nghost-ng-c3634266716="" data-followers="12">
  <article _ngcontent-ng-c3634266716="" class="featured" style="border-color: rgb(195, 0, 47);">
    <img _ngcontent-ng-c3634266716="" width="64" height="64" src="/favicon.ico" alt="Grace Hopper avatar">
    <h2 _ngcontent-ng-c3634266716="">Grace Hopper</h2>
    <p _ngcontent-ng-c3634266716="" aria-label="Followers: 12">12 followers</p>
    ...
```

Notice the `host` metadata at work: `class="card"` and `data-followers` are on the
component's own tag.

**Style encapsulation.** Those odd `_nghost-…` and `_ngcontent-…` attributes are how
Angular scopes component CSS with the default `ViewEncapsulation.Emulated` mode. At
compile time every selector in the component's styles is rewritten to require the
component's unique attribute, and at runtime every element the component's template
creates is stamped with it. The actual stylesheet Angular injected was:

```css
article[_ngcontent-ng-c3634266716]{border:2px solid;padding:1rem;border-radius:8px}
.featured[_ngcontent-ng-c3634266716]{background:#fff7e0}
h2[_ngcontent-ng-c3634266716]{margin:.25rem 0}
```

So `h2 { ... }` in this component cannot affect an `<h2>` in any other component, and
global styles in `src/styles.css` still apply everywhere. Use the `:host` selector to
style the component's own element, and `ViewEncapsulation.ShadowDom` if you want real
Shadow DOM, or `ViewEncapsulation.None` to make a component's styles global (rarely what
you want). The attribute value is generated per component, so never write CSS that
targets it directly.

**Events.** `(click)="follow()"` compiles to an `addEventListener` call made once, in
creation mode. When the event fires, Angular runs your statement and then marks the view
as needing a refresh; since `follow()` changed a signal the template reads, Angular
schedules a render and the new count appears.

## Common mistakes

- **Forgetting to call the signal.** `{{ name }}` renders the signal *function*, not its
  value. Write `{{ name() }}`.
- **Confusing attributes and properties.** `[colspan]` fails because the DOM property is
  `colSpan`; use `[attr.colspan]` or `[colSpan]`. When in doubt, the `attr.` prefix
  always targets the HTML attribute.
- **Calling expensive methods in templates.** `{{ totalFor(items) }}` runs on every
  refresh of the view. Derive it once with `computed()` (lesson 04).
- **Binding user HTML with `[innerHTML]` and expecting scripts or styles to survive.**
  Angular sanitises it (Level 3, lesson 09). Interpolation never renders HTML at all.
- **Styling child components from a parent.** Emulated encapsulation stops this on
  purpose. Pass a class or an input to the child, or use CSS custom properties, which
  inherit through component boundaries.

## Exercise

Build a `TemperatureCard` component:

1. It holds a `celsius` signal starting at `20`.
2. The template shows the Celsius value and the Fahrenheit equivalent (°F = °C × 9/5 + 32).
3. Two buttons change the temperature by ±1.
4. When the temperature is above 30, the card gets a `hot` CSS class (red background);
   below 5, a `cold` class (blue).
5. Add `[attr.aria-live]="'polite'"` to the element showing the value, and check in the
   browser's element inspector that the attribute is present.
6. Deliberately misspell a property in the template and read the build error, then fix it.
