# Logger (@slipher/logger)

Original source URL: https://seyfert-web-git-seyfert-v5-tiramisulabs.vercel.app/docs/plugins/official/logger

Coverage reference: plugins.md

Verification status: Source-verified (core integration) + EXTERNAL package (verify version in target project)

## Page Summary

`@slipher/logger` is an EXTERNAL Seyfert v5 plugin (NOT part of core Seyfert) that gives every command, component, and modal a request-scoped **wide event** logger as `ctx.logger`, emitted automatically once when the interaction ends — turning one interaction into a single queryable entry (its outcome, duration, and every attached field) instead of a scattered pile of log lines. Ordinary level methods (`info`, `warn`, ...) still emit immediately as their own entries; the final wide event accumulates fields added via `ctx.logger.add()` and is flushed with `outcome` + `durationMs` when the handler returns (or `outcome: 'error'` / `'denied'` on a middleware, option, permission, or runtime failure). Output flows through a pluggable adapter: a pretty/JSON console by default, or `createPinoAdapter` / `createEvlogAdapter`. The plugin integrates through Seyfert's standard plugin `ctx` extension mechanism (`createPlugin({ ctx })`), which is what injects `ctx.logger`.

`client.logger` remains Seyfert's OWN logger — this plugin does not replace it. Do not confuse the two (see Gotchas).

Always verify the exact `@slipher/logger` API against the version installed in the target project.

## Key APIs (verified)

Core Seyfert APIs the docs lean on (all confirmed in ./src, the authoritative Seyfert source):

- `definePlugins(...plugins)` / `definePlugins(plugins[])` — both overloads exist; returns the tuple as-is (`src/client/plugins.ts:283-289`). Root import from `'seyfert'`.
- `createPlugin({ name, ctx, client, middlewares, register, setup, teardown, ... })` — the plugin shape; the `ctx` map `(interaction, client) => value` is what a plugin uses to inject per-interaction fields like `logger` (`src/client/plugins.ts:240-265`; `PluginContextMap` at `src/client/plugins/types.ts:250-256`). `ctx` values become typed context properties via the registry.
- `interface SeyfertRegistry {}` — empty, augmentable (`src/client/plugins/types.ts:32`). `{ plugins: typeof plugins }` drives `ctx.logger` inference (`RegisteredPlugins` → `PluginContextMapOf`, types.ts:286-301); `{ middlewares: {...} }` is the middleware key (consumed at `src/commands/decorators.ts`).
- `createMiddleware<T, C>(data)` — `src/commands/applications/options.ts:213`. Root import from `'seyfert'`.
- `Command`, `@Declare`, `@Middlewares`, `CommandContext<Options, Middlewares>` — `src/commands/decorators.ts`; all root imports from `'seyfert'`.
- `context.metadata.<middleware>` carries the value returned by `next(value)` — used as `context.metadata.audit.plan`.

External `@slipher/logger` surface (doc-authoritative — verify version in target project):

- `logger(options)` — plugin factory. Options:
  - `name: string` — a binding label on every record (NOT the plugin name, which is always `@slipher/logger`).
  - `level: LogLevel` — `'trace' | 'debug' | 'info' | 'warn' | 'error' | 'fatal' | 'silent'`, default `'info'`.
  - `bindings: Record<string, unknown>` — static fields attached to every record.
  - `adapter: LoggerAdapter` — where records go; default is the pretty/JSON console adapter.
  - `context: AutoContextConfig` — toggle which Seyfert fields are auto-extracted.
- `ctx.logger` — wide-event logger. `.add(fields)` enriches the single final event; level methods (`.info()`, `.warn()`, ...) emit immediately as their own entries; `.emit(record)` flushes a wide event (only needed OUTSIDE an interaction scope).
- `useLogger()` — returns the active scope's wide-event logger inside an interaction, or a fresh root-backed logger outside any scope. Throws only if the plugin is not set up yet. Reads the active scope (no parameter), so a helper deep in the call stack can enrich the interaction's event without being handed `ctx`.
- `createPinoAdapter(pinoInstance)`, `createEvlogAdapter()` — adapter factories.
- Auto-extracted fields on every wide event: `kind`, `command` / `customId`, `guildId`, `channelId`, `userId`, `interactionId`. `shardId` is OFF by default; toggle via the `context` option.
- Plugin-driven lifecycle: `onBeforeMiddlewares` / `onBeforeOptions` write immediate `debug` breadcrumbs; a failure emits ONE wide event with `outcome: 'error'`/`'denied'` at the point it happens; a successful run emits ONE wide event with `outcome: 'success'` + `durationMs` when the command returns.

## Code Examples (verified)

### Install plugin (core integration verified; root imports from `'seyfert'`)

```ts
import { Client, definePlugins } from 'seyfert';
import { logger } from '@slipher/logger';

// name labels every record; level sets the floor
const loggerPlugin = logger({ name: 'slipher-bot', level: 'debug' });
const plugins = definePlugins(loggerPlugin);

const client = new Client({ plugins });

declare module 'seyfert' {
    interface SeyfertRegistry { plugins: typeof plugins }
}
```

The `SeyfertRegistry { plugins }` augmentation is what makes `ctx.logger` visible to TypeScript. Without it, `ctx.logger` is untyped/absent.

### Carry context through middleware into the single wide event

```ts
import { Command, Declare, Middlewares, createMiddleware, type CommandContext } from 'seyfert';

export const auditMiddleware = createMiddleware<{ requestId: string; plan: 'free' | 'pro' }, CommandContext>(
    async ({ context, next }) => {
        const audit = { requestId: crypto.randomUUID(), plan: await loadUserPlan(context.author.id) } as const;
        context.logger.add(audit); // enriches the final wide event
        return next(audit);        // value lands on context.metadata.audit
    },
);

declare module 'seyfert' {
    interface SeyfertRegistry { middlewares: { audit: typeof auditMiddleware } }
}

@Declare({ name: 'deploy', description: 'Deploy the current project' })
@Middlewares(['audit'])
export default class DeployCommand extends Command {
    async run(context: CommandContext<{}, 'audit'>) {
        context.logger.add({ projectId: 'web', plan: context.metadata.audit.plan });
        context.logger.info('deployment queued'); // immediate, separate entry
        await context.write({ content: 'Deployment queued.' });
    }
}
```

No `emit()` on the happy path — the plugin emits one wide event when `run()` returns. The emitted shape:

```ts
// the single wide event (auto fields + middleware + command + outcome + durationMs)
{
    message: 'command completed',
    kind: 'command', command: 'deploy', userId: '123',
    requestId: '7c5d…', plan: 'pro', projectId: 'web',
    outcome: 'success', durationMs: 42,
}
// the immediate info() call is its own separate entry:
{ level: 'info', message: 'deployment queued' }
```

### Enrich the wide event from deep in the call stack (no ctx threading)

`useLogger()` reads the active interaction scope, so a helper called by a command can add fields to the same wide event without receiving `ctx`.

```ts
import { Command, Declare, type CommandContext } from 'seyfert';
import { useLogger } from '@slipher/logger';

async function loadProfile(userId: string) {
    const profile = await db.profiles.find(userId);
    // reads the active scope → enriches the interaction's wide event
    useLogger().add({ plan: profile.plan, cached: profile.fromCache });
    return profile;
}

@Declare({ name: 'profile', description: 'Show your profile' })
export default class ProfileCommand extends Command {
    async run(context: CommandContext) {
        const profile = await loadProfile(context.author.id);
        await context.write({ content: `Plan: ${profile.plan}` });
    }
}
```

When `run()` returns the plugin emits one wide event that already carries `plan` and `cached` — no `emit()` anywhere.

### Components & modals get `ctx.logger` too

The plugin attaches `ctx.logger` to component and modal contexts as well, with the same lifecycle (one wide event when the handler returns). Auto fields use `customId` instead of `command` for these.

```ts
import { ComponentCommand, type ComponentContext } from 'seyfert';

export default class ConfirmButton extends ComponentCommand {
    componentType = 'Button' as const;

    async run(ctx: ComponentContext) {
        ctx.logger.add({ action: 'confirm' });
        ctx.logger.info('button pressed'); // immediate
        await ctx.update({ content: 'Confirmed.' }); // ComponentContext: no editResponse in v5
        // one wide event auto-emits here with customId, outcome, durationMs
    }
}
```

### Logger outside any interaction (helpers / events / startup)

```ts
import { useLogger } from '@slipher/logger';

useLogger().info('ready'); // immediate, anywhere

// a one-off wide event from e.g. an interactionCreate event handler
const event = useLogger();  // capture the FRESH event (see Gotchas)
event.add({ source: 'event', interactionId: interaction.id });
await event.emit({ message: 'interactionCreate received' }); // you emit yourself outside a scope
```

### Toggle auto-extracted context fields

```ts
// shardId is off by default; turn it on, drop channelId
logger({ context: { shardId: true, channelId: false } });
```

### Pino adapter

```ts
import { Client } from 'seyfert';
import { createPinoAdapter, logger } from '@slipher/logger';
import pino from 'pino';

// your Pino instance owns transports + redaction
const sink = pino({ redact: ['token', 'headers.authorization'] });

const client = new Client({
    plugins: [logger({ name: 'slipher-bot', adapter: createPinoAdapter(sink) })],
});
```

### evlog adapter

`createEvlogAdapter()` takes no options — it only translates records. Configure evlog globally once with `initLogger()` in your entrypoint (drains, redaction, sampling, service envelope). evlog is an optional peer dependency.

```ts
import { Client } from 'seyfert';
import { initLogger } from 'evlog';
import { createFsDrain } from 'evlog/fs';
import { createDrainPipeline } from 'evlog/pipeline';
import { createEvlogAdapter, logger } from '@slipher/logger';

const drain = createDrainPipeline({ batch: { size: 50, intervalMs: 5_000 } })(createFsDrain());

initLogger({
    env: { service: 'slipher-bot', environment: process.env.NODE_ENV ?? 'development' },
    redact: {
        paths: ['token', 'headers.authorization'],
        patterns: [/Bot\s+[A-Za-z0-9._-]+/g], // built-ins do NOT cover Discord bot tokens
    },
    drain,
    silent: true,
});

const client = new Client({
    plugins: [logger({ name: 'slipher-bot', adapter: createEvlogAdapter() })],
});
```

Any evlog drain works — Axiom, OTLP, Sentry, fs, or a custom pipeline.

## Common patterns / gotchas

- **`add()` vs level methods.** `.add(fields)` accumulates onto the ONE per-interaction wide event (emitted at the end). Level methods (`.info()`, `.warn()`, ...) emit a separate entry immediately. Use `add()` for "what happened to this request"; use levels for live breadcrumbs.
- **Never call `emit()` inside a command/component/modal.** The plugin owns that lifecycle and auto-emits with `outcome` + `durationMs`. Calling `emit()` yourself there duplicates the event. `emit()` is only for one-off wide events OUTSIDE a scope.
- **`useLogger()` outside a scope returns a FRESH event each call.** Capture it in a variable before `add()` then `emit()`, or your added context is lost on the next call.
- **Don't re-add auto-extracted fields.** `kind`, `command`/`customId`, `guildId`, `channelId`, `userId`, `interactionId` are already on every wide event. `shardId` is off — enable via `context: { shardId: true }`.
- **`client.logger` ≠ `ctx.logger`.** `client.logger` is Seyfert's own logger (configurable in v5 via `new Client({ logger: { ... } })`, honored on worker clients too). The plugin's wide-event logger lives on `ctx.logger` / `useLogger()`; it does not replace `client.logger`.
- **Redaction is the sink's job.** The console adapter does NOT redact. Configure it in Pino, in evlog's `initLogger()`, or in your collector. evlog's built-ins (`creditCard`, `email`, `jwt`, ...) miss Discord tokens — add a `/Bot\s+[A-Za-z0-9._-]+/g`-style pattern.
- **On field collisions**, data from `add()` and level methods wins over static `bindings`.
- **`ctx.logger` requires the `SeyfertRegistry { plugins }` augmentation** for TypeScript to see it; the plugin injects it via the `createPlugin({ ctx })` map.
- **v5 context note:** ComponentContext has NO `editResponse` (removed in v5) — use `ctx.update(...)` / `ctx.editOrReply(...)`. ModalContext still has `editResponse`. Middleware control flow uses `stop()` (no `pass()`): `stop()` = silent skip, `stop('reason')` = deny. None of this changes the logger examples (they only use `next()` / `write` / `update`), but keep it in mind when expanding handlers.

## Doc vs Source Corrections

- None for core APIs. `definePlugins`, `createPlugin`, `createMiddleware`, `Command`/`@Declare`/`@Middlewares`, `CommandContext`, and the `SeyfertRegistry { plugins }` / `{ middlewares }` augmentations all match ./src exactly (paths confirmed below). The `@slipher/logger`-specific surface (`logger()`, `ctx.logger`, `useLogger`, adapters) lives in the external package and cannot be verified in core Seyfert — treat the MDX as authoritative and confirm against the installed version.
- The `ctx`-injection claim is verified structurally: a plugin's typed `ctx` map drives per-interaction context properties via `PluginContextMap` / `PluginContextMapOf` and `RegisteredPlugins` (derived from `SeyfertRegistry { plugins }`). That is the exact mechanism the MDX relies on for `ctx.logger`.

## Source Anchors

- `src/client/plugins.ts` (definePlugins 283-289, createPlugin 240-265, createPluginFactory, lifecycle, createContextScope)
- `src/client/plugins/types.ts` (SeyfertRegistry:32, PluginContextMap:250, RegisteredPlugins:286, PluginContextMapOf, ExtendOf/ContextOf)
- `src/client/plugins/api.ts` (plugin API surface; reserved context keys)
- `src/commands/applications/options.ts` (createMiddleware:213)
- `src/commands/decorators.ts` (Declare, Middlewares, registry-derived middleware names)
- `src/commands/applications/shared.ts` (SeyfertRegistry consumption for langs/client/middlewares)
- `src/client/index.ts` (`export * from './plugins'`)

## Agent Guidance

- This is an EXTERNAL plugin. Before editing production code, install it from the target project's package manager and inspect the installed `@slipher/logger` types — do not assume option names beyond what is listed here.
- `ctx.logger` only exists once the plugin is installed AND `SeyfertRegistry { plugins: typeof plugins }` is augmented (it is provided through the plugin `ctx` map). Without the augmentation TypeScript will not see `ctx.logger`.
- Use `.add()` for fields on the single per-interaction wide event; use level methods for immediate, separate entries. Never `emit()` inside a handler — the plugin owns that lifecycle.
- For shared helpers, reach for `useLogger()` instead of threading `ctx` through every call — it reads the active scope and enriches the same wide event.
- Don't confuse `client.logger` (core Seyfert logger) with the plugin's `ctx.logger` wide-event logger.
- Redaction lives in the sink/adapter, not in the console adapter; add a Discord-token pattern when using evlog.
