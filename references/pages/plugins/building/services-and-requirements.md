# Shared State and Requirements

Original source URL: https://seyfert-web-git-seyfert-v5-tiramisulabs.vercel.app/docs/plugins/building/services-and-requirements

Coverage reference: plugins.md

Verification status: Source-verified (src/client/plugins.ts + plugins/{shared,api,types,registry}.ts, the authoritative Seyfert source)

## Page Summary

Covers plugin-to-plugin shared state (lazily-built runtime services keyed by `createSharedKey`),
typed shared names via `RegisteredPluginShared` augmentation, plugin `imports` vs `requires`
(presence/semver/capability requirements), and `api.diagnostics.warn(...)` for degraded states.
Shared state lets plugins expose runtime objects to each other WITHOUT adding `client.*` properties
(use `api.client` / the `client` map for that instead). All APIs verified against
`src/client/plugins/{shared,api,types,registry}.ts`.

## Key APIs (verified)

- `createSharedKey<T>()(name)` curried form: carries value type `T` on the key. Returns a frozen
  `SharedKey<T, Name>`. (shared.ts:25,30)
- `createSharedKey(name)` direct form: if `name` is a key of `RegisteredPluginShared`, returns
  `SharedKey<RegisteredPluginShared[name], name>`; otherwise `SharedKey<unknown, name>`. (shared.ts:22-26)
- `api.shared.set(key | name, factory, opts?)` — factory is `(client: BaseClient & E) => T`, called
  lazily on first access (E = imports' + own client extensions). `opts`:
  `{ dispose?: (value) => Awaitable<void>; override?: boolean }` (`PluginSharedOptions<T>`).
  (types.ts:51-54,450-460; api.ts:504-510)
- `api.shared.remove(...names | keys)` and `api.shared.has(name | key)`. (api.ts:511-519)
- `client.shared` is a `PluginSharedRegistry` with: `get(key|name)` -> value | `undefined`;
  `unwrap(key|name)` -> value, THROWS if the name is not registered; `has(key|name)` -> boolean.
  (shared.ts:108-125; types.ts:43-49)
- `SharedValue<K>` extracts the value type from a `SharedKey`. (types.ts:41)
- `createPlugin({ name, version?, meta?, imports, requires, client, ctx, register, setup, teardown, ... })`
  — `imports?: readonly AnySeyfertPlugin[]`, `requires?: readonly PluginRequirementInput[]`. `version`
  is accepted as a passthrough field and survives onto the output. (plugins.ts:165-265, types.ts:492-512)
- `PluginRequirementInput` = `` `plugin:${string}` `` | `{ req: 'plugin:..'; range?: SemverRange; optional?: boolean }`
  | `{ capability: SharedKey; optional?: boolean }`. `req` and `capability` are MUTUALLY EXCLUSIVE;
  `range` is only valid alongside `req`. (types.ts:90-93)
- `SemverRange` = exact `x.y.z`, `>=x.y.z`, `^x.y.z`, or `~x.y.z`. (types.ts:88-89)
- `api.has(req)` — accepts ONLY `` `plugin:${string}` ``; returns boolean. Returns `false` for any
  non-`plugin:` string, so it CANNOT feature-check capabilities. (api.ts:125-127, registry.ts:497-503)
- `api.diagnostics.warn(message, options?)` — `options`: `{ code?; phase?; data? }`. Default phase is
  `'register'`, severity is always `'warn'`. (api.ts:557-567, types.ts:468-473)
- Root imports from `'seyfert'`: `createPlugin`, `createSharedKey`, `definePlugins`,
  `createPluginFactory`, `PluginOrder`, and types `RegisteredPluginShared`, `SharedKey`, `SharedValue`,
  `PluginSharedOptions`, `PluginRequirementInput`, `SemverRange`. (plugins.ts:40-158)

## Code Examples (verified)

### 1. Shared service registered under a typed key

```ts
import { createPlugin, createSharedKey } from 'seyfert';

class LedgerService {
    readBalance(userId: string) {
        return 100;
    }
    close() {}
}

// curried call carries the value type on the key
export const ledgerKey = createSharedKey<LedgerService>()('ledger');

export const ledgerPlugin = createPlugin({
    name: 'ledger',
    register(api) {
        // factory runs lazily on first access; receives the client
        api.shared.set(ledgerKey, () => new LedgerService(), {
            dispose: ledger => ledger.close(), // runs on teardown
        });
    },
});
```

### 2. Resolving shared values from the client

```ts
const ledger = client.shared.get(ledgerKey);            // LedgerService | undefined
const requiredLedger = client.shared.unwrap(ledgerKey); // throws if not registered
const present = client.shared.has(ledgerKey);           // boolean, does not build it
```

### 3. Typed app-wide shared names via module augmentation

```ts
import type { RegisteredPluginShared } from 'seyfert';

declare module 'seyfert' {
    interface RegisteredPluginShared {
        ledger: LedgerService;
    }
}

// now plain string access is typed (no key object needed):
client.shared.get('ledger');    // LedgerService | undefined
client.shared.unwrap('ledger'); // LedgerService
```

### 4. Imports + requirements

```ts
const storage = storagePlugin();

export const economy = createPlugin({
    name: 'economy',
    imports: [storage], // bring storage along; orders economy after it; dedupes the instance
    requires: [
        'plugin:storage',                          // must be present (throws if missing)
        { req: 'plugin:redis', optional: true },   // missing -> diagnostics warn, no throw
        { req: 'plugin:storage', range: '^2.0.0' },// present within a semver range
        { capability: ledgerKey },                 // a shared capability must exist
    ],
    register(api) {
        // api.has only checks plugin:* requirements
        if (api.has('plugin:redis')) {
            api.diagnostics.warn('Redis support enabled.', { code: 'redis-enabled' });
        }
    },
});
```

### 5. Provider + consumer pair (capability, not `client.*`)

A self-contained two-plugin flow: one plugin publishes a service as a capability, another `requires`
it and consumes it inside a command. Nothing touches `client.*`, so there is no augmentation tug-of-war.

```ts
import { createPlugin, createSharedKey, type SharedValue } from 'seyfert';

// ---- provider ----
class RateLimiter {
    private hits = new Map<string, number>();
    take(userId: string, max = 5) {
        const n = (this.hits.get(userId) ?? 0) + 1;
        this.hits.set(userId, n);
        return n <= max;
    }
    reset() {
        this.hits.clear();
    }
}

export const limiterKey = createSharedKey<RateLimiter>()('rate-limiter');

export const limiterPlugin = createPlugin({
    name: 'rate-limiter',
    version: '1.2.0', // read back by semver `requires`
    register(api) {
        api.shared.set(limiterKey, () => new RateLimiter(), {
            dispose: limiter => limiter.reset(),
        });
    },
});

// ---- consumer ----
type Limiter = SharedValue<typeof limiterKey>; // RateLimiter

export const featurePlugin = createPlugin({
    name: 'feature',
    imports: [limiterPlugin],                            // own the ordering
    requires: [{ capability: limiterKey }],             // fail fast if no limiter exists
    register(api) {
        api.commands.observe({
            onBeforeMiddlewares(ctx) {
                // unwrap = mandatory; throws with a clear message if missing
                const limiter = ctx.client.shared.unwrap(limiterKey) as Limiter;
                if (!limiter.take(ctx.author.id)) {
                    api.diagnostics.warn('rate limit hit', { code: 'ratelimited' });
                }
            },
        });
    },
});
```

### 6. Optional dependency with graceful fallback

`requires` with `optional: true` downgrades a missing dependency to a diagnostic instead of a throw.
Pair it with `api.has(...)` to branch behaviour at register time.

```ts
export const analytics = createPlugin({
    name: 'analytics',
    requires: [{ req: 'plugin:metrics', optional: true }],
    register(api) {
        if (api.has('plugin:metrics')) {
            api.rest.observe({
                onSuccess({ method, url }) {
                    // report to the metrics plugin's sink...
                },
            });
        } else {
            // optional peer absent: code 'missing-optional-requirement' is already
            // emitted by the runtime; add your own context if useful.
            api.diagnostics.warn('Metrics plugin not installed; REST timing disabled.', {
                code: 'metrics-disabled',
            });
        }
    },
});
```

### 7. Versioned dependency with a semver range

```ts
export const consumer = createPlugin({
    name: 'consumer',
    requires: [
        { req: 'plugin:storage', range: '^2.0.0' }, // 2.x.x only
        { req: 'plugin:cache', range: '>=1.4.0' },  // 1.4.0 or newer
    ],
});
// version resolves from plugin.version, else plugin.meta.version.
// Ranges: exact, '>=x.y.z', '^x.y.z', '~x.y.z'. A required-but-out-of-range version THROWS;
// add optional:true to downgrade to a 'missing-optional-requirement' warn.
```

### 8. Replacing a shared service (override)

```ts
// A later plugin can claim a name already owned by an earlier one, but ONLY with override.
api.shared.set(ledgerKey, () => new FasterLedgerService(), { override: true });
// Without override, a second set for the same name throws an attributed conflict error.
// override disposes the previous instance (its `dispose` runs) before swapping.
```

## Behavior notes (verified)

- Shared names are claimed once. A second `set` for the same name throws an attributed conflict
  error unless the later call passes `{ override: true }`. Override queues disposal of the prior
  instance and records an override diagnostic. (shared.ts:38-64)
- `unwrap` throws `Shared value "<name>" is not registered.` when the name was never registered;
  `get` returns `undefined`. Both build the value lazily and cache the instance. (shared.ts:108-125)
- The shared factory receives the client and runs once; the cached instance is cleared on plugin
  teardown, invoking `dispose` if provided. A throwing factory is wrapped in a plugin error.
  (shared.ts:87-106, 127-158)
- `imports` affects ordering and dedupes the same plugin instance; `requires` only validates presence.
  A missing REQUIRED requirement throws during resolution; a missing OPTIONAL one emits a warn
  diagnostic with code `missing-optional-requirement`. (registry.ts:1099-1141)
- Plugin requirements are validated during resolution; CAPABILITY requirements are validated AFTER
  all `register` calls run (`validatePluginRequirements(registry, 'capability')`), so a capability
  registered in `register` satisfies it. (plugins.ts:360, registry.ts:1099-1107)
- Semver ranges support exact, `>=`, `^`, `~`. `^` is caret semantics (locks major when major>0,
  minor when major=0, etc.); `~` locks major+minor. Version comes from `plugin.version` or
  `plugin.meta.version`. (registry.ts:1193-1233)
- `requires` is NOT a priority/ordering system: if a required plugin ends up ordered after the
  requiring plugin and the requiring plugin mutates its contributions, Seyfert emits a
  `requires-unordered` warn diagnostic; put the dependency in `imports` to own the order.
  (registry.ts:1109-1119)
- Diagnostics surface at `client.plugins.diagnostics` (array of `PluginDiagnostics`); the individual
  warnings live in each entry's `messages: PluginDiagnosticMessage[]` carrying `plugin`, `instanceId?`,
  `index`, `phase`, `severity`, `code?`, `message`, `data?`. (types.ts:109-118, 514-543)

## Doc vs Source Corrections

- Docs say `client.shared` warnings/diagnostics appear in `client.plugins.diagnostics` "with plugin
  name, index, phase, code, and message" -> src: those fields live on
  `client.plugins.diagnostics[i].messages[]` (`PluginDiagnosticMessage`), one level deeper than the
  prose implies. (types.ts:542)
- Docs imply `api.has(...)` can confirm any dependency; src: `api.has` only accepts `` `plugin:${string}` ``
  and returns `false` for non-plugin strings, so capabilities cannot be checked through it. Use
  `api.shared.has(key)` / `client.shared.has(key)` for capabilities. (api.ts:125-127, registry.ts:497-503)
- Otherwise the page's examples and rules match source exactly.

## Source Anchors

- src/client/plugins.ts (createPlugin / definePlugins / createPluginFactory / re-exports; capability
  validation at :360)
- src/client/plugins/shared.ts (createSharedKey, addPluginShared, createSharedRegistry, dispose)
- src/client/plugins/api.ts (api.shared, api.has, api.diagnostics.warn)
- src/client/plugins/types.ts (SharedKey, SharedValue, PluginSharedRegistry, PluginSharedOptions,
  PluginRequirementInput, SemverRange, RegisteredPluginShared, SeyfertPlugin, PluginDiagnostics)
- src/client/plugins/registry.ts (validatePluginRequirements, hasPluginRequirement, semver matching)
- src/client/base.ts (client.shared)

## Common patterns / gotchas

- Service between plugins: use SHARED STATE, not `client.*`. Build the key with the curried
  `createSharedKey<T>()('name')` so consumers get full typing without any module augmentation.
- App-wide named services consumers reference by string: augment `RegisteredPluginShared` in a
  `declare module 'seyfert'` block; then `createSharedKey('name')` and string `get/unwrap/has` are typed.
- Mandatory vs optional consumption: `unwrap` (fail fast) when the value MUST exist; `get`/`has` when
  optional. `dispose` cleans up on teardown.
- `imports` vs `requires`: choose `imports` when your plugin OWNS the dependency and its ordering;
  choose `requires` only to ASSERT presence/version/capability. Required-but-missing throws at resolve
  time; mark with `{ optional: true }` to downgrade to a diagnostic. Pairing `imports:[x]` with
  `requires:['plugin:x']` is the safe combo: own the order AND assert the version.
- GOTCHA: `api.has` is plugin-only. For capability gating use `api.shared.has(key)` / `client.shared.has(key)`.
- GOTCHA: `{ range }` only pairs with `{ req }`; `{ capability }` cannot take a range, and you cannot
  pass both `req` and `capability` in one entry (throws during normalization). (registry.ts:1063-1086)
- GOTCHA: requiring a plugin does NOT reorder it. If you depend on the dependency's contributions being
  set up first, put it in `imports` — a bare `requires` that ends up ordered later emits `requires-unordered`.
- GOTCHA: capability requirements are checked after all `register` calls, so `{ capability }` is
  satisfied even if the providing plugin registers it in its own `register`; plain `plugin:` requirements
  are checked at resolution (presence/version).
- To version a plugin for semver `requires`, set `version: 'x.y.z'` (or `meta: { version: 'x.y.z' }`)
  on `createPlugin` — both are read by the resolver.

## Agent Guidance

- Default to shared state for plugin-to-plugin services; reserve `client`/`ctx` extension maps for
  things end users call directly on the client/context.
- When writing a reusable plugin pair, ship the `SharedKey` as a named export so consumers can both
  `requires: [{ capability: key }]` and `client.shared.unwrap(key)` with full typing.
- Read diagnostics during development: log `client.plugins.diagnostics.flatMap(d => d.messages)` to
  surface missing-optional-requirement, requires-unordered, contribution-override, etc.
