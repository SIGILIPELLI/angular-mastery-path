---
description: "Pass data between Angular components with signal-based input(), input.required and transforms, emit events with output(), and two-way bind with model()."
---

# 06 · Inputs, Outputs & model()

Components talk to each other through a narrow, typed interface:

- **Inputs** carry data *down*, from a parent's template into a child.
- **Outputs** carry events *up*, from a child to whoever is listening.
- **Models** are both at once — an input the child may also write, giving two-way binding.

All three are declared with functions (`input()`, `output()`, `model()`) that the
compiler recognises. They replaced the older `@Input()` / `@Output()` decorators, which
still work and appear in lots of existing code.

## A reusable quantity picker

```ts title="src/app/quantity.ts"
import { Component, booleanAttribute, input, model, numberAttribute, output } from '@angular/core';

@Component({
  selector: 'app-quantity',
  template: `
    <button type="button" (click)="change(-1)" [disabled]="disabled() || value() <= min()">−</button>
    <span>{{ value() }}</span>
    <button type="button" (click)="change(1)" [disabled]="disabled() || value() >= max()">+</button>
  `,
})
export class Quantity {
  readonly value = model(1);
  readonly min = input(0, { transform: numberAttribute });
  readonly max = input.required({ transform: numberAttribute });
  readonly disabled = input(false, { transform: booleanAttribute });
  readonly limitReached = output<'min' | 'max'>();

  protected change(delta: number) {
    const next = this.value() + delta;
    this.value.set(next);
    if (next === this.max()) this.limitReached.emit('max');
    if (next === this.min()) this.limitReached.emit('min');
  }
}
```

And a parent using it:

```ts title="src/app/cart-line.ts"
import { Component, signal } from '@angular/core';
import { Quantity } from './quantity';

@Component({
  selector: 'app-cart-line',
  imports: [Quantity],
  template: `
    <app-quantity [(value)]="qty" max="3" (limitReached)="onLimit($event)" />
    <p>You are buying {{ qty() }}.</p>
    <app-quantity [value]="5" max="9" disabled />
  `,
})
export class CartLine {
  protected readonly qty = signal(2);
  protected onLimit(which: 'min' | 'max') {
    console.log(`hit the ${which}imum`);
  }
}
```

## Inputs

`input()` returns a **read-only signal**. Inside the child you read it like any signal —
`this.max()` — and can use it in `computed()`:

```ts
readonly price = input.required<number>();
readonly qty = input(1);
readonly lineTotal = computed(() => this.price() * this.qty());
```

- `input(defaultValue)` — optional; the type is inferred from the default.
- `input<T>()` — optional with no default; the type is `T | undefined`.
- `input.required<T>()` — the parent **must** bind it. Forget it and the build fails:

```text
✘ [ERROR] NG8008: Required input 'max' from component Quantity must be specified.
```

- `input(value, { alias: 'label' })` — the public binding name differs from the property
  name. Use sparingly; it makes code harder to search.

The child cannot set its own inputs — they are owned by the parent. If the child needs a
local, editable copy that resets when the input changes, that's `linkedSignal`
(Level 3, lesson 03).

### Transforms

A static attribute such as `max="3"` passes the **string** `"3"`. Transforms convert
incoming values:

- `numberAttribute` — `"3"` → `3`.
- `booleanAttribute` — makes `disabled` (attribute present, empty string) mean `true`,
  and `disabled="false"` mean `false`, like native HTML boolean attributes.

In our test run the first picker (`disabled` not set) worked normally, while the second,
written as a bare `disabled`, had both its buttons disabled. You can pass any function as
a transform, but keep them pure and cheap.

## Outputs

`output<T>()` creates an emitter. Call `.emit(value)` in the child; the parent listens with
`(name)="handler($event)"`, where `$event` is the emitted value — not a DOM event.

```ts
readonly removed = output<void>();       // emit with no payload: this.removed.emit()
readonly rated = output<number>();       // this.rated.emit(4)
```

Outputs don't bubble through the DOM. Only the direct parent binding the output hears
it. If you need to broadcast to unrelated components, use a shared service (lesson 07).

To expose an RxJS stream as an output, `outputFromObservable(stream$)` from
`@angular/core/rxjs-interop` does the wiring (Level 2, lesson 05).

## model(): two-way binding

`model()` declares a writable signal that is also an input. When the child calls
`this.value.set(...)`, Angular emits a matching `valueChange` output. The "banana in a
box" syntax `[(value)]="qty"` is shorthand for binding both:

```html
<app-quantity [(value)]="qty" />
<!-- is equivalent to -->
<app-quantity [value]="qty()" (valueChange)="qty.set($event)" />
```

When the right-hand side of `[( )]` is a writable signal, you pass the signal itself (no
parentheses) and Angular keeps it in sync. We clicked **+** once on a picker bound to
`qty = signal(2)` with `max="3"`; afterwards `qty()` was `3`, the paragraph showed `3`,
the `+` button was disabled, and `limitReached` had fired with `"max"`.

Use `model()` for genuinely two-way values — a picker's value, a dialog's `open` state, a
tab component's active index. For everything else, one input plus one output keeps the
data flow easier to follow.

## Setting inputs from code

Sometimes you create a component yourself — in tests, or dynamically. Use
`ComponentRef.setInput`, which goes through the same machinery as a template binding
(including transforms):

```ts
const fixture = TestBed.createComponent(Quantity);
fixture.componentRef.setInput('max', '4');   // transform turns '4' into 4
fixture.componentRef.setInput('value', 4);
await fixture.whenStable();
// buttons: [false, true] — "+" disabled at the max
```

Creating it *without* the required input and rendering fails at runtime with
`NG0950: Input "max" is required but no value is available yet.`

## How It Actually Works

The compiler treats `input()`, `model()` and `output()` calls in field initialisers as
declarations. When it compiles `Quantity`, it records in the component definition that
`max` is a *signal input* (with its transform) and `limitReached` an output, and it
generates the type-checking code that makes `[max]="'abc'"` or a missing required input a
build error.

At runtime, an input signal is a special signal node whose value is set by the
framework. When the parent's template runs in update mode and the expression bound to
`[max]` produces a new value, Angular runs the transform and writes the result into the
child's input node. Because it is a signal, anything in the child that read `max()` — its
template, its computeds — is marked dirty through the normal signal graph (lesson 04).
There is no `ngOnChanges` needed, although it still fires for decorator-based inputs.

`output()` returns an `OutputEmitterRef`. The parent's `(limitReached)="..."` compiles to
a subscription on that ref created alongside the child; `emit` calls the listeners
synchronously. `model()` is an input node plus an output named `<name>Change`, and
calling `set` on it both updates the local value and emits. The `[(value)]` syntax is
purely a compile-time expansion into the two bindings shown above.

## Common mistakes

- **Trying to `set` an input.** `input()` returns `InputSignal`, which has no `set`. Use
  `model()` if the child really owns part of the value, or emit an output.
- **Passing a string where a number is expected** (`max="3"` without a transform, or
  `[max]="'3'"`). Add `numberAttribute` or bind a number: `[max]="3"`.
- **Two-way binding to a plain property** in new code. `[(value)]="qty"` with `qty` a
  signal keeps everything reactive; with a plain field, the parent's template may not
  refresh under `OnPush`.
- **Naming an output after a native DOM event** (`click`, `change`). The parent's
  `(change)` handler would receive both your emissions and bubbling native events.
  Choose specific names such as `statusChange`.
- **Mutating an object you received as an input.** The parent still holds the same
  reference and won't know it changed. Emit the new value instead.

## Exercise

Build a `ToggleSwitch` component:

1. `checked = model(false)` and `label = input.required<string>()`.
2. `disabled = input(false, { transform: booleanAttribute })`.
3. Render a `<button role="switch" [attr.aria-checked]="checked()">` that flips
   `checked` on click unless disabled.
4. Add an output `turnedOn = output<void>()` that emits only on the false → true transition.
5. Use it in a parent with `[(checked)]="darkMode"` where `darkMode` is a signal, and show
   "Dark mode is on/off" underneath. Add a second, `disabled` switch and confirm it can't be
   toggled.
6. Remove the `label` binding from one usage and read the build error.
