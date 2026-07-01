# Hot Reload

Original source URL: https://seyfert-web-git-seyfert-v5-tiramisulabs.vercel.app/docs/recipes/hot-reload
Coverage reference: setup-runtime.md
Verification status: Source-verified (core, the authoritative Seyfert source) + external package (@slipher/watcher)

## Page Summary

During development you can reload commands, events, components, and language files in place without restarting the bot. Each handler exposes `reload(...)` (single target) and `reloadAll(stopIfFail?)` (every loaded file). Reload works by deleting the module from `require.cache` and re-importing via `magicImport`, so it depends on each loaded item carrying a `__filePath`. Reload is NOT supported under Cloudflare Workers — every handler throws `SeyfertError('RELOAD_NOT_SUPPORTED')`. For automatic file-watching, Seyfert ships the runtime integration types in core and pairs with the external `@slipher/watcher` runner.

## Key APIs (verified)

All four handlers are always-initialized concrete instances on the client (never undefined), so optional chaining is unnecessary:
- `client.commands` -> `CommandHandler` (src/client/base.ts:183)
- `client.components` -> `ComponentHandler` (src/client/base.ts:184)
- `client.langs` -> `LangsHandler` (src/client/base.ts:182)
- `client.events` -> `CustomEventHandler` (typed `CustomEventRunner`, src/client/base.ts:197)

Reload methods (all guard `isCloudflareWorker()` and throw `SeyfertError('RELOAD_NOT_SUPPORTED')` if true):

- `commands.reload(resolve: string | Command): Promise<Command | ContextMenuCommand | EntryPointCommand | null>` — accepts a command name OR a `Command` instance; returns `null` if not found / no `__filePath`. (src/commands/handler.ts:41)
- `commands.reloadAll(stopIfFail = true): Promise<void>` — iterates `this.values`, reloading by `command.name`. (src/commands/handler.ts:77)
- `components.reload(path: string): Promise<unknown | null | undefined>` — keyed by FILE PATH, not name; matches `__filePath` ending in `${path}.js` / `${path}.ts` / `path`, or an exact `__filePath ===` match. Returns early `undefined` if `client.components` is unset. (src/components/handler.ts:341)
- `components.reloadAll(stopIfFail = true): Promise<void>` — iterates loaded components, reloading by each `__filePath`. (src/components/handler.ts:368)
- `events.reload(name: GatewayEvents | CustomEventsKeys): Promise<ClientEvent | null>` (src/events/handler.ts:430)
- `events.reloadAll(stopIfFail = true): Promise<void>` (src/events/handler.ts:448)
- `langs.reload(lang: string): Promise<Record<string, any> | null>` — fires `onReload?.(lang, file)` hook after success. (src/langs/handler.ts:77)
- `langs.reloadAll(stopIfFail = true): Promise<void>` (src/langs/handler.ts:99)

`stopIfFail` (default `true`): when true a thrown error during a single reload aborts the whole `reloadAll`; pass `false` to skip failing files and continue. Not documented upstream.

Watcher integration (core types; runner is external):
- `WatcherOptions` / `WatcherPayload` / `WatcherSendToShard` — src/common/bot/watcher.ts. `WatcherOptions` extends most `ShardManager` options (compress, presence, properties, shardEnd/shardStart, totalShards, url, version, resharding, debug — all partial) plus required `{ filePath: string; transpileCommand: string; srcPath: string }` and optional `argv?: string[]`.
- The gateway `Client` detects `workerData.__USING_WATCHER__` and, when present, listens on `parentPort` for `PAYLOAD` / `SEND_TO_SHARD` messages instead of spawning its own shards (src/client/client.ts:127). This is how `@slipher/watcher` keeps the gateway alive across worker restarts; without the flag the client calls `gateway.spawnShards()` itself.
- These watcher types are exported through the `./common` barrel and are NOT in the root `seyfert` re-export list; the `Watcher` CLASS itself lives in `@slipher/watcher`, not in core.

## Code Examples (verified)

### Basic reload calls

```ts
// Commands
await client.commands.reloadAll();        // all; throws on first failure
await client.commands.reloadAll(false);   // continue past failures
await client.commands.reload('ping');     // by name (or pass a Command instance)

// Events / Components / Langs
await client.events.reloadAll();
await client.components.reloadAll();       // components.reload() is keyed by FILE PATH
await client.langs.reloadAll();
await client.langs.reload('en-US');        // by locale key
```

### Developer reload command (verified against Command / Options / CommandContext)

```ts
import {
  Command,
  Declare,
  Options,
  Middlewares,
  createStringOption,
  type CommandContext,
} from 'seyfert';

const options = {
  // option keys MUST be lowercase in v5 (compile-time enforced)
  target: createStringOption({
    description: 'What to reload',
    required: true,
    choices: [
      { name: 'Commands', value: 'commands' },
      { name: 'Events', value: 'events' },
      { name: 'Components', value: 'components' },
      { name: 'Languages', value: 'langs' },
      { name: 'All', value: 'all' },
    ] as const, // choices is readonly in v5 -> use `as const`
  }),
} as const;

@Declare({ name: 'reload', description: 'Reload bot modules' })
@Middlewares(['ownerOnly']) // gate it — never expose reload publicly
@Options(options)
export default class ReloadCommand extends Command {
  async run(ctx: CommandContext<typeof options>) {
    const target = ctx.options.target;
    const c = ctx.client;
    // reloadAll(false) so one broken file does not abort reloading the rest
    if (target === 'all' || target === 'commands') await c.commands.reloadAll(false);
    if (target === 'all' || target === 'events') await c.events.reloadAll(false);
    if (target === 'all' || target === 'components') await c.components.reloadAll(false);
    if (target === 'all' || target === 'langs') await c.langs.reloadAll(false);
    // editOrReply returns void unless you pass `true` as the 2nd arg
    await ctx.editOrReply({ content: `Reloaded **${target}** successfully.` });
  }
}
```

### Owner-only middleware to gate the command

In v5 the middleware callback arg is `{ context, next, stop }` — there is no `pass`. Use `stop('reason')` to deny (routes to `onMiddlewaresError`), or bare `stop()` to skip silently.

```ts
import { createMiddleware } from 'seyfert';

export const ownerOnly = createMiddleware<void>(({ context, next, stop }) => {
  const owners = ['123456789012345678']; // your user id(s)
  if (!owners.includes(context.author.id)) return stop('You are not the owner.');
  return next();
});

// register it in your middlewares map and SeyfertRegistry:
// export const middlewares = { ownerOnly };
// declare module 'seyfert' {
//   interface SeyfertRegistry { middlewares: typeof middlewares }
// }
```

### Reload a single command and report success/failure

`reload(name)` returns `null` when the command is unknown or was registered without a `__filePath`. Surface that to the user instead of pretending it worked.

```ts
import { Command, Declare, Options, createStringOption, type CommandContext } from 'seyfert';

const options = {
  name: createStringOption({ description: 'Command name to reload', required: true }),
} as const;

@Declare({ name: 'reload-one', description: 'Reload a single command' })
@Middlewares(['ownerOnly'])
@Options(options)
export default class ReloadOne extends Command {
  async run(ctx: CommandContext<typeof options>) {
    const result = await ctx.client.commands.reload(ctx.options.name);
    await ctx.editOrReply({
      content: result
        ? `Reloaded \`${ctx.options.name}\`.`
        : `Could not reload \`${ctx.options.name}\` (not found or no file path).`,
    });
  }
}
```

### Handling the Cloudflare guard with SeyfertError.is

Every reload method throws `SeyfertError('RELOAD_NOT_SUPPORTED')` on edge/HTTP-only runtimes. Narrow it with the typed `SeyfertError.is(error, code)` helper (src/common/it/error.ts:39) rather than string-matching.

```ts
import { SeyfertError } from 'seyfert';

try {
  await client.commands.reloadAll(false);
} catch (error) {
  if (SeyfertError.is(error, 'RELOAD_NOT_SUPPORTED')) {
    client.logger.warn('Hot reload is unavailable in this runtime (Cloudflare/edge).');
  } else {
    throw error;
  }
}
```

### React to lang reloads with onReload

`LangsHandler` exposes an `onReload?(lang, file)` hook that fires after a successful locale reload — handy for logging or invalidating a cached translation map.

```ts
client.langs.onReload = (lang, file) => {
  client.logger.info(`Locale "${lang}" reloaded`, Object.keys(file).length, 'keys');
};
await client.langs.reload('es-ES');
```

### Automatic reload with the external @slipher/watcher runner

External package — verify the installed version and that its `Watcher` constructor still matches the `WatcherOptions` shape in your target project.

```bash
pnpm add -D @slipher/watcher
```

```ts title="src/watcher.ts"
import { join } from 'node:path';
import { Watcher } from '@slipher/watcher'; // Watcher class is external; not in seyfert-core

const watcher = new Watcher({
  filePath: join(process.cwd(), 'dist', 'index.js'), // built bot entry the worker runs
  srcPath: join(process.cwd(), 'src'),               // dir to watch
  transpileCommand: 'tsc --project tsconfig.json',   // rebuild step on change
});

await watcher.spawnShards();
```

```json title="package.json"
{ "scripts": { "watch": "pnpm build && node dist/watcher.js" } }
```

## Common patterns / gotchas

- `components.reload` takes a PATH; commands/events/langs take a NAME/locale. Mixing them up silently returns `null`.
- Reload depends on `__filePath` being set during load. Items registered programmatically (not loaded from a directory) have no file path, so `reload` returns `null` and `reloadAll` skips them.
- Use `reloadAll(false)` in a developer command so one broken file does not abort reloading the rest; use the default `reloadAll(true)` only when you want a hard stop on the first error.
- Reload mutates the live handler registries (splices/replaces entries in place). Never expose a reload command publicly — gate it with an owner-only middleware or guild check.
- Reload does NOT re-upload application commands to Discord; it only swaps the in-memory handler. To push command changes to Discord you still call the upload flow (e.g. on `botReady`/`uploadCommands`).
- `editOrReply(body)` / `write(body)` return `void` in v5 unless you pass the response flag (`true`) — do not assume you get a `Message` back from the reload command's reply.
- Reload is a Node/Bun worker-thread feature; it is unavailable under Cloudflare Workers / HTTP-on-edge (`isCloudflareWorker()` => `process.platform === 'browser'`).

## Doc vs Source Corrections

- Upstream `reloadAll()` shows no args -> src has `reloadAll(stopIfFail = true)`; passing `false` continues past per-file failures (all four handlers).
- Upstream uses `client.events?.reloadAll()` and `client.langs?.reloadAll()` with optional chaining but `client.commands` / `client.components` without -> src initializes all four (`langs`, `commands`, `components` at base.ts:182-184; `events` at base.ts:197) as concrete instances, so `?.` is unnecessary and inconsistent.
- Docs omit that reload THROWS `SeyfertError('RELOAD_NOT_SUPPORTED')` under Cloudflare Workers for every handler.
- Docs show only `components.reloadAll()` -> src also has `components.reload(path: string)`, keyed by FILE PATH (matching `__filePath`), not by a component name like commands/events/langs.
- Docs present `@slipher/watcher` as fully external -> src actually owns the integration contract: `WatcherOptions` / `WatcherPayload` / `WatcherSendToShard` (common/bot/watcher.ts) and the `__USING_WATCHER__` / `parentPort` handling in client.ts:127. Only the `Watcher` class is external.
- `langs.reload` fires an `onReload?.(lang, file)` callback hook not mentioned in docs (langs/handler.ts:95).

## Source Anchors

- src/commands/handler.ts (reload:41, reloadAll:77)
- src/components/handler.ts (reload:341, reloadAll:368)
- src/events/handler.ts (reload:430, reloadAll:448)
- src/langs/handler.ts (reload:77, reloadAll:99; onReload:95)
- src/client/base.ts (handler instances: langs:182, commands:183, components:184, events:197)
- src/client/client.ts (watcher parentPort integration:121-143)
- src/common/bot/watcher.ts (WatcherOptions/WatcherPayload/WatcherSendToShard)
- src/common/it/utils.ts (isCloudflareWorker:313); src/common/it/error.ts (RELOAD_NOT_SUPPORTED:102; SeyfertError.is:37-39)
- src/index.ts (root re-exports; common is selectively re-exported, watcher types excluded)

## Agent Guidance

- Use the per-handler `reload`/`reloadAll` APIs for dev hot-reload; prefer `reloadAll(false)` in a developer command so one broken file does not abort reloading the rest.
- Reload relies on `__filePath` being set during load — items registered programmatically without a file path return `null` from `reload` and are skipped by `reloadAll`.
- Never expose a reload command publicly; gate it (owner-only middleware / guild check). Reload mutates the live handler registries.
- Do NOT call reload in Cloudflare Worker / HTTP-on-edge deployments — it throws `RELOAD_NOT_SUPPORTED`. Catch and narrow with `SeyfertError.is(error, 'RELOAD_NOT_SUPPORTED')`.
- `components.reload` takes a path, the others take a name/locale — easy to get wrong.
- v5 middleware has no `pass()`; use `stop('reason')` to deny and `next()` to continue. The owner-gate example above reflects this.
- `@slipher/watcher` is external: confirm its installed version and that its `Watcher` constructor still matches the `WatcherOptions` shape (`filePath`, `srcPath`, `transpileCommand`) in the target project. The watcher runs the bot in a worker thread (`__USING_WATCHER__`) and proxies gateway payloads, so the gateway stays connected across restarts.
