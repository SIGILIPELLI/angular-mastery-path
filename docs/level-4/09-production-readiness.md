---
description: "A production-readiness review for Angular apps: build, performance, security, accessibility, reliability, observability and operations — with an automated audit script."
---

# 09 · Production Readiness Review

A production-readiness review is a deliberate pass over an app **before** real users
depend on it, asking "what will hurt us at 2 a.m.?" rather than "does the feature work?".
This lesson collects what the course has covered into one checklist, and shows how to
automate the parts a script can check so the review stays honest release after release.

## The checklist

Each item links back to where it was taught. "Evidence" is what you should be able to show,
not just claim.

### Build & delivery

| Check | Evidence |
|-------|----------|
| Production build is the default and is what gets deployed | `defaultConfiguration: "production"`; CI builds without `-c development` (L4-01) |
| Budgets fail the build on regressions | `initial` budget with `maximumError`, set just above current size (L3-05) |
| Environments hold no secrets | review of `src/environments/*` (L4-01, L3-09) |
| Deep links work on the real host | reload `/some/inner/route` in production returns the app with 200 (L4-07) |
| Cache headers: hashed files immutable, HTML revalidated | `curl -I` output (L4-07) |
| Old chunks survive a deploy / chunk-load errors reload once | deploy procedure, `withNavigationErrorHandler` (L4-07, L4-08) |

### Performance

| Check | Evidence |
|-------|----------|
| Initial bundle contains only what the first screen needs | `ng build --stats-json` review (L3-05) |
| Features lazy-loaded; heavy widgets deferred | lazy chunk list in build output (L1-08, L3-04) |
| LCP image has `priority`; images have dimensions | `NgOptimizedImage` warnings clean in dev (L3-05) |
| Public pages render HTML without JavaScript | prerender/SSR, `curl` shows content (L3-06) |
| Core Web Vitals measured in the field | monitoring dashboard (L4-08) |

### Correctness & reliability

| Check | Evidence |
|-------|----------|
| Tests cover services and key components; run in CI | `ng test --watch=false` in pipeline, coverage reviewed for gaps (L3-07) |
| State is signal-based; no reliance on zone-triggered rendering | no plain-field async updates (L3-01) |
| Every data load has loading, error and empty states | UI review; resources never read `value()` in error state (L2-06) |
| Errors are contained | `@boundary` around risky widgets (L4-08) |
| Dev-mode checks enabled in tests | `provideCheckNoChangesConfig({ exhaustive: true })` (L3-01) |

### Security

| Check | Evidence |
|-------|----------|
| No `bypassSecurityTrust*` on untrusted data; no direct `innerHTML` writes | code search (L3-09) |
| XSRF strategy matches how the API authenticates | server + `HttpClient` config (L3-09) |
| CSP in place | `autoCsp` or a CSP header (L3-09) |
| SSR allowed hosts configured | `NG_ALLOWED_HOSTS` / `security.allowedHosts` (L3-06) |
| Authorisation enforced on the server, not only by guards | API review (L2-07) |
| Dependencies audited | `npm audit` in CI, a process for updates (L4-05) |

### Accessibility

| Check | Evidence |
|-------|----------|
| Keyboard-only walkthrough of every main flow | recorded manual test (L3-08) |
| Screen-reader pass on key pages | notes from VoiceOver/NVDA/Narrator run |
| Automated audit clean | Lighthouse / axe report |
| Route changes announced; focus managed in dialogs | L3-08 |

### Observability & operations

| Check | Evidence |
|-------|----------|
| Custom `ErrorHandler` reports to a monitoring service, deduplicated | L4-08 |
| Source maps uploaded, not publicly linked | hidden source maps (L4-08) |
| Release version in every error report | `define`d `BUILD_VERSION` (L4-01) |
| Someone owns alerts; rollback procedure written and tried | runbook |
| Upgrade policy: within Angular's support window | calendar reminder, L4-05 |

## Automating what can be automated

About a third of that list can be checked mechanically on every build. Here is a
dependency-free Node script covering the configuration and output checks:

```js title="tools/check-readiness.mjs"
// Automated part of a production-readiness review for an Angular CLI project.
// Usage: node check-readiness.mjs <project-dir> <project-name>
import { existsSync, readFileSync, readdirSync, statSync } from 'node:fs';
import { join } from 'node:path';

const [dir, name] = process.argv.slice(2);
const ws = JSON.parse(readFileSync(join(dir, 'angular.json'), 'utf8'));
const build = ws.projects[name].architect.build;
const prod = build.configurations?.production ?? {};
const opts = { ...build.options, ...prod };
const out = join(dir, 'dist', name, 'browser');
const results = [];
const check = (ok, label, detail = '') => results.push({ ok, label, detail });

// Build configuration
check(build.defaultConfiguration === 'production', 'production is the default build configuration');
const initial = (opts.budgets ?? []).find((b) => b.type === 'initial');
check(!!initial?.maximumError, 'initial bundle budget has an error limit', initial ? `${initial.maximumWarning} / ${initial.maximumError}` : 'none');
check(opts.optimization !== false, 'optimization enabled in production');
check(opts.outputHashing === 'all' || prod.outputHashing === 'all', 'output hashing = all');
const serverTs = join(dir, 'src', 'server.ts');
const cspHeader = existsSync(serverTs) && readFileSync(serverTs, 'utf8').includes('Content-Security-Policy');
check(!!opts.security?.autoCsp || cspHeader, 'a Content Security Policy is configured', opts.security?.autoCsp ? 'security.autoCsp' : cspHeader ? 'header set in src/server.ts' : 'none');

// Build output
const indexPath = join(out, 'index.html');
if (!existsSync(indexPath)) {
  check(false, 'build output exists', `missing ${indexPath} — run ng build first`);
} else {
  const html = readFileSync(indexPath, 'utf8');
  const js = readdirSync(out).filter((f) => f.endsWith('.js'));
  check(js.every((f) => /-[A-Za-z0-9_-]{8}\.js$/.test(f)), 'all JS file names are content-hashed', `${js.length} files`);
  check(!js.some((f) => readFileSync(join(out, f), 'utf8').includes('sourceMappingURL')), 'bundles do not link source maps publicly');
  check(/<html[^>]*\slang="[^"]+"/.test(html), 'index.html declares a language');
  check(/<title>[^<]+<\/title>/.test(html), 'index.html has a title');
  check(/name="viewport"/.test(html), 'index.html has a viewport meta tag');
  const totalKb = js.reduce((s, f) => s + statSync(join(out, f)).size, 0) / 1024;
  check(true, 'total JavaScript on disk', `${totalKb.toFixed(1)} kB across ${js.length} files`);
}

// Source hygiene
const srcFiles = [];
(function walk(d) {
  for (const f of readdirSync(d)) {
    const p = join(d, f);
    if (statSync(p).isDirectory()) walk(p);
    else if (p.endsWith('.ts') && !p.endsWith('.spec.ts')) srcFiles.push(p);
  }
})(join(dir, 'src'));
const grepSrc = (re) => srcFiles.filter((f) => re.test(readFileSync(f, 'utf8'))).map((f) => f.slice(dir.length + 1));
const bypass = grepSrc(/bypassSecurityTrust/);
check(bypass.length === 0, 'no bypassSecurityTrust* calls', bypass.join(', '));
const logs = grepSrc(/console\.log\(/).filter((f) => f.startsWith('src/app/')); // server.ts may log
check(logs.length === 0, 'no console.log left in src/app', logs.join(', '));
const handler = grepSrc(/provide:\s*ErrorHandler/);
check(handler.length > 0, 'custom ErrorHandler registered');
const specs = [];
(function walk(d) {
  for (const f of readdirSync(d)) {
    const p = join(d, f);
    if (statSync(p).isDirectory()) walk(p);
    else if (p.endsWith('.spec.ts')) specs.push(p);
  }
})(join(dir, 'src'));
check(specs.length > 0, 'unit tests exist', `${specs.length} spec files`);

for (const r of results) console.log(`${r.ok ? '✔' : '✘'} ${r.label}${r.detail ? ` — ${r.detail}` : ''}`);
const failed = results.filter((r) => !r.ok).length;
console.log(`\n${results.length - failed}/${results.length} checks passed`);
process.exit(failed ? 1 : 0);
```

We ran it against two builds from this course. The plain Level 1 reading-list app:

```text
✔ production is the default build configuration
✔ initial bundle budget has an error limit — 500kB / 1MB
✔ optimization enabled in production
✔ output hashing = all
✘ a Content Security Policy is configured — none
✔ all JS file names are content-hashed — 5 files
✔ bundles do not link source maps publicly
✔ index.html declares a language
✔ index.html has a title
✔ index.html has a viewport meta tag
✔ total JavaScript on disk — 258.3 kB across 5 files
✔ no bypassSecurityTrust* calls
✔ no console.log left in src/app
✘ custom ErrorHandler registered
✔ unit tests exist — 4 spec files

13/15 checks passed
```

The copy with lesson 08's error handling and hidden source maps scored `14/15` — the
`ErrorHandler` check now passed; only the CSP remained. The Level 4 capstone (next lesson)
passes all 15. While building the capstone, the first version of this script flagged the
`console.log` in `src/server.ts` — the SSR server's legitimate startup message — so the
rule now only looks at `src/app/`. Expect to tune checks like this for your own project. The script exits non-zero on any
failure, so it can gate a CI job. Two things it deliberately **can't** tell you: whether
the ErrorHandler actually reaches a monitoring service, and whether the budget is tight
enough to matter (500 kB is the generated default; our app was around 260 kB). The review
still needs people.

## Running the review

1. **Schedule it** before first launch and before any launch that changes architecture
   (SSR, a new auth flow, a big dependency).
2. **Fill in evidence**, not ticks. "Tested deep links" is a claim; a `curl` output or a
   screenshot is evidence.
3. **Classify gaps**: *blocker* (security, data loss, total outage), *fix soon*
   (degraded experience), *accepted risk* (documented, with an owner).
4. **Automate** anything you had to check by hand twice.
5. **Repeat** at a fixed cadence — quarterly is common — because apps drift.

## How It Actually Works

Why do reviews catch things tests don't? Unit and end-to-end tests check that features
behave as designed **in the environment you test in**. Most production failures come from
the gaps between environments: a host without an SPA fallback, a cache that holds
`index.html`, a proxy that rewrites the `Host` header, a CDN that strips a CSP, a missing
source map, a dependency with a new vulnerability. None of those is visible from inside
`ng test`. A review looks at the **system** — build configuration, hosting, headers,
monitoring, people and procedures — and the script above encodes the parts of that
system that live in files you control.

## Common mistakes

- **Treating the review as a formality** at the end, when fixing blockers is most
  expensive.
- **Checklists without evidence.**
- **Only automated checks**, or **only manual ones** — you need both.
- **No owner for accepted risks**, so they're forgotten.
- **Never repeating the review** after launch.

## Exercise

1. Copy `check-readiness.mjs` into one of your projects and run it. Fix everything it
   reports, or write down why a failure is acceptable.
2. Add two checks of your own: for example, "no `environment` file contains the word
   `secret` or `key`", and "every lazy chunk is under 100 kB".
3. Run the full checklist for the Level 3 catalog. Mark each row blocker / fix soon /
   accepted risk, with evidence.
4. Add the script to a CI workflow so the build fails on regressions.
