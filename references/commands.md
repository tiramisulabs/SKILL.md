# Commands (Seyfert v5)

Source-verified against `./src` (branch more-qol). Commands are class-based + decorator-configured. Everything imports from the **root `'seyfert'`** unless a deep import is explicitly called out.

## Classes

- `class Command extends BaseCommand` — top-level slash command. Default `type = ApplicationCommandType.ChatInput`. Optional `run(ctx: CommandContext)`. Holds `options`, `groups`, `groupsAliases`, `aliases`. (`src/commands/applications/chat.ts:361`)
- `abstract class SubCommand extends BaseCommand` — `type = Subcommand`, optional `group?: string`, **abstract `run`** (must implement). May carry basic `@Options` but cannot nest subcommands. (`chat.ts:408`)
- `abstract class ContextMenuCommand` — user/message menu. `type!` is REQUIRED non-null (`User | Message`, no default). `abstract run?(ctx: MenuCommandContext<any>)`. No options/description. (`applications/menu.ts`)
- `abstract class EntryPointCommand` — Activities entry point. `type = PrimaryEntryPoint`, requires `handler: EntryPointCommandHandlerType`, `abstract run?(ctx: EntryPointContext)`. (`applications/entryPoint.ts:14`)
- `abstract class ComponentCommand` — persistent component handler. `abstract componentType` + `abstract run`, optional `customId?: string | RegExp` / `filter?`. (`components/componentcommand.ts`)
- `abstract class ModalCommand` — persistent modal handler. `abstract run(ctx: ModalContext)`, optional `customId`/`filter`. (`components/modalcommand.ts`)

Always `export default` file-loaded commands (loader uses `x.default ?? x`).

## Decorators (all from `'seyfert'`, `src/commands/decorators.ts`)

- `@Declare(opts)` — metadata. Chat-input shape requires `name` + `description`.
- `@Options(record | (new()=>SubCommand)[])` — slash options OR subcommand classes. Option keys forced lowercase at type level.
- `@Middlewares(readonly string[])` — per-command middleware names (registered keys). Tuple is `readonly` (no `.push`).
- `@AutoLoad()` — load default-exported sibling SubCommand files from the parent's folder.
- `@Group(name)` / `@Group(defs, name)`, `@Groups(record)`, `@GroupsT(record)` — subcommand grouping; `defineGroups(obj)` identity helper for the type-checked `@Group(defs, name)` overload.
- `@Locales({ name?, description? })` — per-locale `[LocaleString, value][]` tuples → `*_localizations`.
- `@LocalesT(nameKey?, descKey?)` — translation keys (`FlatObjectKeys<DefaultLocale>`) stored on `__t`.

### @Declare fields (chat-input, `decorators.ts:33`)
`name` (req, lowercased via `LowercaseDeclareName`), `description` (req), `botPermissions?`, `defaultMemberPermissions?`, `guildId?: string[]`, `nsfw?`, `integrationTypes?`, `contexts?`, `ignore?: IgnoreCommand`, `aliases?: string[]`, `props?`.

`CommandDeclareOptions` is a union — three mutually exclusive shapes:
- **chat-input** (above).
- **context-menu**: `type: ApplicationCommandType.User | Message` — `description` is OMITTED (`Omit<...,'description'>`); setting both is a type error.
- **entry-point**: `type: PrimaryEntryPoint` + `handler`; omits `ignore`/`aliases`/`guildId`.

`@Declare` behavior (`decorators.ts:195-231`): permission string arrays → resolved to `bigint` via `PermissionsBitField.resolve` (you cannot read the array back; **a bad string THROWS** in v5); `contexts`/`integrationTypes` string keys → numeric enums; `integrationTypes` defaults to `[GuildInstall]` (server-only); chat `name` lowercased at the **type level** (`LowercaseDeclareName`; not transformed at runtime), **menu names NOT**.

`enum IgnoreCommand { Slash = 0, Message = 1 }` (`applications/shared.ts:133`): `Slash` hides from slash registration (text-only); `Message` hides from the prefix resolver (slash-only).

```ts
import { Declare, Command, IgnoreCommand, type CommandContext } from 'seyfert';

@Declare({
  name: 'ping',
  description: 'pong',
  defaultMemberPermissions: ['Administrator'],
  botPermissions: ['SendMessages'],
  contexts: ['Guild'],
  integrationTypes: ['GuildInstall'],
  ignore: IgnoreCommand.Message, // slash-only
})
export default class PingCommand extends Command {
  async run(ctx: CommandContext) { await ctx.write({ content: 'pong' }); }
}
```

## Options (`src/commands/applications/options.ts`)

Factories (root exports), each returns input + `as const` typed `type`:
`createStringOption`, `createIntegerOption`, `createNumberOption`, `createBooleanOption`, `createUserOption`, `createRoleOption`, `createMentionableOption`, `createChannelOption`, `createAttachmentOption`.

- `description` is the ONLY mandatory field on every option.
- Common: `required?`, `value?(data, ok, fail)`, localization (`name_localizations`, `description_localizations`, `locales?`).
- String: `min_length?`, `max_length?`, `choices?`, `autocomplete?`. Integer/Number: `min_value?`, `max_value?`, `choices?`, `autocomplete?`. Channel: `channel_types?`. Boolean/User/Role/Mentionable/Attachment: basic.
- **`choices`/`channel_types` arrays are `readonly`** → add `as const` to ALL choice arrays (string AND integer/number) to preserve the literal value union in `ctx.options`.
- Only String/Integer/Number support `choices`/`autocomplete`; choices vs autocomplete are mutually exclusive (Discord).
- `required: true` → option non-optional in `CommandContext<typeof options>`; otherwise `T | undefined`.
- `value(data, ok, fail)`: `data = { context, value }` (raw resolved). `ok(transformed)` stores; `fail('msg')` rejects → `onOptionsError`. `ok` is `OKFunction<I>`, `fail` is `StopFunction`.
- v5: autocomplete `respond()` values are typed to the option type — numeric options reject string values.

```ts
import { Options, createStringOption, createIntegerOption, createChannelOption, Command, ChannelType, type CommandContext } from 'seyfert';

const options = {
  // KEYS MUST BE LOWERCASE (compile-time). { Query: ... } is a type error.
  query: createStringOption({ description: 'q', required: true, min_length: 2 }),
  mode: createStringOption({ description: 'm', choices: [
    { name: 'Fast', value: 'fast' }, { name: 'Deep', value: 'deep' },
  ] as const }),
  amount: createIntegerOption({
    description: 'n', min_value: 1,
    value: ({ value }, ok, fail) => (value > 0 ? ok(value * 2) : fail('must be positive')),
  }),
  target: createChannelOption({ description: 'where', channel_types: [ChannelType.GuildText] as const }),
} as const;

@Options(options)
class Search extends Command {
  async run(ctx: CommandContext<typeof options>) {
    ctx.options.mode;   // 'fast' | 'deep' | undefined  (literal via `as const`)
    ctx.options.query;  // string (required)
    ctx.options.amount; // number (stored doubled by value())
  }
}
```

### Autocomplete
`autocomplete?: (interaction: AutocompleteInteraction<boolean, V>) => any`, `onAutocompleteError?`.
- `interaction.getInput()` → focused value as a **string** (`''` if none) regardless of option type — convert with `Number()` for numeric options.
- `interaction.respond(choices)` sends results. **Use `respond`, NOT `reply`** (reply throws for autocomplete).

```ts
createIntegerOption({
  description: 'pick',
  autocomplete: i => {
    const focus = i.getInput();              // always a string
    return i.respond(['10','20','30']
      .filter(x => x.includes(focus))
      .map(x => ({ name: `#${x}`, value: Number(x) }))); // value MUST be number
  },
  onAutocompleteError: i => i.respond([]),
})
```

## CommandContext (`src/commands/applications/chatcontext.ts`)

`CommandContext<T extends OptionsRecord = {}, M extends keyof ResolvedRegisteredMiddlewares = never>`. `M` = attached middleware names → types `ctx.metadata`.

Methods/props: `write(body, withResponse?)`, `editOrReply(body, withResponse?)`, `editResponse(body)` *(still exists on chat ctx)*, `deferReply(ephemeral?, withResponse?)`, `deleteResponse()`, `followup(body)`, `fetchResponse()`, `modal(body, options?)`, `channel('rest'|'flow'|'cache')`, `me(...)`, `guild(...)`, `fetchMember(...)`; `options`, `metadata`, `globalMetadata`, `guildId`, `channelId`, `author`, `member`, `t`, `fullCommandName`, `deferred`, `interaction`, `message`.

- **`write`/`editOrReply` return `void` UNLESS the response flag is `true`** (then a message/webhook is returned). Don't `await` them expecting a `Message` by default.
- **`ctx.modal(body, { waitFor })`** works only on interaction contexts; on a prefix/message context it throws `SeyfertError('CANNOT_USE_MODAL')`. With `waitFor` it resolves to `ModalSubmitInteraction | null`; without it returns `undefined` (fire-and-forget).
- Fetch modes: `channel`/`fetchMember`/`guild`/`me` take `'cache' | 'rest' | 'flow'` (default `'flow'`). `'cache'` is sync (`ReturnCache`); `'rest'`/`'flow'` return a Promise.

Guards: `isChat()`, `inGuild(): this is GuildCommandContext` (narrows `guildId`/`member` non-null and `channel()`/`guild()`/`me()`/`fetchMember()` to non-undefined guild returns), `isMenu()`, `isMenuUser()`, `isMenuMessage()`, `isComponent()`, `isModal()`, `isButton()`, select guards, `isEntryPoint()`.

**Channel narrowing** — `await ctx.channel()` returns a channel structure; narrow with `src/structures/channels.ts` guards: `isGuild()`, `isDM()`, `isThread()`, `isVoice()`, `isTextGuild()`, `isNamed()` (→ `AllNamedChannels`), `isTextable()`.

## Middlewares

- `createMiddleware<T = any, C extends AnyContext = AnyContext>(cb)` — identity helper. `cb` receives `{ context, next, stop }`. **There is NO `pass`** (removed in v5).
- `next()` continues; `next(data)` when `T` is not void → `data` lands on `ctx.metadata.<name>` (command) / `ctx.globalMetadata.<name>` (global).
- `stop('msg')` denies → `onMiddlewaresError`. `stop()` / `stop(null)` silently skips (no run, no error) — this is the old `pass()`.
- Register: `client.setServices({ middlewares })` + augment `SeyfertRegistry.middlewares`. Attach with `@Middlewares(['name'])`. Global: `new Client({ globalMiddlewares: [...] })`, augment `GlobalMetadata` with `ParseGlobalMiddlewares<typeof global>`.
- Order: globals (in option order) → command middlewares (in decorator order) → `run`. Call exactly ONE of next/stop (forgetting both hangs forever; repeat calls ignored). Unregistered names warn+skip; a thrown/rejected middleware (sync throw OR async rejection) **rejects the runner → `onInternalError`** — an exception is an internal error, NOT a denial, and is not logged by the runner.
- **Globals run only for application commands** (slash/menu/entry-point) — component & modal handlers run their OWN `middlewares` only.
- Share a typed list: `middlewares('a','b')` infers a literal tuple; `InferMiddlewares<typeof list>` → the name union for `CommandContext`'s 2nd generic.

```ts
import { createMiddleware, type CommandContext } from 'seyfert';

interface CooldownData { remaining: number }
const lastUsed = new Map<string, number>();

export const cooldown = createMiddleware<CooldownData>(({ context, next, stop }) => {
  const prev = lastUsed.get(context.author.id) ?? 0;
  const remaining = 5_000 - (Date.now() - prev);
  if (remaining > 0) return stop(`Wait ${Math.ceil(remaining / 1000)}s.`); // → onMiddlewaresError
  lastUsed.set(context.author.id, Date.now());
  next({ remaining: 0 }); // payload REQUIRED because T is not void
});

export const middlewares = { cooldown };
// run(ctx: CommandContext<never, 'cooldown'>) { ctx.metadata.cooldown.remaining }
```

## Recipe: complete command with options + middleware + error hooks

```ts
// commands/warn.ts
import {
  Command, Declare, Options, Middlewares,
  createUserOption, createStringOption,
  type CommandContext,
} from 'seyfert';
import type { PluginMiddlewareDenialMetadata } from 'seyfert';

const options = {
  target: createUserOption({ description: 'Member to warn', required: true }),
  reason: createStringOption({ description: 'Reason', required: true, max_length: 400 }),
} as const;

@Declare({ name: 'warn', description: 'Warn a member', contexts: ['Guild'], defaultMemberPermissions: ['ModerateMembers'] })
@Options(options)
@Middlewares(['cooldown'])
export default class WarnCommand extends Command {
  async run(ctx: CommandContext<typeof options, 'cooldown'>) {
    if (!ctx.inGuild()) return;
    const { target, reason } = ctx.options;
    await ctx.write({ content: `Warned ${target.username}: ${reason}`, flags: 64 });
  }

  async onMiddlewaresError(ctx: CommandContext, error: string, meta: PluginMiddlewareDenialMetadata) {
    await ctx.write({ content: `[${meta.middleware}] ${error}`, flags: 64 });
  }
  async onRunError(ctx: CommandContext, error: unknown) {
    ctx.client.logger.error(error);
    await ctx.editOrReply({ content: 'Something went wrong.' }).catch(() => {});
  }
}
```

## Subcommands & Groups

Parent holds subcommands and has NO `run`. Register via `@Options([SubA, SubB])` (named or default exports) OR `@AutoLoad()` (default-exported siblings only; never combine the two). Declare groups on the parent with `@Groups`/`@GroupsT` (each needs `defaultDescription`); assign per-subcommand with `@Group('name')`. A `@Group` without a matching `@Groups` key throws at build — use `defineGroups` + `@Group(defs, name)` to catch the typo at compile time. Max nesting: `group > subcommand > options`.

```ts
// commands/mod/groups.ts
import { defineGroups } from 'seyfert';
export const modGroups = defineGroups({
  warns: { defaultDescription: 'Manage warnings', aliases: ['w'] },
});

// commands/mod/index.ts
import { Command, Declare, Options, Groups } from 'seyfert';
import { AddWarn } from './add-warn';
import { modGroups } from './groups';

@Declare({ name: 'mod', description: 'Moderation tools' })
@Groups(modGroups)
@Options([AddWarn])
export default class ModCommand extends Command {} // no run()

// commands/mod/add-warn.ts
import { SubCommand, Declare, Group, Options, createUserOption, type CommandContext } from 'seyfert';
import { modGroups } from './groups';

const options = { user: createUserOption({ description: 'Who', required: true }) } as const;

@Declare({ name: 'add', description: 'Add a warning' })
@Group(modGroups, 'warns') // 'warns' type-checked against keyof modGroups → /mod warns add
@Options(options)
export class AddWarn extends SubCommand {
  async run(ctx: CommandContext<typeof options>) {
    if (!ctx.inGuild()) return;
    const member = await ctx.fetchMember(); // GuildMemberStructure (non-undefined)
    await ctx.write({ content: `Warned ${ctx.options.user.username} in ${member.guildId}.` });
  }
}
```

## Context Menus (type is explicit, no description)

```ts
import { ContextMenuCommand, Declare, ApplicationCommandType, type MenuCommandContext, type MessageCommandInteraction } from 'seyfert';

@Declare({ name: 'Report Message', type: ApplicationCommandType.Message }) // type REQUIRED; name keeps case/spaces
export default class ReportMessage extends ContextMenuCommand {
  async run(ctx: MenuCommandContext<MessageCommandInteraction>) {
    const message = ctx.target; // MessageStructure — getter rebuilds each access, cache it
    await ctx.write({ content: `Reported: ${message.content.slice(0, 80)}`, flags: 64 });
  }
}
```
Parameterize with `MenuCommandContext<UserCommandInteraction>` or `<MessageCommandInteraction>` so `ctx.target` narrows (`UserStructure` vs `MessageStructure`). `ctx.target` is a getter — assign to a local const if read twice. Menu `write`/`editOrReply` return `void` by default (pass `true` for the `WebhookMessageStructure`); they always go through the interaction. Guards: `isMenuUser()`, `isMenuMessage()`, `inGuild()` → `GuildMenuCommandContext`.

## Component & Modal handlers

`componentType` keys: `'Button' | 'StringSelect' | 'UserSelect' | 'RoleSelect' | 'MentionableSelect' | 'ChannelSelect'`. Handlers are persistent (survive restarts); match by static `customId` (string/RegExp) and/or `filter`. They support `middlewares` + `onBeforeMiddlewares`/`onRunError`/`onMiddlewaresError`/`onAfterRun`/`onInternalError` (NO options/permission hooks).

```ts
// components/confirm.button.ts
import { ComponentCommand, type ComponentContext } from 'seyfert';

export default class ConfirmButton extends ComponentCommand {
  componentType = 'Button' as const;
  customId = 'confirm'; // or a RegExp; or override filter()

  filter(ctx: ComponentContext<'Button'>) {
    // only the original invoker may press
    return ctx.interaction.message.interactionMetadata?.user.id === ctx.author.id;
  }
  async run(ctx: ComponentContext<'Button'>) {
    await ctx.update({ content: 'Confirmed!', components: [] });
  }
}
```

```ts
// modals/feedback.modal.ts  — full accessor set (getInputValue, getChannels, getRoles, getUsers, ...)
import { ModalCommand, type ModalContext } from 'seyfert';

export default class FeedbackModal extends ModalCommand {
  customId = 'feedback';
  async run(ctx: ModalContext) {
    const text = ctx.getInputValue('opinion', true); // required → throws if missing
    await ctx.write({ content: `Thanks: ${text}`, flags: 64 });
  }
}
```

`ComponentContext` (v5): `write(body, fetchReply?)`, `editOrReply(body, fetchReply?)`, `deferReply(...)`, `update(...)`, `deferUpdate()`, `ctx.message` getter. **NO `editResponse`** (removed in v5 — use `editOrReply(body, true)` for the message). `isButton()` reads `interaction.componentType`; `StringSelectValues` generic keeps `interaction.values` typed. `ModalContext.editResponse` STILL exists.

## Recipe: button + collector flow (ephemeral, scoped to a sent message)

When you need a one-off interaction tied to a specific message (not a global handler), send a message and attach a collector. The callback gets `(interaction, stop, refresh)`.

```ts
import { Command, Declare, ActionRow, Button, ButtonStyle, type CommandContext } from 'seyfert';

@Declare({ name: 'confirm', description: 'Confirm an action' })
export default class Confirm extends Command {
  async run(ctx: CommandContext) {
    const row = new ActionRow<Button>().setComponents([
      new Button().setCustomId('yes').setStyle(ButtonStyle.Success).setLabel('Yes'),
      new Button().setCustomId('no').setStyle(ButtonStyle.Danger).setLabel('No'),
    ]);

    // fetchReply=true → get the Message back so we can attach a collector
    const message = await ctx.write({ content: 'Are you sure?', components: [row] }, true);

    const collector = message.createComponentCollector({
      idle: 30_000,
      filter: i => i.user.id === ctx.author.id,
      onStop: reason => ctx.client.logger.debug(`collector stopped: ${reason}`),
    });

    collector.run('yes', async (i, stop) => {
      await i.update({ content: 'Confirmed!', components: [] });
      stop('done');
    });
    collector.run('no', async (i, stop) => {
      await i.update({ content: 'Cancelled.', components: [] });
      stop('done');
    });
  }
}
```
`createComponentCollector(options?)` returns `{ run(customId, cb), stop(reason?), waitFor(customId, timeout?), resetTimeouts() }`. `customId` accepts a string, string[], or RegExp. Collectors attach to the message, so they work for prefix commands too (modals do not).

## Recipe: modal opened from a slash command, awaited inline

```ts
import { Command, Declare, Modal, Label, TextInput, TextInputStyle, type CommandContext } from 'seyfert';

@Declare({ name: 'suggest', description: 'Submit a suggestion' })
export default class Suggest extends Command {
  async run(ctx: CommandContext) {
    if (!ctx.interaction) return ctx.write({ content: 'Use the slash version for the form.' });

    const modal = new Modal()
      .setCustomId('suggest-modal')
      .setTitle('New suggestion')
      .addComponents(
        new Label() // v5: modal fields are Label-wrapped (not ActionRow); label lives on Label, not TextInput
          .setLabel('Your idea')
          .setComponent(
            new TextInput().setCustomId('idea').setStyle(TextInputStyle.Paragraph).setRequired(true),
          ),
      );

    const submitted = await ctx.modal(modal, { waitFor: 60_000 }); // ModalSubmitInteraction | null
    if (!submitted) return; // timed out / dismissed
    const idea = submitted.getInputValue('idea', true);
    await submitted.write({ content: `Got it: ${idea}`, flags: 64 });
  }
}
```

## Prefix (Message) Commands

Opt-in via `commands.prefix`, reusing the SAME command classes.

```ts
const client = new Client({ commands: {
  prefix: async msg => (msg.guildId ? [await db.getPrefix(msg.guildId), `<@${client.botId}>`] : ['!']),
  reply: ctx => true,                                     // true → message.reply, else channel send
  deferReplyResponse: ctx => ({ content: 'Working...' }), // default placeholder is 'Thinking...'
}});

declare module 'seyfert' { interface InternalOptions { withPrefix: true } } // REQUIRED for ctx.message typing
```
- Prefixes are sorted longest-first and matched with `content.startsWith`. Signature is `Awaitable<string[]>` (async supported).
- Without the `InternalOptions` augmentation, `ctx.message` is typed `undefined`. Guard message-only logic with `if (ctx.message)`, modal/interaction-only with `if (ctx.interaction)`.
- `ctx.modal()` throws `SeyfertError('CANNOT_USE_MODAL')` from prefix. `ctx.t` falls back to `defaultLang` (no interaction locale) — even with `preferGuildLocale`.
- Prefix dispatch still enforces `contexts`/`guildId`, member/bot permissions, options, and middlewares before `run` (guild owner bypasses member-perm check).
- Default parser: `-flag value` chunks, greedy to next ` -` (regex `/-(.*?)(?=\s-|$)/gs`). Booleans accept `yes|y|true|treu` (sic). Quotes are not special.
- **`HandleCommand` is NOT root-exported** — deep import to subclass: `import { HandleCommand } from 'seyfert/lib/commands/handle'`; override `argsParser`/`resolveCommandFromContent`; register with `client.setServices({ handleCommand: MyHandleCommand })` (pass the **class**, not an instance — v5 change).

## Errors, Lifecycle Hooks & Defaults

Optional `BaseCommand` overrides (`chat.ts:349-358`):
- `onBeforeMiddlewares(ctx)`, `onBeforeOptions(ctx)` *(chat only)*, `run(ctx)`, `onAfterRun(ctx, error: unknown | undefined)` *(fires on success AND failure)*.
- `onRunError(ctx, error)`, `onOptionsError(ctx, metadata: OnOptionsReturnObject)` *(chat only)*.
- `onMiddlewaresError(ctx, error: string, metadata: PluginMiddlewareDenialMetadata)` — **3 args**; `metadata = { middleware, scope }`.
- `onPermissionsFail(ctx, perms)` *(chat only)*, `onBotPermissionsFail(ctx, perms)`.
- `onInternalError(client, command, error?)` — **`command` is 2nd arg, error is 3rd** (copying `(client, error)` is a real bug now).

Which hook fires: thrown in `run` → `onRunError` then `onAfterRun(error)`; option `fail()` → `onOptionsError`; middleware `stop('msg')` → `onMiddlewaresError` (bare `stop()` = silent, no hook); a middleware that **throws/rejects** → `onInternalError` (exceptions are internal errors, not denials); missing member perms → `onPermissionsFail`; missing bot perms → `onBotPermissionsFail`; one of YOUR hooks throwing → `onInternalError`.

Client-wide defaults (applied with `??=`, per-command wins): `new Client({ commands: { defaults: {...} } })`. `commands.defaults` accepts all chat hooks + `props`. `components.defaults` / `modals.defaults` accept ONLY `onBeforeMiddlewares`, `onRunError`, `onInternalError`, `onMiddlewaresError`, `onAfterRun`.

`OnOptionsReturnObject` (`shared.ts:119`): `Record<name, { failed:false; value } | { failed:true; value: string }>` — failed `value` is the `fail()` string. `PluginMiddlewareDenialMetadata` is re-exported at root `seyfert` (`src/client/plugins.ts`); runtime hooks work without importing it.

`SeyfertError.is(err, 'CANNOT_USE_MODAL')` narrows a caught error by code inside a handler.

## Localization (i18n)

Register your default lang on `SeyfertRegistry.langs` so `DefaultLocale` (and `ctx.t`) is typed:

```ts
import type english from './langs/en';
declare module 'seyfert' {
  interface SeyfertRegistry {
    langs: typeof english; // DefaultLocale is derived from this
  }
}
// langs is a setServices option, NOT a Client constructor option:
client.setServices({ langs: { default: 'en', preferGuildLocale: true } });
```
- `@Locales({ name: [[lang,val]], description: [[lang,val]] })` for static localizations, or `@LocalesT('a.b.name','a.b.desc')` with lang-file keys; `@GroupsT` for groups; option-level `locales: { name, description }`.
- Resolve at runtime: `ctx.t.<path>.get(locale?)`. `ctx.t` returns a named `SeyfertLocale` proxy. `preferGuildLocale: true` makes `ctx.t` resolve the guild locale before the user's (interaction contexts only).

## Extending context

`extendContext(cb)` (identity helper) + `new Client({ context })`; `cb` receives the wide interaction/message union (narrow before use), returns a plain object `Object.assign`-ed onto every context **synchronously**. Type it by augmenting `interface ExtendContext` in `declare module 'seyfert'`. Both runtime option AND augmentation are required. Plugins' `createPlugin({ ctx })` feed the same `ctx.*` surface and are auto-typed (no manual augmentation).

```ts
import { Client, extendContext } from 'seyfert';
const context = extendContext(interaction => ({
  actorId: 'user' in interaction ? interaction.user.id : interaction.author.id,
}));
const client = new Client({ context });
declare module 'seyfert' { interface ExtendContext { actorId: string } }
```

## Module augmentation (v5)

Augment the single `SeyfertRegistry` — `UsingClient`/`RegisteredMiddlewares`/`DefaultLocale` are derived now (`ParseMiddlewares` REMOVED).

```ts
import type { Client, ParseClient } from 'seyfert';
import type { middlewares } from './middlewares';
declare module 'seyfert' {
  interface SeyfertRegistry {
    client: ParseClient<Client<true>>;
    middlewares: typeof middlewares;   // bare typeof — no ParseMiddlewares
    // langs: typeof english;
    // plugins: typeof client.options.plugins;
  }
}
```

## Import policy
- Root `'seyfert'`: all command/component/modal classes, `EntryPointCommand`, all decorators, all `create*Option`, `createMiddleware`, `middlewares`, `InferMiddlewares`, `defineGroups`, `extendContext`, `createPlugin`/`definePlugins`, `CommandContext`, `MenuCommandContext`, `ComponentContext`, `ModalContext`, builders (`Modal`, `ActionRow`, `Button`, `TextInput`, `Label`, `Embed`), `ApplicationCommandType`, `ChannelType`, `ButtonStyle`, `TextInputStyle`, `IgnoreCommand`, `SeyfertError`, `ParseClient`, `ParseGlobalMiddlewares`.
- **Deep import required**: `HandleCommand` (`seyfert/lib/commands/handle`). `PluginMiddlewareDenialMetadata` IS root-exported from `seyfert`.

## Review Checklist
- Command `export default`ed (or programmatically registered)? Subcommands for `@AutoLoad`/`@Options([])` default-exported (AutoLoad) or imported (Options)?
- Chat-input has `name` (lowercase) + `description`; context menus pass `type` and OMIT `description`; entry points pass `handler`.
- Option record keys lowercase; `choices`/`channel_types` use `as const`; `run` typed `CommandContext<typeof options>`.
- Autocomplete uses `respond` (not `reply`); numeric `respond` values are numbers; `getInput()` is always a string.
- Middlewares use `next`/`stop` (NEVER `pass`); exactly one is called; `CommandContext<Opts, 'mw'>` for typed `ctx.metadata`; global data via `GlobalMetadata`; globals don't run for component/modal handlers.
- `onMiddlewaresError` has 3 args; `onInternalError` is `(client, command, error?)`.
- `write`/`editOrReply` return `void` unless the response flag is `true` (pass `true` to get a Message for collectors).
- Parent-with-subcommands has no `run`; `@Group` names match a `@Groups` key (prefer `defineGroups` + `@Group(defs, name)`).
- Component handlers set `componentType`; `ComponentContext` has no `editResponse` (use `editOrReply(body, true)`); modal accessors via `getInputValue`/`getChannels`/etc.
- Prefix: `InternalOptions.withPrefix` augmented; no `ctx.modal()` on prefix paths; custom parser via deep-imported `HandleCommand` (registered as a class).
- Module augmentation targets `SeyfertRegistry` (no `UsingClient`/`RegisteredMiddlewares`/`ParseMiddlewares`).
- `HandleCommand` uses a deep import; `PluginMiddlewareDenialMetadata` is root-exported from `seyfert`.
- `uploadCommands` re-run after schema changes.
