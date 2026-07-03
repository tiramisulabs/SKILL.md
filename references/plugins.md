# Plugins

Source of truth: `src/client/plugins.ts` + `src/client/plugins/{types,api,registry,shared,order,errors}.ts`.
All public symbols are root exports from `'seyfert'` (`src/client/index.ts` does `export * from './plugins'`). Never deep-import (`seyfert/lib/...`).

Root exports: `createPlugin`, `createPluginFactory`, `definePlugins`, `createSharedKey`, `createContextScope`, `PluginOrder`, `SeyfertPluginError`, `SeyfertPluginAggregateError`, and the `Plugin*`/`Seyfert*` types (`SeyfertPlugin`, `SeyfertPluginApi`, `SeyfertRegistry`, `SharedKey`, `SharedValue`, `PluginRequirementInput`, `SemverRange`, `RegisteredPluginShared`, `PluginDiagnostics`, …).

A plugin is the v5 replacement for monkey-patching the `Client`. It can add/remove commands, components, modals; register middlewares (incl. global); extend `client.*` and `ctx.*` with typed factories; add cache resources and lang overlays; observe REST and command execution; intercept gateway traffic; expose shared services; and emit diagnostics — all with types, a lifecycle, and teardown.

## Install (definePlugins + SeyfertRegistry)

```ts
import { Client, definePlugins } from 'seyfert';
import { myPlugin } from './plugins/my-plugin';

export const plugins = definePlugins(myPlugin());     // one canonical tuple
declare module 'seyfert' {
  interface SeyfertRegistry { plugins: typeof plugins }
}
const client = new Client({ plugins });
```

`definePlugins(...plugins | plugins[])` is an identity helper preserving the tuple (accepts spread, a single array, or nothing). The `SeyfertRegistry.plugins` augmentation is what makes `client.*`, `ctx.*`, shared keys, and plugin middleware names typed in app code — a plugin that *runs* but is absent from this tuple has no global types. Build stateful plugins once and reuse the instance (`const s = storagePlugin(); definePlugins(s, dependentPlugin(s))`) — calling the factory twice yields two unrelated instances.

`SeyfertRegistry` is the single v5 augmentation surface (replaces `UsingClient`/`RegisteredMiddlewares`/`DefaultLocale`). Augment its `client` / `middlewares` / `langs` / `plugins` keys:

```ts
declare module 'seyfert' {
  interface SeyfertRegistry {
    client: ParseClient<Client<true>>;
    middlewares: typeof middlewares;     // bare typeof — ParseMiddlewares is GONE
    plugins: typeof plugins;
  }
}
```

## Authoring (createPlugin)

`createPlugin(config)` is an identity function at runtime (`return plugin`); its only job is type inference (extension `E`, context `C`, imports `I`, middlewares `M`, optional `meta`). No runtime validation — typos surface as type errors, not throws. Full `SeyfertPlugin` shape (`types.ts:492`):

- `name: string` (required, `const`-inferred), `instanceId?`, `version?` (semver, used by `requires` range checks), `meta?`
- `imports?: readonly AnySeyfertPlugin[]` — other plugins whose `E`/`C`/`M` flow into this plugin's api/client types; resolved before the importer
- `requires?: readonly PluginRequirementInput[]` — validated, NOT ordering (see below)
- `client?: { [k]: (client) => value }` — synchronous factories; become `client.k`
- `ctx?: { [k]: (interaction, client) => value }` — per-interaction factories; become `ctx.k`. Factory receives a 2nd `client` arg (`types.ts:255`).
- `middlewares?: Record<string, MiddlewareContext>`, `globalMiddlewares?: readonly (keyof middlewares & string)[]`
- `options?(current: Readonly<BaseClientOptions>): SeyfertPluginOptions` — contribute option fragments at resolve
- `register?(api): void` — synchronous; declares contributions
- `setup?(client, api?): Awaitable<void>` — async startup; `api` is the FULL `SeyfertPluginApi`
- `teardown?(client, api?): Awaitable<void>` — cleanup; `api` is the read-only `SeyfertPluginTeardownApi`

Keep `client`/`ctx` factories **synchronous**. `client` factories run **eagerly at client construction** in object **insertion order** — `installPluginClientMaps` (`registry.js`) does `Object.defineProperty(client, key, { value: factory(client) })` per key, so a later factory's `client` arg already sees earlier-installed keys (declare `redis` before an `api` factory that reads `client.redis`). Installation **throws** if `key in client`: the key collides with a native client member (`rest`, `cache`, `logger`, `langs`, `commands`, `components`, `shared`, `events`, `users`, `guilds`, `channels`, `webhooks`, …) or with another plugin's key — pick app-specific names. Do NOT add a typed client key by assigning in `setup` — declare it in `client` so it is typed and installed at the right time.

```ts
import { createPlugin } from 'seyfert';
export function loggerPlugin(opts: { name?: string } = {}) {
  const root = new SimpleLogger(opts.name);
  return createPlugin({
    name: 'example-logger',
    client: { exampleLogger: () => root },                              // client.exampleLogger (sync)
    ctx: { logger: interaction => root.child({ id: interaction.id }) }, // ctx.logger (per-interaction)
    register(api) { api.commands.add(PingCommand); },
    setup(client) { client.exampleLogger.info('ready'); },             // async startup
    teardown(client) { return client.exampleLogger.flush(); },         // cleanup
  });
}
```

`createPluginFactory({ defaults, validate?, factory })` returns `(options?: Partial<O>) => P`: merges `{...defaults, ...options}`, runs `validate`, then `factory`; thrown errors are wrapped via `wrapPluginError`.

```ts
export const economyPlugin = createPluginFactory({
  defaults: { startingBalance: 0 },
  validate: o => { if (o.startingBalance < 0) throw new Error('negative balance'); },
  factory: o => createPlugin({
    name: 'economy',
    client: { economy: () => new EconomyService(o.startingBalance) },
    setup: c => c.economy.connect(),
    teardown: c => c.economy.close(),
  }),
});
// economyPlugin() uses defaults; economyPlugin({ startingBalance: 100 }) overrides.
```

## Lifecycle

`register(api)` (sync, at construction) → `client`/`ctx` install → `setup(client, api?)` (async, during `client.start()`) → `teardown(client, api?)` (async, during `client.close()`, REVERSE resolved order). Status (`PluginLifecycleStatus`): `registered → setting-up → ready` (or `failed`); teardown `closing → closed` (or `failed`).

Startup order (`src/client/base.ts`): construction resolves/validates/registers/merges/binds → `client.start()` runs setup → disk commands load → plugin commands load after → `commands:afterLoad` → disk components → plugin components+modals → `components:afterLoad` → events load → gateway starts.

**Setup failures roll back:** if one plugin's `setup` throws, already-completed plugins are torn down in reverse and their shared values disposed (`plugins.ts:699-715`); the failure surfaces as `SeyfertPluginError` or — if cleanup also fails — `SeyfertPluginAggregateError`.

Important: `client.close()` only runs plugin `teardown` (reverse) and waits for in-flight setup. It does NOT close the gateway, REST transport, or cache adapter — wire that yourself if your plugin owns those resources.

## register-time api (SeyfertPluginApi)

The `api` passed to `register` (and `setup`). Methods returning a `PluginDisposer` can be detached at runtime; others return `void`.

- `api.has(req): boolean` — accepts ONLY `` `plugin:${string}` ``; returns `false` for anything else (CANNOT feature-check capabilities — use `api.shared.has(key)` for those).
- `events`: `on(name, handler, { once?, order? })` → disposer · `once(name, handler)` → disposer · `onAny(handler, { order? })` → disposer · `onError(handler, { order? })` → disposer. `name` is `ClientNameEvents | CustomEventsKeys | GatewayEvents`.
- `commands`: `add(...cmds, opts?)` · `remove(...names)` · `observe(observer, { order? })` → disposer · `defaults(hooks, { suppressDefault?, order? })`. `opts` may include `{ override?, guilds? }` (`PluginCommandContributionOptions`; guild-scoped emits a `command-guild-scope` info diagnostic).
- `components` / `modals`: `add(...items, opts?)` (`opts: { override? }`) · `remove(...customIds)` · `defaults(hooks, { suppressDefault?, order? })`.
- `middlewares`: `add(name, middleware, { global?, order?, override? })` · `remove(...names)`. `{ global: true }` registers it into `globalMiddlewares`.
- `rest.observe(observer, { order? })` → disposer — `onRequest?/onSuccess?/onFail?/onRatelimit?`. Payloads (`api/shared.ts:44-69`) are readonly and carry `{ client, method, url, request }`; success/ratelimit add `response: Response`; fail adds `error: unknown, statusCode?`. There is **no `route` field**.
- `hooks.on(name, handler, { order? })` → disposer — lifecycle hooks below.
- `handlers`: `construct(creator, opts?)` · `transform(transformer, opts?)` (`opts: { kinds?, order? }`; events have no construct step — `transform` with `kinds:['event']` only).
- `autocomplete.wrap(wrapper, { order? })` → void — `(payload, next) => Awaitable<void>`; composes around the handler.
- `gateway`: `addIntents(...intents)` → void (unknown skipped + `unknown-intent-bits` diagnostic) · `wrapSendPayload(wrapper, { order? })` → void · `onDispatch(interceptor, { order? })` → disposer (return `null` to veto a packet).
- `cache.resource(name, ResourceCtor, { onPacket?, intents? })` → void — throws on reserved/duplicate names (reserved set incl. `guilds`/`members`/`roles`/`channels`/`flush`/`bulkGet`/`constructor`/`__proto__`…, `api.ts:40-79`).
- `shared`: `set(key|name, factory, { dispose?, override? })` · `remove(...keys)` · `has(key|name)`.
- `langs.contribute(locale, values, { prefix })` → void — `prefix` REQUIRED, throws if no non-empty segment; app reads `ctx.t.<prefix>.<key>`.
- `diagnostics.warn(message, { code?, phase?, data? })`, `options.set(fragment)`, `reload(): Promise<void>`.

teardown's `api` is `SeyfertPluginTeardownApi = Pick<SeyfertPluginApi,'has'|'diagnostics'> & { shared: { has } }`. ANY mutation during teardown throws a `SeyfertPluginError` (via `assertCanMutate`).

## Lifecycle hooks (api.hooks.on) vs events

These are `api.hooks.on(name, handler)` keys (`SeyfertPluginHooks`, `types.ts:200`) — NOT `api.events.on`:

`plugins:ready` · `plugins:setupComplete` · `commands:beforeLoad` · `commands:afterLoad` · `components:beforeLoad` · `components:afterLoad` · `events:beforeLoad` · `events:afterLoad` · `client:close`.

Only `commands:afterLoad` and `components:afterLoad` receive a `PluginLoadedMetadata` (`{ kind, total, items, plugin: { total, sources } }`, `types.ts:129`). **`events:afterLoad` instead receives `[client, dir]`** (same shape as the `*:beforeLoad` hooks, `types.ts:208`) — easy to conflate. The first arg of `beforeLoad`/`ready`/`close` hooks is the client widened with this plugin's extension `E`.

`uploadCommands` is a CUSTOM EVENT (`api.events.on('uploadCommands', handler)`), not a hook. Its `PluginUploadCommandsMetadata.commands` is a COUNT (number), not a list.

## Shared state (createSharedKey)

Typed, lazy, single-owner runtime values for **plugin-to-plugin** services — prefer over global singletons and over `client.*` when end users won't call it directly.

```ts
import { createSharedKey } from 'seyfert';

class Ledger { balance(id: string) { return 0; } close() {} }
export const ledgerKey = createSharedKey<Ledger>()('ledger');   // curried form carries value type T

// producer (register/setup):
api.shared.set(ledgerKey, () => new Ledger(), { dispose: l => l.close() });

// consumer:
const maybe = client.shared.get(ledgerKey);     // Ledger | undefined (built lazily, then memoized)
const ledger = client.shared.unwrap(ledgerKey); // Ledger, THROWS if name not registered
client.shared.has(ledgerKey);                   // boolean, does not build it
```

`createSharedKey(name)` overloads (`shared.ts:22-32`): `createSharedKey<T>()(name)` for a typed key (no augmentation needed), or `createSharedKey('name')`. Factories are lazy (run on first `get`/`unwrap`, memoized, receive the client). Claiming an owned name throws unless `{ override: true }`; override queues disposal of the previous instance. Setting a shared also satisfies `{ capability: key }` requirements. `SharedValue<typeof key>` extracts the value type.

**Optional string-name access** — augment `RegisteredPluginShared` mapping the name to its **value type** (NOT `typeof key`):

```ts
declare module 'seyfert' {
  interface RegisteredPluginShared { ledger: Ledger }   // name → value type
}
client.shared.get('ledger');    // Ledger | undefined
client.shared.unwrap('ledger'); // Ledger
// with the augmentation, createSharedKey('ledger') also infers SharedKey<Ledger, 'ledger'>
```

## imports vs requires (NOT a dependency/priority API)

There is NO `dependsOn`, numeric priority, or `PluginEnforce`. Use:
- `imports`: bring another plugin along, flow its types, AND own its relative ordering (resolved before the importer; the same instance is deduped).
- `requires: PluginRequirementInput[]`: VALIDATION only —
  - `` `plugin:name` `` (string)
  - `{ req: 'plugin:name', range?: SemverRange, optional? }` (`range` only valid with `req`)
  - `{ capability: sharedKey, optional? }` (`req` and `capability` are mutually exclusive)
- `api.has('plugin:...')` for runtime plugin-presence checks; `api.shared.has(key)` / `client.shared.has(key)` for capabilities.

`SemverRange` = exact `x.y.z`, `>=x.y.z`, `^x.y.z`, `~x.y.z`; version resolves from `plugin.version` (else `plugin.meta.version`). Missing required requirement → throws (`SeyfertPluginError`). Missing optional → `missing-optional-requirement` diagnostic. Capability requirements are validated AFTER all `register` calls (`validatePluginRequirements(..., 'capability')`), so a capability set in `register` satisfies them. `requires` does NOT reorder: if a required plugin ends up ordered after the requirer and the requirer mutates contributions, a `requires-unordered` warn diagnostic fires — put it in `imports` to own the order. The safe combo is `imports: [x]` + `requires: ['plugin:x']` (own the order AND assert the version).

## Options, defaults, ordering

Plugins contribute client option fragments via top-level `options(current)` or `api.options.set(fragment)`. Merge precedence: framework defaults < plugin fragments < user `new Client({...})` options (user wins for plain merges). `context`/`contextScopes`/`globalMiddlewares` and command/component/modal `defaults` hooks are COMPOSED (chained, all run) rather than overwritten.

`api.commands.defaults(hooks, { suppressDefault? })` (and components/modals) compose lifecycle hooks across plugins; `suppressDefault: true | string[]` drops the framework default for those keys. Use these for cross-cutting hooks instead of repeating boilerplate per command.

Contribution ordering (`order.ts`, `orderBand`): `PluginOrder.Before` (band 0) → **any** `number` (band 1, sorted ascending, so negatives precede positives) → unspecified/default order (band 2) → `PluginOrder.After` (band 3); ties within a band broken by registration sequence. Note `0` is just a normal number in band 1 — it is NOT the "default", which is the *absence* of an `order` value (band 2, after all numbers). `PluginOrderOpt = PluginOrder.Before | PluginOrder.After | number`. The `order` option is accepted on `events.on`, `hooks.on`, `middlewares.add`, `rest.observe`, `autocomplete.wrap`, `gateway.wrapSendPayload`/`gateway.onDispatch`, and `*.defaults` (note `gateway.addIntents` takes NO `order`).

## Recipes

### App-wide services as a plugin (instead of augmenting `SeyfertRegistry.client`)

Prefer this over hand-augmenting `SeyfertRegistry.client` with `db`/`redis`/… props and assigning them imperatively after `new Client()`. Build the singletons once in the factory closure (so an `api` reuses the same `redis`), expose them via sync `client` factories, and do async startup in `setup`. Types reach `ctx.client.*` through the `plugins` augmentation — you DROP the `client` intersection entirely.

```ts
import { createClient } from '@redis/client';
import { createPlugin, definePlugins, type ParseClient, Client } from 'seyfert';

export function servicesPlugin() {
  const redis = createClient();
  const api = new Api(API_KEY, redis);              // build once; api reuses this redis
  return createPlugin({
    name: 'services',
    client: { redis: () => redis, api: () => api }, // sync → client.redis / client.api
    async setup(client) {                            // async init lives here, NOT in factories
      redis.on('error', e => client.logger.error('redis', e));
      await redis.connect();
      await api.warmup();
    },
    // teardown: () => redis.close(),                // close owned transports on client.close()
  });
}

export const plugins = definePlugins(servicesPlugin());
// new Client({ plugins });
declare module 'seyfert' {
  interface SeyfertRegistry {
    client: ParseClient<Client<true>>;   // NO { api; redis } intersection
    plugins: typeof plugins;             // <- THIS is what types ctx.client.api / .redis
  }
}
```

`ctx.client.api` / `ctx.client.redis` are now typed everywhere with zero `SeyfertRegistry.client` extra-prop augmentation. Closure-building `api` from `redis` sidesteps factory ordering; if instead a factory reads `client.redis`, declare `redis` **first** (factories install eagerly in insertion order — see Authoring).

### Feature plugin: typed middleware (global) + command + ctx helper

```ts
import { createPlugin, createMiddleware, PluginOrder } from 'seyfert';

// v5: stop() with no arg skips silently (replaces removed pass()); stop('reason') denies.
const balanceGuard = createMiddleware<void>(({ context, next, stop }) => {
  if (!context.guildId) return stop();                 // skip silently
  if (isBroke(context.author.id)) return stop('Insufficient balance');
  return next();
});

export const economyPlugin = createPlugin({
  name: 'economy',
  client: { economy: () => new EconomyService() },     // client.economy (typed)
  ctx: { wallet: (_i, client) => client.economy.wallet() }, // ctx.wallet
  middlewares: { balanceGuard },                       // names become a typed union
  globalMiddlewares: ['balanceGuard'],                 // run on every command
  register(api) {
    api.commands.add(BalanceCommand);
    // imperative alternative with ordering:
    // api.middlewares.add('balanceGuard', balanceGuard, { global: true, order: PluginOrder.Before });
  },
  setup: client => client.economy.connect(),
  teardown: client => client.economy.close(),
});
```

### Shared provider + consumer (capability, never touches client.*)

```ts
import { createPlugin, createSharedKey, type SharedValue } from 'seyfert';

class RateLimiter {
  private hits = new Map<string, number>();
  take(id: string, max = 5) { const n = (this.hits.get(id) ?? 0) + 1; this.hits.set(id, n); return n <= max; }
  reset() { this.hits.clear(); }
}
export const limiterKey = createSharedKey<RateLimiter>()('rate-limiter');

export const limiterPlugin = createPlugin({
  name: 'rate-limiter',
  version: '1.2.0',                                     // read by semver requires
  register(api) {
    api.shared.set(limiterKey, () => new RateLimiter(), { dispose: l => l.reset() });
  },
});

type Limiter = SharedValue<typeof limiterKey>;         // RateLimiter
export const featurePlugin = createPlugin({
  name: 'feature',
  imports: [limiterPlugin],                            // own the ordering
  requires: [{ capability: limiterKey }],             // fail fast if no limiter exists
  register(api) {
    api.commands.observe({
      onBeforeMiddlewares(ctx) {
        const limiter = ctx.client.shared.unwrap(limiterKey) as Limiter;
        if (!limiter.take(ctx.author.id)) api.diagnostics.warn('rate limit hit', { code: 'ratelimited' });
      },
    });
  },
});
```

### Observability plugin: events + hooks + REST observer + gateway intents

```ts
import { createPlugin, PluginOrder } from 'seyfert';

export const observabilityPlugin = createPlugin({
  name: 'observability',
  register(api) {
    api.gateway.addIntents('Guilds', 'GuildMembers');  // ensure listeners have intents

    api.events.on('guildCreate', guild => metrics.increment('guild.join', { id: guild.id }));

    api.hooks.on('commands:afterLoad', metadata => console.info(`loaded ${metadata.total} commands`));
    api.hooks.on('events:afterLoad', (client, dir) => client.logger.info(`events from ${dir}`)); // [client, dir]!

    const dispose = api.rest.observe({                 // stackable; no `route` field anywhere
      onRequest({ method, url }) { /* about to fire */ },
      onSuccess({ method, url, response }) { metrics.increment('rest.ok', { method, url, status: response.status }); },
      onFail({ method, url, error, statusCode }) { metrics.increment('rest.fail', { method, url, statusCode }); },
      onRatelimit({ method, url }) { metrics.increment('rest.ratelimit', { method, url }); },
    }, { order: PluginOrder.Before });
    // dispose() to unsubscribe early; otherwise cleaned up automatically.
  },
});
```

### Plugin-contributed i18n overlay (langs.contribute)

```ts
export const supportLangPlugin = createPlugin({
  name: 'support-lang',
  register(api) {
    // prefix is REQUIRED and namespaces the values; app reads ctx.t.support.<key>
    api.langs.contribute('en-US', { ticket: { open: 'Ticket opened', closed: 'Ticket closed' } }, { prefix: 'support' });
    api.langs.contribute('es-ES', { ticket: { open: 'Ticket abierto', closed: 'Ticket cerrado' } }, { prefix: 'support' });
  },
});
// in a command: const msg = ctx.t.support.ticket.open.get();   // ctx.t resolves via SeyfertLocale proxy
```

### imports — composing so types flow through

```ts
import { createPlugin } from 'seyfert';
import { loggerPlugin } from './logger-plugin';

const logger = loggerPlugin();
export const auditPlugin = createPlugin({
  name: 'audit',
  imports: [logger],                 // client.exampleLogger is typed inside setup below
  setup(client) { client.exampleLogger.info('audit ready'); },
});
// app: include BOTH so global types resolve → definePlugins(logger, auditPlugin)
```

## Diagnostics & errors

`client.plugins` is a `ResolvedPluginList`: a frozen array that also carries `.resolved` and `.diagnostics: readonly PluginDiagnostics[]`. Each `PluginDiagnostics` has `name`, `instanceId?`, `index`, `status`, `imports`, `clientKeys`, `ctxKeys`, count fields (`commands`/`components`/`modals`/`restObservers`/`commandObservers`/`autocompleteWrappers`/`gatewayIntents`/`gatewaySendPayloadWrappers`/`gatewayDispatchInterceptors`/`handlerCreators`/`handlerTransformers`/`anyEvents`/`eventErrors`), name lists (`events`/`middlewares`/`shared`/`cacheResources`/`langs`/`hooks`), `requirements` (with `satisfied`/`optional`), and `messages: PluginDiagnosticMessage[]` (where info/warn/error live — there is NO `warnings` field). Lists cap at 50; overflow in `truncated`.

```ts
for (const d of client.plugins.diagnostics) {
  client.logger.info(d.name, d.status);
  for (const m of d.messages) client.logger.warn(m.phase, m.severity, m.message);
}
// dev sweep: client.plugins.diagnostics.flatMap(d => d.messages)
```

Errors (`errors.ts`, both root-exported): `SeyfertPluginError` (fields `plugin`, `instanceId?`, `phase`, `index`, `cause`; default code `PLUGIN_FAILED`). `SeyfertPluginAggregateError` (`.errors: unknown[]`) thrown when setup+cleanup both fail, or reverse teardown collects multiple errors. When wrapping `client.start()`/`client.close()`, catch BOTH with `instanceof` (checking `error.name` misses the aggregate case).

## Official plugins (EXTERNAL — `@slipher/*`, doc-authoritative, VERSION-VERIFY)

None vendored in core Seyfert. Confirm package name/version/exports against installed typings. Adapters (cache/HTTP/REST/gateway) are NOT plugins — they go to their own client options, not `plugins`.

- `@slipher/cooldown` — `cooldown({ middleware?: true | { global?, name?, message? } })`. Decorators `@Cooldown.user|guild|channel|global|custom(interval, { uses?, group? }?)` or `@Cooldown({ type, interval, uses?, group? })`. Manager on `client.cooldown`/`ctx.cooldown`: zero-arg `consume()`/`check()` (in-handler only; throw outside) or explicit `{ name, target, guildId?, cost? }`; `reset(...)`. `check`/`consume` return `undefined` when the command has no cooldown. If `middleware` is an object, the name isn't inferred — augment `SeyfertRegistry.middlewares` with `CooldownMiddlewares<'cooldown'>`.

```ts
import { cooldown } from '@slipher/cooldown';
const plugins = definePlugins(cooldown({ middleware: true }));
// @Cooldown.user(5_000) on a command; const r = await ctx.cooldown.consume();
// if (r && !r.allowed) return ctx.write({ content: `Retry ${Formatter.timestamp(r.retryAfter)}` });
```

- `@slipher/scheduler` — `scheduler({ driver: memory() | persistent({...}), tasks: [...] })`; exposes `.registry` (same object as `ctx.scheduler`/`client.scheduler`). `@Cron(expr, { id })`, `@Interval(duration, { id, runImmediately? })`; imperative `registry.cron/interval/add/pause/resume/remove/list/on`. Always give a stable `id`. Persistent uses BullMQ/Redis; wire `client.close()` to SIGTERM/SIGINT.
- `@slipher/logger` — request-scoped `ctx.logger`; `.add(...)` enriches a wide event, level methods emit immediately; `useLogger()`. Sinks: console/Pino/event-log. Redaction is sink-level.
- `@slipher/queues` — `queues({ driver, processors: [...] })`; augment `RegisteredQueues`. `@Processor`+`@Process`(+`@OnWorkerEvent`/`@OnQueueEvent`); produce via `ctx.queues.get(name).add(...)`.
- `@slipher/chartjs` — utility, not a client plugin: `NapiChartjsCanvas({...})` + `renderToBuffer(config)` → attachment buffer.

Other official: adapters `@slipher/redis-adapter`, `@slipher/uws-adapter`, `@slipher/generic-adapter`, `@slipher/proxy`; utilities `yunaforseyfert`, `@slipher/webhooks`, `@slipher/watcher`, `@slipher/testing`.

## Review Checklist

- Installed via `definePlugins(...)` and `SeyfertRegistry.plugins` augmented? (else no global types)
- `client`/`ctx` factories synchronous; async work in `setup`, cleanup in `teardown`?
- Typed client keys declared in `client` (not assigned inside `setup`)?
- Shared services use `createSharedKey` + `api.shared.set`, not globals? Consumers use `requires: [{ capability }]`?
- `RegisteredPluginShared` augmentation maps name → VALUE type (`{ ledger: Ledger }`), not `typeof key`?
- Middleware uses `{ context, next, stop }` (no `pass`); `stop()` skips, `stop('reason')` denies?
- Ordering/validation via `imports`/`requires`/`api.has` — not an invented `dependsOn`/priority API? `api.has` only for `plugin:*` (capabilities via `shared.has`)?
- Hook names use the exact `SeyfertPluginHooks` keys (`commands:afterLoad`, not `commandsLoaded`); `events:afterLoad` is `[client, dir]`; `uploadCommands` via `api.events.on`?
- REST observer payloads use `{ client, method, url, request (+response/error/statusCode) }` — no `route` field?
- No mutation in `teardown` (api is read-only there)?
- `cache.resource`/`langs.contribute` name/prefix valid (they throw)?
- `client.close()` doesn't close gateway/REST/cache — owned transports closed explicitly?
- Catch BOTH `SeyfertPluginError` and `SeyfertPluginAggregateError` around start/close?
- Inspect `diagnostic.messages` (not `warnings`); watch `truncated` (50-item cap)?
- External `@slipher/*` versions/exports verified in the target project?
