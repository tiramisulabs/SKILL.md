# Handling Errors

Original source URL: https://seyfert-web-git-seyfert-v5-tiramisulabs.vercel.app/docs/learn/commands/handling-errors

Coverage reference: commands.md

Verification status: Source-verified (the authoritative Seyfert source)

## Page Summary

Seyfert lets a command override per-lifecycle error/hook methods so failures are handled in a structured way and reported back to the user. The error hooks cover thrown `run` errors, option `value()` validation failures, middleware stops, and missing member/bot permissions; lifecycle hooks add before/after control points. Any hook can also be set globally via `client.options.commands.defaults` (and lighter `components.defaults` / `modals.defaults`), with per-command definitions taking precedence (applied via `??=`). Component (`ComponentCommand`) and modal (`ModalCommand`) handlers expose the same family of hooks (minus options/permissions). SOURCE NOTES vs the docs MDX: `onMiddlewaresError` now takes a 3rd `metadata` arg, `onInternalError` takes the offending `command`/`component`/`modal` as its 2nd arg (error is last), and middleware no longer has `pass` — use `stop()`.

## Key APIs (verified)

All chat hooks are optional methods on `BaseCommand` (extended by `Command` / `SubCommand`); `ContextMenuCommand` supports the subset noted. Signatures from `src/commands/applications/chat.ts:349-358`:

- `onBeforeMiddlewares?(context: CommandContext): any`
- `onBeforeOptions?(context: CommandContext): any` — chat `Command` only (not on `ContextMenuCommand`)
- `run?(context: CommandContext): any`
- `onAfterRun?(context: CommandContext, error: unknown | undefined): any` — second arg is the run error if one was thrown, else `undefined`. Fires on BOTH success and failure (see flow below).
- `onRunError?(context: CommandContext, error: unknown): any`
- `onOptionsError?(context: CommandContext, metadata: OnOptionsReturnObject): any` — chat `Command` only
- `onMiddlewaresError?(context: CommandContext, error: string, metadata: PluginMiddlewareDenialMetadata): any` — 3 args
- `onBotPermissionsFail?(context: CommandContext, permissions: PermissionStrings): any`
- `onPermissionsFail?(context: CommandContext, permissions: PermissionStrings): any` — chat `Command` only
- `onInternalError?(client: UsingClient, command: Command | SubCommand, error?: unknown): any` — `command` is 2nd, `error` is 3rd

Component / modal hooks (same shape, fewer hooks — NO options/permissions hooks):
- `ComponentCommand` (`src/components/componentcommand.ts:41-49`): `onBeforeMiddlewares`, `onAfterRun(ctx, error?)`, `onRunError(ctx, error)`, `onMiddlewaresError(ctx, error, metadata)`, `onInternalError(client, component: ComponentCommand, error?)`. Plus `abstract run(context)` and `filter(context)`.
- `ModalCommand` (`src/components/modalcommand.ts:31-35`): same set; `onInternalError(client, modal: ModalCommand, error?)`; `abstract run(context: ModalContext)`.

Supporting types/exports (importable from root `'seyfert'` unless noted):
- `OnOptionsReturnObject` (`src/commands/applications/shared.ts`): `Record<string, { failed: false; value: unknown } | { failed: true; value: string; parseError: ... }>`. For failed entries `value` is the error string passed to `fail()`.
- `PluginMiddlewareDenialMetadata` (`src/client/plugins/types.ts:64`): `{ middleware: string; scope: 'global' | 'command' }`. The `middleware` names the one that called `stop('reason')`.
- `PermissionStrings` — array of permission name strings.
- `createMiddleware<T = any, C = AnyContext>(data: MiddlewareContext<T, C>)` (`src/commands/applications/options.ts:213`).
- `MiddlewareContext` payload (`src/commands/applications/shared.ts:55-59`): `{ context, next: NextFunction<T>, stop: StopFunction }`. There is NO `pass`.
  - `StopFunction = (error?: string | null) => void` (shared.ts:20)
  - `NextFunction<T>` = `() => void` when `T` is undefined, else `(data: T) => void` (shared.ts:21)
  - `OKFunction<T> = (value: T) => void` (shared.ts:19)
- Option `value` callback (`src/commands/applications/options.ts`): `value(data, ok: OKFunction<I>, fail: StopFunction)`.
- `SeyfertError.is(error, code?)` (`src/common/it/error.ts:37-39`) narrows a caught error by `SeyfertErrorCode` (e.g. `'CANNOT_USE_MODAL'`). Useful inside `onInternalError`/`onRunError`.

Client defaults (`src/client/base.ts:1256-1298`):
- `commands.defaults` accepts: `onBeforeMiddlewares`, `onBeforeOptions`, `onRunError`, `onPermissionsFail`, `onBotPermissionsFail`, `onInternalError`, `onMiddlewaresError`, `onOptionsError`, `onAfterRun`, `props`.
  - Here `onInternalError` is `(client, command: Command | SubCommand | ContextMenuCommand, error?) => unknown` and `onMiddlewaresError` is `(context, error: string, metadata) => unknown`.
- `components.defaults` and `modals.defaults` accept ONLY: `onBeforeMiddlewares`, `onRunError`, `onInternalError`, `onMiddlewaresError`, `onAfterRun` (no options/permission hooks).
- Defaults are applied by `stablishCommandDefaults` (handler.ts:592-607), `stablishContextCommandDefaults` (576-590), and `setComponentDefaults` (components/handler.ts:257-261) using `??=`, so a per-command/per-component hook wins over the default.

## Hook firing flow (verified)

Chat command path (`src/commands/handle.ts:120-251, 425, 448-485, 678-755`):
1. `onBotPermissionsFail` / `onPermissionsFail` fire if perms are missing (these `return` — `run` never executes).
2. `onBeforeOptions` (Command only) → option parsing. A failing `value()` → `onOptionsError` and stops.
3. `onBeforeMiddlewares` → middlewares. A `stop('reason')` → `onMiddlewaresError`.
4. `run()` executes inside a try/catch:
   - success → `onAfterRun(context, undefined)`
   - thrown → `onRunError(context, error)` THEN `onAfterRun(context, error)`.
5. If ANY of the above hooks themselves throw, the outer catch runs `onInternalError(client, command, error)`. `onInternalError` is the safety net for your own handlers blowing up.
6. Each hook also fans out to plugin command observers (`runPluginCommandObservers`), so plugins can observe `onRunError`, `onAfterRun`, etc.

Component / modal path (`src/components/handler.ts:380-426`): `onBeforeMiddlewares` → global middlewares → command middlewares (`stop` → `onMiddlewaresError`) → `run` (success → `onAfterRun(ctx, undefined)`, throw → `onRunError(ctx, error)` + `onAfterRun(ctx, error)`). Any hook throwing → `onInternalError(client, component/modal, error)`; if THAT throws too, it is logged via `client.logger.error`.

## Code Examples (verified)

Run error:
```ts
import { Command, type CommandContext } from 'seyfert';

export default class HandlingErrors extends Command {
  async run(context: CommandContext) {
    throw new Error('Error, ehm, lol player detected');
  }

  // Receives the thrown error
  async onRunError(context: CommandContext, error: unknown) {
    context.client.logger.fatal(error);
    await context.editOrReply({
      content: error instanceof Error ? error.message : `Error: ${error}`,
    });
  }
}
```

Option validation error:
```ts
import {
  Command, Options, createStringOption,
  type CommandContext, type OnOptionsReturnObject, type OKFunction,
} from 'seyfert';

declare const isURL: (url: string) => boolean;

const options = {
  url: createStringOption({
    description: 'how to be a gamer',
    value(data, ok: OKFunction<URL>, fail) {
      if (isURL(data.value)) return ok(new URL(data.value));
      fail('expected a valid URL'); // triggers onOptionsError
    },
  }),
};

@Options(options)
export default class HandlingErrors extends Command {
  async onOptionsError(context: CommandContext, metadata: OnOptionsReturnObject) {
    await context.editOrReply({
      content: Object.entries(metadata)
        .filter(([, v]) => v.failed)
        .map(([name, v]) => `${name}: ${v.value}`) // value is the fail() string
        .join('\n'),
    });
  }
}
```

Middleware stop + handler (note `stop`, not `pass`; 3-arg `onMiddlewaresError`):
```ts
import { createMiddleware } from 'seyfert';

const Devs: string[] = [];

export const OnlyDev = createMiddleware<void>(({ context, next, stop }) => {
  if (!Devs.includes(context.author.id)) return stop('User is not a developer');
  next();
});
```
```ts
import { Command, Middlewares, type CommandContext } from 'seyfert';
import type { PluginMiddlewareDenialMetadata } from 'seyfert';

@Middlewares(['OnlyDev'])
export default class HandlingErrors extends Command {
  async onMiddlewaresError(
    context: CommandContext,
    error: string,
    metadata: PluginMiddlewareDenialMetadata, // { middleware, scope }
  ) {
    // metadata.middleware is the exact middleware that called stop()
    await context.editOrReply({ content: `[${metadata.middleware}] ${error}` });
  }
}
```

Silent skip vs. deny (the `pass()` replacement):
```ts
import { createMiddleware } from 'seyfert';

declare const isIgnoredChannel: (id: string) => boolean;
declare const isBlocked: (id: string) => boolean;

export const Gate = createMiddleware<void>(({ context, next, stop }) => {
  if (isIgnoredChannel(context.channelId)) return stop();        // skip silently, NO onMiddlewaresError
  if (isBlocked(context.author.id)) return stop('You are blocked'); // deny → onMiddlewaresError
  return next();
});
```

Permission hooks:
```ts
import { Command, type CommandContext, type PermissionStrings } from 'seyfert';

export default class HandlingErrors extends Command {
  async onPermissionsFail(context: CommandContext, permissions: PermissionStrings) {
    await context.editOrReply({ content: `You need: ${permissions.join(', ')}` });
  }
  async onBotPermissionsFail(context: CommandContext, permissions: PermissionStrings) {
    await context.editOrReply({ content: `I need: ${permissions.join(', ')}` });
  }
}
```

Other lifecycle hooks (note `onInternalError` arg order — `command` is 2nd):
```ts
import { Command, type CommandContext, type UsingClient, type SubCommand } from 'seyfert';

export default class MyCommand extends Command {
  async onBeforeMiddlewares(context: CommandContext) {}
  async onBeforeOptions(context: CommandContext) { await context.deferReply(); }
  async onAfterRun(context: CommandContext, error: unknown | undefined) {
    if (!error) context.client.logger.info(`${this.name} ran ok`);
  }
  async onInternalError(client: UsingClient, command: Command | SubCommand, error?: unknown) {
    client.logger.fatal(`[${command.name}] internal:`, error);
  }
}
```

Client-wide defaults (corrected signatures: `onMiddlewaresError` 3-arg, `onInternalError` command 2nd):
```ts
import { Client } from 'seyfert';

const client = new Client({
  commands: {
    defaults: {
      onRunError: (context, error) => { context.editOrReply({ content: 'Something went wrong!' }); },
      onOptionsError: (context) => { context.editOrReply({ content: 'Invalid options.' }); },
      onPermissionsFail: (context, permissions) => { context.editOrReply({ content: `Missing: ${permissions.join(', ')}` }); },
      onBotPermissionsFail: (context, permissions) => { context.editOrReply({ content: `I need: ${permissions.join(', ')}` }); },
      onMiddlewaresError: (context, error, _metadata) => { context.editOrReply({ content: error }); },
      onInternalError: (client, _command, error) => { client.logger.fatal(error); },
    },
  },
  components: { defaults: { onRunError: (context, _error) => { context.editOrReply({ content: 'Component error!' }); } } },
  modals: { defaults: { onRunError: (context, _error) => { context.editOrReply({ content: 'Modal error!' }); } } },
});
```

## Recipes / Common patterns

Recipe 1 — Component & modal handler error hooks (v5 arg order: `component`/`modal` before `error`):
```ts
import { ComponentCommand, ModalCommand, type ComponentContext, type ModalContext, type UsingClient } from 'seyfert';

export class ConfirmButton extends ComponentCommand {
  componentType = 'Button' as const;
  filter(ctx: ComponentContext<'Button'>) { return ctx.customId === 'confirm'; }

  async run(ctx: ComponentContext<'Button'>) {
    throw new Error('boom');
  }

  async onRunError(ctx: ComponentContext<'Button'>, error: unknown) {
    await ctx.editOrReply({ content: 'That button broke. Try again.' });
  }
  // component is the 2nd arg now, error is last
  async onInternalError(client: UsingClient, component: ComponentCommand, error?: unknown) {
    client.logger.error(`[${component.constructor.name}]`, error);
  }
}

export class FeedbackModal extends ModalCommand {
  filter(ctx: ModalContext) { return ctx.customId === 'feedback'; }
  async run(ctx: ModalContext) {
    const text = ctx.getInputValue('opinion', true);
    await ctx.write({ content: `Thanks: ${text}` });
  }
  async onMiddlewaresError(ctx: ModalContext, error: string) {
    await ctx.write({ content: error });
  }
}
```

Recipe 2 — Narrow a caught error with `SeyfertError.is` inside a handler:
```ts
import { Command, SeyfertError, type CommandContext } from 'seyfert';

export default class Profile extends Command {
  async run(ctx: CommandContext) {
    // ... something that may throw a SeyfertError (e.g. modal on a prefix ctx)
  }
  async onRunError(ctx: CommandContext, error: unknown) {
    if (SeyfertError.is(error, 'CANNOT_USE_MODAL')) {
      return ctx.editOrReply({ content: 'This command needs a slash interaction.' });
    }
    ctx.client.logger.fatal(error);
    await ctx.editOrReply({ content: 'Unexpected error.' });
  }
}
```

Recipe 3 — Ephemeral, user-friendly error reply with a fallback chain:
```ts
import { Command, MessageFlags, type CommandContext } from 'seyfert';

export default class Risky extends Command {
  async run(ctx: CommandContext) {
    await ctx.deferReply(); // visible "thinking…"
    throw new Error('db down');
  }
  async onRunError(ctx: CommandContext, error: unknown) {
    ctx.client.logger.error(error);
    // editOrReply works whether or not run() already deferred/replied
    await ctx.editOrReply({
      content: 'Something failed on our side. The team has been notified.',
      flags: MessageFlags.Ephemeral,
    });
  }
}
```

Recipe 4 — Global default + per-command override (default wins only when the class omits the hook):
```ts
// client.ts: a global fallback for everything
const client = new Client({
  commands: { defaults: { onRunError: (ctx) => ctx.editOrReply({ content: 'Oops.' }) } },
});

// ping.ts: this command opts into a custom message; the default is NOT used here
export default class Ping extends Command {
  async run(ctx: CommandContext) { throw new Error('x'); }
  async onRunError(ctx: CommandContext) { await ctx.editOrReply({ content: 'Ping failed specifically.' }); }
}
// any command WITHOUT onRunError still gets the global 'Oops.'
```

## Doc vs Source Corrections

- `onMiddlewaresError`: docs MDX show `(context, error)` -> src has `(context, error: string, metadata: PluginMiddlewareDenialMetadata)` (chat.ts:355, base.ts:1271). The 3rd param is optional to use but exists; `metadata.middleware` names the denying middleware.
- `onInternalError`: docs MDX show `(client, error)` -> src has `(client, command, error?)` — `command` is 2nd (chat.ts:358); in `commands.defaults` it is `Command | SubCommand | ContextMenuCommand` (base.ts:1266). For components/modals it is `(client, component, error?)` / `(client, modal, error?)`. The docs' `onInternalError: (client, error) => ...` binds the command instance to `error` — correct to `(client, _command, error)`.
- Middleware `pass`: docs MDX destructure `{ context, next, stop, pass }` -> src `MiddlewareContext` only exposes `next` and `stop` (shared.ts:55-59). `pass` was removed (the authoritative Seyfert source: "replace middleware pass() with stop()"). Use `stop()`/`stop(null)` to skip silently, `stop('reason')` to deny → `onMiddlewaresError`.
- `onAfterRun` second arg is `error: unknown | undefined` (required param, may be `undefined`), not optional `error?` (chat.ts:352). It fires on success (undefined) AND after `onRunError` (the error) — handle.ts:133-138.
- `onBeforeOptions`, `onOptionsError`, `onPermissionsFail` are chat `Command`-only; `ContextMenuCommand` (stablishContextCommandDefaults, handler.ts:576-590), `components`, and `modals` defaults do not accept them.
- `PluginMiddlewareDenialMetadata` is re-exported at the root `seyfert` (`src/client/plugins.ts`), so `import type { PluginMiddlewareDenialMetadata } from 'seyfert'` if you need the type annotation (runtime hooks work without importing it).

## Source Anchors

- `src/commands/applications/chat.ts` (BaseCommand hook signatures 349-358)
- `src/commands/handle.ts` (chat execution flow + hook firing order 105-251, 425, 448-485, 678-755)
- `src/commands/handler.ts` (`stablishCommandDefaults` 592-607, `stablishContextCommandDefaults` 576-590, `stablishSubCommandDefaults` 609-659)
- `src/client/base.ts` (`BaseClientOptions` defaults 1256-1298)
- `src/components/handler.ts` (component/modal execution + defaults 257-261, 380-426)
- `src/components/componentcommand.ts` (ComponentCommand hooks 20-49)
- `src/components/modalcommand.ts` (ModalCommand hooks 14-35)
- `src/commands/applications/shared.ts` (`MiddlewareContext`/`StopFunction`/`NextFunction`/`OKFunction` 19-60, `OnOptionsReturnObject`)
- `src/commands/applications/options.ts` (`createMiddleware` 213, option `value` callback)
- `src/client/plugins/types.ts` (`PluginMiddlewareDenialMetadata` 64)
- `src/common/it/error.ts` (`SeyfertError.is` 37-39, `SeyfertErrorCode`/`CANNOT_USE_MODAL` 101, 142)

## Agent Guidance

- Use per-command hooks for command-specific UX; use `commands.defaults` for a global fallback (applied with `??=`, so a method defined on the class always wins). Set `components.defaults` / `modals.defaults` for component & modal handlers — but only the 5 supported keys (`onBeforeMiddlewares`, `onRunError`, `onInternalError`, `onMiddlewaresError`, `onAfterRun`).
- Which hook fires: thrown error in `run` -> `onRunError`; option `value()` calling `fail()` -> `onOptionsError`; a middleware calling `stop('reason')` -> `onMiddlewaresError` (a bare `stop()` skips silently with NO hook); missing member perms (`defaultMemberPermissions`) -> `onPermissionsFail`; missing bot perms (`botPermissions`) -> `onBotPermissionsFail`; one of YOUR hooks throwing -> `onInternalError`.
- `onAfterRun` always runs (cleanup/metrics) — check its `error` arg to branch success vs failure. `onInternalError` is the last-resort net for handler bugs; keep it simple so it cannot throw (component path logs via `client.logger.error` if it does).
- Gotcha: copying the docs' `onInternalError: (client, error) =>` is a real bug now — the 2nd arg is the command/component/modal, not the error. Always treat the LAST arg as the error.
- Gotcha: any middleware still written with `pass` will fail to type-check / run on this branch; migrate to `stop()`.
- Gotcha: `editOrReply`/`write` return `void` unless you pass the response flag (`true`); don't `await` them expecting a `Message` inside an error handler unless you opt in.
- `SubCommand` hooks are auto-bound and fall back to the parent command's hook then the client default (`stablishSubCommandDefaults`), so you usually only define hooks on the parent unless a subcommand needs special handling.
