# Modals

Original source URL: https://seyfert-web-git-seyfert-v5-tiramisulabs.vercel.app/docs/learn/components/modals
Coverage reference: components.md (+ builders.md)
Verification status: Source-verified against ./src (the authoritative Seyfert source)

## Page Summary

Modals request structured user input via a popup. Seyfert's v5 modal supports far more than text inputs: any interactive component (TextInput, all five select menus, FileUpload, Checkbox, CheckboxGroup, RadioGroup) is placed inside a `Label`, while `TextDisplay` static text can sit at the modal's top level. Modals are built with the `Modal` builder and handled either by a durable `ModalCommand` handler class or inline via an awaitable `ctx.modal(...)` / `interaction.modal(...)`. All builders and `ModalCommand`/`ModalContext` are root exports from `'seyfert'`.

## Key APIs (verified)

Modal builder — `src/builders/Modal.ts`
- `new Modal(data?)` — top-level components are typed `ModalBuilderComponents = Label | TextDisplay` (`src/builders/types.ts:46`).
- `.setTitle(title)`, `.setCustomId(id)` — both required; `toJSON()` throws `SeyfertError('MISSING_MODAL_CUSTOM_ID' | 'MISSING_MODAL_TITLE')` if unset.
- `.setComponents(components: ModalBuilderComponents[])` — REPLACES; takes a single array argument (NOT rest).
- `.addComponents(...components)` — appends; accepts rest OR a single array (`RestOrArray`).
- `.run(callback)` — inline submit callback (`ModalSubmitCallback = (interaction: ModalSubmitInteraction) => any`). Stored on `__exec`.

Label builder — `src/builders/Label.ts`
- `.setLabel(label)`, `.setDescription(description)`, `.setComponent(component)`.
- `LabelBuilderComponents = TextInput | BuilderSelectMenus | FileUpload | Checkbox | CheckboxGroup | RadioGroup` (`Label.ts:9`). `toJSON()` throws `SeyfertError('MISSING_COMPONENT')` if no component set. A Label holds exactly ONE component.

TextInput builder — `src/builders/Modal.ts`
- `.setCustomId(id)`, `.setStyle(TextInputStyle.Short|Paragraph)`, `.setPlaceholder(s)`, `.setLength({ max, min })`, `.setValue(v)`, `.setRequired(required = true)`.

TextDisplay builder — `src/builders/TextDisplay.ts`
- `.setId(id: number)`, `.setContent(content: string)`. (Class JSDoc example wrongly shows `.content(...)`; the real method is `.setContent`.) Can be placed at the modal top level WITHOUT a Label.

FileUpload builder — `src/builders/FileUpload.ts`
- `.setCustomId(id)`, `.setId(n)`, `.setMinValues(n)`, `.setMaxValues(n)`, `.setRequired(bool)`. Must be wrapped in a Label.

Checkbox builder (single checkbox) — `src/builders/Checkbox.ts`
- `.setCustomId(id)`, `.setId(n)`, `.setDefault(value: boolean)`. No validation throw.

CheckboxGroup builder — `src/builders/CheckboxGroup.ts`
- `.setCustomId(id)`, `.setId(n)`, `.setSelectionLimit({ max, min })`, `.setRequired(bool)`, `.setOptions(...)`, `.addOptions(...)` taking `CheckboxGroupOption` (rest-or-array). `toJSON()` throws `INVALID_OPTIONS_LENGTH` unless 2–10 options.
- `CheckboxGroupOption(data)`: `.setValue`, `.setLabel`, `.setDescription`, `.setDefault(value: boolean)`. Its `toJSON()` does NOT throw on missing fields (unlike RadioGroupOption).

RadioGroup builder — `src/builders/RadioGroup.ts`
- `.setCustomId(id)`, `.setId(n)`, `.setRequired(bool)`, `.setOptions(...)`, `.addOptions(...)` taking `RadioGroupOption` (rest-or-array). `toJSON()` throws `INVALID_OPTIONS_LENGTH` unless 2–10 options.
- `RadioGroupOption(data)`: `.setLabel`, `.setValue`, `.setDescription`, `.setDefault(value = true)`. Option `toJSON()` throws `MISSING_RADIO_GROUP_OPTION_LABEL` / `MISSING_RADIO_GROUP_OPTION_VALUE` if missing.

Select menus inside modals — `src/builders/SelectMenu.ts`: `StringSelectMenu`, `RoleSelectMenu`, `ChannelSelectMenu`, `UserSelectMenu`, `MentionableSelectMenu` (`BuilderSelectMenus`). `StringSelectMenu.setOptions(...)` / `.addOption(...)` accept `StringSelectOption` instances OR raw `APISelectMenuOption` objects (rest-or-array). `StringSelectOption(data?)` supports both `new StringSelectOption({ label, value })` and `new StringSelectOption().setLabel(...).setValue(...)`.

ModalCommand handler — `src/components/modalcommand.ts`
- `abstract class ModalCommand` with `type = InteractionCommandType.MODAL`, `abstract run(context: ModalContext): any`.
- Matching: optional `customId?: string | RegExp` AND/OR optional `filter?(context): boolean | Promise<boolean>`. Internal `_filter` requires the `customId` match (string equality, or `.match()` for RegExp) to pass FIRST, then runs `filter` if present. With neither set, it matches EVERY modal submit.
- `middlewares: readonly (keyof ResolvedRegisteredMiddlewares)[] = []` (readonly — no `.push`), plus hooks `onBeforeMiddlewares`, `onAfterRun(ctx, error)`, `onRunError(ctx, error)`, `onMiddlewaresError(ctx, error, metadata)`, and `onInternalError(client, modal, error?)` — v5 adds the `modal` instance param before `error`.

ModalContext — `src/components/modalcontext.ts` (readers delegate to `ModalSubmitInteraction`, `src/structures/Interaction.ts`)
- `getInputValue(id, required?)` -> `string | string[] | undefined` (returns `value`, or `values` array for value-array components). `required: true` overload returns `string | string[]`.
- `getRadioValues(id, required?)` -> `string`. `getCheckboxValues(id, required?)` -> `string[]`. `getCheckbox(id, required?)` -> `boolean`.
- `getFiles(id, required?)` -> `Attachment[]`. `getUsers`/`getRoles`/`getChannels`/`getMentionables(id, required?)` -> arrays of resolved structures (`UserStructure[]`, `GuildRoleStructure[]`, `AllChannels[]`, mentionable union[]).
- Non-`required` overloads return `… | void` (or `| undefined` for input/files). All readers throw `SeyfertError('INTERNAL_ERROR')` when `required: true` and the component/value is absent.
- Response helpers: `write(body, fetchReply?)`, `editOrReply(body, fetchReply?)`, `deferReply(ephemeral?, fetchReply?)`, `update(body, withResponse?)`, `deferUpdate()`, `editResponse(body)` (STILL EXISTS on modal ctx, returns the message), `followup`, `fetchResponse`, `deleteResponse`.
- `write`/`editOrReply`/`deferReply` return `void`/`undefined` UNLESS the response flag is `true` (then a `WebhookMessageStructure`).
- Context data: `customId`, `components`, `t` (respects `preferGuildLocale`), `author` (=`interaction.user`), `member`, `guildId`, `channelId`, `channel(mode?)`, `guild(mode?)`, `me(mode?)`, `isModal()`, `inGuild()` -> narrows to `GuildModalContext`. Can re-open a modal via `ctx.modal(...)`.

Sending a modal — `interaction.modal(body, options?)` / `ctx.modal(body, options?)` (`modalcontext.ts:217`, type `src/common/types/write.ts:76`). Also available on chat-input command and component contexts.
- `ModalCreateBodyRequest = APIModalInteractionResponse['data'] | Modal` — pass a `Modal` builder directly.
- No options -> `Promise<undefined>`. With `ModalCreateOptions { waitFor?: number }` -> `Promise<ModalSubmitInteraction | null>` (awaitable inline collection; `null` on timeout).

## Code Examples (verified)

Text input modal:
```ts
import { Modal, Label, TextInput, TextInputStyle } from 'seyfert';

const textInput = new TextInput()
  .setCustomId('text')
  .setStyle(TextInputStyle.Short)
  .setLength({ max: 200, min: 10 });

const modal = new Modal()
  .setTitle('My modal')
  .setCustomId('myfirstmodal')
  .setComponents([new Label().setLabel('Write something').setComponent(textInput)]);

// inside a command/component context:
await ctx.modal(modal);
```

Static text + a select menu in one modal:
```ts
import { Modal, Label, TextDisplay, StringSelectMenu, StringSelectOption } from 'seyfert';

const select = new StringSelectMenu()
  .setCustomId('strings')
  .setOptions(
    new StringSelectOption({ label: 'Option 1', value: 'option1' }),
    new StringSelectOption({ label: 'Option 2', value: 'option2' }),
  );

const modal = new Modal()
  .setTitle('My modal')
  .setCustomId('myfirstmodal')
  .setComponents([
    new TextDisplay().setContent('Pick one:'),       // top-level static text (no Label)
    new Label().setLabel('Choose an option').setComponent(select), // interactive -> wrapped in Label
  ]);
```

Radio group inside a modal (2–10 options required):
```ts
import { Modal, Label, RadioGroup, RadioGroupOption } from 'seyfert';

const radio = new RadioGroup()
  .setCustomId('plan')
  .setRequired(true)
  .setOptions(
    new RadioGroupOption({ label: 'Free', value: 'free' }),
    new RadioGroupOption({ label: 'Pro', value: 'pro' }),
  );

const modal = new Modal()
  .setTitle('Plan')
  .setCustomId('plan-modal')
  .setComponents([new Label().setLabel('Choose a plan').setComponent(radio)]);
```

Durable handler (matches via `customId` and/or `filter`):
```ts
import { ModalCommand, type ModalContext } from 'seyfert';

export default class MyModal extends ModalCommand {
  customId = 'myfirstmodal'; // string or RegExp; preferred over filter() for exact matches

  async run(ctx: ModalContext) {
    const text = ctx.getInputValue('text', true); // string | string[]
    return ctx.write({ content: `Your input was: ${text}` });
  }
}
```

Triggering from a component collector:
```ts
const collector = message.createComponentCollector();
collector.run('open', interaction => interaction.modal(modal));
```

Awaiting a submission inline (no separate handler):
```ts
const submission = await ctx.modal(modal, { waitFor: 60_000 });
if (submission) await submission.write({ content: submission.getInputValue('text', true) as string });
```

## Recipes / Worked Examples

Multi-field "profile" modal handled by one ModalCommand, reading several component types:
```ts
// builder (define once, reuse where you open it)
import { Modal, Label, TextInput, TextInputStyle, StringSelectMenu, StringSelectOption, CheckboxGroup, CheckboxGroupOption } from 'seyfert';

export const profileModal = new Modal()
  .setTitle('Edit profile')
  .setCustomId('profile-edit')
  .setComponents([
    new Label().setLabel('Display name').setComponent(
      new TextInput().setCustomId('name').setStyle(TextInputStyle.Short).setRequired(true).setLength({ max: 32, min: 1 }),
    ),
    new Label().setLabel('Bio').setComponent(
      new TextInput().setCustomId('bio').setStyle(TextInputStyle.Paragraph).setLength({ max: 200 }),
    ),
    new Label().setLabel('Favourite color').setComponent(
      new StringSelectMenu().setCustomId('color').setOptions(
        new StringSelectOption({ label: 'Red', value: 'red' }),
        new StringSelectOption({ label: 'Blue', value: 'blue' }),
      ),
    ),
    new Label().setLabel('Notifications').setComponent(
      new CheckboxGroup().setCustomId('notifs').setSelectionLimit({ max: 2, min: 0 }).setOptions(
        new CheckboxGroupOption({ label: 'DMs', value: 'dm' }),
        new CheckboxGroupOption({ label: 'Mentions', value: 'mention' }),
      ),
    ),
  ]);
```
```ts
// handler
import { ModalCommand, type ModalContext } from 'seyfert';

export default class ProfileEdit extends ModalCommand {
  customId = 'profile-edit';

  async run(ctx: ModalContext) {
    const name = ctx.getInputValue('name', true) as string;
    const bio = ctx.getInputValue('bio') ?? '';            // optional -> may be undefined
    const [color] = ctx.getInputValue('color', true);       // select value comes back as string[]
    const notifs = ctx.getCheckboxValues('notifs', true);   // string[]

    await ctx.write({
      content: `Saved **${name}** (${color}) — bio: ${bio || 'none'} — notifs: ${notifs.join(', ') || 'none'}`,
      flags: 64, // ephemeral
    });
  }
}
```

Dynamic per-entity modals routed by a RegExp customId (e.g. `report-<messageId>`):
```ts
import { ModalCommand, type ModalContext, Modal, Label, TextInput, TextInputStyle } from 'seyfert';

// open it with the target id baked into the customId
export function buildReportModal(messageId: string) {
  return new Modal()
    .setTitle('Report message')
    .setCustomId(`report-${messageId}`)
    .setComponents([
      new Label().setLabel('Reason').setComponent(
        new TextInput().setCustomId('reason').setStyle(TextInputStyle.Paragraph).setRequired(true),
      ),
    ]);
}

export default class ReportHandler extends ModalCommand {
  customId = /^report-(\d+)$/; // RegExp — _filter runs .match() before run()

  async run(ctx: ModalContext) {
    const [, messageId] = ctx.customId.match(/^report-(\d+)$/)!; // re-extract the id
    const reason = ctx.getInputValue('reason', true) as string;
    await ctx.client.messages.write(MOD_LOG_CHANNEL, { content: `Report on ${messageId}: ${reason}` });
    await ctx.write({ content: 'Thanks, your report was sent.', flags: 64 });
  }
}
```

Command flow: defer-aware open + inline await + guild narrowing:
```ts
import { Command, type CommandContext } from 'seyfert';
import { profileModal } from './profileModal';

export default class ProfileCommand extends Command {
  async run(ctx: CommandContext) {
    // ctx.modal throws CANNOT_USE_MODAL on a prefix (message) context — interaction only.
    const submitted = await ctx.modal(profileModal, { waitFor: 120_000 });
    if (!submitted) return; // timed out

    const name = submitted.getInputValue('name', true) as string;
    // submitted is a fresh ModalContext: respond to IT, not the original ctx.
    await submitted.write({ content: `Welcome, ${name}!`, flags: 64 });
  }
}
```

Resolved selects + file upload inside a modal:
```ts
import { ModalCommand, type ModalContext, Modal, Label, UserSelectMenu, FileUpload } from 'seyfert';

export const ticketModal = new Modal()
  .setTitle('Open ticket')
  .setCustomId('ticket')
  .setComponents([
    new Label().setLabel('Assign to').setComponent(new UserSelectMenu().setCustomId('assignee')),
    new Label().setLabel('Attach a screenshot').setComponent(
      new FileUpload().setCustomId('proof').setMaxValues(1).setRequired(false),
    ),
  ]);

export default class TicketHandler extends ModalCommand {
  customId = 'ticket';

  async run(ctx: ModalContext) {
    const [assignee] = ctx.getUsers('assignee', true);  // UserStructure[]
    const files = ctx.getFiles('proof') ?? [];          // Attachment[] | undefined
    await ctx.write({
      content: `Ticket assigned to ${assignee.username}${files.length ? ` with ${files.length} file(s)` : ''}.`,
    });
  }
}
```

## Common patterns / gotchas

- `ctx.modal(...)` / `interaction.modal(...)` is interaction-only. On a prefix (message) command context it throws `SeyfertError('CANNOT_USE_MODAL')` — guard with `ctx.isChatInput()`/interaction checks, or catch via `SeyfertError.is(err, 'CANNOT_USE_MODAL')`. You also cannot show a modal after already replying/deferring the interaction.
- You can NOT respond to a slash command with both a deferred reply AND a modal — show the modal first as the initial response.
- `getInputValue` returns `string | string[]`: plain text inputs give a `string`, but select menus surfaced via `getInputValue` give a `string[]`. Use the dedicated readers when you can (`getRadioValues` -> `string`, `getCheckboxValues` -> `string[]`, `getCheckbox` -> `boolean`), or narrow/cast.
- Read every value by the WRAPPED component's `customId`, not the Label's. Labels carry no customId.
- `setComponents([...])` REPLACES and takes a single array; `addComponents(...)` appends and takes rest-or-array. Mixing them up silently wipes earlier components.
- Inline `await ctx.modal(modal, { waitFor })` returns a NEW `ModalSubmitInteraction`/context — respond to the returned object, not the original ctx. It is `null` on timeout.
- Prefer the `customId` property over `filter()` for simple exact/RegExp matches; `_filter` checks `customId` FIRST, then `filter`. With neither, the handler matches EVERY modal submit (a footgun).
- Validation throws to expect from `toJSON()` (called on send): missing modal title/custom_id, Label without a component, RadioGroup/CheckboxGroup outside 2–10 options, RadioGroupOption missing label/value.
- A modal can hold up to 5 top-level rows (Discord limit). Each interactive component needs its own Label; only `TextDisplay` sits bare at the top level.
- `editResponse` exists on `ModalContext` (unlike `ComponentContext`, where it was removed in v5).

## Doc vs Source Corrections

- Upstream MDX's component list omits the boolean/choice components: src `Label` also accepts `Checkbox`, `CheckboxGroup`, and `RadioGroup` (`src/builders/Label.ts:9`). The doc only lists TextInput, the five selects, FileUpload, TextDisplay.
- Upstream shows modal matching only via `filter() { return context.customId === '…' }`; src `ModalCommand` also supports `customId: string | RegExp`, and `_filter` requires the customId match before `filter` runs (`src/components/modalcommand.ts`).
- Docs imply `getInputValue` returns a string; src returns `string | string[]` (array for value-array components) (`Interaction.ts`).
- Docs do not mention that `Modal.setComponents` takes a single array (use `addComponents` for rest), nor that `toJSON()` throws when title/custom_id are missing (`src/builders/Modal.ts`).
- Docs do not mention `ctx.modal(modal, { waitFor })` returning an awaitable `ModalSubmitInteraction | null` (`modalcontext.ts:217`, `write.ts:76`).
- Docs do not mention RadioGroup/CheckboxGroup requiring 2–10 options or `toJSON()` throwing `INVALID_OPTIONS_LENGTH`.
- Minor: `TextDisplay` class JSDoc shows `.content(...)`; the real method is `.setContent(...)` (the doc page example correctly uses `.setContent`).

## Source Anchors

- src/builders/index.ts, src/index.ts (root exports)

## Agent Guidance

- For one-off flows use `ctx.modal(modal, { waitFor })` and respond to the returned submission; for reusable flows define a `ModalCommand` class file.
- A thrown error in `run()` routes to `onRunError` → `onAfterRun` — the handler wraps `run()` for you, so don't add a `try/catch` just to report it. Set `modals.defaults.onRunError` once (it's separate from `commands.defaults`/`components.defaults`) or override `onRunError` on the class; with none of them a modal error is swallowed silently. Full decision guide: `handling-errors.md` → "Decision: try/catch vs onRunError".
