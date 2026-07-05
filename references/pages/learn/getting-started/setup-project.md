# Configuring Seyfert (Setup Project)

Original source URL: https://seyfert-web-git-seyfert-v5-tiramisulabs.vercel.app/docs/learn/getting-started/setup-project

Coverage reference: setup-runtime.md

Verification status: Source-verified (core, seyfert@5.0.0) + external HTTP adapter package

## Page Summary

Covers the post-install setup: enabling TypeScript decorators in `tsconfig.json`, choosing a bot type (Gateway vs HTTP), writing a `seyfert.config.*` file via `config.bot()` / `config.http()`, creating the `src/index.ts` entrypoint that constructs and `start()`s a client, and wiring up the `SeyfertRegistry` module augmentation so the framework knows your client type. For HTTP bots an external HTTP adapter package is required. `start()` loads handlers (and, for gateway, connects) but never uploads application commands to Discord.

## Key APIs (verified)

- `config.bot(data: RuntimeConfig)` — returns `InternalRuntimeConfig`; resolves `intents` through `resolveGatewayIntents(data.intents)`. (src/index.ts:83, src/client/intents.ts)
- `config.http(data: RuntimeConfigHTTP)` — defaults `port` to `8080`, spreads your data over it; if running in a Cloudflare Worker it stores the config on `BaseClient._seyfertCfWorkerConfig`. (src/index.ts:95)
- `RuntimeConfig` fields (src/client/base.ts:1357-1384, via interface `RC`): `token: string` (required), `locations: { base: string; commands?; langs?; events?; components? }` (only `base` required), `intents?: GatewayIntentInput` (string bit-name / number / array of names), `applicationId?`, `port?`, `publicKey?`, `debug?`.
- `RuntimeConfigHTTP` (src/client/base.ts:1379): like above but `publicKey` and `applicationId` are REQUIRED, `intents` is omitted, and `locations` omits `events` (HTTP clients have no gateway events).
- `BotConfig = InternalRuntimeConfig`, `HttpConfig = InternalRuntimeConfigHTTP` — both exported root types (src/client/base.ts:1385-1386).
- `Client`, `HttpClient`, `WorkerClient` — all exported from the package root `'seyfert'` (via `export * from './client'` in src/index.ts:1; defined in src/client/client.ts, httpclient.ts, workerclient.ts).
- `client.start(options?)` — `options` is a partial `StartOptions` (`token`, `connection`, `langsDir`, `commandsDir`, `componentsDir`, gateway also `eventsDir`). Loads langs, commands, components (and events for gateway `Client`); for gateway it also connects. It does NOT upload application commands. (src/client/base.ts:374, src/client/client.ts:145)
- `client.uploadCommands({ applicationId?, cachePath? })` — SEPARATE call to register commands with Discord; not part of `start()`. `applicationId` falls back to RC `applicationId` then `client.applicationId`. (src/client/base.ts:1056)
- `HttpClient.start(options?)` — calls `super.start()` then `this.execute(options.httpConnection)`; no event loading. You still need an adapter to receive requests. (src/client/httpclient.ts:11)
- `getRC()` config resolution — looks for `seyfert.config` with extension `.js`, `.mjs`, `.cjs`, `.ts`, `.mts`, `.cts` in `process.cwd()`, using its `default` export (falls back to the module namespace). Result is cached (`_rcCache`). Can be overridden by passing `getRC` in client options, or supplied automatically on Cloudflare Workers via `_seyfertCfWorkerConfig`. (src/client/base.ts:1188-1213, 1301)
- Module augmentation point: `interface SeyfertRegistry {}` in `'seyfert'` (declared in src/client/plugins/types.ts:32). Keys: `client`, `middlewares`, `langs`, `plugins`. `UsingClient`, `RegisteredMiddlewares`, `DefaultLocale` are all DERIVED from it (do not augment those directly in v5).
- `ParseClient<T extends BaseClient> = T & { readonly [__seyfertClientType]?: T }` (src/commands/applications/shared.ts:48). `UsingClient` is derived from `SeyfertRegistry.client` via `RegisteredClient` (shared.ts:30-43).
- `extendContext(cb)` — root helper to add custom properties to every command `ctx`; pass via `new Client({ context: customContext })`. (src/index.ts, `extendContext`)

## Code Examples (verified)

tsconfig.json (decorators are mandatory):
```json
{
    "compilerOptions": {
        "experimentalDecorators": true,
        "emitDecoratorMetadata": true
    }
}
```

seyfert.config.mjs — Gateway:
```ts
import { config } from "seyfert";

export default config.bot({
    token: process.env.BOT_TOKEN ?? "",
    locations: {
        base: "dist",       // use "src" if running with bun/tsx without a build
        commands: "commands"
        // optional: events, components, langs
    },
    intents: ["Guilds"],
    // optional: also receive interactions over HTTP alongside the gateway
    // publicKey: "...",
    // port: 4444,
});
```

seyfert.config.mjs — HTTP (publicKey + applicationId required):
```ts
import { config } from "seyfert";

const { BOT_TOKEN, BOT_APP_ID, BOT_PUBLIC_KEY } = process.env;

export default config.http({
    token: BOT_TOKEN ?? "",
    locations: {
        base: "dist",
        commands: "commands"
    },
    applicationId: BOT_APP_ID ?? "",
    publicKey: BOT_PUBLIC_KEY ?? "",
    port: 3000, // default is 8080
});
```

src/index.ts — Gateway:
```ts
import { Client } from "seyfert";

const client = new Client();

// connects to the gateway and loads commands, events, components, langs
client.start();
// to register commands with Discord, call separately (e.g. after ready):
// client.uploadCommands();
```

src/index.ts — HTTP (needs an external adapter package):
```ts
import { HttpClient } from "seyfert";
import { UwsAdapter } from "@slipher/uws-adapter";
// import { GenericAdapter } from "@slipher/generic-adapter";

const client = new HttpClient();
const adapter = new UwsAdapter(client); // also accepts Client / WorkerClient

client.start();
adapter.start();
```

Module augmentation (declare your client type) — verified shape:
```ts
import type { ParseClient, Client, HttpClient, WorkerClient } from 'seyfert';

declare module 'seyfert' {
    interface SeyfertRegistry {
        client: ParseClient<Client<true>>;       // Gateway
        // client: ParseClient<HttpClient>;          // HTTP
        // client: ParseClient<WorkerClient<true>>;  // Worker
    }
}
```

## Recipes / Common patterns

Full `SeyfertRegistry` augmentation (client + middlewares + langs + plugins) — keep it in one `declare module` block, often `src/index.ts`:
```ts
import { type ParseClient, type ParseLocales, Client } from 'seyfert';
import type * as defaultLang from './langs/en'; // your base lang module
import { middlewares } from './middlewares';     // export const middlewares = { ... }

declare module 'seyfert' {
    interface SeyfertRegistry {
        client: ParseClient<Client<true>>;
        middlewares: typeof middlewares;             // NOT ParseMiddlewares (removed in v5)
        langs: ParseLocales<typeof defaultLang>;     // enables typed ctx.t.<key>
    }
}
```

Upload commands on ready — `start()` does NOT publish them, so do it explicitly. Put this in `src/events/botReady.ts` (auto-loaded from `locations.events`):
```ts
import { createEvent } from 'seyfert';

export default createEvent({
    data: { name: 'botReady', once: true },
    run(user, client) {
        client.logger.info(`Logged in as ${user.username}`);
        // Publish global commands to Discord. Pass { applicationId } / { cachePath } if needed.
        client.uploadCommands();
    },
});
```

Custom context properties with `extendContext` — wire the callback into the client, then augment the context type:
```ts
import { extendContext, Client } from 'seyfert';

const context = extendContext(interaction => ({
    // available as ctx.<key> in every command
    isOwner: interaction.user.id === process.env.OWNER_ID,
}));

const client = new Client({ context });
client.start();

declare module 'seyfert' {
    interface ExtendContext extends ReturnType<typeof context> {}
}
```

Programmatic config / no `seyfert.config` file — supply `getRC` directly (handy for tests, monorepos, or env-only setups):
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

Configure the logger inline (now honored on worker clients too):
```ts
import { Client, LogLevels } from 'seyfert';

// LoggerOptions: { active?, logLevel?, name?, saveOnFile? }
const client = new Client({ logger: { active: true, logLevel: LogLevels.Debug } });
```

package.json scripts + .env (typical TS build flow):
```jsonc
{
    "scripts": {
        "build": "tsc",
        "start": "node dist/index.js",
        "dev": "tsx src/index.ts"   // run TS directly; set locations.base to "src"
    }
}
```
```dotenv
# .env  (load via node --env-file=.env, or dotenv)
BOT_TOKEN=...
BOT_APP_ID=...
BOT_PUBLIC_KEY=...
```

## Doc vs Source Corrections

- Largely aligned with the v5 MDX. The doc's `SeyfertRegistry` augmentation matches current source (the older `UsingClient extends ParseClient<...>` / `RegisteredMiddlewares extends ParseMiddlewares<...>` forms are gone; augment `SeyfertRegistry.{client,middlewares,langs,plugins}` — src/client/plugins/types.ts:32, src/commands/applications/shared.ts:30-43).
- Doc implies `client.start()` is all you need; clarify that `start()` does NOT upload commands to Discord — `uploadCommands()` is a separate call (src/client/base.ts:1056), usually in a `botReady` event.
- Doc only shows `seyfert.config.mjs`; source accepts `.js .mjs .cjs .ts .mts .cts` resolved from `process.cwd()` (src/client/base.ts:1196).
- Doc's HTTP MDX assigns possibly-`undefined` env values to required fields (`token`, `applicationId`, `publicKey`); under `strictNullChecks` add `?? ""` fallbacks as shown above (these fields are required in `RuntimeConfigHTTP`).
- `config.http` default `port` is `8080` (matches doc comment); `config.bot` has no default port.
- Error codes: a missing config file throws `NO_SEYFERT_CONFIG` (metadata detail `"No seyfert.config file found"`); a config that exists but fails to import throws `SEYFERT_CONFIG_LOAD_ERROR` with the original error as `cause`; a missing/empty token throws `INVALID_TOKEN`. All surface at `start()` time via `getRC()`. (src/client/base.ts:1207-1213, 383)

## Source Anchors

- package.json (name "seyfert", version 5.0.0, root export ./lib/index)

## Agent Guidance

- Always add the `SeyfertRegistry.client` augmentation in `declare module 'seyfert'` — without it, command/event handler `ctx.client` typing falls back to `BaseClient`. Add `middlewares` and `langs` keys in the same block when you have them.
- `locations.base` must point at the directory that actually contains your command/event/component/lang folders: `dist` after a `tsc` build, `src` when running TS directly under bun/tsx.
- HTTP bots need an adapter: `@slipher/uws-adapter` or `@slipher/generic-adapter` are EXTERNAL packages (not in core Seyfert) — verify their versions in the target project. The core `HttpClient`/`config.http` API is verified here. Cloudflare Workers have a dedicated recipe; config is picked up via `_seyfertCfWorkerConfig`.