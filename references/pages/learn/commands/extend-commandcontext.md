# Extending CommandContext

Original source URL: https://seyfert-web-git-seyfert-v5-tiramisulabs.vercel.app/docs/learn/commands/extend-commandcontext
Coverage reference: commands.md
Verification status: Source-verified (the authoritative Seyfert source)

## Page Summary

Seyfert lets you inject custom data into every command/component/modal context via the
client `context` option. You pass a callback (optionally wrapped in the `extendContext`
helper for inference) that receives the raw interaction (or message for prefix commands)
and returns a plain object; its properties are `Object.assign`-ed onto each context at
dispatch time. The runtime data is NOT typed automatically — you augment the
`ExtendContext` interface via `declare module 'seyfert'` to get types.

Two ways to feed `ExtendContext`:
1. The client-level `context` / `extendContext` hook (this page) — manual augmentation required.
2. A plugin's `ctx` map (`createPlugin({ ctx: { ... } })`) — auto-typed via
   `RegisteredPluginContext`, no manual augmentation. Both feed the SAME `ctx.*` surface and
   compose together at runtime.

## Key APIs (verified)

- `extendContext<T extends {}>(cb)` — `src/index.ts:119`. Identity helper that just
  returns `cb`; its only job is to give the callback parameter the correct interaction
  union type and infer the return type `T`. Signature:
  `extendContext<T extends {}>(cb: (interaction) => T)`.
- `BaseClientOptions.context?` — `src/client/base.ts:1245`. The actual hook.
  Type: `(interaction) => Record<string, unknown>`. The `interaction` union is:
  `ChatInputCommandInteraction<boolean> | UserCommandInteraction<boolean> |
  MessageCommandInteraction<boolean> | ComponentInteraction | ModalSubmitInteraction |
  EntryPointInteraction<boolean> | When<InferWithPrefix, MessageStructure, never>`.
  (The `MessageStructure` arm only appears when prefix/text commands are enabled.)
- `interface ExtendContext extends RegisteredPluginContext` — `src/commands/applications/shared.ts:27`.
  This is the interface you augment to type your custom properties. It is the shared base
  that every context type extends. Because it extends `RegisteredPluginContext`
  (`src/client/plugins/types.ts:290`), plugin-contributed context props appear here too,
  automatically.
- Plugin context map: `createPlugin({ ctx: PluginContextMap })` — `src/client/plugins.ts:177`,
  `:220`. Each `ctx` entry is `(interaction, client) => value`; types flow into
  `ExtendContext` with NO manual `declare module`. (`PluginContextMap` —
  `src/client/plugins/types.ts:250`.)
- All contexts inherit `ExtendContext`:
  - `CommandContext` — `src/commands/applications/chatcontext.ts:38`
  - `MenuCommandContext` — `src/commands/applications/menucontext.ts:37`
  - `EntryPointContext` — `src/commands/applications/entrycontext.ts:27`
  - `ComponentContext` — `src/components/componentcontext.ts:46`
  - `ModalContext` — `src/components/modalcontext.ts:30`
  - `InteractionResponseContext` — `src/components/interactioncontext.ts:24`
- All contexts also extend `BaseContext` (`src/commands/basecontext.ts`) which provides the
  `is*` type guards (`isChat()`, `isMenu()`, `isComponent()`, `isModal()`, `isButton()`,
  `isStringSelectMenu()`, …) used to narrow `ctx` after the merge.

## Code Examples (verified)

### 1. Define and register the extender (root import from `seyfert`)

```ts
import { Client, extendContext } from 'seyfert';

const context = extendContext((interaction) => {
  // `interaction` is the full interaction/message union — narrow if you need
  // command-specific data. Return any plain object (synchronously).
  return {
    myCoolProp: 'seyfert>>',
  };
});

const client = new Client({ context });
```

### 2. Type the added properties by augmenting `ExtendContext`

Required — runtime extension alone does not add types:

```ts
import type { ParseClient, Client } from 'seyfert';

declare module 'seyfert' {
  interface ExtendContext {
    myCoolProp: string;
  }
  // Usually combined with your other module declarations. On v5 the client is
  // registered through SeyfertRegistry — `UsingClient` is now a *derived type
  // alias* (shared.ts:43) and can NO LONGER be interface-merged.
  interface SeyfertRegistry {
    client: ParseClient<Client<true>>;
  }
}
```

Now every context is typed:

```ts
// inside any command run(ctx) / component / modal handler
ctx.myCoolProp; // string
```

### 3. Inline callback without the helper

`extendContext` only adds inference; an inline `context(interaction){...}` is identical:

```ts
const client = new Client({
  context(interaction) {
    return { myCoolProp: 'seyfert>>' };
  },
});
```

### 4. Narrowing the union before reading interaction-specific fields

The callback parameter is a wide union, so `.options`, `.data`, `.values` etc. are NOT
available without narrowing. Use a discriminator (the interaction `type`, the presence of a
field, or `instanceof`) to narrow inside the callback:

```ts
import { Client, extendContext, ChatInputCommandInteraction } from 'seyfert';

const context = extendContext((interaction) => {
  // Always-available basics live on the union.
  const base = {
    userId: 'user' in interaction ? interaction.user.id : interaction.author.id,
    guildId: interaction.guildId ?? null,
    requestedAt: Date.now(),
  };

  // Narrow to a specific interaction class for type-safe access.
  if (interaction instanceof ChatInputCommandInteraction) {
    return { ...base, commandName: interaction.data.name };
  }
  return { ...base, commandName: null };
});

const client = new Client({ context });

declare module 'seyfert' {
  interface ExtendContext {
    userId: string;
    guildId: string | null;
    requestedAt: number;
    commandName: string | null;
  }
}
```

Note: `interaction.author` (message arm) only exists when prefix commands are enabled, hence
the `'user' in interaction` narrowing above.

### 5. Owner / permission helpers and a per-dispatch id

A realistic flow: stamp every context with whether the actor is a bot owner plus a unique id
for tracing/logging.

```ts
import { Client, extendContext } from 'seyfert';
import { randomUUID } from 'node:crypto';

const OWNER_IDS = ['123456789012345678', '234567890123456789'] as const;

const context = extendContext((interaction) => {
  const actorId = 'user' in interaction ? interaction.user.id : interaction.author.id;
  return {
    traceId: randomUUID(),
    isOwner: (OWNER_IDS as readonly string[]).includes(actorId),
  };
});

const client = new Client({ context });

declare module 'seyfert' {
  interface ExtendContext {
    traceId: string;
    isOwner: boolean;
  }
}

// later, in any command:
export default class Ban extends Command {
  async run(ctx: CommandContext) {
    if (!ctx.isOwner) return ctx.editOrReply({ content: 'Owners only.' });
    ctx.client.logger.info(`[${ctx.traceId}] running ban`);
  }
}
```

### 6. Plugin-contributed context (auto-typed, no `declare module`)

A plugin's `ctx` map is the v5-preferred way to ship reusable per-context helpers: the types
flow into `ExtendContext` automatically because `ExtendContext extends RegisteredPluginContext`.
Register the plugin's type in `SeyfertRegistry.plugins` and you are done — no manual
`interface ExtendContext` block.

```ts
import { Client, createPlugin, definePlugins } from 'seyfert';

const tracing = createPlugin({
  name: 'tracing',
  // (interaction, client) => value — value type is inferred onto ctx.*
  ctx: {
    traceId: () => crypto.randomUUID(),
    locale: (interaction) =>
      ('locale' in interaction ? interaction.locale : undefined) ?? 'en-US',
  },
});

const client = new Client({ plugins: definePlugins(tracing) });

declare module 'seyfert' {
  interface SeyfertRegistry {
    // plugins must be registered here for ctx.traceId / ctx.locale to be typed
    plugins: typeof client.options.plugins;
  }
}

// ctx.traceId (string) and ctx.locale (string) are now typed everywhere — no
// `interface ExtendContext` augmentation needed.
```

Client-level `context` and every plugin `ctx` fragment COMPOSE: at start Seyfert reduces all
callbacks with `Object.assign` into one merged context function
(`src/client/plugins.ts:832-843`), then merges the result onto each context at dispatch
(`src/commands/handle.ts:262`, etc.). Later callbacks win on key collisions.

## Doc vs Source Corrections

- Docs say "declare the Seyfert module" but do not name the interface. Source: the
  interface to augment is `ExtendContext` (`src/commands/applications/shared.ts:27`),
  exported from `seyfert`. Augment it inside `declare module 'seyfert' { interface
  ExtendContext { ... } }`.
- Docs callback example narrows mentally to "the interaction", but source typing
  (`src/client/base.ts:1245`) shows the parameter is a wide union covering chat-input,
  user/message context menus, components, modals, entry points, and (when prefix is on)
  a raw `MessageStructure`. Narrow before accessing interaction-type-specific fields.
- The doc only covers the client `context` hook; it omits the v5 plugin `ctx` path, which is
  the auto-typed, reusable alternative (`src/client/plugins.ts:177`). Recipe 6 above.
- The runtime merge is `Object.assign(context, this.client.options?.context?.(interaction) ?? {})`
  in `src/commands/handle.ts` (interaction sites lines 261, 269, 318, 337, 746; message/prefix
  site line 420), so returning `undefined`/nothing is safe (falls back to `{}`).

## Source Anchors

- `src/index.ts` (extendContext helper, line 119; identity `return cb`)
- `src/client/base.ts` (BaseClientOptions.context type, lines 1245-1254)
- `src/commands/applications/shared.ts` (ExtendContext interface, line 27; UsingClient is a
  derived type alias at line 43 — not interface-mergeable)
- `src/client/plugins/types.ts` (RegisteredPluginContext line 290; PluginContextMap line 250)
- `src/client/plugins.ts` (createPlugin line 240; `ctx`/`client` plugin keys lines 176-177,
  219-220; context-callback composition lines 832-843)
- `src/commands/applications/chatcontext.ts`, `menucontext.ts`, `entrycontext.ts`
- `src/components/componentcontext.ts`, `modalcontext.ts`, `interactioncontext.ts`
- `src/commands/handle.ts` (Object.assign merge sites 261, 269, 318, 337, 420, 746)
- `src/commands/basecontext.ts` (BaseContext — base of all contexts, `is*` guards)

## Common patterns / gotchas

- TWO steps for typed custom props via the client hook: (1) the runtime `context` option, and
  (2) the `declare module 'seyfert' { interface ExtendContext {} }` augmentation. Doing only
  (1) leaves `ctx.myProp` untyped; doing only (2) makes TS believe a prop exists that is
  `undefined` at runtime. (Plugin `ctx` does both in one step.)
- The callback runs on EVERY context build (commands, menus, components, modals, entry
  points), so keep it cheap and side-effect-free.
- It must be SYNCHRONOUS as far as the merge is concerned — `Object.assign` copies the
  returned object directly. Returning a `Promise` assigns the Promise, not awaited values. If
  you need async data, store an id/handle here and `await` a service inside the command.
- The interaction parameter is a UNION. Accessing `.options`, `.data`, `.values`, or
  `.author` without narrowing is a type error. Narrow with `instanceof`, `'field' in
  interaction`, or by checking the interaction type. (`BaseContext` `is*` guards narrow the
  built `ctx` later — they are not available inside the `context` callback.)
- `ctx.author` vs `ctx.user`: inside the callback the message arm exposes `.author`, the
  interaction arms expose `.user`. Guard with `'user' in interaction`.
- Prefer plugins (`ctx`/`client` maps) for app-wide reusable features — they get types, a
  lifecycle (`setup`/`teardown`), and compose safely. Use the bare `context` hook for one-off,
  app-local computed helpers.
- Key collisions: when both the client `context` hook and plugin `ctx` fragments set the same
  key, the merge order (`src/client/plugins.ts:842`) means the user `context` callback runs
  last and wins; plugin fragments run in registration order before it.
- v5 reminder: do NOT augment `UsingClient` or `RegisteredMiddlewares` — both are derived from
  `SeyfertRegistry` now. Register the client via `SeyfertRegistry.client` and middlewares via
  `SeyfertRegistry.middlewares` (`typeof middlewares`; `ParseMiddlewares` was removed).

## Agent Guidance

- Use `context`/`extendContext` for small, synchronous, per-context computed helpers derived
  from the interaction (parsed locale flag, owner check, trace id, quick cache lookup). For
  app-wide services/clients prefer plugins, which feed `RegisteredPluginContext` →
  `ExtendContext` automatically.
- When writing a reusable library/feature, expose context props through `createPlugin({ ctx })`
  rather than asking consumers to wire a `context` callback and augment `ExtendContext` by hand.
- `extendContext` is purely a typing convenience (identity function); inline
  `context(interaction){...}` is equivalent.
