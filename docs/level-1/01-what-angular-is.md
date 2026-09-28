---
description: "What Angular is and what it includes: components, signals, DI, router, forms, HTTP, the AOT compiler, the CLI, and its six-month release cycle."
---

# 01 · What Angular Is

Angular is an open-source framework for building web applications in TypeScript,
maintained by a team at Google. "Framework" is the important word. A library such as React
gives you a way to describe UI and leaves routing, data fetching, forms, testing and build
tooling for you to choose. Angular ships all of those pieces as one coordinated set of
packages, versioned together, with a command-line tool that knows how to generate, build,
test and upgrade them.

That is the whole trade-off in one sentence: you get fewer decisions to make and a
consistent structure across every Angular codebase, and in return you learn Angular's way
of doing each thing.

## Angular vs AngularJS

You will still meet the name **AngularJS** in old blog posts and job descriptions. It is a
different framework — the 1.x line released in 2010, written in JavaScript, built around
scopes and two-way data binding with a digest loop. Angular (versions 2 and up, first
released in 2016) was a rewrite, not an upgrade. AngularJS reached end of support at the
end of 2021. Anything that mentions `$scope`, `ng-controller` or `angular.module(...)` is
about AngularJS and does not apply to this course.

## What is in the box

| Concern | What Angular provides | Package |
|---------|----------------------|---------|
| UI | Components, templates, directives, pipes | `@angular/core`, `@angular/common` |
| Reactivity | Signals (`signal`, `computed`, `effect`, `linkedSignal`, `resource`) | `@angular/core` |
| Wiring | Dependency injection (`inject()`, providers, hierarchical injectors) | `@angular/core` |
| Navigation | Router with lazy loading, guards and resolvers | `@angular/router` |
| Forms | Reactive forms, template-driven forms, and signal forms | `@angular/forms` |
| Server calls | `HttpClient` with interceptors, plus `httpResource` | `@angular/common/http` |
| Server rendering | SSR, prerendering and hydration | `@angular/ssr` |
| Tooling | CLI, dev server, production build (esbuild + Vite), test runner integration | `@angular/cli`, `@angular/build` |
| Extras | Component Dev Kit, Material components, accessibility primitives, i18n | `@angular/cdk`, `@angular/material`, `@angular/aria`, `@angular/localize` |

Every `@angular/*` package in a project should be on the same version. The CLI's
`ng update` command upgrades them together and runs code migrations for you — one of the
biggest practical benefits of the "one framework" model (Level 4, lesson 05).

## A first look at the code

Here is a complete component. Don't worry about every detail yet — lessons 03 and 04 take
it apart line by line.

```ts title="src/app/hello.ts"
import { Component, signal } from '@angular/core';

@Component({
  selector: 'app-hello',
  template: `
    <h1>Hello, {{ name() }}!</h1>
    <button (click)="name.set('Angular')">Rename</button>
  `,
})
export class Hello {
  protected readonly name = signal('world');
}
```

Three things to notice:

- A component is a **TypeScript class with a decorator**. The decorator says what HTML
  tag it answers to (`selector`) and what it renders (`template`).
- The template is **HTML with a small set of extra syntax**: `{{ }}` to show a value,
  `(click)` to listen to an event. It is not JSX and not a string you concatenate — the
  compiler parses it and type-checks it against the class.
- The state is a **signal**. The template calls `name()` to read it; the button calls
  `name.set(...)` to change it. When it changes, Angular updates exactly the one text
  node that read it.

## Where Angular fits well — and where it doesn't

Angular is a strong fit for:

- **Large, long-lived applications** built by several teams: admin consoles, banking and
  insurance portals, internal line-of-business tools, dashboards. Shared conventions and
  strict typing pay off as the codebase and team grow.
- **Teams who want batteries included.** Routing, forms, HTTP and testing are decided,
  documented and upgraded together.
- **Enterprises that need predictable upgrades.** A published release schedule and
  automated migrations make it realistic to keep a big app current.

It is a weaker fit for a small marketing page or a widget embedded in someone else's
site, where a whole framework's conventions are more than you need (though Angular
Elements, Level 4 lesson 04, covers the embedding case). If you are choosing between
frameworks, compare the same small app in each — the
[React Mastery Path](https://sigilipelli.github.io/react-mastery-path/) and this course
build similar projects on purpose.

## The release cycle

Angular follows a published schedule: a **major version roughly every six months**, with
minor and patch releases in between. Each major gets about 6 months of active support
followed by 12 months of long-term support (security and critical fixes only), so any
given major is supported for around 18 months. Majors are where defaults change and
deprecated APIs are removed — after a deprecation has been announced for at least two
majors.

That is why this course says what version it targets (22.2) and calls out defaults that
changed recently. You can check the current release and support table any time on the
official site at `angular.dev/reference/releases`.

## How It Actually Works

Angular is a **compiled** framework. When you run `ng build` or `ng serve`, the Angular
compiler (part of `@angular/compiler-cli`) reads each component's template and decorator
metadata and generates plain JavaScript instructions for creating and updating the DOM.
This is called **ahead-of-time (AOT) compilation**, and it is the only mode used by
modern builds.

To make that concrete, here is what the compiler generated for the `Hello` component
above, copied from a development build (`ng build --configuration development`) of
Angular 22.2. The `ɵɵ` prefix marks private framework instructions — you never write
these yourself:

```js
static ɵcmp = ɵɵdefineComponent({ type: _Hello, selectors: [["app-hello"]], decls: 4, vars: 1,
  consts: [[3, "click"]], template: function Hello_Template(rf, ctx) {
    if (rf & 1) {
      ɵɵdomElementStart(0, "h1");
      ɵɵtext(1);
      ɵɵdomElementEnd();
      ɵɵdomElementStart(2, "button", 0);
      ɵɵdomListener("click", function Hello_Template_button_click_2_listener() {
        return ctx.name.set("Angular");
      });
      ɵɵtext(3, "Rename");
      ɵɵdomElementEnd();
    }
    if (rf & 2) {
      ɵɵadvance();
      ɵɵtextInterpolate1("Hello, ", ctx.name(), "!");
    }
  }, encapsulation: 2 });
```

The template function has two modes, selected by the `rf` (render flags) bit mask:

- **Creation mode (`rf & 1`)** runs once per component instance. It creates the `<h1>`,
  an empty text node, the `<button>` and wires up the click listener.
- **Update mode (`rf & 2`)** runs whenever Angular refreshes this view. It walks to the
  text node (`ɵɵadvance()`) and calls `ɵɵtextInterpolate1`, which reads `ctx.name()` and
  only touches the DOM if the resulting string differs from the last one it wrote.

Notice there is no virtual DOM and no diffing of element trees. The compiler already
knows the shape of the template, so at runtime Angular only compares the handful of
*bound values* (here, one string). And because `ctx.name()` is a signal read inside a
template, Angular records that this view depends on `name`; when `name.set()` is called,
it schedules a refresh of just the views that read it (lesson 04, and Level 3 lesson 01
in depth).

Type checking happens in the same step. Because the compiler understands the template,
a typo such as `{{ nme() }}` fails the build with an error pointing at the template line,
rather than rendering `undefined` in production.

## Common misconceptions

- **"Angular is the same as AngularJS."** It isn't; see above. Tutorials for 1.x do not
  apply.
- **"Angular needs NgModules."** Older code (before roughly version 15) organised
  everything into `@NgModule` classes. Components have been standalone by default since
  Angular 19 and new projects contain no NgModules at all. You will still need to *read*
  NgModule code in older projects; Level 4 lesson 05 covers migrating it.
- **"Angular needs Zone.js to detect changes."** New projects since Angular 21 are
  zoneless: they update the screen in response to signals, events and explicit
  notifications, not by patching every browser API. Zone.js still works for existing
  apps that use it (Level 3, lesson 01).
- **"It's too heavy for anything."** The production build tree-shakes unused framework
  code. The initial bundle for the Level 1 project app in this course, built with
  `ng build`, was 261 kB raw / about 73 kB estimated transfer size, and that includes the
  router and HTTP client. Measure your own app instead of relying on reputation.

## Exercise

1. Open `angular.dev/reference/releases` and write down the current major version, the
   date it was released, and until when the previous major receives long-term support.
2. In your own words (three or four sentences), explain what "ahead-of-time compilation"
   means for a template, and name one class of bug it catches before your app ships.
3. List three features from the "What is in the box" table that you would have to choose
   a separate library for if you were using a view-only library instead. For each, name a
   popular third-party option.
