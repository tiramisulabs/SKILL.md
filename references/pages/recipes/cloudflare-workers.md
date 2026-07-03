# Using Cloudflare Workers

Original source URL: https://seyfert-web-git-seyfert-v5-tiramisulabs.vercel.app/docs/recipes/cloudflare-workers
Coverage reference: setup-runtime.md
Verification status: Source-verified (core, the authoritative Seyfert source) + external package (`@slipher/generic-adapter`)

## Page Summary

Deploy a Seyfert HTTP-interaction bot to Cloudflare Workers. Because Workers have no
filesystem (`fs`), Seyfert cannot auto-discover commands/langs/components from disk, so you
import each one and register it programmatically with `client.commands.set` /
`client.langs.set` / `client.components.set`. The Discord request bridge is the **external**
`@slipher/generic-adapter`, which turns a `fetch(Request)` into a Seyfert HTTP interaction
response (signature verification + routing). The key core mechanism: calling `config.http(...)`
while running inside a Worker stashes the config statically
(`BaseClient._seyfertCfWorkerConfig`) so `getRC()` never touches the filesystem.

Note: this is **Cloudflare Workers (HTTP interactions)** — completely unrelated to Seyfert's
gateway `WorkerManager`/`WorkerClient` (sharded gateway across Node worker threads). The client
here is `HttpClient`, never `WorkerClient`.

## Key APIs (verified)

- `HttpClient` (root import from `'seyfert'`; `src/client/httpclient.ts`) — `BaseClient` subclass.
  - `constructor(options?: BaseClientOptions)`.
  - `start(options?: DeepPartial<Omit<StartOptions, 'connection' | 'eventsDir'>>)` — runs
    `super.start()` then `this.execute(options.httpConnection)`. `execute(...)` is a protected
    no-op stub (`src/client/base.ts:365`); `HttpClient` has **no built-in HTTP listener** — you
    must attach an adapter to receive requests.
  - `StartOptions.httpConnection` = `{ publicKey: string; port: number }` (`src/client/base.ts:1342`).
  - `onInteractionRequest(rawBody)` (`src/client/base.ts:961`) — the method the adapter calls per
    request; answers `InteractionType.Ping` with `{ type: Pong }` directly (no command handler),
    otherwise routes the interaction and resolves `{ headers, response }` (JSON or `FormData` when
    files are attached).
- `config.http(data: RuntimeConfigHTTP)` (root import; `src/index.ts:95`) — returns
  `InternalRuntimeConfigHTTP`, defaults `port: 8080`, and **if `isCloudflareWorker()` is true it
  assigns `BaseClient._seyfertCfWorkerConfig = obj`** (`src/index.ts:100`). This is what lets the
  Worker run without reading `seyfert.config` from disk.
- `RuntimeConfigHTTP` (`src/client/base.ts:1379`) — requires `publicKey` and `applicationId`, plus
  `token`, `locations` (a `RCLocations` minus `events`), optional `port`, `debug`. No `intents`
  (HTTP only). `HttpConfig` is the exported alias of `InternalRuntimeConfigHTTP`.
- `getRC()` (`src/client/base.ts:1188`) — resolution order: `BaseClient._seyfertCfWorkerConfig`
  -> `options.getRC?.()` -> `Promise.any` dynamic import of
  `seyfert.config.{js,mjs,cjs,ts,mts,cts}` from `process.cwd()`. The last branch throws
  `SeyfertError('SEYFERT_CONFIG_LOAD_ERROR')` (with the import error as `cause`) or
  `'NO_SEYFERT_CONFIG'`, and touches `fs`/`process.cwd()` — which is why the static path matters
  on Workers. Result is cached in `_rcCache`.
- `isCloudflareWorker()` (`src/common/it/utils.ts:313`) — returns `process.platform === 'browser'`
  (the value the Workers runtime reports).
- Programmatic registration (all synchronous, accept arrays, return the loaded instances):
  - `client.commands.set(commands: SeteableCommand[], optionsOrTransform?)` (`src/commands/handler.ts:309`).
    `SeteableCommand = HandleableCommand | HandleableCommandInstance` (a command class **or**
    instance). Second arg is a `CommandLoadOptions` or a transform fn.
  - `client.langs.set(instances: LangInstance[])` (`src/langs/handler.ts:71`);
    `LangInstance = { name: string; file: FileLoaded<Record<string, any>>; path?: string }` (`:155`).
    `file` may be the lang object itself, an ESM namespace with a `default`, or a single named
    object export (`onFile`, `:109`).
  - `client.components.set(instances: SeteableComponentCommand[], optionsOrTransform?)` (`src/components/handler.ts:286`).
- `client.uploadCommands({ applicationId?, cachePath? })` (`src/client/base.ts:1056`) — registers
  your loaded commands with Discord. `applicationId` falls back to `getRC().applicationId`. You
  call this **once** (deploy step), not on every Worker request.
- `GenericAdapter` — **external** (`@slipher/generic-adapter`, not in core Seyfert). Provides
  `new GenericAdapter(client)`, `adapter.start()`, and `adapter.fetch(req)`. Verify its exact
  API/version in the target project.

## Code Examples (verified)

### 1. Install + tsconfig

```bash
npm add @slipher/generic-adapter
```

`tsconfig.json` must emit ESM (Workers are ESM-only):

```jsonc
{
  "compilerOptions": {
    "module": "ESNext",
    "moduleResolution": "Bundler", // or "NodeNext"
    "target": "ESNext",
    "experimentalDecorators": true,
    "emitDecoratorMetadata": true
  }
}
```

### 2. `seyfert.config.mjs` — the side-effect that powers the Worker

```ts
import { config } from 'seyfert';

// Calling config.http() inside a Worker sets BaseClient._seyfertCfWorkerConfig,
// so getRC() never imports from disk. publicKey + applicationId are REQUIRED.
export default config.http({
  token: process.env.BOT_TOKEN ?? '',
  publicKey: process.env.BOT_PUBLIC_KEY ?? '',
  applicationId: process.env.BOT_APP_ID ?? '',
  locations: {
    base: 'src',
    commands: 'commands',
    components: 'components',
    langs: 'languages',
  },
});
```

### 3. `index.ts` — manual loading + Worker `fetch` handler

```ts
import '../seyfert.config.mjs'; // side-effect import: must run BEFORE getRC()
import { HttpClient } from 'seyfert';
import { GenericAdapter } from '@slipher/generic-adapter';

// commands
import Ping from './commands/ping.js';
// langs
import EnLang from './languages/en.js';
// components
import ButtonC from './components/buttonHandle.js';

const client = new HttpClient();
const adapter = new GenericAdapter(client);

// .set(...) calls are synchronous; async is unnecessary on this callback.
const ready = client
  .start()
  .then(() => {
    client.commands.set([Ping]);                       // commands (classes or instances)
    client.langs.set([{ name: 'en', file: EnLang }]);  // languages
    client.components.set([ButtonC]);                  // component/modal handlers
  })
  .then(() => adapter.start());

export default {
  async fetch(req: Request) {
    await ready;            // ensure registration finished before first request
    return adapter.fetch(req);
  },
};
```

### 4. A command file that works under manual `.set([...])`

Nothing about the command itself changes for Workers — only how it gets loaded.

```ts
// src/commands/ping.ts
import { Command, Declare, type CommandContext } from 'seyfert';

@Declare({ name: 'ping', description: 'Pong!' })
export default class Ping extends Command {
  async run(ctx: CommandContext) {
    // ctx.client.gateway is unavailable on HTTP; use REST/ping latency-free reply.
    await ctx.write({ content: 'Pong! 🏓' });
  }
}
```

`client.commands.set([Ping])` accepts the class directly (it is a `SeteableCommand`). It also
accepts an instance: `client.commands.set([new Ping()])`.

### 5. A lang file + i18n

```ts
// src/languages/en.ts  — default export is the lang object
export default {
  greet: (name: string) => `Hello, ${name}!`,
};
```

```ts
// register it, then resolve in a command via ctx.t
client.langs.set([{ name: 'en', file: EnLang }]);

// inside a command:
const t = ctx.t.get(ctx.interaction.locale); // SeyfertLocale proxy -> resolved strings
await ctx.write({ content: t.greet(ctx.author.username) });
```

Gotcha: `langs.reload(...)` throws `SeyfertError('RELOAD_NOT_SUPPORTED')` inside a Worker
(`src/langs/handler.ts:78`). Hot lang reloading is a dev-only feature — never call it in the
Worker request path.

### 6. A component handler file

```ts
// src/components/buttonHandle.ts
import { ComponentCommand, type ComponentContext } from 'seyfert';

export default class ButtonC extends ComponentCommand {
  componentType = 'Button' as const;

  filter(ctx: ComponentContext<'Button'>) {
    return ctx.customId === 'my-button';
  }

  async run(ctx: ComponentContext<'Button'>) {
    await ctx.update({ content: 'Clicked!' });
  }
}
```

### 7. Typed client augmentation (so `ctx.client` / commands stay typed)

Augment the central `SeyfertRegistry` (v5) — **not** `UsingClient`/`RegisteredMiddlewares`,
which are derived now.

```ts
// src/index.ts (or a types.d.ts)
import type { ParseClient, HttpClient } from 'seyfert';

declare module 'seyfert' {
  interface SeyfertRegistry {
    client: ParseClient<HttpClient>;
    // middlewares: typeof middlewares; // if you use any
  }
}
```

### 8. Deploying commands to Discord (one-off, not per request)

`client.commands.set([...])` only loads them into memory; Discord still needs them uploaded.
Run a small Node script locally (it can read `seyfert.config` from disk normally there):

```ts
// scripts/deploy.ts  — run with node/tsx outside the Worker
import { HttpClient } from 'seyfert';
import Ping from '../src/commands/ping.js';

const client = new HttpClient();
await client.start();
client.commands.set([Ping]);
await client.uploadCommands(); // applicationId resolved from getRC()
```

## Doc vs Source Corrections

- Upstream MDX (and the original draft) never explain *why* importing `seyfert.config.mjs` works
  on Workers. Source: the side effect of `config.http()` sets `BaseClient._seyfertCfWorkerConfig`
  only when `isCloudflareWorker()` (`process.platform === 'browser'`); `getRC()` reads that first,
  avoiding the filesystem import (`src/index.ts:100`, `src/client/base.ts:1193`).
- Earlier drafts listed `src/client/workerclient.ts` and `src/websocket/discord/**` as anchors —
  those belong to the gateway `WorkerManager` (sharded gateway across Node worker threads),
  unrelated to Cloudflare Workers / HTTP interactions. The relevant client is `HttpClient`.
- Upstream's `.then(async () => {...})` callback contains no `await`; the `.set(...)` calls are
  synchronous, so `async` is unnecessary (dropped above). Behaviorally identical.
- `langs.reload()` throws `SeyfertError('RELOAD_NOT_SUPPORTED')` inside a Worker
  (`src/langs/handler.ts:78`) — do not rely on hot lang reloading there.
- Otherwise the upstream example matches source: `HttpClient` import, `client.start()`,
  `commands/langs/components.set([...])`, and the `{ name, file }` lang shape are all correct.

## Common patterns / gotchas

- **Config side effect must run first.** Put `import '../seyfert.config.mjs';` at the very top of
  `index.ts`, before anything triggers `getRC()`. Skip it and `getRC()` falls through to the
  filesystem branch and throws (`NO_SEYFERT_CONFIG`) on a Worker.
- **Secrets, not `.env`.** Workers have no `.env` file. Set `BOT_TOKEN`, `BOT_PUBLIC_KEY`,
  `BOT_APP_ID` with `wrangler secret put <NAME>` (or the dashboard). `process.env.X` reads them at
  runtime; `publicKey` is what the adapter uses to verify Discord's Ed25519 signatures.
- **Keep `await ready;` in `fetch`.** It guarantees `start()` + registration finished before the
  first interaction is processed. The bundler may keep the module warm across requests, so
  `ready` resolves once and is reused.
- **No `fs` / `process.cwd()` at request time.** Directory auto-loading (`load()`/`loadFilesK`)
  relies on `fs`; that's exactly why you `.set([...])` manually. Avoid any per-request code path
  that touches the filesystem, `process.cwd()`, or `langs.reload()`.
- **`port` is irrelevant on Workers.** No socket is opened — the runtime calls your exported
  `fetch`. `config.http` still defaults it to `8080`; it's only meaningful for a Node HTTP server.
- **HTTP only = no gateway features.** No `client.gateway`, no presence, no voice, no
  message-content events. Only interaction-driven flows (slash/context commands, components,
  modals, autocomplete) work.
- **Other runtimes, same shape.** Any no-`fs`, ESM-only host (Bun/Deno deploy, a custom
  `node:http` adapter) follows the identical pattern: emit ESM, register manually, attach an
  adapter that calls `client.onInteractionRequest(...)`.

## Source Anchors

- `src/client/httpclient.ts` — `HttpClient` class + `start`.
- `src/index.ts:76-103` — `config.bot` / `config.http`, static-config assignment.
- `src/client/base.ts:209,365,961,1056,1188-1239,1336-1386` — `_seyfertCfWorkerConfig`, `execute`
  stub, `onInteractionRequest`, `uploadCommands`, `getRC`, `StartOptions`, RC types.
- `src/common/it/utils.ts:313` — `isCloudflareWorker`.
- `src/commands/handler.ts:309,674-684` — `commands.set`, `SeteableCommand`, `FileLoaded`.
- `src/langs/handler.ts:71,78,109,155` — `langs.set`, reload throw, `onFile`, `LangInstance`.
- `src/components/handler.ts:286,471` — `components.set`, `SeteableComponentCommand`.

## Agent Guidance

- Use when targeting Cloudflare Workers (or any no-`fs`, ESM-only runtime). The two
  non-negotiables: (1) emit ESM (`module: "ESNext"`); (2) register everything manually with
  `.set([...])` because directory auto-loading relies on `fs`.
- Import `seyfert.config.mjs` (side-effect) at the top of `index.ts` so the static config is in
  place before `getRC()` runs.
- `HttpClient` only validates/handles interactions; the HTTP transport is the adapter's job.
  `@slipher/generic-adapter` is external — confirm its `start()`/`fetch()` API and version in the
  user's project before editing.
- Augment `SeyfertRegistry` (not `UsingClient`) for typed `ctx.client`. `config.http` requires
  `publicKey` + `applicationId`; `port` is irrelevant on Workers.
- Command registration to Discord (`uploadCommands`) is a separate one-off deploy step, not part
  of the Worker request lifecycle.
