---
description: "Compose Angular components: ng-content slots with fallbacks, template outlets with context, and signal queries like contentChildren."
---

# 02 · Composition: Projection & Queries

Inputs pass *data* into a component. Often you want to pass *markup* instead — a card
whose title and body are decided by the caller, a tab set whose panels are arbitrary
content, a list whose row layout the caller controls. Angular gives you three tools:

- **Content projection** with `<ng-content>` — the caller's markup is placed into slots.
- **Template outlets** — the caller passes an `<ng-template>` and the component stamps it
  out, with data, as many times as it likes.
- **Queries** — `viewChild`, `viewChildren`, `contentChild`, `contentChildren` — let a
  component get references to elements, directives and components in its own template or
  in projected content, as signals.

All the code below was rendered in tests on Angular 22.2; the observed DOM is shown.

## Slots with `<ng-content>`

```ts title="src/app/card.ts"
@Component({
  selector: 'app-card',
  template: `
    <article>
      <header><ng-content select="[card-title]">Untitled</ng-content></header>
      <section><ng-content /></section>
      <footer><ng-content select="app-card-actions, [card-actions]" /></footer>
    </article>
  `,
})
export class Card {}
```

Used like this:

```html
<app-card>
  <h2 card-title>Quarterly report</h2>
  <p>Revenue grew.</p>
  <button card-actions>Share</button>
</app-card>
<app-card><p>No title here.</p></app-card>
```

Rendered `<article>` contents:

```html
<header><h2 card-title="">Quarterly report</h2></header>
<section><p>Revenue grew.</p></section>
<footer><button card-actions="">Share</button></footer>

<header>Untitled</header><section><p>No title here.</p></section><footer></footer>
```

- `select` takes a CSS selector (element, attribute, class, or a comma-separated list).
- The `<ng-content>` **without** `select` receives everything that didn't match another
  slot.
- Content inside `<ng-content>…</ng-content>` is **fallback** shown when nothing was
  projected — the second card got "Untitled".
- Each piece of content goes to one slot only; you can't project the same node twice.

## Queries as signals

```ts title="src/app/search-box.ts"
@Component({
  selector: 'app-search-box',
  template: `<input #box type="search" [placeholder]="placeholder()" /> <button type="button" (click)="focus()">Focus</button>`,
})
export class SearchBox {
  readonly placeholder = input('Search…');
  private readonly box = viewChild.required<ElementRef<HTMLInputElement>>('box');
  focus() { this.box().nativeElement.focus(); }
}
```

`viewChild.required<ElementRef<HTMLInputElement>>('box')` finds the element with the
`#box` reference in this component's template. Clicking **Focus** focused the input
(`document.activeElement` was the `INPUT` of type `search`). The four query functions:

| Query | Looks in | Returns |
|-------|----------|---------|
| `viewChild(x)` / `viewChild.required(x)` | this component's own template | `Signal<T \| undefined>` / `Signal<T>` |
| `viewChildren(x)` | own template | `Signal<readonly T[]>` |
| `contentChild(x)` / `.required` | projected content | `Signal<T \| undefined>` / `Signal<T>` |
| `contentChildren(x)` | projected content | `Signal<readonly T[]>` |

`x` is a template reference name (`'box'`), a component/directive class, or a token such
as `TemplateRef`. Use `{ read: ElementRef }` to get a different thing from the matched
node — for example the element of a component instead of its instance.

Because queries are signals, they update when content appears or disappears: they can
feed `computed` and `effect`, and there's no `ngAfterViewInit` timing to get right. Don't
read them in the constructor — nothing has rendered yet; read them in templates,
computeds, event handlers or `afterNextRender`.

## A tab set: projection + `contentChildren` + DI

```ts title="src/app/tabs.ts"
@Component({
  selector: 'app-tab',
  host: { role: 'tabpanel', '[hidden]': '!active()' },
  template: `<ng-content />`,
})
export class Tab {
  readonly label = input.required<string>();
  private readonly tabs = inject(Tabs);
  readonly active = computed(() => this.tabs.selectedTab() === this);
}

@Component({
  selector: 'app-tabs',
  template: `
    <div role="tablist">
      @for (tab of tabs(); track tab; let i = $index) {
        <button role="tab" type="button" [attr.aria-selected]="tab.active()" (click)="selectedIndex.set(i)">
          {{ tab.label() }}
        </button>
      }
    </div>
    <ng-content />
  `,
})
export class Tabs {
  protected readonly tabs = contentChildren(Tab);
  readonly selectedIndex = signal(0);
  readonly selectedTab = computed(() => this.tabs()[this.selectedIndex()]);
  readonly count = computed(() => this.tabs().length);
}
```

```html
<app-tabs #tabs>
  <app-tab label="Overview">Overview body</app-tab>
  <app-tab label="Details">Details body</app-tab>
  @if (showExtra()) { <app-tab label="Extra">Extra body</app-tab> }
</app-tabs>
<p>{{ tabs.count() }} tabs</p>
```

Observed:

```text
initially        panels: ['Overview body', '(hidden)']     count 2
click "Details"  panels: ['(hidden)', 'Details body']
showExtra → true tabs:   ['Overview', 'Details', 'Extra']  count 3
```

Notice the design: `Tabs` owns one piece of state (`selectedIndex`) and derives
`selectedTab`; each `Tab` **injects its parent** (a component can inject any ancestor
component) and derives its own `active` with `computed`. No effects, no manual syncing,
no lifecycle hooks — and adding a tab later just works, because `contentChildren` is a
signal and everything downstream recomputes.

## Template outlets: let the caller render each item

```ts title="src/app/data-list.ts"
@Component({
  selector: 'app-data-list',
  imports: [NgTemplateOutlet, JsonPipe],
  template: `
    <ul>
      @for (item of items(); track $index) {
        <li>
          <ng-container
            [ngTemplateOutlet]="itemTemplate() ?? defaultItem"
            [ngTemplateOutletContext]="{ $implicit: item, index: $index }" />
        </li>
      } @empty {
        <li>Nothing here.</li>
      }
    </ul>
    <ng-template #defaultItem let-item>{{ item | json }}</ng-template>
  `,
})
export class DataList<T> {
  readonly items = input.required<T[]>();
  protected readonly itemTemplate = contentChild(TemplateRef<{ $implicit: T; index: number }>);
}
```

```html
<app-data-list [items]="people">
  <ng-template let-p let-i="index">{{ i + 1 }}. {{ p.name }} ({{ p.role }})</ng-template>
</app-data-list>
<app-data-list [items]="[{ a: 1 }]" />
```

Rendered rows: `1. Ada (admin)`, `2. Grace (dev)` for the first list; the second, with no
template, fell back to the JSON view.

`<ng-template>` markup is **not rendered** where it's written; it's a blueprint.
`contentChild(TemplateRef)` grabs it, and `ngTemplateOutlet` instantiates it for every
item with a **context** object. `$implicit` is what a bare `let-p` receives; other keys
are read with `let-i="index"`. The component decides *how many* and *where*; the caller
decides *what it looks like*. That's the pattern behind data tables, virtual scroll
viewports and autocomplete option lists.

## How It Actually Works

**Projection happens at compile time for the parent.** When the compiler compiles the
*caller's* template, the elements between `<app-card>` and `</app-card>` are created as
part of the **caller's** view, bound to the caller's data, and then distributed into the
card's slots by matching them against the card's `select` selectors (known from the
card's metadata). The card only decides *where* the nodes go.

Two consequences follow, and both surprise people:

1. **Projected content is created even if the slot is never shown.** Wrapping
   `<ng-content />` in `@if` hides it but doesn't stop the caller from instantiating it —
   components inside are still constructed and their bindings still run. We checked:
   a component projected into `@if (open()) { <ng-content /> }` with `open` false logged
   `Heavy constructed` even though it was not in the DOM. That's why the
   tabs above use `[hidden]` on each panel; if panel content is expensive, pass it as an
   `<ng-template>` instead and render it only when active (or use `@defer`, lesson 04).
2. **Projected content belongs to its author.** Its bindings, change detection and DI
   lookups (for `viewProviders`, Level 2 lesson 03) follow the *caller's* view tree, not
   the component it is projected into.

**Templates are instantiated at runtime.** `<ng-template>` compiles to a separate template
function and a `TemplateRef`. `ngTemplateOutlet` calls
`ViewContainerRef.createEmbeddedView(templateRef, context)` — the same mechanism the
control-flow blocks use internally — so each row is a real embedded view with its own
bindings, created lazily and as many times as needed.

**Queries** are compiled into instructions that collect matching nodes as views are
created and destroyed. Signal queries expose the result through a signal that is marked
dirty when the underlying list changes, so a `computed` over `tabs()` recomputes when a
tab is added by an `@if`.

## Common mistakes

- **Expecting `@if` around `<ng-content>` to prevent creation** of projected components.
- **Projecting the same content twice** (e.g. into a desktop and a mobile slot). It
  moves; use a template instead.
- **Reading queries in the constructor**, before the view exists.
- **Querying by CSS selector.** Queries match template reference names, component or
  directive types, or tokens — not arbitrary selectors.
- **Deep query needs.** `contentChildren` looks at direct content by default; pass
  `{ descendants: true }` when the items are nested inside other elements.

## Exercise

1. Build an `app-dialog` with `[dialog-title]`, default and `[dialog-actions]` slots,
   and a fallback title "Dialog".
2. Build an `app-accordion` with `app-accordion-item` children, using the same
   inject-the-parent pattern as `Tabs`, so that only one item is open at a time. Add
   keyboard support (Up/Down arrows move focus between headers) using `viewChildren`.
3. Build `app-select<T>` that takes `options: T[]` and an optional `<ng-template>` for
   rendering each option, falling back to `String(option)`.
4. Put a component with a `console.log` in its constructor inside a tab that is
   never selected. Confirm it's still constructed, then change the tab to accept an
   `<ng-template>` and confirm it no longer is.
