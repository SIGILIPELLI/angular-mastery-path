---
description: "Angular security in practice: automatic XSS sanitization, the danger of bypassSecurityTrust*, unsafe URLs, XSRF tokens in HttpClient, strict CSP with autoCsp, and secrets."
---

# 09 · Security

Front-end security is mostly about one thing: **never letting data become code**. Most
attacks on web apps are cross-site scripting (XSS), where text an attacker controls ends
up executed as JavaScript in someone else's browser. Angular's template system is designed
to prevent that by default, and the few escape hatches are clearly named. This lesson
shows the defences working (and failing when bypassed), then covers the other front-end
duties: XSRF, Content Security Policy, and keeping secrets off the client.

## A hostile comment, rendered four ways

```ts title="src/app/security-lab.ts"
import { Component, inject } from '@angular/core';
import { DomSanitizer } from '@angular/platform-browser';

@Component({
  selector: 'app-security-lab',
  template: `
    <p id="interp">{{ comment }}</p>
    <div id="inner" [innerHTML]="comment"></div>
    <a id="link" [href]="website">Website</a>
    <a id="safe-link" [href]="goodSite">Good site</a>
    <div id="bypassed" [innerHTML]="trusted"></div>
  `,
})
export class SecurityLab {
  // Imagine these came from another user via your API.
  protected readonly comment =
    '<b>Nice post!</b><img src="x" onerror="window.pwned = true"><script>window.pwned2 = true</script>';
  protected readonly website = 'javascript:window.pwned3 = true';
  protected readonly goodSite = 'https://example.com/profile?id=7';
  // DON'T do this with user input — shown to demonstrate what bypassing does.
  protected readonly trusted = inject(DomSanitizer).bypassSecurityTrustHtml(
    '<img src="x" onerror="window.pwned4 = true">',
  );
}
```

What Chromium ended up with (after we also clicked the "Website" link):

```html
<p id="interp">&lt;b&gt;Nice post!&lt;/b&gt;&lt;img src="x" onerror="window.pwned = true"&gt;&lt;script&gt;…</p>
<div id="inner"><b>Nice post!</b><img src="x"></div>
<a id="link" href="unsafe:javascript:window.pwned3 = true">Website</a>
<a id="safe-link" href="https://example.com/profile?id=7">Good site</a>
<div id="bypassed"><img src="x" onerror="window.pwned4 = true"></div>
```

```text
{ pwned: undefined, pwned2: undefined, pwned3: undefined, pwned4: true }
```

And the development build logged:

```text
WARNING: sanitizing HTML stripped some content, see https://angular.dev/best-practices/security#preventing-cross-site-scripting-xss
WARNING: sanitizing unsafe URL value javascript:window.pwned3 = true (see https://angular.dev/best-practices/security#…)
```

Line by line:

1. **Interpolation** never interprets HTML. The whole string was shown as text.
2. **`[innerHTML]`** is sanitised: safe formatting (`<b>`) survived; the `onerror`
   attribute and the `<script>` element were stripped.
3. **URL bindings** are checked: a `javascript:` URL was prefixed with `unsafe:`, making
   it inert. Normal `https:` URLs pass through untouched.
4. **`bypassSecurityTrustHtml`** told Angular "I've checked this" — and the injected
   handler ran (`pwned4: true`). That's the only one of the four attacks that worked.

## The rule for `bypassSecurityTrust*`

`DomSanitizer` has `bypassSecurityTrustHtml`, `…Style`, `…Script`, `…Url` and
`…ResourceUrl`. Each one turns off protection for one value. Use them only for values
**your own code** constructs from constants — never for anything that came from a user,
an API, the URL or `localStorage`. If you need to render user-supplied rich text, sanitise
it on the server with a well-maintained HTML sanitiser, and still let Angular's sanitiser
run on the client. Grep for `bypassSecurityTrust` in code review; every hit deserves a
comment explaining why it's safe.

Other ways to break the protection that don't mention "bypass":

- Using `ElementRef.nativeElement.innerHTML = ...` or `document.write` directly.
- Building templates from strings at runtime (Angular's AOT compiler doesn't support
  this for a reason).
- Passing user input to `eval`, `new Function`, `setTimeout('string')`.
- Server-side: interpolating user data into the HTML page *before* Angular sees it.

## XSRF (CSRF) protection

If your API authenticates with **cookies**, another site can make the user's browser send
requests to it. The standard defence is a token the attacker's page can't read: the
server sets a cookie (by default named `XSRF-TOKEN`), and your app echoes it in a header
(`X-XSRF-TOKEN`) that the server checks. `HttpClient` does the echo automatically. With
the cookie set to `abc123`, our test requests got:

```text
GET  /api/x                          X-XSRF-TOKEN: null
POST /api/x                          X-XSRF-TOKEN: abc123
POST https://other.example.com/api   X-XSRF-TOKEN: null
```

The header is added only to **mutating** requests (not GET/HEAD) to **relative** (same
origin) URLs, so the token never leaks to third parties. The server must still set the
cookie and verify the header — Angular only handles the client half. Customise names with
`withXsrfConfiguration({ cookieName, headerName })` or turn it off with
`withNoXsrfProtection()` inside `provideHttpClient(...)`. If your API uses bearer tokens
in an `Authorization` header instead of cookies, CSRF doesn't apply in the same way, but
you must then protect the token from XSS.

## Content Security Policy

A CSP header tells the browser which scripts may run. Even if an XSS bug slips through,
a strict CSP stops injected inline scripts from executing. Angular's build can generate a
hash-based strict CSP for you (the option's own description calls it experimental, and it
defaults to `false`):

```json title="angular.json (build options)"
"security": { "autoCsp": true }
```

With it enabled, `ng build` injected this into the reading-list app's `index.html`:

```html
<meta http-equiv="Content-Security-Policy" content="script-src 'strict-dynamic'
  'sha256-5ww8xZUmholxSa2m5lcAniPl7Q2P5lW7baZMtioA4VI=' https: 'unsafe-inline';
  object-src 'none';base-uri 'self';">
```

`'strict-dynamic'` plus the hash allows exactly the bootstrap script Angular emitted and
anything it loads; `'unsafe-inline'` and `https:` are ignored by modern browsers when a
hash is present and exist only as fallbacks for very old ones. If you serve your own CSP
header instead (often better, because you can add `report-to` and other directives),
provide a per-request nonce to Angular with the `ngCspNonce` attribute on the root element
or the `CSP_NONCE` token so the styles it injects are allowed.

Angular also supports **Trusted Types**, a browser feature that makes dangerous DOM sinks
reject plain strings; enabling it via CSP (`require-trusted-types-for 'script'`) makes an
accidental `innerHTML = userInput` throw instead of executing.

## Secrets and the client

Everything in your bundle is public: environment files, API keys, "hidden" routes, feature
flags. Anyone can open DevTools and read them. So:

- Never put secret keys (payment, database, private API keys) in Angular code. Call your
  own server, which holds the secret.
- Treat guards, disabled buttons and hidden fields as UX. Authorisation happens on the
  server (Level 2, lesson 07).
- Validate every input on the server, even if the Angular form validates it too.
- With SSR, configure allowed hosts (lesson 06) and be careful not to cache personalised
  server-rendered pages for everyone.

## How It Actually Works

The compiler knows, for every property and attribute binding, which **security context**
applies: `HTML` for `innerHTML`, `URL` for `href`/`src` on most elements,
`RESOURCE_URL` for things that load code (`<script src>`, `<iframe src>`), `STYLE` for
style bindings. It emits a sanitisation call alongside the binding instruction
(`ɵɵsanitizeHtml`, `ɵɵsanitizeUrl`, …). At runtime:

- The HTML sanitiser parses the value in an inert document, walks the tree and keeps only
  allow-listed elements and attributes (no `<script>`, no `on*` handlers, URL attributes
  re-checked), then serialises the result.
- The URL sanitiser allows safe schemes (`http`, `https`, `mailto`, `tel`, relative URLs,
  and a few others) and prefixes anything else with `unsafe:`.
- `RESOURCE_URL` values are not sanitised at all — they're rejected unless explicitly
  trusted, because no string check can make loading arbitrary code safe.

`bypassSecurityTrust*` wraps the value in a marker object (`SafeHtml`, `SafeUrl`, …). The
sanitisation functions see the marker and return the inner value untouched. That's the
whole mechanism — which is why it's only as safe as the code that created the value.

## Common mistakes

- **`bypassSecurityTrustHtml(userInput)`** to "make the HTML show up".
- **Direct DOM writes** via `nativeElement.innerHTML`.
- **API keys in `environment.ts`.**
- **Relying on client-side checks** for authorisation or validation.
- **Disabling XSRF protection** because the server "doesn't need it" — confirm how the
  API authenticates first.

## Exercise

1. Reproduce the security lab. Add `<iframe [src]="url">` with a user-controlled URL and
   read the error Angular throws; explain why `RESOURCE_URL` is stricter than `URL`.
2. Search a codebase you know for `bypassSecurityTrust` and `innerHTML`. For each hit,
   write down where the value comes from and whether it could ever contain user input.
3. Enable `autoCsp` in a project, build it, and inspect the generated policy. Serve the
   build, then in DevTools add `onclick="alert(1)"` to any element and click it: the
   inline handler should be blocked, with a CSP violation in the console. (Don't test by
   adding a `<script>` to `index.html` — `autoCsp` hashes the scripts it finds there, so
   that one would be allowed.)
4. Write a test proving that `HttpClient` sends `X-XSRF-TOKEN` on a relative POST but not
   on a cross-origin one.
