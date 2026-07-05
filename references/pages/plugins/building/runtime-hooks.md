# Runtime Hooks

Original source URL: https://seyfert-web-git-seyfert-v5-tiramisulabs.vercel.app/docs/plugins/building/runtime-hooks

Coverage reference: plugins.md

Verification status: Source-verified (src/client/plugins/api.ts + types.ts + src/api/shared.ts, the authoritative Seyfert source)

## Page Summary

Plugins hook framework-level runtime paths from inside their `register(api)` callback (and `setup`'s
optional 2nd arg). The `SeyfertPluginApi` exposes namespaced hook groups: `autocomplete.wrap`,
`gateway` (intents, outbound send-payload wrappers, inbound dispatch interceptors), `rest.observe`,
`langs.contribute`, lifecycle `hooks.on`, and custom events via `events.on` (e.g. `uploadCommands`).
These are low-level: prefer normal events / shared state unless your package genuinely needs to bend
runtime behavior. Several hooks veto via `null` and record a typed diagnostic; some return a
`PluginDisposer` for runtime teardown.

## Key APIs (verified)

All hook methods live on the object passed to `register(api)`. Source: `src/client/plugins/api.ts`,
types in `src/client/plugins/types.ts`.

- `api.autocomplete.wrap(wrapper, opts?)` -> `void` (NOT a disposer). `opts?: { order?: PluginOrderOpt }`.
  - `wrapper: (payload: PluginAutocompletePayload, next: PluginAutocompleteNext) => Awaitable<void>`.
  - `PluginAutocompletePayload = { client: BaseClient, command?: CommandAutocompleteOption, interaction: AutocompleteInteraction, optionsResolver: OptionResolverStructure }`. `command` is optional.
  - `next: () => Awaitable<void>`. Wrappers compose around the original handler (`reduceRight`, runtime `runPluginAutocompleteWrappers` in `src/client/plugins.ts:471`).
- `api.gateway.addIntents(...intents: PluginIntentResolvable[])` -> `void` (variadic).
  - `PluginIntentResolvable = keyof typeof GatewayIntentBits | GatewayIntentBits | number`. Unknown names/bits are skipped with a `unknown-intent-bits` diagnostic (does NOT throw).
- `api.gateway.wrapSendPayload(wrapper, opts?)` -> `void`. `opts?: { order? }`.
  - `wrapper: (p: PluginGatewaySendPayload) => Awaitable<GatewaySendPayload | null | undefined | void>`.
  - `PluginGatewaySendPayload = { client: BaseClient, payload: GatewaySendPayload, shardId: number }`.
  - Return a payload to replace, `undefined`/`void` to keep, `null` to consume (records `gateway-send-payload-veto`).
- `api.gateway.onDispatch(interceptor, opts?)` -> `PluginDisposer`. `opts?: { order? }`.
  - `interceptor: (packet: GatewayDispatchPayload, next: PluginGatewayDispatchNext, meta: PluginGatewayDispatchMeta) => Awaitable<GatewayDispatchPayload | null | undefined | void>`.
  - `next: (packet?) => Awaitable<GatewayDispatchPayload | null>`; `meta = { readonly client, readonly shardId }`.
  - Runs BEFORE cache and events. Return `null` to veto (records `gateway-dispatch-veto`). Calling `next()` twice throws `gateway.onDispatch next() was called multiple times.` (`plugins.ts:577`)
- `api.rest.observe(observer, opts?)` -> `PluginDisposer`. `opts?: { order? }`. Observers stack; a throwing observer is isolated.
  - `observer: PluginRestObserver` (= `RestObserver<BaseClient>`, `src/api/shared.ts:64`): `onRequest?`, `onSuccess?`, `onFail?`, `onRatelimit?`.
  - Payloads share `{ readonly client, method: HttpMethods, url: \`/${string}\`, request }`. `onSuccess`/`onRatelimit` add `response: Response`; `onFail` adds `error: unknown, statusCode?: number`. There is NO `route` field.
- `api.langs.contribute(locale, values, opts)` -> `void`. `opts: PluginLangOptions = { readonly prefix: string }` is REQUIRED.
  - Throws (`createPluginConflictError`) if `prefix` has no non-empty path segment (`isValidLangPrefix`, `api.ts:524`). App then reads `ctx.t.<prefix>.<key>`.
- `api.hooks.on(name, handler, opts?)` -> `PluginDisposer`. Lifecycle hooks (`SeyfertPluginHooks`, `types.ts:200`).
  - Names + payloads: `plugins:ready`/`plugins:setupComplete`/`client:close` -> `[client]`; `commands:beforeLoad`/`components:beforeLoad`/`events:beforeLoad`/`events:afterLoad` -> `[client, dir?]`; `commands:afterLoad`/`components:afterLoad` -> `[metadata: PluginLoadedMetadata<...>]`. The `client` arg is widened with this plugin's own extension `E`.
- `api.events.on(name, handler, opts?)` -> `PluginDisposer`. Also `.once(name, handler)`, `.onAny(handler, opts?)`, `.onError(handler, opts?)`. Accepts client/gateway/custom event names. `opts?: { once?, order? }`.
  - The custom `uploadCommands` event: `handler: (metadata: PluginUploadCommandsMetadata) => unknown`.
  - `PluginUploadCommandsMetadata = { applicationId: string, cachePath?: string, commands: number, guildId?: string, reason: 'cache-hit'|'cache-miss'|'forced', scope: 'global'|'guild', status: 'skipped'|'uploaded' }`. Note `commands` is a COUNT, not an array.

Ordering: `opts.order` is `PluginOrderOpt = PluginOrder.Before | PluginOrder.After | number` (`PluginOrder` is a string enum: `Before='before'`, `After='after'`). Lower numbers / `Before` run earlier.

Imports (all from `'seyfert'`): `createPlugin`, `definePlugins`, `PluginOrder`. Types like
`PluginAutocompletePayload`, `PluginRestObserver`, `PluginGatewayDispatchMeta`,
`PluginUploadCommandsMetadata`, `SeyfertPluginApi` are re-exported from `'seyfert'` too
(`plugins.ts:43`+type re-exports).

## Code Examples (verified)

### 1. All runtime hooks in one plugin

```ts
import { createPlugin, PluginOrder } from 'seyfert';
import { metrics } from './metrics';

export const metricsPlugin = createPlugin({
    name: 'autocomplete-metrics',
    register(api) {
        // (a) Autocomplete timing — wrap returns void, NOT a disposer
        api.autocomplete.wrap(async ({ command, interaction, optionsResolver }, next) => {
            const startedAt = Date.now();
            try {
                await next(); // run the original autocomplete handler
            } finally {
                metrics.recordAutocomplete({
                    command: command?.name, // command is optional
                    fullCommandName: optionsResolver.fullCommandName,
                    interactionId: interaction.id,
                    duration: Date.now() - startedAt,
                });
            }
        });

        // (b) Contribute gateway intents (variadic; unknown names skipped, not thrown)
        api.gateway.addIntents('Guilds', 'GuildMembers');

        // (c) Mutate outbound gateway payloads
        api.gateway.wrapSendPayload(({ payload, shardId }) => {
            if (shouldDrop(payload, shardId)) return null; // consume + veto diagnostic
            return { ...payload, d: patch(payload.d) };     // replace; undefined keeps as-is
        });

        // (d) Inbound dispatch interceptor (runs before cache + events); returns a disposer
        const disposeDispatch = api.gateway.onDispatch((packet, next, meta) => {
            metrics.increment('gateway.dispatch', { event: packet.t ?? 'unknown', shardId: meta.shardId });
            return next(packet); // return null to veto the packet
        });

        // (e) REST observers stack; order via opts. NOTE: no `route` field exists.
        const disposeRest = api.rest.observe({
            onRequest({ method, url }) {},
            onSuccess({ method, url, response }) {
                metrics.increment('rest.success', { method, url, status: response.status });
            },
            onFail({ method, url, error, statusCode }) {},
            onRatelimit({ method, url, response }) {}, // <- method/url/response, no `route`
        }, { order: PluginOrder.Before });

        // (f) Namespaced i18n overlay (opts.prefix REQUIRED). Read via ctx.t.plugins.economy.balance
        api.langs.contribute('en-US', { balance: 'Balance' }, { prefix: 'plugins.economy' });

        // (g) Observe command uploads (custom event)
        api.events.on('uploadCommands', metadata => {
            metadata.applicationId;
            metadata.commands; // number (count), not the command list
            metadata.scope;    // "global" | "guild"
            metadata.status;   // "uploaded" | "skipped"
            metadata.reason;   // "cache-hit" | "cache-miss" | "forced"
        });

        // disposers only needed for runtime detach; teardown auto-cleans register-scope hooks
        void disposeDispatch;
        void disposeRest;
    },
});
```

### 2. Lifecycle hooks via `api.hooks.on`

`api.hooks.on(...)` fires on the framework's own load/ready/close milestones — handy for warm-up,
deploy logging, and graceful shutdown without owning the `Client`.

```ts
import { createPlugin } from 'seyfert';

export const lifecyclePlugin = createPlugin({
    name: 'lifecycle-logger',
    register(api) {
        // commands finished loading -> metadata carries counts, not a Client
        api.hooks.on('commands:afterLoad', metadata => {
            console.log(`[commands] loaded ${metadata.total} (plugin-contributed: ${metadata.plugin.total})`);
        });

        // events about to load -> [client, dir?]
        api.hooks.on('events:beforeLoad', (client, dir) => {
            console.log(`[events] loading from ${dir ?? '<inline>'}`);
        });

        // everything is up
        api.hooks.on('plugins:ready', client => {
            console.log(`[ready] ${client.applicationId ?? 'app'} online`);
        });

        // graceful shutdown — flush buffers, close sockets
        api.hooks.on('client:close', async client => {
            await flushPendingMetrics();
        });
    },
});
```

### 3. Full feature plugin: hooks + extension + lifecycle + i18n

A realistic reusable package: a typed client extension, a global middleware, intents, a namespaced
lang overlay, REST instrumentation, and a connection that opens on `setup` and closes on `teardown`.

```ts
import { createPlugin, createMiddleware, definePlugins, Client } from 'seyfert';

const balanceGuard = createMiddleware<void>(({ context, next, stop }) => {
    // stop() with no args = skip silently; stop('reason') = deny -> onMiddlewaresError (v5: no pass())
    if (!context.guildId) return stop();
    return next();
});

export const economyPlugin = createPlugin({
    name: 'economy',
    version: '1.0.0',
    // Typed client extension: `client.economy` is real & inferred everywhere.
    client: { economy: () => new EconomyService() },
    register(api) {
        api.middlewares.add('balanceGuard', balanceGuard, { global: true });
        api.gateway.addIntents('Guilds');
        api.langs.contribute('en-US', { insufficient: 'Not enough coins.' }, { prefix: 'economy' });

        // Plugin observers run before direct observers and legacy callbacks.
        api.rest.observe({
            onFail({ method, url, statusCode }) {
                if (statusCode === 429) console.warn(`[economy] throttled ${method} ${url}`);
            },
        });
    },
    setup: client => client.economy.connect(),   // runs once client is ready
    teardown: client => client.economy.close(),  // cleanup that actually runs
});

const client = new Client({ plugins: definePlugins(economyPlugin) });
```

### 4. Shard traffic recorder (send wrapper + dispatch interceptor)

```ts
import { createPlugin } from 'seyfert';

export const trafficPlugin = createPlugin({
    name: 'shard-traffic',
    register(api) {
        // Outbound: count + optionally rewrite presence updates
        api.gateway.wrapSendPayload(({ payload, shardId }) => {
            recordOut(shardId, payload.op);
            return undefined; // keep payload unchanged (void/undefined = no-op)
        });

        // Inbound: drop noisy TYPING_START before it ever hits cache/events
        api.gateway.onDispatch((packet, next, meta) => {
            recordIn(meta.shardId, packet.t);
            if (packet.t === 'TYPING_START') return null; // veto -> gateway-dispatch-veto diagnostic
            return next(packet);
        });
    },
});
```

## Common patterns / gotchas

- WHEN to reach for these: only framework-level packages (metrics, gateway/REST instrumentation, i18n
  bundles, deploy tooling). For app logic prefer regular events (`createEvent`) and plugin shared state.
- Disposers vs void: `onDispatch`, `rest.observe`, `hooks.on`, and `events.on/once/onAny/onError`
  return a `PluginDisposer`; `autocomplete.wrap`, `gateway.wrapSendPayload`, and `gateway.addIntents`
  return `void`. Save a disposer only if you need to detach at RUNTIME — register-scope contributions
  are cleaned up automatically on plugin `teardown`.
- Veto semantics: return `null` to drop (recorded as a diagnostic, never silent); return
  `undefined`/`void` to keep the current value. `onDispatch` MUST call `next(packet)` to continue, and
  exactly once (twice throws).
- Keep `onDispatch` interceptors fast — they run before cache and events on EVERY inbound packet.
- `langs.contribute` requires `{ prefix }` and throws on an empty/dotted-only prefix. App reads land
  under `ctx.t.<prefix>...`, so namespace it (`'plugins.myfeature'`) to avoid colliding with app keys.
- `uploadCommands` metadata `.commands` is a COUNT, not the command array. Use it for deploy metrics,
  not to inspect uploaded commands.
- `addIntents` never throws on a bad name — it logs an `unknown-intent-bits` diagnostic and skips it.
  Watch plugin diagnostics in dev if an intent seems missing.
- Teardown is read-only: `SeyfertPluginTeardownApi` only exposes `has`, `diagnostics`, and
  `shared.has`. Calling any mutator (`*.add`, `*.observe`, `wrap`, `addIntents`, `contribute`, ...)
  during teardown throws via `assertCanMutate` -> `createPluginConflictError`.
- Ordering across plugins: pass `{ order: PluginOrder.Before | PluginOrder.After | <number> }`. For
  REST, the documented run order is plugin observers, then direct `client.rest.observe(...)`, then
  legacy callbacks.

## Doc vs Source Corrections

- REST `onRatelimit`: docs show `onRatelimit({ route })` -> src `RestObserverRatelimitPayload` has NO
  `route` field; available fields are `client, method, url, request, response` (`src/api/shared.ts:60`).
  Use `method`/`url`.
- `uploadCommands` metadata: docs imply `metadata.commands` is the command list -> src
  `PluginUploadCommandsMetadata.commands` is a NUMBER (count). Src also adds `cachePath?` / `guildId?`
  not shown in docs (`src/client/plugins/types.ts:139`).
- `langs.contribute`: docs present `{ prefix }` informally -> src REQUIRES the 3rd argument and THROWS
  if `prefix` resolves to no non-empty segment (`api.ts:521-541`).
- Return types: `autocomplete.wrap` / `gateway.wrapSendPayload` / `gateway.addIntents` return `void`;
  only `gateway.onDispatch`, `rest.observe`, `hooks.on`, `events.*` return a `PluginDisposer`. Docs are
  silent; verified in `src/client/plugins/api.ts`.
- The MDX page omits `api.hooks.on` (lifecycle hooks) — it exists and is a genuine runtime hook
  (`SeyfertPluginHooks`, `types.ts:200`); added above.
- Otherwise the doc examples are accurate: veto-via-`null`, `gateway-dispatch-veto` /
  `gateway-send-payload-veto` diagnostics, intents resolved from `GatewayIntentBits`, stacking
  observers, and `ctx.t.<prefix>` reads all match source.

## Source Anchors

- `src/events/event.ts` — `CustomEvents.uploadCommands` signature.
- `src/client/base.ts` — `uploadCommands()` emits the custom event with the metadata.

## Agent Guidance

- For lifecycle reactions (load/ready/close) use `api.hooks.on`; for Discord/custom events use
  `api.events.on`; for command-deploy telemetry use `api.events.on('uploadCommands', ...)`.
