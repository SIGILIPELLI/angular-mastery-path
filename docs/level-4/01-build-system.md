---
description: "Master the Angular build: angular.json targets and configurations, environments with fileReplacements, define constants, dev-server proxying and what production mode changes."
---

# 01 · The Build System & Configurations

Every Angular app you've built in this course was compiled by the same machinery:
`@angular/build:application` (esbuild + the Angular compiler) for builds, and
`@angular/build:dev-server` (Vite-based) for `ng serve`. Level 4 starts here because
almost every production concern — environments, deployment paths, i18n, SSR, CSP,
budgets — is ultimately an option on those builders. Knowing where they live saves hours.

## `angular.json` in one picture

```text
angular.json
└── projects
    └── reading-list
        └── architect                     ← "targets"
            ├── build   builder: @angular/build:application
            │   ├── options              ← apply to every build
            │   └── configurations
            │       ├── production        ← default for `ng build`
            │       └── development
            ├── serve   builder: @angular/build:dev-server
            │   └── configurations        ← point at build:production / build:development
            └── test    builder: @angular/build:unit-test
```

`ng build` runs the `build` target with its `defaultConfiguration` (`production` in a new
project). `ng build -c development` picks the other one, and configurations can be
combined: `-c production,staging` applies both in order, later ones overriding earlier.

The build target has a lot of options — in Angular 22's schema they include `assets`,
`baseHref`, `budgets`, `define`, `externalDependencies`, `fileReplacements`, `i18n`
settings (`localize`), `optimization`, `outputHashing`, `outputMode`, `prerender`,
`security`, `server`, `serviceWorker`, `sourceMap`, `ssr`, `statsJson`,
`subresourceIntegrity` and more. You rarely need most of them, but it's worth skimming
the list once so you know they exist.

## Production vs development: what actually changes

We built the reading-list app both ways:

```text
ng build                  Initial total  261.39 kB   (73.12 kB estimated transfer)
ng build -c development   Initial total    1.47 MB   plus a .js.map file per chunk
```

The generated `development` configuration sets `optimization: false`,
`extractLicenses: false` and `sourceMap: true`. The production build minifies, tree-shakes
more aggressively, removes development-only checks (`ngDevMode` code such as the NG0100
"expression changed" check and most warnings you've seen in this course), hashes file
names for caching, and enforces budgets. **Never deploy a development build**, and never
judge performance from one.

## Environments

Per-environment values (API base URLs, feature flags, analytics IDs) are handled with
**file replacement**. The CLI sets it up:

```bash
ng generate environments
```

```text
CREATE src/environments/environment.ts (31 bytes)
CREATE src/environments/environment.development.ts (31 bytes)
UPDATE angular.json (2270 bytes)
```

It added this to the `development` configuration:

```json title="angular.json (excerpt)"
"fileReplacements": [
  {
    "replace": "src/environments/environment.ts",
    "with": "src/environments/environment.development.ts"
  }
]
```

Fill the two files with the same shape:

```ts title="src/environments/environment.ts"
export const environment = {
  production: true,
  apiUrl: 'https://api.example.com',
};
```

```ts title="src/environments/environment.development.ts"
export const environment = {
  production: false,
  apiUrl: 'http://localhost:3000',
};
```

and always import the **base** file: `import { environment } from '../environments/environment';`.
The production bundle contained only `api.example.com`; the development bundle contained
only `localhost:3000`. The other file's contents never reached the output.

For more environments (staging, QA), add a configuration with its own
`fileReplacements` and build with `-c staging`.

!!! warning "Environments are public"
    Everything in an environment file ends up in JavaScript anyone can read. Put
    URLs and flags there, never secrets (Level 3, lesson 09).

## Compile-time constants with `define`

For values injected by CI — a version number, a commit hash — `define` replaces an
identifier with a literal at build time:

```json title="angular.json (build options)"
"define": { "BUILD_VERSION": "'1.4.0'" }
```

```ts title="src/app/build-info.ts"
import { Component } from '@angular/core';
import { environment } from '../environments/environment';

declare const BUILD_VERSION: string;

@Component({
  selector: 'app-build-info',
  template: `<small>v{{ version }} · API {{ apiUrl }}</small>`,
})
export class BuildInfo {
  protected readonly version = BUILD_VERSION;
  protected readonly apiUrl = environment.apiUrl;
}
```

The value must be a string containing a JavaScript expression — note the inner quotes.
In the production bundle the identifier was gone and the literal `"1.4.0"` was inlined.
You can pass it from the command line in CI: `ng build --define "BUILD_VERSION='2.0.0'"`
inlined `"2.0.0"` in our test.

## The dev server and a backend

During development your API usually runs on another port. Rather than enabling CORS on
the API, **proxy** API calls through the dev server so the browser sees one origin:

```json title="proxy.conf.json"
{
  "/api": {
    "target": "http://localhost:3000",
    "secure": false,
    "changeOrigin": true
  }
}
```

```json title="angular.json (serve options)"
"serve": {
  "builder": "@angular/build:dev-server",
  "options": { "proxyConfig": "proxy.conf.json" }
}
```

Now `http.get('/api/books')` in the app is forwarded to `localhost:3000/api/books`. The
dev server also has options for `port`, `host`, `ssl`, custom response `headers`,
`allowedHosts` and `hmr` (hot module replacement, which in recent versions updates
component templates and styles without a full reload).

## Other options you will reach for

| Option | Use it for |
|--------|-----------|
| `baseHref` | deploying under a sub-path (`/app/`) — Level 4, lesson 07 |
| `outputHashing` | `all` (default in production) for cache-busting file names |
| `sourceMap` | `{ "scripts": true, "hidden": true }` to generate maps for error tracking without linking them from the bundle |
| `subresourceIntegrity` | add `integrity` hashes to script tags |
| `allowedCommonJsDependencies` | silence the CommonJS warning for a dependency you've accepted |
| `externalDependencies` | leave an import unbundled (import maps, micro-frontends) |
| `security.autoCsp` | generated strict CSP — Level 3, lesson 09 |

## How It Actually Works

The CLI reads `angular.json`, finds the target, merges `options` with the selected
configurations (left to right), merges command-line flags on top, validates the result
against the builder's JSON schema, and calls the builder. That's why an unknown option
in `angular.json` fails immediately with a schema error, and why `ng build --help` lists
exactly the options in the schema.

Inside the application builder, `fileReplacements` and `define` are applied **before
bundling**: the TypeScript/esbuild module resolution is told "when anything asks for
`environment.ts`, load `environment.development.ts`", and esbuild's `define` feature
substitutes identifiers during parsing. The two behave differently for dead-code
elimination, and we checked:

```ts
if (!environment.production) console.log('debug-marker-xyz');
if (ENABLE_DEBUG) console.log('define-marker-abc');   // built with --define ENABLE_DEBUG=false
```

```text
debug-marker-xyz in production bundle: 1
define-marker-abc in production bundle: 0
```

`define` substitutes the literal `false` while parsing, so the minifier sees
`if (false)` and deletes the branch. `environment.production` is a property of an
exported object; the bundler doesn't constant-fold it, so the check (and its string)
ships — it's just never true. Use `define` for anything that must not reach production
at all (debug tooling, internal endpoints); use environments for values.

The dev server wraps the same compilation in watch mode and serves output from memory
through Vite's server, applying your proxy rules to matching request paths before
falling back to the SPA's `index.html`.

## Common mistakes

- **Importing `environment.development.ts` directly**, which bypasses replacement.
- **Secrets in environments** — they're public.
- **Different shapes** in environment files, so a property is `undefined` in one build.
  Share a type (`export interface Env`) to prevent it.
- **Forgetting the inner quotes in `define`.** `--define BUILD_VERSION=1.4.0` failed
  with `✘ [ERROR] Invalid define value (must be an entity name or JS literal): 1.4.0`;
  `--define "BUILD_VERSION='2.0.0'"` worked and inlined `"2.0.0"`.
- **Expecting `environment.production` checks to strip code.** They don't (above).
- **Measuring performance with `ng serve`** or a development build.

## Exercise

1. Add a `staging` configuration with its own environment file and build it with
   `ng build -c staging`. Confirm with `grep` which API URL is in the output.
2. Add a `define` for `BUILD_COMMIT` set from `git rev-parse --short HEAD` in a small npm
   script, and show it in the footer.
3. Create a `proxy.conf.json` that forwards `/api` to a local JSON server and remove any
   hard-coded `localhost` URL from your services.
4. Put `if (!environment.production) console.log('debug')` in a component, build for
   production, and search the output for `debug` — it's there. Replace the check with a
   `define`d `ENABLE_DEBUG` flag set to `false` and confirm the string disappears.
