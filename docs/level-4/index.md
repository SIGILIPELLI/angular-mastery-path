---
description: "Level 4 of Angular: builds, libraries, i18n, custom elements, upgrades, architecture, deployment, observability and a production capstone."
---

# Level 4 · Master <span class="level-badge">Production</span>

Goal: own an Angular application in production — build it for different environments,
share code across apps, ship it in several languages, deploy it safely, keep it
upgraded, find out when it breaks, and judge honestly whether it's ready.

The first three levels were about what happens inside the app. Level 4 is about
everything around it: `angular.json`, the package format, the translation pipeline,
the host and its headers, the monitoring service, the upgrade calendar and the people who
review a release. It's the level where most real-world Angular problems actually live.

Three ideas run through this level:

1. **The build is configuration you own.** Environments, `define`, budgets, source maps,
   base href, locales, CSP — all builder options, all reviewable in version control.
2. **Production is a system, not a bundle.** Hosting rewrites, cache headers, allowed
   hosts, chunk availability across deploys and error reporting decide whether users
   see a working app.
3. **Verify with evidence.** Every lesson shows real build output, real browser
   behaviour, or real tool results — including several places where the first attempt
   was wrong (an `environment.production` check that doesn't remove code, a missing
   translation that doesn't warn, `autoCsp` refusing to combine with SSR). Expect the same
   surprises in your own projects, and check.

## Modules

1. [The Build System & Configurations](01-build-system.md) — `angular.json` targets, environments, `define`, dev-server proxying
2. [Libraries & Monorepos](02-libraries-monorepos.md) — multi-project workspaces, public APIs, ng-packagr and partial compilation
3. [Internationalization](03-i18n.md) — `i18n` markers, ICU plurals, `$localize`, extraction, per-locale builds and missing translations
4. [Angular Elements & Micro-Frontends](04-elements-microfrontends.md) — components as custom elements, and an honest look at micro-frontends
5. [Upgrading & Migrations](05-upgrading-migrations.md) — `ng update`, the support cycle, and real results from the modernisation migrations
6. [Large-App Architecture](06-large-app-architecture.md) — feature-first structure, dependency rules and how to enforce them
7. [Deployment](07-deployment.md) — base href, SPA fallbacks, cache headers, SSR on Node, containers and CI
8. [Error Handling & Observability](08-errors-observability.md) — `ErrorHandler`, `@boundary`, navigation errors, source maps and monitoring
9. [Production Readiness Review](09-production-readiness.md) — a full checklist and an automated audit script
10. [Capstone — A Production Angular App](10-capstone.md) — the Level 3 catalog taken to production standard, verified end to end

## Before you start

- Finish Level 3, especially SSR (lesson 06) and testing (lesson 07); the capstone builds
  on the Level 3 catalog project.
- Several lessons install packages (`@angular/localize`, `@angular/elements`,
  `ng-packagr` via `ng generate library`). All run locally.
- Lesson 07's Dockerfile and hosting-provider specifics couldn't be exercised on the
  machine used to write this course; those parts are marked as not run.
