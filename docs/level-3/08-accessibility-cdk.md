---
description: "Build accessible Angular apps: semantic HTML first, @angular/aria headless tabs with keyboard support, CDK focus trapping, live announcements and focus management."
---

# 08 · Accessibility & the CDK

An accessible app works for people using screen readers, keyboards, switch devices,
voice control and magnification. It is also a legal requirement in many places and, in
practice, a quality bar: the things that break for assistive technology — unlabeled
buttons, focus that disappears, custom widgets that only respond to a mouse — are bugs
for everyone.

Angular helps in three layers:

1. **Your templates.** Most accessibility comes from plain, semantic HTML and correct
   labels. No library fixes a `<div (click)>` pretending to be a button.
2. **`@angular/aria`** — headless directives (no styles) that implement the WAI-ARIA
   patterns for complex widgets: accordion, combobox, grid, listbox, menu, tabs, toolbar
   and tree. Version 22.2.0 ships all of those as separate entry points
   (`@angular/aria/tabs`, `@angular/aria/menu`, …).
3. **The Component Dev Kit (`@angular/cdk/a11y`)** — lower-level tools: focus trapping,
   live announcements, focus monitoring and keyboard list navigation.

## Layer 1: semantic HTML

- Use `<button>` for actions and `<a href>` (or `routerLink`) for navigation. Both are
  focusable and keyboard-operable for free.
- Every form control needs a label: wrap it in `<label>`, or use `for`/`id`,
  or `aria-label` when there's no visible text.
- One `<h1>` per page and a logical heading order; landmarks (`<header>`, `<nav>`,
  `<main>`) so screen-reader users can jump around.
- Text alternatives: `alt` on images (empty `alt=""` for decorative ones).
- Don't rely on colour alone to convey state; keep visible focus outlines.

The course's earlier components follow these rules — the reading list's rating control
(Level 1, lesson 10) is a `radiogroup` of labelled buttons, and forms show errors with
`role="alert"`.

## Layer 2: `@angular/aria` — a tab set done properly

The hand-made tabs in lesson 02 were fine for teaching composition, but a real tab
widget has a detailed keyboard contract: arrow keys move between tabs, Home/End jump to
the ends, only one tab is in the Tab order, and panels are linked to their tabs. The
`@angular/aria` directives implement that contract; you supply the markup and styles.

```ts title="src/app/account-settings.ts"
import { Component, ElementRef, inject, signal, viewChild } from '@angular/core';
import { Tab, TabContent, TabList, TabPanel, Tabs } from '@angular/aria/tabs';
import { A11yModule, LiveAnnouncer } from '@angular/cdk/a11y';

@Component({
  selector: 'app-a11y-lab',
  imports: [Tabs, TabList, Tab, TabPanel, TabContent, A11yModule],
  template: `
    <h1>Account</h1>

    <div ngTabs>
      <ul ngTabList [(selectedTab)]="tab" aria-label="Account sections">
        <li ngTab value="profile">Profile</li>
        <li ngTab value="security">Security</li>
        <li ngTab value="billing">Billing</li>
      </ul>
      <div ngTabPanel value="profile"><ng-template ngTabContent>Profile settings</ng-template></div>
      <div ngTabPanel value="security"><ng-template ngTabContent>Security settings</ng-template></div>
      <div ngTabPanel value="billing"><ng-template ngTabContent>Billing settings</ng-template></div>
    </div>

    <button #opener type="button" (click)="open.set(true)">Delete account…</button>

    @if (open()) {
      <div class="backdrop">
        <div role="dialog" aria-modal="true" aria-labelledby="dlg-title" cdkTrapFocus [cdkTrapFocusAutoCapture]="true"
             (keydown.escape)="close()">
          <h2 id="dlg-title">Delete account?</h2>
          <p>This cannot be undone.</p>
          <button type="button" (click)="close()">Cancel</button>
          <button type="button" (click)="confirm()">Delete</button>
        </div>
      </div>
    }
  `,
})
export class A11yLab {
  protected readonly tab = signal('profile');
  protected readonly open = signal(false);
  private readonly announcer = inject(LiveAnnouncer);
  private readonly opener = viewChild.required<ElementRef<HTMLButtonElement>>('opener');

  protected close() {
    this.open.set(false);
    this.opener().nativeElement.focus(); // return focus to where the user was
  }

  protected confirm() {
    this.close();
    this.announcer.announce('Account scheduled for deletion.', 'polite');
  }
}
```

## What the directives produced

We rendered this in Chromium and inspected the DOM. The tab list:

```html
<ul role="tablist" aria-label="Account sections" tabindex="-1" aria-orientation="horizontal">
  <li role="tab" value="profile" id="ng-tab-a19769-0" tabindex="0" aria-selected="true"
      aria-controls="ng-tabpanel-a19769-0">Profile</li>
  <li role="tab" value="security" id="ng-tab-a19769-1" tabindex="-1" aria-selected="false"
      aria-controls="ng-tabpanel-a19769-1">Security</li>
  <li role="tab" value="billing" id="ng-tab-a19769-2" tabindex="-1" aria-selected="false"
      aria-controls="ng-tabpanel-a19769-2">Billing</li>
</ul>
```

and the panels:

```html
<div role="tabpanel" value="profile" id="ng-tabpanel-a19769-0" tabindex="0"
     aria-labelledby="ng-tab-a19769-0">Profile settings</div>
<div role="tabpanel" value="security" id="ng-tabpanel-a19769-1" tabindex="-1" inert="true"
     aria-labelledby="ng-tab-a19769-1"></div>
<div role="tabpanel" value="billing" id="ng-tabpanel-a19769-2" tabindex="-1" inert="true"
     aria-labelledby="ng-tab-a19769-2"></div>
```

Every role, generated ID and `aria-controls`/`aria-labelledby` link was filled in for
us. Inactive panels are `inert` (hidden from assistive technology and not focusable) and
their `ngTabContent` templates weren't rendered at all — the content is created lazily.

Then we drove it with the keyboard:

```text
click Profile, press ArrowRight   focus: Security   aria-selected: [false, true, false]   visible: "Security settings"
press End                         focus: Billing
press Enter                       aria-selected: [false, false, true]  visible: "Billing settings"
```

This is the **roving tabindex** pattern: only the active tab has `tabindex="0"`, so the
whole tab list is a single stop in the page's Tab order, and arrow keys move within it.
By default, moving focus also selects the tab ("selection follows focus"); set
`selectionMode="explicit"` on the tab list to require Enter/Space instead — better when
switching tabs is expensive.

## Layer 3: CDK tools

### Focus trapping for dialogs

A modal dialog must keep keyboard focus inside itself while open, and return focus to the
element that opened it when it closes. `cdkTrapFocus` (from `A11yModule`) does the first
part; `cdkTrapFocusAutoCapture` moves focus into the dialog when it appears and restores
it on destroy; our `close()` also focuses the opener explicitly. Observed:

```text
dialog opened         focus: Cancel (first focusable element in the dialog)
pressed Tab 3 times   focus: Delete — still inside the dialog
pressed Escape        dialog removed, focus back on "Delete account…"
```

For production dialogs, prefer the CDK's `Dialog` service (`@angular/cdk/dialog`) or
Angular Material's `MatDialog`, which add the overlay, backdrop, `aria-modal`, scroll
blocking and focus restoration for you — the snippet above shows the mechanics.

### Announcing changes

Screen readers don't notice DOM changes unless they happen inside a **live region**.
`LiveAnnouncer.announce(message, 'polite' | 'assertive')` manages one for you. After
confirming the deletion, the page contained:

```text
aria-live="polite": "Account scheduled for deletion."
```

Use it for results of actions that don't move focus: "Item added to cart", "3 results",
"Saved".

### Other CDK a11y tools

- `FocusMonitor` — tells you whether focus came from keyboard, mouse, touch or program,
  so you can show focus rings only for keyboard users.
- `ListKeyManager` / `ActiveDescendantKeyManager` — arrow-key navigation for your own
  list widgets, if `@angular/aria` doesn't cover your pattern.
- `InteractivityChecker` — "is this element focusable / visible?".

## Focus and routing

In a single-page app, clicking a link doesn't reload the page, so screen-reader users
aren't told that anything changed and focus stays on the link. A common fix: after each
navigation, move focus to the new page's `<h1>` (give it `tabindex="-1"`) or to `<main>`,
and let the router set a meaningful `title` for every route (Level 1, lesson 08).

## How It Actually Works

`@angular/aria` separates **behaviour** from **rendering**. Each directive creates a
*pattern* object (the type definitions import `TabPattern`, `TabListPattern` and
`TabPanelPattern`) that holds the widget's state as signals — which item is active,
which is selected, orientation, disabled items — and implements the keyboard and pointer
rules from the WAI-ARIA Authoring Practices. The directives then bind that state to host
attributes (`role`, `tabindex`, `aria-selected`, `aria-controls`, `inert`) and forward
DOM events to the pattern. Because everything is signals, the ARIA attributes can never
drift out of sync with what's selected.

`cdkTrapFocus` inserts two invisible, focusable "anchor" elements before and after the
trapped region. When focus lands on one of them (by tabbing past the last or before the
first element), the trap redirects it to the opposite end of the region. `LiveAnnouncer`
keeps one visually hidden element with `aria-live` in the document and swaps its text;
screen readers read the change.

## Common mistakes

- **Clickable `<div>`s and `<span>`s.** Not focusable, not announced as buttons, no
  keyboard support.
- **Removing focus outlines** with `outline: none` and no replacement.
- **Custom widgets without keyboard support.** Use `@angular/aria` or follow the ARIA
  pattern exactly — partial ARIA is worse than none.
- **Opening dialogs without trapping and restoring focus.**
- **Testing only with a mouse.** Unplug it: can you do everything with Tab, Enter, Space,
  arrows and Escape?

## Exercise

1. Replace the hand-made tabs from lesson 02 with `@angular/aria/tabs`. Verify the
   keyboard behaviour listed above.
2. Build an accordion with `@angular/aria/accordion` (check the directive names in the
   package's type definitions or the guide at angular.dev) and confirm headers expose
   `aria-expanded`.
3. Add a "Remove" button to a list and use `LiveAnnouncer` to announce what was removed;
   test with your OS screen reader (VoiceOver, NVDA or Narrator).
4. After every router navigation, move focus to the page's `<h1>` and confirm a screen
   reader announces the new page.
5. Run an automated checker (Lighthouse's accessibility audit or the axe browser
   extension) on one page and fix everything it reports — then remember that automated
   tools only catch part of the problems, so repeat step 3 manually.
