# Setup & Runtime

Source-verified against the target project installed `seyfert` package or provided Seyfert source (`seyfert@5.0.0` where applicable). Covers config, the three clients, `start()`, `uploadCommands()`, `setServices`, intents, events, sharding/workers, presence, plugins, logger, and module augmentation — with copy-paste recipes.

## Import policy

Most setup imports come from the package root `'seyfert'`. Use deep imports only for APIs not root-exported in the installed package.
Root barrel (`src/index.ts`) exports: `Client`, `HttpClient`, `WorkerClient`, `config`, `createEvent`, `extendContext`, `createPlugin`, `definePlugins`, `ShardManager`, `WorkerManager`, `WorkerAdapter`, `Logger`, `GatewayIntentBits`, `ActivityType`, `PresenceUpdateStatus`, `SeyfertError`, plus types `ParseClient`, `ParseLocales`, `ParseGlobalMiddlewares`, `UsingClient`, `ShardData`, `ShardManagerOptions`, `WorkerData`, `WorkerManagerOptions`, `WorkerInfo`, `WorkerShardInfo`, `BotConfig`, `HttpConfig`. Deep-import `LogLevels`/`LoggerOptions` from `seyfert/lib/common`, and internals like `GatewayIntentInput`/`resolveGatewayIntents` from `seyfert/lib/client/intents`.

## tsconfig (decorators are mandatory)

```json
{
  "compilerOptions": {
    "experimentalDecorators": true,
    "emitDecoratorMetadata": true,
    "target": "ESNext",
    "module": "NodeNext"
  }
}
```

`@Declare`, `@Options`, `@Middlewares` are decorators — without these two flags commands won't load. For Cloudflare/edge use `"module": "ESNext"`, `"moduleResolution": "Bundler"`.

## Runtime config (`seyfert.config.*`)

Two factory helpers on the `config` object (`src/index.ts:76-103`). `getRC()` resolves `seyfert.config` with any of `.js .mjs .cjs .ts .mts .cts` from `process.cwd()`, reads its `default` export, and caches the result (`src/client/base.ts:1188-1239`).

```ts
// seyfert.config.mjs — Gateway bot
import { config } from 'seyfert';

export default config.bot({
  token: process.env.BOT_TOKEN ?? '',          // required (INVALID_TOKEN if empty)
  intents: ['Guilds', 'GuildMessages'],        // string names | number | number[]
  locations: {
    base: 'dist',                              // 'src' under bun/tsx; 'dist' after tsc
    commands: 'commands',
    events: 'events',                          // gateway only
    components: 'components',
    langs: 'langs',
  },
  // optional: applicationId, publicKey, port, debug
});
```

```ts
// seyfert.config.mjs — HTTP-interactions bot
import { config } from 'seyfert';

export default config.http({
  token: process.env.BOT_TOKEN ?? '',
  applicationId: process.env.BOT_APP_ID ?? '',  // REQUIRED for http
  publicKey: process.env.BOT_PUBLIC_KEY ?? '',  // REQUIRED for http
  port: 3000,                                    // defaults to 8080
  locations: { base: 'dist', commands: 'commands' /* NO `events` for http */ },
});
```

- `config.bot` returns `InternalRuntimeConfig` (alias `BotConfig`) with `intents` already resolved to a number via `resolveGatewayIntents`. `config.http` (alias `HttpConfig`) defaults `port: 8080`, omits `intents`, requires `publicKey`+`applicationId`, and — inside a Cloudflare Worker — stashes the config on `BaseClient._seyfertCfWorkerConfig` so `getRC()` never touches the filesystem (`src/index.ts:100`).
- Only `locations.base` is required; other location keys join to `process.cwd()/<base>/<value>` at load time.
- Errors at `start()` time: missing file → `NO_SEYFERT_CONFIG`; file found but import threw → `SEYFERT_CONFIG_LOAD_ERROR` (original error as `cause`); empty/bad token → `INVALID_TOKEN`. Narrow with `SeyfertError.is(err, 'NO_SEYFERT_CONFIG')`.
- Skip the file entirely (tests, monorepos) by passing `getRC` in client options — see recipe below.

## The three clients

All extend `BaseClient` (`src/client/base.ts`). Pick the one matching the deployment:

- **`Client<Ready>`** (`src/client/client.ts`) — gateway bot. Owns `gateway: ShardManager`, `events: EventHandler`, `collectors`, `me`, `latency`. `start()` connects shards and loads events.
- **`HttpClient`** (`src/client/httpclient.ts`) — pure HTTP-interactions endpoint. No gateway, no gateway events. `start()` loads handlers then calls `execute(httpConnection)`. Needs an external adapter (`@slipher/uws-adapter` / `@slipher/generic-adapter`) to receive requests.
- **`WorkerClient<Ready>`** (`src/client/workerclient.ts`) — runs inside a worker spawned by `WorkerManager`. Owns `shards: Map<number, Shard>` (NO `.gateway`), `events`, `latency`, `tellWorker`/`tellWorkers`.

```ts
// src/index.ts — minimal gateway bot
import { Client, type ParseClient } from 'seyfert';

const client = new Client();
client.start().then(() => client.uploadCommands({ cachePath: './commands.json' }));

declare module 'seyfert' {
  interface SeyfertRegistry {
    client: ParseClient<Client<true>>;   // REQUIRED — see Module augmentation
  }
}
```

`Client` is a value typed `ClientConstructor`; `new Client<true>()` and `new Client<[typeof plugin]>()` both work via overloads.

## `start()` does NOT upload commands

`start()` (`src/client/base.ts:374` + `client.ts:145`) validates the token, sets up plugins, starts the cache adapter, loads langs → commands → components (and, for gateway, events) from disk, resolves intents, then connects the gateway. It NEVER calls Discord's command API. Register explicitly with `client.uploadCommands({ applicationId?, cachePath? })` (`src/client/base.ts:1056`) — commonly inside a `botReady` event. With `cachePath` it diffs a JSON snapshot and uploads only on change; without it, it always uploads.

`start()` overrides: `token`, `commandsDir`, `componentsDir`, `langsDir`, `connection.intents` (gateway), `eventsDir` (gateway/worker), `httpConnection` (http). `Client.start(options, execute = true)` — pass `execute = false` to build the `ShardManager` without spawning shards.

Startup order for `Client`: token/REST → plugin setup + hooks → `cache.adapter.start()` → langs → commands → components → events → resolve intents → `ShardManager.spawnShards()` (unless watcher mode or `execute=false`).

## `setServices(...)`

Configure/replace core services before `start()` (`src/client/base.ts:309`, `ServicesOptions` at `1388`):

```ts
client.setServices({
  rest,                                  // custom ApiHandler
  cache: { adapter, disabledCache },     // adapter + bool | DisabledCache | (key)=>bool
  langs: { default: 'en-US', preferGuildLocale: true, aliases: { en: ['en-US'] } },
  middlewares,                           // ClientMiddlewares map (merged, not replaced)
  handleCommand,                         // CLASS, not instance: new (client) => HandleCommand
});
```

`Client.setServices` also accepts `gateway?: ShardManager` to inject a pre-built shard manager (`src/client/client.ts:63`). `WorkerClient.setServices` wires `WorkerAdapter.postMessage` when `options.postMessage` is set. `preferGuildLocale: true` flips `ctx.t` to resolve the guild's locale before the user's.

## Intents

`GatewayIntentInput = number | IntentStrings | number[]` (`src/client/intents.ts`). Set in `seyfert.config` (`config.bot`) or override at `start({ connection: { intents } })` (the override wins). Resolution happens once at config-load via `resolveGatewayIntents`.

```ts
intents: ['Guilds']                                                     // slash-only
intents: ['Guilds', 'GuildMessages', 'MessageContent']                 // prefix commands
intents: ['Guilds', 'GuildMessages', 'GuildMembers', 'GuildModeration']// moderation
intents: ['Guilds', 'GuildVoiceStates']                                // music
intents: GatewayIntentBits.NonPrivilaged                               // all non-privileged
intents: GatewayIntentBits.NonPrivilaged | GatewayIntentBits.OnlyPrivilaged // truly everything
```

- Enum helpers are spelled WITHOUT the "e": `NonPrivilaged` (53575421), `OnlyPrivilaged` (33026 = members|presences|messageContent). `NonPrivileged` will NOT compile.
- Privileged intents (`GuildMembers`, `GuildPresences`, `MessageContent`) must ALSO be enabled in the Developer Portal or the gateway rejects the connection.
- `message.content` is EMPTY without `MessageContent` (privileged) — the classic blank-prefix-message bug.
- The cache gates what it stores by intent (`hasIntent`) — missing cache entries are often a missing-intent problem. `config.http` ignores intents entirely.
- Do NOT assume `32767` is "all intents" — it omits bit 15 (`MessageContent`) and everything above it.

Per-environment override:

```ts
const intents = process.env.NODE_ENV === 'development'
  ? GatewayIntentBits.NonPrivilaged
  : (['Guilds', 'GuildMessages'] as const);
await client.start({ connection: { intents } }); // overrides seyfert.config
```

## Events

```ts
// src/events/botReady.ts
import { createEvent } from 'seyfert';

export default createEvent({
  data: { name: 'botReady', once: true },
  run(user, client, shardId) {            // gateway events keep shardId
    client.logger.info(`Ready as ${user.username} (shard #${shardId})`);
    client.uploadCommands({ cachePath: './commands.json' }); // good place to publish
  },
});
```

- `once` defaults to `false`. `run` is typed `Awaitable<unknown>` (widened from `void` so a handler may `return` a value, e.g. `return ctx.write(...)`; dev builds ≥ `28477111245`).
- Gateway handlers receive `(payload, client, shardId)`; CUSTOM (non-gateway) handlers receive `(...yourArgs, client)` — NO trailing `shardId`.
- One handler per event `name` — a second file with the same name OVERWRITES the first. Combine logic instead.
- Some payloads are TUPLES: `voiceStateUpdate` → `[state, old?]`, `voiceChannelStatusUpdate` → `[status, channel?]` (channel resolved). Destructure the first param.
- Events load only on gateway `Client`/`WorkerClient` — `HttpClient` has no gateway dispatch. Custom events (`client.events.emit`/`runCustom`) still work everywhere.
- Built-in custom events need no augmentation: `commandsLoaded`, `componentsLoaded`, `uploadCommands`. Declare your own via `interface CustomEvents` augmentation; fire with `client.events.emit(name, ...args)` (the client is appended automatically).
- A default-exported ARRAY of `createEvent(...)` is flattened on load.

## Presence (gateway `Client` only)

Static per-shard via `ClientOptions.presence`; worker variant takes `(shardId, workerId)`.

```ts
import { Client, ActivityType, PresenceUpdateStatus } from 'seyfert';

const client = new Client({
  presence: (shardId) => ({
    status: PresenceUpdateStatus.Online,
    activities: [{ name: `shard ${shardId}`, type: ActivityType.Watching }],
    since: null,
    afk: false,
  }),
});
```

- ALL FOUR fields are REQUIRED: `since` (`number | null`), `activities`, `status`, `afk`. Activity objects only honor `name`, `state`, `type`, `url` (a `Pick`) — `details`/`timestamps`/`emoji` are receive-only.
- `ActivityType.Custom` displays the `state` field (`name` still required). `ActivityType.Streaming` uses `url` (Twitch/YouTube) for the Live badge.
- Runtime updates: `client.gateway.setPresence(payload)` (all shards) or `client.gateway.setShardPresence(shardId, payload)` (one). Gateway-rate-limited — rotate on a `>=15s` timer.
- `HttpClient` has no `gateway` → no presence.

Rotating presence:

```ts
// src/events/botReady.ts
import { createEvent, ActivityType, PresenceUpdateStatus } from 'seyfert';

const rotation = [
  { name: 'with Seyfert', type: ActivityType.Playing },
  { name: 'your commands', type: ActivityType.Listening },
] as const;

export default createEvent({
  data: { name: 'botReady', once: true },
  run(_user, client) {
    let i = 0;
    setInterval(() => {
      client.gateway.setPresence({
        status: PresenceUpdateStatus.Online,
        activities: [rotation[i++ % rotation.length]],
        since: null,
        afk: false,
      });
    }, 30_000);
  },
});
```

## Sharding & workers

`Client` shards internally by default (`client.gateway` is a `ShardManager`). Reach for `WorkerManager` only for parallelism across threads/processes.

`WorkerManagerOptions.mode` is a discriminated union (`src/websocket/discord/shared.ts:99-114`):
- `'threads'` (DEFAULT) — `path` required; `node:worker_threads`.
- `'clusters'` — note PLURAL; `path` required; `node:cluster` processes. `'cluster'` is invalid.
- `'custom'` — `adapter: CustomManagerAdapter` required; `path` optional.

Other options: `workers?`, `shardsPerWorker?` (default 16), `totalShards?`, `shardStart?`/`shardEnd?`, `heartbeaterInterval?` (default 15000), `presence?`, `resharding?`, `getRC?`, `token`, `intents`.

```ts
// manager.ts (main process)
import { WorkerManager } from 'seyfert';

const manager = new WorkerManager({
  mode: 'threads',
  path: './dist/client.js',  // COMPILED entry for Node; TS source for bun/tsx
});
manager.start();
```

```ts
// client.ts (each worker)
import { WorkerClient, WorkerAdapter, type ParseClient } from 'seyfert';

const client = new WorkerClient();
client.setServices({ cache: { adapter: new WorkerAdapter(client.workerData) } }); // shared cache
await client.start();
await client.uploadCommands();

declare module 'seyfert' {
  interface SeyfertRegistry { client: ParseClient<WorkerClient<true>> }
}
```

- For shared cache: put `WorkerAdapter` on every worker AND set the real backend on the manager: `manager.setCache(new RedisAdapter(...))` (default is `MemoryAdapter`). The `WorkerAdapter` reaches the manager via `parentPort`/`process.send` automatically; 60s timeout → `CACHE_TIMEOUT`.
- Worker helpers: `client.workerId`, `client.workerData`, `client.calculateShardId(guildId)`, `client.tellWorker(id, (worker, vars) => ..., vars)`, `client.tellWorkers(fn, vars)`. `tellWorker*` serialize the fn via `.toString()` + `eval` in the target — closures do NOT transfer; pass everything through `vars` (JSON-serialized); 60s timeout → `WORKER_TIMEOUT`.
- Inspect shards: on `Client` use `client.gateway.latency` / `client.gateway.get(id)?.ping()` / `client.gateway.totalShards`; on `WorkerClient` use `client.latency` / `client.shards.get(id)?.ping()` (NO `.gateway`).
- `onShardDisconnect`/`onShardReconnect` callbacks are DEPRECATED — listen to `SHARD_DISCONNECT` (`{shardId, code, reason}`) / `SHARD_RECONNECT` (`{shardId}`) events instead (they fire on both `Client` and `WorkerClient`).
- `WorkerClientOptions.sendPayloadToParent` defaults to `false`; set `true` only if the manager must observe raw gateway packets.

## Module augmentation

Keep ONE `declare module 'seyfert'` block in a file inside `tsconfig` include scope (commonly `src/index.ts`). If it lives outside scope, every type silently falls back to its default (`BaseClient` / `{}` / `false`).

```ts
import type { ParseClient, ParseLocales, ParseGlobalMiddlewares, Client } from 'seyfert';
import type * as allMiddlewares from './middlewares';
import type * as globalMiddlewares from './globalMiddlewares';
import type * as defaultLang from './langs/en';

declare module 'seyfert' {
  interface SeyfertRegistry {
    client: ParseClient<Client<true>>;        // or ParseClient<HttpClient> / WorkerClient<true>
    middlewares: typeof allMiddlewares;        // Record<string, AnyMiddlewareContext>
    langs: ParseLocales<typeof defaultLang>;   // enables typed ctx.t.<key>
    plugins: [typeof myPlugin];                // optional — list plugins you pass the client
  }
  interface GlobalMetadata extends ParseGlobalMiddlewares<typeof globalMiddlewares> {}
  interface ExtraProps { category?: string; onlyForAdmins?: boolean }   // @Declare({ props })
  interface ExtendedRC { developers: string[] }                         // read via getRC()
}
```

- `SeyfertRegistry` UNIFIES `client`/`middlewares`/`langs`/`plugins` (`src/client/plugins/types.ts:32`). `UsingClient`, `RegisteredMiddlewares`, `DefaultLocale` are DERIVED — do NOT augment them directly (the v4 / early-v5 `interface UsingClient extends ...` pattern silently fails to merge).
- `ParseMiddlewares` is GONE — register the bare `typeof middlewares` (`createMiddleware` already returns the right shape).
- ALWAYS brand the client with `ParseClient<...>`; a bare `Client<true>` won't resolve `UsingClient` (it falls back to `BaseClient`).
- Still-separate augmentable interfaces: `GlobalMetadata`, `ExtendContext`, `ExtraProps`, `InternalOptions`, `ExtendedRC` / `ExtendedRCLocations`, `CustomEvents`, `CustomStructures`. `Cache` is a runtime class, not a `SeyfertRegistry` key; add an explicit `interface Cache { ... }` augmentation or use an access-site cast for custom cache resources.

```ts
declare module 'seyfert' {
  interface InternalOptions { withPrefix: true; asyncCache: false }
}
```

`withPrefix: true` makes both `ctx.interaction` and `ctx.message` optional (text commands). `asyncCache: true` when the adapter returns promises (e.g. `@slipher/redis-adapter`) — flips `ReturnCache<T>` to `Promise<T>` so you `await` cache reads.

## Recipe: full command with options + middleware

```ts
// src/middlewares/index.ts
import { createMiddleware } from 'seyfert';

export const ownerOnly = createMiddleware<void>(({ context, next, stop }) => {
  const owners = ['123456789012345678'];
  if (!owners.includes(context.author.id)) return stop('You are not the owner.'); // deny
  return next();
});

export const middlewares = { ownerOnly };
```

```ts
// src/commands/echo.ts
import {
  Command, Declare, Options, Middlewares,
  createStringOption, createBooleanOption, MessageFlags,
  type CommandContext,
} from 'seyfert';

const options = {
  // keys MUST be lowercase (compile-time enforced in v5)
  text: createStringOption({ description: 'What to echo', required: true }),
  hide: createBooleanOption({ description: 'Reply ephemerally' }),
  tone: createStringOption({
    description: 'Tone',
    choices: [
      { name: 'Loud', value: 'loud' },
      { name: 'Quiet', value: 'quiet' },
    ] as const, // choices is readonly → `as const` narrows the value union
  }),
};

@Declare({ name: 'echo', description: 'Echo your text' })
@Middlewares(['ownerOnly'])  // readonly tuple of registered names
@Options(options)
export default class Echo extends Command {
  async run(ctx: CommandContext<typeof options>) {
    const content = ctx.options.tone === 'loud'
      ? ctx.options.text.toUpperCase()
      : ctx.options.text;
    await ctx.write({
      content,
      flags: ctx.options.hide ? MessageFlags.Ephemeral : undefined,
    });
  }
}
```

- Middleware callback arg is `{ context, next, stop }` — there is no `pass()`. `stop()` (no arg) skips silently; `stop('reason')` denies and routes to `onMiddlewaresError`.
- `write`/`editOrReply`/`deferReply` return `void` unless you pass the response flag (`true`) — only then do you get a Message/webhook back.
- Defer for slow work: `await ctx.deferReply(true)` (ephemeral), then `await ctx.editOrReply({ content })`.

## Recipe: i18n setup

```ts
// src/langs/en.ts — base lang module (default export drives the typed shape)
export default {
  hello: (name: string) => `Hello, ${name}!`,
  ping: { result: (ms: number) => `Latency: ${ms}ms` },
};
```

```ts
// in seyfert.config: locations.langs = 'langs'; setServices default locale:
client.setServices({ langs: { default: 'en-US', preferGuildLocale: true } });

// in a command — ctx.t is a SeyfertLocale proxy; .get(locale?) resolves strings
const t = ctx.t.get(ctx.interaction?.locale);
await ctx.write({ content: t.hello(ctx.author.username) });
```

Register `langs: ParseLocales<typeof defaultLang>` in `SeyfertRegistry` so `ctx.t.<key>` is typed. On no-`fs` runtimes register manually: `client.langs.set([{ name: 'en', file: EnLang }])`.

## Recipe: a plugin

```ts
// plugins.ts
import { createPlugin, createMiddleware } from 'seyfert';

class EconomyService { connect() {} close() {} balance(_id: string) { return 0; } }
const balanceGuard = createMiddleware<void>(({ next }) => next());

export const economyPlugin = createPlugin({
  name: 'economy',
  client: { economy: () => new EconomyService() },     // typed client.economy
  register(api) {
    api.middlewares.add('balanceGuard', balanceGuard, { global: true });
  },
  setup: client => client.economy.connect(),           // runs on start
  teardown: client => client.economy.close(),          // runs on shutdown
});
```

```ts
// index.ts
import { Client, definePlugins, type ParseClient } from 'seyfert';
import { economyPlugin } from './plugins';

const client = new Client({ plugins: definePlugins(economyPlugin) });

declare module 'seyfert' {
  interface SeyfertRegistry {
    client: ParseClient<Client<true>>;
    plugins: [typeof economyPlugin];   // makes client.economy / ctx.economy typed everywhere
  }
}
```

Plugin input fields (`src/client/plugins/types.ts:498-511`): `name`, `client`, `ctx`, `middlewares`, `register(api)`, `setup(client, api?)`, `teardown(client, api?)`, plus `imports`, `requires`, `version`, `meta`. The full lifecycle/ordering/diagnostics live in the dedicated plugins reference.

## Recipe: extendContext + custom logger

```ts
import { Client, extendContext } from 'seyfert';
import { LogLevels } from 'seyfert/lib/common';

const context = extendContext(interaction => ({
  isOwner: interaction.user.id === process.env.OWNER_ID,
}));

const client = new Client({
  context,
  logger: { active: true, logLevel: LogLevels.Debug }, // honored on worker clients too
});
// langs is a setServices option, NOT a Client constructor option:
client.setServices({ langs: { preferGuildLocale: true } });
client.start();

declare module 'seyfert' {
  interface ExtendContext extends ReturnType<typeof context> {} // ctx.isOwner typed
}
```

## Recipe: programmatic config (no `seyfert.config` file)

```ts
import { Client, config } from 'seyfert';

const client = new Client({
  getRC: () => config.bot({
    token: process.env.BOT_TOKEN ?? '',
    locations: { base: 'dist', commands: 'commands' },
    intents: ['Guilds', 'GuildMessages', 'MessageContent'],
  }),
});
client.start();
```

## Recipe: HTTP / Cloudflare (no filesystem)

```ts
import '../seyfert.config.mjs';            // side-effect: must run BEFORE getRC()
import { HttpClient, type ParseClient } from 'seyfert';
import { GenericAdapter } from '@slipher/generic-adapter'; // EXTERNAL
import Ping from './commands/ping.js';

const client = new HttpClient();
const adapter = new GenericAdapter(client);

const ready = client.start().then(() => {
  client.commands.set([Ping]);            // register manually — no fs auto-load
  client.langs.set([{ name: 'en', file: EnLang }]);
  client.components.set([ButtonC]);
}).then(() => adapter.start());

export default { async fetch(req: Request) { await ready; return adapter.fetch(req); } };

declare module 'seyfert' { interface SeyfertRegistry { client: ParseClient<HttpClient> } }
```

Emit ESM (`module: "ESNext"`). Set secrets with `wrangler secret put`. `uploadCommands()` is a one-off deploy step (run a local Node script), not part of the request path. `reload()`/`langs.reload()` throw `RELOAD_NOT_SUPPORTED` on edge.

## Hot reload (dev)

`client.commands.reload(name | Command)` / `client.events.reload(name)` / `client.langs.reload(locale)` / `client.components.reload(filePath)`; each has `reloadAll(stopIfFail = true)` — pass `false` to skip broken files. `components.reload` is keyed by FILE PATH, the rest by name/locale. Reload mutates live registries (gate it owner-only) and does NOT re-upload to Discord. Throws `RELOAD_NOT_SUPPORTED` on Cloudflare. Auto file-watching uses the external `@slipher/watcher`.

## Gotchas

- `start()` never uploads commands — call `uploadCommands()` separately (commonly in `botReady`); use `cachePath` to skip unchanged uploads. Avoid uploading on every boot in prod (rate limits).
- Augment `SeyfertRegistry`, never `UsingClient`/`RegisteredMiddlewares`/`DefaultLocale` (derived). Always brand with `ParseClient<...>`. `ParseMiddlewares` removed.
- `Logger` is root-exported, but `LogLevels` is not; import it from `seyfert/lib/common`.
- `mode: 'clusters'` is PLURAL. `path` required for `threads`/`clusters`; `adapter` required for `custom`.
- `handleCommand` in `setServices` takes the CLASS, not an instance.
- `client.gateway` exists only on gateway `Client` (undefined on `WorkerClient`, absent on `HttpClient`). Presence/runtime shard APIs live there.
- Presence is gateway-only and requires all four fields; activities are stripped to `name|state|type|url`.
- Intent helper names omit the "e": `NonPrivilaged`/`OnlyPrivilaged`. `MessageContent` privileged + portal toggle required for `message.content`.
- `tellWorker`/`tellWorkers` pass data only via `vars` (function is serialized + eval'd); 60s timeout.
- `locations.base` must match what you ship — `dist` after `tsc`, `src` under bun/tsx. Wrong base = "starts but no commands".
- Keep `token`/`publicKey`/`applicationId` in env/secrets; never bake into image layers or commits.

## Review checklist

- [ ] Imports from `'seyfert'` root only (no deep imports for setup).
- [ ] Correct client class for the deployment (gateway/http/worker).
- [ ] `seyfert.config` has required fields (`token`; http also `applicationId`+`publicKey`); `locations.base` points at the right dir (`dist` after build, `src` under bun/tsx).
- [ ] `SeyfertRegistry.client` registered with `ParseClient<...>`; augmentation file in tsconfig scope; `middlewares`/`langs`/`plugins` keys added when used.
- [ ] `experimentalDecorators` + `emitDecoratorMetadata` enabled (ESM module for edge).
- [ ] `uploadCommands()` called explicitly (not assumed inside `start()`); `cachePath` used in prod.
- [ ] Intents include privileged ones only if enabled in the Developer Portal; `MessageContent` present for prefix bots.
- [ ] `mode: 'clusters'` (plural) for process clusters; `path` set for threads/clusters, `adapter` for custom.
- [ ] Presence payload has all four fields; runtime presence only on gateway `Client`.
- [ ] Middlewares use `stop()`/`stop('reason')` (no `pass()`); `@Middlewares` names are registered.
- [ ] `handleCommand` passed as a class; cache adapter matches `asyncCache` setting.
