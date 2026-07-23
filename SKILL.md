---
name: seyfert
description: Deep Seyfert v5 guidance for building, updating, debugging, reviewing, and testing Discord bots that use the Seyfert TypeScript framework. Use when a task mentions Seyfert, imports from the `seyfert` package, `seyfert.config` files, Seyfert class/decorator APIs (`Command`, `Declare`, `Options`, `ComponentCommand`, `createEvent`, `createPlugin`), Slipher packages, @slipher/testing, or yunaforseyfert.
---

# Seyfert (v5)

## Overview

Source-aware guidance for Seyfert v5 Discord bots and for the Seyfert framework itself. The public docs are a good workflow map, but **the source of truth** whenever names, signatures, exports, or behavior conflict is the target project's installed `node_modules/seyfert`, the current HEAD of a Seyfert source checkout when working on the framework, or Seyfert source explicitly provided by the user. `references/source-truth.md` lists the doc-vs-source corrections to re-check against that authority.

## Source Of Truth (read this order)

1. **Inspect the project first:** `package.json` + lockfile (confirm the installed `seyfert` version and any `@slipher/*`, `yunaforseyfert`, lavalink deps), `seyfert.config.*`, `tsconfig.json`, the command/component/event/lang folders, module augmentations (`declare module 'seyfert'`), and existing tests. If `seyfert` is below `5.0.0`, apply the Version Scope rule in `references/source-truth.md`.
2. **Prefer root imports from `seyfert`.** Use a deep import only for an API not root-exported in the installed version — known case: custom prefix parsers via `seyfert/lib/commands/handle` (`HandleCommand`). Before adding a deep import, check `lib/index.d.ts` / `src/index.ts` / the barrels.
3. **Verify core APIs against the target project's installed `node_modules/seyfert`, the current Seyfert source checkout, or source explicitly provided by the user before copying any docs example.** Docs can be ahead of (or behind) the installed build.
4. **External packages** (`@slipher/cooldown|logger|opentelemetry|queues|scheduler|chartjs`, `@slipher/testing`, `yunaforseyfert`, lavalink clients) are **NOT** in the `seyfert` package. Verify the installed package first; when maintaining this skill, use the matching `tiramisulabs/extra/packages/*/src` plus `package.json` as the official-package source of truth. Use docs only as a workflow map.

`references/source-truth.md` holds the curated, source-cited corrections and local landmarks — **read it before trusting any docs example for a typed API.**

## Quick Start (minimal gateway bot)

Four files — config, entry, one command, one event. Class-based + file auto-discovery.

```ts
// seyfert.config.mjs (or .ts) — MUST default-export config.bot() / config.http()
import { config } from 'seyfert';

export default config.bot({
  token: process.env.TOKEN ?? '',
  intents: ['Guilds'],                 // string names or a bitfield number
  locations: {
    base: 'src',                       // folder that contains the dirs below
    commands: 'commands',
    events: 'events',
    // components: 'components', langs: 'langs'   // optional
  },
});
```

```ts
// src/index.ts
import { Client, type ParseClient } from 'seyfert';

const client = new Client();

// start() connects the gateway but does NOT upload application commands:
client.start().then(() => client.uploadCommands());

// Type ctx.client etc. by augmenting SeyfertRegistry (NOT `interface UsingClient`):
declare module 'seyfert' {
  interface SeyfertRegistry {
    client: ParseClient<Client<true>>;
  }
}
```

```ts
// src/commands/ping.ts — default export, auto-discovered
import { Command, Declare, type CommandContext } from 'seyfert';

@Declare({ name: 'ping', description: 'Ping the bot' })
export default class Ping extends Command {
  async run(ctx: CommandContext) {
    await ctx.write({ content: `Pong! ${ctx.client.latency}ms` });
  }
}
```

```ts
// src/events/ready.ts — default export of createEvent
import { createEvent } from 'seyfert';

export default createEvent({
  data: { name: 'botReady', once: true },
  run(user, client) {
    client.logger.info(`Ready as ${user.username}`);
  },
});
```

## Common Patterns (verified)

Slash command **with typed options**:

```ts
import { Command, Declare, Options, createStringOption, type CommandContext } from 'seyfert';

const options = { query: createStringOption({ description: 'Search query', required: true }) };

@Declare({ name: 'search', description: 'Search' })
@Options(options)
export default class Search extends Command {
  async run(ctx: CommandContext<typeof options>) {
    await ctx.write({ content: ctx.options.query }); // `query` is a typed string
  }
}
```

**Embed** reply, and a **middleware** guard:

```ts
import { Embed } from 'seyfert';
await ctx.write({ embeds: [new Embed().setTitle('Hi').setDescription('World').setColor('Blurple')] });
```

```ts
import { createMiddleware } from 'seyfert';

export const middlewares = {
  // { context, next, stop } — there is NO `pass`. stop() = silent skip, stop('msg') = deny.
  ownerOnly: createMiddleware<void>(({ context, next, stop }) =>
    context.author.id === process.env.OWNER_ID ? next() : stop('Owner only.')),
};
// client.setServices({ middlewares });  +  @Middlewares(['ownerOnly'])
// declare module 'seyfert' { interface SeyfertRegistry { middlewares: typeof middlewares } }
```

For components/buttons, modals, collectors, polls, i18n, cache, and plugins, read the matching reference — each has full copy-paste recipes.

## Reference Routing

Open only what the task needs:

| Task area | Reference |
|---|---|
| Project setup, clients (gateway/HTTP/worker), `start()`, `seyfert.config`, intents, sharding, presence, module augmentation, events | `references/setup-runtime.md` |
| Slash commands, options/autocomplete, middlewares, subcommands/groups, context menus, prefix commands, errors/defaults, `extendContext` | `references/commands.md` |
| Message components, component/modal handlers, collectors, polls, Components v2 | `references/components.md` |
| Builders: `Embed`, `Button`, `ActionRow`, select menus, `Modal`, v2 (`Container`/`Section`/…), attachments, `Formatter` | `references/builders.md` |
| **Shorters** (`client.messages`/`channels`/`members`/… resource methods), full method catalog, structure shorters, and the shorter→`proxy`→`rest` fallback chain | `references/shorters.md` |
| Plugin authoring: `createPlugin`/`definePlugins`, lifecycle, runtime hooks, services/requirements, ordering, diagnostics, official `@slipher/*` plugins | `references/plugins.md` |
| i18n/langs, cache (adapters/resources), structures/transformers, and recipes (DB, logger, API access, monetization, music, yuna) | `references/i18n-cache-recipes.md` |
| Testing with `@slipher/testing` (mock bot, dispatching, world, assertions, fixtures, gateway, defaults) + local unit tests | `references/testing.md` |
| Per-page deep notes for a specific docs URL (each page's source URL, coverage ref, and verification label live in its header + `pages/index.md`) | `references/pages/index.md` → matching page note |

## Implementation Defaults

- Keep Seyfert artifacts **class-based** unless the project clearly differs: `Command`, `SubCommand`, `ContextMenuCommand`, `ComponentCommand`, `ModalCommand`; events via `createEvent({ data, run })`.
- **Module augmentation lives on `interface SeyfertRegistry`** (keys `client`, `middlewares`, `langs`, `plugins`). Do **not** augment `interface UsingClient` / `RegisteredMiddlewares` / `DefaultLocale` — on v5 those are *derived types* and can no longer be interface-merged. See `source-truth.md`.
- File-loaded commands/components/events/langs use **default exports**. Programmatic registration (`commands.set`, `components.set`, `langs.set`) suits serverless / no-filesystem deployments.
- Commands with subcommands should live in a dedicated folder. With `@AutoLoad()`, this is required: Seyfert scans the parent command's directory recursively, so keep unrelated helpers (`shared.ts`, constants, group maps, resolvers) outside that auto-loaded folder, preferably in `src/lib/**`.
- `client.start()` does **not** upload application commands — call `client.uploadCommands({ cachePath })` deliberately when registration is part of the task.
- Middleware control flow is `next()` / `stop()` only — **there is no `pass()`** (use `stop()` / `stop(null)` to silently skip).
- **Shorter-first for every Discord API call.** Reach for a client shorter (`client.messages.write`, `client.members.fetch`, `client.roles.list`, …) or a structure method (`member.ban()`, `channel.messages.write()`) before anything else — they take raw ids, cache, transform, resolve files, and return typed structures. If a method **seems missing**, walk the chain: **① find the shorter** (`references/shorters.md` has the full catalog) **→ ② `client.proxy.<route>()`** for raw routes with no shorter **→ ③ `client.rest.request<T>(...)`**, typing `T` from the Discord API v10 reference. Never hand-roll `fetch`/`axios`. Accessors are plural (`client.messages`, not `client.message`).
- Discord constraints: slash names lowercase; context-menu names may use spaces/case **and require an explicit `@Declare({ type })` with no `description`**; autocomplete must `respond()` (not `reply()`); action-row/component limits apply.
- For app-wide services prefer **plugins** (`createPlugin`/`definePlugins`); for one-off context sugar use `extendContext` (augment `ExtendContext`).

## Verification

- **Seyfert core (library/source) changes:** run the narrowest relevant test, then `pnpm test` when practical (the repo's test script builds, runs type contracts, then Vitest).
- **Bot projects:** `tsc --noEmit` + existing unit/integration tests; if command schemas changed, exercise `uploadCommands` behavior explicitly.
- **Plugins:** test lifecycle (`register`/`setup`/`teardown`), ordering/imports/requirements, diagnostics, and type augmentation.
- **HTTP/serverless:** verify request handling, public-key/adapter wiring, and no reliance on filesystem-loaded folders the platform can't read.

When behavior is uncertain, read the installed declarations or `src/**` before editing — start from the landmarks in `references/source-truth.md`.
