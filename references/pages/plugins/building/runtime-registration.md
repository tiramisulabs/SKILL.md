# Runtime Registration

Original source URL: https://seyfert-web-git-seyfert-v5-tiramisulabs.vercel.app/docs/plugins/building/runtime-registration

Coverage reference: plugins.md

Verification status: Source-verified (the authoritative Seyfert source)

## Page Summary

A plugin's `register(api)` is a synchronous hook that declares what the plugin contributes to a Seyfert client: commands, components, modals, named/global middlewares, event listeners, lifecycle hooks, gateway intents, cache resources, shared values, langs, REST/gateway observers, and partial client-option fragments. `register` runs during option resolution (`resolveClientPlugins`, before the client/event runner exists), so it MUST NOT return a promise and must NOT open connections or emit events — defer side effects to `setup` and undo them in `teardown`. The doc shows a representative subset of `api`; the real `SeyfertPluginApi` surface in src is much larger.

Lifecycle ordering: plugin `options(current)` callback → `register(api)` (both during `resolveClientPlugins`) → `setup(client, api)` (after client bind, can be async) → `teardown(client, api)` (reverse order, read-only api).

## Key APIs (verified)

All root imports come from `'seyfert'` (re-exported via `src/client/index.ts` -> `src/index.ts`).

- `createPlugin({ name, register?, setup?, teardown?, options?, client?, ctx?, middlewares?, globalMiddlewares?, imports?, requires?, instanceId?, meta?, ...extra })` — `src/client/plugins.ts:240`. Identity function with rich typing; returns the plugin object unchanged (arbitrary extra keys are preserved on the result).
- `definePlugins(...plugins | [plugins])` — `src/client/plugins.ts:283`. Tuple helper, accepts spread or a single array; returns the tuple for `new Client({ plugins })`.
- `createPluginFactory({ defaults, validate?, factory })` -> `(options?: Partial<O>) => plugin` — `src/client/plugins.ts:267`. Merges `defaults` with caller options, runs `validate`, calls `factory`; errors are wrapped as plugin errors.
- `createSharedKey(name)` / `createSharedKey<T>()(name)` — typed shared-value key, `src/client/plugins/shared.ts:22`. `PluginOrder` enum (`Before='before'`, `After='after'`) and `createContextScope` also exported from `'seyfert'`.
- `register?(api: SeyfertPluginApi): void` — return type is `void` (sync, enforced by the type). `setup?`/`teardown?` are `Awaitable<void>` and receive `(client, api?)`; teardown's api is the narrowed `SeyfertPluginTeardownApi`. `src/client/plugins/types.ts:509`.

`SeyfertPluginApi` namespaces (verified in `src/client/plugins/api.ts` + `types.ts:373`):
- `api.has(req)` — check a plugin requirement (`plugin:name`).
- `api.events.on(name, handler, { once?, order? })` / `.once(name, handler)` / `.onAny(handler, {order?})` / `.onError(handler, {order?})`. `name` is `ClientNameEvents | CustomEventsKeys | GatewayEvents`. All four return a `PluginDisposer` (idempotent unregister fn).
- `api.commands.add(...cmds, opts?)` / `.remove(...names)` / `.observe(observer, {order?})` / `.defaults(hooks, {suppressDefault?, order?})`. Command add opts: `{ override?: boolean; guilds?: readonly string[] }` (`guilds` emits a `command-guild-scope` diagnostic).
- `api.components.add(...)/.remove(...customIds)/.defaults(...)`; `api.modals.add(...)/.remove(...customIds)/.defaults(...)`. add opts: `{ override?: boolean }`.
- `api.middlewares.add(name, middleware, { global?, order?, override? })` / `.remove(...names)`. `global: true` also appends the name to a `globalMiddlewares` option fragment (`api.ts:363`).
- `api.hooks.on(name, handler, {order?})` — `name` is a `PluginHookName`: `plugins:ready`, `plugins:setupComplete`, `commands:beforeLoad`, `commands:afterLoad`, `components:beforeLoad`, `components:afterLoad`, `events:beforeLoad`, `events:afterLoad`, `client:close` (`types.ts:200`). Payloads: `*:afterLoad` for commands/components carry `[PluginLoadedMetadata]`; the `beforeLoad`/`events:afterLoad`/`plugins:*`/`client:close` hooks carry `[client, dir?]` or `[client]`.
- `api.options.set(fragment)` — contribute a partial `SeyfertPluginOptions` (= `Partial<Omit<BaseClientOptions,'plugins'>>`).
- `api.gateway.addIntents(...)/.wrapSendPayload(wrapper,{order?})/.onDispatch(interceptor,{order?})`; `api.cache.resource(name, ctor, opts?)`; `api.shared.set(key|name, factory, opts?)/.remove(...)/.has(...)`; `api.langs.contribute(locale, values, { prefix })` (opts is REQUIRED); `api.rest.observe(observer,{order?})`; `api.autocomplete.wrap(wrapper,{order?})`; `api.handlers.construct/transform`; `api.diagnostics.warn(msg, {code?,phase?,data?})`; `api.reload()` (Promise; only valid after the client exists — i.e. from `setup`/runtime, not `register`).
- Plugin-level `options(current)` callback (sibling of `register`, not on `api`) reads the already-composed options; `src/client/plugins/types.ts:508`.
- `PluginLoadedMetadata<TKind, TItem>` = `{ kind, total, items, plugin: { total, sources } }` (`types.ts:129`); `plugin.sources` is `Readonly<Record<pluginName, count>>`.
- Custom events `commandsLoaded` / `componentsLoaded` carry `PluginLoadedMetadata` (`src/events/event.ts:7`, emitted from `src/client/base.ts:418` alongside the equivalent `commands:afterLoad`/`components:afterLoad` hooks).

## Code Examples (verified)

Canonical plugin (commands/components/modals + global middleware + client extension + async setup):

```ts
import { createPlugin } from 'seyfert';
import HealthCommand from './commands/health';
import RefreshButton from './components/refresh-button';
import ProfileModal from './modals/profile-modal';
import { auditService } from './audit-service';
import { auditMiddleware } from './middlewares/audit';

export const auditPlugin = createPlugin({
    name: 'audit-plugin',
    client: {
        audit: () => auditService, // exposed as client.audit (typed)
    },
    register(api) {
        api.commands.add(HealthCommand);
        api.components.add(RefreshButton);
        api.modals.add(ProfileModal);
        api.middlewares.add('audit', auditMiddleware, { global: true });

        // commandsLoaded is a custom event carrying PluginLoadedMetadata
        api.events.on('commandsLoaded', metadata => {
            auditService.recordCommandsLoaded(metadata);
        });

        api.options.set({ allowedMentions: { parse: [] } });
    },
    // open connections HERE, not in register
    async setup(client) {
        await auditService.connect();
    },
    async teardown() {
        await auditService.close(); // cleanup that actually runs
    },
});
```

Loaded metadata via a typed plugin hook (equivalent to the `commandsLoaded` event, but `PluginHookName`-typed):

```ts
register(api) {
    api.hooks.on('commands:afterLoad', metadata => {
        metadata.kind;          // "commands"
        metadata.total;         // all loaded commands
        metadata.items;         // loaded command objects
        metadata.plugin.total;  // plugin-contributed commands
        metadata.plugin.sources;// Record<pluginName, count>
    });
}
```

Typed middleware contribution so app code stays cast-free:

```ts
import { createPlugin, type CommandContext, type MiddlewareContext, type SeyfertPlugin } from 'seyfert';

type AuthMiddleware = MiddlewareContext<{ userId: string }, CommandContext>;
const auth: AuthMiddleware = ({ next }) => next({ userId: '1' });

// SeyfertPlugin<E, C, I, M> — middlewares map is the 4th type arg
export const authPlugin: SeyfertPlugin<{}, {}, readonly [], { auth: AuthMiddleware }> = createPlugin({
    name: 'auth',
    register(api) {
        api.middlewares.add('auth', auth); // name is typed against M
    },
});
```

Once the app lists `authPlugin` in the `SeyfertRegistry` plugin tuple, `Middlewares(['auth'])` and `ctx.metadata.auth` are typed in app commands.

Option fragments that compose (plugin hook AND app hook both run — they are merged, not overwritten):

```ts
register(api) {
    api.options.set({
        commands: { defaults: { onRunError(ctx, error) { ctx.client.logger.error(error); } } },
    });
}
```

Plugin-level `options(current)` (only set a default when the app didn't):

```ts
export const plugin = createPlugin({
    name: 'mentions',
    options(current) {
        return { allowedMentions: current.allowedMentions ?? { parse: [] } };
    },
});
```

## Recipes (new, source-checked)

### Shared services + cross-plugin dependency (`requires`)

A provider plugin claims a typed shared value; a consumer plugin declares a `requires` on it and reads it at runtime. Shared values are reference-counted and disposed on teardown.

```ts
import { createPlugin, createSharedKey } from 'seyfert';

class Database { async connect() {} async close() {} query(_: string) { return []; } }

// typed key: createSharedKey<T>() returns a (name) => SharedKey<T, name> factory
export const dbKey = createSharedKey<Database>()('db');

export const dbPlugin = createPlugin({
    name: 'db',
    version: '1.0.0',
    register(api) {
        // factory runs lazily once the client exists; dispose runs on teardown
        api.shared.set(dbKey, () => new Database(), { dispose: db => db.close() });
    },
    setup: client => client.shared.unwrap(dbKey).connect(),
});

export const economyPlugin = createPlugin({
    name: 'economy',
    // hard dependency — resolution fails if db isn't installed
    requires: [{ req: 'plugin:db', range: '^1.0.0' }],
    setup(client) {
        const db = client.shared.unwrap(dbKey); // throws if missing; .get(...) returns T | undefined
        client.logger.info('economy using db', db.query('SELECT 1'));
    },
});
```

`requires` accepts `'plugin:name'` shorthand, `{ req, range?, optional? }`, or `{ capability: SharedKey, optional? }`. Missing required deps throw at resolve; missing `optional` ones emit a `missing-optional-requirement` diagnostic.

### Instrumentation: REST + gateway observers + diagnostics

Stackable observers (multiple plugins can instrument the same surface without clobbering). `onDispatch`/`wrapSendPayload` can return `null` to veto a packet (logged as a diagnostic).

```ts
import { createPlugin } from 'seyfert';

export const metricsPlugin = createPlugin({
    name: 'metrics',
    register(api) {
        // REST: four lazily-built, readonly payloads
        api.rest.observe({
            onSuccess({ method, url, response }) {
                metrics.inc('rest.ok', { method, url, status: response.status });
            },
            onFail({ method, url }) { metrics.inc('rest.fail', { method, url }); },
            onRatelimit({ url }) { metrics.inc('rest.ratelimit', { url }); },
        });

        // Gateway: inspect inbound dispatches; call next() to continue, return null to drop
        api.gateway.onDispatch((packet, next) => {
            metrics.inc('gateway.dispatch', { t: packet.t });
            return next();
        });

        api.diagnostics.warn('metrics enabled', { code: 'info', phase: 'register' });
    },
});
```

### Custom cache resource + intents

```ts
import { createPlugin, GuildBasedResource } from 'seyfert';

class ReactionRolesResource extends GuildBasedResource {
    namespace = 'reaction_roles';
}

export const reactionRolesPlugin = createPlugin({
    name: 'reaction-roles',
    register(api) {
        api.cache.resource('reactionRoles', ReactionRolesResource, {
            intents: ['GuildMessageReactions'],
            onPacket(event, cache) { /* feed the resource from raw gateway packets */ },
        });
        // ensure the gateway sends the data this plugin needs
        api.gateway.addIntents('GuildMessageReactions');
    },
});
```

### Context extension + langs overlay

`ctx` adds typed getters to every command/component context; `api.langs.contribute` overlays locale strings under a prefix.

```ts
import { createPlugin } from 'seyfert';

export const i18nKitPlugin = createPlugin({
    name: 'i18n-kit',
    ctx: {
        // available as ctx.now in every command/component context
        now: () => Date.now(),
    },
    register(api) {
        api.langs.contribute('en-US', { greeting: 'Hello!' }, { prefix: 'kit' });
        api.langs.contribute('es-ES', { greeting: '¡Hola!' }, { prefix: 'kit' });
    },
});
```

### Configurable plugin via `createPluginFactory`

```ts
import { createPluginFactory } from 'seyfert';

export const ratelimit = createPluginFactory<{ perSecond: number }, ReturnType<typeof build>>({
    defaults: { perSecond: 5 },
    validate: o => { if (o.perSecond <= 0) throw new Error('perSecond must be > 0'); },
    factory: build,
});

function build(opts: { perSecond: number }) {
    return createPlugin({
        name: 'ratelimit',
        meta: opts,
        register(api) { api.middlewares.add('ratelimit', makeLimiter(opts.perSecond), { global: true }); },
    });
}

// usage: new Client({ plugins: definePlugins(ratelimit({ perSecond: 10 })) })
```

### Ordering contributions

```ts
import { createPlugin, PluginOrder } from 'seyfert';

register(api) {
    api.middlewares.add('first', mwA, { global: true, order: PluginOrder.Before });
    api.middlewares.add('mid',   mwB, { global: true, order: 10 });   // numeric band sits between Before and default
    api.middlewares.add('last',  mwC, { global: true, order: PluginOrder.After });
}
```

Bands run: `Before` (0) < numeric (1, lower number first) < default/unset (2) < `After` (3); ties broken by registration sequence (`src/client/plugins/order.ts`).

## Common patterns / gotchas

- `register` is sync — its type is `void`. Anything async (DB/socket connect, fetch, `client.*` runtime) belongs in `setup`; undo it in `teardown`. Returning a promise from `register` is a type error and would not be awaited anyway.
- `register` runs BEFORE the event runner and client are bound: do NOT call `api.reload()` (throws), do NOT emit events, do NOT read `client.*`. The `api` has no `emit` — emit via `client.events.emit(...)` from `setup`/listeners/runtime callbacks. Gateway event names cannot be emitted from plugins.
- Teardown gets a narrowed `SeyfertPluginTeardownApi` exposing only `has`, `diagnostics`, and `shared.has`. Any mutator `api.*` call (add/remove/observe/etc.) throws during teardown.
- Setup failure auto-rolls-back: event listeners, shared values, and dynamic contributions for the failed plugin are cleaned up, then already-completed plugins are torn down in reverse. Failures aggregate into `SeyfertPluginAggregateError`.
- Conflicts surface at install time, attributed to the offending plugin: duplicate command names, duplicate static component/modal custom IDs, and double-claimed shared names throw `SeyfertPluginError` unless you pass `{ override: true }`. Subcommands cannot be added as top-level plugin commands.
- Plugin commands/components/modals load AFTER disk ones. A command referencing an unregistered middleware name warns and continues.
- For public packages, type the middleware map as the 4th `SeyfertPlugin` type arg AND list the plugin in the `SeyfertRegistry` plugin tuple so app `Middlewares([...])`/`ctx.metadata` resolve without casts.
- Option fragments compose: command/component/modal `defaults` hooks all run (use `suppressDefault` to drop the framework default); plain values still follow normal option-merge precedence (user options win). Use `options(current)` when your fragment must read options composed by earlier plugins.

## Doc vs Source Corrections

- Doc presents `api` as roughly `{ events, commands, components, modals, middlewares, options }`. Source `SeyfertPluginApi` (`types.ts:373`) additionally exposes `has`, `hooks`, `handlers`, `gateway`, `cache`, `shared`, `langs`, `rest`, `autocomplete`, `diagnostics`, `reload`, plus `commands.remove/observe/defaults`, `components/modals.remove/defaults`, `middlewares.remove`, `events.onError`. Treat the doc list as a subset.
- Doc shows `api.middlewares.add(name, mw, { global })`. Source opts are `{ global?, order?, override? }` (`api.ts:349`).
- Doc treats `commandsLoaded`/`componentsLoaded` purely as events. Source confirms they are `CustomEvents` (`src/events/event.ts:7`) AND there are equivalent `PluginHookName` hooks `commands:afterLoad`/`components:afterLoad` carrying the same `PluginLoadedMetadata` via `api.hooks.on` (`types.ts:204-206`). Both fire from `reloadPluginContributions` (`base.ts:415-423`).
- Doc command-add example omits per-add options; source supports `api.commands.add(...cmds, { override?, guilds? })` (`api.ts:163`, `types.ts:60`). `guilds` emits a `command-guild-scope` diagnostic.
- Doc's metadata fields are correct; `plugin.sources` is `Readonly<Record<string, number>>` keyed by plugin name (`types.ts:135`).
- "Do not return a promise from register" is enforced by the type (`register?(api): void`) and by `register` running inside synchronous `resolveClientPlugins` (`plugins.ts:357`); there is no `await`. Confirmed accurate.
- `api.langs.contribute(locale, values, opts)` — `opts` is REQUIRED (`{ prefix }`), not optional (`types.ts:465`).

## Agent Guidance

- Prefer typed hooks (`api.hooks.on('commands:afterLoad', ...)`) for loaded metadata when you want `PluginHookName` autocompletion; the `commandsLoaded`/`componentsLoaded` events deliver the identical payload on the client event bus.
