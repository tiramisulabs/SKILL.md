# Command Middlewares

Original source URL: https://seyfert-web-git-seyfert-v5-tiramisulabs.vercel.app/docs/learn/commands/middlewares
Coverage reference: commands.md
Verification status: Source-verified (v5, the authoritative Seyfert source)

## Page Summary

Middlewares are functions run before a command's `run` executes, used for checks, logging, rate-limits, and passing typed data into the command. You define them with `createMiddleware`, register them as a named record via `client.setServices({ middlewares })`, and attach them per-command with the `@Middlewares([...])` decorator or globally with the `globalMiddlewares` client option. Each middleware receives `{ context, next, stop }` and must call exactly one of `next()` (continue, optionally writing metadata) or `stop()` (halt). NOTE: older v4 docs/tutorials show `middle.pass()` — that method NO LONGER EXISTS in v5; the silent-skip behavior is now `stop()` called with no argument (`stop(null)` works too).

## Key APIs (verified)

- `createMiddleware<T = any, C extends AnyContext = AnyContext>(data: MiddlewareContext<T, C>)` — identity helper that just returns `data` for typing. `T` = the metadata shape passed via `next(data)`; `C` = the context type. (`src/commands/applications/options.ts:213`)
- `AnyContext` = `CommandContext | MenuCommandContext<...> | ComponentContext | ModalContext | EntryPointContext` — the default `C`. Narrow it when you need context-specific fields. (`src/commands/applications/options.ts:206-211`)
- `MiddlewareContext<T = any, C = any>` callback shape: `(context: { context: C; next: NextFunction<T>; stop: StopFunction }) => any`. There is NO `pass` member. (`src/commands/applications/shared.ts:55-59`)
- `NextFunction<T>` = `IsStrictlyUndefined<T> extends true ? () => void : (data: T) => void` — with `createMiddleware<void>` you call `next()`; with data you call `next(data)`. (`src/commands/applications/shared.ts:21`)
- `StopFunction` = `(error?: string | null) => void`. `stop("msg")` denies the command (routes to `onMiddlewaresError`); `stop()` / `stop(null)` silently skips (command does not run, no error). (`src/commands/applications/shared.ts:20`, `src/commands/applications/chat.ts:247-256`)
- `@Middlewares(cbs: readonly MiddlewareKey[])` class decorator — sets `middlewares` on the command instance. Accepts a readonly array of registered middleware name keys. (`src/commands/decorators.ts:188-193`)
- `middlewares(...cbs)` (lowercase helper) — returns the rest-arg tuple with names INFERRED (`const T extends readonly MiddlewareKey[]`); use it to share a typed middleware list across commands. (`src/commands/decorators.ts:184-186`)
- `InferMiddlewares<T extends readonly MiddlewareKey[]> = T[number]` — extracts the middleware-name UNION from a `middlewares(...)` tuple, for typing `CommandContext`'s second generic. (`src/commands/decorators.ts:22`)
- Client option `globalMiddlewares?: readonly (keyof ResolvedRegisteredMiddlewares)[]` on `ClientOptions`. (`src/client/base.ts:1255`)
- `ParseGlobalMiddlewares<T>` — maps a middleware record to its metadata map; used to populate `GlobalMetadata`. (`src/commands/applications/shared.ts:49-51`)
- `GlobalMetadata` (augmentable interface) — extended in your `declare module 'seyfert'` block. (`src/commands/applications/shared.ts:25`)
- `SeyfertRegistry` augmentation key `middlewares` is what `client.setServices` registers and what types resolve against. `RegisteredMiddlewares` / `ResolvedRegisteredMiddlewares` are derived (the latter also folds in plugin middlewares). `ParseMiddlewares` is REMOVED in v5 — `typeof middlewares` is enough. (`src/commands/decorators.ts:14-19`)
- Context fields: `ctx.metadata` (= `CommandMetadata<M>`, per-command middleware data) and `ctx.globalMetadata` (= `GlobalMetadata`, global middleware data). (`src/commands/applications/chatcontext.ts:69-70`)
- `CommandContext<T extends OptionsRecord = {}, M extends keyof ResolvedRegisteredMiddlewares = never>` — second generic `M` is the union of attached middleware names that feed `ctx.metadata`. (`src/commands/applications/chatcontext.ts:45-48`)
- `onMiddlewaresError(ctx, error: string, metadata: PluginMiddlewareDenialMetadata)` — command hook for a `stop("reason")` deny. `metadata = { middleware: string; scope: 'global' | 'command' }` and names the middleware that called `stop('reason')`. A default can be set at `client.options.commands.defaults.onMiddlewaresError`. (`src/client/plugins/types.ts:64-67`, `src/commands/handler.ts:586`)

Behavior verified in `src/commands/applications/chat.ts:200-282` (`BaseCommand.__runMiddlewares`):
- Middlewares run sequentially in the order listed; `next(data)` writes `data` into `metadata`/`globalMetadata` under that middleware's name, then invokes the next one.
- Global middlewares run BEFORE command middlewares, both before `run`. (`src/commands/handle.ts:126-132, 663-725`)
- An unregistered middleware name logs `Command "x" has middleware "y" assigned, but it is not registered.` (`logger.warn`) and is skipped — not fatal. (`chat.ts:208-219`)
- If a middleware throws (sync) or its returned promise rejects (async), the runner **rejects** and the error routes to `onInternalError` — it is NOT a denial and is NOT logged by the runner. Use `stop('reason')` for an intentional deny. (`chat.ts` `BaseCommand.__runMiddlewares` → `invoke`)
- `next`/`stop` are guarded by a `running` flag: only the FIRST call has effect; later calls are silently ignored.

## Code Examples (verified)

Create a middleware:
```ts title="logger.middleware.ts"
import { createMiddleware } from "seyfert";

// Generic <void> => next() takes no data
export const loggerMiddleware = createMiddleware<void>((middle) => {
  // `resolver` lives only on the chat CommandContext; `middle.context` is AnyContext, so narrow first.
  const command = 'resolver' in middle.context ? middle.context.resolver.fullCommandName : 'unknown';
  console.log(`${middle.context.author.username} (${middle.context.author.id}) ran /${command}`);
  // Prefer the built-in logger for consistent output:
  // middle.context.client.logger.info("...");
  middle.next();
});
```

Register them (named record + service + type augmentation):
```ts title="middlewares.ts"
import { loggerMiddleware } from "./logger.middleware";

export const middlewares = {
  logger: loggerMiddleware, // key = name used in @Middlewares([...])
};
```
```ts title="index.ts"
import { Client, type ParseClient } from "seyfert";
import { middlewares } from "./middlewares";

const client = new Client();
client.setServices({ middlewares });

declare module "seyfert" {
  interface SeyfertRegistry {
    client: ParseClient<Client<true>>;
    middlewares: typeof middlewares;
  }
}
```

Attach to a command:
```ts title="ping.command.ts"
import { Middlewares, Declare, Command, type CommandContext } from "seyfert";

@Declare({ name: "ping", description: "Ping the bot" })
@Middlewares(["logger"])
export default class PingCommand extends Command {
  async run(ctx: CommandContext) {
    await ctx.write({ content: "Pong!" });
  }
}
```

Stop vs silent skip (`pass()` removed in v5):
```ts title="guild-only.middleware.ts"
import { createMiddleware, ChannelType } from "seyfert";

export const guildOnly = createMiddleware<void>((middle) => {
  if (middle.context.interaction.channel?.type === ChannelType.DM) {
    // Deny WITH a message -> routed to onMiddlewaresError
    return middle.stop("This command can only be used in a guild.");
  }
  // Silently ignore the interaction (old `pass()`): stop() with NO args
  // return middle.stop();
  middle.next();
});
```

Pass typed data into the command:
```ts title="cooldown.middleware.ts"
import { createMiddleware } from "seyfert";

interface CooldownData { remaining: number }

const lastUsed = new Map<string, number>();
const COOLDOWN = 5_000;

export const cooldown = createMiddleware<CooldownData>((middle) => {
  const key = middle.context.author.id;
  const now = Date.now();
  const prev = lastUsed.get(key) ?? 0;
  const remaining = COOLDOWN - (now - prev);
  if (remaining > 0) {
    return middle.stop(`Wait ${Math.ceil(remaining / 1000)}s before reusing this command.`);
  }
  lastUsed.set(key, now);
  middle.next({ remaining: 0 }); // payload REQUIRED because T is not void
});
```
```ts title="ping.command.ts"
import { Middlewares, Declare, Command, type CommandContext } from "seyfert";

@Declare({ name: "ping", description: "Ping the bot" })
@Middlewares(["cooldown"])
export default class PingCommand extends Command {
  // second generic = attached middleware names; union them with `|` for several
  async run(ctx: CommandContext<never, "cooldown">) {
    ctx.metadata.cooldown.remaining; // typed via ctx.metadata
    await ctx.write({ content: "Pong!" });
  }
}
```

Global middlewares (data lands in `ctx.globalMetadata`, not `ctx.metadata`):
```ts title="index.ts"
import { Client, type ParseClient, type ParseGlobalMiddlewares } from "seyfert";
import { middlewares } from "./middlewares";
import { global } from "./globals";

const globalMiddlewares: (keyof typeof global)[] = ["logger"];

const client = new Client({ globalMiddlewares });
client.setServices({ middlewares: { ...global, ...middlewares } });

declare module "seyfert" {
  interface SeyfertRegistry {
    client: ParseClient<Client<true>>;
    middlewares: typeof middlewares & typeof global;
  }
  interface GlobalMetadata extends ParseGlobalMiddlewares<typeof global> {}
}
```

## Recipes / Common patterns

### Auth/permission deny + a user-facing error handler

`stop("reason")` only flags the deny — YOU respond in `onMiddlewaresError`. Define it on the command (or as a global default).

```ts title="admin-only.middleware.ts"
import { createMiddleware, type CommandContext } from "seyfert";

// Narrow the context generic so author/member fields are typed
export const adminOnly = createMiddleware<void, CommandContext>((middle) => {
  const member = middle.context.member;
  if (!member?.permissions.has("Administrator")) {
    return middle.stop("You need the Administrator permission.");
  }
  middle.next();
});
```
```ts title="kick.command.ts"
import { Middlewares, Declare, Command, type CommandContext } from "seyfert";
import type { PluginMiddlewareDenialMetadata } from "seyfert";

@Declare({ name: "kick", description: "Kick a member" })
@Middlewares(["adminOnly"])
export default class KickCommand extends Command {
  async run(ctx: CommandContext) {
    await ctx.write({ content: "Kicked." });
  }

  // Called when a middleware denies via stop("reason")
  async onMiddlewaresError(
    ctx: CommandContext,
    error: string,
    metadata: PluginMiddlewareDenialMetadata,
  ) {
    // metadata.middleware === "adminOnly", metadata.scope === "command"
    await ctx.write({ content: `Blocked: ${error}`, flags: 64 /* ephemeral */ });
  }
}
```

### Share a typed middleware list with `middlewares()` + `InferMiddlewares`

`middlewares('a','b')` keeps the names as a literal tuple; `InferMiddlewares` turns that tuple into the union you feed `CommandContext`.

```ts title="shared.ts"
import { middlewares } from "seyfert";

// Names are inferred as a literal tuple, not a loose string[]
export const guarded = middlewares("logger", "cooldown");
```
```ts title="secret.command.ts"
import { Middlewares, Declare, Command, type CommandContext, type InferMiddlewares } from "seyfert";
import { guarded } from "./shared";

@Declare({ name: "secret", description: "Guarded command" })
@Middlewares(guarded)
export default class SecretCommand extends Command {
  // InferMiddlewares<typeof guarded> === "logger" | "cooldown"
  async run(ctx: CommandContext<never, InferMiddlewares<typeof guarded>>) {
    ctx.metadata.cooldown.remaining;
    await ctx.write({ content: "ok" });
  }
}
```

### Middlewares on components & modals (not just slash commands)

`ComponentCommand`, `ModalCommand`, and context-menu commands all expose a `middlewares` field, so the same registered middlewares apply. Use the matching context generic. IMPORTANT: `globalMiddlewares` run only for application commands (slash/menu/entry-point) — component & modal handlers run their OWN `middlewares` only, never the global list.

```ts title="confirm.button.ts"
import { createMiddleware, type ComponentContext } from "seyfert";
import { ComponentCommand, type ComponentContext as CC } from "seyfert";

// A component-scoped middleware (narrow C to ComponentContext)
export const sameUser = createMiddleware<void, ComponentContext>((middle) => {
  const ownerId = middle.context.interaction.message.interactionMetadata?.user.id;
  if (ownerId && middle.context.author.id !== ownerId) {
    return middle.stop("This button isn't for you.");
  }
  middle.next();
});

export default class ConfirmButton extends ComponentCommand {
  componentType = "Button" as const;
  middlewares = ["sameUser"] as const; // or use @Middlewares(["sameUser"])

  filter(ctx: CC<"Button">) {
    return ctx.customId === "confirm";
  }
  async run(ctx: CC<"Button">) {
    await ctx.update({ content: "Confirmed!" });
  }
}
```

### Gotchas

- Call EXACTLY one of `next` / `stop` per middleware. Forgetting both leaves the command pending forever (the promise never resolves). Calls after the first are ignored.
- `stop("reason")` does NOT message the user — implement `onMiddlewaresError` (per command) or `commands.defaults.onMiddlewaresError` (global). Bare `stop()` drops the interaction silently with no handler call.
- A throwing/rejecting middleware (sync or async) **rejects the runner → `onInternalError`** (an exception is an internal error, not a denial). It will NOT crash the process (the command consumer catches it), but the command does not run. Use `stop('reason')` to deny.
- Unregistered middleware names only `logger.warn` and are skipped — a typo'd key silently does nothing instead of throwing.
- Global metadata is on `ctx.globalMetadata` (typed via `GlobalMetadata`/`ParseGlobalMiddlewares`), separate from per-command `ctx.metadata` (typed via the `CommandContext` second generic). Don't read global data off `ctx.metadata`.
- Order: globals first (in `globalMiddlewares` order), then per-command (in `@Middlewares` order), then `run`. Put auth/short-circuit checks early.
- `command.middlewares` is a `readonly` tuple in v5 — mutating it in place (`.push(...)`) is a type error.

## Doc vs Source Corrections

- Docs/older tutorials say `middle.pass()` to ignore an interaction -> src has NO `pass` method; the callback arg is only `{ context, next, stop }`. Silent-skip is now `stop()` / `stop(null)` (resolves internally to `{ pass: true }`, treated as "don't run, no error"). The current v5 MDX is already updated to `stop()`. (`src/commands/applications/shared.ts:55-59`, `src/commands/applications/chat.ts:247-256`)
- `stop` is dual-purpose — a string denies (error -> `onMiddlewaresError`), no/null arg skips silently. (`chat.ts:247-256`)
- Module augmentation uses `SeyfertRegistry` (key `middlewares`); the v4 `RegisteredMiddlewares extends ParseMiddlewares<...>` pattern is gone — `ParseMiddlewares` was removed. (`src/commands/decorators.ts:14-19`)
- MDX example string ``ran /(${...fullCommandName}`` has an unbalanced `(` typo -> corrected above.
- Exception handling (throw/reject -> reject -> `onInternalError`) and missing-middleware warning are not in the MDX -> documented above from src.
- Global middleware metadata is on `ctx.globalMetadata`, separate from per-command `ctx.metadata`. (`src/commands/applications/chatcontext.ts:69-70`)

## Source Anchors

- `src/components/componentcommand.ts`, `src/commands/applications/menu.ts`, `src/components/modalcommand.ts` (`middlewares` field on component/menu/modal commands)

## Agent Guidance

- All root imports (`createMiddleware`, `Middlewares`, `middlewares`, `InferMiddlewares`, `Command`, `CommandContext`, `ComponentContext`, `ChannelType`, `Client`, `ParseClient`, `ParseGlobalMiddlewares`, `PluginMiddlewareDenialMetadata`) come from `'seyfert'` — no deep imports.
