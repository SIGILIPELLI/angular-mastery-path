---
description: "How Angular 22 decides when to update the DOM: zoneless scheduling, OnPush as the default, the Eager strategy, markForCheck and what does and doesn't trigger a render."
---

# 01 · Change Detection & Zoneless

**Change detection** is how Angular keeps the DOM in sync with your component state: it
walks component views, re-evaluates template bindings, and writes the ones that changed.
Two questions define it:

1. **When** does a change-detection pass run?
2. **Which** views does a pass actually check?

Angular's answers changed a lot in recent versions, and Angular 22 has new defaults for
both. If you learned Angular before about 2025, some of your instincts are now wrong —
this lesson shows exactly which, with behaviour observed in a real browser.

## The defaults in Angular 22

- **Zoneless.** New applications don't load Zone.js. In our test app,
  `typeof Zone` in the browser console was `undefined`.
- **`OnPush` by default.** The type definitions for `ChangeDetectionStrategy` in 22 say
  *"NOTE: OnPush is enabled by default."* The old always-check behaviour is now called
  `ChangeDetectionStrategy.Eager` (`Default` remains as a deprecated alias for it).

## When does a pass run? (the "when")

With Zone.js, Angular patched `setTimeout`, promises, XHR, DOM events and more, and ran
change detection after *any* of them finished — whether or not anything changed. Zoneless
Angular only schedules a pass when it is **told** something changed. The API documentation
for `provideZonelessChangeDetection` lists the notifications:

- updating a **signal that is read in a template**
- a bound **template or host listener** firing (`(click)`, `host: { '(keydown)': ... }`)
- `ChangeDetectorRef.markForCheck()`
- `ComponentRef.setInput(...)`
- attaching a view that was marked dirty, or removing a view
- registering a render hook (which then must do one of the above)

Plain assignments to fields are **not** on the list.

## Which views are checked? (the "which")

An `OnPush` component is only refreshed during a pass if it was **marked dirty**, which
happens when:

- a signal its template reads changes,
- one of its inputs receives a **new value** (compared with `Object.is`),
- an event listener in its template or host fires,
- `markForCheck()` is called on it (which also marks its ancestors).

An `Eager` component is refreshed on every pass that reaches it.

## What we observed

We built a lab component (OnPush by default) with a plain field, a signal, an OnPush
child and an Eager child — both children receive the same `profile` object input — and
clicked buttons in Chromium, reading the DOM 150 ms after each click:

```ts title="src/app/cd-lab.ts (excerpt)"
plainCount = 0;
readonly sigCount = signal(0);
profile: Profile = { name: 'Ada' };

laterPlain()    { setTimeout(() => this.plainCount++, 0); }
laterSignal()   { setTimeout(() => this.sigCount.update((n) => n + 1), 0); }
laterMarked()   { setTimeout(() => { this.plainCount++; this.cdr.markForCheck(); }, 0); }
mutateProfile() { this.profile.name = 'Grace'; }           // same object
replaceProfile(){ this.profile = { name: 'Linus' }; }      // new object
```

```text
action                          plain  sig  OnPush child  Eager child
initial                         0      0    Ada           Ada
plain field in setTimeout       0      0    Ada           Ada
signal in setTimeout            1      1    Ada           Ada
plain + markForCheck            2      1    Ada           Ada
mutate input object (click)     2      1    Ada           Grace
replace input object (click)    2      1    Linus         Linus
```

Reading it row by row:

- **Plain field in `setTimeout`:** the value changed to 1 in memory, but nothing told
  Angular. The screen still said 0. (With Zone.js this would have updated — this is the
  single most common surprise when moving to zoneless.)
- **Signal in `setTimeout`:** the signal notified Angular, the view refreshed — and the
  stale plain field was picked up *by accident* in the same pass, showing 1. Bugs of the
  first kind often hide behind this: something else refreshes the view, so it "usually
  works".
- **`markForCheck()`:** explicitly marking the view works for non-signal state.
- **Mutating the input object:** the click marked the parent dirty, so the parent
  re-evaluated `[profile]="profile"` — but it's the **same object**, so the OnPush child's
  input didn't change and the child was skipped. The Eager child was checked anyway and
  showed "Grace".
- **Replacing the object:** a new reference, so both children updated.

## The practical rules

1. **Keep template state in signals** (or in things that become signals: inputs,
   `toSignal`, resources, forms). Then "when" and "which" take care of themselves.
2. **Replace, don't mutate** objects and arrays passed as inputs or held in signals.
3. If you must change non-signal state asynchronously (a third-party callback, a
   `setTimeout`), call `markForCheck()` — or better, write it into a signal.
4. Use `Eager` only as a migration crutch for components that rely on mutation.

## Zone.js apps

Many existing apps still use Zone.js. They include `zone.js` in `polyfills` and either
rely on older defaults or call `provideZoneChangeDetection({ eventCoalescing: true })`.
Everything in this lesson about *which* views are checked still applies; the difference is
*when*: Zone.js triggers a pass after every async task. Migrating means removing
`zone.js`, adding `provideZonelessChangeDetection()` if needed, and fixing the places that
relied on async tasks to trigger rendering — the "plain field in `setTimeout`" row is
the pattern to look for. `NgZone.onStable`/`onMicrotaskEmpty` stop firing without Zone.js,
so code using them must move to `afterNextRender`/`afterEveryRender`.

## How It Actually Works

Every component view has a set of **flags** — "dirty", "refresh view", "has signal
consumers that changed" — and the view tree mirrors the component tree.

**Scheduling.** Each notification in the list above calls into an internal
*change-detection scheduler*. The first notification schedules a single `tick()` (racing
`setTimeout` against `requestAnimationFrame`, so several notifications in the same turn
coalesce into one pass); later notifications before it runs are no-ops. That's why ten `set()` calls
cause one render, and why the DOM doesn't update synchronously in the click handler — we
had to wait before reading it in the test.

**Traversal.** `ApplicationRef.tick()` walks from the root view. For each component view
it decides:

- `Eager` → refresh this view's bindings, then visit children.
- `OnPush` and marked dirty → refresh, then visit children.
- `OnPush`, not dirty, but a **descendant** had a signal change → skip this view's own
  bindings and continue down to the descendant (signal changes mark the path to the root
  for *traversal* only, not for refreshing every ancestor).
- otherwise → skip the whole subtree.

Refreshing a view means running its template function in update mode (Level 1,
lesson 01), which compares each bound value with the stored previous value. Setting a
component input during that parent refresh is what marks an OnPush child dirty — but only
when `Object.is(previous, next)` is false, which is why mutation was invisible to it.

**Development checks.** In dev mode, after each `tick()` Angular runs a second
"checkNoChanges" pass and throws `NG0100: ExpressionChangedAfterItHasBeenCheckedError`
if a binding produced a different value the second time — a sign that rendering itself
changed state. By default that pass skips `OnPush` views that aren't dirty — which, in
Angular 22, is most views. We bound `{{ random() }}` (returning `Math.random()`) and got:

```text
OnPush (default)                                   no error
OnPush + provideCheckNoChangesConfig({ exhaustive: true })   NG0100: ExpressionChangedAfterItHasBeenCheckedError
Eager                                              NG0100: ExpressionChangedAfterItHasBeenCheckedError
```

Add `provideCheckNoChangesConfig({ exhaustive: true })` to your **development** or test
configuration to catch these bugs in OnPush components too.

The scheduler detail above comes from the framework source: in Angular 22 the zoneless
scheduler races a `setTimeout` against a `requestAnimationFrame` and runs the tick on
whichever fires first.

## Common mistakes

- **Relying on async callbacks to update plain fields** in a zoneless app.
- **Mutating inputs** and expecting OnPush children to notice.
- **Calling `detectChanges()` everywhere** to "fix" rendering. It hides the real problem
  and can run far more often than needed. Prefer signals, then `markForCheck()`.
- **Making everything `Eager`** after an upgrade instead of fixing the few mutation sites.
- **Using `NgZone.run` / `onStable`** in new code; they do nothing useful without Zone.js.

## Exercise

1. Reproduce the lab: an OnPush component with a plain `count` field incremented in
   `setInterval` every second. Confirm the screen doesn't change. Fix it three ways —
   `markForCheck()`, a signal, and `toSignal(interval(1000))` — and explain which you'd
   ship.
2. Give a child component an array input and push to the array from the parent. Show that
   the child doesn't update; fix it by replacing the array.
3. Add `changeDetection: ChangeDetectionStrategy.Eager` to the child and observe the
   difference. Write down why this is a worse fix than replacing the array.
4. Trigger `NG0100` on purpose: bind `{{ random() }}` where `random()` returns
   `Math.random()` in a dev build. Confirm there's no error with default OnPush, then add
   `provideCheckNoChangesConfig({ exhaustive: true })` and read the error.
