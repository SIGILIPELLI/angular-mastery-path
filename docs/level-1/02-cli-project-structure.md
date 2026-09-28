---
description: "Create an Angular 22 project with ng new, understand every generated file, and use ng serve, ng build, ng test and ng generate day to day."
---

# 02 · The CLI & Project Structure

The Angular CLI (`@angular/cli`, command name `ng`) is how you create, run, build, test,
generate and upgrade Angular projects. You can build an Angular app without it, but almost
nobody does — the CLI owns the build configuration so that `ng update` can later migrate it
for you.

## Creating a project

You do not need to install the CLI globally. `npx` runs a specific version once:

```bash
npx @angular/cli@22 new reading-list
```

Without flags, `ng new` asks a few questions (stylesheet format, whether to enable
server-side rendering, whether to generate AI-assistant configuration files). To accept the
defaults and skip prompts, add `--defaults`. The options you are most likely to care about:

| Flag | What it does | Default in 22 |
|------|--------------|---------------|
| `--style` | `css`, `scss`, `sass`, `less` or `tailwind` | `css` |
| `--ssr` | Adds server-side rendering and prerendering (Level 3, lesson 06) | asked |
| `--routing` | Generates `app.routes.ts` and wires the router | yes |
| `--test-runner` | `vitest` or `karma` | `vitest` |
| `--zoneless` | App does not load `zone.js` | yes for new apps |
| `--inline-template` / `--inline-style` | Root component uses `template:` / `styles:` instead of separate files | no |
| `--skip-git`, `--skip-install` | Skip `git init` / `npm install` | no |

If you prefer a global install (`npm install -g @angular/cli`), keep in mind that inside a
project the CLI always hands off to the **project's local** copy in `node_modules`, so each
project uses the version it was built with.

## What was generated

Running `ng new demo --defaults --ssr=false` with Angular CLI 22.2.0 created this tree
(minus `node_modules`):

```text
demo/
├── .editorconfig
├── .gitignore
├── .prettierrc
├── .vscode/            extensions.json, launch.json, tasks.json
├── angular.json
├── package.json
├── README.md
├── tsconfig.json
├── tsconfig.app.json
├── tsconfig.spec.json
├── public/
│   └── favicon.ico
└── src/
    ├── index.html
    ├── main.ts
    ├── styles.css
    └── app/
        ├── app.ts
        ├── app.html
        ├── app.css
        ├── app.spec.ts
        ├── app.config.ts
        └── app.routes.ts
```

Walking through it from the outside in:

**`package.json`** — dependencies. Runtime packages are the `@angular/*` framework
packages, `rxjs` and `tslib`. Development packages include `@angular/cli`,
`@angular/build` (the build system), `@angular/compiler-cli`, `typescript` (6.0 for
Angular 22), `vitest` and `jsdom` for tests. Notice what is **not** there: `zone.js`. New
projects are zoneless.

**`angular.json`** — the workspace configuration. It declares one project (`demo`) and
its **architect targets**: `build` uses the `@angular/build:application` builder with
`src/main.ts` as the entry point, `serve` uses `@angular/build:dev-server`, and `test`
uses `@angular/build:unit-test`. The `production` build configuration also declares
**budgets** — size limits that turn into warnings or errors (500 kB warning / 1 MB error
for the initial bundle by default). Level 4 lesson 01 covers this file in depth.

**`tsconfig*.json`** — TypeScript settings. The base file enables options such as
`noImplicitReturns` and `noPropertyAccessFromIndexSignature`, and Angular-specific
compiler options under `angularCompilerOptions`. `tsconfig.app.json` covers application
code and excludes `*.spec.ts`; `tsconfig.spec.json` covers tests and adds the
`vitest/globals` types so `describe`/`it`/`expect` need no import. There is no
`"strict": true` line in the generated file because TypeScript 6 turns strict mode on by
default — a parameter without a type fails the build:

```text
✘ [ERROR] TS7006: Parameter 'a' implicitly has an 'any' type. [plugin angular-compiler]
```

**`src/index.html`** — the single HTML page. Its `<body>` contains just `<app-root></app-root>`,
the root component's selector. `<base href="/">` tells the router where the app lives;
change it when deploying under a sub-path (Level 4, lesson 07).

**`src/main.ts`** — the entry point:

```ts title="src/main.ts"
import { bootstrapApplication } from '@angular/platform-browser';
import { appConfig } from './app/app.config';
import { App } from './app/app';

bootstrapApplication(App, appConfig)
  .catch((err) => console.error(err));
```

**`src/app/app.config.ts`** — application-wide providers. Generated with
`provideBrowserGlobalErrorListeners()` (report uncaught errors and unhandled promise
rejections to Angular's `ErrorHandler`) and `provideRouter(routes)`. You will add
`provideHttpClient()` here in lesson 09.

**`src/app/app.ts`, `app.html`, `app.css`** — the root component, split into class,
template and styles. `app.spec.ts` is its test.

**`public/`** — files copied as-is to the build output (favicon, `robots.txt`, images you
reference by URL).

## The four commands you will use every day

```bash
ng serve            # dev server with rebuild on save, http://localhost:4200 by default
ng build            # production build into dist/<project>/browser
ng test             # run unit tests (Vitest), in watch mode when run interactively
ng generate ...     # scaffold components, services, guards, pipes, and more
```

Inside a project you can run them as `npx ng ...` or through the npm scripts the CLI
added (`npm start`, `npm run build`, `npm test`).

Here is `ng build` on the Level 1 project app — note the **lazy chunk files** for routes
that are loaded on demand (lesson 08):

```text
Initial chunk files | Names         |  Raw size | Estimated transfer size
chunk-YEAVOUZW.js   | -             | 160.83 kB |                47.20 kB
chunk-2BLNMBOE.js   | -             |  95.29 kB |                24.11 kB
main-LSBGFHN4.js    | main          |   5.37 kB |                 1.88 kB
styles-5INURTSO.css | styles        |   0 bytes |                 0 bytes

                    | Initial total | 261.48 kB |                73.19 kB

Lazy chunk files    | Names         |  Raw size | Estimated transfer size
chunk-SFWQEAUE.js   | book-search   |   2.25 kB |                 1.02 kB
chunk-R3WAKTBP.js   | book-detail   | 878 bytes |               878 bytes

Application bundle generation complete. [1.202 seconds]
```

The hashes in file names (`main-LSBGFHN4.js`) change whenever content changes, so you
can cache these files forever on a CDN.

### Generating code

```bash
ng generate component book-card     # or: ng g c book-card
ng generate service books           # or: ng g s books
```

With CLI 22.2 that produced:

```text
CREATE src/app/book-card/book-card.css (0 bytes)
CREATE src/app/book-card/book-card.spec.ts (547 bytes)
CREATE src/app/book-card/book-card.ts (196 bytes)
CREATE src/app/book-card/book-card.html (24 bytes)
CREATE src/app/books.spec.ts (315 bytes)
CREATE src/app/books.ts (76 bytes)
```

Useful flags: `--inline-template` (`-t`) and `--inline-style` (`-s`) to keep everything
in one `.ts` file, `--skip-tests` to omit the spec, `--flat` to avoid creating a folder,
and `--dry-run` to see what *would* be created without touching disk.

Other schematics you will meet later in this course: `guard`, `interceptor`, `resolver`,
`pipe`, `directive`, `environments`, `library` and `web-worker`.

## Naming conventions (the 2025 style guide)

Before Angular 20, generated files were named `book-card.component.ts` with a class
called `BookCardComponent`. The current style guide drops the type suffix: the file is
`book-card.ts` and the class is `BookCard`. Both work — it is only a convention — and
`ng new --file-name-style-guide=2016` keeps the old style. Existing projects upgraded with
`ng update` keep their old names; nothing is renamed for you.

## How It Actually Works

`ng` is a thin command dispatcher. Each command maps to a **builder** configured in
`angular.json`. `ng build` runs `@angular/build:application`, which does roughly this:

1. **TypeScript + Angular compilation.** The Angular compiler plugin type-checks your
   `.ts` files *and* your templates, and emits JavaScript with templates compiled into
   instruction functions (see lesson 01).
2. **Bundling with esbuild.** The emitted modules are bundled, dynamic `import()`s become
   separate lazy chunks, unused code is tree-shaken, and output is minified.
3. **Styles and assets.** Global styles (`src/styles.css`) and component styles are
   processed; files in `public/` are copied.
4. **Index generation.** `src/index.html` is rewritten to reference the hashed bundles.
5. **Budget check.** Output sizes are compared with the budgets in `angular.json`.

`ng serve` uses the same compilation pipeline but serves files from memory through a
Vite-based dev server, rebuilding incrementally on save. `ng test` compiles your spec
files with the same Angular compiler and hands them to Vitest running in a `jsdom`
environment (Level 3, lesson 07). That shared pipeline is why a template error shows up
identically in `serve`, `build` and `test`.

Schematics — the code generators behind `ng generate`, `ng new`, `ng add` and the
migrations run by `ng update` — operate on a **virtual file tree**. They compute all the
changes first and only write them to disk at the end, which is why `--dry-run` can show
you exactly what would happen.

## Common mistakes

- **Mixing CLI versions.** A global `ng` from an older major can create a project with
  outdated defaults. Use `npx @angular/cli@22 new ...` to be explicit.
- **Editing files in `dist/`.** They are regenerated on every build. Change the source.
- **Putting images in `src/assets/` because an old tutorial said so.** New projects use
  `public/`; files there are served from the site root (`/logo.svg`, not
  `/assets/logo.svg`).
- **Ignoring budget warnings.** They are the cheapest performance alarm you have.
- **Committing `.angular/cache`.** It is a local build cache and is already in the
  generated `.gitignore`.

## Exercise

1. Create a new project with `npx @angular/cli@22 new sandbox --defaults`.
2. Run `ng build` and record the initial total size it reports.
3. Generate a component called `greeting` with inline template and style, and without a
   spec file. Which single file was created?
4. Use `<app-greeting />` in `app.html`. Run `ng build` again — it fails. Read the error,
   then fix it by adding `Greeting` to the root component's `imports` array. (Lesson 03
   explains why this is required.)
5. Open `angular.json`, lower the initial budget's `maximumError` to `100kB`, and build
   again. Note the error, then put it back.
