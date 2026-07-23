# Cooldown (@slipher/cooldown)

Original source URL: https://seyfert-web-git-seyfert-v5-tiramisulabs.vercel.app/docs/plugins/official/cooldown

Coverage reference: plugins.md

Verification status: Source-verified against current `tiramisulabs/extra/packages/cooldown` + Seyfert core

## Contents

- [Page summary](#page-summary)
- [Key APIs](#key-apis-verified)
- [Code examples](#code-examples-verified)
- [Recipes and gotchas](#recipes--common-patterns)
- [Source anchors](#source-anchors)

## Page Summary

`@slipher/cooldown` is an EXTERNAL plugin (not part of core Seyfert) that adds per-command
cooldowns scoped per user, guild, channel, or globally. You declare a cooldown with the
`@Cooldown` decorator (or its scope shortcuts), and either let the bundled `cooldown`
middleware gate commands automatically or drive the `CooldownManager` (`client.cooldown` /
`ctx.cooldown`) directly. Storage is backed by the client cache adapter, with an optional
atomic Redis-style `eval` fast path for `consume`. Requires Seyfert v5 and keeps `seyfert` as a
peer dependency. Verify the installed version in consumer projects.

## Key APIs (verified)

CORE seyfert surfaces the plugin plugs into (all verified in ./src, all importable from `'seyfert'`):

- `definePlugins(...plugins)` / `definePlugins(plugins[])` — variadic OR single-array form,
  returns the plugin tuple unchanged (identity for typing). `src/client/plugins.ts:283-285`.
- `createPlugin({ name, imports?, requires?, client?, ctx?, middlewares?, globalMiddlewares?, register?, setup?, teardown?, options?, meta? })`
  — the integration shape an external plugin like cooldown is built with. `src/client/plugins.ts:240-265`.
- `interface SeyfertRegistry {}` — empty, augmentation target. Augment `plugins` to type the
  client (`RegisteredPlugins = SeyfertRegistry['plugins']`) and `middlewares` to register
  middleware names. `src/client/plugins/types.ts:32`, `:286`, `:291`.
- `Middlewares(cbs: readonly MiddlewareKey[])` decorator — accepts middleware names including
  plugin-contributed ones (`RegisteredPluginMiddlewares`). `src/commands/decorators.ts:188`,
  `:14-20`.

EXTERNAL `@slipher/cooldown` API (verified against current `extra` source):

- `cooldown(options?)` factory → a Seyfert plugin. Options: `{ middleware?: boolean | { global?: boolean; name?: string; message?: string | ((result, ctx) => string) } }`.
- `Cooldown` decorator + scope shortcuts:
  - `Cooldown(props: CooldownProps)`
  - `Cooldown.user(interval, { uses?, group? }?)`
  - `Cooldown.guild(interval, { uses?, group? }?)`
  - `Cooldown.channel(interval, { uses?, group? }?)`
  - `Cooldown.global(interval, { uses?, group? }?)`
  - `Cooldown.custom(resolver: (ctx) => string | undefined, interval, { uses?, group? }?)`
- `interface CooldownProps { type?: 'user'|'guild'|'channel'|'global'|((ctx: AnyContext) => string | undefined); interval: number; uses?: number; group?: string }` — `type` defaults to `'user'`, `uses` defaults to `1`.
- `CooldownManager` exposed as `client.cooldown` and `ctx.cooldown`:
  - `consume()` / `consume({ cost? })` in a live handler; explicit `consume({ name, target, guildId?, cost? })`
  - `check()` / `check({ cost? })` in a live handler; explicit `check({ name, target, guildId?, cost? })`
  - `reset()` in a live handler; explicit `reset({ name, target, guildId? })`
  - implicit forms throw outside a live cooldown scope
- `type CooldownResult` — discriminated union on `allowed`; fields `remainingMs`, `retryAfter: Date`, `limit`, `remainingUses`, `key`. `check`/`consume` return `undefined` when the command has no cooldown.
- `type CooldownMiddlewares<Name extends string>` — helper to register middleware type by name.
- `AtomicCooldownAdapter` marker interface: `{ supportsAtomicCooldowns: true; eval(script, keys, args) }`.

## Code Examples (verified)

Install plugin + register types (core `definePlugins` / `SeyfertRegistry` verified):

```ts
import { Client, definePlugins } from 'seyfert';
import { cooldown } from '@slipher/cooldown';

const plugins = definePlugins(
    cooldown({ middleware: true }),
);

declare module 'seyfert' {
    interface SeyfertRegistry { plugins: typeof plugins }
}

const client = new Client({ plugins });
```

Declare a cooldown via shortcut:

```ts
import { Command, type CommandContext, Declare } from 'seyfert';
import { Cooldown } from '@slipher/cooldown';

@Declare({ name: 'ping', description: 'Ping' })
@Cooldown.user(5_000) // 1 use per user every 5s
export default class PingCommand extends Command {
    async run(ctx: CommandContext) {
        await ctx.write({ content: 'Pong!' });
    }
}
```

Raw decorator with all fields:

```ts
@Cooldown({ type: 'user', interval: 5_000, uses: 3, group: 'moderation' })
```

Assign middleware by name (core `Middlewares` decorator verified):

```ts
import { Middlewares } from 'seyfert';
@Cooldown.user(5_000)
@Middlewares(['cooldown'])
```

Global middleware + custom denial message; when `middleware` is an options object the type is
NOT auto-inferred, so register it with `CooldownMiddlewares`:

```ts
import type { CooldownMiddlewares } from '@slipher/cooldown';

declare module 'seyfert' {
    interface SeyfertRegistry { middlewares: CooldownMiddlewares<'cooldown'> }
}

const plugins = definePlugins(
    cooldown({
        middleware: {
            global: true,
            message: (result, ctx) =>
                `${ctx.author.username}, try again in ${Math.ceil(result.remainingMs / 1000)}s.`,
        },
    }),
);
```

Manager use inside a command (context-scoped zero-arg form):

```ts
const result = await ctx.cooldown.consume();
if (result && !result.allowed) {
    return ctx.write({ content: `Try again in ${Math.ceil(result.remainingMs / 1000)}s.` });
}
```

Manager use outside a handler (explicit form — implicit forms throw outside a Seyfert handler):

```ts
await client.cooldown?.check({ name: 'ping', target: userId, guildId });
await client.cooldown?.consume({ name: 'ping', target: userId, guildId, cost: 2 });
await client.cooldown?.reset({ name: 'ping', target: userId, guildId });
```

## Recipes / Common patterns

These flows are copy-paste-ready. Core imports (`Command`, `Declare`, `Formatter`, `Middlewares`,
`createMiddleware`, `CommandContext`) come from `'seyfert'` and are verified against ./src; the
cooldown surface (`Cooldown`, `cooldown`, `CooldownMiddlewares`) comes from `'@slipher/cooldown'`
and is doc-authoritative (verify the installed version).

Relative "try again" message via `Formatter.timestamp` (verified: `Formatter.timestamp(date)`
defaults to `'R'` relative style and accepts a `Date`, `common/it/formatter.ts:244`). `retryAfter`
is a `Date`, so it drops straight in:

```ts
import { Command, type CommandContext, Declare, Formatter } from 'seyfert';
import { Cooldown } from '@slipher/cooldown';

@Declare({ name: 'daily', description: 'Claim your daily reward' })
@Cooldown.user(24 * 60 * 60 * 1000) // once per 24h per user
export default class DailyCommand extends Command {
    async run(ctx: CommandContext) {
        const result = await ctx.cooldown.consume();
        if (result && !result.allowed) {
            // e.g. "you can claim again in 5 hours"
            return ctx.write({ content: `You can claim again ${Formatter.timestamp(result.retryAfter)}.` });
        }
        await ctx.write({ content: 'Reward claimed!' });
    }
}
```

Read the middleware result from `ctx.metadata` (the bundled middleware calls `next(result)` on
allow, so a gated command can still inspect remaining uses without consuming again):

```ts
@Declare({ name: 'spin', description: 'Spin the wheel' })
@Cooldown.user(10_000, { uses: 3 }) // 3 spins per 10s
@Middlewares(['cooldown'])
export default class SpinCommand extends Command {
    async run(ctx: CommandContext) {
        const cd = ctx.metadata.cooldown; // CooldownResult | undefined, populated by the middleware
        const left = cd?.remainingUses ?? '?';
        await ctx.write({ content: `Spun! Spins left this window: ${left}` });
    }
}
```

Owner bypass + manual reset from an admin command (explicit `{ name, target }` form because the
target is some OTHER user, not the invoker — context resolution only covers `ctx`'s own target):

```ts
import { Command, type CommandContext, Declare, Options, createUserOption } from 'seyfert';

const options = {
    user: createUserOption({ description: 'User to clear', required: true }),
};

@Declare({ name: 'clearcd', description: 'Reset a user cooldown (admin)' })
@Options(options)
export default class ClearCooldownCommand extends Command {
    async run(ctx: CommandContext<typeof options>) {
        const target = ctx.options.user.id;
        // reset returns false when the command has no cooldown configured
        const existed = await ctx.client.cooldown?.reset({ name: 'daily', target });
        await ctx.write({ content: existed ? `Cleared <@${target}>'s cooldown.` : 'That command has no cooldown.' });
    }
}
```

Gate a button handler manually (component contexts have their own scope; the zero-arg form still
works inside a Seyfert handler, but `check` first to avoid consuming when already blocked):

```ts
import { ComponentCommand, type ComponentContext } from 'seyfert';

export default class ClaimButton extends ComponentCommand {
    componentType = 'Button' as const;
    filter(ctx: ComponentContext) {
        return ctx.customId === 'claim';
    }
    async run(ctx: ComponentContext<'Button'>) {
        const peek = await ctx.client.cooldown?.check({ name: 'claim', target: ctx.author.id });
        if (peek && !peek.allowed) {
            return ctx.write({ content: `Slow down — ${Math.ceil(peek.remainingMs / 1000)}s left.`, flags: 64 });
        }
        await ctx.client.cooldown?.consume({ name: 'claim', target: ctx.author.id });
        await ctx.write({ content: 'Claimed!' });
    }
}
```

Per-guild-per-user shared bucket via a custom resolver + group (one bucket spans several heavy
commands, keyed by both guild and user):

```ts
import { Command, Declare } from 'seyfert';
import { Cooldown } from '@slipher/cooldown';

@Declare({ name: 'render', description: 'Expensive render job' })
// returning undefined from the resolver SKIPS the cooldown for that invocation
@Cooldown.custom((ctx) => (ctx.guildId ? `${ctx.guildId}:${ctx.author.id}` : undefined), 30_000, { group: 'heavy' })
export default class RenderCommand extends Command {}
```

Gotchas (all verified against the MDX + the core surfaces below):
- Implicit `consume(...)`/`check(...)` forms ONLY work inside a live Seyfert handler — they throw outside one.
  The same applies to `reset(...)`. In jobs/tests/cross-user admin actions use the explicit form.
- `check`/`consume` return `undefined` when the command has no cooldown — always null-check before
  reading `.allowed`. `client.cooldown` is also optional (`?.`) until the plugin is installed.
- `cost` greater than the bucket `limit` throws `RangeError` — it is a programmer error, not a denial.
- Passing `middleware` as an OBJECT disables type inference of the middleware name; re-add it with
  `CooldownMiddlewares<'name'>` augmenting `SeyfertRegistry.middlewares`.
- Only `consume` can be atomic (adapter must set `supportsAtomicCooldowns: true` + Redis `eval`);
  `check`/`reset` are never atomic, so make `consume` the admission authority in multi-worker bots.
- `group` shares a bucket across commands but does NOT make subcommands inherit a cooldown;
  subcommand resolution is `subcommand.cooldown ?? parent.cooldown`.

## Doc vs Source Corrections

- None for CORE APIs: `definePlugins`, `createPlugin`, `SeyfertRegistry` augmentation, and the
  `Middlewares(['cooldown'])` decorator all match the MDX against ./src.
- The package-specific surface is verified against current `tiramisulabs/extra/packages/cooldown`.
  In a consumer project, the installed package remains authoritative.

## Source Anchors

- `src/components/componentcommand.ts` — `ComponentCommand` (`componentType` 17, `customId` 18,
  `filter` 19, `run` 20), backing the button-gating recipe.
- `src/index.ts` / `src/client/index.ts` — barrels re-export `./plugins`, so all of the above
  resolve from the `'seyfert'` root import.

## Agent Guidance

- For `guild`/`channel` scopes, DMs fall back to `author.id`; a custom resolver returning
  `undefined` skips the cooldown for that invocation. Subcommands use
  `subcommand.cooldown ?? parent.cooldown`.
- Shared `group` buckets use an encoded namespace and target:
  `${encodeURIComponent(group ?? resolvedCommandName)}:${typeLabel}:${encodeURIComponent(target)}`.
  Global buckets use the fixed suffix `:global:global`.
