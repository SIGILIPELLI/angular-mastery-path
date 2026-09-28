---
description: "Build typed reactive forms in Angular: FormBuilder, built-in and custom validators, cross-field group validation, FormArray, status and submit handling."
---

# 01 · Reactive Forms

Forms are where most business apps spend their complexity: validation rules, fields that
depend on each other, repeating sections, server errors, disabled states. Angular has
three form systems:

| System | Model lives in | Best for |
|--------|----------------|----------|
| **Template-driven** (`FormsModule`, `ngModel`) | The template | Very small forms; older code |
| **Reactive** (`ReactiveFormsModule`) | A tree of `FormControl` / `FormGroup` objects in the class | Most existing production code; complex dynamic forms |
| **Signal forms** (`@angular/forms/signals`) | A signal holding your data model | New code in Angular 22+ (next lesson) |

Reactive forms have been the workhorse of Angular apps for years, so you *will* meet them
— in almost every existing codebase and many job interviews. This lesson teaches them
properly; the next lesson shows the signal-based successor, which became stable in
Angular 22.

## A complete sign-up form

```ts title="src/app/signup-form.ts"
import { Component, inject, signal } from '@angular/core';
import { AbstractControl, NonNullableFormBuilder, ReactiveFormsModule, ValidationErrors, ValidatorFn, Validators } from '@angular/forms';
import { toSignal } from '@angular/core/rxjs-interop';

export const passwordsMatch: ValidatorFn = (group: AbstractControl): ValidationErrors | null => {
  const pw = group.get('password')?.value;
  const confirm = group.get('confirm')?.value;
  return pw && confirm && pw !== confirm ? { passwordsMismatch: true } : null;
};

export function forbiddenWords(words: string[]): ValidatorFn {
  return (control) => {
    const v = String(control.value ?? '').toLowerCase();
    const hit = words.find((w) => v.includes(w));
    return hit ? { forbidden: { word: hit } } : null;
  };
}

@Component({
  selector: 'app-signup-form',
  imports: [ReactiveFormsModule],
  template: `
    <form [formGroup]="form" (ngSubmit)="submit()">
      <label>Username <input formControlName="username" autocomplete="username" /></label>
      @if (form.controls.username.touched && form.controls.username.errors; as e) {
        @if (e['required']) { <p class="err">Username is required.</p> }
        @if (e['minlength']) { <p class="err">At least {{ e['minlength'].requiredLength }} characters.</p> }
        @if (e['forbidden']) { <p class="err">“{{ e['forbidden'].word }}” is not allowed.</p> }
      }

      <label>Email <input formControlName="email" type="email" autocomplete="email" /></label>

      <fieldset formGroupName="passwords">
        <label>Password <input formControlName="password" type="password" autocomplete="new-password" /></label>
        <label>Confirm <input formControlName="confirm" type="password" autocomplete="new-password" /></label>
        @if (form.controls.passwords.errors?.['passwordsMismatch']) { <p class="err">Passwords don't match.</p> }
      </fieldset>

      <fieldset>
        <legend>Interests</legend>
        @for (ctrl of interests.controls; track ctrl; let i = $index) {
          <input [formControl]="ctrl" [attr.aria-label]="'Interest ' + (i + 1)" />
          <button type="button" (click)="interests.removeAt(i)">✕</button>
        }
        <button type="button" (click)="addInterest()">Add interest</button>
      </fieldset>

      <label><input type="checkbox" formControlName="terms" /> I accept the terms</label>

      <button [disabled]="form.invalid || submitting()">Sign up</button>
      <p>Status: {{ status() }}</p>
    </form>
  `,
})
export class SignupForm {
  private readonly fb = inject(NonNullableFormBuilder);
  protected readonly submitting = signal(false);
  readonly submitted = signal<unknown>(null);

  readonly form = this.fb.group({
    username: ['', [Validators.required, Validators.minLength(3), forbiddenWords(['admin', 'root'])]],
    email: ['', [Validators.required, Validators.email]],
    passwords: this.fb.group(
      { password: ['', [Validators.required, Validators.minLength(8)]], confirm: [''] },
      { validators: passwordsMatch },
    ),
    interests: this.fb.array([this.fb.control('')]),
    terms: [false, Validators.requiredTrue],
  });

  protected readonly status = toSignal(this.form.statusChanges, { initialValue: this.form.status });

  protected get interests() {
    return this.form.controls.interests;
  }

  protected addInterest() {
    this.interests.push(this.fb.control(''));
  }

  protected submit() {
    if (this.form.invalid) {
      this.form.markAllAsTouched();
      return;
    }
    this.submitted.set(this.form.getRawValue());
    this.form.reset();
  }
}
```

## Building the model

`NonNullableFormBuilder` creates controls whose `reset()` goes back to their initial value
instead of `null`. The array shorthand `['', [validators]]` means *initial value, sync
validators*. The result is **fully typed**: `form.value` is inferred as

```ts
Partial<{
  username: string; email: string;
  passwords: Partial<{ password: string; confirm: string }>;
  interests: string[]; terms: boolean;
}>
```

It is `Partial` because disabled controls are left out of `value`. Use
`form.getRawValue()` when you want every field regardless of disabled state — it returns
the non-partial type.

Binding is done with directives from `ReactiveFormsModule`:

- `[formGroup]="form"` on the `<form>`.
- `formControlName="username"` on an input inside it; `formGroupName="passwords"` on a
  container for a nested group.
- `[formControl]="ctrl"` binds a standalone control object — handy inside `@for` over a
  `FormArray`.
- `(ngSubmit)` fires on submit and prevents the browser's page reload.

## Validators

Built-in validators live on `Validators`: `required`, `requiredTrue` (for "accept terms"
checkboxes), `email`, `minLength`, `maxLength`, `min`, `max`, `pattern`.

A **custom validator** is just a function from a control to either `null` (valid) or an
errors object:

```ts
export function forbiddenWords(words: string[]): ValidatorFn {
  return (control) => {
    const v = String(control.value ?? '').toLowerCase();
    const hit = words.find((w) => v.includes(w));
    return hit ? { forbidden: { word: hit } } : null;
  };
}
```

The error key (`forbidden`) is yours to choose; its value can carry details for the
message. In our test, typing `Admin42` produced:

```text
username errors {"forbidden":{"word":"admin"}} | msg: “admin” is not allowed.
```

Notice `required` and `minlength` are gone from that object — errors are recomputed from
scratch on every change, and `Admin42` satisfies both.

### Cross-field validation

"Passwords must match" is not a property of either field alone, so the validator goes on
the **group**, where it can read both children. The error appears on the group, not on a
field:

```text
group errors {"passwordsMismatch":true} status INVALID
```

That's why the template checks `form.controls.passwords.errors`.

### Async validators

A third array element (or the `asyncValidators` option) takes validators that return a
`Promise` or `Observable` — typically a "username taken?" server call. While one is
pending the control's status is `PENDING`. Async validators run only after all sync
validators pass, which keeps you from hitting the server on every invalid keystroke.

## Showing errors at the right time

Showing "required" before the user has touched a field is noisy. The usual rule is to
show errors once the control is **touched** (blurred at least once) or after a submit
attempt. The submit handler calls `markAllAsTouched()` when the form is invalid, so every
missing field lights up at once.

## FormArray: repeating fields

`interests` is a `FormArray` — a list of controls whose length can change at runtime.
`push`, `removeAt`, `insert` and `clear` change it; `controls` is the array to iterate.
Track by the control object itself (`track ctrl`): controls are stable objects, so this
identity is exactly right, whereas `track $index` would mix up inputs when removing from
the middle (Level 1, lesson 05).

## Reacting to changes

Every control exposes `valueChanges` and `statusChanges` observables. To use them in a
template, turn them into signals:

```ts
protected readonly status = toSignal(this.form.statusChanges, { initialValue: this.form.status });
```

Recent versions also add a unified `events` observable (value, status, touched, pristine,
submit and reset events) on every control.

## How It Actually Works

A reactive form is a **tree of model objects** that exists independently of the DOM.
Each `FormControl` holds a value, a status (`VALID`, `INVALID`, `PENDING`, `DISABLED`),
errors, and `touched`/`dirty` flags. `FormGroup` and `FormArray` aggregate their
children.

When the user types, the `formControlName` directive — through a **ControlValueAccessor**
(CVA) for that element type — calls `control.setValue(newValue)`. Then
`updateValueAndValidity()` runs: the control's validators execute, its status is
computed, it emits on `valueChanges`/`statusChanges`, and it asks its **parent** to do the
same. The group runs its own validators (our `passwordsMatch`) and recomputes its status
from its children: invalid if any child is invalid, pending if any is pending, and so on
up to the root.

The CVA works in the other direction too: `setValue` or `patchValue` from code calls the
accessor's `writeValue`, which sets `input.value`. The CVA is the adapter that makes a
`<select>`, a checkbox, a date picker or your own custom component look the same to the
form model — you can implement the `ControlValueAccessor` interface to make any
component usable with `formControlName`.

Two details from our test run that follow from this design:

- After a successful submit, `reset()` restored every control to its initial value
  (`terms: false`, empty strings) but the `interests` array **still had two entries**.
  Reset changes values; it never adds or removes controls. To shrink the array, call
  `clear()` and push fresh controls.
- The model updates synchronously on each `input` event, but the template does not
  re-render until change detection runs. That is why the test awaits
  `fixture.whenStable()` between typing and asserting.

## Common mistakes

- **Mixing `ngModel` with `formControlName`** on the same element. It has been deprecated
  for years; pick one system.
- **Typing a getter as `FormArray`** (untyped). You lose the control types and the
  template stops compiling — we hit
  `TS2739: Type 'AbstractControl<any, any, any>' is missing the following properties from type 'FormControl<any>'`.
  Return `this.form.controls.interests` and let TypeScript infer it.
- **Disabling via the `disabled` attribute** on an input bound to a control. Call
  `control.disable()`; the template attribute is ignored and Angular warns.
- **Reading `form.value` and expecting disabled fields.** Use `getRawValue()`.
- **Trusting the client.** Validators improve UX; the server must validate again.

## Exercise

Build a "Shipping address" form:

1. Fields: `name`, `street`, `city`, `postcode`, `country` (a `<select>`), and an optional
   `phone`.
2. `postcode` must match `/^[0-9]{5}$/` when country is "DE" and `/^[A-Z0-9 ]{5,8}$/i`
   when it's "GB". Implement this as a **group** validator and show the error under the
   postcode field.
3. Add a "Use a different billing address" checkbox. When checked, enable a nested
   `billing` group; when unchecked, disable it (and confirm it disappears from
   `form.value` but not from `getRawValue()`).
4. Add a `FormArray` of delivery notes with add/remove buttons, and make `reset()` really
   reset the array to one empty note.
