# Getting Started: Introduction & Installation

Original source URL: https://seyfert-web-git-seyfert-v5-tiramisulabs.vercel.app/docs/learn/getting-started/index
Coverage reference: setup-runtime.md
Verification status: Source-verified (seyfert-core, branch more-qol)

## Page Summary
The upstream page is a short marketing/installation intro: prerequisites (Node v18+, TypeScript v4.9+, an npm-like package manager), `npm add seyfert`, and the `@dev` github-mirror tag. It also mentions Deno/Bun support "from version 2.2.0" (stale lore — see Corrections). It carries no real API. This note keeps the install facts and grounds them in the actual `package.json` and the root exports / runtime entry points an agent needs when scaffolding a Seyfert v5 project, then adds copy-paste "first bot" recipes.

## Key APIs (verified)
- Package name `seyfert`, version `5.0.0` — `package.json`. Root import specifier is `'seyfert'`; deep imports use `'seyfert/lib/...'`.
- `config.bot(data: RuntimeConfig)` and `config.http(data: RuntimeConfigHTTP)` — `src/index.ts:76`. `config.bot` resolves `intents` via `resolveGatewayIntents` (accepts `['Guilds']` string names OR `GatewayIntentBits` numbers). `config.http` defaults `port` to `8080` and, on a Cloudflare worker, stashes config on `BaseClient._seyfertCfWorkerConfig`.
- `createEvent({ data: { name, once? }, run })` — `src/index.ts:68`. `once` defaults to `false`. `run` is typed `Awaitable<unknown>`. Gateway events still receive `(payload, client, shardId)`; v5 dropped the trailing `shardId` only for CUSTOM (non-gateway) events.
- `extendContext(cb)` — `src/index.ts:119`. Callback receives the command interaction and returns extra props merged onto every command context.
- Runtime clients (all extend `BaseClient`): `Client` (`src/client/client.ts`, gateway), `HttpClient` (`src/client/httpclient.ts:6`, interactions over HTTP), `WorkerClient` (`src/client/workerclient.ts:83`, sharded workers). Exported from root via `export * from './client'` (`src/index.ts:1`).
- `client.start(options?)` — `src/client/base.ts:374`. Accepts a partial of `langsDir | commandsDir | connection | token | componentsDir`. Reads token from the `start()` arg or runtime config (`getRC()`); throws `SeyfertError('INVALID_TOKEN')` if the token is not a non-empty string (`base.ts:382`). `start()` loads langs/commands/components and connects — it does NOT upload application commands.
- `HttpClient.start(options?)` — `src/client/httpclient.ts:10`. Calls `super.start()` then `this.execute(options.httpConnection)`; omits `connection`/`eventsDir` (no gateway).
- `client.uploadCommands({ applicationId?, cachePath? })` — `src/client/base.ts:1056`. Separate, explicit step to publish commands to Discord. Resolves `applicationId` from arg → RC → `this.applicationId`.
- `client.setServices({ rest, cache, langs, middlewares, handleCommand })` — `src/client/base.ts:309`. Swap custom services before `start()`. `handleCommand` takes the `HandleCommand` CONSTRUCTOR, not an instance. `langs.preferGuildLocale` (boolean) flips `ctx.t` to resolve guild locale before user locale (`base.ts:358,1396`).
- `Client` constructor options include `plugins?: readonly AnySeyfertPlugin[]` (`base.ts:1243`), `logger?: LoggerOptions` (`base.ts:1300`, honored on worker clients too), and `langs?: { preferGuildLocale?: boolean }` (`base.ts:1394`).
- Plugin factories `createPlugin(...)` / `definePlugins(...)` — `src/client/plugins.ts:240,283` (re-exported from root). `definePlugins` accepts an array or rest args.
- Module-augmentation registry `SeyfertRegistry` — `src/client/plugins/types.ts:32` (`export interface SeyfertRegistry {}`). Consumed in `src/commands/applications/shared.ts` to derive `client`, `langs`, `middlewares`, `plugins` types.
- `ParseClient<T extends BaseClient>` and `UsingClient` (now a TYPE ALIAS = `BaseClient & RegisteredPluginExtension & RegisteredClient`) — `src/commands/applications/shared.ts:43,48`. `UsingClient` is no longer an interface and CANNOT be interface-merged.

## Code Examples (verified)
Install (from the page):
```bash
npm add seyfert
# latest github-mirror changes:
npm add seyfert@dev
```

Minimal gateway entrypoint (root imports from 'seyfert'):
```ts
import { Client } from 'seyfert';

const client = new Client();
// start() loads commands/events/langs from configured folders and connects;
// it does NOT register commands with Discord.
client.start();
```

`seyfert.config.mjs` (or `.js`/`.ts`) at project root:
```ts
import { config } from 'seyfert';

export default config.bot({
  token: process.env.TOKEN ?? '',
  intents: ['Guilds'], // string names; numeric GatewayIntentBits also accepted
  locations: {
    base: 'dist',          // build output dir
    commands: 'commands',
    events: 'events',
    components: 'components',
    langs: 'langs',
  },
});
```

Required module augmentation (v5 pattern — augment `SeyfertRegistry`, NOT `UsingClient`):
```ts
import type { ParseClient, Client } from 'seyfert';

declare module 'seyfert' {
  interface SeyfertRegistry {
    client: ParseClient<Client<true>>;
    // langs: typeof import('./locales/en').default;          // optional, for ctx.t typing
    // middlewares: typeof import('./middlewares').middlewares; // optional
    // plugins: typeof import('./plugins').plugins;            // optional
  }
}
```

## Recipes (worked, copy-paste-ready)

### 1. A complete first bot (config + index + command + ready event + deploy)
`seyfert.config.ts` — as above.

`src/index.ts`:
```ts
import { Client, type ParseClient } from 'seyfert';

const client = new Client();

client.start().then(() => client.uploadCommands()); // upload AFTER start

declare module 'seyfert' {
  interface SeyfertRegistry {
    client: ParseClient<Client<true>>;
  }
}
```

`src/events/ready.ts` (auto-loaded from the `events` folder):
```ts
import { createEvent } from 'seyfert';

export default createEvent({
  data: { name: 'botReady', once: true },
  run(user, client, shardId) {
    client.logger.info(`Logged in as ${user.username} on shard #${shardId}`);
  },
});
```

`src/commands/ping.ts` (auto-loaded from the `commands` folder):
```ts
import { Command, Declare } from 'seyfert';

@Declare({ name: 'ping', description: 'Pong! Show latency.' })
export default class PingCommand extends Command {
  async run(ctx: CommandContext) {
    // write() returns void unless you pass `true` (withResponse) for the Message
    await ctx.write({ content: `Pong! ${ctx.client.gateway.latency}ms` });
  }
}
```
(Import `CommandContext` from `'seyfert'` and add it to the run signature, or rely on the inferred type. Command option-record keys must be lowercase in v5.)

### 2. HTTP-interactions bot (no gateway)
For serverless / webhook interaction endpoints, swap `Client` for `HttpClient` and `config.bot` for `config.http`.
```ts
// seyfert.config.ts
import { config } from 'seyfert';

export default config.http({
  token: process.env.TOKEN ?? '',
  publicKey: process.env.PUBLIC_KEY ?? '',
  applicationId: process.env.APP_ID ?? '',
  port: 3000,                 // optional; defaults to 8080
  locations: { base: 'dist', commands: 'commands' },
});
```
```ts
// src/index.ts
import { HttpClient, type ParseClient } from 'seyfert';

const client = new HttpClient();
client.start().then(() => client.uploadCommands());

declare module 'seyfert' {
  interface SeyfertRegistry {
    client: ParseClient<HttpClient>;
  }
}
```
`HttpClient.start()` does not connect a gateway; it stands up the interaction server (and calls `execute(httpConnection)`).

### 3. Custom context props with extendContext
```ts
import { extendContext } from 'seyfert';

export const context = extendContext(interaction => {
  return {
    owners: ['123456789012345678'],
    isOwner: interaction.user.id === '123456789012345678',
  };
});
```
Pass it via the client constructor so `ctx.owners` / `ctx.isOwner` are typed on every command context:
```ts
import { Client } from 'seyfert';
import { context } from './context';

const client = new Client({ context });
```

### 4. Swapping services / locale preference before start
```ts
import { Client } from 'seyfert';

const client = new Client({
  // logger is configurable here (also honored on worker clients)
  logger: { active: true, name: 'MyBot' },
});
// langs is a setServices option (NOT a Client ctor option); call before start():
client.setServices({ langs: { preferGuildLocale: true } });

// Replace built-in services before start(); handleCommand takes a CONSTRUCTOR.
// client.setServices({ handleCommand: MyHandleCommand });

client.start();
```

### 5. Reusable feature as a plugin (v5 first-class plugins)
```ts
import { Client, createPlugin, definePlugins } from 'seyfert';

const economy = createPlugin({
  name: 'economy',
  client: { economy: () => new EconomyService() }, // typed client extension
  register(api) {
    api.middlewares.add('balanceGuard', balanceGuard, { global: true });
  },
  setup: client => client.economy.connect(),
  teardown: client => client.economy.close(), // cleanup that actually runs
});

const client = new Client({ plugins: definePlugins(economy) });
```
(Register plugin-extended types via `SeyfertRegistry.plugins`. The full plugin lifecycle/ordering/diagnostics lives on the dedicated `/docs/plugins` pages.)

## Common patterns / gotchas
- `start()` connecting ≠ commands live. Slash commands only appear after `uploadCommands()` — run it once per deploy, e.g. `client.start().then(() => client.uploadCommands())`. Avoid uploading on every boot in production (rate limits); gate it behind a deploy script or flag.
- Augment `SeyfertRegistry`, never `UsingClient`. `UsingClient` is a type alias in v5 (`shared.ts:43`); the old `interface UsingClient extends ParseClient<Client<true>>` silently fails to merge and degrades client typing to `BaseClient`. `ParseMiddlewares` and per-interface augmentation of `RegisteredMiddlewares`/`DefaultLocale` are gone — use `SeyfertRegistry` keys `client`/`middlewares`/`langs`/`plugins`.
- Intents accept inline strings now: `intents: ['Guilds']` in `config.bot`, or `client.start({ connection: { intents: ['Guilds'] } })`. `GatewayIntentBits` numbers still work.
- A bad/missing token throws `SeyfertError('INVALID_TOKEN')` at start and per request — not a generic assertion. Catch with `SeyfertError.is(err, 'INVALID_TOKEN')`.
- A failed `seyfert.config` load throws `SEYFERT_CONFIG_LOAD_ERROR` with the original error preserved as `cause`.
- Folder names in `locations` are relative to `locations.base` (your build output, e.g. `dist`). Run the compiled JS, not the TS source, unless using a TS runtime.
- Custom (non-gateway) events: `run(payload, client)` — no trailing `shardId`. Gateway events keep `(payload, client, shardId)`.
- For sharded/worker setups use `WorkerClient` + `WorkerManager`; `WorkerManagerOptions` is a discriminated union over `mode` (`threads`/`clusters` need `path`, `custom` needs `adapter`).

## Doc vs Source Corrections
- Docs say Deno/Bun supported "from version **2.2.0**" -> stale legacy versioning; current package is `seyfert@5.0.0` (`package.json`). Treat the 2.2.0 number as doc lore, not a real v5 constraint.
- Draft/v4 note used `interface UsingClient extends ParseClient<Client<true>>` -> in v5 `UsingClient` is a type alias (`shared.ts:43`) and cannot be interface-merged. Register through `declare module 'seyfert' { interface SeyfertRegistry { client: ParseClient<Client<true>> } }` (`plugins/types.ts:32`, resolved at `shared.ts:30`).
- `ParseMiddlewares` REMOVED — augment `SeyfertRegistry.middlewares` with `typeof middlewares` directly (`createMiddleware` already returns the right shape).
- Clarify (not in page): `start()` does not upload commands; `uploadCommands()` is the separate step (`base.ts:1056`).

## Source Anchors
- `package.json` (name/version)
- `src/index.ts` (root exports, `config` :76, `createEvent` :68, `extendContext` :119)
- `src/client/base.ts` (`setServices` :309, `start` :374, `INVALID_TOKEN` :382, `uploadCommands` :1056, ctor `plugins` :1243 / `logger` :1300 / `langs` :1394, `preferGuildLocale` :358/:1396)
- `src/client/httpclient.ts` (`HttpClient`, `start` :10), `src/client/workerclient.ts`, `src/client/client.ts`
- `src/client/plugins.ts` (`createPlugin` :240, `definePlugins` :283)
- `src/commands/applications/shared.ts` (`UsingClient` :43 alias, `ParseClient` :48, `RegisteredClient` :30, `DefaultLocale` :26)
- `src/client/plugins/types.ts:32` (`SeyfertRegistry`)
- `src/commands/decorators.ts` (`Declare` :195, `Options` :161), `src/commands/applications/chat.ts:361` (`Command`)

## Agent Guidance
- Use this page only for project bootstrap: install, set `engines`/TS target, create `seyfert.config.*`, scaffold `commands`/`events`/`components`/`langs` folders, and add the `SeyfertRegistry` augmentation so `CommandContext`/`client.t`/middleware metadata are typed.
- Default first-bot shape: `Client` + `config.bot` + a `ready`-style event + one `@Declare`d command + `start().then(uploadCommands)` + `SeyfertRegistry.client` augmentation. Reach for `HttpClient`/`config.http` for serverless interaction bots, `WorkerClient`/`WorkerManager` for sharding.
- Gotcha: forgetting the `SeyfertRegistry.client` augmentation degrades client typing to base `BaseClient`. Do not resurrect `interface UsingClient extends ...` — it silently won't merge.
- Gotcha: `start()` succeeding does NOT mean slash commands are live; call `uploadCommands()` (typically once / on deploy).
- Always import public API from `'seyfert'`; reach into `'seyfert/lib/...'` only for types/utilities not re-exported by the root barrel.
