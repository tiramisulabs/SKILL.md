# Introduction to Commands

Original source URL: https://seyfert-web-git-seyfert-v5-tiramisulabs.vercel.app/docs/learn/commands/intro

Coverage reference: commands.md

Verification status: Source-verified against ./src (the authoritative Seyfert source) + v5 checklist

## Page Summary

Commands are the primary entry point for a Seyfert bot. They are class-based: every top-level command extends `Command`, and each subcommand extends the abstract `SubCommand`. Configuration is done with TypeScript decorators (`@Declare`, plus `@Options`, `@Group`/`@Groups`, `@Middlewares`, `@AutoLoad`, `@Locales`). `@Declare` carries the command's metadata; `name` and `description` are mandatory for chat-input commands. The intro page focuses on the `@Declare` metadata surface; options/subcommands/middleware/groups have their own pages.

## Key APIs (verified)

All of these are re-exported from the package root `'seyfert'` (via `src/commands/index.ts` barrel and `src/index.ts`). No deep import needed.

- `class Command extends BaseCommand` — `src/commands/applications/chat.ts:361`. Default `type = ApplicationCommandType.ChatInput`. Holds `groups`, `groupsAliases`, optional `run(context: CommandContext)`.
- `abstract class SubCommand extends BaseCommand` — `chat.ts:408`. `type = ApplicationCommandOptionType.Subcommand`, optional `group`, and an `abstract run(context: CommandContext)` (subcommands MUST implement `run`).
- `Declare(input)` decorator — `decorators.ts:195`. Accepts `CommandDeclareOptions`.
- `DecoratorDeclareOptions` (the chat-input shape) — `decorators.ts:33`:
  - `name: string` (required)
  - `description: string` (required)
  - `botPermissions?: PermissionStrings`
  - `defaultMemberPermissions?: PermissionStrings`
  - `guildId?: string[]`
  - `nsfw?: boolean`
  - `integrationTypes?: (keyof typeof ApplicationIntegrationType)[] | ApplicationIntegrationType[]`
  - `contexts?: (keyof typeof InteractionContextType)[] | InteractionContextType[]`
  - `ignore?: IgnoreCommand`
  - `aliases?: string[]`
  - `props?: ExtraProps`
- `CommandDeclareOptions` union — `decorators.ts:24`: chat-input shape, OR a context-menu shape (`type: ApplicationCommandType.User | Message` with `description` OMITTED), OR an entry-point shape (`type: ApplicationCommandType.PrimaryEntryPoint` + `handler`, with `ignore`/`aliases`/`guildId` omitted).
- `enum IgnoreCommand` — `src/commands/applications/shared.ts`: `Slash = 0`, `Message = 1`.
- Sibling decorators (own pages, all exported from `'seyfert'`): `Options` (`decorators.ts:161`), `Group`/`Groups`/`GroupsT` (`decorators.ts:151/107/91`), `Middlewares` + `middlewares` helper (`decorators.ts:188/184`), `AutoLoad` (`decorators.ts:177`), `Locales`/`LocalesT` (`decorators.ts:47/61`), `defineGroups` (`decorators.ts:85`).
- Option helpers (own page): `createStringOption`, `createIntegerOption`, `createNumberOption`, `createChannelOption`, `createBooleanOption`, `createUserOption`, `createRoleOption`, `createMentionableOption`, `createAttachmentOption` (`options.ts:146-199`).

### Behavior applied by `@Declare` (verified, decorators.ts:195-231)

- `name` for chat-input is coerced to lowercase at the type level (`LowercaseDeclareName`); User/Message menu names are NOT lowercased.
- `defaultMemberPermissions` and `botPermissions` are resolved from string arrays to a `bigint` via `PermissionsBitField.resolve(...)` (stored as bigint, not the original array). In v5 the static `resolve` throws on an unknown permission string — typos fail loudly.
- `contexts` string keys are mapped to `InteractionContextType` numbers; default when omitted = all `InteractionContextType` values.
- `integrationTypes` string keys are mapped to `ApplicationIntegrationType` numbers; default when omitted = `[ApplicationIntegrationType.GuildInstall]` (server-only).
- `type` defaults to `ApplicationCommandType.ChatInput`; `description` defaults to `''`.

### Lifecycle hooks on `BaseCommand` (optional overrides, chat.ts:349-358)

Override these on the command class (signatures verified):
- `onBeforeMiddlewares(ctx)`, `onBeforeOptions(ctx)`
- `run(ctx)` — the main handler
- `onAfterRun(ctx, error)` — always runs after `run`
- `onRunError(ctx, error)` — `run` threw
- `onOptionsError(ctx, metadata)` — option parsing/validation failed
- `onMiddlewaresError(ctx, error: string, metadata)` — a middleware called `stop('reason')`; `metadata.middleware` names the one that denied
- `onBotPermissionsFail(ctx, permissions)` / `onPermissionsFail(ctx, permissions)`
- `onInternalError(client, command, error?)` — note the v5 `command` param sits before `error`

## Code Examples (verified)

### 1. Chat-input command (full `@Declare` surface)

```ts
import { Declare, Command, IgnoreCommand, type CommandContext } from 'seyfert';

@Declare({
    name: 'your-command',          // must be lowercase for chat-input
    description: 'A description for this command',
    props: {},                     // ExtraProps metadata bag
    defaultMemberPermissions: ['Administrator'],
    botPermissions: ['ManageGuild'],
    guildId: ['100000'],
    nsfw: false,
    aliases: ['an-alias'],         // text-command alternate names
    integrationTypes: ['GuildInstall', 'UserInstall'],
    contexts: ['BotDM', 'Guild', 'PrivateChannel'],
    ignore: IgnoreCommand.Slash,   // Slash = 0, Message = 1
})
export default class MyCommand extends Command {
    async run(ctx: CommandContext) {
        await ctx.write({ content: 'hello' });
    }
}
```

### 2. Context-menu (User) command — `description` MUST be omitted

```ts
import { Declare, Command, type MenuCommandContext, ApplicationCommandType, type UserCommandInteraction } from 'seyfert';

@Declare({
    name: 'My User Command',                 // not forced lowercase for menus
    type: ApplicationCommandType.User,
})
export default class MyUserCtx extends Command {
    async run(ctx: MenuCommandContext<UserCommandInteraction>) {
        await ctx.write({ content: `Target: ${ctx.target.username}` });
    }
}
```

> In source, setting `type: User | Message` uses `Omit<DecoratorDeclareOptions, 'description'>`, so a menu command cannot declare `description` (the doc's commented-out `/// type: ApplicationCommandType.User` hint is what would trigger this).

### 3. Command with options + the typed context generic

`@Options` takes a lowercase-keyed record; the option type flows into `ctx.options` via the `CommandContext<typeof options>` generic.

```ts
import {
    Command, Declare, Options, createStringOption, createUserOption,
    type CommandContext,
} from 'seyfert';

const options = {
    reason: createStringOption({ description: 'Why?', required: true }),
    target: createUserOption({ description: 'Who?', required: true }),
    // KEY: 'User' here would be a COMPILE error in v5 — keys must be lowercase.
};

@Declare({ name: 'warn', description: 'Warn a member' })
@Options(options)
export default class WarnCommand extends Command {
    async run(ctx: CommandContext<typeof options>) {
        const { reason, target } = ctx.options; // fully typed: string, User
        await ctx.write({ content: `Warned ${target.username}: ${reason}` });
    }
}
```

### 4. Subcommands + groups (`@Options` of classes, `@Group`)

A parent `Command` declares groups and lists child `SubCommand` classes; each subcommand pins itself to a group with `@Group('name')`. `@Options(...)` here uses the `(new () => SubCommand)[]` overload (`decorators.ts:161`).

```ts
// commands/config/index.ts
import { Command, Declare, Options, Groups } from 'seyfert';
import { SetPrefix } from './set-prefix';
import { ResetPrefix } from './reset-prefix';

@Declare({ name: 'config', description: 'Server configuration' })
@Groups({
    prefix: { defaultDescription: 'Manage the command prefix' },
})
@Options([SetPrefix, ResetPrefix])
export default class ConfigCommand extends Command {}
```

```ts
// commands/config/set-prefix.ts
import { SubCommand, Declare, Group, Options, createStringOption, type CommandContext } from 'seyfert';

const options = { value: createStringOption({ description: 'New prefix', required: true }) };

@Declare({ name: 'set', description: 'Set the prefix' })
@Group('prefix')
@Options(options)
export class SetPrefix extends SubCommand {
    async run(ctx: CommandContext<typeof options>) {
        await ctx.editOrReply({ content: `Prefix set to ${ctx.options.value}` });
    }
}
```

`SubCommand.run` is `abstract` — every subcommand must implement it, unlike `Command.run` which is optional.

### 5. Guild-only command with `inGuild()` narrowing + lifecycle hooks

`ctx.inGuild()` is a type guard returning `this is GuildCommandContext`, which makes `guildId`/`member` non-optional and narrows `channel()`/`fetchMember()`/`guild()`/`me()` to non-undefined guild returns (`chatcontext.ts:260`, `GuildCommandContext` at `:265`).

```ts
import { Command, Declare, type CommandContext } from 'seyfert';

@Declare({
    name: 'kick',
    description: 'Kick a member',
    contexts: ['Guild'],            // slash-level guard
    botPermissions: ['KickMembers'],
    defaultMemberPermissions: ['KickMembers'],
})
export default class KickCommand extends Command {
    async run(ctx: CommandContext) {
        if (!ctx.inGuild()) return; // runtime + type narrowing
        // ctx is now GuildCommandContext: these are non-undefined
        const me = await ctx.fetchMember();      // GuildMemberStructure
        const channel = await ctx.channel();     // GuildCommandChannel
        await ctx.write({ content: `In ${channel.name}, ${me.user.username} ready.` });
    }

    // Runs if the BOT lacks botPermissions
    async onBotPermissionsFail(ctx: CommandContext, permissions: import('seyfert/lib/common').PermissionStrings) {
        await ctx.write({ content: `I'm missing: ${permissions}`, flags: 64 });
    }

    async onRunError(ctx: CommandContext, error: unknown) {
        ctx.client.logger.error(error);
        await ctx.editOrReply({ content: 'Something went wrong.' }).catch(() => {});
    }
}
```

### 6. Autoloaded subcommands

`@AutoLoad()` tells the loader to pull default-exported `SubCommand` files from the parent command's own folder tree (sets `__autoload = true`), so you skip the explicit `@Options([...])` class list. Because the scan is recursive from `dirname(parentFile)`, put the parent in a dedicated folder and keep helper-only files outside that folder.

```ts
// commands/tag/tag.ts
import { Command, Declare, AutoLoad } from 'seyfert';

@Declare({ name: 'tag', description: 'Tag system' })
@AutoLoad()
export default class TagCommand extends Command {}
```

```ts
// commands/tag/commands/create.ts
import { Declare, SubCommand, type CommandContext } from 'seyfert';

@Declare({ name: 'create', description: 'Create a tag' })
export default class CreateTag extends SubCommand {
    async run(ctx: CommandContext) {
        await ctx.write({ content: 'created' });
    }
}
```

## Common patterns / gotchas

- **`export default` the top-level command.** The loader resolves `x.default ?? x` (`chat.ts:344`). Subcommand classes passed to `@Options([...])` can be named exports.
- **Decorator order doesn't matter**, but every command needs `@Declare`. Subcommands need `@Declare` + (usually) `@Group`; the parent needs `@Groups`/`@Group` defining the group keys those children reference.
- **`@AutoLoad()` scans recursively from the parent file's folder.** Isolate the parent (`commands/tag/tag.ts` + `commands/tag/commands/*.ts`) and keep helpers (`shared.ts`, group maps, resolvers) outside that folder.
- **Option keys must be lowercase** — enforced at compile time in v5 via `LowercaseOptionsRecord` (`decorators.ts:135`). `{ User: ... }` is a type error.
- **`choices` / `channel_types`** on options are `readonly`; use `as const` when defining them inline.
- **Permissions read-back:** after `@Declare`, `defaultMemberPermissions`/`botPermissions` are stored as resolved `bigint`, not the original string array. A bad permission string throws (static `PermissionsBitField.resolve`).
- **`write` / `editOrReply` return `void`** unless you pass the response flag `true` (`write(body, true)` → returns the message/webhook). This is the v5 narrowing — don't assume a `Message` comes back by default (`chatcontext.ts:88,145`).
- **Modals from a context:** `ctx.modal(body, { waitFor })` works only on interaction contexts; on a prefix/message context it throws `SeyfertError('CANNOT_USE_MODAL')` (`chatcontext.ts:99-105`). Catch with `SeyfertError.is(err, 'CANNOT_USE_MODAL')`.
- **Context fetch modes:** `channel`/`fetchMember`/`guild`/`me` accept `'cache' | 'rest' | 'flow'` (default `'flow'`). `'cache'` is sync (returns the value / a `ReturnCache`), `'rest'`/`'flow'` return a Promise.
- **Menu contexts** use `MenuCommandContext` with `ctx.target`, not `CommandContext`. Entry-point commands require `handler` and forbid `ignore`/`aliases`/`guildId`.
- **`ignore: IgnoreCommand.Slash`** disables the slash version (text-only); `IgnoreCommand.Message` disables the text version (slash-only).

## Doc vs Source Corrections

- Doc imports `Declare, Command, IgnoreCommand` from `'seyfert'` -> CORRECT (all root exports).
- Doc lists `props`, `defaultMemberPermissions`, `botPermissions`, `guildId`, `nsfw`, `aliases`, `integrationTypes`, `contexts`, `ignore` -> all match `DecoratorDeclareOptions` (`decorators.ts:33`).
- Doc comment "default is server-only" for `integrationTypes` -> CORRECT: src defaults to `[ApplicationIntegrationType.GuildInstall]` (`decorators.ts:205-207`).
- Doc implies `type: ApplicationCommandType.User` can sit alongside `description` -> src FORBIDS it: menu variant omits `description` (`decorators.ts:26-28`). (Correction.)
- Doc does not mention permission arrays are stored as resolved `bigint` -> src resolves via `PermissionsBitField.resolve` (`decorators.ts:208-213`). (New detail.)
- Doc does not mention chat-input `name` is type-coerced to lowercase -> `LowercaseDeclareName` (`decorators.ts:139-149`). (New detail.)
- Doc omits the entry-point Declare variant (`type: PrimaryEntryPoint` + `handler`) -> present in src (`decorators.ts:29-32`). (New detail.)
- Doc shows the bare class `class MyCommand extends Command {}` (no `export default`) -> for real files you MUST `export default` (loader uses `x.default ?? x`, `chat.ts:344`). (Clarification.)

## Source Anchors

- `src/structures/extra/Permissions.ts` (PermissionsBitField.resolve)

## Agent Guidance

- Use this page when scaffolding or reviewing top-level commands and their `@Declare` metadata. For option helpers, subcommands, groups, and middleware, follow the dedicated pages.
- For `@AutoLoad()`, use a dedicated parent folder and default-export every leaf `SubCommand`; use explicit `@Options([...])` when you want named exports or same-file examples.
- v5 essentials to enforce in generated code: option keys lowercase; `write`/`editOrReply` return `void` unless the response flag is `true`; middleware control flow uses `stop()` (no `pass()`); `onInternalError(client, command, error?)` has `command` before `error`; `ctx.inGuild()` narrows to `GuildCommandContext`. Channel structures also gained `isGuild()`/`isNamed()` guards for narrowing `await ctx.channel()` results.
