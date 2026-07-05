# Understanding 'declare module'

Original source URL: https://seyfert-web-git-seyfert-v5-tiramisulabs.vercel.app/docs/learn/getting-started/declare-module
Coverage reference: setup-runtime.md
Verification status: Source-verified (the authoritative Seyfert source)

## Page Summary

Seyfert relies on TypeScript module augmentation (`declare module 'seyfert'`) to wire project-specific types: which client you run, your middlewares, langs, context extensions, command props, runtime config, and folder locations. In v5 the client/middlewares/langs/plugins registrations are unified into ONE `SeyfertRegistry` interface; the other extension points (`GlobalMetadata`, `ExtendContext`, `ExtraProps`, `InternalOptions`, `ExtendedRC`, `ExtendedRCLocations`, `CustomStructures`) remain separate augmentable interfaces. All helper types (`ParseClient`, `ParseLocales`, `ParseGlobalMiddlewares`) import from the root `'seyfert'`. Convention: keep one `declare module 'seyfert'` block in `src/index.ts` (it must be inside `tsconfig.json` include scope, or every type silently falls back to its default — `{}` / `BaseClient` / `false`).

## Key APIs (verified)

- `interface SeyfertRegistry {}` — central registration interface. `src/client/plugins/types.ts:32` (re-exported via `src/client/plugins.ts` and root). Recognized keys read by core:
  - `client` -> drives `UsingClient`. `src/commands/applications/shared.ts:30-43` (`RegisteredClient` / `UsingClient`).
  - `middlewares` -> drives `RegisteredMiddlewares`. `src/commands/decorators.ts:14-19`.
  - `langs` -> drives `DefaultLocale`. `src/commands/applications/shared.ts:26`.
  - `plugins` -> drives `RegisteredPlugins`. `src/client/plugins/types.ts:286-291` (NOT documented in MDX).
- `type ParseClient<T extends BaseClient> = T & { readonly [__seyfertClientType]?: T }` — `src/commands/applications/shared.ts:48`. Brands a client type so `UsingClient` resolves to it.
- `type UsingClient = BaseClient & RegisteredPluginExtension & RegisteredClient` — `src/commands/applications/shared.ts:43`. DERIVED type; do not augment directly — register via `SeyfertRegistry.client`.
- `type DefaultLocale = SeyfertRegistry extends { langs: infer L extends Record<string, any> } ? L : {}` — `src/commands/applications/shared.ts:26`. DERIVED; register via `SeyfertRegistry.langs`.
- `type RegisteredMiddlewares = SeyfertRegistry extends { middlewares: infer M extends Record<string, AnyMiddlewareContext> } ? M : {}` — `src/commands/decorators.ts:14`. DERIVED; register via `SeyfertRegistry.middlewares`. `ParseMiddlewares` REMOVED in v5 — `typeof middlewares` is already the right shape.
- `type ParseLocales<T extends Record<string, any>> = T` — `src/langs/router.ts:62`.
- `type ParseGlobalMiddlewares<T extends Record<string, AnyMiddlewareContext>> = { [K in keyof T]: MetadataMiddleware<T[K]> }` — `src/commands/applications/shared.ts:49`.
- `interface GlobalMetadata {}` — `src/commands/applications/shared.ts:25`. Augment with `extends ParseGlobalMiddlewares<...>`.
- `interface ExtendContext extends RegisteredPluginContext {}` — `src/commands/applications/shared.ts:27`. Plugin ctx contributions already merge here; augment with your `extendContext` return type.
- `function extendContext<T extends {}>(cb): cb` — `src/index.ts:119`.
- `interface ExtraProps {}` — `src/commands/applications/shared.ts:28`. Used by `@Declare({ props })` and `commands.defaults.props`.
- `interface InternalOptions {}` — `src/commands/applications/shared.ts:52`.
  - `withPrefix` -> `InferWithPrefix` (`src/commands/applications/shared.ts:23`).
  - `asyncCache` -> changes `ReturnCache<T>` to `Promise<T>` (`src/cache/index.ts`).
- `interface ExtendedRC {}` -> merged into `RC`. `src/commands/applications/shared.ts:46`, `src/client/base.ts`.
- `interface ExtendedRCLocations {}` -> merged into `RCLocations`. `src/commands/applications/shared.ts:47`.
- `interface CustomStructures {}` — `src/commands/applications/shared.ts:53`. Override a built-in structure class with your own subclass; read via `InferCustomStructure<T, N>` (`src/client/transformers.ts:182`).
- `BaseClient.getRC()` — returns `ResolvedRC` with your `ExtendedRC` fields plus `locations` carrying `ExtendedRCLocations`.
- Client classes `Client`, `HttpClient`, `WorkerClient` all export from root (`src/client/index.ts`).
- `RegisteredPlugins = SeyfertRegistry extends { plugins: infer T extends readonly AnySeyfertPlugin[] } ? T : readonly []` — `src/client/plugins/types.ts:286`. Feeds `RegisteredPluginExtension` (client extension), `RegisteredPluginContext` (ctx extension), `RegisteredPluginMiddlewares`.

## Code Examples (verified)

Clients — pick exactly one:
```ts
import type { ParseClient, Client, HttpClient, WorkerClient } from 'seyfert';

declare module 'seyfert' {
    interface SeyfertRegistry {
        client: ParseClient<Client<true>>;       // Gateway
        // client: ParseClient<HttpClient>;        // HTTP
        // client: ParseClient<WorkerClient<true>>; // Worker
    }
}
```

Middlewares (register the object directly; must be `Record<string, AnyMiddlewareContext>`):
```ts
// from './middlewares' you export e.g. `export const middlewares = { ... }`
import type * as allMiddlewares from './middlewares';

declare module 'seyfert' {
    interface SeyfertRegistry { middlewares: typeof allMiddlewares }
}
```

Global middlewares (separate interface, uses `ParseGlobalMiddlewares`):
```ts
import type * as globalMiddlewares from './globalMiddlewares';
import type { ParseGlobalMiddlewares } from 'seyfert';

declare module 'seyfert' {
    interface GlobalMetadata extends ParseGlobalMiddlewares<typeof globalMiddlewares> {}
}
```

Languages (base lang default export):
```ts
import type * as defaultLang from './langs/en';
import type { ParseLocales } from 'seyfert';

declare module 'seyfert' {
    interface SeyfertRegistry { langs: ParseLocales<typeof defaultLang> }
}
```

Context extension:
```ts
import { extendContext } from 'seyfert';

const context = extendContext(ctx => ({ otter: 'cute' }));

declare module 'seyfert' {
    interface ExtendContext extends ReturnType<typeof context> {}
}
```

Command props:
```ts
import { Client, Declare, Command } from 'seyfert';

const client = new Client({ commands: { defaults: { props: { category: 'none' } } } });

@Declare({ name: 'test', description: 'test command', props: { onlyForAdmins: true } })
class Test extends Command {}

declare module 'seyfert' {
    interface ExtraProps {
        onlyForAdmins?: boolean;
        disabled?: true;
        category?: string;
    }
}
```

Internal options:
```ts
declare module 'seyfert' {
    interface InternalOptions {
        withPrefix: true;  // context may carry a message; .interaction becomes optional
        asyncCache: false; // true when adapter returns promises (e.g. @slipher/redis-adapter)
    }
}
```

Config + locations + reading at runtime:
```ts
declare module 'seyfert' {
    interface ExtendedRC { developers: string[] }
    interface ExtendedRCLocations { models: string }
}

const rc = await client.getRC();
console.log(rc.developers);        // ExtendedRC property
console.log(rc.locations.models);  // ExtendedRCLocations property
```

Plugins (NEW, not in MDX) — register the plugin tuple so its extensions/ctx/middlewares are typed everywhere:
```ts
import type { economyPlugin } from './plugins';

declare module 'seyfert' {
    // tuple order does not matter for types; list every plugin you pass to the client
    interface SeyfertRegistry { plugins: [typeof economyPlugin] }
}
```

## Recipes / Worked Examples

### A complete `src/index.ts` (Gateway bot, one block)

The canonical single-block setup. Everything the bot's types depend on lives here so editors resolve `UsingClient`, middleware metadata, langs and config in every file.

```ts
import { Client, type ParseClient, type ParseLocales, type ParseGlobalMiddlewares } from 'seyfert';
import type * as allMiddlewares from './middlewares';
import type * as globalMiddlewares from './globalMiddlewares';
import type * as defaultLang from './langs/en';

const client = new Client();

client.start().then(() => client.uploadCommands({ cachePath: './commands.json' }));
// NOTE: start() does NOT upload commands in v5 — call uploadCommands() yourself.

declare module 'seyfert' {
    interface SeyfertRegistry {
        client: ParseClient<Client<true>>;
        middlewares: typeof allMiddlewares;
        langs: ParseLocales<typeof defaultLang>;
    }
    interface GlobalMetadata extends ParseGlobalMiddlewares<typeof globalMiddlewares> {}
    interface ExtraProps { category?: string; onlyForAdmins?: boolean }
    interface ExtendedRC { developers: string[] }
}
```

### Using the registered client type anywhere

After `SeyfertRegistry.client` is set, import `UsingClient` instead of hand-writing the union. This is what command/event handlers receive as `client`.

```ts
import type { UsingClient } from 'seyfert';

export function reportError(client: UsingClient, message: string) {
    // client is your Client<true> & any plugin extensions — fully typed
    return client.logger.warn(message);
}
```

### Worker bot (multi-process)

Only the `client` line changes; everything else (middlewares, langs, props) is identical.

```ts
import type { ParseClient, WorkerClient } from 'seyfert';

declare module 'seyfert' {
    interface SeyfertRegistry { client: ParseClient<WorkerClient<true>> }
}
```

### Plugin that extends the client + context (typed end to end)

Register the plugin in `SeyfertRegistry.plugins` and its `client`/context contributions become typed without any manual `ExtendContext`/client augmentation — those are derived from `RegisteredPlugins`.

```ts
// plugins.ts
import { createPlugin, createMiddleware } from 'seyfert';

class EconomyService { connect() {} close() {} balance(_id: string) { return 0; } }

const balanceGuard = createMiddleware<void>(({ next }) => next());

export const economyPlugin = createPlugin({
    name: 'economy',
    client: { economy: () => new EconomyService() }, // -> client.economy is typed
    register(api) {
        api.middlewares.add('balanceGuard', balanceGuard, { global: true });
    },
    setup: client => client.economy.connect(),
    teardown: client => client.economy.close(),
});

// index.ts
import { Client, definePlugins } from 'seyfert';
import { economyPlugin } from './plugins';

const client = new Client({ plugins: definePlugins(economyPlugin) });

declare module 'seyfert' {
    interface SeyfertRegistry {
        client: import('seyfert').ParseClient<Client<true>>;
        plugins: [typeof economyPlugin];
    }
}
// Now `client.economy` and `ctx.economy` (if the plugin contributes context) are typed everywhere.
```

### Async cache (Redis adapter) — flip `asyncCache`

When the cache adapter returns promises, set `asyncCache: true` so `ReturnCache<T>` becomes `Promise<T>` and you `await` reads instead of getting a sync value.

```ts
// @slipher/redis-adapter is EXTERNAL — verify version in target project.
declare module 'seyfert' {
    interface InternalOptions { asyncCache: true }
}

// downstream: cache getters now return promises
const guild = await client.cache.guilds?.get(guildId);
```

### Prefix (text) commands — flip `withPrefix`

`withPrefix: true` makes BOTH `ctx.interaction` and `ctx.message` optional across contexts (the context may originate from a message). Only enable it if you actually run text commands.

```ts
declare module 'seyfert' {
    interface InternalOptions { withPrefix: true }
}

// in a command handler:
if (ctx.interaction) { /* slash invocation */ }
else if (ctx.message) { /* prefix invocation */ }
```

### Override a built-in structure (`CustomStructures`)

Subclass a structure and register it; Seyfert's transformers will hand you your subclass type (`InferCustomStructure`).

```ts
import { GuildMember } from 'seyfert';

class MyGuildMember extends GuildMember {
    get isOwner() { return this.id === this.guild()?.ownerId; }
}

declare module 'seyfert' {
    interface CustomStructures { GuildMember: MyGuildMember }
}
```

## Common patterns / gotchas

- ALWAYS brand the client with `ParseClient<...>`. Registering a bare `Client<true>` to `SeyfertRegistry.client` will not resolve `UsingClient` to your type (the `__seyfertClientType` brand check in `shared.ts:30-42` falls back to `BaseClient`).
- Do NOT augment `UsingClient`, `RegisteredMiddlewares`, or `DefaultLocale` directly in v5 — they are derived from `SeyfertRegistry`. The old v4/early-v5 pattern (`interface UsingClient extends ParseClient<...>`, `interface RegisteredMiddlewares extends ParseMiddlewares<...>`) is gone.
- `ParseMiddlewares` was REMOVED. Use the bare `typeof middlewares` — `createMiddleware(...)` already returns the metadata-bearing shape.
- The `declare module 'seyfert'` block must be inside `tsconfig.json`'s include scope (e.g. `src/index.ts`). If it's outside, every interface silently falls back to its default and you get `BaseClient`/no middleware metadata with no error.
- Use ONE registration per key. Multiple `SeyfertRegistry` blocks merge via interface declaration merging, but duplicate keys conflict — keep `client`/`middlewares`/`langs`/`plugins` declared once.
- `start()` does NOT upload commands. After `client.start()` call `client.uploadCommands(...)` to register slash commands with Discord.
- `Client<true>` (Ready=true) makes ready-only getters non-nullable; use it in the registry so handlers see a ready client.
- Type rename: `APIGuildCreateOverwrite` -> `RESTAPIGuildCreateOverwrite` (`seyfert/lib/types`). `EntryPointInteraction.withReponse` typo removed -> `withResponse`.

## Doc vs Source Corrections

- Docs/MDX and src agree on `SeyfertRegistry` for `client`, `middlewares`, `langs`. The older seyfert@5.0.0 release-blog pattern (augmenting `interface UsingClient`, `interface RegisteredMiddlewares`, `interface DefaultLocale` directly) is GONE on this branch — all three are derived from `SeyfertRegistry` and must not be augmented directly (`shared.ts:43`/`26`, `decorators.ts:14`).
- `ParseMiddlewares` no longer exists in src — register `middlewares: typeof allMiddlewares` directly (field constrained to `Record<string, AnyMiddlewareContext>`). `ParseGlobalMiddlewares` still exists and is still correct for `GlobalMetadata`.
- MDX omits the `plugins` key on `SeyfertRegistry`, which src supports and uses to derive client/context/middleware extensions (`types.ts:286-291`).
- MDX `ExtendContext` example shows `extends ReturnType<typeof context>`; the src default is already `extends RegisteredPluginContext`, so plugin ctx contributions merge automatically — your augmentation adds to that base.
- MDX names the config file `seyfert.config.mjs`; `getRC()` resolves any supported config extension — the file name is just an example.
- MDX does not mention `CustomStructures` (structure override interface) — present in src (`shared.ts:53`, consumed at `transformers.ts:182`).
