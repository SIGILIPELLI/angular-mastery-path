---
description: "Angular DI in depth: InjectionToken, useValue, useClass, useFactory, useExisting, multi providers, the injector tree and viewProviders."
---

# 03 · Dependency Injection in Depth

Level 1 used DI in its simplest form: a class marked `@Service()` is created once and
shared. Real applications need more — configuration objects that aren't classes, swapping
an implementation in tests or per environment, plug-in lists, and services scoped to part
of the UI or one feature's routes. All of that is the same mechanism with a few more
options.

## Tokens: what you ask for

`inject(X)` looks up a **token**. A class is a token. For anything that isn't a class — a
config object, a function, a string — create an `InjectionToken`:

```ts title="src/app/config.ts"
import { InjectionToken } from '@angular/core';

export interface AppConfig { apiUrl: string; retries: number }

export const APP_CONFIG = new InjectionToken<AppConfig>('APP_CONFIG');

// A token can carry its own default, making it tree-shakable and optional to provide.
export const FEATURE_FLAGS = new InjectionToken<Record<string, boolean>>('FEATURE_FLAGS', {
  factory: () => ({ newCheckout: false }),
});
```

The string is only a description for error messages. Two tokens with the same
description are still different tokens — identity is the object.

## Providers: how the value is made

A provider says "for this token, produce the value like this." The four recipes:

```ts title="src/app/app.config.ts (excerpt)"
providers: [
  // 1. A ready-made value
  { provide: APP_CONFIG, useValue: { apiUrl: 'https://api.example.com', retries: 2 } },

  // 2. A class to instantiate for an abstract token
  { provide: Storage, useClass: MemoryStorage },

  // 3. A factory function — can call inject() for its own dependencies
  {
    provide: APP_CONFIG,
    useFactory: () => ({
      apiUrl: 'https://staging.example.com',
      retries: inject(FEATURE_FLAGS)['newCheckout'] ? 5 : 1,
    }),
  },

  // 4. An alias to another token (same instance, second name)
  { provide: 'legacyApi', useExisting: Api },
]
```

A service then consumes the config like anything else:

```ts
@Service()
export class Api {
  private readonly config = inject(APP_CONFIG);
  url(path: string) { return `${this.config.apiUrl}${path}`; }
}
```

In our test, `Api.url('/books')` returned `https://api.example.com/books`, `Storage`
resolved to a `MemoryStorage` instance, the `FEATURE_FLAGS` default applied without any
provider, and `inject('legacyApi') === inject(Api)` was `true`.

### Abstract classes as tokens

An abstract class is a good token for "some implementation of this contract": it works
as a TypeScript type *and* as a runtime value, unlike an interface.

```ts
export abstract class Storage {
  abstract get(key: string): string | null;
  abstract set(key: string, value: string): void;
}
export class MemoryStorage extends Storage { /* Map-based, for tests and SSR */ }
export class LocalStorage extends Storage { /* wraps window.localStorage */ }
```

Components inject `Storage` and never know which one they got.

### Multi providers

`multi: true` collects every provider for a token into an array — the standard way to let
several parts of an app contribute plug-ins (validators, interceptors, initialisers):

```ts
{ provide: VALIDATORS, useValue: (v: string) => (v ? null : 'empty'), multi: true },
{ provide: VALIDATORS, useValue: (v: string) => (v.length < 20 ? null : 'too long'), multi: true },
```

`inject(VALIDATORS)` returned both functions; calling each with `''` gave
`['empty', null]`.

### Packaging providers

Libraries (and your own feature setup code) expose **provider functions** instead of
asking users to write provider objects — just like `provideRouter()` and
`provideHttpClient()`:

```ts
import { makeEnvironmentProviders } from '@angular/core';

export function provideStorage(kind: 'memory' | 'local') {
  return makeEnvironmentProviders([
    { provide: Storage, useClass: kind === 'memory' ? MemoryStorage : LocalStorage },
  ]);
}
```

`makeEnvironmentProviders` marks them as app/route-level only, so the compiler rejects
putting them in a component's `providers`.

### Running code at startup

`provideAppInitializer(() => ...)` runs a function (inside an injection context) during
bootstrap, before the first render. Return a promise or observable to delay startup
until, for example, remote configuration has loaded. Keep it fast — users stare at a
blank page until it finishes.

## The injector tree

There are two hierarchies:

1. **Environment injectors**: the root injector (from `bootstrapApplication`'s
   providers), plus one per lazy-loaded route that declares `providers` (lesson 07). You
   can also create one yourself with `createEnvironmentInjector(providers, parent)`.
2. **Element injectors**: one for each component or directive instance that has
   `providers` or `viewProviders`.

`inject()` in a component first walks **up the element injector tree** (the component
itself, then its parent in the rendered DOM, and so on), and only if nothing there provides
the token does it continue into the **environment injector** chain up to root. The first
match wins.

### `providers` vs `viewProviders`

We rendered this:

```ts
@Component({ selector: 'app-parent', imports: [Child],
  providers: [{ provide: Theme, useValue: { name: 'dark' } }],
  template: `<app-child />` })
class Parent {}

@Component({ selector: 'app-vp', imports: [Child],
  viewProviders: [{ provide: Theme, useValue: { name: 'view-only' } }],
  template: `<app-child /><ng-content />` })
class Vp {}

// Host template:
// <app-parent />
// <app-vp><app-child id="projected" /></app-vp>
```

where each `Child` injects `Theme` three ways — plain, with `{ self: true }` and with
`{ skipSelf: true }`, all `optional`. The output:

```text
child dark       | self null | skipSelf dark        ← inside Parent's template
child view-only  | self null | skipSelf view-only   ← inside Vp's own template
child undefined  | self null | skipSelf null        ← projected into Vp via <ng-content>
```

`viewProviders` are visible only to a component's **own template**, not to content
projected into it from outside. Library authors use that to keep internal services
private to a component while still letting its template share them.

### Resolution options

The second argument to `inject()` changes the lookup:

| Option | Effect |
|--------|--------|
| `{ optional: true }` | Return `null` instead of throwing `NG0201` when nothing provides it |
| `{ self: true }` | Look only at the current element's injector |
| `{ skipSelf: true }` | Start at the parent — how a nested component finds "the enclosing one of me" (a nested menu finding its parent menu) |
| `{ host: true }` | Stop the search at the host component's boundary |

`self: true` was `null` for every child above because `Child` itself provides nothing.

## How It Actually Works

Each injector stores a map of **records**: token → (factory, cached value or "not yet
created"). `inject(token)` asks the current injector for the record; on a miss it
delegates to its parent. On a hit with no cached value, it runs the factory **with that
injector set as the current injection context**, caches the result, and returns it.
That's why `useFactory` functions and the field initialisers of a `useClass` class can
call `inject()` themselves — and why dependencies are resolved from the injector that
*owns the provider*, not from the component that asked.

For element injectors, Angular doesn't keep a map per element. Each component view
stores the providers declared on its nodes, and a compact **bloom filter** per view lets
Angular skip views that definitely don't contain a given token while walking up. Only
when the filter says "maybe" does it inspect the actual providers. This keeps DI lookup
cheap even in deep component trees.

`multi: true` changes the record's factory: instead of "create this one value", it
becomes "create each contributed value and return the array." Providing the same token
without `multi` in the same injector replaces the earlier record — last one wins.

Tokens with a `factory` in their definition (like `FEATURE_FLAGS`) and `@Service()`
classes work the same way at the root level: when the root injector misses, it checks
the token's own definition for a default factory before giving up with:

```text
NG0201: No provider found for `InjectionToken APP_CONFIG`.
```

## Common mistakes

- **Using an interface as a token.** Interfaces vanish at runtime. Use an
  `InjectionToken<MyInterface>` or an abstract class.
- **Providing a root singleton again lower down** by accident, creating a second instance
  with its own state.
- **Forgetting `multi: true` on one of several contributions**, which silently replaces
  the whole list with a single value.
- **Heavy work in `provideAppInitializer`**, delaying first paint. Load only what the
  first screen truly needs.
- **Expecting `viewProviders` to reach projected content.** They don't — use `providers`.

## Exercise

1. Create a `Clock` abstract class with `now(): Date`, a `SystemClock` implementation, and
   a `FixedClock` for tests. Provide `SystemClock` at root with `useClass`.
2. Write a `GreetingService` that says "Good morning/afternoon/evening" based on
   `inject(Clock).now()`. Test it with `FixedClock` supplied via `TestBed` providers.
3. Add a `GREETING_SUFFIXES` multi-provider token with two contributions and append all
   of them to the greeting.
4. Build a `Tabs`/`Tab` component pair where each `Tab` injects its parent `Tabs` with
   `inject(Tabs)` to register itself. Then nest a second `Tabs` inside a tab and use
   `skipSelf` somewhere to prove you understand which instance each `Tab` receives.
