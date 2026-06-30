# Creating Your First Command

Original source URL: https://seyfert-web-git-seyfert-v5-tiramisulabs.vercel.app/docs/learn/getting-started/first-command
Coverage reference: setup-runtime.md
Verification status: Source-verified (seyfert-core, branch more-qol)

## Page Summary

Minimal slash-command tutorial: default-export a class extending `Command`, annotate it with the `@Declare` decorator, implement `async run(ctx)`, and reply with `ctx.write`. Seyfert auto-loads every command class found in the configured `commands` folder. Registration with Discord is a separate step — `client.start()` only boots the client and loads files; you call `client.uploadCommands(...)` to push commands to Discord. The page then adds typed options via `@Options` + `createBooleanOption`, exposing them through `CommandContext<typeof options>`.

The original MDX hides a `SeyfertRegistry` augmentation behind a Twoslash `---cut---` preamble; that block is REQUIRED in real projects for `ctx.client.gateway` to type-check, so this note surfaces it.

## Key APIs (verified)

- `Command` — abstract base class for chat-input commands; subclass + default-export it. `src/commands/applications/chat.ts`.
- `@Declare(input)` — class decorator; sets `name` (lowercased for ChatInput at the type level via `LowercaseDeclareName`), `description`, and optional `contexts`, `integrationTypes`, `guildId`, `nsfw`, `botPermissions`, `defaultMemberPermissions`, `aliases`, `ignore`, `props`, `handler`, `type`. `contexts`/`integrationTypes` accept string names (`'Guild'`, `'GuildInstall'`) or enum values; defaults to all contexts + `GuildInstall`. `src/commands/decorators.ts:195`.
- `@Options(options)` — class decorator; accepts either an `OptionsRecord` object (keys MUST be lowercase) or an array of `SubCommand` constructors. `src/commands/decorators.ts:161`.
- Option creators (all in `src/commands/applications/options.ts`): `createStringOption` (146), `createIntegerOption` (154), `createNumberOption` (162), `createChannelOption` (170), `createBooleanOption` (178), `createUserOption` (184), `createRoleOption` (188), `createMentionableOption` (192), `createAttachmentOption` (199). Each returns `{ ...data, type: ApplicationCommandOptionType.X }`.
  - Choices: `choices` is a `readonly SeyfertChoice[]` — use `as const`. Choice value type is tied to the option type (string options reject numeric values and vice-versa).
  - Validators: string → `min_length`/`max_length`/`autocomplete`; integer/number → `min_value`/`max_value`/`autocomplete`; channel → `channel_types` (readonly, use `as const`).
  - `autocomplete(interaction)` receives an `AutocompleteInteraction<boolean, ValueType>`; reply with `interaction.respond([{ name, value }])`. `value` is typed to the option's value type. `src/commands/applications/chat.ts:63`, `respond` at `src/structures/Interaction.ts:465`.
  - `required?: boolean` controls whether `ctx.options.<name>` is non-nullable.
  - `value?(data, ok, fail)` is an optional per-option transform/validate hook.
- `CommandContext<typeof options>` — generic carries the options shape; access parsed values via `ctx.options.<name>`. `src/commands/applications/chatcontext.ts`.
- `ctx.write(body, withResponse?)` — first reply to the interaction (or message in prefix mode). Returns `void` unless `withResponse === true`. `chatcontext.ts:88`.
- `ctx.deferReply(ephemeral?, withResponse?)` — sends a "Thinking…" deferral; reply later with `editOrReply`. `chatcontext.ts:107`.
- `ctx.editOrReply(body, withResponse?)` — edits the existing response or sends a fresh one. Returns `void` unless `withResponse === true`. `chatcontext.ts:145`.
- `ctx.editResponse(body)` — edits the already-sent response (exists on CommandContext). `chatcontext.ts:126`.
- `ctx.followup(body)` — sends an additional message after the first reply. `chatcontext.ts:156`.
- `ctx.modal(body, options?)` — opens a modal; with options returns the `ModalSubmitInteraction | null`, without returns `undefined`. Throws `SeyfertError('CANNOT_USE_MODAL')` in prefix mode. `chatcontext.ts:98`.
- `ctx.inGuild()` — type guard narrowing to `GuildCommandContext`, where `channel(...)` returns a guild channel and `fetchMember(...)` is non-undefined. `chatcontext.ts:260`.
- `ctx.client` — the running client; its static type comes from the `SeyfertRegistry` augmentation. `src/commands/basecontext.ts`.
- `client.gateway.latency` — averaged shard latency. `gateway` is the `ShardManager` (`src/client/base.ts:41`); `get latency()` averages each shard's latency (`src/websocket/discord/sharder.ts:81`).
- `client.uploadCommands({ applicationId?, cachePath? })` — pushes loaded commands to Discord; `cachePath` writes a JSON snapshot so unchanged commands are not re-uploaded. `src/client/base.ts:1056`.
- `client.start(options?, execute?)` — boots client + loads command/event/component files. Does NOT upload commands. `src/client/client.ts:145`.
- `MessageFlags` — re-exported from `seyfert` (from `./types`); `MessageFlags.Ephemeral` for invisible replies.

All names above import from the root `'seyfert'` barrel (verified via `src/index.ts` → `export * from './commands'` and `./types`). No deep imports needed.

## Code Examples (verified)

### Basic ping command (`src/commands/ping.ts`)

```ts
import { Declare, Command, type CommandContext } from 'seyfert';

@Declare({
  name: 'ping',
  description: 'Show latency with Discord',
})
export default class PingCommand extends Command {
  async run(ctx: CommandContext) {
    const ping = ctx.client.gateway.latency; // averaged shard latency
    await ctx.write({
      content: `The latency is \`${ping}\``,
    });
  }
}
```

### Start + register with Discord (`src/index.ts`)

```ts
import { Client } from 'seyfert';

const client = new Client();

client
  .start()
  .then(() => client.uploadCommands({ cachePath: './commands.json' }));
```

### With a typed boolean option (`src/commands/ping.ts`)

```ts
import {
  Command,
  Declare,
  Options,
  createBooleanOption,
  MessageFlags,
  type CommandContext,
} from 'seyfert';

const options = {
  hide: createBooleanOption({
    description: "Hide the command's response",
  }),
};

@Declare({
  name: 'ping',
  description: 'Show latency with Discord',
})
@Options(options)
export default class PingCommand extends Command {
  async run(ctx: CommandContext<typeof options>) {
    const flags = ctx.options.hide ? MessageFlags.Ephemeral : undefined;
    const ping = ctx.client.gateway.latency;

    await ctx.write({
      content: `The latency is \`${ping}\``,
      flags,
    });
  }
}
```

### Required module augmentation (place once, e.g. in `src/index.ts`)

The upstream MDX hides this in a Twoslash `---cut---` preamble; it is REQUIRED in real projects for `ctx.client.gateway` to type-check, and matches current source exactly:

```ts
import type { ParseClient, Client } from 'seyfert';

declare module 'seyfert' {
  interface SeyfertRegistry {
    client: ParseClient<Client<true>>;
  }
}
```

Mechanism (source-verified): `SeyfertRegistry` is an empty augmentable interface (`src/client/plugins/types.ts:32`). `UsingClient = BaseClient & RegisteredPluginExtension & RegisteredClient`, where `RegisteredClient` reads `SeyfertRegistry['client']` via the `ParseClient` marker symbol (`src/commands/applications/shared.ts`). Use `ParseClient<Client<true>>` for gateway bots, `ParseClient<HttpClient>` for HTTP, `ParseClient<WorkerClient<true>>` for workers.

### Required string option with choices (`as const`)

```ts
import {
  Command,
  Declare,
  Options,
  createStringOption,
  type CommandContext,
} from 'seyfert';

const options = {
  // lowercase key (compile-time enforced in v5)
  animal: createStringOption({
    description: 'Pick an animal',
    required: true,
    choices: [
      { name: 'Dog', value: 'dog' },
      { name: 'Cat', value: 'cat' },
    ] as const, // choices is readonly — `as const` narrows the value union
  }),
};

@Declare({ name: 'pick', description: 'Pick an animal' })
@Options(options)
export default class PickCommand extends Command {
  async run(ctx: CommandContext<typeof options>) {
    // required:true → ctx.options.animal is 'dog' | 'cat' (non-nullable)
    await ctx.write({ content: `You picked ${ctx.options.animal}` });
  }
}
```

### Autocomplete option

`value` on an autocomplete choice is typed to the option type — numeric for integer/number, string for string options.

```ts
import {
  Command,
  Declare,
  Options,
  createStringOption,
  type CommandContext,
} from 'seyfert';

const fruits = ['apple', 'banana', 'cherry'];

const options = {
  fruit: createStringOption({
    description: 'Search a fruit',
    required: true,
    autocomplete(interaction) {
      const focused = interaction.getInput().toLowerCase();
      const matches = fruits.filter(f => f.startsWith(focused)).slice(0, 25);
      return interaction.respond(
        matches.map(name => ({ name, value: name })),
      );
    },
  }),
};

@Declare({ name: 'fruit', description: 'Fruit picker' })
@Options(options)
export default class FruitCommand extends Command {
  async run(ctx: CommandContext<typeof options>) {
    await ctx.write({ content: `Selected: ${ctx.options.fruit}` });
  }
}
```

### Defer + edit for slow work (ephemeral)

When work may exceed Discord's ~3s window, defer first then edit.

```ts
import { Command, Declare, type CommandContext } from 'seyfert';

@Declare({ name: 'stats', description: 'Crunch some numbers' })
export default class StatsCommand extends Command {
  async run(ctx: CommandContext) {
    await ctx.deferReply(true); // true = ephemeral "Thinking..."
    const data = await expensiveQuery();
    await ctx.editOrReply({ content: `Result: ${data}` });
  }
}
```

### Guild-only narrowing with `inGuild()`

```ts
import { Command, Declare, type CommandContext } from 'seyfert';

@Declare({
  name: 'serverinfo',
  description: 'Info about this server',
  contexts: ['Guild'], // restrict to guilds (string name accepted)
})
export default class ServerInfo extends Command {
  async run(ctx: CommandContext) {
    if (!ctx.inGuild()) {
      return ctx.write({ content: 'Guild only.' });
    }
    // ctx is now GuildCommandContext: guildId/member are present,
    // channel() resolves to a guild channel.
    const channel = await ctx.channel();
    await ctx.write({
      content: `Channel: ${channel.name} • Guild: ${ctx.guildId}`,
    });
  }
}
```

### Guild-scoped (instant) registration during development

Global uploads can take up to an hour to propagate; a `guildId` makes the command appear instantly in that guild.

```ts
@Declare({
  name: 'dev',
  description: 'Dev-only test command',
  guildId: ['123456789012345678'], // appears instantly in this guild
})
export default class Dev extends Command { /* ... */ }
```

## Common patterns / gotchas

- `start()` does NOT upload commands. Always chain `uploadCommands(...)` (or call it elsewhere) to register them with Discord. The `cachePath` snapshot lets Seyfert skip re-uploading unchanged commands.
- Option record keys MUST be lowercase — a compile-time error in v5 (Discord rejects non-lowercase names at deploy time otherwise).
- `choices` and `channel_types` are `readonly` — add `as const` so the value union narrows correctly.
- Autocomplete choice `value` is typed to the option type: integer/number options take numbers, string options take strings. The old v4 trick of returning `'four'` on an integer option no longer type-checks.
- Always type the options generic: `CommandContext<typeof options>`. Without it, `ctx.options` is untyped.
- `write` / `editOrReply` / `deferReply` return `void` unless you pass the response flag as `true` — only then do you get a message/webhook object back. Don't `await` them expecting a `Message` by default.
- Ephemeral replies: `flags: MessageFlags.Ephemeral` on `write`, or pass `true` as the first arg to `deferReply`.
- `ctx.modal(...)` throws `SeyfertError('CANNOT_USE_MODAL')` if the command ran as a prefix/text command (no interaction). Guard with `ctx.inGuild()`/interaction presence if you support prefix commands.
- `ctx.client.gateway` only exists on the gateway `Client`. HTTP-only bots (`HttpClient`) have no `gateway` — augment `SeyfertRegistry` with `ParseClient<HttpClient>` and read latency differently. Without the augmentation, `ctx.client` is `BaseClient` and `.gateway` won't type-check.
- Context-menu commands: `@Declare({ type: ApplicationCommandType.User | .Message, name })` — no `description` allowed for menus.
- For middleware on a command, use `@Middlewares([...])` (readonly tuple accepted in more-qol) and the typed `middlewares('a','b')` helper. v5 removed `pass()`; use `stop()` (no arg) for a silent skip, `stop('reason')` to deny.

## Doc vs Source Corrections

- None. Every API, decorator, option helper, and example in the MDX matches current `more-qol` source.
- The MDX option example contains a Twoslash compiler directive comment (`// @exactOptionalPropertyTypes: false`) — strip it from copied code; it is not runtime code.
- The MDX also hides the `SeyfertRegistry` augmentation behind `---cut---`; surfaced above because real projects need it.
- Folder name (`commands`) is configurable via `seyfert.config` `locations.commands`; the literal `commands` folder is the documented default, not hardcoded.

## Source Anchors

- `src/index.ts` (root barrel: `export * from './commands'`, `./types`, `Client` re-export)
- `src/commands/decorators.ts` (`Declare`, `Options`, `Middlewares`, `middlewares`)
- `src/commands/applications/options.ts` (`createBooleanOption`/`createStringOption` + sibling creators, choices/validators)
- `src/commands/applications/chat.ts` (`Command`, `AutocompleteCallback`)
- `src/commands/applications/shared.ts` (`UsingClient`, `RegisteredClient`, `ParseClient`)
- `src/client/plugins/types.ts` (`SeyfertRegistry`)
- `src/commands/applications/chatcontext.ts` (`write`, `deferReply`, `editOrReply`, `editResponse`, `followup`, `modal`, `inGuild`, `channel`, `fetchMember`)
- `src/commands/basecontext.ts` (`ctx.client`, `ctx.options`)
- `src/structures/Interaction.ts` (`AutocompleteInteraction.respond`)
- `src/client/base.ts` (`uploadCommands`, `gateway` field)
- `src/client/client.ts` (`start`)
- `src/websocket/discord/sharder.ts` (`get latency`)
- `tests/command-context-client-type.test.mts` (confirms `ParseClient<Client<true>>` augmentation contract)

## Agent Guidance

- Commands must be a default-export class extending `Command` (or `SubCommand` / context-menu / entry-point variants) inside the configured commands folder; Seyfert loads them on `start()`.
- `@Declare` requires `name` + `description` for chat-input commands; the name is forced lowercase at the type level. For user/message context menus, use `@Declare({ type: ApplicationCommandType.User | .Message, name })` (no description).
- Always type the options generic: `CommandContext<typeof options>`.
- Reply with `ctx.write` for the first response; use `ctx.editOrReply` / `ctx.followup` for later messages. Ephemeral = `flags: MessageFlags.Ephemeral`.
- `start()` ≠ register. Call `uploadCommands` (commonly chained after `start()`); `cachePath` avoids redundant uploads via a JSON diff. Use `guildId` in `@Declare` for instant dev registration.
- `ctx.client.gateway` only exists on the gateway `Client`; augment `SeyfertRegistry` accordingly. Without the augmentation, `ctx.client` falls back to `BaseClient`.
- `more-qol` branch notes: `@Options` / middleware decorators accept readonly arrays; middleware `pass()` was replaced by `stop()` (no arg = silent skip).
