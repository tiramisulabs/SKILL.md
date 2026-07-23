# Logger (`@slipher/logger`)

Original source URL: https://seyfert-web-git-seyfert-v5-tiramisulabs.vercel.app/docs/plugins/official/logger

Coverage reference: `plugins.md`

Verification status: Source-verified against current `tiramisulabs/extra/packages/logger` + Seyfert core

## Contents

- [Contract](#contract)
- [Setup](#setup)
- [Interaction logging](#interaction-logging)
- [Background work](#background-work)
- [Renderers and transports](#renderers-and-transports)
- [Lifecycle and gotchas](#lifecycle-and-gotchas)
- [Source anchors](#source-anchors)

## Contract

`logger(options?)` installs structured logging for Seyfert v5. Each command, component, and
modal gets a request-scoped `WideEventLogger` as `ctx.logger`. `.add(fields)` enriches one final
wide event; level methods (`trace` through `fatal`) emit separate entries immediately.

Current `LoggerPluginOptions`:

- `name?: string` — record binding / console tag; the plugin name remains `@slipher/logger`.
- `level?: 'trace' | 'debug' | 'info' | 'warn' | 'error' | 'fatal' | 'silent'`.
- `bindings?: Record<string, unknown>` — static fields on every record.
- `renderer?: LoggerAdapter` — the single terminal renderer; defaults to `prettyRenderer()`.
- `transports?: readonly LoggerAdapter[]` — structured sinks that ship entries.
- `context?: Partial<Record<AutoContextField, boolean>>` — auto-extracted Seyfert fields.

There is no `adapter` option and no `createPinoAdapter` / `createEvlogAdapter` export.

Root exports:

- Plugin/scope: `logger`, `useLogger`, `withLoggerScope`, `extractSeyfertLogContext`.
- Core: `createLogger`, `RootLogger`, `WideEventLogger`, `LoggerAdapter`, logger data/options types.
- Console: `ConsoleLoggerAdapter`, `prettyRenderer`, `silentRenderer`.
- Pino: `pinoAdapter`.
- evlog: `evlogRenderer`, `evlogTransport`.

## Setup

```ts
import { Client, definePlugins } from 'seyfert';
import { logger } from '@slipher/logger';

const loggerPlugin = logger({
  name: 'my-bot',
  level: 'debug',
});
const plugins = definePlugins(loggerPlugin);

declare module 'seyfert' {
  interface SeyfertRegistry {
    plugins: typeof plugins;
  }
}

export const client = new Client({ plugins });
```

The `SeyfertRegistry.plugins` augmentation makes `ctx.logger` visible to TypeScript.

## Interaction logging

```ts
import { Command, Declare, type CommandContext } from 'seyfert';
import { useLogger } from '@slipher/logger';

async function loadProfile(userId: string) {
  const profile = await db.profiles.find(userId);
  useLogger().add({ plan: profile.plan, cached: profile.fromCache });
  return profile;
}

@Declare({ name: 'profile', description: 'Show your profile' })
export default class ProfileCommand extends Command {
  async run(ctx: CommandContext) {
    const profile = await loadProfile(ctx.author.id);
    ctx.logger.add({ profileId: profile.id });
    await ctx.logger.info('profile loaded'); // immediate entry
    await ctx.write({ content: `Plan: ${profile.plan}` });
    // The scoped wide event emits automatically.
  }
}
```

Auto fields default on: `kind`, `command`/`customId`, `guildId`, `channelId`, `userId`,
`interactionId`. `shardId` defaults off:

```ts
logger({ context: { shardId: true, channelId: false } });
```

Outcomes are `success`, `error`, `denied`, or `skipped`. Do not manually emit the scoped
interaction event: `emit()` is idempotent, but early emission closes the event before later
fields and can lock in the wrong outcome.

## Background work

Outside an active scope, `useLogger(owner?)` returns a fresh root-backed wide event. Capture it
before adding fields:

```ts
const event = useLogger(client);
event.add({ kind: 'job', jobId });
await event.emit({ message: 'job completed' });
```

Bare `useLogger()` works when exactly one logger installation is active. It throws before setup
or when multiple logger-equipped clients are active. Pass the owning client in ambiguous code.

Use `withLoggerScope` to emit success/failure automatically and make the event available through
`useLogger()`:

```ts
await withLoggerScope(
  { kind: 'job', jobId },
  async () => {
    const processed = await processJob(jobId);
    useLogger().add({ processed });
  },
  client,
);
```

## Renderers and transports

Default console:

```ts
logger({
  name: 'my-bot',
  renderer: prettyRenderer(),
  transports: [],
});
```

Pino:

```ts
import pino from 'pino';
import { logger, pinoAdapter } from '@slipher/logger';

const sink = pino({ redact: ['token', 'headers.authorization'] });
const loggerPlugin = logger({
  renderer: pinoAdapter(sink),
});
```

Keep Slipher's console and ship structured entries through Pino:

```ts
logger({
  renderer: prettyRenderer(),
  transports: [pinoAdapter(filePino)],
});
```

evlog:

```ts
import { evlogTransport, logger } from '@slipher/logger';

logger({
  name: 'my-bot',
  transports: [
    evlogTransport({
      env: { environment: process.env.NODE_ENV ?? 'development' },
      drain,
    }),
  ],
});
```

`evlogRenderer(config?)` prints and drains. `evlogTransport(config?)` drains without printing.
When given config, the adapter initializes evlog and derives `env.service` from the logger name.
When called without config, it assumes the host already configured evlog. Install `evlog` only
when using these adapters.

Redaction belongs to the configured sink or collector; the default console does not redact.

## Lifecycle and gotchas

- `client.logger` remains Seyfert's logger. `@slipher/logger` also bridges Seyfert subsystem and
  internal log output into its root logger; it does not expose a wide-event logger as
  `client.logger`.
- Setup instruments commands/components/modals and installs logger bridges. Teardown flushes
  pending writes and restores replaced logger properties.
- Adapter failures are isolated and reported to `console.error`; they do not fail application
  logic.
- Outside a scope, each `useLogger()` call returns a new event. Reuse one variable for
  `.add()` + `.emit()`.
- Data added to the event wins over static bindings on field collisions.

## Source anchors

- `packages/logger/src/index.ts` — public exports.
- `packages/logger/src/core.ts` — options, root logger, wide events, emit idempotence.
- `packages/logger/src/plugin.ts` — plugin lifecycle, scopes, outcomes, `useLogger(owner?)`.
- `packages/logger/src/{console,pino,evlog}.ts` — current adapters.
- Seyfert `src/client/plugins.ts` and `src/client/plugins/types.ts` — plugin/context typing.
