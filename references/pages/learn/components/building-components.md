# Building components

Original source URL: https://seyfert-web-git-seyfert-v5-tiramisulabs.vercel.app/docs/learn/components/building-components
Coverage reference: components.md (+ builders.md)
Verification status: Source-verified against ./src (branch more-qol) — all builder/context/command APIs below confirmed.

## Page Summary

Components are constructed with builder classes that you place inside an `ActionRow`. A message holds up to 5 action rows; a row holds up to 5 buttons OR a single select menu (mixing is invalid at the Discord layer). Every interactive component carries a `customId` used later to route the interaction to a `ComponentCommand` (or to an inline collector). All builders here are exported from the `'seyfert'` root (re-export of `src/builders`).

## Key APIs (verified)

ActionRow — `src/builders/ActionRow.ts`
- `class ActionRow<T extends ActionBuilderComponents = ActionBuilderComponents>` — generic over the contained builder, e.g. `ActionRow<Button>`. `ActionBuilderComponents = Button | BuilderSelectMenus`.
- `new ActionRow(data?: Partial<APIActionRowComponent<...>>)` — `components` in `data` are auto-revived via `fromComponent`.
- `.setComponents(...component: RestOrArray<FixedComponents<T>>)` — array OR rest args (replaces existing).
- `.addComponents(...component: RestOrArray<FixedComponents<T>>)` — array OR rest args (appends).
- `.toJSON()` — emits `{ type: ComponentType.ActionRow, components }` (maps each child's `toJSON()`).
- Public field `components: FixedComponents<T>[]`.

Button — `src/builders/Button.ts`
- `new Button(data?: Partial<APIButtonComponent>)` — defaults `type` to `ComponentType.Button`.
- `.setCustomId(id)`, `.setLabel(label)`, `.setStyle(style: ButtonStyle)`, `.setURL(url)`, `.setSKUId(skuId)`, `.setEmoji(emoji: EmojiResolvable)`, `.setDisabled(disabled = true)`.
- `.setEmoji` THROWS `SeyfertError('INVALID_EMOJI')` if the emoji cannot be resolved.
- `ButtonStyle` and `ComponentType` are exported from `'seyfert'` (re-exported from `discord-api-types` via `src/types`).

SelectMenu builders — `src/builders/SelectMenu.ts`
- Base `SelectMenu`: `.setCustomId(id)`, `.setPlaceholder(text)`, `.setValuesLength({ max, min })`, `.setDisabled(d = true)`, `.setRequired(r = true)` (required is for modals only).
- `StringSelectMenu<Value extends string = string>`: `.addOption(...RestOrArray<StringSelectOption | APISelectMenuOption<Value>>)`, `.setOptions(...)`, `.setRequired()`, custom `.toJSON()`. Options accept `StringSelectOption` instances OR raw objects. A typed `StringSelectMenu<'a' | 'b'>` rejects raw values outside the union.
- `StringSelectOption`: `.setLabel(v)`, `.setValue(v)`, `.setDescription(v)`, `.setDefault(v = true)`, `.setEmoji(emoji)`. `.toJSON()` THROWS (`MISSING_STRING_SELECT_OPTION_LABEL` / `_VALUE`) if `label` or `value` is missing.
- `UserSelectMenu`: `.setDefaultUsers(...RestOrArray<string>)`, `.addDefaultUsers(...)`.
- `RoleSelectMenu`: `.setDefaultRoles(...)`, `.addDefaultRoles(...)`.
- `ChannelSelectMenu`: `.setDefaultChannels(...)`, `.addDefaultChannels(...)`, `.setChannelTypes(types: ChannelType[])`.
- `MentionableSelectMenu`: `.setDefaultMentionables(...RestOrArray<{ id: string; type: keyof typeof SelectMenuDefaultValueType }>)`, `.addDefaultMentionables(...)`. `type` is `'User' | 'Role'` (key of `SelectMenuDefaultValueType`).
- Exported helper type `BuilderSelectMenus` = union of all five select builders.
- v5: the select-menu `.disabled` typo property was removed — use `.setDisabled()`.

ComponentCommand (how customId is consumed) — `src/components/componentcommand.ts`
- `abstract class ComponentCommand` with `abstract componentType: keyof ContextComponentCommandInteractionMap` (`'Button' | 'StringSelect' | 'UserSelect' | 'RoleSelect' | 'MentionableSelect' | 'ChannelSelect'`).
- `customId?: string | RegExp` — internal `_filter` matches exact string equality OR `customId.match(RegExp)`; optional `filter(ctx): boolean | Promise<boolean>` runs after the customId check.
- `abstract run(ctx: ComponentContext<this['componentType']>)`.
- `middlewares: readonly (keyof ResolvedRegisteredMiddlewares)[] = []` (readonly in v5 — no `.push`).
- Lifecycle hooks: `onBeforeMiddlewares`, `onAfterRun(ctx, error)`, `onRunError(ctx, error)`, `onMiddlewaresError(ctx, error, metadata)`, `onInternalError(client, component, error?)` — note v5 signature now passes the offending `component` BEFORE `error`.

ComponentContext — `src/components/componentcontext.ts`
- Generics: `ComponentContext<Type, M, StringSelectValues extends string[] = string[]>`.
- `ctx.customId` getter; `ctx.message` getter (the message the component lives on); `ctx.author`, `ctx.member`, `ctx.guildId`, `ctx.channelId`.
- Reply helpers: `write(body, fetchReply?)`, `editOrReply(body, fetchReply?)`, `deferReply(ephemeral?, fetchReply?)`, `followup(body)`, `update(body)`, `deferUpdate()`, `modal(body, options?)`, `fetchResponse()`, `deleteResponse()`. **NO `editResponse`** (removed in v5 for ComponentContext — `ModalContext` still has it).
- `write`/`editOrReply`/`deferReply` return `void`/`undefined` UNLESS `fetchReply`/the response flag is `true` (then a `WebhookMessageStructure` is returned).
- Fetch helpers (mode-aware): `channel(mode)`, `me(mode)`, `guild(mode, query?)` where `mode = 'cache' | 'rest' | 'flow'` (`'cache'` may be sync). `ctx.t` (i18n proxy, honors `preferGuildLocale`).
- Type guards: `isButton()`, `isStringSelectMenu()`, `isUserSelectMenu()`, `isRoleSelectMenu()`, `isChannelSelectMenu()`, `isMentionableSelectMenu()`, `isComponent()`, `inGuild()`. v5: `isButton()` etc. read `interaction.componentType` (was `interaction.data.componentType`).
- Selected values live on the interaction: `ctx.interaction.values` (typed by the `StringSelectValues` generic for string selects). Resolved entities: `ctx.interaction.channels` / `.roles` / `.users` / `.members` on the matching select interaction.

## Code Examples (verified)

Basic ActionRow (builder accepts array or rest args):
```ts
import { ActionRow } from 'seyfert';

const row = new ActionRow().setComponents([]).addComponents();
```

Button row (+ link & premium buttons, disabled state):
```ts
import { ActionRow, Button, ButtonStyle } from 'seyfert';

const action = new Button()
  .setCustomId('first-button')
  .setStyle(ButtonStyle.Primary)
  .setLabel('First Button')
  .setEmoji('👍');

const link = new Button() // link buttons have no customId
  .setStyle(ButtonStyle.Link)
  .setLabel('Docs')
  .setURL('https://seyfert.dev');

const disabled = new Button()
  .setCustomId('locked')
  .setStyle(ButtonStyle.Secondary)
  .setLabel('Soon')
  .setDisabled();

const row = new ActionRow<Button>().setComponents([action, link, disabled]);
```

Select menus (all five types — rest OR array both work):
```ts
import {
  ActionRow,
  StringSelectMenu,
  StringSelectOption,
  UserSelectMenu,
  RoleSelectMenu,
  ChannelSelectMenu,
  MentionableSelectMenu,
} from 'seyfert';
import { ChannelType } from 'seyfert';

const stringMenu = new StringSelectMenu()
  .setCustomId('string-menu')
  .setPlaceholder('Select a string option')
  .addOption(
    new StringSelectOption().setLabel('Option 1').setValue('1'),
    new StringSelectOption().setLabel('Option 2').setValue('2'),
  );
const stringRow = new ActionRow<StringSelectMenu>().setComponents([stringMenu]);

const userMenu = new UserSelectMenu()
  .setCustomId('user-menu')
  .setPlaceholder('Select a user')
  .setValuesLength({ min: 1, max: 3 })
  .setDefaultUsers('123456789', '987654321');

const roleMenu = new RoleSelectMenu().setCustomId('role-menu').setDefaultRoles('123');

const channelMenu = new ChannelSelectMenu()
  .setCustomId('channel-menu')
  .setChannelTypes([ChannelType.GuildText, ChannelType.GuildVoice]);

const mentionableMenu = new MentionableSelectMenu()
  .setCustomId('mentionable-menu')
  .setDefaultMentionables(
    { type: 'User', id: '123456789' },
    { type: 'Role', id: '987654321' },
  );
```

Typed StringSelectMenu (compiler enforces the value union):
```ts
import { StringSelectMenu, StringSelectOption } from 'seyfert';

const menu = new StringSelectMenu<'red' | 'green' | 'blue'>()
  .setCustomId('color')
  .setOptions(
    new StringSelectOption().setLabel('Red').setValue('red'),
    new StringSelectOption().setLabel('Green').setValue('green'),
    // .setValue('purple') would be a type error
  );
```

Sending components from a command:
```ts
import { Declare, Command, type CommandContext, ActionRow, Button, ButtonStyle } from 'seyfert';

@Declare({ name: 'ping', description: 'Show the ping with discord' })
export default class PingCommand extends Command {
  async run(ctx: CommandContext) {
    const ping = ctx.client.gateway.latency; // average latency across shards
    const row = new ActionRow<Button>().setComponents([
      new Button().setCustomId('pingbtn').setLabel(`Ping: ${ping}`).setStyle(ButtonStyle.Primary),
    ]);
    await ctx.write({ content: `The ping is \`${ping}\``, components: [row] });
  }
}
```

## Recipes (verified)

### 1. Handle a button with a ComponentCommand

A registered handler routes by `componentType` + `customId`. Use `ctx.update(...)` to edit the source message in place (no new reply), or `ctx.deferUpdate()` to ACK silently.
```ts
import { ComponentCommand, type ComponentContext } from 'seyfert';

export default class PingButton extends ComponentCommand {
  componentType = 'Button' as const;
  customId = 'pingbtn'; // exact-match this customId

  async run(ctx: ComponentContext<'Button'>) {
    // edit the message the button lives on
    await ctx.update({ content: `Clicked by ${ctx.author.username}`, components: [] });
  }
}
```

### 2. Read selected values from a StringSelect

The generic types `ctx.interaction.values`. Narrow with the guard if a handler covers several component types.
```ts
import { ComponentCommand, type ComponentContext } from 'seyfert';

export default class ColorSelect extends ComponentCommand {
  componentType = 'StringSelect' as const;
  customId = 'color';

  async run(ctx: ComponentContext<'StringSelect'>) {
    const [picked] = ctx.interaction.values; // string[]
    await ctx.update({ content: `You picked ${picked}` });
  }
}
```
For entity selects, resolved structures are on the interaction: `ctx.interaction.users`, `.members`, `.roles`, `.channels`.

### 3. Open a modal from a button and await the submission

`ctx.modal(builder, { waitFor })` resolves to the `ModalSubmitInteraction | null`. In v5 modal text inputs are wrapped in a `Label`.
```ts
import {
  ComponentCommand,
  type ComponentContext,
  Modal,
  Label,
  TextInput,
  TextInputStyle,
} from 'seyfert';

export default class FeedbackButton extends ComponentCommand {
  componentType = 'Button' as const;
  customId = 'open-feedback';

  async run(ctx: ComponentContext<'Button'>) {
    const modal = new Modal()
      .setCustomId('feedback-modal')
      .setTitle('Feedback')
      .addComponents(
        new Label()
          .setLabel('Your message')
          .setComponent(
            new TextInput().setCustomId('msg').setStyle(TextInputStyle.Paragraph),
          ),
      );

    const submitted = await ctx.modal(modal, { waitFor: 60_000 });
    if (!submitted) return; // timed out
    const value = submitted.getInputValue('msg', true);
    await submitted.write({ content: `Thanks: ${value}` });
  }
}
```

### 4. Inline collector (no separate handler file)

`message.createComponentCollector(options?)` gives you `.run(customId, cb)` + `.waitFor(...)`, scoped to that message. Filter by user so other people can't drive your buttons.
```ts
import { ActionRow, Button, ButtonStyle, type CommandContext } from 'seyfert';

async function confirm(ctx: CommandContext) {
  const row = new ActionRow<Button>().setComponents([
    new Button().setCustomId('yes').setStyle(ButtonStyle.Success).setLabel('Yes'),
    new Button().setCustomId('no').setStyle(ButtonStyle.Danger).setLabel('No'),
  ]);

  const message = await ctx.write({ content: 'Proceed?', components: [row] }, true);

  const collector = message.createComponentCollector({
    idle: 30_000, // ms with no interaction before onStop('idle')
    filter: i => i.user.id === ctx.author.id,
    onStop: reason => ctx.editOrReply({ content: `Closed (${reason})`, components: [] }),
  });

  collector.run('yes', i => i.update({ content: 'Confirmed', components: [] }));
  collector.run('no', i => i.update({ content: 'Cancelled', components: [] }));
}
```

### 5. Dynamic / paginated ids with a RegExp customId + filter

Encode state in the customId, match with a `RegExp`, and refine with `filter()`.
```ts
import { ComponentCommand, type ComponentContext } from 'seyfert';

// matches "page:next:0", "page:prev:3", ...
export default class Paginator extends ComponentCommand {
  componentType = 'Button' as const;
  customId = /^page:(next|prev):\d+:\d+$/; // opener id encoded: "page:next:0:<openerId>"

  filter(ctx: ComponentContext<'Button'>) {
    return ctx.customId.split(':')[3] === ctx.author.id; // only the opener (id encoded in customId)
  }

  async run(ctx: ComponentContext<'Button'>) {
    const [, dir, raw] = ctx.customId.split(':');
    const page = Number(raw) + (dir === 'next' ? 1 : -1);
    await ctx.update({ content: `Page ${page}` });
  }
}
```

## Common patterns / gotchas

- One select per row, max 5 buttons per row, max 5 rows. Buttons and a select cannot share a row.
- Link buttons (`ButtonStyle.Link`) take `setURL` and NO `customId`; premium buttons (`ButtonStyle.Premium`) take `setSKUId`. Every other button needs a `customId`.
- Builders THAT THROW on bad input: `Button.setEmoji` / `StringSelectOption.setEmoji` on an unresolvable emoji (`INVALID_EMOJI`); `StringSelectOption.toJSON()` (and the menu's, since it maps options) when `label` or `value` is unset.
- Use `ctx.update(...)` / `ctx.deferUpdate()` to act on the existing message; use `ctx.write(...)` to send a brand-new (optionally ephemeral) reply. `ctx.write`/`editOrReply` return `void` unless you pass the fetch-reply flag `true`.
- There is NO `ctx.editResponse` on a ComponentContext in v5 — use `editOrReply(body, true)` when you need the `Message` back. (`ModalContext.editResponse` still exists.)
- A ComponentCommand persists across restarts (re-routes by customId); an inline collector lives only as long as the process/timeout. Prefer collectors for short ephemeral flows, ComponentCommands for permanent buttons.
- `customId` length cap is 100 chars — keep encoded RegExp state short.
- `command.middlewares` is a readonly tuple; declare it inline, don't `.push`.

## Doc vs Source Corrections

- Docs `SeyfertRegistry` augmentation snippet (`interface SeyfertRegistry { client: ParseClient<Client<true>> }`) matches v5 — the old `UsingClient`/`RegisteredMiddlewares` augmentation is gone.
- Docs show `setDefaultUsers/Roles/Channels` + `setDefaultMentionables`; source also provides additive `addDefault*` variants (not in docs).
- Docs show only `setCustomId/setStyle/setLabel` on Button; source also has `setURL`, `setSKUId`, `setEmoji`, `setDisabled`.
- `setComponents`/`addComponents` accept `RestOrArray` — both `setComponents(a, b)` and `setComponents([a, b])` work; docs only show the array form.
- Docs omit `customId` may be a `RegExp` and the `filter()` hook on `ComponentCommand` — both exist in source.
- v5 deltas vs older docs: ComponentContext lost `editResponse`; `onInternalError(client, component, error?)` gained the component param; select-menu `.disabled` typo property removed (use `setDisabled`); modal text inputs are wrapped in `Label` (Components v2).

## Source Anchors

- `src/builders/ActionRow.ts`
- `src/builders/Button.ts`
- `src/builders/SelectMenu.ts`
- `src/builders/Modal.ts` (Modal, TextInput) + `src/builders/Label.ts`
- `src/builders/types.ts` (ListenerOptions, ComponentCollectorStopReason, FixedComponents)
- `src/builders/index.ts` (root builder re-exports + `fromComponent`)
- `src/components/componentcommand.ts`
- `src/components/componentcontext.ts`
- `src/components/handler.ts` (createComponentCollector)
- `src/structures/Interaction.ts` (componentType getter, `.values`, resolved entities)
- `src/structures/Message.ts` (`createComponentCollector`)

## Agent Guidance

- Import every builder, `ButtonStyle`, `ComponentType`, `ChannelType`, `TextInputStyle` from `'seyfert'` — no deep `seyfert/lib/...` import needed.
- Type the row for autocomplete: `new ActionRow<Button>()` / `<StringSelectMenu>()`. A row mixing buttons and selects is invalid at the Discord layer.
- Always set a `customId` on interactive components (except link/premium buttons) — it is what `ComponentCommand._filter` matches (exact string or `RegExp.match`). Use a `RegExp` customId + `filter()` for dynamic/paginated ids.
- For "press a button → answer here" flows, decide: persistent `ComponentCommand` (survives restarts) vs inline `message.createComponentCollector` (ephemeral, time-boxed).
- Read selections from `ctx.interaction.values` (string selects, typed by the `StringSelectValues` generic) or `ctx.interaction.users/roles/channels/members` (entity selects).
- `setRequired`/`setValuesLength` on selects matter mostly inside modals; for message rows you typically just set customId, placeholder, and options/defaults.
