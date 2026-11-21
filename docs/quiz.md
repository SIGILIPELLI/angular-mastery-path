---
title: "Final Quiz"
description: "Test what you have learned across all four levels of Angular Mastery Path with a 19-question final quiz."
---

# Final Quiz

19 questions drawn from all four levels of this course. You need **70% (14 out of 19)** to pass —
you can retake it as many times as you like. Your best result is saved in this browser.

<div id="mp-quiz-root"></div>

<script type="application/json" id="mp-quiz-data">
[
  {
    "question": "What does this component's `count` display after two clicks?\n```typescript\nreadonly count = signal(0);\nprotected inc() { this.count.update((n) => n + 1); }\n```\n",
    "options": [
      "1",
      "NaN",
      "2, since update computes the next value from the current one",
      "0"
    ],
    "correctIndex": 2,
    "explanation": "signal.update(fn) passes the current value."
  },
  {
    "question": "A signal holds `['a']`, and `list().push('b')` mutates it in place. What does `computed(() => list().length)` return afterwards?",
    "options": [
      "0",
      "2",
      "Still 1: in-place mutation doesn't notify; replace with list.update(l => [...l, 'b'])",
      "It throws"
    ],
    "correctIndex": 2,
    "explanation": "Signals compare with Object.is. Verified with a minimal signal/computed implementation: 1, 1, then 3 after replacement."
  },
  {
    "question": "price = signal(20), quantity = signal(3), total = computed(() => price() * quantity()). After quantity.set(4), what is total()?",
    "options": [
      "60",
      "80",
      "24",
      "23"
    ],
    "correctIndex": 1,
    "explanation": "computed tracks the signals it reads. Verified by running: 60 then 80."
  },
  {
    "question": "A computed's function reads signal `s`. You call `s.set(1)` when s is already 1. Does the computed recompute?",
    "options": [
      "Yes, always",
      "Only once",
      "No: Object.is says nothing changed, so no notification",
      "Only in dev"
    ],
    "correctIndex": 2,
    "explanation": "Verified: the function ran once across redundant sets."
  },
  {
    "question": "Which template renders the list correctly?",
    "options": [
      "@for (b of books(); track b.id) { <li>{{ b.title }}<\/li> } @empty { <li>None<\/li> }",
      "<li *ngFor=\"let b of books\">",
      "<li v-for>",
      "@for (b of books()) { }"
    ],
    "correctIndex": 0,
    "explanation": "@for needs track; @empty handles no items. *ngFor is deprecated since v20."
  },
  {
    "question": "With routes [ {path:'', redirectTo:'books', pathMatch:'full'}, {path:'books'}, {path:'books/:id'}, {path:'**', redirectTo:'books'} ], where does /books/42 go, and what are its params?",
    "options": [
      "books, with no params",
      "The books/:id route with { id: '42' }",
      "The empty redirect",
      "The wildcard"
    ],
    "correctIndex": 1,
    "explanation": "The router takes the first match in order. Verified with a small matcher."
  },
  {
    "question": "Where must the '**' route go, and why?",
    "options": [
      "Last: routes are checked in order and the first match wins, so it would otherwise swallow everything",
      "Anywhere",
      "First",
      "It's not allowed"
    ],
    "correctIndex": 0,
    "explanation": "The same reason `path: ''` redirects need pathMatch: 'full'."
  },
  {
    "question": "A component with OnPush (default in v22) mutates a plain field inside a setTimeout. Does the view update?",
    "options": [
      "Yes, timers always trigger it",
      "Only in production",
      "No: zoneless Angular only updates after a notification such as a signal read in the template changing; use a signal",
      "Only for numbers"
    ],
    "correctIndex": 2,
    "explanation": "Plain assignments aren't on the notification list."
  },
  {
    "question": "Where should a filtered list derived from `items` and `query` signals live?",
    "options": [
      "In the template only",
      "In localStorage",
      "In a computed(), never stored separately",
      "In a signal updated by an effect"
    ],
    "correctIndex": 2,
    "explanation": "Derive, don't copy."
  },
  {
    "question": "A child needs a required numeric input. Which declaration is right?",
    "options": [
      "max = model()",
      "readonly max = input.required({ transform: numberAttribute })",
      "@Input() max",
      "max = input()"
    ],
    "correctIndex": 1,
    "explanation": "input.required has no default and the parent must supply it."
  },
  {
    "question": "Which provider recipe gives the same instance a second name?",
    "options": [
      "useClass",
      "useValue",
      "useFactory",
      "useExisting"
    ],
    "correctIndex": 3,
    "explanation": "An alias to another token."
  },
  {
    "question": "A service needs configuration that isn't a class. What do you inject?",
    "options": [
      "An InjectionToken<AppConfig> provided with useValue",
      "A pipe",
      "A component",
      "A string literal class"
    ],
    "correctIndex": 0,
    "explanation": "Tokens are the lookup keys for non-class values."
  },
  {
    "question": "A search box should wait until typing pauses and cancel stale requests. Which operators?",
    "options": [
      "debounceTime then distinctUntilChanged and switchMap",
      "tap",
      "startWith",
      "map"
    ],
    "correctIndex": 0,
    "explanation": "switchMap cancels the previous inner observable."
  },
  {
    "question": "Which API loads data for a component when an `id` input changes and cancels the in-flight request?",
    "options": [
      "An effect writing a field",
      "A global variable",
      "httpResource(() => `/api/users/${this.id()}`)",
      "HttpClient in ngOnInit"
    ],
    "correctIndex": 2,
    "explanation": "Resources keep value, error and isLoading signals correct."
  },
  {
    "question": "Which guard stops a user leaving a form with unsaved changes?",
    "options": [
      "canMatch",
      "resolve",
      "canDeactivate",
      "canActivate"
    ],
    "correctIndex": 2,
    "explanation": "canActivate controls entry; canMatch chooses between configs."
  },
  {
    "question": "User text includes <script>. How does Angular render it in `{{ comment }}`?",
    "options": [
      "As escaped text: interpolation never interprets HTML",
      "Throws",
      "Executes it",
      "Strips tags silently"
    ],
    "correctIndex": 0,
    "explanation": "bypassSecurityTrustHtml on user input would defeat this."
  },
  {
    "question": "Which render mode fits a page that depends on localStorage?",
    "options": [
      "Prerender",
      "Server",
      "Client",
      "Eager"
    ],
    "correctIndex": 2,
    "explanation": "The server has no localStorage."
  },
  {
    "question": "Which command upgrades Angular packages and runs their migrations?",
    "options": [
      "ng update @angular/core@22 @angular/cli@22 (one major at a time)",
      "ng serve",
      "npm install latest",
      "ng upgrade"
    ],
    "correctIndex": 0,
    "explanation": "Commit first, then build and test."
  },
  {
    "question": "Which structure scales for large apps?",
    "options": [
      "One flat folder",
      "Feature-first folders with feature/ui/data-access roles, dependency rules enforced in CI",
      "Folders by file type",
      "Folders per developer"
    ],
    "correctIndex": 1,
    "explanation": "Features expose a public API via index.ts."
  }
]
</script>

## Coding challenges

Try these on your own computer, then compare your output. Nothing you write is sent anywhere.

### A tiny signal

Implement signal(initial) with a read function plus set(v) and update(fn). It must only notify subscribers when the new value differs by Object.is. The provided computed() tracks reads. Run the checks to see why mutating an array in place leaves a computed stale.

**Starter code**

```typescript
let active = null;
export function signal(initial) {
  // return a read function with .set(v) and .update(fn); notify only when !Object.is(new, old)
}
export function computed(fn) {
  let cache, dirty = true;
  const mark = () => { dirty = true; };
  const read = () => {
    if (dirty) { const prev = active; active = mark; try { cache = fn(); } finally { active = prev; } dirty = false; }
    return cache;
  };
  return read;
}
const list = signal(['a']);
const n = computed(() => list().length);
console.log(n());
list().push('b');
console.log(n());
list.update((l) => [...l, 'c']);
console.log(n());
const price = signal(20), qty = signal(3);
const total = computed(() => price() * qty());
console.log(total()); qty.set(4); console.log(total());
const s = signal(1); let runs = 0; const d = computed(() => { runs++; return s() * 2; });
d(); d(); s.set(1); d(); console.log(runs); s.set(2); console.log(d(), runs);
```

**Expected output**

```text
1
1
3
60
80
1
4 2
```

<details><summary>Show solution and explanation</summary>

<pre><code>let active = null;
export function signal(initial) {
  let value = initial; const subs = new Set();
  const read = () =&gt; { if (active) subs.add(active); return value; };
  read.set = (v) =&gt; { if (Object.is(v, value)) return; value = v; subs.forEach((s) =&gt; s()); };
  read.update = (fn) =&gt; read.set(fn(value));
  return read;
}
export function computed(fn) {
  let cache, dirty = true;
  const mark = () =&gt; { dirty = true; };
  const read = () =&gt; {
    if (dirty) { const prev = active; active = mark; try { cache = fn(); } finally { active = prev; } dirty = false; }
    return cache;
  };
  return read;
}
const list = signal([&#x27;a&#x27;]);
const n = computed(() =&gt; list().length);
console.log(n());
list().push(&#x27;b&#x27;);
console.log(n());
list.update((l) =&gt; [...l, &#x27;c&#x27;]);
console.log(n());
const price = signal(20), qty = signal(3);
const total = computed(() =&gt; price() * qty());
console.log(total()); qty.set(4); console.log(total());
const s = signal(1); let runs = 0; const d = computed(() =&gt; { runs++; return s() * 2; });
d(); d(); s.set(1); d(); console.log(runs); s.set(2); console.log(d(), runs);
</code></pre>

<p>A signal notifies only through set/update and compares with Object.is, so list().push(&#x27;b&#x27;) is invisible while list.update(l =&gt; [...l, &#x27;c&#x27;]) is seen. computed tracks what it reads and recomputes only when notified. Verified by running the solution in Node.</p>
</details>

### First match wins

Write match(routes, url) returning the first route that matches: '' matches only the empty URL when pathMatch is 'full', ':name' segments capture params, '**' matches anything. Print where each URL ends up.

**Starter code**

```typescript
function match(routes, url) {
  // your code
}
const routes = [
  { path: '', redirectTo: 'books', pathMatch: 'full' },
  { path: 'books' }, { path: 'books/:id' }, { path: '**', redirectTo: 'books' },
];
for (const u of ['/', '/books', '/books/42', '/nope/x']) { const m = match(routes, u); console.log(u, '->', m.redirectTo ?? m.path, JSON.stringify(m.params ?? {})); }
```

**Expected output**

```text
/ -> books {}
/books -> books {}
/books/42 -> books/:id {"id":"42"}
/nope/x -> books {}
```

<details><summary>Show solution and explanation</summary>

<pre><code>function match(routes, url) {
  const segs = url.split(&#x27;/&#x27;).filter(Boolean);
  for (const r of routes) {
    if (r.path === &#x27;**&#x27;) return r;
    const p = r.path.split(&#x27;/&#x27;).filter(Boolean);
    if (r.path === &#x27;&#x27;) { if (r.pathMatch === &#x27;full&#x27; ? segs.length === 0 : true) return r; continue; }
    if (p.length !== segs.length) continue;
    if (p.every((x, i) =&gt; x.startsWith(&#x27;:&#x27;) || x === segs[i])) return { ...r, params: Object.fromEntries(p.map((x, i) =&gt; x.startsWith(&#x27;:&#x27;) ? [x.slice(1), segs[i]] : null).filter(Boolean)) };
  }
  return null;
}
const routes = [
  { path: &#x27;&#x27;, redirectTo: &#x27;books&#x27;, pathMatch: &#x27;full&#x27; },
  { path: &#x27;books&#x27; }, { path: &#x27;books/:id&#x27; }, { path: &#x27;**&#x27;, redirectTo: &#x27;books&#x27; },
];
for (const u of [&#x27;/&#x27;, &#x27;/books&#x27;, &#x27;/books/42&#x27;, &#x27;/nope/x&#x27;]) { const m = match(routes, u); console.log(u, &#x27;-&gt;&#x27;, m.redirectTo ?? m.path, JSON.stringify(m.params ?? {})); }
</code></pre>

<p>The router checks routes in order and takes the first match, so the wildcard must be last and an empty-path redirect needs pathMatch: &#x27;full&#x27;. Verified by running the solution in Node.</p>
</details>
