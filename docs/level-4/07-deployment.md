---
description: "Deploy Angular apps: base href, SPA fallbacks, cache headers for hashed files, SSR on Node, containers and a CI pipeline."
---

# 07 · Deployment

`ng build` produces files; deployment is getting them in front of users with the right
URLs, the right HTTP behaviour and the right caching. What you need depends on how the
app renders (Level 3, lesson 06):

| App type | Output | Host on |
|----------|--------|---------|
| Client-rendered (CSR) | `dist/<app>/browser/` — static files | any static host or CDN: object storage + CDN, GitHub Pages, Netlify, Vercel, Firebase Hosting, Nginx |
| Prerendered only | static files, one `index.html` per route | same as above |
| Server-rendered routes | `browser/` + `server/server.mjs` | a Node runtime: a VM, a container platform, or a platform with an Angular SSR adapter |

## Static hosting: three things to get right

### 1. The base href

If the app isn't served from the domain root, tell the build:

```bash
ng build --base-href /reading-list/
```

The output's `index.html` then contained `<base href="/reading-list/">`, so the router
and relative asset URLs resolve under that path. (Setting `"baseHref"` in a build
configuration does the same.) A wrong base href shows as a blank page with 404s for
`main-*.js` in the Network panel.

### 2. The SPA fallback

The browser asks the server for whatever URL is in the address bar. A user who bookmarks
`/reading-list/books/abc` and comes back asks the server for that path — which isn't a
file. We deployed the reading-list app under `/reading-list/` and requested that deep link
from three kinds of server:

```text
no fallback   home OK | deep link: HTTP 404, app not loaded at all
404.html      home OK | deep link: HTTP 404, app loaded and showed "No book with id “abc”."
SPA rewrite   home OK | deep link: HTTP 200, app loaded and showed "No book with id “abc”."
```

- **No fallback** is broken: every reload of an inner page fails.
- **The `404.html` trick** — copy `index.html` to `404.html` — is what hosts without
  rewrite rules (such as GitHub Pages) allow. The app works, but every deep link is
  served with **status 404**, which search engines and monitoring tools treat as an error.
  Fine for internal tools and demos; not for public content.
- **A real rewrite rule** returns `index.html` with 200 for any path that isn't a file.
  Every serious host supports one. Nginx:

```nginx title="nginx.conf (excerpt)"
location /reading-list/ {
  try_files $uri $uri/ /reading-list/index.html;
}
```

(For public content, prerendering (Level 3, lesson 06) is better still: each route is a
real file with a real status, and only unknown URLs need a fallback — ideally one that
returns 404.)

### 3. Cache headers

The build hashes file names (`main-VM2DRPL3.js`), so a changed file always gets a new
name. That enables a simple, safe policy:

| File | Header | Why |
|------|--------|-----|
| `*.js`, `*.css`, hashed media | `Cache-Control: public, max-age=31536000, immutable` | the name changes whenever the content does |
| `index.html` (and prerendered HTML) | `Cache-Control: no-cache` | always revalidate, so users pick up new bundle names |
| files in `public/` with fixed names (`favicon.ico`, `robots.txt`) | short max-age | names never change |

Getting `index.html` wrong is the classic "users still see the old version after a deploy"
bug. The generated SSR server already serves static files with `maxAge: '1y'` and
`index: false`, leaving HTML to the Angular engine.

## Server-side rendering on Node

A build with SSR produces `dist/<app>/server/server.mjs`. Running it is one command:

```bash
NG_ALLOWED_HOSTS=shop.example.com PORT=8080 node dist/catalog/server/server.mjs
```

(We ran exactly this in Level 3 with `NG_ALLOWED_HOSTS=localhost`; without allowed hosts
every request is rejected with 400.)

A container image is the usual way to run it anywhere:

```dockerfile title="Dockerfile"
# Build stage
FROM node:24-slim AS build
WORKDIR /app
COPY package*.json ./
RUN npm ci
COPY . .
RUN npx ng build

# Runtime stage: only the build output and production dependencies
FROM node:24-slim
WORKDIR /app
ENV NODE_ENV=production PORT=8080
COPY package*.json ./
RUN npm ci --omit=dev
COPY --from=build /app/dist ./dist
USER node
EXPOSE 8080
CMD ["node", "dist/catalog/server/server.mjs"]
```

!!! note "Not run here"
    Docker wasn't installed on the machine used to write this course, so this Dockerfile
    was not built or run. It follows the standard multi-stage pattern; check the Node
    base image tag against the versions Angular supports (Node 22.22+, 24.15+ or 26+ for
    Angular 22) and set `NG_ALLOWED_HOSTS` in your deployment environment.

Behind a load balancer or proxy, make sure the original host and protocol reach the
server (the engine validates host headers; the `NG_TRUST_PROXY_HEADERS` setting controls
which forwarded headers it trusts). Some platforms (Firebase App Hosting, Netlify, Vercel,
Cloudflare and others) offer their own Angular SSR adapters or presets; check their
current documentation for Angular 22 support.

## A CI pipeline

Whatever the host, the pipeline is the same shape:

```yaml title=".github/workflows/deploy.yml (sketch)"
name: build-and-deploy
on:
  push:
    branches: [main]
jobs:
  build:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-node@v4
        with:
          node-version: 24
          cache: npm
      - run: npm ci
      - run: npx ng test --watch=false
      - run: npx ng build --define "BUILD_VERSION='${{ github.sha }}'"
      # then upload dist/<app>/browser (static) or build and push an image (SSR)
```

Order matters: tests, then the production build (which enforces budgets), then deploy.
For CI in depth — environments, approvals, preview deployments, rollbacks — see
[GitHub & Git Mastery Path](https://sigilipelli.github.io/github-mastery-path/) and
[Docker Mastery Path](https://sigilipelli.github.io/docker-mastery-path/).

## Releasing safely

- **Keep the previous version's files for a while.** Users with the old `index.html`
  still request old chunk names when they navigate to a lazy route. If the deploy deleted
  them, they get a load error. Deploy new files alongside old ones and clean up later, or
  handle chunk-load failures by prompting a reload (lesson 08).
- **Service workers** (`ng add @angular/pwa`) cache the app for offline use and
  notify you of new versions (`SwUpdate`); they also make cache mistakes stickier, so add
  one deliberately, not by default.
- **Roll back by redeploying the previous build artefact**, not by rebuilding an old
  commit under pressure.

## How It Actually Works

A deployed Angular app is, from the browser's point of view, one HTML document plus
JavaScript modules. On the first request the server returns `index.html`; the `<base>`
element makes every relative URL — scripts, lazy chunks, router URLs — resolve under the
base path. After that the router uses `history.pushState` and never asks the server for
HTML again **until the user reloads or opens a deep link**. That single moment is why the
server-side fallback matters so much and why it's so easy to miss in testing: clicking
around never triggers it.

With SSR, the server does real work per request: `express.static` answers files that
exist (hashed bundles, prerendered HTML); everything else goes to the Angular engine,
which renders the route, sets the status code and returns HTML.

## Common mistakes

- **Missing SPA fallback**, so reloads 404.
- **Wrong base href** for a sub-path deployment.
- **Long-caching `index.html`**, so users never see new releases.
- **Deleting old chunks on deploy**, breaking users mid-session.
- **Deploying a development build** — 1.47 MB instead of 261 kB for the same app
  (lesson 01).

## Exercise

1. Build the reading-list app with `--base-href /reading-list/`, serve the `dist` folder
   under that path with a simple static server, and reproduce the deep-link 404.
2. Fix it with a rewrite rule for a server of your choice (Nginx `try_files`, a hosting
   provider's `rewrites` config, or a few lines of Node).
3. Add cache headers per the table and verify them with `curl -I`.
4. Deploy the Level 3 catalog's SSR server somewhere that runs Node (a VM, a container
   platform, or a provider with an Angular preset) with `NG_ALLOWED_HOSTS` set to your
   domain.
5. Write a CI workflow that runs tests and a production build on every pull request.
