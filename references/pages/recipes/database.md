# Database Integration

Original source URL: https://seyfert-web-git-seyfert-v5-tiramisulabs.vercel.app/docs/recipes/database
Coverage reference: i18n-cache-recipes.md
Verification status: Source-verified (core)

## Page Summary

Seyfert is database-agnostic — attach any client (Prisma, Drizzle, Mongoose, …).
The idiomatic v5 pattern is a small **plugin** (`createPlugin`) that owns the
connection and exposes it through the plugin `client` map as `client.db`, so every
command, event, and middleware reaches the same typed instance with no manual
`Client` augmentation. The same plugin can hang per-interaction helpers off the
command context via the `ctx` map (e.g. `ctx.user()`). Registering the plugin
tuple under `declare module 'seyfert' { interface SeyfertRegistry { plugins } }`
is what makes `client.db` and `ctx.*` typed everywhere. This is a pattern recipe;
the actual DB driver is target-project-specific (verify its version there).

## Key APIs (verified)

All of the following are exported from the root `'seyfert'` (via `src/client/index.ts`
→ `src/client/plugins.ts`; root barrel `src/index.ts` does `export * from './client'`).

- `createPlugin(plugin)` — identity helper that returns its input with inferred
  generics; no runtime transform (`return plugin`). src/client/plugins.ts:240-265.
- `definePlugins(...plugins | plugins[])` — returns the plugin tuple; accepts either
  a single array or variadic args. src/client/plugins.ts:283-289.
- `createPluginFactory({ defaults, validate?, factory })` — builds an
  `(options?) => plugin` factory. src/client/plugins.ts:267-281.
- `createSharedKey`, `PluginOrder` (enum) — also re-exported. src/client/plugins.ts:42-43.
- `interface SeyfertRegistry {}` — augmentation point; add `plugins: typeof plugins`.
  src/client/plugins/types.ts:32.

Plugin shape — `CreatePluginInputBase` / `SeyfertPlugin` (src/client/plugins.ts:165-184,
src/client/plugins/types.ts:492-512):
- `name: string` (required), optional `instanceId`, `version`, `imports`, `requires`, `meta`.
- `client?: PluginClientMap<E, I>` — `{ [K]: (client) => T[K] }`. Adds typed props to
  the client. src/client/plugins/types.ts:244-246.
- `ctx?: PluginContextMap<C, I, E>` — `{ [K]: (interaction, client) => T[K] }`. Adds
  per-interaction props to command context. The factory runs **synchronously** before
  `run()`; return an async function if you need an async lookup. src/client/plugins/types.ts:250-256.
- `middlewares?`, `globalMiddlewares?`.
- `options?(current)`, `register?(api)` — **synchronous** (`=> void`).
- `setup?(client, api?)` / `teardown?(client, api?)` — `Awaitable<void>` (async OK).
  src/client/plugins/types.ts:508-511.

Lifecycle (src/client/plugins.ts): `register` (sync) → `setup` (async, before the bot
handles anything) → … → `teardown` (async, reverse order). Open connections in `setup`,
close them in `teardown`. If a `setup` throws, already-completed plugins are torn down and
a `SeyfertPluginError` / `SeyfertPluginAggregateError` is raised (setupClientPlugins, ~line 656).

## Code Examples (verified)

### 1. The database plugin (root import; `client` + `ctx` maps, `setup`/`teardown`)

```ts
import { createPlugin } from 'seyfert';

// Replace with your real client (Prisma, Drizzle, …)
declare class DatabaseClient {
    connect(): Promise<void>;
    disconnect(): Promise<void>;
    findUser(id: string): Promise<{ id: string; balance: number } | null>;
    createUser(id: string): Promise<void>;
    addBalance(id: string, amount: number): Promise<void>;
}

const db = new DatabaseClient();

export const databasePlugin = createPlugin({
    name: 'database',
    client: {
        db: () => db, // exposed as client.db, typed everywhere
    },
    ctx: {
        // factory is sync; return an async fn so `await ctx.user()` does the lookup
        user: interaction => async () => {
            const { db } = interaction.client;
            const record =
                (await db.findUser(interaction.user.id)) ??
                { id: interaction.user.id, balance: 0 };
            return {
                ...record, // id, balance
                add: (amount: number) => db.addBalance(record.id, amount),
            };
        },
    },
    async setup(client) {
        await client.db.connect();
    },
    async teardown(client) {
        await client.db.disconnect();
    },
});
```

### 2. Register the plugin (typing comes from SeyfertRegistry)

```ts
import { Client, definePlugins } from 'seyfert';
import { databasePlugin } from './plugins/database';

const plugins = definePlugins(databasePlugin);

declare module 'seyfert' {
    interface SeyfertRegistry { plugins: typeof plugins }
}

const client = new Client({ plugins });
```

### 3. Using it in a command

```ts
import { Command, Declare, type CommandContext } from 'seyfert';

@Declare({ name: 'balance', description: 'Check your balance' })
export default class BalanceCommand extends Command {
    async run(ctx: CommandContext) {
        const user = await ctx.user(); // scoped to the caller
        await user.add(50);            // daily bonus
        await ctx.write({ content: `Your balance: **${user.balance}** coins` });
    }
}
```

### 4. Reaching the connection without a context (events)

```ts
import { createEvent } from 'seyfert';

export default createEvent({
    data: { name: 'guildMemberAdd' },
    async run(member, client) {
        await client.db.createUser(member.id);
    },
});
```

`createEvent` shape verified: `{ data: { name; once? }, run }` (src/index.ts:68-74).

### 5. Configurable plugin via `createPluginFactory` (connection string as an option)

Use a factory when the same DB plugin must accept per-bot options (URL, pool size).
`createPluginFactory({ defaults, validate?, factory })` returns `(options?) => plugin`;
`options` is `Partial<defaults>` merged over `defaults`, and `validate` throws before
the plugin is built (wrapped in a `SeyfertPluginError`). src/client/plugins.ts:267-281.

```ts
import { createPlugin, createPluginFactory } from 'seyfert';

declare class DatabaseClient {
    constructor(url: string, opts: { poolSize: number });
    connect(): Promise<void>;
    disconnect(): Promise<void>;
}

interface DbOptions {
    url: string;
    poolSize: number;
}

export const databasePlugin = createPluginFactory<DbOptions, ReturnType<typeof build>>({
    defaults: { url: '', poolSize: 5 },
    validate(o) {
        if (!o.url) throw new Error('database plugin: `url` is required');
    },
    factory: build,
});

function build(o: DbOptions) {
    const db = new DatabaseClient(o.url, { poolSize: o.poolSize });
    return createPlugin({
        name: 'database',
        client: { db: () => db },
        setup: () => db.connect(),
        teardown: () => db.disconnect(),
    });
}

// usage: definePlugins(databasePlugin({ url: process.env.DATABASE_URL! }))
```

### 6. A DB-backed middleware shipped *with* the plugin (`register` + `api.middlewares.add`)

The plugin can register its own middleware so consumers get the guard for free.
`register(api)` is **synchronous**; `api.middlewares.add(name, mw, { global })` mounts
it (set `global: true` to run on every command). `PluginMiddlewareOptions.global`
verified at src/client/plugins/types.ts:354-355; `api.middlewares.add` at :431-437.

```ts
import { createPlugin, createMiddleware } from 'seyfert';

// `stop()` with no arg = silent skip; `stop('reason')` = deny -> onMiddlewaresError.
const requireAccount = createMiddleware<void>(async ({ context, next, stop }) => {
    const exists = await context.client.db.findUser(context.author.id);
    if (!exists) return stop('You need an account first. Run /register.');
    return next();
});

export const databasePlugin = createPlugin({
    name: 'database',
    client: { db: () => db },
    register(api) {
        api.middlewares.add('requireAccount', requireAccount, { global: true });
    },
    setup: client => client.db.connect(),
    teardown: client => client.db.disconnect(),
});
```

If you do NOT mark it global, reference it per-command with the typed helper
`@Middlewares(['requireAccount'])` (the name is inferred once the plugin is in
`SeyfertRegistry.plugins`).

### 7. Cached lookup helper on the context map

The `ctx` factory runs once per interaction; cache a read in the closure so two
`await ctx.user()` calls in the same command don't hit the DB twice.

```ts
import { createPlugin } from 'seyfert';

export const databasePlugin = createPlugin({
    name: 'database',
    client: { db: () => db },
    ctx: {
        user: interaction => {
            let cached: { id: string; balance: number } | undefined;
            return async () => {
                cached ??=
                    (await interaction.client.db.findUser(interaction.user.id)) ??
                    { id: interaction.user.id, balance: 0 };
                return cached;
            };
        },
    },
});
```

## Common patterns / gotchas

- **Two typed access paths, one source.** `client.<key>` (from the `client` map) is the
  app-wide singleton; `ctx.<key>()` (from the `ctx` map) is a per-interaction helper. Both
  become typed only after the plugin tuple is in `SeyfertRegistry.plugins`.
- **`ctx` factory is synchronous.** It cannot be `await`ed, so an async lookup MUST be a
  returned function: `interaction => async () => {...}`, called as `await ctx.user()`. A
  factory that returns the awaited value directly will not compile/behave.
- **Open in `setup`, close in `teardown`.** `register` is sync and runs before the client is
  fully wired — never `await` a connection there. `teardown` runs in reverse registration
  order; if a `setup` throws, already-set-up plugins are torn down and a
  `SeyfertPluginError`/`SeyfertPluginAggregateError` is raised.
- **Name collisions throw.** `client`/`ctx`/`shared`/cache-resource names are guarded
  (`assertSafePluginResourceName`); pick keys that don't clash with reserved client members.
- **Driver is out of scope.** Prisma/Drizzle/Mongoose/postgres.js are target-project deps —
  `DatabaseClient` above is a placeholder; verify the real driver's API and version there.
- **Sharing across workers:** under the worker/sharding manager each worker is its own
  process, so each gets its own `client.db` instance — point them all at the same DB server,
  don't expect a shared in-process connection.

## Doc vs Source Corrections

- None. Every API the MDX uses (`createPlugin`, `definePlugins`, the `client`/`ctx`
  maps, `setup`/`teardown`, `SeyfertRegistry`) matches src exactly, and all import from
  the root `'seyfert'`. The MDX claim that `register` is synchronous while `setup` is
  async is confirmed (`register?(api): void` vs `setup?(...): Awaitable<void>`,
  src/client/plugins/types.ts:509-510).
- Draft note was thin (only listed structure) but not wrong; this note adds the verified
  signatures and lifecycle detail.

## Source Anchors

- src/client/plugins.ts (createPlugin:240, definePlugins:283, createPluginFactory:267,
  setupClientPlugins:656, teardownClientPlugins:717)
- src/client/plugins/types.ts (SeyfertRegistry:32, PluginClientMap:244, PluginContextMap:250,
  SeyfertPlugin:492)
- src/client/plugins/types.ts (SeyfertPluginApi.middlewares.add:431, PluginMiddlewareOptions.global:354, SeyfertPlugin.register:509/setup:510/teardown:511)
- src/client/plugins/api.ts (register-time `api` surface, teardown restrictions)
- src/commands/applications/options.ts:213 (createMiddleware)
- src/client/index.ts:13, src/index.ts:1 + :68 (root re-exports; createEvent)

## Agent Guidance

- Use this when wiring any datastore into a Seyfert v5 bot. Prefer the plugin pattern over
  augmenting `Client` by hand — `client` map gives app-wide typed access, `ctx` map gives
  ergonomic per-interaction helpers, and `SeyfertRegistry.plugins` propagates both types.
- Gotcha: the `ctx` factory runs **synchronously** before `run()`. If you need an async DB
  read, the factory must return a function (`(interaction) => async () => {...}`), then call
  `await ctx.helper()` in the command. Returning the awaited value directly from the factory
  will not work — the factory itself cannot be awaited.
- Open the connection in `setup`, not `register` (register is sync and runs before the client
  is fully wired). Close it in `teardown` for clean shutdown.
- `client` map keys must not collide with reserved client members; `ctx`/`shared`/cache
  resource names are guarded (`assertSafePluginResourceName`, reserved-name sets in
  src/client/plugins/api.ts) and throw on conflict.
- The DB driver itself (Prisma/Drizzle/Mongoose/…) is NOT part of seyfert-core — verify its
  version and API in the target project. `DatabaseClient` here is a placeholder.
