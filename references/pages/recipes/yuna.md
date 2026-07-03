# Yuna Parser

Original source URL: https://seyfert-web-git-seyfert-v5-tiramisulabs.vercel.app/docs/recipes/yuna
Canonical MDX (v5): content/docs/plugins/official/yuna.mdx (the page moved under `plugins/official/` on the seyfert-v5 branch).
Coverage reference: i18n-cache-recipes.md
Verification status: Source-verified (core integration points) + EXTERNAL package (yunaforseyfert) — version-verify in target project

## Page Summary

[Yuna](https://www.npmjs.com/package/yunaforseyfert) (`yunaforseyfert`) is an EXTERNAL advanced parser for prefix/text commands in Seyfert: named-option syntax, dynamic per-guild prefixes, and message watchers for follow-up input. In Seyfert v5 it ships as a Seyfert **plugin** — build the list with `definePlugins(Yuna.plugin(...))` and pass it to `new Client({ plugins })`; the plugin wires the parser + resolver for you (no `setServices` / `HandleCommand` subclass needed, the v4-era path). The Yuna parser, resolver, watchers, and `@Watch` decorator are owned by `yunaforseyfert` and must be version-verified in the target project; the integration points (`definePlugins`, `ClientOptions.plugins`, `ClientOptions.commands.prefix`/`reply`, `ParseClient`, and the `InternalOptions { withPrefix: true }` / `SeyfertRegistry` augmentations) are core and are confirmed against ./src below.

## Key APIs (verified)

- `definePlugins(...plugins)` — `src/client/plugins.ts:283-285`. Two overloads: array form `definePlugins([a, b])` and variadic `definePlugins(a, b)`. Identity helper returning the plugin tuple (const-typed: `<const TPlugins extends readonly AnySeyfertPlugin[]>`). Exported from root `seyfert` (`src/index.ts` -> `src/client/index.ts:13` -> `./plugins`).
- `ClientOptions.plugins?: TPlugins` — `src/client/client.ts:305`. Accepts a `readonly AnySeyfertPlugin[]`; defaults to `RegisteredPlugins` (derived from `SeyfertRegistry.plugins`).
- `ClientOptions.commands.prefix?: (message: MessageStructure) => Awaitable<string[]>` — `src/client/client.ts:317`. The prefix list lives on the CLIENT, not in plugin options. Async allowed (return value is awaited).
- `ClientOptions.commands.reply?: (ctx: CommandContext) => Awaitable<boolean>` — `src/client/client.ts:319`. Whether the bot replies (vs. plain send) to text commands.
- `ClientOptions.commands.deferReplyResponse?: (ctx) => Awaitable<Parameters<Message['write']>[0]>` — `src/client/client.ts:318`. (Nearby option; not used by Yuna docs but present.)
- `ParseClient<T extends BaseClient>` — `src/commands/applications/shared.ts:48` (`T & { readonly [__seyfertClientType]?: T }`). Type-only helper for the `SeyfertRegistry { client }` augmentation. Exported from root `seyfert`.
- `interface InternalOptions {}` — `src/commands/applications/shared.ts:52`. Augment with `{ withPrefix: true }` to enable prefix/text commands. Consumed type-level by `InferWithPrefix` (`shared.ts:23`: `InternalOptions extends { withPrefix: infer P } ? P : false`). NOT set by the plugin.
- `interface SeyfertRegistry {}` — `src/client/plugins/types.ts:32`. Central v5 augmentation point. Augment with `plugins: typeof plugins` (canonical for Yuna — this is what types `client.yuna`/`ctx.yuna`), and optionally `client: ParseClient<...>`, `middlewares`, `langs`. `RegisteredPlugins` is derived from it (`types.ts:286-288`).
- Plugin shape (`createPlugin`, `src/client/plugins.ts:240-265`): `{ name, client?, register?(api), setup?(client, api?), teardown?(client, api?) }` — `Yuna.plugin(...)` returns one of these. (External, but the shape it must satisfy is core.)
- EXTERNAL (yunaforseyfert, version-verify): `Yuna.plugin({ parser, resolver })`, `Yuna.watchers.find(client, { userId, command })` -> watcher with `.stop(reason)`, `@Watch({ idle, beforeCreate })`, and the injected `client.yuna` / `ctx.yuna` accessors. None of these exist in core Seyfert.

## Code Examples (verified)

### 1. Plugin setup (canonical — register on `SeyfertRegistry.plugins`)

Core integration points verified; `Yuna.plugin` is external. Registering the plugin list through `SeyfertRegistry.plugins` is what makes the plugin's `client.yuna` / `ctx.yuna` accessors typed everywhere.

```ts
import { Client, definePlugins } from 'seyfert';
import { Yuna } from 'yunaforseyfert';

const plugins = definePlugins(
    Yuna.plugin({
        parser: { syntax: { namedOptions: ['-', '--'] }, logResult: true },
        resolver: { logResult: true },
    }),
);

declare module 'seyfert' {
    interface SeyfertRegistry { plugins: typeof plugins }
    interface InternalOptions { withPrefix: true }
}

const client = new Client({
    plugins,
    commands: {
        prefix: () => ['!', '?'],
        reply: () => true,
    },
});

await client.start();
```

`InternalOptions { withPrefix: true }` is required to enable prefix/text commands — the plugin does not set it. The prefix stays on `commands.prefix`, not in plugin options.

### 2. Also keeping `ctx.client` typed (combine augmentations)

`SeyfertRegistry` is one interface — add every key you need in one `declare module` block. Add `client: ParseClient<Client<true>>` so `ctx.client` keeps its registered type (the framework otherwise erases custom client typing), alongside `plugins` and your `middlewares`.

```ts
import { Client, type ParseClient, definePlugins } from 'seyfert';
import { Yuna } from 'yunaforseyfert';

const plugins = definePlugins(Yuna.plugin({ parser: {}, resolver: {} }));
const middlewares = { /* ...createMiddleware(...) entries... */ };

declare module 'seyfert' {
    interface SeyfertRegistry {
        client: ParseClient<Client<true>>;
        plugins: typeof plugins;
        middlewares: typeof middlewares;
    }
    interface InternalOptions { withPrefix: true }
}
```

### 3. Dynamic per-guild prefix (pure core)

`commands.prefix` receives a `MessageStructure` and may be async; return `string[]`. Always include a fallback so DMs/uncached guilds still work.

```ts
import { Client } from 'seyfert';

const client = new Client({
    commands: {
        prefix: async (message) => {
            const config = message.guildId
                ? await db.getGuildConfig(message.guildId)
                : null;
            return config?.prefixes ?? ['!'];
        },
        // reply to the user's message instead of sending a fresh message
        reply: () => true,
    },
});
```

### 4. A prefix/slash command with options (core; Yuna parses the text form)

Yuna parses named-option syntax for text commands, but the command itself is a normal Seyfert command — the same class works for slash AND prefix invocation once `withPrefix: true` is set. Options use lowercase keys (v5 compile-time rule).

```ts
import {
    Command,
    Declare,
    Options,
    createStringOption,
    createUserOption,
    type CommandContext,
} from 'seyfert';

const options = {
    reason: createStringOption({ description: 'Why?', required: true }),
    target: createUserOption({ description: 'Who?', required: true }),
};

@Declare({ name: 'warn', description: 'Warn a member' })
@Options(options)
export default class WarnCommand extends Command {
    async run(ctx: CommandContext<typeof options>) {
        const { reason, target } = ctx.options;
        // text form (namedOptions ['-','--']):  !warn -target @user -reason spam
        await ctx.write({ content: `Warned ${target.username}: ${reason}` });
    }
}
```

### 5. Watchers via the `@Watch` decorator (external decorator; command pieces are core)

`@Watch` / `Yuna.watchers` are EXTERNAL. `Command`/`Declare`/`CommandContext`/`ctx.write`/`ctx.author`/`ctx.client` are core. `beforeCreate` lets you cancel a user's prior watcher before opening a new one.

```ts
import { Command, Declare, type CommandContext } from 'seyfert';
import { Watch, Yuna } from 'yunaforseyfert';

@Declare({ name: 'help', description: 'Interactive help' })
@Watch({
    idle: 60_000, // close after 60s of no follow-up
    beforeCreate(ctx) {
        const existing = Yuna.watchers.find(ctx.client, {
            userId: ctx.author.id,
            command: this,
        });
        if (existing) existing.stop('replaced');
    },
})
export default class HelpCommand extends Command {
    async run(ctx: CommandContext) {
        await ctx.write({ content: 'What do you need help with? Reply with a topic.' });
    }
}
```

## Common patterns / gotchas

- **Two independent switches.** Prefix/text commands need BOTH (1) `InternalOptions { withPrefix: true }` (type-level switch) and (2) a `commands.prefix` function (runtime prefix list). The Yuna plugin sets neither — you provide both yourself.
- **Prefix lives on the client, not the plugin.** Put `prefix`/`reply` under `ClientOptions.commands`. `Yuna.plugin(...)` only takes `parser`/`resolver` options.
- **Register via `SeyfertRegistry.plugins`** (not just passing `plugins` to the client) so `client.yuna` / `ctx.yuna` are typed. The runtime works either way; the typing does not without the augmentation.
- **One `declare module 'seyfert'` block, one `SeyfertRegistry`.** v5 collapsed `UsingClient`/`RegisteredMiddlewares`/`DefaultLocale` into `SeyfertRegistry` keys (`client`/`middlewares`/`langs`/`plugins`). Don't augment the old interfaces; `ParseMiddlewares` is gone (use bare `typeof middlewares`).
- **The command is shared.** A single `Command` class serves slash and prefix; no separate "text command" class. Keep option keys lowercase (v5 rejects non-lowercase at compile time).
- **`prefix` fallback.** Always return a non-empty array even for DMs / uncached guilds, or text commands silently won't match.
- **Version-verify Yuna option names.** `parser.syntax.namedOptions`, `resolver`, `logResult`, and watcher `idle`/`beforeCreate`/`.stop()` are doc-authoritative (yunaforseyfert), not source-verified here — confirm against the installed version.

## Doc vs Source Corrections

- v4-era draft said the extension point is a custom `HandleCommand` service via deep import -> STALE. v5 MDX and core use the **plugin** path: `Yuna.plugin(...)` built with `definePlugins` and passed to `ClientOptions.plugins`. No `setServices` / `HandleCommand` subclass for Yuna in v5. (`src/client/plugins.ts`, `src/client/client.ts:305`)
- Primary example now augments `SeyfertRegistry { plugins: typeof plugins }` to match the canonical MDX (this is what types `client.yuna`/`ctx.yuna`). Augmenting `client: ParseClient<...>` is still valid and shown as a combinable add-on (Example 2). Earlier draft augmented only `client`.
- Verification status corrected from "External package" only to "Source-verified (core) + external package": the integration surface (`definePlugins`, `commands.prefix/reply`, `ParseClient`, `InternalOptions`/`SeyfertRegistry`) is all in core and verified.
- MDX vs core: aligned. `definePlugins`, `ParseClient`, `commands.prefix`/`reply` signatures match the MDX usage exactly.

## Source Anchors

- `src/client/plugins.ts` — `createPlugin` (240-265), `definePlugins` (283-285)
- `src/client/plugins/types.ts` — `SeyfertRegistry` (32), `AnySeyfertPlugin` (259), `RegisteredPlugins` (286), `SeyfertPlugin` shape with `register`/`setup`/`teardown` (509-511)
- `src/client/client.ts` — `ClientOptions.plugins` (305); `commands.prefix` (317) / `deferReplyResponse` (318) / `reply` (319)
- `src/commands/applications/shared.ts` — `InferWithPrefix` (23), `ParseClient` (48), `InternalOptions` (52)
- `src/index.ts` -> `src/client/index.ts:13` (root barrel re-export of `./plugins`, incl. `definePlugins`)

## Agent Guidance

- Use this page when a user wants prefix/text commands, named-option parsing, dynamic per-guild prefixes, or interactive message watchers. For v5, ALWAYS use the plugin path (`Yuna.plugin` + `definePlugins`) — do NOT recommend the v4 `setServices`/`HandleCommand`-subclass approach.
- Walk users through both required switches (`withPrefix: true` AND `commands.prefix`) plus the `SeyfertRegistry { plugins: typeof plugins }` augmentation — missing any one is the most common breakage.
- The same `Command` class handles slash + prefix; do not generate a separate text-command type.
- GOTCHA: `Yuna.plugin`, `@Watch`, `Yuna.watchers`, and `client.yuna`/`ctx.yuna` are NOT in core Seyfert — they come from `yunaforseyfert`. Their option names are doc-authoritative; tell users to verify the installed `yunaforseyfert` version. Everything under `seyfert` root imports (`definePlugins`, `ParseClient`, `commands.prefix/reply`, `InternalOptions`, `SeyfertRegistry`, `Command`/`Declare`/`Options`/`create*Option`) IS source-verified above.
