---
description: "Angular pipes and directives: built-in formatting pipes with real output, writing a pure custom pipe, attribute directives with host bindings, and hostDirectives."
---

# 09 · Pipes & Directives

Components are one kind of **directive** — the kind with a template. Angular has two
more building blocks for reuse in templates:

- **Pipes** transform a value for display: `{{ price | currency: 'EUR' }}`.
- **Attribute directives** add behaviour or styling to an existing element:
  `<p appHighlight>`.

Both are standalone classes you import into a component's `imports`, exactly like
components.

## Built-in pipes, with real output

All live in `@angular/common`. We rendered each of these in a test (locale `en-US`, the
default; the machine's time zone was IST, UTC+5:30):

```ts
when   = new Date(Date.UTC(2026, 8, 28, 14, 5));   // 28 Sep 2026, 14:05 UTC
price  = 1234.5;   ratio = 0.4567;   big = 9876543.21;   pi = Math.PI;
```

| Template | Output |
|----------|--------|
| `{{ when \| date }}` | `Sep 28, 2026` |
| `{{ when \| date: 'yyyy-MM-dd HH:mm' : 'UTC' }}` | `2026-09-28 14:05` |
| `{{ when \| date: 'fullDate' : 'UTC' }}` | `Monday, September 28, 2026` |
| `{{ price \| currency }}` | `$1,234.50` |
| `{{ price \| currency: 'EUR' }}` | `€1,234.50` |
| `{{ price \| currency: 'INR' : 'symbol' : '1.0-0' }}` | `₹1,235` |
| `{{ ratio \| percent: '1.1-1' }}` | `45.7%` |
| `{{ big \| number }}` | `9,876,543.21` |
| `{{ pi \| number: '1.2-2' }}` | `3.14` |
| `{{ 'the lord of the rings' \| titlecase }}` | `The Lord Of The Rings` |
| `{{ 'Angular' \| uppercase }}` | `ANGULAR` |
| `{{ ['a','b','c','d'] \| slice: 1 : 3 }}` | `b,c` |
| `{{ { a: 1 } \| json }}` | `{ "a": 1 }` |
| `@for (e of { zoe: 3, adam: 5 } \| keyvalue; ...)` | `adam=5 zoe=3` (sorted by key) |

Notes:

- **Digits info** `'1.2-2'` means *minimum integer digits . minimum fraction digits -
  maximum fraction digits*.
- **Time zones:** `date` formats in the **browser's** time zone unless you pass one. Server
  timestamps displayed to users in several regions usually want an explicit zone, or
  `'UTC'` for logs. Without it, the same test on a machine in another zone can print a
  different date.
- **Other locales** need their locale data registered (Level 4, lesson 03).
- **`titlecase`** capitalises every word, including "Of" and "The" — it's not a
  style-guide title case.
- **`async`** subscribes to an Observable or Promise (Level 2, lesson 04).
- **`json`** is for debugging only.

Pipes can be chained (`{{ name | slice: 0 : 20 | uppercase }}`) and take arguments after
colons.

## A custom pipe

```ts title="src/app/relative-time.pipe.ts"
import { Pipe, PipeTransform } from '@angular/core';

@Pipe({ name: 'relativeTime' })
export class RelativeTimePipe implements PipeTransform {
  private readonly rtf = new Intl.RelativeTimeFormat('en', { numeric: 'auto' });

  transform(value: Date | string | number, now: Date = new Date()): string {
    const diffSec = Math.round((new Date(value).getTime() - now.getTime()) / 1000);
    const units: [Intl.RelativeTimeFormatUnit, number][] = [
      ['year', 31536000], ['month', 2592000], ['day', 86400],
      ['hour', 3600], ['minute', 60], ['second', 1],
    ];
    for (const [unit, secs] of units) {
      if (Math.abs(diffSec) >= secs || unit === 'second') {
        return this.rtf.format(Math.round(diffSec / secs), unit);
      }
    }
    return '';
  }
}
```

With `now` at 12:00, a timestamp from 12:00 the previous day rendered `yesterday` and
one at 15:00 the same day rendered `in 3 hours`. `ng generate pipe relative-time`
scaffolds the file.

### Pure pipes and why `now` is an argument

Pipes are **pure** by default: Angular calls `transform` again only when an input
*reference* changes. `{{ post.createdAt | relativeTime }}` would compute "5 minutes ago"
once and never update, because `createdAt` never changes. Making `now` an explicit
argument, fed from a signal that ticks every minute, makes the pipe re-run exactly when it
should:

```ts
protected readonly now = toSignal(interval(60_000).pipe(map(() => new Date())), {
  initialValue: new Date(),
});
// template: {{ post.createdAt | relativeTime: now() }}
```

`@Pipe({ name, pure: false })` makes a pipe run on every change detection of its view.
It's occasionally necessary (the `async` pipe is impure) but a performance trap for
anything non-trivial. Similarly, pure pipes don't notice *mutation*: `items.push(x)` keeps
the same array reference, so `items | slice: 0 : 3` won't update.

A pure pipe is a good replacement for calling a method in a template, which runs on every
refresh. A `computed()` in the component is the other option; use a pipe when the same
transformation is reused across many components.

## Attribute directives

```ts title="src/app/highlight.ts"
import { Directive, input, signal } from '@angular/core';

@Directive({
  selector: '[appHighlight]',
  host: {
    '[style.background-color]': 'active() ? color() : null',
    '(mouseenter)': 'active.set(true)',
    '(mouseleave)': 'active.set(false)',
  },
})
export class Highlight {
  readonly color = input('#fff3b0', { alias: 'appHighlight' });
  protected readonly active = signal(false);
}
```

```html
<p appHighlight="lightblue">Hover me</p>
<p appHighlight>Default colour</p>
```

The selector `[appHighlight]` matches any element with that attribute. Aliasing the input
to the selector name lets the attribute double as its argument. The `host` object binds
properties and listens to events on the element the directive sits on — the modern
replacement for the `@HostBinding` / `@HostListener` decorators you'll see in older code.

Hovering the first paragraph set `background-color: lightblue`; leaving cleared it back to
empty. The second, with no value, used the default only when hovered.

A directive can also **inject** the element (or other directives on it):

```ts
@Directive({ selector: '[appAutofocus]' })
export class Autofocus {
  private readonly el = inject<ElementRef<HTMLElement>>(ElementRef);
  readonly enabled = input(true, { alias: 'appAutofocus', transform: booleanAttribute });
  constructor() {
    queueMicrotask(() => { if (this.enabled()) this.el.nativeElement.focus(); });
  }
}
```

`<input appAutofocus />` had focus after rendering (`document.activeElement` was that
input). Touching `nativeElement` directly is fine for focus and measurement, but prefer
host bindings for anything that can be expressed declaratively — they work with
server-side rendering, where there's no real DOM to call `.focus()` on.

## Composition with `hostDirectives`

A component or directive can **apply other directives to its own host** and choose which
of their inputs and outputs to expose:

```ts
@Directive({
  selector: '[appTooltip]',
  host: { '[attr.title]': 'text()', '[attr.aria-label]': 'text()' },
})
export class Tooltip {
  readonly text = input.required<string>({ alias: 'appTooltip' });
}

@Component({
  selector: 'app-icon-button',
  hostDirectives: [{ directive: Tooltip, inputs: ['appTooltip: label'] }, Highlight],
  host: { role: 'button', tabindex: '0' },
  template: `<ng-content />`,
})
export class IconButton {}
```

`<app-icon-button label="Delete item">🗑</app-icon-button>` rendered as:

```html
<app-icon-button role="button" tabindex="0" label="Delete item"
                 title="Delete item" aria-label="Delete item">🗑</app-icon-button>
```

and turned yellow (`rgb(255, 243, 176)`) on hover — `Tooltip` and `Highlight` behaviour
without the user of `IconButton` adding either attribute. Host directives must be
standalone and are matched statically; they can't be added conditionally.

## Structural directives, briefly

Directives used with `*` (like the legacy `*ngIf`) are **structural**: they receive an
`<ng-template>` and decide whether and how often to stamp it out using
`ViewContainerRef.createEmbeddedView`. The built-in `@if`/`@for` blocks cover nearly every
use, so you'll mostly write structural directives for library-style features (a
`*appPermission="'admin'"` directive, for example).

## How It Actually Works

**Pipes.** The compiler turns `{{ price | currency: 'EUR' }}` into a pipe instruction
that creates one pipe instance per usage in the view and calls a *pure function binding*:
it stores the last arguments and last result in the view's data, compares new arguments
with `Object.is`, and returns the cached result if nothing changed. That cache is the
entire meaning of "pure" — and why mutation isn't detected. Impure pipes skip the cache.

**Directive matching.** At compile time, the compiler matches every element in a template
against the selectors of the directives in the component's `imports`. For each match it
records "create directive X on this element", its input bindings and host bindings. At
runtime, directives on an element are instantiated together with the element; each gets
its own node injector entry, which is how `inject(ElementRef)` or `inject(OtherDirective)`
finds its host.

**Host bindings** compile to a small *host binding function* that runs during change
detection of the view containing the host element — the same update pass as the
template's own bindings. Host listeners are attached once at creation.

**Unused imports** are detected too: importing `LowerCasePipe` without using it produced
`▲ [WARNING] NG8113: LowerCasePipe is not used within the template of PipesDemo`.

## Common mistakes

- **Forgetting to import a pipe** — `NG0302: The pipe 'currency' could not be found` (or
  a build error).
- **Impure pipes for convenience** — they run constantly.
- **Expecting a pure pipe to see mutated arrays/objects.** Replace, don't mutate.
- **Formatting dates without thinking about time zones.**
- **Manipulating the DOM directly** when a host binding would do.

## Exercise

1. Write a pure `fileSize` pipe: `1536 | fileSize` → `1.5 KB`, with an argument for
   decimal places. Unit-test it without TestBed (it's just a class).
2. Write an `appClickOutside` directive with an `output()` that fires when a click happens
   outside the host element (listen with `host: { '(document:click)': ... }`), and use it
   to close a dropdown.
3. Build a `Card` component that uses `hostDirectives` to include your `Highlight`
   directive, exposing its colour input as `accent`.
4. Show a list of posts with `relativeTime`, fed by a `now` signal that updates every 30
   seconds, and verify the labels change without a page reload.
