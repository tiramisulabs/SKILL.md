# Components, Modals, Collectors, Polls, Components v2

Source of truth: `src/builders/**`, `src/components/**`, `src/client/collectors.ts`, `src/structures/Message.ts`.
Import policy: EVERYTHING here is a root export from `'seyfert'` (builders re-exported via `src/builders/index.ts` -> `src/index.ts`). No deep `seyfert/lib/...` imports.

v5 deltas that bite here: `ComponentContext.editResponse` REMOVED (use `editOrReply(body, true)`); modal inputs wrap in `Label` (NO `ActionRow` in modals); middleware `pass()` GONE (use `stop()`); `onInternalError(client, component, error?)` gained the component param; select-menu `.disabled` property typo removed (use `setDisabled`); builders validate on `toJSON()` (throw before Discord); `CollectorRunPameters`->`CollectorRunParameters`.

## Classic builders (ActionRow + children)

A message holds up to 5 `ActionRow`s; a row holds up to 5 buttons OR one select menu (never mixed).

- `ActionRow<T extends ActionBuilderComponents>` (`Button | BuilderSelectMenus`): `.setComponents(...RestOrArray<T>)` (replace), `.addComponents(...RestOrArray<T>)` (append). Both take an array OR rest args. Type the row: `new ActionRow<Button>()`.
- `Button`: `.setCustomId(id)`, `.setLabel(l)`, `.setStyle(ButtonStyle)`, `.setURL(url)`, `.setSKUId(id)`, `.setEmoji(EmojiResolvable)`, `.setDisabled(d=true)`. `setEmoji` THROWS `SeyfertError('INVALID_EMOJI')` if unresolvable. Link/SKU buttons have no `customId` (`ButtonLink = Omit<Button,'setCustomId'>`); `ButtonStyle.Link` needs `setURL`, `ButtonStyle.Premium` needs `setSKUId`.
- Select menus (`BuilderSelectMenus`): `StringSelectMenu`, `UserSelectMenu`, `RoleSelectMenu`, `ChannelSelectMenu`, `MentionableSelectMenu`. Base: `.setCustomId`, `.setPlaceholder`, `.setValuesLength({max,min})`, `.setDisabled`, `.setRequired` (required matters in modals).
  - `StringSelectMenu<Value extends string = string>`: `.addOption(...)/.setOptions(...)` take `StringSelectOption` instances OR raw `APISelectMenuOption` (rest-or-array). A typed `StringSelectMenu<'a'|'b'>` rejects raw values outside the union.
  - `StringSelectOption`: `.setLabel/.setValue/.setDescription/.setDefault/.setEmoji` (or `new StringSelectOption({ label, value })`). `toJSON()` THROWS if label/value unset.
  - User/Role/Channel/Mentionable: `.setDefaultX(...)/.addDefaultX(...)`; `ChannelSelectMenu.setChannelTypes(ChannelType[])`; `MentionableSelectMenu.setDefaultMentionables(...{ id, type })` where `type` is a `keyof typeof SelectMenuDefaultValueType` (e.g. `'User'`/`'Role'`).

`ButtonStyle`, `ComponentType`, `ChannelType`, `TextInputStyle`, `MessageFlags` are root exports (re-exported discord-api-types).

```ts
import { ActionRow, Button, ButtonStyle, ChannelSelectMenu, ChannelType } from 'seyfert';
const buttons = new ActionRow<Button>().addComponents(
  new Button().setCustomId('confirm:yes').setLabel('Confirm').setStyle(ButtonStyle.Success),
  new Button().setStyle(ButtonStyle.Link).setLabel('Docs').setURL('https://seyfert.dev'),
);
const channels = new ActionRow<ChannelSelectMenu>().setComponents(
  new ChannelSelectMenu().setCustomId('pick-chan').setChannelTypes([ChannelType.GuildText]),
);
await ctx.write({ content: 'Confirm?', components: [buttons, channels] });
```

## ComponentCommand (durable component handlers)

`src/components/componentcommand.ts`. File-based: `export default` a class in the configured `components` dir (config `locations.components`). Survives restarts (unlike collectors). Programmatic: `client.components.set([...])`.

- `abstract componentType: keyof ContextComponentCommandInteractionMap` — `'Button' | 'StringSelect' | 'UserSelect' | 'RoleSelect' | 'MentionableSelect' | 'ChannelSelect'` (SINGULAR `'Button'`). Use `componentType = 'Button' as const`.
- `customId?: string | RegExp` — declarative match. `_filter` checks customId FIRST (string `===` or `RegExp.match`), then optional `filter?(ctx): boolean | Promise<boolean>`. With neither, matches EVERY interaction of that type (footgun).
- `abstract run(ctx)`. `middlewares: readonly (...)[] = []` (readonly — no `.push`). Hooks: `onBeforeMiddlewares`, `onAfterRun(ctx, error)`, `onRunError(ctx, error)`, `onMiddlewaresError(ctx, error, metadata)`, `onInternalError(client, component, error?)`.
- Engine (`handler.ts executeComponent`) runs EVERY loaded handler whose type+filter match (no early `break`) — keep customIds specific to avoid double-handling. The file MUST `export default` the class or it is skipped with a warning.

```ts
import { ComponentCommand, type ComponentContext } from 'seyfert';
export default class Confirm extends ComponentCommand {
  componentType = 'Button' as const;
  customId = /^confirm:/; // string or RegExp; encode params here

  async run(ctx: ComponentContext<'Button'>) {
    const [, choice] = ctx.customId.split(':'); // confirm:yes -> 'yes'
    await ctx.update({ content: `Confirmed: ${choice}`, components: [] });
  }
}
```

## ComponentContext

`src/components/componentcontext.ts`. Generic: `ComponentContext<Type, M, StringSelectValues extends string[] = string[]>`.
- Getters: `customId`, `message` (the message the component lives on), `author`, `member`, `guildId`, `channelId`, `t` (honors `preferGuildLocale`).
- Reply: `write(body, fetchReply?)`, `editOrReply(body, fetchReply?)`, `deferReply(ephemeral?, fetchReply?)`, `followup`, `fetchResponse`, `deleteResponse`. Return `void`/`undefined` UNLESS `fetchReply === true` (then a `WebhookMessageStructure`).
- Component-specific: `update(body)` (edit source message), `deferUpdate()` (silent ACK), `modal(body, options?)`.
- **NO `editResponse`** (removed in v5 — use `editOrReply(body, true)` for the Message). `ModalContext.editResponse` still exists.
- Fetchers: `channel(mode?)`, `guild(mode?)`, `me(mode?)` — `mode = 'cache'|'rest'|'flow'` (`'cache'` may be sync, others Promise).
- Guards: `isButton()`, `isStringSelectMenu()`, `isUserSelectMenu()`, `isRoleSelectMenu()`, `isChannelSelectMenu()`, `isMentionableSelectMenu()`, `isComponent()`, `inGuild()` (-> `GuildComponentContext`). v5: `isButton()` reads `interaction.componentType`.

After a guard, read selected values via `ctx.interaction.values` (string selects, typed by `StringSelectValues`) or `ctx.interaction.users/roles/channels/members` (entity selects). Use `update`/`deferUpdate` to mutate the originating message; `write`/`deferReply` for a new (optionally ephemeral) response. Exactly one ack per interaction or Discord shows "interaction failed".

```ts
export default class ColorSelect extends ComponentCommand {
  componentType = 'StringSelect' as const;
  customId = 'color';
  async run(ctx: ComponentContext<'StringSelect'>) {
    if (!ctx.isStringSelectMenu()) return;
    const [picked] = ctx.interaction.values; // string[]
    await ctx.update({ content: `You picked ${picked}`, components: [] });
  }
}
```

### Recipe — gate a ComponentCommand with middlewares (v5 `stop`, no `pass`)

`middlewares` is a readonly tuple of registered middleware names. A middleware that calls `stop('reason')` routes to `onMiddlewaresError`; `stop()` with no arg skips silently (v5 — `pass()` removed). `metadata.middleware` names the denying middleware.

```ts
import { ComponentCommand, type ComponentContext, MessageFlags } from 'seyfert';
export default class AdminPanel extends ComponentCommand {
  componentType = 'Button' as const;
  customId = 'admin-action';
  middlewares = ['isAdmin'] as const; // names from your SeyfertRegistry middlewares

  async run(ctx: ComponentContext<'Button'>) {
    await ctx.update({ content: 'Action performed.' });
  }
  onMiddlewaresError(ctx: ComponentContext<'Button'>, error: string) {
    return ctx.write({ content: error || 'Not allowed.', flags: MessageFlags.Ephemeral });
  }
}
```

The middleware itself (defined elsewhere, registered via `SeyfertRegistry.middlewares`):
```ts
import { createMiddleware } from 'seyfert';
export const isAdmin = createMiddleware<void>(({ context, next, stop }) => {
  if (!context.member?.permissions.has('Administrator')) return stop('Admins only.');
  return next();
});
```

## Modals

Modal top level accepts ONLY `Label | TextDisplay` (`ModalBuilderComponents`). Every INTERACTIVE component (TextInput, any select menu, FileUpload, Checkbox, CheckboxGroup, RadioGroup) MUST be wrapped in its own `Label`. v5 modals do NOT use `ActionRow` — a raw input or `ActionRow<TextInput>` in a modal is a type error.

- `Modal`: `.setTitle(t)` + `.setCustomId(id)` (both required — `toJSON()` THROWS `MISSING_MODAL_TITLE`/`MISSING_MODAL_CUSTOM_ID`). `.setComponents(arr: ModalBuilderComponents[])` takes a SINGLE array (NOT rest); `.addComponents(...)` is rest-or-array. `.run(cb)` registers an inline submit callback keyed by user id (default expiry 10 min; Discord emits no cancel event).
- `Label`: `.setLabel`, `.setDescription`, `.setComponent(one)` — holds exactly ONE component (`LabelBuilderComponents = TextInput | BuilderSelectMenus | FileUpload | Checkbox | CheckboxGroup | RadioGroup`). `toJSON()` THROWS `MISSING_COMPONENT`.
- `TextInput`: `.setCustomId`, `.setStyle(TextInputStyle.Short|Paragraph)`, `.setPlaceholder`, `.setLength({max,min})`, `.setValue`, `.setRequired`.
- `FileUpload`: `.setCustomId`, `.setId(n)`, `.setMinValues`, `.setMaxValues`, `.setRequired`.
- `Checkbox`: `.setCustomId`, `.setId`, `.setDefault(bool)`. `CheckboxGroup`/`RadioGroup`: `.setCustomId`, `.setId`, `.setOptions/.addOptions` (`CheckboxGroupOption`/`RadioGroupOption` with `.setLabel/.setValue/.setDescription/.setDefault`), `.setRequired`; CheckboxGroup also `.setSelectionLimit({max,min})`. Both THROW `INVALID_OPTIONS_LENGTH` unless 2–10 options. `RadioGroupOption.toJSON()` also throws on missing label/value; `CheckboxGroupOption` does not.
- `TextDisplay`: `.setContent(string)`, `.setId(n)` (top-level static text; method is `setContent`, NOT `content`).

```ts
import { Modal, Label, TextInput, TextInputStyle } from 'seyfert';
const modal = new Modal().setTitle('Feedback').setCustomId('feedback')
  .setComponents([
    new Label().setLabel('Message').setComponent(
      new TextInput().setCustomId('msg').setStyle(TextInputStyle.Paragraph).setRequired()),
  ]);
await ctx.modal(modal);
```

### ModalCommand + ModalContext
`src/components/modalcommand.ts` — `customId?: string | RegExp` and/or `filter?`, `abstract run(ctx: ModalContext)`, same hooks (`onInternalError(client, modal, error?)`). Value readers on `ModalContext` keyed by the WRAPPED component's customId (NOT the Label's):
- `getInputValue(id, required?)` -> `string | string[] | undefined` (array for value-array components — narrow before string use).
- `getRadioValues(id,req?)`->`string`, `getCheckboxValues`->`string[]`, `getCheckbox`->`boolean`, `getFiles`->`Attachment[]`, `getUsers/getRoles/getChannels/getMentionables`->resolved arrays.
- All throw `SeyfertError('INTERNAL_ERROR')` when `required:true` and absent. ModalContext also has `write/editOrReply/deferReply/update/deferUpdate/editResponse(STILL EXISTS)/followup/t/author/member/guildId/inGuild()`.

```ts
// multi-field profile modal handled by one ModalCommand
import { ModalCommand, type ModalContext, MessageFlags } from 'seyfert';
export default class ProfileEdit extends ModalCommand {
  customId = 'profile-edit';
  async run(ctx: ModalContext) {
    const name = ctx.getInputValue('name', true) as string;   // text -> string
    const bio = ctx.getInputValue('bio') ?? '';               // optional -> string | undefined
    const [color] = ctx.getInputValue('color', true);          // select via getInputValue -> string[]
    const notifs = ctx.getCheckboxValues('notifs', true);      // string[]
    await ctx.write({ content: `Saved ${name} (${color}); notifs: ${notifs.join(',') || 'none'}`, flags: MessageFlags.Ephemeral });
  }
}
```

### Modal waits / inline / RegExp routing
- `ctx.modal(modal)` / `interaction.modal(modal)` -> `Promise<undefined>`. With options -> `ctx.modal(modal, { waitFor: ms })` -> `Promise<ModalSubmitInteraction | null>` (await submit inline; `null` on timeout). The ONLY option is `waitFor` (no `idle`/`timeout`). Respond to the RETURNED submission, not the original ctx.
- Modals are interaction-only: on a prefix (message) context `ctx.modal` throws `SeyfertError('CANNOT_USE_MODAL')` (catch via `SeyfertError.is(err, 'CANNOT_USE_MODAL')`). You cannot show a modal after already replying/deferring.
- Encode per-entity state in a RegExp customId (e.g. `report-<id>`); `_filter` runs `.match()` before `run`, then re-extract inside `run`.

```ts
// open with the target id baked in, route by RegExp
import { ModalCommand, type ModalContext, Modal, Label, TextInput, TextInputStyle } from 'seyfert';
export function buildReportModal(messageId: string) {
  return new Modal().setTitle('Report').setCustomId(`report-${messageId}`)
    .setComponents([new Label().setLabel('Reason').setComponent(
      new TextInput().setCustomId('reason').setStyle(TextInputStyle.Paragraph).setRequired(true))]);
}
export default class ReportHandler extends ModalCommand {
  customId = /^report-(\d+)$/;
  async run(ctx: ModalContext) {
    const [, messageId] = ctx.customId.match(/^report-(\d+)$/)!;
    const reason = ctx.getInputValue('reason', true) as string;
    await ctx.write({ content: `Report on ${messageId}: ${reason}`, flags: 64 });
  }
}
```

## Collectors (in-process, message-scoped)

`message.createComponentCollector(options?)` (`src/components/handler.ts`). Lives ONLY in the creating process — dies on restart, not cross-shard/worker. For durable flows use `ComponentCommand` with encoded customId state. One collector per message id (re-creating overwrites).

Options (`ListenerOptions`): `timeout?` (absolute ms), `idle?` (inactivity ms, reset per matched+filtered interaction), `filter?(i)`, `onPass?(i)` (filter rejected — ack other users here), `onStop?(reason, refresh)`, `onError?(i, err, stop, refresh)` (callback threw; logged if absent).
Result: `run<T>(customId, cb)`, `stop(reason?)`, `waitFor<T>(customId, timeout?)` -> `Promise<T|null>`, `resetTimeouts()`.
- `customId` accepts `string | string[] | RegExp` (string = `===`, array = `includes`, RegExp = `.test()`). `run` callback is `(interaction, stop, refresh) => any`.
- `onStop` reasons (`ComponentCollectorStopReason`, exported): `'idle' | 'timeout' | 'messageDelete' | 'channelDelete' | 'guildDelete' | (string&{})`. Two refresh semantics: `onStop`'s `refresh()` RE-CREATES the collector (resurrect after idle/timeout); the `run` callback's `refresh()` only resets timers.

```ts
import { ActionRow, Button, ButtonStyle, MessageFlags, type CommandContext } from 'seyfert';
async function confirm(ctx: CommandContext) {
  const row = new ActionRow<Button>().setComponents(
    new Button().setCustomId('yes').setStyle(ButtonStyle.Success).setLabel('Yes'),
    new Button().setCustomId('no').setStyle(ButtonStyle.Danger).setLabel('No'),
  );
  const message = await ctx.write({ content: 'Proceed?', components: [row] }, true); // true => fetchReply

  const collector = message.createComponentCollector({
    idle: 30_000, timeout: 120_000,
    filter: i => i.user.id === ctx.author.id,
    onPass: i => i.write({ content: 'Not your prompt.', flags: MessageFlags.Ephemeral }),
    onStop: (reason, refresh) => { if (reason === 'idle') return refresh(); },
  });
  collector.run(['yes', 'no'], async (i, stop) => {
    const ok = i.customId === 'yes';
    await i.update({ content: ok ? 'Confirmed' : 'Cancelled', components: [] });
    stop('done');
  });
}
```

### Recipe — pagination via `waitFor` loop (closure state, no handler files)

```ts
import { ActionRow, Button, ButtonStyle, type CommandContext, type ButtonInteraction } from 'seyfert';
const pages = ['Page 1', 'Page 2', 'Page 3'];
async function paginate(ctx: CommandContext) {
  let i = 0;
  const row = () => new ActionRow<Button>().setComponents(
    new Button().setCustomId('prev').setLabel('◀').setStyle(ButtonStyle.Secondary).setDisabled(i === 0),
    new Button().setCustomId('next').setLabel('▶').setStyle(ButtonStyle.Secondary).setDisabled(i === pages.length - 1),
  );
  const message = await ctx.write({ content: pages[i], components: [row()] }, true);
  const collector = message.createComponentCollector({ filter: x => x.user.id === ctx.author.id });
  while (true) {
    const btn = await collector.waitFor<ButtonInteraction>(['prev', 'next'], 60_000);
    if (!btn) { await ctx.editOrReply({ components: [] }); break; } // timed out -> strip buttons
    i += btn.customId === 'next' ? 1 : -1;
    await btn.update({ content: pages[i], components: [row()] });
  }
}
```

NOTE: `client.collectors` (`src/client/collectors.ts`) is a SEPARATE event-level API: `client.collectors.create({ event: 'messageCreate', filter, run: (arg, stop) => {}, idle, timeout, onStop, onRunError })`. v5: option type `CollectorRunParameters` (typo fixed); camelCase client event names (`'messageCreate'`, NOT `'MESSAGE_CREATE'`); runs EVERY matching collector (no first-match `break`). Don't confuse with message collectors.

## Polls

`PollBuilder` (`src/builders/Poll.ts`); pass the INSTANCE straight into the message body `poll` field (no `.toJSON()` — the writer serializes it).
- `.setQuestion(PollMedia)` — takes an OBJECT `{ text?, emoji? }`, NOT a bare string. `.setAnswers(...RestOrArray<PollMedia>)` (replace), `.addAnswers(...)` (append). `.setDuration(hours)` — HOURS not ms (Discord max 768h/32d). `.allowMultiselect(v=true)`.
- `toJSON()` THROWS `MISSING_POLL_QUESTION` / `MISSING_POLL_ANSWERS`; a bad emoji throws `INVALID_EMOJI` AT ADD TIME. Discord caps 10 answers (builder does NOT enforce).
- Answer ids are 1-based, insertion order (`ValidAnswerId = 1..10`). Polls cannot be edited after creation — only ended.

```ts
import { Command, Declare, PollBuilder, type CommandContext } from 'seyfert';
@Declare({ name: 'deploy', description: 'Approve a deploy.' })
export default class DeployPoll extends Command {
  async run(ctx: CommandContext) {
    const channel = await ctx.channel('rest');
    if (!channel.isTextGuild()) return;
    const poll = new PollBuilder()
      .setQuestion({ text: 'Ship to production?' })
      .setAnswers({ text: 'Yes', emoji: '✅' }, { text: 'No', emoji: '❌' })
      .allowMultiselect(false).setDuration(2); // hours
    await channel.messages.write({ content: 'Vote:', poll });
    await ctx.write({ content: 'Poll posted.' });
  }
}
```

Receiving votes needs intents `GuildMessagePolls` / `DirectMessagePolls`. Events `messagePollVoteAdd` / `messagePollVoteRemove` — handler `data` is camelCase: `userId, messageId, channelId, answerId, guildId?` (no trailing `shardId`). Detect end via `messageUpdate` -> `newMessage.poll?.results?.isFinalized`. End: `Message#endPoll()` (hold the message) or `Poll#end()` (hold the structure). Voters: `getAnswerVoters(answerId, checkAnswer?)`. `results.answerCounts` is camelCase `{ id, count, meVoted }`.

```ts
import { createEvent } from 'seyfert';
export default createEvent({
  data: { name: 'messageUpdate' },
  run: async ([message]) => { // payload is [newMessage, oldMessage]
    const poll = message.poll;
    if (!poll?.results?.isFinalized) return;
    const top = [...poll.results.answerCounts].sort((a, b) => b.count - a.count)[0];
    if (!top) return;
    const label = poll.answers.find(a => a.answerId === top.id)?.pollMedia.text ?? `answer ${top.id}`;
    await message.edit({ content: `Closed. Winner: ${label} (${top.count})` });
  },
});
```

## Components v2

Composable layout builders; MUST set `flags: MessageFlags.IsComponentsV2` (`1 << 15`) on EVERY send AND edit — Seyfert never auto-applies it; drop it on an edit and Discord rejects. A v2 message cannot also carry top-level `content`/`embeds`/`poll`/`stickers` — all text goes in `TextDisplay` (markdown works). All root exports.
- `Container`: `.addComponents/.setComponents(...RestOrArray)` children = `ActionRow | TextDisplay | Section | MediaGallery | Separator | File` (a bare `Button` must be wrapped in `ActionRow` or used as a Section accessory). `.setColor(ColorResolvable)` (-> accent_color; `resolveColor` THROWS on malformed `#hex`), `.setSpoiler`, `.setId`.
- `Section<Ac extends Button | Thumbnail>`: `.addComponents/.setComponents(...TextDisplay)` (TextDisplay children only, 1–3) + `.setAccessory(Button | Thumbnail)`. `toJSON()` THROWS `MISSING_ACCESSORY` if none.
- `TextDisplay`: `.setContent(string)`, `.setId`.
- `Thumbnail`: `.setMedia(url)` (NOT `setURL`), `.setDescription`, `.setSpoiler`, `.setId`.
- `MediaGallery`: `.addItems/.setItems(...MediaGalleryItem)`, `.setId`. `MediaGalleryItem` (plain class, no setId): `.setMedia(url)`, `.setDescription`, `.setSpoiler`; `toJSON()` THROWS `MISSING_MEDIA`.
- `Separator`: `.setSpacing(Spacing)`, `.setDivider(d=false)`, `.setId`. `Spacing` enum has ONLY `Small=1`, `Large=2` (no `None`/`Medium`).
- `File`: `.setMedia(url)` (use `attachment://name`), `.setSpoiler`, `.setId`. Reference uploads with a matching `AttachmentBuilder` in `files:[...]`.

```ts
import { Container, Section, TextDisplay, ActionRow, Button, Separator, ButtonStyle, MessageFlags } from 'seyfert';
const container = new Container().setColor('Blurple').addComponents(
  new Section()
    .addComponents(new TextDisplay().setContent('## Docs\nRead the guide.'))
    .setAccessory(new Button().setStyle(ButtonStyle.Link).setLabel('Open').setURL('https://seyfert.dev')),
  new Separator().setDivider(true),
  new ActionRow<Button>().addComponents( // a ROW of buttons, not a section accessory
    new Button().setCustomId('confirm').setStyle(ButtonStyle.Success).setLabel('Confirm'),
  ),
);
await ctx.write({ components: [container], flags: MessageFlags.IsComponentsV2 });
// On a later edit/update, RE-SEND the flag:
await ctx.update({ components: [container], flags: MessageFlags.IsComponentsV2 });
```

## Review Checklist

- Imports come from `'seyfert'` root; no deep imports.
- `ActionRow` typed (`<Button>` / `<StringSelectMenu>`); rows never mix buttons + selects; ≤5 buttons/row, ≤5 rows.
- `componentType` literal is singular (`'Button'`, `'StringSelect'`, ...); use `as const`.
- `customId` set on every interactive component and matched by handler `customId`/`filter` (string or RegExp). Encode dynamic state in a RegExp customId (≤100 chars). Link/SKU buttons can't be handled.
- Handler/modal files `export default` the class (or `client.components.set`). Multiple matching ComponentCommands ALL run — keep customIds specific (anchor RegExp `^...$`).
- ComponentContext: NO `editResponse` (use `editOrReply(body, true)`); `write`/`editOrReply` return `void` unless 2nd arg `true`. Guard (`isStringSelectMenu()`) before reading `ctx.interaction.values`.
- Modals: top level only `Label`/`TextDisplay`; every input wrapped in its own `Label.setComponent`; NO `ActionRow`. Title + customId set. `setComponents([...])` is a single array; `addComponents` is rest. Read values by the WRAPPED customId; narrow `getInputValue` (`string | string[]`). Modal inline option is `{ waitFor }` only; respond to the returned submission.
- Modals are interaction-only (prefix ctx throws `CANNOT_USE_MODAL`); can't show after replying/deferring.
- Middlewares: `middlewares = [...] as const` (readonly); use `stop()` to skip silently, `stop('reason')` to deny -> `onMiddlewaresError`. `pass()` removed.
- Response method matches intent: `update`/`deferUpdate` for source message, `write`/`deferReply` for new reply; exactly one ack.
- Collectors: set `idle`/`timeout`; not durable across restarts/workers; one per message id. `filter`+`onPass`; `refresh()` (onStop re-creates, run-cb resets timers); `waitFor` -> `null` on timeout. Strip components on stop.
- Polls: `setQuestion` takes `{text}` not a string; `setDuration` is HOURS; ≤10 answers; add poll intents; pass the `PollBuilder` instance to `poll`. `endPoll()` (Message) vs `end()` (Poll). Answer ids 1-based.
- Components v2: ALWAYS `flags: MessageFlags.IsComponentsV2` on send AND edit; no top-level content/embeds/poll. `Spacing` only `Small`/`Large`. `Thumbnail.setMedia` (not setURL). Wrap bare buttons in `ActionRow`.
- THROW-on-serialize guards before send: missing emoji, missing option label/value, RadioGroup/CheckboxGroup 2–10 options, missing modal title/customId, Label without component, missing poll question/answers, Section accessory, MediaGalleryItem media, malformed `#hex`.
