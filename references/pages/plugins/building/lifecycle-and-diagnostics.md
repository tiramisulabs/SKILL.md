# Lifecycle and Diagnostics

Original source URL: https://seyfert-web-git-seyfert-v5-tiramisulabs.vercel.app/docs/plugins/building/lifecycle-and-diagnostics

Coverage reference: plugins.md

Verification status: Source-verified (the authoritative Seyfert source)

## Page Summary

A Seyfert plugin has three lifecycle jobs: `register(api)` declares contributions synchronously during client construction; `setup(client, api?)` starts async resources during `client.start()`; `teardown(client, api?)` releases them during `client.close()`. The page documents the startup ordering, reverse teardown with error aggregation, the per-plugin diagnostics snapshot exposed on `client.plugins.diagnostics`, the lifecycle hooks (`commands:afterLoad`, `plugins:ready`, …), and the `SeyfertPluginError` / `SeyfertPluginAggregateError` wrappers that attribute failures to a plugin / phase / index. All APIs verified against `src/client/plugins.ts` and `src/client/plugins/*`.

## Key APIs (verified)

Lifecycle hooks on a plugin (`SeyfertPlugin` / `createPlugin` input, `src/client/plugins/types.ts:508-511`, `src/client/plugins.ts:180-184`):
- `options?(current: Readonly<BaseClientOptions>): SeyfertPluginOptions` — contribute option fragments during resolution.
- `register?(api: SeyfertPluginApi): void` — synchronous; declares contributions. One argument only (the api).
- `setup?(client, api?): Awaitable<void>` — `api` is the FULL `SeyfertPluginApi` (mutations allowed; created with scope `'setup'`).
- `teardown?(client, api?): Awaitable<void>` — `api` is `SeyfertPluginTeardownApi`, a read-only subset: `Pick<SeyfertPluginApi, 'has' | 'diagnostics'> & { shared: { has } }` (`types.ts:479-481`). Calling any mutating method during teardown throws a `SeyfertPluginError` (phase like `commands.add`), via `assertCanMutate` in `src/client/plugins/api.ts:98-107`.

`createPlugin(...)` is the verified factory and is identity at runtime (`plugins.ts:261-265` — it just returns its argument; all the work is in the types). `definePlugins(...)` accepts either a single array or rest args (`plugins.ts:283-289`). Both are root exports.

Lifecycle status — `PluginLifecycleStatus` (`types.ts:86`): `'registered' | 'setting-up' | 'ready' | 'closing' | 'closed' | 'failed'`. Verified transitions in `plugins.ts`: setup sets `setting-up`→`ready`/`failed`; teardown sets `closing`→`closed`/`failed`.

Lifecycle/runtime phases — `PluginLifecyclePhase` (`types.ts:95-107`): `'resolve' | 'requires' | 'options' | 'register' | 'client' | 'ctx' | 'shared' | 'commands.add' | 'components.add' | 'setup' | 'teardown' | 'event:${string}'`. The actual `phase` field is an open string; more phases are emitted in practice (e.g. `autocomplete.wrap`, `gateway.onDispatch`, `gateway.wrapSendPayload`, `hook:<name>`, `cache.resource`, `reload`, `shared.<name>`, `middlewares.<name>`).

Plugin hooks — `SeyfertPluginHooks` (`types.ts:199-209`), subscribed with `api.hooks.on(name, handler)`. Payload shapes DIFFER per hook (do not assume `[client]` everywhere):
- `'plugins:setupComplete'` / `'plugins:ready'` — `[client]`.
- `'commands:beforeLoad'` — `[client, dir?]`; `'commands:afterLoad'` — `[metadata: PluginLoadedMetadata<'commands'>]`.
- `'components:beforeLoad'` — `[client, dir?]`; `'components:afterLoad'` — `[metadata: PluginLoadedMetadata<'components'>]`.
- `'events:beforeLoad'` AND `'events:afterLoad'` — both `[client, dir?]` (events:afterLoad is NOT metadata, unlike commands/components).
- `'client:close'` — `[client]`.

`PluginLoadedMetadata<TKind>` carries `kind`, `total`, `items`, and `plugin.{ total, sources }` (`types.ts:128+`). Handlers return `Awaitable<unknown>`; a throwing hook is caught, wrapped (`hook:<name>`), routed to plugin event-error observers, and logged (`plugins.ts:518-558`) — it does not abort startup.

Diagnostics — `client.plugins` is a `ResolvedPluginList` (`types.ts:483-486`): a frozen array also carrying `.resolved` and `.diagnostics` (`readonly PluginDiagnostics[]`). Each `PluginDiagnostics` (`types.ts:514-544`, built in `registry.ts`) has: `name`, `instanceId?`, `index`, `status`, `imports`, `clientKeys`, `ctxKeys`, count fields `commands`/`components`/`modals`/`restObservers`/`commandObservers`/`autocompleteWrappers`/`gatewayIntents`/`gatewaySendPayloadWrappers`/`gatewayDispatchInterceptors`/`handlerCreators`/`handlerTransformers`/`anyEvents`/`eventErrors`, name lists `events`/`middlewares`/`shared`/`cacheResources`/`langs`/`hooks`, `requirements` (`PluginRequirementDiagnostic[]`), `messages` (`PluginDiagnosticMessage[]` — where warnings/info live), and `truncated?` (per-key overflow counts; each list is capped at 50).

`PluginDiagnosticMessage` (`types.ts:109-118`): `{ plugin, instanceId?, index, phase, severity: 'info'|'warn'|'error', code?, message, data? }`.
`PluginRequirementDiagnostic` (`types.ts:120-127`): `{ kind: 'plugin'|'capability', req, range?, resolvedVersion?, optional, satisfied }`.

Requirements — a plugin's `requires?: readonly PluginRequirementInput[]` (`types.ts:90-93`) is one of:
- a bare `` `plugin:${string}` `` string (a `PluginRequirement`),
- `{ req: 'plugin:name', range?: SemverRange, optional?: boolean }`,
- `{ capability: SharedKey, optional?: boolean }` (a shared-value requirement).
At runtime `api.has(req)` checks a `` `plugin:${string}` `` requirement; capability requirements are validated after register (`plugins.ts:360`). `range` accepts `'1.2.3'`, `'>=1.2.3'`, `'^1.2.3'`, `'~1.2.3'`.

Shared state — `createSharedKey<T>(name)` (or curried `createSharedKey<T>()(name)`) returns a frozen `SharedKey<T, Name>` (`shared.ts:22-31`). Producers call `api.shared.set(key, factory, { dispose?, override? })`; consumers read `client.shared.get(key)` / `client.shared.unwrap(key)` / `client.shared.has(key)` (`types.ts:43-48`). `dispose` runs during teardown.

Ordering — `PluginOrder` enum (`types.ts:69-72`): `Before = 'before'`, `After = 'after'`. `PluginOrderOpt = PluginOrder.Before | PluginOrder.After | number`. Pass `{ order }` to contribution methods (events, observers, wrappers, hooks, …) to bias placement within its band.

Errors (`src/client/plugins/errors.ts`):
- `SeyfertPluginError extends SeyfertError` — `.name === 'SeyfertPluginError'`, fields `plugin`, `instanceId?`, `phase`, `index`, and `cause` (original error). Default code `'PLUGIN_FAILED'`.
- `SeyfertPluginAggregateError extends SeyfertError` — `.name === 'SeyfertPluginAggregateError'`, `.errors: unknown[]`, `.cause`. Thrown when setup fails AND cleanup/teardown also fails, and when reverse teardown collects multiple errors. Code `'PLUGIN_FAILED'` or `'PLUGIN_TEARDOWN_FAILED'`. Has a `toJSON()`.
- `SeyfertPluginErrorCode = 'PLUGIN_FAILED' | 'PLUGIN_TEARDOWN_FAILED'`.
- All three are re-exported from root `'seyfert'` (`plugins.ts:40-41`, barrel `src/client/index.ts`).

Not a plugin API (confirmed absent in source): there is no `dependsOn`, numeric plugin priority, or `PluginEnforce`. Use `imports` (bring plugins along), `requires`, `api.has(req)` for feature checks, and shared state (`api.shared.set/has`, `createSharedKey`).

## Startup & Close Order (verified `base.ts:391-422`)

`client.start()` plugin-loading sequence (in order):
1. `setupPlugins()` runs every plugin `setup(...)` (in resolved order; failures tear completed plugins back down).
2. plugin cache resources + plugin middlewares refresh.
3. hooks fire: `plugins:setupComplete`, then `plugins:ready`.
4. cache adapter `start()`.
5. langs load.
6. `commands:beforeLoad` → disk commands load → plugin commands apply.
7. `components:beforeLoad` → disk components load → plugin components/modals apply.
8. `reloadPluginContributions()` fires `commands:afterLoad` + `commandsLoaded`, then `components:afterLoad` + `componentsLoaded`.
For a gateway `Client`, disk event files load after this, with `events:beforeLoad`/`events:afterLoad` around them, before gateway execution.

`client.close()` waits for in-flight setup, then runs plugin `teardown` in REVERSE resolved order (`[...records].reverse()`, `plugins.ts:731`), disposing each plugin's shared values along the way. It does NOT close the gateway, REST client, or cache adapter — do that yourself.

## Code Examples (verified)

Lifecycle plugin with client + ctx helpers (matches doc; `plugins.ts:240`):

```ts
import { createPlugin } from 'seyfert';

export function schedulerPlugin(options: SchedulerOptions) {
    const registry = new SchedulerRegistry(options);

    return createPlugin({
        name: 'scheduler',
        client: { scheduler: () => registry }, // client.scheduler
        ctx: { scheduler: () => registry },    // ctx.scheduler
        setup(client) {
            // start async resources during client.start()
            return client.scheduler.setup(client);
        },
        teardown(client) {
            // release them during client.close()
            return client.scheduler.close();
        },
    });
}
```

Shared state across plugins, with a `dispose` that runs on teardown:

```ts
import { createPlugin, createSharedKey } from 'seyfert';

// A typed handle two plugins can agree on without importing each other.
export const dbKey = createSharedKey<Database>('db');

export const databasePlugin = createPlugin({
    name: 'database',
    register(api) {
        // factory runs lazily; dispose runs in reverse order during close()
        api.shared.set(dbKey, () => new Database(process.env.DATABASE_URL!), {
            dispose: db => db.end(),
        });
    },
    setup(client) {
        return client.shared.unwrap(dbKey).connect();
    },
});

// A consumer plugin: declare the capability it needs and read it lazily.
export const economyPlugin = createPlugin({
    name: 'economy',
    requires: [{ capability: dbKey }], // validated after register
    client: {
        balances: () => undefined as unknown as BalanceService, // placeholder
    },
    setup(client) {
        const db = client.shared.unwrap(dbKey); // throws if unset
        (client as { balances: BalanceService }).balances = new BalanceService(db);
    },
});
```

Plugin-version requirement + runtime feature check via `api.has`:

```ts
import { createPlugin } from 'seyfert';

export const moderationPlugin = createPlugin({
    name: 'moderation',
    // hard dependency with a semver range; optional soft dep checked at runtime
    requires: [{ req: 'plugin:logging', range: '^2.0.0' }],
    register(api) {
        if (api.has('plugin:analytics')) {
            // wire up an extra observer only when analytics is present
            api.commands.observe({ onAfterRun: () => {/* count usage */} });
        }
    },
});
```

Observability via lifecycle hooks (counts loaded commands after disk + plugin load):

```ts
import { createPlugin } from 'seyfert';

export const auditPlugin = createPlugin({
    name: 'audit',
    register(api) {
        api.hooks.on('commands:afterLoad', metadata => {
            // metadata: PluginLoadedMetadata<'commands'>
            console.log(`loaded ${metadata.total} commands`, metadata.plugin.sources);
        });
        // events:afterLoad is [client, dir] — NOT metadata
        api.hooks.on('events:afterLoad', (client, dir) => {
            client.logger.info('events loaded from', dir);
        });
        api.hooks.on('client:close', client => client.logger.info('shutting down'));
    },
});
```

Walk diagnostics for a startup health report (verified — `.diagnostics` on `client.plugins`):

```ts
for (const diagnostic of client.plugins.diagnostics) {
    client.logger.info(diagnostic.name, diagnostic.status,
        `(${diagnostic.commands} cmds, ${diagnostic.middlewares.length} mws)`);

    // requirement results: surface unmet hard deps
    for (const req of diagnostic.requirements) {
        if (!req.satisfied && !req.optional) {
            client.logger.error('unmet requirement', diagnostic.name, req.req, req.range);
        }
    }

    // diagnostic.messages holds warnings/info (severity info|warn|error) — NOT `warnings`
    for (const message of diagnostic.messages) {
        client.logger.warn(message.phase, message.severity, message.message);
    }

    if (diagnostic.truncated) {
        client.logger.warn('diagnostic lists capped at 50', diagnostic.truncated);
    }
}
```

Catch attributed failures — match BOTH error classes (verified):

```ts
import { SeyfertPluginError, SeyfertPluginAggregateError } from 'seyfert';

try {
    await client.start();
} catch (error) {
    if (error instanceof SeyfertPluginError) {
        client.logger.error(error.plugin, error.phase, error.index, error.cause);
    } else if (error instanceof SeyfertPluginAggregateError) {
        // teardown failures, and setup+cleanup double-failures, land here
        client.logger.error('multiple plugin failures', error.errors);
    } else {
        throw error;
    }
}
```

Manual shutdown — teardown is read-only (the gotcha):

```ts
export const connectionPlugin = createPlugin({
    name: 'connection',
    setup(client, api) {
        // api is the FULL mutable SeyfertPluginApi here
        api?.events.on('ready', () => client.logger.info('up'));
    },
    teardown(client, api) {
        // api is read-only: only has / diagnostics.warn / shared.has
        if (api?.has('plugin:metrics')) api.diagnostics.warn('metrics still loaded');
        // api?.commands.add(...) would THROW (cannot mutate during teardown)
        return client.myConnection.close();
    },
});

// you still own the rest of shutdown:
await client.close();          // runs plugin teardown (reverse order)
await client.gateway.disconnectAll(); // close() does NOT touch the gateway
```

## Common patterns / gotchas

- `setup` gets the mutable api; `teardown` gets a read-only api — any contribution mutation in teardown throws a `SeyfertPluginError`. teardown should only `has`, `diagnostics.warn`, `shared.has`, and release its own resources.
- Catch BOTH `SeyfertPluginError` and `SeyfertPluginAggregateError`. Checking `error.name === 'SeyfertPluginError'` (as older docs show) misses the aggregate case thrown by teardown and by setup+cleanup double-failures.
- The diagnostics field is `messages` (with `severity`), not `warnings`. There is no single `wrappers` count — use `autocompleteWrappers`, `gatewaySendPayloadWrappers`, `gatewayDispatchInterceptors`. `shared`/`events`/`middlewares` are name lists, not counts.
- `commands:afterLoad` / `components:afterLoad` hand you `PluginLoadedMetadata`; `events:afterLoad` and all `*:beforeLoad` hand you `[client, dir]`. Don't conflate.
- A throwing lifecycle hook is caught and logged (`hook:<name>`), not fatal. A throwing `setup` IS fatal and tears down already-completed plugins.
- `client.close()` only runs plugin teardown + waits for in-flight setup; close the gateway, REST, and cache adapter yourself.
- Diagnostic lists are capped at 50 per key — check `diagnostic.truncated` if a list looks short.
- No `dependsOn` / numeric priority / `PluginEnforce`. Express dependencies with `requires` (+ `api.has`) and ordering with `imports` / `{ order: PluginOrder.Before | After | number }`.

## Doc vs Source Corrections

- Diagnostics field name: docs list "warnings" -> src exposes `messages: PluginDiagnosticMessage[]`; there is no `warnings` field (`types.ts:542`).
- Docs say diagnostics include "contribution counts for commands, components, modals, wrappers, and shared values" -> src has no single `wrappers` count (real fields: `autocompleteWrappers`, `gatewaySendPayloadWrappers`, `gatewayDispatchInterceptors`), and `shared` is a name list, not a count (`types.ts:530-539`).
- Docs say teardown errors "are aggregated" but only name `SeyfertPluginError` -> src throws `SeyfertPluginAggregateError` (with `.errors`) for teardown and for setup+cleanup double-failure (`plugins.ts:686,703,774`). Catch both.
- Docs error sample uses `error.name === 'SeyfertPluginError'` -> prefer `instanceof`, and also handle `SeyfertPluginAggregateError`.
- Docs list error phases including `gateway.wrapSendPayload` but omit several -> `phase` is an open string; also emitted: `gateway.onDispatch`, `autocomplete.wrap`, `cache.resource`, `hook:<name>`, `reload`, `requires`, `resolve`, `shared.<name>` (`api.ts`, `plugins.ts`).
- Otherwise the lifecycle description, startup-order list, reverse-teardown behavior, and "what is not a plugin API" section are accurate against source.

## Source Anchors

- `src/client/plugins.ts` — `createPlugin`/`definePlugins`/`createPluginFactory`/`createContextScope`, `setupClientPlugins`, `teardownClientPlugins` (reverse order + aggregate errors), `runPluginHooks`, re-exports (`SeyfertPluginError`, `SeyfertPluginAggregateError`, `createSharedKey`, `PluginOrder`).
- `src/client/plugins/types.ts` — `SeyfertPlugin`, `SeyfertPluginApi`, `SeyfertPluginTeardownApi`, `SeyfertPluginHooks`, `PluginHookName`, `PluginLoadedMetadata`, `PluginLifecycleStatus`, `PluginLifecyclePhase`, `PluginDiagnostics`, `PluginDiagnosticMessage`, `PluginRequirementDiagnostic`, `PluginRequirementInput`, `PluginOrder`, `ResolvedPluginList`, `SharedKey`.
- `src/client/plugins/api.ts` — `createPluginApi`, scope `'setup'`/`'teardown'`, `assertCanMutate` (teardown read-only), full api surface (events/commands/rest/hooks/handlers/components/modals/middlewares/autocomplete/gateway/cache/shared/langs/reload/diagnostics/options).
- `src/client/plugins/registry.ts` — diagnostics field assembly, list cap 50 / `truncated`, requirement validation.
- `src/client/plugins/errors.ts` — `SeyfertPluginError`, `SeyfertPluginAggregateError`, `wrapPluginError`, `createPluginConflictError`.
- `src/client/plugins/shared.ts` — `createSharedKey`, `addPluginShared`/`removePluginShared`, disposal.
- `src/client/base.ts:391-422` — verified startup sequence (`setupPlugins` → hooks → cache → langs → commands → components) and `client.close()` (does NOT close gateway/REST/cache adapter).

## Agent Guidance

- Import everything from root: `import { createPlugin, definePlugins, createSharedKey, PluginOrder, SeyfertPluginError, SeyfertPluginAggregateError } from 'seyfert'`. No deep `seyfert/lib/...` import needed.
- For health checks / optional-dependency UX, read `client.plugins.diagnostics`: inspect `status`, `requirements` (`satisfied`/`optional`/`range`), and `messages` (severity `info|warn|error`) — not a `warnings` field.
- When detecting plugin failures at `client.start()`/`client.close()`, match BOTH `SeyfertPluginError` and `SeyfertPluginAggregateError`.
- Put async resource startup in `setup` (it can use this plugin's and imported plugins' client helpers) and release in `teardown`. teardown is read-only — it inspects (`has`, `diagnostics.warn`, `shared.has`) and cleans up its own resources, never `*.add(...)`.
- For cross-plugin contracts use `createSharedKey` + `api.shared.set`/`client.shared.unwrap`; declare hard deps with `requires` and gate optional behavior with `api.has(...)`.
- For observability hook into `commands:afterLoad` (metadata), `plugins:ready`, or `client:close`; remember `events:afterLoad` is `[client, dir]`, not metadata.
- `client.close()` only handles plugin teardown — close the gateway, REST, and cache adapter yourself.
