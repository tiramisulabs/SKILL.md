# Context Menu Commands

Original source URL: https://seyfert-web-git-seyfert-v5-tiramisulabs.vercel.app/docs/learn/commands/context-menus

Coverage reference: commands.md

Verification status: Source-verified (seyfert-core, branch more-qol)

## Page Summary

Context menu commands appear in Discord's right-click **Apps** submenu, targeting either a user or a message. They have NO options and NO user-visible description. Seyfert models them with the abstract `ContextMenuCommand` class plus `MenuCommandContext`, whose `ctx.target` resolves to the clicked `User` (a `UserStructure`) or `Message` (a `MessageStructure`). Unlike slash command names, a context-menu `@Declare.name` may contain spaces and uppercase letters. The `type` (`ApplicationCommandType.User` or `.Message`) is REQUIRED in `@Declare`, and `description` must be OMITTED — both are enforced at compile time by the `@Declare` overload.

## Key APIs (verified)

- `ContextMenuCommand` (abstract class) — `src/commands/applications/menu.ts`. Subclass and `export default`. Notable members:
  - `type!: ApplicationCommandType.User | ApplicationCommandType.Message` — REQUIRED, non-null, no default. `@Declare` only copies `type` when present (`if ('type' in declare)`); omitting it leaves `type` undefined and the command will not register correctly.
  - `abstract run?(context: MenuCommandContext<any>): any`
  - `middlewares: readonly (keyof ResolvedRegisteredMiddlewares)[] = []`
  - Optional hooks: `onBeforeMiddlewares(ctx)`, `onAfterRun(ctx, error)`, `onRunError(ctx, error)`, `onMiddlewaresError(ctx, error, metadata)`, `onBotPermissionsFail(ctx, permissions)`, `onInternalError(client, command, error?)`.
  - Config fields populated by `@Declare`: `name`, `nsfw`, `integrationTypes`, `contexts`, `defaultMemberPermissions?` (bigint), `botPermissions?` (bigint), `dm?`, `guildId?`, `name_localizations`, `description_localizations`, `props`.
  - Note: the class also declares a `description!: string` field, but for menu commands `@Declare` cannot receive `description` (see corrections) — it defaults to `''`.
- `MenuCommandContext<T extends MessageCommandInteraction | UserCommandInteraction, M = never>` — `src/commands/applications/menucontext.ts`. Notable members:
  - `get target(): InteractionTarget<T>` — builds a `Message` (for `ApplicationCommandType.Message`) or `User` (for `.User`) from `interaction.data.resolved` via `Transformers` on EVERY access (it is a getter, not a cached field — store it in a local const if you use it more than once).
  - `interaction: T`, `client`, `shardId`, `command: ContextMenuCommand`, `author` (`UserStructure`), `member` (`InteractionGuildMemberStructure | undefined`), `guildId`, `channelId`.
  - `get fullCommandName()` returns `this.command.name` (no subcommand path).
  - `get t()` — i18n proxy; respects `client.langs.preferGuildLocale` (guild locale first, else user locale).
  - `get deferred()` — `!!interaction.deferred`.
  - Response helpers: `write(body, withResponse?)`, `editOrReply(body, withResponse?)`, `editResponse(body)`, `deferReply(ephemeral?, withResponse?)`, `deleteResponse()`, `followup(body)`, `fetchResponse()`, `modal(body, options?)`, `channel(mode?)`, `me(mode?)`, `guild(mode?, query?)`.
  - `write` / `editOrReply` return `Promise<When<WR, WebhookMessageStructure, void>>`: by default (`withResponse` falsy) they resolve to `void`; pass `true` to get the `WebhookMessageStructure`. There is NO message-mode union (no prefix/message fallback) — a menu context always has an interaction.
  - `modal(body)` → `Promise<undefined>` (fire and forget); `modal(body, { waitFor })` → `Promise<ModalSubmitInteraction | null>` (awaits submission).
  - Type guards: `isMenu()` (always true here), `isMenuUser()` (→ `MenuCommandContext<UserCommandInteraction>`), `isMenuMessage()` (→ `MenuCommandContext<MessageCommandInteraction>`), `inGuild()` (→ `GuildMenuCommandContext`).
- `InteractionTarget<T>` — exported helper type: `T extends MessageCommandInteraction ? MessageStructure : UserStructure`.
- `GuildMenuCommandContext<T, M>` — guild-narrowed context with required `guildId`/`member` and a `guild()` that resolves to a non-undefined guild.
- `ApplicationCommandType.User` / `.Message`, `UserCommandInteraction`, `MessageCommandInteraction` — `src/structures/Interaction.ts` / `src/types`.
- `AnyContext` union includes `MenuCommandContext<...>` — `src/commands/applications/options.ts`.
- All of `ContextMenuCommand`, `MenuCommandContext`, `Declare`, `ApplicationCommandType`, `Command`, `AnyContext`, `UserCommandInteraction`, `MessageCommandInteraction`, `Modal`, `ActionRow`, `TextInput` import from the root `'seyfert'`. No deep imports needed.

## Code Examples (verified)

### User command

```ts
import {
  ContextMenuCommand,
  Declare,
  type MenuCommandContext,
  type UserCommandInteraction,
  ApplicationCommandType,
} from 'seyfert';

@Declare({
  name: 'User Info',
  type: ApplicationCommandType.User, // REQUIRED; no `description` allowed
})
export default class UserInfoCommand extends ContextMenuCommand {
  async run(ctx: MenuCommandContext<UserCommandInteraction>) {
    const target = ctx.target; // UserStructure — cache it (getter rebuilds each access)
    await ctx.write({
      content: `**${target.username}** (${target.id})\nCreated: <t:${Math.floor(target.createdTimestamp / 1000)}:R>`,
    });
  }
}
```

### Message command

```ts
import {
  ContextMenuCommand,
  Declare,
  type MenuCommandContext,
  type MessageCommandInteraction,
  ApplicationCommandType,
} from 'seyfert';

@Declare({
  name: 'Bookmark',
  type: ApplicationCommandType.Message,
})
export default class BookmarkCommand extends ContextMenuCommand {
  async run(ctx: MenuCommandContext<MessageCommandInteraction>) {
    const message = ctx.target; // MessageStructure
    await ctx.write({
      content: `Bookmarked from **${message.author.username}**: ${message.content.slice(0, 100)}`,
      flags: 64, // MessageFlags.Ephemeral — keep the bookmark private
    });
  }
}
```

### Restricting context

`contexts`/`integrationTypes` accept string keys (or enum values); `defaultMemberPermissions`/`botPermissions` accept a `PermissionStrings` array and are resolved to a bigint internally by `@Declare`.

```ts
@Declare({
  name: 'Report User',
  type: ApplicationCommandType.User,
  contexts: ['Guild'],                       // guild-only
  integrationTypes: ['GuildInstall'],        // guild-installed apps only
  defaultMemberPermissions: ['ModerateMembers'],
})
export default class ReportUserCommand extends ContextMenuCommand {
  async run(ctx: MenuCommandContext<UserCommandInteraction>) {
    await ctx.write({ content: `Reported ${ctx.target.username}`, flags: 64 });
  }
}
```

### Type guards over `AnyContext` (e.g. in middleware)

```ts
import { createMiddleware } from 'seyfert';

export const logTarget = createMiddleware<void>(({ context, next }) => {
  if (context.isMenu()) {
    // context: MenuCommandContext<User|Message interaction>
    const target = context.target;
    context.client.logger.info(`Context menu "${context.command.name}" -> ${target.id}`);
    // Narrow further:
    if (context.isMenuUser()) context.client.logger.debug(`user: ${context.target.username}`);
    if (context.isMenuMessage()) context.client.logger.debug(`msg by ${context.target.author.username}`);
  }
  if (context.isChat()) {
    context.client.logger.info(`Slash command: /${context.fullCommandName}`);
  }
  return next();
});
```

## Recipes / worked examples

### Open a modal from a message command, then act on the submission

A menu context can open a modal and await its submission via `ctx.modal(body, { waitFor })`.

```ts
import {
  ContextMenuCommand,
  Declare,
  Modal,
  Label,
  TextInput,
  TextInputStyle,
  type MenuCommandContext,
  type MessageCommandInteraction,
  ApplicationCommandType,
} from 'seyfert';

@Declare({ name: 'Report Message', type: ApplicationCommandType.Message })
export default class ReportMessageCommand extends ContextMenuCommand {
  async run(ctx: MenuCommandContext<MessageCommandInteraction>) {
    const message = ctx.target;

    const modal = new Modal()
      .setCustomId(`report:${message.id}`)
      .setTitle('Report message')
      .setComponents([
        new Label()
          .setLabel('Why are you reporting this?')
          .setComponent(
            new TextInput()
              .setCustomId('reason')
              .setStyle(TextInputStyle.Paragraph)
              .setRequired(true),
          ),
      ]);

    // waitFor makes modal() resolve with the submission (or null on timeout)
    const submitted = await ctx.modal(modal, { waitFor: 60_000 });
    if (!submitted) return; // timed out / dismissed

    const reason = submitted.getInputValue('reason', true);
    await submitted.write({
      content: `Reported message from ${message.author.username}: ${reason}`,
      flags: 64,
    });
  }
}
```

### Forward a flagged message to a mod-log channel

Defer first (work may take a moment), then post elsewhere and confirm.

```ts
import {
  ContextMenuCommand,
  Declare,
  Embed,
  type MenuCommandContext,
  type MessageCommandInteraction,
  ApplicationCommandType,
} from 'seyfert';

const MOD_LOG_CHANNEL = '123456789012345678';

@Declare({
  name: 'Flag to Mods',
  type: ApplicationCommandType.Message,
  contexts: ['Guild'],
  defaultMemberPermissions: ['ManageMessages'],
})
export default class FlagToModsCommand extends ContextMenuCommand {
  async run(ctx: MenuCommandContext<MessageCommandInteraction>) {
    await ctx.deferReply(true); // ephemeral defer
    const message = ctx.target;

    const embed = new Embed()
      .setAuthor({ name: message.author.username })
      .setDescription(message.content || '*no text content*')
      .setFooter({ text: `Flagged by ${ctx.author.username}` })
      .setTimestamp();

    await ctx.client.messages.write(MOD_LOG_CHANNEL, { embeds: [embed] });
    await ctx.editOrReply({ content: 'Sent to the mod team.' });
  }
}
```

### Guild-only user command with member/role checks

`inGuild()` narrows to `GuildMenuCommandContext`, making `guildId`/`member` non-optional and `guild()` return a non-undefined guild.

```ts
import {
  ContextMenuCommand,
  Declare,
  type MenuCommandContext,
  type UserCommandInteraction,
  ApplicationCommandType,
} from 'seyfert';

@Declare({
  name: 'Show Top Role',
  type: ApplicationCommandType.User,
  contexts: ['Guild'],
})
export default class ShowTopRoleCommand extends ContextMenuCommand {
  async run(ctx: MenuCommandContext<UserCommandInteraction>) {
    if (!ctx.inGuild()) {
      return ctx.write({ content: 'Guild only.', flags: 64 });
    }
    // ctx is GuildMenuCommandContext now: guildId + member guaranteed
    const target = ctx.target; // UserStructure
    const member = await ctx.client.members.fetch(ctx.guildId, target.id);
    const highest = await member.roles.highest(); // GuildRole | undefined
    await ctx.write({
      content: highest
        ? `${target.username}'s top role: ${highest.name}`
        : `${target.username} has no roles.`,
      flags: 64,
    });
  }
}
```

### Per-command hooks: middleware denial and bot-permission failure

```ts
import {
  ContextMenuCommand,
  Declare,
  Middlewares,
  type MenuCommandContext,
  type UserCommandInteraction,
  type PermissionStrings,
  ApplicationCommandType,
} from 'seyfert';

@Declare({ name: 'Warn User', type: ApplicationCommandType.User, contexts: ['Guild'] })
@Middlewares(['cooldown']) // names from your registered middlewares
export default class WarnUserCommand extends ContextMenuCommand {
  async run(ctx: MenuCommandContext<UserCommandInteraction>) {
    await ctx.write({ content: `Warned ${ctx.target.username}`, flags: 64 });
  }

  // stop('reason') in a middleware lands here; metadata.middleware names the offender
  async onMiddlewaresError(ctx: MenuCommandContext<UserCommandInteraction>, error: string) {
    await ctx.write({ content: `Blocked: ${error}`, flags: 64 });
  }

  async onBotPermissionsFail(ctx: MenuCommandContext<UserCommandInteraction>, permissions: PermissionStrings) {
    await ctx.write({ content: `I'm missing: ${permissions.join(', ')}`, flags: 64 });
  }
}
```

## Doc vs Source Corrections

- Docs say "you don't need to manually set the `type` in `@Declare`" -> SOURCE requires it. `ContextMenuCommand.type` is non-null with no default, and `@Declare` only assigns `type` when present (`src/commands/decorators.ts:223`). The `@Declare` overload for menus is `Omit<DecoratorDeclareOptions,'description'> & { type: ApplicationCommandType.User | ApplicationCommandType.Message }` (`src/commands/decorators.ts:24-28`), so `type` is mandatory and `description` is rejected at compile time. All examples must pass `type` and must NOT pass `description`.
- Doc example interpolates `ctx.target` directly (`Context menu used on: ${ctx.target}`) — `ctx.target` is a `User`/`Message` object, so that stringifies to `[object Object]`; use `ctx.target.id`/`.username`/`.content` instead (`src/commands/applications/menucontext.ts:55`).
- `ctx.target` is a getter that rebuilds the structure from `interaction.data.resolved` via `Transformers` on every access — assign it to a local const if you read it more than once (`src/commands/applications/menucontext.ts:55-66`).
- `write`/`editOrReply` return `When<WR, WebhookMessageStructure, void>` (default `void`; pass `true` for the message), not always a `WebhookMessageStructure` (`src/commands/applications/menucontext.ts:84-118`).
- Docs' type-guard section shows only `isMenu()`/`isChat()`. SOURCE additionally provides `isMenuUser()` / `isMenuMessage()` (narrow the target type) and `inGuild()` -> `GuildMenuCommandContext` (`src/commands/applications/menucontext.ts:181-204`).
- Context menu names are NOT lowercased: `LowercaseDeclareName` passes the `name` through unchanged when `type` is `User`/`Message` (`src/commands/decorators.ts:139-149`), so spaces/uppercase are preserved (slash command names are not).

## Source Anchors

- `src/commands/applications/menu.ts` — `ContextMenuCommand` class, required `type`, hooks, `toJSON()`.
- `src/commands/applications/menucontext.ts` — `MenuCommandContext`, `target`, `InteractionTarget`, `GuildMenuCommandContext`, `isMenu*`, `inGuild`, response helpers, `modal`.
- `src/commands/decorators.ts:24-32` — `CommandDeclareOptions` union (menu type omits `description`, requires `type`); `:139-149` `LowercaseDeclareName` (menu names not lowercased); `:195-231` `Declare`.
- `src/commands/basecontext.ts` — base type-guard stubs (`isMenu`, `isMenuUser`, `isMenuMessage`, `isChat`).
- `src/commands/applications/options.ts` — `AnyContext` union.
- `src/structures/Interaction.ts:839-859` — `UserCommandInteraction`, `MessageCommandInteraction`.
- `src/common/types/write.ts:78-80` — `ModalCreateOptions` (`waitFor`).
- `src/builders/Modal.ts` — `Modal`, `TextInput` builders.
- `src/commands/index.ts`, `src/structures/index.ts`, `src/index.ts` — root `'seyfert'` exports.

## Agent Guidance

- Always set `type: ApplicationCommandType.User` or `.Message` in `@Declare`; never pass `description` (the type rejects it). Do not trust the docs' claim that `type` can be omitted.
- Parameterize the context with the matching interaction (`MenuCommandContext<UserCommandInteraction>` vs `<MessageCommandInteraction>`) so `ctx.target` narrows to `User` vs `Message`.
- Cache `ctx.target` in a local const — it is a getter that rebuilds the structure on each access.
- Context menu names support spaces/uppercase; only chat (slash) command names are lowercased.
- `ctx.write`/`ctx.editOrReply` always go through the interaction; default return is `void`, pass `true` for the `WebhookMessageStructure`. Add `flags: 64` (Ephemeral) for private mod/utility replies.
- Open modals with `ctx.modal(body, { waitFor })` to await a `ModalSubmitInteraction | null`; read fields off the returned submission (`getInputValue`, etc.).
- In generic middleware over `AnyContext`, narrow with `ctx.isMenu()` then optionally `ctx.isMenuUser()`/`ctx.isMenuMessage()`; use `ctx.inGuild()` for guild-only narrowing.
- Use `stop('reason')` (not the removed `pass()`) in middlewares; the denial lands in `onMiddlewaresError(ctx, error, metadata)` with `metadata.middleware` naming the offender.
- `ctx.fullCommandName` for a menu is just the declared name (no subcommand path).
