# Creating Plugins

Original source URL: https://seyfert-web-git-seyfert-v5-tiramisulabs.vercel.app/docs/plugins/building/creating-plugins
Coverage reference: plugins.md
Verification status: Source-verified (src/client/plugins.ts + src/client/plugins/{types,api,shared,order}.ts, the authoritative Seyfert source)

## Page Summary

Define a Seyfert v5 plugin with `createPlugin(...)`, an identity function at runtime (`return plugin`)
typed via overloads to preserve `const` inference. Plugins contribute commands, components, modals,
events, middlewares, cache resources, lang overlays and shared state through a `register(api)` callback,
expose app-wide helpers via `client` factories, and per-interaction helpers via `ctx` factories. They run
a real lifecycle (`register` -> `client`/`ctx` install -> `setup` on `client.start()` -> `teardown` on
`client.close()`). Install plugins with `definePlugins(...)` to build one canonical tuple shared by runtime
and types, then register that tuple's types through `SeyfertRegistry.plugins`. `createPlugin`,
`definePlugins`, `createPluginFactory`, `createSharedKey` and the `PluginOrder` enum are all root exports
from `seyfert` (no deep import).

## Key APIs (verified)

All from `src/client/plugins.ts` and re-exported via `src/client/index.ts` (`export * from './plugins'`) -> root `seyfert`.

- `createPlugin(config)` — identity at runtime (plugins.ts:261-265), two overloads (with/without `meta`). Config (`CreatePluginInputBase`, plugins.ts:165-184; mirrored by `SeyfertPlugin`, types.ts:492-512):
  - `name: string` (required, `const` inferred), `instanceId?: string`, `version?: string`, `meta?` (unknown; reserved key)
  - `imports?: readonly AnySeyfertPlugin[]` — other plugins whose extensions/ctx/middlewares flow into this one's typings
  - `requires?: readonly PluginRequirementInput[]` — `` `plugin:${string}` `` string, `{ req, range?, optional? }`, or `{ capability: SharedKey, optional? }` (types.ts:90-93)
  - `client?: PluginClientMap` — `{ [key]: (client) => value }`; SYNC factories, run before `setup`
  - `ctx?: PluginContextMap` — `{ [key]: (interaction, client) => value }` (factory gets a SECOND `client` arg; types.ts:255)
  - `middlewares?: PluginMiddlewareMap` (`Record<string, AnyMiddlewareContext>`), `globalMiddlewares?: readonly (keyof M & string)[]`
  - `options?(current)`, `register?(api)`, `setup?(client, api?)`, `teardown?(client, api?)`
- `definePlugins(...)` (plugins.ts:283-289) — overloads accept rest args, a single array, or nothing; returns the tuple unchanged.
- `createPluginFactory({ defaults, validate?, factory })` (plugins.ts:267-281) — returns `(options?: Partial<O>) => P`; merges `{...defaults, ...options}`, runs `validate`, then `factory`; errors wrapped in a Seyfert plugin error.
- `createSharedKey` (shared.ts:22-32) — `createSharedKey('name')` -> `SharedKey<unknown,'name'>`; `createSharedKey<T>()('name')` -> typed `SharedKey<T,'name'>`; or augment `RegisteredPluginShared` for `createSharedKey<'name'>('name')` typing. Keys are frozen.
- `PluginOrder` enum (types.ts:69-72): `Before='before'`, `After='after'`. `PluginOrderOpt = PluginOrder.Before | PluginOrder.After | number` (types.ts:74).
- `register(api)` receives `SeyfertPluginApi` (types.ts:373-477) — far broader than docs show:
  - `commands`/`components`/`modals`: `add(...)`, `remove(...)`, `defaults(hooks, opts?)`; `commands` also has `observe(observer, opts?)`
  - `events`: `on(name, handler, {once?, order?})`, `once(name, handler)`, `onAny(handler, {order?})`, `onError(handler, {order?})` — all return a `PluginDisposer`
  - `hooks.on(name, handler, {order?})`; `middlewares.add(name, mw, opts?)` / `remove(...)`
  - `rest.observe(observer, {order?})`; `autocomplete.wrap(wrapper, {order?})`
  - `gateway`: `addIntents(...)`, `wrapSendPayload(wrapper, {order?})`, `onDispatch(interceptor, {order?})`
  - `cache.resource(name, ctor, opts?)`; `shared`: `set(key|name, factory, opts?)`, `remove(...)`, `has(...)`
  - `langs.contribute(locale, values, { prefix })`; `handlers`: `construct(...)`, `transform(...)`
  - `diagnostics.warn(message, opts?)`; `options.set(fragment)`; `reload(): Promise<void>`; `has(req)`
- Lifecycle hook names (`api.hooks.on`) — `SeyfertPluginHooks` (types.ts:200-210). Note payload shapes:
  - `plugins:ready` `[client]`, `plugins:setupComplete` `[client]`
  - `commands:beforeLoad` `[client, dir?]`, `commands:afterLoad` `[metadata]` (PluginLoadedMetadata<'commands'>)
  - `components:beforeLoad` `[client, dir?]`, `components:afterLoad` `[metadata]`
  - `events:beforeLoad` `[client, dir?]`, `events:afterLoad` `[client, dir?]` (BOTH are `[client, dir]`, unlike commands/components afterLoad)
  - `client:close` `[client]`
- Registration types: `SeyfertRegistry` (augment with `plugins: typeof plugins`), `RegisteredPlugins`, `PluginUsingClient`, `RegisteredPluginShared` (augment for typed shared names).

### Ordering bands (order.ts:20-38)

Contributions sort into bands: `PluginOrder.Before` (0) -> numeric `order` (1, ascending value) ->
default/no-order (2) -> `PluginOrder.After` (3). Within a band, insertion sequence (registration order)
breaks ties. Use `order` on `events.on`, `hooks.on`, `middlewares.add`, `rest.observe`,
`autocomplete.wrap`, `gateway.*`, `*.defaults` etc. to position a contribution relative to other plugins.

## Code Examples (verified)

### Minimal plugin + install

```ts title="src/ping-plugin.ts"
import { createPlugin } from 'seyfert';
import PingCommand from './commands/ping';

export const pingPlugin = createPlugin({
    name: 'ping-plugin',
    register(api) {
        api.commands.add(PingCommand); // contribute through the registration api
    },
});
```

```ts title="src/index.ts"
import { Client, definePlugins } from 'seyfert';
import { pingPlugin } from './ping-plugin';

const plugins = definePlugins(pingPlugin); // one canonical tuple

declare module 'seyfert' {
    interface SeyfertRegistry { plugins: typeof plugins }
}

const client = new Client({ plugins });
```

```ts
// definePlugins shapes (all return the tuple unchanged)
const a = definePlugins(loggerPlugin(), economyPlugin()); // rest args
const b = definePlugins([loggerPlugin(), economyPlugin()]); // existing array
const c = definePlugins();                                  // empty

// stateful plugins: build once, reuse the same instance
const storage = storagePlugin();
const economy = economyPlugin({ storage });
const plugins = definePlugins(storage, economy);
```

### Client + ctx helpers

```ts title="src/logger-plugin.ts"
import { createPlugin } from 'seyfert';

export function loggerPlugin(options: { name?: string } = {}) {
    const root = new SimpleLogger(options.name);
    return createPlugin({
        name: 'example-logger',
        client: {
            exampleLogger: () => root,                 // -> client.exampleLogger (sync, before setup)
        },
        ctx: {
            // factory also receives client as a 2nd arg if needed
            logger: interaction => root.child({ interactionId: interaction.id }),
        },
        setup(client) {
            client.exampleLogger.info('logger plugin ready'); // async startup, on client.start()
        },
        teardown(client) {
            return client.exampleLogger.flush();              // cleanup, on client.close()
        },
    });
}
```

```ts title="src/commands/profile.ts"
import { Command, type CommandContext } from 'seyfert';

export default class ProfileCommand extends Command {
    async run(ctx: CommandContext) {
        ctx.logger.info('profile opened');                  // per-interaction ctx helper
        ctx.client.exampleLogger.info('root logger ready'); // app-wide client helper
    }
}
```

### Factory helper (createPluginFactory)

```ts
import { createPlugin, createPluginFactory } from 'seyfert';

export const economyPlugin = createPluginFactory({
    defaults: { startingBalance: 0 },
    validate: o => { if (o.startingBalance < 0) throw new Error('negative balance'); },
    factory: o => createPlugin({
        name: 'economy',
        client: { economy: () => new EconomyService(o.startingBalance) },
        setup: client => client.economy.connect(),
        teardown: client => client.economy.close(),
    }),
});
// economyPlugin() uses defaults; economyPlugin({ startingBalance: 100 }) overrides.
```

### NEW: typed middlewares + global middleware contribution

```ts title="src/balance-guard-plugin.ts"
import { createPlugin, createMiddleware } from 'seyfert';

// v5: a middleware that skips silently uses stop() with no args; stop('reason') denies.
const balanceGuard = createMiddleware<void>(({ context, next, stop }) => {
    if (!context.guildId) return stop();          // skip silently (was pass() in v4)
    if (isBroke(context.author.id)) return stop('Insufficient balance');
    return next();
});

export const economyPlugin = createPlugin({
    name: 'economy',
    // typed map: keys become the union accepted by api.middlewares.add / globalMiddlewares
    middlewares: { balanceGuard },
    // mark it global so every command runs it (equivalent to api.middlewares.add(..,{global:true}))
    globalMiddlewares: ['balanceGuard'],
    register(api) {
        // or register imperatively with ordering control:
        // api.middlewares.add('balanceGuard', balanceGuard, { global: true, order: PluginOrder.Before });
    },
});
```

### NEW: shared state + a consumer that requires the capability

A plugin can publish a runtime object for OTHER plugins (not on `client.*`) via `api.shared.set` keyed by
a `SharedKey`. Consumers declare `requires: [{ capability: key }]` and read it with
`client.shared.get(key)` / `unwrap(key)` (throws if absent).

```ts title="src/shared-keys.ts"
import { createSharedKey } from 'seyfert';

export interface RateLimiter { take(id: string): boolean }
// typed key: createSharedKey<T>()(name)
export const rateLimiterKey = createSharedKey<RateLimiter>()('rateLimiter');
```

```ts title="src/rate-limit-plugin.ts"
import { createPlugin } from 'seyfert';
import { rateLimiterKey } from './shared-keys';

export const rateLimitPlugin = createPlugin({
    name: 'rate-limit',
    register(api) {
        api.shared.set(rateLimiterKey, () => new TokenBucket(), {
            dispose: limiter => limiter.stop(), // runs on teardown
        });
    },
});
```

```ts title="src/feature-plugin.ts"
import { createPlugin } from 'seyfert';
import { rateLimiterKey } from './shared-keys';

export const featurePlugin = createPlugin({
    name: 'feature',
    // hard dependency — startup fails if rate-limit plugin is missing
    requires: ['plugin:rate-limit', { capability: rateLimiterKey }],
    setup(client) {
        const limiter = client.shared.unwrap(rateLimiterKey); // throws if not provided
        if (!limiter.take('boot')) client.logger.warn('rate limited at boot');
    },
});
```

### NEW: events, hooks, REST observer + gateway intents in one plugin

```ts title="src/observability-plugin.ts"
import { createPlugin, PluginOrder } from 'seyfert';

export const observabilityPlugin = createPlugin({
    name: 'observability',
    register(api) {
        // ensure the plugin's listeners need the right intents
        api.gateway.addIntents('Guilds', 'GuildMembers');

        // typed client event listener (returns a disposer)
        api.events.on('guildCreate', guild => {
            api.diagnostics.warn(`joined ${guild.name}`, { code: 'joined-guild' });
        });

        // lifecycle hook: commands:afterLoad hands you metadata, not (client, dir)
        api.hooks.on('commands:afterLoad', metadata => {
            console.info(`loaded ${metadata.total} commands`);
        });
        // events:afterLoad is [client, dir] (differs from commands/components afterLoad)
        api.hooks.on('events:afterLoad', (client, dir) => {
            client.logger.info(`events loaded from ${dir}`);
        });

        // stackable REST observer (replaces overwriting client.rest.onSuccessRequest)
        const dispose = api.rest.observe({
            onSuccess({ method, url, response }) {
                metrics.increment('rest.success', { method, url, status: response.status });
            },
            onRatelimit({ url }) { metrics.increment('rest.ratelimit', { url }); },
        }, { order: PluginOrder.Before });
        // dispose() to unsubscribe early; otherwise cleaned up automatically.
    },
});
```

### NEW: imports — composing plugins so types flow through

`imports` makes another plugin's `client`/`ctx`/middleware types visible inside this plugin's
`register`/`setup`. Put the imported plugin in the `definePlugins` tuple too so app code sees it.

```ts
import { createPlugin } from 'seyfert';
import { loggerPlugin } from './logger-plugin';

const logger = loggerPlugin();

export const auditPlugin = createPlugin({
    name: 'audit',
    imports: [logger],            // client.exampleLogger is now typed inside setup below
    setup(client) {
        client.exampleLogger.info('audit ready');
    },
});

// app: include BOTH so global types resolve
// const plugins = definePlugins(logger, auditPlugin);
```

## Common patterns / gotchas

- v4 -> v5 middleware: `pass()` is GONE. `stop()` (no arg) skips silently; `stop('reason')` denies and
  routes to `onMiddlewaresError`. Middleware arg object is `{ context, next, stop }` — no `pass`.
- `client`/`ctx` factories must be SYNCHRONOUS (they run before/around context creation). Do async startup
  in `setup`, async cleanup in `teardown`. Never add a typed client key by assigning inside `setup` —
  declare it under `client` so it is typed and installed at the right time.
- `teardown`'s `api` is the narrower `SeyfertPluginTeardownApi` — only `has`, `diagnostics`, and
  `shared.has` (types.ts:479-481). Trying to mutate contributions (e.g. `api.commands.add`,
  `api.shared.set`) during teardown throws a conflict error (api.ts:98-107). `setup`'s `api` is the full
  `SeyfertPluginApi`.
- A plugin must appear in the `definePlugins(...)` tuple registered via `SeyfertRegistry.plugins` for its
  global `client.*` / `ctx.*` / middleware-name types to be visible in app code — even if it runs at
  runtime, its types stay invisible otherwise.
- For stateful factory plugins, build the instance ONCE and reuse it (`const s = storagePlugin()`); calling
  the factory twice creates two unrelated plugin instances.
- `cache.resource(name, ...)` names are validated against a reserved set (e.g. `guilds`, `members`,
  `roles`, `channels`, `flush`, `bulkGet`, `constructor`, `__proto__` ...; api.ts:40-79) — pick a unique
  resource name.
- Ordering: use `order: PluginOrder.Before | PluginOrder.After | <number>` to slot a contribution; bands
  are Before < numeric(asc) < default < After, ties broken by registration order (order.ts).
- Setup failures roll back: if one plugin's `setup` throws, already-completed plugins are torn down in
  reverse and shared values disposed (plugins.ts:699-715). Aggregate failures surface as
  `SeyfertPluginAggregateError`.

## Doc vs Source Corrections

- Docs `ctx` factory shows `(interaction) => ...`; src `PluginContextMap` (types.ts:255) passes
  `(interaction, client)` — a second `client` arg is available.
- Docs only show `client`/`ctx`/`setup`/`teardown`/`register.commands.add`; src exposes much more
  (`imports`, `requires`, `middlewares`, `globalMiddlewares`, `options`, `instanceId`, `version`, `meta`)
  and a full `SeyfertPluginApi` (events, hooks, gateway, cache, shared, langs, handlers, rest,
  autocomplete, diagnostics, options, reload). Not wrong — incomplete.
- Docs omit `createPluginFactory` and `createSharedKey` — both are real root exports.
- `teardown`'s `api` is the narrower `SeyfertPluginTeardownApi` (only `has`, `diagnostics`, `shared.has`);
  docs say "both receive the plugin api as an optional second argument" without noting the teardown api is
  restricted and mutation throws. Source wins.
- `events:afterLoad` payload is `[client, dir]`, while `commands:afterLoad` / `components:afterLoad` are
  `[metadata]` — easy to conflate; verified at types.ts:204-208.
- `package.json` `peerDependencies.seyfert: ">=5.0.0-0"` is doc-authored; pin to the actual v5 range your
  target project uses (a published version cannot be verified from source).

## Source Anchors

- src/client/plugins.ts (createPlugin:240-265, definePlugins:283-289, createPluginFactory:267-281,
  resolveClientPlugins, setup/teardown lifecycle:656-783, hook/observer runners)
- src/client/plugins/types.ts (SeyfertPlugin:492, SeyfertPluginApi:373, SeyfertRegistry:32,
  SeyfertPluginHooks:200, PluginOrder:69, requirement/shared types:36-93, SeyfertPluginTeardownApi:479)
- src/client/plugins/api.ts (createPluginApi — full register-time surface, teardown guard:98-107, reserved cache names:40-79)
- src/client/plugins/shared.ts (createSharedKey:22-32, addPluginShared/override+dispose)
- src/client/plugins/order.ts (ordering bands:20-38)
- src/client/index.ts (`export * from './plugins'`), src/index.ts (`export * from './client'`)

## Agent Guidance

- Use `createPlugin` for the plugin object and ALWAYS install via `definePlugins(...)` + augment
  `SeyfertRegistry.plugins` so `client.*` / `ctx.*` helpers and middleware names are typed in app code.
- Keep `client`/`ctx` factories synchronous; put async startup in `setup`, cleanup in `teardown`. Don't add
  typed client keys by assigning inside `setup` — declare them in `client`.
- For stateful plugins, create the instance once and pass it in.
- To share a runtime object with OTHER plugins (not on `client.*`), use `api.shared.set` + `createSharedKey`,
  and declare `requires: [{ capability: key }]` for consumers (see services-and-requirements page).
- Anything not in the `definePlugins` tuple won't expose global types to app code even if it runs.
- Third-party packages: keep `seyfert` as a peerDependency, export factories from package root, and use root
  `seyfert` imports in public examples (avoid `seyfert/lib/...`).
