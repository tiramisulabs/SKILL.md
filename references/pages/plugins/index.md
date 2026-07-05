# Plugins: Introduction

Original source URL: https://seyfert-web-git-seyfert-v5-tiramisulabs.vercel.app/docs/plugins/index

Coverage reference: plugins.md

Verification status: Source-verified (the authoritative Seyfert source; src/client/plugins.ts + src/client/plugins/{types,api,registry,shared,order,errors}.ts)

## Page Summary

Seyfert v5 ships a first-class, typed plugin runtime that replaces the old pattern of manually mutating `Client`. A plugin is a single declarative object passed to `new Client({ plugins })`; it *declares* what it contributes (client/ctx helpers, command/component/modal/event/middleware handlers, shared state, cache resources & lang overlays, gateway/REST hooks, lifecycle) and Seyfert wires it up with types, ordering, lifecycle, and diagnostics. Plugins are also the unit of sharing — package once, install with one line. This index page is conceptual; the authoring API lives under `plugins/building/*` and official packages under `plugins/official`.

## Key APIs (verified)

All of the following are re-exported from the package root `'seyfert'` (barrel: `src/index.ts` -> `src/client/index.ts` -> `src/client/plugins.ts`). No deep import is required.

- `definePlugins(...plugins)` / `definePlugins(plugins[])` — identity helper that keeps the plugin tuple typed; accepts either a spread or a single array and returns it unchanged. (`src/client/plugins.ts:283`)
- `createPlugin(plugin)` — authoring entry; returns the input object unchanged at runtime, but its overloads infer extension (`E`), context (`C`), imports (`I`), middleware map (`M`), `meta`, and arbitrary extra keys. (`src/client/plugins.ts:240`)
- `createPluginFactory({ defaults, validate?, factory })` — builds an `(options?: Partial<O>) => Plugin` factory; merges `defaults` with caller options, runs optional `validate`, then `factory`, wrapping any throw via `wrapPluginError`. (`src/client/plugins.ts:267`)
- `createSharedKey(...)` — typed shared-state key factory; `createSharedKey('name')` or `createSharedKey<T>()('name')`, returns a frozen `SharedKey<T, Name>`. (`src/client/plugins/shared.ts:22`)
- `createContextScope(scope)` — identity helper typing a `ContextScope`. (`src/client/plugins.ts:291`)
- `PluginOrder` enum — `Before = 'before'`, `After = 'after'`; ordering opt is `PluginOrderOpt = PluginOrder.Before | PluginOrder.After | number`. (`src/client/plugins/types.ts:69`)
- Errors: `SeyfertPluginError`, `SeyfertPluginAggregateError`, type `SeyfertPluginErrorCode` (`'PLUGIN_FAILED' | 'PLUGIN_TEARDOWN_FAILED'`). (`src/client/plugins.ts:40`, errors.ts:3)

Plugin object shape — `interface SeyfertPlugin<E, C, I, M>` (`src/client/plugins/types.ts:492`). Notable fields:
- `name: string` (required); optional `instanceId?`, `version?`, `meta?`.
- `imports?: I` (other plugins this composes; their `E`/`C`/`M` flow into `client`/`ctx`/middleware typing), `requires?: readonly PluginRequirementInput[]`.
- `client?: PluginClientMap<E,I>` — map of `key -> (client) => value`, added to `client.*` (app-wide).
- `ctx?: PluginContextMap<C,I,E>` — map of `key -> (interaction, client) => value`, added to `ctx.*` (per-interaction).
- `middlewares?: M`, `globalMiddlewares?: readonly (keyof M & string)[]`.
- Lifecycle/registration methods: `options(current)`, `register(api)`, `setup(client, api?)`, `teardown(client, api?)`.

Lifecycle hook *names* (the `SeyfertPluginHooks` map subscribed via `api.hooks.on(...)`, distinct from `setup`/`teardown`) — `src/client/plugins/types.ts:200`:
`plugins:ready`, `plugins:setupComplete`, `commands:beforeLoad`, `commands:afterLoad`, `components:beforeLoad`, `components:afterLoad`, `events:beforeLoad`, `events:afterLoad`, `client:close`.
NOTE the payloads differ: `commands:afterLoad`/`components:afterLoad` receive a single `[metadata]` object (`PluginLoadedMetadata`), while `events:afterLoad` (and every `*:beforeLoad`) receive `[client, dir]`. `events:afterLoad` does NOT get a metadata object. (`types.ts:203-208`)

Runtime phases: `register(api)` runs synchronously during `resolveClientPlugins`; `setup` runs on client `start()` and `teardown` on `close()`. Setup failures roll back (event listeners cleaned, shared values disposed, dynamic contributions reverted) and already-completed plugins are torn down in reverse order, aggregating errors into `SeyfertPluginAggregateError`. (`src/client/plugins.ts:656` setup, `:717` teardown)

The full authoring surface is `SeyfertPluginApi` (`src/client/plugins/types.ts:373`): `events` (`on`/`once`/`onAny`/`onError`), `commands` (`add`/`remove`/`observe`/`defaults`), `components`, `modals`, `middlewares` (`add`/`remove`), `rest` (`observe`), `hooks` (`on`), `handlers` (`construct`/`transform`), `autocomplete` (`wrap`), `gateway` (`addIntents`, `wrapSendPayload`, `onDispatch`), `cache` (`resource`), `shared` (`set`/`remove`/`has`), `langs` (`contribute`), `reload()`, `diagnostics` (`warn`), `options` (`set`), and `has(req)`. Teardown receives the narrowed `SeyfertPluginTeardownApi` (only `has`, `diagnostics`, `shared.has`). (`types.ts:479`)

Ordering bands (`src/client/plugins/order.ts:33`): contributions sort into `before` (band 0) -> numeric `order: n` (band 1, ascending) -> default/no-order (band 2) -> `after` (band 3); ties break by registration sequence. This applies to events, middlewares, hooks, observers, gateway interceptors, etc.

Requirements (`PluginRequirementInput`, `types.ts:90`) take three forms: a bare `` `plugin:${name}` `` string, `{ req: 'plugin:x', range?, optional? }` (semver range), or `{ capability: SharedKey, optional? }`. Read presence at runtime with `api.has('plugin:x')`.

Typed registry augmentation: declare-merge `interface SeyfertRegistry { plugins: [...] }` to globally type installed plugins; `RegisteredPlugins`, `RegisteredPluginExtension`, `RegisteredPluginContext`, `RegisteredPluginMiddlewares`, `RegisteredPluginShared` derive from it. (`src/client/plugins/types.ts:32`, `:286`)

## Code Examples (verified)

Installing plugins (matches the MDX):

```ts
import { Client, definePlugins } from 'seyfert';
import { cooldownPlugin } from './plugins/cooldown';

const client = new Client({
    // install one or more plugins; definePlugins keeps them typed
    plugins: definePlugins(cooldownPlugin),
});
```

`definePlugins` also accepts a plain array (both overloads exist):

```ts
plugins: definePlugins([pluginA, pluginB]); // equivalent to definePlugins(pluginA, pluginB)
```

Minimal authored plugin (verified shape against `createPlugin` / `SeyfertPlugin`):

```ts
import { createPlugin } from 'seyfert';

export const cooldownPlugin = createPlugin({
    name: 'cooldown',
    register(api) {
        api.hooks.on('plugins:ready', client => {
            client.logger.info('cooldown plugin ready');
        });
    },
    async setup(client) {
        // runs on client start()
    },
    async teardown(client) {
        // runs on client close()
    },
});
```

### Recipe: client + ctx extension, globally typed

A plugin contributes app-wide state (`client.*`) and per-interaction helpers (`ctx.*`) as factory functions. Augment `SeyfertRegistry.plugins` once and the types flow into every command/component handler.

```ts
import { Client, createPlugin, definePlugins } from 'seyfert';

class EconomyService {
    async connect() {/* ... */}
    async close() {/* ... */}
    balanceOf(userId: string) { return 0; }
}

const economyPlugin = createPlugin({
    name: 'economy',
    // factory receives the (typed) client; value lands on client.economy
    client: { economy: () => new EconomyService() },
    // factory receives (interaction, client); value lands on ctx.economy
    ctx: { myBalance: (interaction, client) => client.economy.balanceOf(interaction.member?.id ?? '') },
    setup: client => client.economy.connect(),     // open on start()
    teardown: client => client.economy.close(),    // close on close()
});

const plugins = definePlugins(economyPlugin);
const client = new Client({ plugins });

// Make client.economy / ctx.myBalance globally inferred:
declare module 'seyfert' {
    interface SeyfertRegistry {
        plugins: typeof plugins;
    }
}
// now inside any command: ctx.client.economy and ctx.myBalance are typed.
```

### Recipe: shared state + a capability requirement

`createSharedKey` makes a typed handle for runtime state another plugin can read without a `client.*` property. A consumer plugin declares it `requires` that capability.

```ts
import { createPlugin, createSharedKey } from 'seyfert';
import type { Pool } from 'pg';

// One key, shared by reference between provider and consumer.
export const dbKey = createSharedKey<Pool>()('db');

export const databasePlugin = createPlugin({
    name: 'database',
    register(api) {
        // factory runs lazily; client.shared resolves it on first read
        api.shared.set(dbKey, () => createPool());
    },
    teardown: async client => {
        await client.shared.get(dbKey)?.end();
    },
});

export const auditPlugin = createPlugin({
    name: 'audit',
    // refuse to load unless the db capability is present
    requires: [{ capability: dbKey }],
    register(api) {
        api.hooks.on('plugins:ready', client => {
            // unwrap throws if missing; get(...) returns T | undefined
            const pool = client.shared.unwrap(dbKey);
            void pool;
        });
    },
});
```

### Recipe: middleware + REST observer + ordering

Plugins register named middlewares (optionally global) and stack isolated REST observers. Use `PluginOrder` / numeric `order` to position contributions.

```ts
import { createMiddleware, createPlugin, PluginOrder } from 'seyfert';

const balanceGuard = createMiddleware<void>(({ next, stop }) => {
    // stop() with no args = skip silently; stop('reason') = deny (v5: no pass())
    return next();
});

export const metricsPlugin = createPlugin({
    name: 'metrics',
    // declare the middleware map so api.middlewares.add(name) is name-checked
    middlewares: { balanceGuard },
    register(api) {
        // run this middleware on every command, before others
        api.middlewares.add('balanceGuard', balanceGuard, { global: true, order: PluginOrder.Before });

        // observers stack and fail in isolation; returns a disposer
        api.rest.observe({
            onRequest({ method, url }) {/* about to fire */},
            onSuccess({ method, url, response }) {
                metrics.increment('rest.success', { method, url, status: response.status });
            },
            onFail({ method, url, error }) {/* failed */},
            onRatelimit({ url }) {/* slow down */},
        });
    },
});
```

### Recipe: configurable plugin via createPluginFactory

`createPluginFactory` turns a plugin into a configurable `(options?) => Plugin`. `defaults` merge with caller options; `validate` throws are wrapped as plugin errors.

```ts
import { createPlugin, createPluginFactory } from 'seyfert';

interface CooldownOptions { defaultSeconds: number }

export const cooldown = createPluginFactory<CooldownOptions, ReturnType<typeof build>>({
    defaults: { defaultSeconds: 3 },
    validate(opts) {
        if (opts.defaultSeconds < 0) throw new Error('defaultSeconds must be >= 0');
    },
    factory: opts => build(opts),
});

function build(opts: CooldownOptions) {
    return createPlugin({
        name: 'cooldown',
        meta: opts,
        register(api) {
            api.commands.observe({
                onBeforeMiddlewares(ctx) {/* enforce opts.defaultSeconds */},
            });
        },
    });
}

// install: plugins: definePlugins(cooldown({ defaultSeconds: 10 }))
```

### Recipe: gateway intents + dispatch interceptor

A plugin can require extra intents and observe/veto inbound gateway packets.

```ts
import { createPlugin } from 'seyfert';

export const voicePlugin = createPlugin({
    name: 'voice',
    register(api) {
        api.gateway.addIntents('GuildVoiceStates'); // string | GatewayIntentBits | number

        // return the (possibly mutated) packet, call next(), or return null to veto
        api.gateway.onDispatch(async (packet, next) => {
            if (packet.t === 'VOICE_STATE_UPDATE') {/* observe */}
            return next();
        });
    },
});
```

## Common patterns / gotchas

- `createPlugin` / `definePlugins` / `createContextScope` are runtime no-ops (return input as-is); their entire value is type inference. They do NOT validate anything.
- Two distinct "hook" concepts: `setup`/`teardown` are lifecycle *methods* on the plugin object; `api.hooks.on('plugins:ready', ...)` subscribes to a fixed set of framework lifecycle *events*. Passing an unknown hook name will not type-check.
- `events:afterLoad` gives you `[client, dir]`, NOT a metadata object — only `commands:afterLoad` / `components:afterLoad` get `[PluginLoadedMetadata]`.
- `client`/`ctx` entries are factories, not values: `client: { x: (client) => value }`, `ctx: { y: (interaction, client) => value }`. Returning the value directly is wrong.
- v5 middleware: callback arg is `{ context, next, stop }` — `pass()` is gone. `stop()`/`stop(null)` skips silently; `stop('reason')` denies (surfaces to `onMiddlewaresError`).
- `api.middlewares.add(name, mw)` is name-checked against the plugin's own `middlewares` map; `{ global: true }` makes it run on every command.
- Shared state: `api.shared.set(key, factory)` claims a key (throws on conflict unless `{ override: true }`); read with `client.shared.get(key)` (`T | undefined`) or `client.shared.unwrap(key)` (throws if missing). Keys must be shared by reference (`createSharedKey`) between provider and consumer.
- Augment `SeyfertRegistry.plugins` (e.g. `plugins: typeof myDefinePluginsTuple`) to get `client.*` / `ctx.*` / middleware names typed globally. A bare array works at runtime but loses inference without this augmentation.
- If `setup` throws, Seyfert auto-rolls-back; when cleanup also fails you get a `SeyfertPluginAggregateError` — inspect `.errors` (array) and `.cause`.
- Reserved keys on the `createPlugin` input (name, instanceId, imports, requires, client, ctx, middlewares, globalMiddlewares, options, register, setup, teardown, meta) cannot double as custom extra properties; everything else passes through as an extra key (e.g. `version`).

## Doc vs Source Corrections

- Draft note's header URL was `/docs/plugins` and only listed `definePlugins`; root also exports `createPlugin`, `createPluginFactory`, `createSharedKey`, `createContextScope`, `PluginOrder`, and the plugin error classes. URL corrected to `/docs/plugins/index`. (Re-confirmed this release.)
- MDX prose ("setup on `start()`, teardown on `close()`, rollback on failure, attributed diagnostics") matches `setupClientPlugins`/`teardownClientPlugins` (`plugins.ts:656`, `:717`). No conflict.
- MDX capability buckets (client/ctx helpers, handlers, shared state, cache/langs, gateway/REST, lifecycle) map 1:1 to real `SeyfertPluginApi` members. No invented capability.
- Confirmed `events:afterLoad` payload is `[client, dir]` (not metadata) — added as a gotcha. No other drift found.

## Agent Guidance

- This is the conceptual landing page. To author a plugin route to `plugins/building/creating-plugins` (full `register` API); for ready-made packages route to `plugins/official`.
- Prefer `api.shared.*` + `createSharedKey` for cross-plugin state; prefer `client.*` (the `client` map) only for values the bot author should call directly.
- Do not invent API members: the authoring surface is exactly the `SeyfertPluginApi` shape listed above.
