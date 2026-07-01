# Prefix (Message) Commands

Original source URL: https://seyfert-web-git-seyfert-v5-tiramisulabs.vercel.app/docs/learn/commands/prefix-commands
Coverage reference: commands.md
Verification status: Source-verified (seyfert-core, the authoritative Seyfert source)

## Page Summary

Seyfert dispatches the same `Command`/`SubCommand` classes used for application commands from plain text messages ("prefix commands"). They are opt-in: you supply `commands.prefix` in `ClientOptions` (a callback that receives the `Message` and returns the possible prefixes). To get `ctx.message` typed you must augment `InternalOptions` with `withPrefix: true`. Response routing (`reply`), deferral text (`deferReplyResponse`), and option parsing are all configurable; the default option parser lives on the `HandleCommand` service, which is overridable via `client.setServices`.

A single command class can run as BOTH a slash command and a prefix command — there is one `run(ctx)`. The context object abstracts the difference: `ctx.interaction` is present for slash, `ctx.message` is present for prefix, and the response helpers (`write`/`editOrReply`/`deferReply`/`followup`) route to the right transport automatically.

## Key APIs (verified)

- `commands.prefix?: (message: MessageStructure) => Awaitable<string[]>` — `src/client/client.ts:317`. Return value may be sync or a Promise. Returned prefixes are sorted longest-first and matched with `content.startsWith` (`src/commands/handle.ts:368-369`).
- `commands.reply?: (ctx: CommandContext) => Awaitable<boolean>` — `src/client/client.ts:319`. When it resolves truthy, the first response uses `message.reply`; otherwise `message.write` (plain channel send). Used by `ctx.write`/`ctx.deferReply` (`src/commands/applications/chatcontext.ts:95,122`).
- `commands.deferReplyResponse?: (ctx: CommandContext) => Awaitable<Parameters<Message['write']>[0]>` — `src/client/client.ts:318`. Supplies the placeholder body sent by `ctx.deferReply()` for prefix commands. Default fallback is `{ content: 'Thinking...' }` (`chatcontext.ts:123`).
- `interface InternalOptions {}` + `type InferWithPrefix = InternalOptions extends { withPrefix: infer P } ? P : false` — `src/commands/applications/shared.ts:23,52`. Augmenting it with `withPrefix: true` flips `ctx.message`/`ctx.interaction` from `undefined`-only to `MessageStructure | undefined` / `ChatInputCommandInteraction | undefined` (`chatcontext.ts:49-52`).
- `enum IgnoreCommand { Slash = 0, Message = 1 }` — `src/commands/applications/shared.ts:133`. Set via `@Declare({ ignore })`. `IgnoreCommand.Message` hides a command from the prefix resolver (`handle.ts:606`); `IgnoreCommand.Slash` hides it from application-command registration (text-only command).
- `Command.aliases?: string[]` (`chat.ts:142`) and `Command.groupsAliases?: Record<string, string>` (`chat.ts:365`) — alternate names resolved by the prefix parser. SubCommands also support `aliases` (`handle.ts:569,586`). Aliases are prefix-only; they do not register with Discord.
- `class HandleCommand` — defined in `src/commands/handle.ts:64`. NOT root-exported: the `commands` barrel re-exports only `type CommandFromContent` from this file (`src/commands/index.ts:11`). To subclass it use a deep import: `import { HandleCommand } from 'seyfert/lib/commands/handle'`.
  - `argsParser(content: string, command, message): Record<string, string>` — `handle.ts:492`. Default regex `/-(.*?)(?=\s-|$)/gs`: options are introduced by `-name` and the value runs until the next ` -` or end of string. Maps parsed `{ name: value }` onto declared option names.
  - Override and register via `client.setServices({ handleCommand: MyHandleCommand })` — pass the class, NOT an instance (`src/client/base.ts:309,1400-1405`). (v5 change: it takes the constructor.)
- `ctx.modal(...)` throws `SeyfertError('CANNOT_USE_MODAL')` ("Cannot use modal without an interaction.") when invoked from a prefix command (no interaction) — `chatcontext.ts:102`, `src/common/it/error.ts:101`. Narrow with `SeyfertError.is(err, 'CANNOT_USE_MODAL')` if you catch it.

### Context behavior under prefix vs slash (verified)

- `ctx.write(body, withResponse?)` / `ctx.editOrReply(body, withResponse?)` — return `void` unless `withResponse` is `true`. For prefix, `withResponse: true` returns a `MessageStructure`; for slash it returns a `WebhookMessageStructure` (`chatcontext.ts:88-97,145-154`).
- `ctx.editResponse(body)` STILL EXISTS on `CommandContext` (only `ComponentContext.editResponse` was removed in v5) — `chatcontext.ts:126`.
- `ctx.deferReply(ephemeral?, withResponse?)` — for prefix, `ephemeral` is ignored (there is no ephemeral message); it sends `deferReplyResponse` (or `Thinking...`) and sets `__deferred` (`chatcontext.ts:107-124`).
- `ctx.author` / `ctx.guildId` / `ctx.channelId` / `ctx.member` all resolve from `interaction ?? message` (`chatcontext.ts:236-254`).
- `ctx.inGuild()` narrows to `GuildCommandContext` (`chatcontext.ts:260`); `ctx.t` resolves `defaultLang` for prefix (no `interaction.locale`/`guildLocale`) — `chatcontext.ts:72-78`.

## Code Examples (verified)

### 1. Configure prefixes (root import from `seyfert`)

```ts
import { Client } from 'seyfert';

const client = new Client({
    commands: {
        // Static prefixes — sorted longest-first, matched with startsWith.
        prefix: () => ['!', '?', '.'],
    },
});
```

Async / per-guild prefixes are fully supported (the signature is `Awaitable<string[]>`):

```ts
const client = new Client({
    commands: {
        prefix: async (message) => {
            const fromDb = message.guildId ? await db.getPrefix(message.guildId) : null;
            // Always include a bot-mention prefix as a fallback.
            return [fromDb ?? '!', `<@${client.botId}>`];
        },
    },
});
```

### 2. Enable `ctx.message` typing + a text-only command

```ts
// Place this augmentation once (e.g. alongside your other declare module blocks).
declare module 'seyfert' {
    interface InternalOptions {
        withPrefix: true;
    }
}

import { Command, type CommandContext, Declare, IgnoreCommand } from 'seyfert';

@Declare({
    name: 'crosspost',
    description: 'Crosspost an announcement message.',
    ignore: IgnoreCommand.Slash, // text-only: never registered as a slash command
    aliases: ['cp', 'publish'], // prefix-only alternate names
})
export default class CrosspostCommand extends Command {
    async run(ctx: CommandContext) {
        // ctx.message is MessageStructure | undefined thanks to withPrefix.
        if (ctx.message) await ctx.message.crosspost();
        return ctx.write({ content: 'I have crossposted your announcement.' });
    }
}
```

### 3. Full client with reply routing + deferral text

```ts
import { Client } from 'seyfert';

const client = new Client({
    commands: {
        prefix: () => ['!'],
        // First response uses message.reply (mention reply) instead of a plain send.
        reply: () => true,
        // Placeholder shown by ctx.deferReply() before the real answer.
        deferReplyResponse: () => ({ content: 'Please wait, processing your request...' }),
    },
});
```

### 4. Options — default `-flag value` parser

Parsed input: `!crosspost -id 123456789`

```ts
import { Command, type CommandContext, Declare, Options, createStringOption } from 'seyfert';

const options = {
    // Keys MUST be lowercase (v5 compile-time rule).
    id: createStringOption({
        description: 'The ID of the message we are going to crosspost',
        required: true,
    }),
};

@Declare({ name: 'crosspost', description: 'Crosspost an announcement message.' })
@Options(options)
export default class CrosspostCommand extends Command {
    async run(ctx: CommandContext<typeof options>) {
        await ctx.deferReply(); // sends deferReplyResponse, or 'Thinking...' by default
        await ctx.client.messages.crosspost(ctx.options.id, ctx.channelId);
        return ctx.editOrReply({ content: 'I have crossposted your announcement.' });
    }
}
```

### 5. Dual-mode command — gate interaction-only features

The same class can run via slash OR prefix. `ctx.modal()` throws on prefix, so branch on `ctx.interaction`.

```ts
import { Command, type CommandContext, Declare, Modal, Label, TextInput } from 'seyfert';
import { TextInputStyle } from 'seyfert';

@Declare({ name: 'feedback', description: 'Send feedback.' })
export default class FeedbackCommand extends Command {
    async run(ctx: CommandContext) {
        if (ctx.interaction) {
            // Slash path: a modal is available.
            const modal = new Modal()
                .setCustomId('feedback-modal')
                .setTitle('Feedback')
                .setComponents([
                    new Label()
                        .setLabel('Your feedback')
                        .setComponent(
                            new TextInput()
                                .setCustomId('body')
                                .setStyle(TextInputStyle.Paragraph),
                        ),
                ]);
            return ctx.modal(modal); // safe: interaction exists
        }
        // Prefix path: no modal, ask in-channel instead.
        return ctx.write({ content: 'Reply with your feedback in the next message.' });
    }
}
```

### 6. Guild-only logic with `inGuild()` narrowing

```ts
import { Command, type CommandContext, Declare } from 'seyfert';

@Declare({ name: 'serverinfo', description: 'Show server info.' })
export default class ServerInfo extends Command {
    async run(ctx: CommandContext) {
        if (!ctx.inGuild()) {
            return ctx.write({ content: 'This command only works in a server.' });
        }
        // ctx is now GuildCommandContext: guildId/member are non-undefined,
        // and channel()/guild() narrow to guild types.
        const guild = await ctx.guild();
        return ctx.write({
            content: `${guild.name} — requested by ${ctx.member.user.username}`,
        });
    }
}
```

### 7. Subcommands + aliases over prefix

The resolver reads up to 3 leading words (`parent`, `group`, `sub`) and honors `aliases` / `groupsAliases` (`handle.ts:505-597`). Same classes you use for slash subcommands.

```ts
import { Command, SubCommand, type CommandContext, Declare, Options, AutoLoad } from 'seyfert';

@Declare({ name: 'ban', description: 'Ban a user.', aliases: ['b'] })
export class BanSub extends SubCommand {
    async run(ctx: CommandContext) {
        return ctx.write({ content: 'banned' });
    }
}

// !mod ban ...   or   !mod b ...   (subcommand alias)
@Declare({ name: 'mod', description: 'Moderation.', aliases: ['m'] })
@AutoLoad()
export default class ModCommand extends Command {}
```

### 8. Custom arg parser — positional args (deep import required)

`HandleCommand` is not root-exported; subclass via the deep import and return a `{ optionName: rawValue }` record. Seyfert then runs each declared option's resolver/validation over those raw strings.

```ts
import { HandleCommand } from 'seyfert/lib/commands/handle';
import type { Command, SubCommand } from 'seyfert';

class PositionalHandleCommand extends HandleCommand {
    // e.g. `!give @user 100` -> { target: '<@id>', amount: '100' } in declared order
    argsParser(content: string, command: Command | SubCommand) {
        const declared = (command.options ?? []) as { name: string }[];
        const parts = content.split(/\s+/).filter(Boolean);
        const args: Record<string, string> = {};
        declared.forEach((opt, i) => {
            if (parts[i] !== undefined) args[opt.name] = parts[i];
        });
        return args;
    }
}

// During setup — pass the CLASS, not an instance:
client.setServices({ handleCommand: PositionalHandleCommand });
```

## Common patterns / gotchas

- ALWAYS add `declare module 'seyfert' { interface InternalOptions { withPrefix: true } }`, or `ctx.message` is typed `undefined` and you cannot touch message-only APIs. The augmentation is global — one block for the whole bot.
- Boolean options parse loosely: `['yes', 'y', 'true', 'treu']` (the `treu` typo is intentional in src) → `true`; anything else → `false` (`handle.ts:788`). There is no `-flag` shorthand-without-value; you must write `-active yes`.
- The default parser is greedy `-name value` and consumes up to the next ` -`. Quotes are NOT special. For natural-language / positional / quoted parsing, either subclass `HandleCommand` (Example 8) or use the external `yunaforseyfert` package (verify its version in your project).
- Prefix dispatch enforces the same gates as slash before `run`: `command.contexts` (Guild / BotDM), `command.guildId` scoping (`handle.ts:384-386`), and member/bot permission checks (`defaultMemberPermissions` / `botPermissions`) resolved from the gateway message (`handle.ts:442-459`). The guild owner bypasses the member-permission check. `onOptionsError` / `onPermissionsFail` / `onBotPermissionsFail` / middlewares all fire normally.
- `ctx.modal()` throws `CANNOT_USE_MODAL` on prefix — gate any modal on `if (ctx.interaction)`. Component collectors (buttons/selects) DO work in prefix commands because they attach to the sent message, not the (absent) interaction.
- Locale: prefix invocations have no per-user/guild locale, so `ctx.t` resolves `langs.defaultLang` (`chatcontext.ts:72-78`). Even with `langs.preferGuildLocale: true`, prefix still falls through to `defaultLang` because `ctx.interaction` is undefined. Don't rely on per-user language in text commands.
- Need a text-only command? `ignore: IgnoreCommand.Slash`. Need a slash-only command in a bot that has prefixes enabled globally? `ignore: IgnoreCommand.Message`.
- Always include a bot-mention prefix (`<@${client.botId}>`) as a fallback so users can recover when they forget the configured prefix.

## Doc vs Source Corrections

- Docs say `deferReply()` shows a `Loading...` message by default → src default placeholder is `{ content: 'Thinking...' }` (`chatcontext.ts:123`).
- Docs imply `HandleCommand` is importable like other classes → src only exports the `CommandFromContent` type from the barrel; the class needs a deep import `seyfert/lib/commands/handle` (`src/commands/index.ts:11`).
- Docs show `prefix` returning a synchronous array → src signature is `Awaitable<string[]>`, so async prefix resolvers are supported (`client.ts:317`).
- v5: `setServices({ handleCommand })` takes the `HandleCommand` CONSTRUCTOR, not an instance (`base.ts:1400-1405`). v4 passed `new HandleCommand(client)`.
- Docs don't mention it: prefix dispatch also enforces `command.contexts` / `command.guildId` and runs member/bot permission checks before running (`handle.ts:384-386,442-459`); the guild owner bypasses the member-permission check.

## Source Anchors

- `src/commands/handle.ts` — `HandleCommand`, `message()` (entry, `:364`), `argsParser` (`:492`), `resolveCommandFromContent` (`:500`), `resolveCommandFromNameParts` (`:547`), `argsOptionsParser`/boolean parse (`:788`), prefix sort+match (`:368-369`), `IgnoreCommand.Message` filtering (`:606`), permission gates (`:442-459`).
- `src/client/client.ts:316-320` — `commands.prefix` / `deferReplyResponse` / `reply` option types.
- `src/commands/applications/shared.ts:23,52,133` — `InternalOptions`, `InferWithPrefix`, `IgnoreCommand` enum.
- `src/commands/applications/chatcontext.ts` — `message`/`interaction` typing (`:49-52`), `t` (`:72-78`), `write` (`:88`), `modal` (`:99-105`), `deferReply` (`:107`), `editResponse` (`:126`), `editOrReply` (`:145`), `author/guildId/channelId/member` (`:236-254`), `inGuild` (`:260`).
- `src/commands/applications/chat.ts:142,365` — `aliases`, `groupsAliases`.
- `src/commands/index.ts:11` — barrel export (type-only) confirming `HandleCommand` class is not root-exported.
- `src/client/base.ts:309,1400-1405` — `setServices({ handleCommand })` (constructor), default `HandleCommand`.
- `src/common/it/error.ts:101` — `CANNOT_USE_MODAL` message.

## Agent Guidance

- Prefix commands reuse the exact same `Command`/`SubCommand`/`@Options` classes as slash commands; you do not write separate handlers. Write one `run(ctx)`; branch on `ctx.interaction` / `ctx.message` only for transport-specific bits (modals, `message.crosspost()`, etc.).
- Always add the `withPrefix: true` augmentation or `ctx.message` is `undefined`-typed.
- Guard message-only logic with `if (ctx.message)` and modal/interaction-only logic with `if (ctx.interaction)`.
- Prefer `ctx.author`, `ctx.guildId`, `ctx.channelId`, `ctx.member`, `ctx.write/editOrReply/deferReply/followup` over reaching into `ctx.message`/`ctx.interaction` — they already abstract both transports.
- The default option parser is `-flag value` greedy-to-next-`-`. For positional/quoted/natural parsing, subclass `HandleCommand` (deep import) and register with `client.setServices({ handleCommand: MyClass })` (the class). Many bots use `yunaforseyfert` (external — verify version in target project) for richer parsing.
- Don't rely on per-user locale in prefix commands; `ctx.t` uses `defaultLang`.
