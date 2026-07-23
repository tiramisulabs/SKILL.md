# Official Plugins (index)

Original source URL: https://seyfert-web-git-seyfert-v5-tiramisulabs.vercel.app/docs/plugins/official/index
Coverage reference: plugins.md
Verification status: Source-verified (core integration API) + official packages verified against current `tiramisulabs/extra`

## Contents

- [Page summary](#page-summary)
- [Catalog](#catalog)
- [Core plugin API](#key-apis-verified--core-integration-from-core-seyfert)
- [Examples](#code-examples-verified)
- [Source anchors](#source-anchors)

## Page Summary

This is an index/catalog page. It lists the official packages the Seyfert team maintains under the `@slipher` scope (plus `yunaforseyfert`), grouped into three categories:
- **Plugins** — installed through `new Client({ plugins })` and composed by the client.
- **Adapters** — swap a piece of infrastructure (cache, HTTP server, REST, gateway); passed to their own client options, NOT the `plugins` array.
- **Utilities** — helpers you import and call directly; no plugin lifecycle.

A plugin can use adapters and utilities internally — they only become "installed" when you pass a plugin object to the `plugins` option. None of these packages live in core Seyfert. In consumer work, verify the installed version. When maintaining this skill, verify official packages against current `tiramisulabs/extra/packages/*/src`, metadata, and tests. Core Seyfert remains authoritative for the integration shape.

## Catalog

Plugins (installed via `new Client({ plugins })`, each has its own guide):
- `@slipher/cooldown` — per-command rate limits by user, guild, channel, or globally.
- `@slipher/scheduler` — cron and interval jobs managed from the Seyfert lifecycle.
- `@slipher/logger` — structured, request-aware logging with pluggable backends.
- `@slipher/opentelemetry` — automatic traces and duration metrics for interactions, events, REST, and cache.
- `@slipher/queues` — background job queues with producers and processors.
- `yunaforseyfert` — advanced prefix/text command parser: named options, dynamic prefixes, watchers. (Guide: `/docs/plugins/official/yuna`.) `pnpm add yunaforseyfert`

Adapters (replace infrastructure — passed to cache/server/rest/gateway options, not `plugins`):
- `@slipher/redis-adapter` — distributed Redis-backed cache. `pnpm add @slipher/redis-adapter`
- `@slipher/uws-adapter` — HTTP server (uWebSockets.js). `pnpm add @slipher/uws-adapter`
- `@slipher/generic-adapter` — HTTP server for webhook-based bots. `pnpm add @slipher/generic-adapter`
- `@slipher/proxy` — REST proxy. `pnpm add @slipher/proxy`

Utilities (import and call directly):
- `@slipher/webhooks` — Discord webhook event handling. `pnpm add @slipher/webhooks`
- `@slipher/watcher` — hot-reload for development. `pnpm add -D @slipher/watcher`
- `@slipher/testing` — test fixtures/helpers for Seyfert apps. `pnpm add -D @slipher/testing`
- `@slipher/chartjs` — render Chart.js charts to images for attachments. `pnpm add @slipher/chartjs`

Always confirm the installed version and exact export surface against the package's own typings in a consuming project. Use each package guide for its current contract.

## Key APIs (verified — core integration, from core Seyfert)

All exported from root `seyfert` (src/index.ts -> src/client/index.ts:13 `export * from './plugins'`). The public helpers live in `src/client/plugins.ts`; the registry/api/types internals live under `src/client/plugins/`.

- `createPlugin(plugin)` — identity helper that returns the plugin object typed as a `SeyfertPlugin`. At runtime it literally `return plugin` (no validation); its whole job is inference of the generics `Name`, imports `I`, extension `E`, context `C`, middlewares `M`, optional `Meta`. src/client/plugins.ts:240-265
- `createPluginFactory({ defaults, validate?, factory })` — returns `(options?: Partial<O>) => P`; merges `{ ...defaults, ...options }`, runs optional `validate(resolved)`, then `factory(resolved)`; any throw is wrapped via `wrapPluginError('<createPluginFactory>', 'options', -1, error)`. This is how external packages expose `cooldown({...})`, `scheduler({...})`, etc. src/client/plugins.ts:267-281
- `definePlugins(...plugins | [plugins])` — identity helper returning the tuple of plugins for typing `new Client({ plugins })`. Accepts either a spread or a single array (`Array.isArray(plugins[0]) ? plugins[0] : plugins`). src/client/plugins.ts:283-289
- `SeyfertPlugin<E, C, I, M>` interface — the authored shape: required `name`; optional `instanceId`, `version`, `imports` (`I`), `requires` (`PluginRequirementInput[]`), `meta`, `client` (`PluginClientMap` — extends the client, typed), `ctx` (`PluginContextMap` — extends command/component context), `middlewares` (`M`), `globalMiddlewares` (`(keyof M & string)[]`); lifecycle hooks `options(current)`, `register(api)`, `setup(client, api?)` (Awaitable), `teardown(client, api?)` (Awaitable, receives the restricted `SeyfertPluginTeardownApi`). src/client/plugins/types.ts:492-512
- `SeyfertPluginApi<M, E>` — object passed to `register`/`setup`. Namespaces: `has(req)`; `events` (`on`/`once`/`onAny`/`onError`, each returns a `PluginDisposer`); `commands` (`add`/`remove`/`observe`/`defaults`); `rest.observe`; `hooks.on`; `handlers` (`construct`/`transform`); `components` & `modals` (`add`/`remove`/`defaults`); `middlewares` (`add(name, mw, opts?)`/`remove`); `autocomplete.wrap`; `gateway` (`addIntents`/`wrapSendPayload`/`onDispatch`); `cache.resource(name, ctor, opts?)`; `shared` (`set`/`remove`/`has`); `langs.contribute(locale, values, opts)`; `reload()`; `diagnostics.warn`; `options.set`. src/client/plugins/types.ts:373-477, src/client/plugins/api.ts
- `SeyfertPluginTeardownApi` — what `teardown` actually gets: `Pick<SeyfertPluginApi, 'has' | 'diagnostics'> & { shared: Pick<…'shared', 'has'> }`. You can read/diagnose but NOT mutate contributions during teardown. src/client/plugins/types.ts:479-481
- `SeyfertRegistry` (augmentable interface, starts empty `{}`) — declare `{ plugins: [...] }` (and `client`/`middlewares`/`langs`) to register plugin types globally so `client.plugins`, shared keys, and context/client extensions resolve. src/client/plugins/types.ts:32
- `PluginOrder` enum (`Before` / `After`) and `PluginOrderOpt` (`PluginOrder.Before | PluginOrder.After | number`) for ordering contributions via each method's `opts.order`. src/client/plugins/types.ts:69-74
- `PluginMiddlewareOptions` — `{ override?, global?, order? }`. `{ global: true }` registers the middleware as a global middleware. src/client/plugins/types.ts:354-357

## Code Examples (verified)

### 1. Minimal plugin: events + setup
```ts
import { Client, createPlugin, definePlugins } from 'seyfert';

const heartbeat = createPlugin({
  name: 'heartbeat',
  register(api) {
    api.events.on('messageCreate', message => {
      if (message.content === '!ping') message.reply({ content: 'pong' });
    });
  },
  async setup(client) {
    client.logger.info('heartbeat plugin ready');
  },
});

const client = new Client({ plugins: definePlugins(heartbeat) });
```

### 2. Reusable feature: typed client extension + middleware + lifecycle
The headline v5 pattern. `client.economy` is real and inferred — no `(client as any)`.
```ts
import { Client, createPlugin, definePlugins, createMiddleware } from 'seyfert';

class EconomyService {
  connect() {/* open pool */}
  close() {/* close pool */}
  async balance(userId: string) { return 0; }
}

const balanceGuard = createMiddleware<void>(({ next, stop }) => {
  // stop() with no args = skip silently (v5: pass() was removed)
  return next();
});

const economy = createPlugin({
  name: 'economy',
  client: { economy: () => new EconomyService() }, // typed extension
  middlewares: { balanceGuard },                    // declare names for inference
  register(api) {
    api.middlewares.add('balanceGuard', balanceGuard, { global: true });
  },
  setup: client => client.economy.connect(),
  teardown: client => client.economy.close(),       // cleanup that actually runs
});

const client = new Client({ plugins: definePlugins(economy) });
// register the extension type globally so client.economy type-checks elsewhere:
declare module 'seyfert' {
  interface SeyfertRegistry { plugins: [typeof economy] }
}
```

### 3. Configurable plugin via createPluginFactory
How external packages (cooldown, scheduler…) are built — and how to write your own configurable one.
```ts
import { createPlugin, createPluginFactory } from 'seyfert';

interface MetricsOptions { prefix: string; sampleRate: number }

export const metrics = createPluginFactory<MetricsOptions, ReturnType<typeof build>>({
  defaults: { prefix: 'bot', sampleRate: 1 },
  validate(opts) {
    if (opts.sampleRate <= 0 || opts.sampleRate > 1) throw new Error('sampleRate must be (0,1]');
  },
  factory: build,
});

function build(opts: MetricsOptions) {
  return createPlugin({
    name: 'metrics',
    register(api) {
      // isolated, stackable REST observer (v5: replaces overwriting onSuccessRequest)
      api.rest.observe({
        onSuccess({ method, url, response }) {
          if (Math.random() <= opts.sampleRate)
            console.log(`${opts.prefix}.rest.success`, method, url, response.status);
        },
        onFail({ method, error }) { console.error(`${opts.prefix}.rest.fail`, method, error); },
      });
    },
  });
}

// usage: new Client({ plugins: definePlugins(metrics({ sampleRate: 0.25 })) })
```

### 4. Composing official packages with your own
```ts
import { Client, definePlugins } from 'seyfert';
import { cooldown } from '@slipher/cooldown';
import { memory, scheduler } from '@slipher/scheduler';

const client = new Client({
  plugins: definePlugins(
    cooldown({ middleware: true }),
    scheduler({ driver: memory() }),
    economy, // your own from example 2
  ),
});
```
The installed package version remains authoritative in consumer projects.

### 5. Adapter (NOT a plugin) — passed to its own option
```ts
import { Client } from 'seyfert';
import { RedisAdapter } from '@slipher/redis-adapter'; // external — verify

const client = new Client();
client.setServices({
  cache: { adapter: new RedisAdapter({ /* redis opts */ }) },
});
// Adapters go to cache/rest/gateway/server options — never the `plugins` array.
```

### 6. Plugin contributing cache, langs, gateway intents
```ts
import { createPlugin } from 'seyfert';

const presencePlugin = createPlugin({
  name: 'presence-watch',
  register(api) {
    api.gateway.addIntents('GuildPresences');          // string intents (v5)
    api.langs.contribute('en-US', { hi: 'Hello' }, { prefix: 'presence' }); // prefix REQUIRED (non-empty)
    api.diagnostics.warn('presence plugin loaded in dev', { phase: 'register' });
  },
});
```

## Common patterns / gotchas

- `createPlugin`/`definePlugins` are pure identity functions — NO runtime validation. Typos surface as TS type errors, not throws. Misordered fields just won't infer.
- Register the plugin types in `SeyfertRegistry.plugins` (`declare module 'seyfert'`) or `client.economy` / shared keys / ctx extensions won't type-check at call sites outside the plugin.
- `teardown` gets the restricted `SeyfertPluginTeardownApi` (only `has`, `diagnostics`, `shared.has`). Trying to mutate contributions during teardown THROWS ("cannot mutate plugin contributions during teardown"). src/client/plugins/api.ts:99-105
- `cache.resource(name, …)` THROWS on a reserved cache resource name and on a duplicate already registered by another plugin. src/client/plugins/api.ts:455-462
- `langs.contribute(locale, values, opts)` requires `opts.prefix` with at least one non-empty `.`-segment, else it THROWS. src/client/plugins/api.ts:524-529, 610-612
- Middlewares: in v5 the callback arg is `{ context, next, stop }` — `pass()` is gone. `stop()`/`stop(null)` = silent skip; `stop('reason')` = deny -> `onMiddlewaresError`. Register a plugin middleware as global with `api.middlewares.add(name, mw, { global: true })`.
- REST instrumentation: use `api.rest.observe({...})` (stackable, isolated) or `client.rest.observe({...})` from app code — do NOT overwrite `onSuccessRequest` (v4 pattern, last-write-wins).
- Ordering: every contribution method accepts `opts.order` (`PluginOrder.Before | PluginOrder.After | number`); event/observe/dispatch methods return a `PluginDisposer` you can call to unsubscribe.
- Adapters vs plugins: adapters replace infrastructure and go to `setServices`/cache/server/rest/gateway options; only objects authored with `createPlugin` belong in `plugins`.

## Doc vs Source Corrections

- Catalog grouping fixed to match current MDX (ref seyfert-v5): `yunaforseyfert` is listed under **Plugins** (with a `/yuna` guide link), not Utilities. Utilities are now `@slipher/webhooks`, `@slipher/watcher`, `@slipher/testing`, `@slipher/chartjs`.
- All core integration APIs and line anchors re-verified against `./src` on the authoritative Seyfert source — no stale/v4 signatures found. `createPlugin` runtime is `return plugin` (240-265); `createPluginFactory` wraps throws via `wrapPluginError` (267-281); `definePlugins` accepts spread-or-array (283-289).
- `SeyfertRegistry` starts as empty `interface SeyfertRegistry {}` (types.ts:32) and is the single augmentation point in v5 (replaces v4 `UsingClient`/`RegisteredMiddlewares`/`DefaultLocale`, which are now derived).
- Official `@slipher` package contracts are verified against current `tiramisulabs/extra`; core Seyfert cannot verify their package-specific exports because none are vendored there.

## Source Anchors

- src/client/plugins/{errors,order,registry,shared}.ts (internals)

## Agent Guidance

- Treat this page as a directory: for "what official plugins exist" answer from the catalog; for any specific package defer to its own guide and the installed typings (versions drift — always `verify version in target project`).
