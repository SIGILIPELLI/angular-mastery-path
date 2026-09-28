---
description: "Angular signal forms, stable in v22: form() over a model signal, schema validators, validate() for custom rules, [formField], disabled logic, arrays and submit()."
---

# 02 · Signal Forms

Reactive forms (previous lesson) keep a *second* copy of your data inside a tree of
`FormControl` objects, and you move data in and out of it with `setValue`, `patchValue`
and `getRawValue`. **Signal forms** flip that around: your data stays in an ordinary
signal, and the form is a typed *view* over it that adds validation, touched/dirty state
and binding. They live in `@angular/forms/signals` and were marked stable
(`@publicApi 22.0`) in Angular 22.

## The model is just a signal

```ts
interface Signup {
  username: string;
  email: string;
  password: string;
  confirm: string;
  age: number | null;
  interests: string[];
  terms: boolean;
}

readonly model = signal<Signup>({
  username: '', email: '', password: '', confirm: '', age: null, interests: [''], terms: false,
});

readonly f = form(this.model, (p) => { /* rules, see below */ });
```

`form(model, schema)` returns a **field tree** shaped like your data: `f.username`,
`f.interests[0]`, and so on. Each node is callable — `f.username()` returns that field's
**state**, an object of signals:

| Signal | Meaning |
|--------|---------|
| `value` | The field's value (writable — `f.username().value.set('ada')` updates the model) |
| `errors` | Array of `{ kind, message? }` objects for this field |
| `valid` / `invalid` / `pending` | Validation status, `pending` while async rules run |
| `touched` / `dirty` | Interaction state |
| `disabled` / `readonly` / `hidden` | Computed from your schema rules |
| `required`, `min`, `max`, `minLength`, `maxLength` | Metadata derived from validators, useful for hints and native attributes |
| `submitting` | True while a `submit()` action runs |

`f()` — the root — aggregates all of them: `f().valid()` is true only when every field is.

## A complete example

```ts title="src/app/signup-signal-form.ts"
import { Component, signal } from '@angular/core';
import {
  FormField, applyEach, disabled, email, form, maxLength, min, minLength, required, submit, validate,
} from '@angular/forms/signals';

interface Signup {
  username: string;
  email: string;
  password: string;
  confirm: string;
  age: number | null;
  interests: string[];
  terms: boolean;
}

const empty: Signup = {
  username: '', email: '', password: '', confirm: '', age: null, interests: [''], terms: false,
};

@Component({
  selector: 'app-signup-signal-form',
  imports: [FormField],
  template: `
    <form (submit)="onSubmit($event)" novalidate>
      <label>Username <input [formField]="f.username" autocomplete="username" /></label>
      @if (f.username().touched()) {
        @for (e of f.username().errors(); track e.kind) { <p class="err">{{ e.message }}</p> }
      }

      <label>Email <input type="email" [formField]="f.email" /></label>
      <label>Age <input type="number" [formField]="f.age" /></label>

      <label>Password <input type="password" [formField]="f.password" /></label>
      <label>Confirm <input type="password" [formField]="f.confirm" /></label>
      @for (e of f.confirm().errors(); track e.kind) { <p class="err">{{ e.message }}</p> }

      <fieldset>
        <legend>Interests</legend>
        @for (item of f.interests; track item; let i = $index) {
          <input [formField]="item" [attr.aria-label]="'Interest ' + (i + 1)" />
        }
        <button type="button" (click)="addInterest()">Add interest</button>
      </fieldset>

      <label><input type="checkbox" [formField]="f.terms" /> I accept the terms</label>

      <button [disabled]="f().submitting()">Sign up</button>
      <p>Valid: {{ f().valid() }}</p>
    </form>
  `,
})
export class SignupSignalForm {
  readonly model = signal<Signup>({ ...empty });
  readonly saved = signal<Signup | null>(null);

  readonly f = form(this.model, (p) => {
    required(p.username, { message: 'Username is required.' });
    minLength(p.username, 3, { message: 'At least 3 characters.' });
    validate(p.username, ({ value }) =>
      /admin|root/i.test(value()) ? { kind: 'forbidden', message: 'That name is reserved.' } : null,
    );
    required(p.email, { message: 'Email is required.' });
    email(p.email, { message: 'Enter a valid email.' });
    min(p.age, 13, { message: 'You must be 13 or older.' });
    required(p.password);
    minLength(p.password, 8, { message: 'Use at least 8 characters.' });
    disabled(p.confirm, ({ valueOf }) => valueOf(p.password) === '');
    validate(p.confirm, ({ value, valueOf }) =>
      value() !== valueOf(p.password) ? { kind: 'mismatch', message: "Passwords don't match." } : null,
    );
    applyEach(p.interests, (item) => maxLength(item, 20, { message: 'Keep it under 20 characters.' }));
    required(p.terms, { message: 'Please accept the terms.' });
  });

  protected addInterest() {
    this.model.update((m) => ({ ...m, interests: [...m.interests, ''] }));
  }

  protected async onSubmit(event: Event) {
    event.preventDefault();
    await submit(this.f, {
      action: async () => {
        await new Promise((r) => setTimeout(r, 10)); // stand-in for an HTTP call
        if (this.model().username === 'taken') {
          return [{ fieldTree: this.f.username, kind: 'server', message: 'Username already taken.' }];
        }
        this.saved.set(this.model());
        return undefined;
      },
    });
  }
}
```

## Binding: `[formField]`

Import the `FormField` directive and bind a field to any native control:
`<input [formField]="f.username" />`. The directive keeps the element and the model in
sync both ways, marks the field touched on blur, and also sets native attributes it can
derive — `disabled`, `required`, `min`, `max` and so on. Here is what three of the
inputs in the example looked like in the DOM (Angular's scoping attributes removed):

```html
<input autocomplete="username" minlength="3" name="a.form1.username" required="">
<input type="number" min="13" name="a.form1.age">
<input type="checkbox" name="a.form1.terms" required="">
```

The `minlength`, `min` and `required` attributes came from the schema rules, not from the
template, and each control also received a generated `name`.

It converts types for you. In our test, typing `12` into `<input type="number">` stored
the **number** `12` in the model (with a `min` error), and clearing the input stored
`null` — no manual `Number(...)` parsing. A checkbox binds to a `boolean`.

## Schema rules

The second argument to `form()` is a **schema function**. It receives `p`, a tree of
*paths* mirroring your model, and you attach rules to paths:

- **Built-in validators:** `required`, `email`, `min`, `max`, `minLength`, `maxLength`,
  `pattern`, `minDate`, `maxDate`. Each accepts `{ message }`, and `required` accepts a
  `when` function to make it conditional. Per the API docs, `required` treats `''`,
  `null`, `undefined`, `false` and `NaN` as empty — so `required(p.terms)` means "must
  be checked".
- **Custom rules with `validate`:** a function receiving a *field context* and returning
  `null` (or `undefined`/`[]`) for valid, or one or more `{ kind, message }` errors.
  `ctx.value()` is this field's value; `ctx.valueOf(otherPath)` reads another field, which
  is how the "passwords match" rule is written directly on `confirm`.
- **State logic:** `disabled(path, fn)`, `readonly(path, fn)` and `hidden(path, fn)`
  compute field state from the model. Above, `confirm` is disabled while `password` is
  empty. We saw exactly that: initially `f.confirm().disabled()` was `true` and the DOM
  input was `disabled`; after setting a password it was enabled.
- **Arrays:** `applyEach(p.interests, (item) => ...)` applies rules to every element,
  including ones added later.
- **Async:** `validateAsync` (resource-based) and `validateHttp` (an `httpResource`
  request per value, with optional `debounce`) run only after the field's synchronous
  rules pass.
- **Reuse:** `schema(fn)` wraps a set of rules so you can `apply(p.address, addressSchema)`
  in several forms.

Because rules are *reactive functions*, they re-evaluate automatically whenever the
signals they read change — there is no `updateValueAndValidity()` to remember.

## Submitting

`submit(form, { action })`:

1. Marks every field touched (so errors appear).
2. If the form is invalid, stops and resolves `false` (the optional `onInvalid` callback
   runs).
3. Otherwise sets `submitting()` to `true`, awaits your `action`, and resolves.

The action can return **server errors**, each targeted at a field. From our test:

```text
submitting during true
submit result false username errors [["server","Username already taken."]]
server error after edit []
```

The server error showed up on `username` alongside any client errors, and it cleared as
soon as the user edited that field — the behaviour you usually want for "already taken"
messages. After fixing the value, a second submit stored the model:

```text
saved {"username":"grace","email":"ada@example.com","password":"correct-horse",
       "confirm":"correct-horse","age":36,"interests":["chess",""],"terms":true}
```

An invalid submit (email `bad`) did not call the action; it left `saved` as `null` and
marked the email field touched with an `email` error.

## Reactive forms or signal forms?

- **New forms in an Angular 22+ app:** signal forms. Less code, the model is plain data,
  validation is declarative, and everything composes with `computed` and `effect`.
- **Existing reactive forms:** leave them. They are not deprecated, and rewriting a working
  form is rarely worth it. For gradual adoption the package includes interop helpers
  (`@angular/forms/signals/compat`) for using existing `FormControl`s inside signal forms.
- **Custom form controls:** reactive forms need a `ControlValueAccessor`. With signal
  forms, a component takes part by implementing the `FormValueControl<T>` interface —
  essentially a `value` `model()` plus optional inputs such as `disabled`, `errors` and
  `touched` (or `FormCheckboxControl` with a `checked` model for boolean controls).

## How It Actually Works

A signal form is a **graph of computed signals over one source of truth**. The model
signal holds the data. For each path, the field tree lazily creates a node whose `value`
is a *derived writable signal*: reading it reads `model().username`, and writing it does
an immutable update of the model (a new root object with the changed property). There is
no copy of the data to keep in sync, which is why `this.model.set({...})` in our test
immediately showed up in the inputs, including the confirm input's value `nope`.

Each rule you register in the schema becomes a computed signal attached to that node.
`errors` is a computed that runs all validators for the field and concatenates their
results; `valid` is computed from the field's own errors plus its children's validity;
`disabled` is computed from the `disabled` rules and the parent's disabled state. Since
validators read the value through signals, the dependency tracking from Level 1 lesson 04
decides what to re-run: editing `password` re-evaluates `confirm`'s `mismatch` rule
(which read `valueOf(p.password)`), but not `email`'s rules.

The schema function itself runs **once**, when the form is created, to register rules
against paths — it is not re-run on every change. That's why the rule bodies are
functions of a context (`({ value }) => ...`) rather than code that reads values directly.

The `[formField]` directive is the only part that touches the DOM: it listens to
`input`/`change`/`blur` events and writes into the field's `value` signal, and it uses an
`effect`-style binding to write the field's value and state back to the element.

## Common mistakes

- **Mutating the model object.** `this.model().interests.push('')` changes nothing
  reactive. Use `model.update(...)` with a new object, or write through the field:
  `f.interests().value.update((xs) => [...xs, ''])`.
- **Reading values in the schema body.** `if (p.age...)` inside the schema function runs
  once. Put conditions inside rule functions or use `applyWhen`/`required(..., { when })`.
- **Forgetting `novalidate` and `preventDefault`.** Native browser validation bubbles
  and page reloads will fight your form.
- **Showing errors immediately.** Gate messages on `touched()` (and rely on `submit()`
  marking everything touched).
- **Tracking array items by `$index`.** Track the field node itself (`track item`) so
  removing an element in the middle doesn't shuffle inputs.

## Exercise

Rebuild the shipping-address form from lesson 01 with signal forms:

1. A `signal<Address>` model with `name`, `street`, `city`, `postcode`, `country`,
   `phone` and `billingSame: boolean` plus an optional nested `billing` address.
2. A reusable `addressSchema` created with `schema(...)` and applied to both the shipping
   fields and `p.billing`.
3. `hidden(p.billing, ...)` when `billingSame` is true, and render the billing fieldset
   only when it's not hidden.
4. A `validate` rule on `postcode` that depends on `country` via `valueOf`.
5. A `submit` action that returns a server error on `postcode` when it is `00000`, and
   check that the error clears after editing the field.
