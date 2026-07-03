# Builders (Embed, Components, Modal, Attachments, Poll, Formatter)

Source of truth: `src/builders/**`, `src/common/it/formatter.ts`. Verified against the authoritative Seyfert source.

## Import policy

EVERY builder, enum, and the `Formatter` import from the ROOT `'seyfert'` package. Never use deep
`'seyfert/lib/...'` imports for these. The root re-exports `src/builders/index.ts` plus
`ButtonStyle`/`ComponentType`/`TextInputStyle`/`Spacing`/`TimestampStyle`/`HeadingLevel`/`EmbedColors`/
`MessageFlags`/`ChannelType` (discord-api-types enums re-exported through `src/types`).

```ts
import {
  Embed, Formatter, AttachmentBuilder, PollBuilder,
  ActionRow, Button, ButtonStyle,
  StringSelectMenu, StringSelectOption, UserSelectMenu, RoleSelectMenu,
  ChannelSelectMenu, MentionableSelectMenu,
  Modal, TextInput, TextInputStyle, Label, FileUpload, Checkbox, CheckboxGroup,
  CheckboxGroupOption, RadioGroup, RadioGroupOption,
  Container, Section, TextDisplay, Thumbnail, MediaGallery, MediaGalleryItem,
  Separator, File, Spacing, TimestampStyle, HeadingLevel, EmbedColors,
  ComponentType, MessageFlags,
} from 'seyfert';
```

Class names that differ from the concept: file attachments = `AttachmentBuilder` (not `Attachment` —
that's a structure); polls = `PollBuilder` (not `Poll`). `TextInput` lives in `./Modal`. Serializer for
every builder is `.toJSON()` (NOT `.json()`); you rarely call it yourself — pass builders straight into
`ctx.write({ ... })`. In v5 several `toJSON()`s THROW on incomplete data (see each section), and that
throw surfaces inside `ctx.write`/`update`, so build fully before sending.

## Embed — `src/builders/Embed.ts`

`new Embed(data: Partial<APIEmbed> = {})`. All setters chain (`this`).
- `.setTitle(t?)`, `.setDescription(d?)`, `.setURL(u?)`, `.setColor(color?: ColorResolvable)`.
- `.setAuthor({ name, iconUrl, url })`, `.setFooter({ text, iconUrl })` — camelCase keys (→ snake_case).
- `.addFields(...RestOrArray<APIEmbedField>)` (appends), `.setFields(fields?)` (replaces). Field: `{ name, value, inline? }`.
- `.setImage(url?)`, `.setThumbnail(url?)`, `.setTimestamp(time = Date.now())` (stores ISO string).

`ColorResolvable = `#${string}` | number | keyof typeof EmbedColors | 'Random' | [r,g,b]`. Named keys e.g.
`'Blurple'` (0x5865f2), `'Blue'`, `'Green'`, `'Red'`. `'Random'` → random; UNKNOWN plain string → silently
`EmbedColors.Default` (black); negative/non-integer number OR malformed `#hex` (`'#zzz'`) → THROWS
`SeyfertError('INTERNAL_ERROR')` (v5 hardening — was silent `NaN`). Validate user-supplied colors.

```ts
const embed = new Embed()
  .setTitle('Server Info').setColor('Blurple')
  .setAuthor({ name: ctx.author.username, iconUrl: ctx.author.avatarURL() })
  .setFooter({ text: 'Seyfert' })
  .addFields({ name: 'Members', value: '1234', inline: true })
  .setImage('attachment://chart.png').setTimestamp();
await ctx.write({ embeds: [embed] });
```

## Classic interactive components

**ActionRow** — `src/builders/ActionRow.ts`. `new ActionRow<T>(data?)`. Holds up to 5 buttons OR ONE select
menu; cannot nest rows; a message accepts up to 5 rows. `.setComponents(...RestOrArray)` replaces,
`.addComponents(...RestOrArray)` appends (both accept array or rest args). Type it: `new ActionRow<Button>()`.

**Rebuilding an EXISTING message's row (disable/toggle buttons after a click).** A very common component-handler
task. `message.components` holds read-side component structures, not editable builders — call `.toBuilder()`
(from `BaseComponent`) on the one you want, then narrow the resulting builder union with `instanceof ActionRow`.
No casts, root imports only:

```ts
import { ActionRow, type ComponentContext } from 'seyfert';

async run(ctx: ComponentContext<'Button'>) {
  const row = ctx.message.components[0].toBuilder(); // BaseComponent#toBuilder() -> builder union
  if (!(row instanceof ActionRow)) return;           // type guard narrows to ActionRow

  for (const child of row.components) child.setDisabled(true); // every row child (button/select) has setDisabled
  await ctx.update({ components: [row] }); // or ctx.message.edit({ components: [row] })
}
```

`.toBuilder()` is the clean path — `instanceof ActionRow` narrows without any `as` (use `instanceof Button` /
`instanceof StringSelectMenu` on a child if you need a type-specific method). Do NOT reach for
`new ActionRow(component.toJSON())`: `.toJSON()` on a message component returns a broad union that fails the
constructor's `APIActionRowComponent` param. `fromComponent()` also won't help — it only rebuilds leaf components
and returns rows unchanged.

**Button** — `src/builders/Button.ts`. `.setCustomId(id)`, `.setLabel(l)`, `.setStyle(ButtonStyle)`,
`.setURL(url)` (Link style, no customId), `.setSKUId(id)` (Premium style), `.setEmoji(EmojiResolvable)`,
`.setDisabled(d = true)`. `.setEmoji` THROWS `SeyfertError('INVALID_EMOJI')` if unresolvable. The v4
`.disabled`/`.disabed` property is GONE — use `.setDisabled()`.

**Select menus** — `src/builders/SelectMenu.ts`. Common: `.setCustomId(id)`, `.setPlaceholder(t)`,
`.setValuesLength({ max, min })`, `.setDisabled(d=true)`, `.setRequired(r=true)` (modals only).
- `StringSelectMenu<Value extends string = string>`: `.addOption(...RestOrArray<StringSelectOption|raw>)`,
  `.setOptions(...)`. Accepts raw API option objects; a typed `StringSelectMenu<'a'|'b'>` rejects
  out-of-union raw values. `StringSelectOption`: `.setLabel(v)`, `.setValue(v)`, `.setDescription(v)`,
  `.setDefault(v=true)`, `.setEmoji(e)`. `.toJSON()` THROWS if `label`/`value` unset.
- `UserSelectMenu`/`RoleSelectMenu`/`ChannelSelectMenu`/`MentionableSelectMenu`: `.setDefault{Users,Roles,Channels,Mentionables}(...)`
  + additive `.addDefault*(...)`. `ChannelSelectMenu.setChannelTypes(ChannelType[])`.
  Mentionable default entries are `{ type: 'User' | 'Role', id }`.

```ts
const row = new ActionRow<Button>().setComponents(
  new Button().setCustomId('btn').setLabel('Click').setStyle(ButtonStyle.Primary),
  new Button().setStyle(ButtonStyle.Link).setLabel('Docs').setURL('https://seyfert.dev'),
);
const menu = new ActionRow<StringSelectMenu>().setComponents(
  new StringSelectMenu().setCustomId('m').setPlaceholder('Pick')
    .addOption(new StringSelectOption().setLabel('A').setValue('a')),
);
await ctx.write({ components: [row, menu] });
```

## Modals — `src/builders/Modal.ts`

`Modal` is a plain class (not a component builder). `.setCustomId(id)`, `.setTitle(t)`,
`.addComponents(...RestOrArray)`, `.setComponents(arr)`, `.run(callback)` (inline submit handler),
`.toJSON()`. `.toJSON()` THROWS `MISSING_MODAL_CUSTOM_ID` / `MISSING_MODAL_TITLE` if unset.

Modern (v5) modals wrap each input in a `Label` (`src/builders/Label.ts`): `.setLabel(text)`,
`.setDescription(d)`, `.setComponent(child)` where child ∈ `TextInput | select menus | FileUpload |
Checkbox | CheckboxGroup | RadioGroup`. `Label.toJSON()` THROWS `MISSING_COMPONENT` if no component set.

`TextInput`: `.setCustomId(id)`, `.setStyle(TextInputStyle.Short|Paragraph)`, `.setPlaceholder(p)`,
`.setLength({ max, min })`, `.setValue(v)`, `.setRequired(r=true)`.

Modal-only inputs:
- `FileUpload`: `setCustomId`, `setMinValues`, `setMaxValues`, `setRequired`, `setId`.
- `Checkbox`: `setCustomId`, `setDefault(bool)`, `setId` (no `setRequired`).
- `CheckboxGroup`: `setCustomId`, `setOptions`/`addOptions(...RestOrArray<CheckboxGroupOption>)`,
  `setSelectionLimit({ max, min })`, `setRequired(bool)`, `setId`. `.toJSON()` THROWS
  `INVALID_OPTIONS_LENGTH` unless 2–10 options.
- `RadioGroup`: same as above MINUS `setSelectionLimit` (radio = single choice). Same 2–10 throw.
- `CheckboxGroupOption` / `RadioGroupOption` constructors REQUIRE data up front in v5 —
  `new RadioGroupOption({ label, value })` (the empty `new RadioGroupOption()` is gone). They also expose
  chainable `.setLabel/.setValue/.setDescription/.setDefault`. `RadioGroupOption.toJSON()` THROWS if `label`/`value` missing.

```ts
const modal = new Modal().setCustomId('feedback').setTitle('Feedback').addComponents(
  new Label().setLabel('Your message').setComponent(
    new TextInput().setCustomId('msg').setStyle(TextInputStyle.Paragraph).setRequired(),
  ),
  new Label().setLabel('Priority').setComponent(
    new RadioGroup().setCustomId('prio').setOptions(
      new RadioGroupOption({ label: 'Low', value: 'low' }),
      new RadioGroupOption({ label: 'High', value: 'high' }),
    ),
  ),
);
await ctx.modal(modal); // present from a command/component interaction (NOT a prefix ctx → CANNOT_USE_MODAL)
```

## Components v2 — layout builders

REQUIRE `flags: MessageFlags.IsComponentsV2` (`1 << 15`) on EVERY send AND every edit/update; Seyfert
NEVER auto-applies it, and a v2 message cannot also carry top-level `content`/`embeds`/`poll`/`stickers`
(move all text into `TextDisplay`). All extend `BaseComponentBuilder`; each adds its own `setId(n)`.
- `Container`: `.addComponents/.setComponents(...)` children = `ActionRow | TextDisplay | Section |
  MediaGallery | Separator | File`; `.setColor(ColorResolvable)` (accent), `.setSpoiler(s=true)`, `.setId(n)`.
  Does NOT accept a bare `Button` — wrap buttons in an `ActionRow`.
- `Section<Button|Thumbnail>`: `.addComponents(...TextDisplay)`/`.setComponents(...)`, `.setAccessory(Button|Thumbnail)`.
  `.toJSON()` THROWS `MISSING_ACCESSORY` if no accessory.
- `TextDisplay`: `.setContent(string)` (markdown, incl. `##` headings / `-#` small text), `.setId(n)`.
- `Thumbnail`: `.setMedia(url)`, `.setDescription(d)`, `.setSpoiler(s=true)`, `.setId(n)`.
- `MediaGallery`: `.addItems/.setItems(...MediaGalleryItem)`, `.setId(n)`.
  `MediaGalleryItem` (plain class, NO setId): `.setMedia(url)`, `.setDescription(d)`, `.setSpoiler(s=true)`;
  `.toJSON()` THROWS `MISSING_MEDIA`.
- `Separator`: `.setSpacing(Spacing)`, `.setDivider(d=false)`, `.setId(n)`. `Spacing` has ONLY
  `Small=1`, `Large=2` (NO `None`/`Medium`). `setDivider` default is `false` — pass `true` for a line.
- `File`: `.setMedia(url)` (use `attachment://name`), `.setSpoiler(s=true)`, `.setId(n)`.

```ts
const container = new Container().addComponents(
  new Section().setAccessory(new Thumbnail().setMedia('attachment://x.png'))
    .addComponents(new TextDisplay().setContent('## Hi\nbody text')),
  new Separator().setSpacing(Spacing.Small).setDivider(true),
  new ActionRow<Button>().addComponents(
    new Button().setCustomId('ok').setStyle(ButtonStyle.Success).setLabel('OK'),
  ),
).setColor('#5865f2');
await ctx.write({ components: [container], flags: MessageFlags.IsComponentsV2 });
```

## Attachments — `src/builders/Attachment.ts`

`AttachmentBuilder`: `.setName(name)` (sets filename), `.setDescription(d)`, `.setSpoiler(bool)`
(prefixes `SPOILER_`), `.setFile(type: 'url'|'buffer'|'path', data)`. `buffer` accepts
`ReadableStream | Buffer | ArrayBuffer | Uint8Array | Uint8ClampedArray`. Resolution does real I/O at send
time and THROWS `SeyfertError('INVALID_ATTACHMENT_TYPE')` on a non-http url, non-OK fetch, or missing path.
Send via `ctx.write({ files: [att] })`; reference inside embeds/v2 via `attachment://<name>`.

```ts
const att = new AttachmentBuilder().setName('chart.png').setFile('url', 'https://e.com/c.png');
await ctx.write({ embeds: [new Embed().setImage('attachment://chart.png')], files: [att] });
```

## Polls — `src/builders/Poll.ts`

`PollBuilder`: `.setQuestion({ text, emoji? })`, `.addAnswers/.setAnswers(...RestOrArray<PollMedia>)`,
`.setDuration(hours)`, `.allowMultiselect(v=true)` (NOTE: it is `allowMultiselect`, NOT `setMultiSelect`).
`PollMedia = { text?, emoji?: EmojiResolvable }`. `.toJSON()` THROWS `MISSING_POLL_QUESTION` /
`MISSING_POLL_ANSWERS`; invalid emoji → `INVALID_EMOJI`. Send via `ctx.write({ poll })`.

```ts
const poll = new PollBuilder()
  .setQuestion({ text: 'Deploy on Friday?' })
  .setAnswers({ text: 'Yes' }, { text: 'No', emoji: '🙅' })
  .setDuration(24).allowMultiselect(false);
await ctx.write({ poll });
```

## Formatter — `src/common/it/formatter.ts`

`Formatter` is a CONST OBJECT (not a class; no `new`). Markdown helpers return narrow template-literal
types in v5: `bold`→`**${string}**`, `italic`, `underline`, `strikeThrough`, `spoiler`, `inlineCode(c)`,
`quote`→`> ${string}`, `blockQuote`→`>>> ${string}`, `hyperlink(content, url)`,
`header(content, level=HeadingLevel.H1)`, `list(items[], ordered=false)`.

CRITICAL arg order: `codeBlock(content: string, language = 'txt')` — CONTENT FIRST, language second.

```ts
Formatter.codeBlock('const x = 1', 'ts'); // ```ts\nconst x = 1\n```
```

Mentions/links: `userMention(id)` `<@id>`, `roleMention(id)` `<@&id>`, `channelMention(id)` `<#id>`,
`emojiMention(id, name, animated=false)` (ID FIRST, then name; null name → `_`), `messageLink(g,c,m)`,
`channelLink(c, g='@me')`, `generateOAuth2URL(appId, { scopes, permissions?, disableGuildSelect? })`.
`timestamp(ts: Date | number, style = TimestampStyle.RelativeTime)` — accepts Date OR a millisecond
number (NOT a string) → `<t:seconds:style>`; DEFAULTS to relative time ('R') in v5. `TimestampStyle`:
ShortTime`t` MediumTime`T` ShortDate`d` LongDate`D` LongDateShortTime`f` FullDateShortTime`F`
RelativeTime`R` (+ S/s variants). `HeadingLevel`: H1/H2/H3.

---

## Recipes (copy-paste, source-verified)

### 1. Embed + Formatter `userinfo` command

```ts
import { Embed, Formatter, Command, Declare, type CommandContext } from 'seyfert';

@Declare({ name: 'userinfo', description: 'Show info about you' })
export default class UserInfo extends Command {
  async run(ctx: CommandContext) {
    const u = ctx.author;
    const embed = new Embed()
      .setTitle(Formatter.bold(u.username))
      .setThumbnail(u.avatarURL())
      .setDescription(Formatter.blockQuote(u.globalName ?? 'No display name'))
      .addFields(
        { name: 'Mention', value: Formatter.userMention(u.id), inline: true },
        { name: 'ID', value: Formatter.inlineCode(u.id), inline: true },
        { name: 'Created', value: Formatter.timestamp(u.createdAt), inline: true },
      )
      .setColor('Blurple');
    await ctx.write({ embeds: [embed] });
  }
}
```

### 2. Button + inline collector confirm flow

`ctx.write(body, true)` returns the `Message` so you can attach a collector scoped to that message.
`message.createComponentCollector({...})` gives `.run(customId, cb)`. Filter by user so others can't drive it.

```ts
import { ActionRow, Button, ButtonStyle, Command, Declare, type CommandContext } from 'seyfert';

@Declare({ name: 'confirm', description: 'Confirm an action' })
export default class Confirm extends Command {
  async run(ctx: CommandContext) {
    const row = new ActionRow<Button>().setComponents(
      new Button().setCustomId('yes').setStyle(ButtonStyle.Success).setLabel('Yes'),
      new Button().setCustomId('no').setStyle(ButtonStyle.Danger).setLabel('No'),
    );
    const message = await ctx.write({ content: 'Proceed?', components: [row] }, true);

    const collector = message.createComponentCollector({
      idle: 30_000, // ms with no interaction → onStop('idle')
      filter: i => i.user.id === ctx.author.id,
      onStop: reason => ctx.editOrReply({ content: `Closed (${reason})`, components: [] }),
    });
    collector.run('yes', i => i.update({ content: 'Confirmed', components: [] }));
    collector.run('no', i => i.update({ content: 'Cancelled', components: [] }));
  }
}
```

### 3. Persistent button handler (ComponentCommand)

A `ComponentCommand` re-routes by `customId` and survives restarts (unlike a collector). `customId` may be
a `RegExp`; an optional `filter(ctx)` refines after the match. Use `ctx.update(...)` to edit the source
message; `ctx.deferUpdate()` to ACK silently. There is NO `editResponse` on a `ComponentContext` in v5.

```ts
import { ComponentCommand, type ComponentContext } from 'seyfert';

export default class Paginator extends ComponentCommand {
  componentType = 'Button' as const;
  customId = /^page:(next|prev):\d+:\d+$/; // "page:next:0:<openerId>" (opener encoded)

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

### 4. Open a modal from a button, then read fields with accessors

`ctx.modal(builder, { waitFor })` resolves to the submit interaction (or null on timeout). The submit
context exposes the full v5 accessor set: `getInputValue`, `getRadioValues`, `getCheckboxValues`,
`getCheckbox`, `getChannels`, `getRoles`, `getUsers`, `getMentionables`, `getFiles`.

```ts
import {
  ComponentCommand, type ComponentContext,
  Modal, Label, TextInput, TextInputStyle, RadioGroup, RadioGroupOption,
} from 'seyfert';

export default class FeedbackButton extends ComponentCommand {
  componentType = 'Button' as const;
  customId = 'open-feedback';

  async run(ctx: ComponentContext<'Button'>) {
    const modal = new Modal().setCustomId('feedback').setTitle('Feedback').addComponents(
      new Label().setLabel('Your message').setComponent(
        new TextInput().setCustomId('msg').setStyle(TextInputStyle.Paragraph).setRequired(),
      ),
      new Label().setLabel('Priority').setComponent(
        new RadioGroup().setCustomId('prio').setOptions(
          new RadioGroupOption({ label: 'Low', value: 'low' }),
          new RadioGroupOption({ label: 'High', value: 'high' }),
        ),
      ),
    );

    const submitted = await ctx.modal(modal, { waitFor: 60_000 });
    if (!submitted) return; // timed out
    const msg = submitted.getInputValue('msg', true);     // required → string
    const prio = submitted.getRadioValues('prio');        // string (single value)
    await submitted.write({ content: `[${prio}] ${msg}` });
  }
}
```

### 5. Components v2 announcement (Container + gallery + sections + file)

```ts
import {
  Container, MediaGallery, MediaGalleryItem, TextDisplay,
  Section, Button, Separator, ButtonStyle, MessageFlags, AttachmentBuilder,
} from 'seyfert';

const banner = new AttachmentBuilder().setName('banner.png').setFile('url', 'https://e.com/banner.png');

const container = new Container().addComponents(
  new MediaGallery().addItems(new MediaGalleryItem().setMedia('attachment://banner.png')),
  new TextDisplay().setContent('## New Components!\nFully composable, in any order.'),
  new Section()
    .addComponents(new TextDisplay().setContent('Read the overview:'))
    .setAccessory(new Button().setStyle(ButtonStyle.Link).setLabel('Docs').setURL('https://seyfert.dev')),
  new Separator().setDivider(true),
  new TextDisplay().setContent('-# Composed entirely from v2 components.'),
);

await ctx.write({ components: [container], files: [banner], flags: MessageFlags.IsComponentsV2 });
// Editing later? RE-SEND the flag, or Discord rejects the edit:
await ctx.editOrReply({ components: [container], flags: MessageFlags.IsComponentsV2 });
```

### 6. Paginated embeds (collector switching pages)

```ts
import { Embed, ActionRow, Button, ButtonStyle, Command, Declare, type CommandContext } from 'seyfert';

@Declare({ name: 'pages', description: 'Browse paginated embeds' })
export default class Pages extends Command {
  async run(ctx: CommandContext) {
    const pages = [new Embed().setTitle('Page 1'), new Embed().setTitle('Page 2')];
    let i = 0;
    const row = new ActionRow<Button>().addComponents(
      new Button().setCustomId('prev').setLabel('Prev').setStyle(ButtonStyle.Secondary),
      new Button().setCustomId('next').setLabel('Next').setStyle(ButtonStyle.Primary),
    );
    const message = await ctx.write({ embeds: [pages[i]], components: [row] }, true);
    const collector = message.createComponentCollector({
      idle: 60_000, filter: x => x.user.id === ctx.author.id,
    });
    collector.run('prev', x => { i = (i - 1 + pages.length) % pages.length; return x.update({ embeds: [pages[i]] }); });
    collector.run('next', x => { i = (i + 1) % pages.length; return x.update({ embeds: [pages[i]] }); });
  }
}
```

## Common patterns / gotchas

- Serializer is `.toJSON()`, never `.json()`. v5 builders THROW on incomplete data at serialize time
  (inside `ctx.write`/`update`): `PollBuilder` needs question + ≥1 answer; `Modal` needs custom_id + title;
  `Section` needs an accessory; `MediaGalleryItem` needs media; `Checkbox/RadioGroup` need 2–10 options;
  `StringSelectOption`/`RadioGroupOption` need label + value. Catch via `SeyfertError.is(err, 'CODE')`.
- v2 layout messages MUST set `flags: MessageFlags.IsComponentsV2` on send AND every edit; no top-level
  text/embeds/poll allowed — all text goes in `TextDisplay`.
- `Spacing` only `Small`/`Large`; `Separator.setDivider` default is `false` — pass `true` explicitly.
- `Container` rejects a bare `Button`; wrap buttons in an `ActionRow` (or use one as a `Section` accessory).
- ActionRow: ≤5 buttons OR 1 select; no nesting; typed `<Button>`/`<StringSelectMenu>`; ≤5 rows/message.
- Link buttons use `setURL` + `ButtonStyle.Link` (no customId); premium use `setSKUId` + `ButtonStyle.Premium`.
- `Formatter.codeBlock(content, language)` — content FIRST. `emojiMention(id, name)` — id FIRST.
  `Formatter` is a plain object — never `new Formatter()`. `timestamp` takes Date/number ms, defaults to relative.
- `setColor`: unknown plain string → black; negative/non-int number OR malformed `#hex` THROWS. Validate user input.
- Reference uploads via `attachment://<name>` matching `AttachmentBuilder.setName(...)`; pass in `files: [...]`.
  URL/path attachments do real I/O and throw `INVALID_ATTACHMENT_TYPE` — wrap untrusted sources in try/catch.
- `ctx.write`/`editOrReply` return `void` for interactions UNLESS you pass the response flag `true`
  (then a `WebhookMessageStructure`). Pass `true` when you need the `Message` to attach a collector.
- `RadioGroupOption`/`CheckboxGroupOption` need data in the constructor — `new RadioGroupOption({ label, value })`.
- `ctx.modal(...)` (v5) opens a modal; on a prefix context it throws `CANNOT_USE_MODAL`.

## Review Checklist

- [ ] Imported every builder/enum/`Formatter` from `'seyfert'` root — no deep imports.
- [ ] Correct class names: `AttachmentBuilder`, `PollBuilder`, `TextInput` (from Modal), `MediaGalleryItem`.
- [ ] `Formatter.codeBlock(content, language)` content FIRST; `emojiMention(id, name)` id FIRST; `Formatter` no `new`.
- [ ] Serialize with `.toJSON()`, not `.json()`; only on send (throws if required fields unset).
- [ ] v2 layout messages set `flags: MessageFlags.IsComponentsV2` on send AND every edit; no top-level content/embeds.
- [ ] `Spacing` only `Small`/`Large`; `Separator.setDivider` default `false` — pass explicitly.
- [ ] Section has an accessory; MediaGalleryItem has media; Checkbox/RadioGroup have 2-10 options.
- [ ] `Container` children are rows/text/sections/gallery/separator/file — never a bare `Button`.
- [ ] ActionRow: ≤5 buttons OR 1 select; no nesting; typed `<Button>`/`<StringSelectMenu>`.
- [ ] Set `customId` on interactive components; link buttons use `setURL`+`ButtonStyle.Link` (no customId).
- [ ] Poll uses `allowMultiselect` (not `setMultiSelect`); has a question + answers.
- [ ] `RadioGroupOption`/`CheckboxGroupOption` constructed with `{ label, value }` (no empty ctor).
- [ ] `setColor`: validate user input (unknown string → black; negative/non-int/malformed hex throws).
- [ ] Reference uploads via `attachment://<name>` matching `setName(...)`; pass in `files: [...]`.
- [ ] Need the sent `Message` (e.g. for a collector)? Pass the response flag: `ctx.write(body, true)`.
- [ ] No `editResponse` on `ComponentContext` (removed in v5) — use `editOrReply`/`update`.
