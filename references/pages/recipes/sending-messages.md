# Sending Messages

Original source URL: https://seyfert-web-git-seyfert-v5-tiramisulabs.vercel.app/docs/recipes/sending-messages
Coverage reference: builders.md (+ i18n-cache-recipes.md)
Verification status: Source-verified (branch more-qol)

## Page Summary

Shows the main ways to emit messages in Seyfert v5: replying to a command with `ctx.write()`, deferring then editing with `ctx.deferReply()` + `ctx.editOrReply()`, sending a standalone message to a channel via `ctx.client.messages.write()`, and attaching embeds, components (buttons / select menus inside `ActionRow`), files, and polls. All builders and context methods used by the doc examples exist on the current `more-qol` branch and are exported from the `seyfert` root.

Key v5 behavior: `ctx.write()` / `ctx.editOrReply()` / `ctx.deferReply()` return `void` for interactions UNLESS you pass the response flag (`withResponse: true`), in which case interaction calls resolve to a `WebhookMessageStructure`. Prefix commands always return a `MessageStructure`.

## Key APIs (verified)

Context methods (src/commands/applications/chatcontext.ts — `CommandContext`, also shared shape with menu/entry contexts):
- `ctx.write<WR>(body: InteractionCreateBodyRequest, withResponse?: WR)` — sends the first response. Interactions delegate to `interaction.write`; prefix commands `reply` or `write` the source message depending on `client.options.commands.reply`. (line 88)
- `ctx.deferReply<WR>(ephemeral = false, withResponse?: WR)` — defers. Interactions pass `MessageFlags.Ephemeral` when `ephemeral` is true; prefix commands send `{ content: 'Thinking...' }` (or `options.deferReplyResponse`). Sets `ctx.deferred`. (line 107)
- `ctx.editOrReply<WR>(body, withResponse?: WR)` — edits the existing response if one was sent, otherwise calls `write`. Body type is `InteractionCreateBodyRequest | InteractionMessageUpdateBodyRequest`. (line 145)
- `ctx.editResponse(body: InteractionMessageUpdateBodyRequest)` (line 126) — STILL EXISTS on `CommandContext` and `ModalContext`. NOTE: removed from `ComponentContext` in v5 (use `ctx.editOrReply` there).
- `ctx.deleteResponse()` (line 140), `ctx.followup(body: MessageWebhookCreateBodyRequest)` (line 156), `ctx.fetchResponse()` (line 163).
- `ctx.modal(body, options?)` (line 99) — opens a modal; throws `CANNOT_USE_MODAL` on a prefix context (no interaction).
- Response-flag semantics (`When<WR, ...>`): with `withResponse: true`, interaction calls resolve to `WebhookMessageStructure`; without it, interaction calls resolve to `void`. Prefix commands always resolve to a `MessageStructure`.

Standalone channel send — client `messages` shorter (src/common/shorters/messages.ts, class `MessageShorter`):
- `client.messages.write(channelId: string, body: MessageCreateBodyRequest): Promise<MessageStructure>` (line 16)
- `client.messages.edit(messageId, channelId, body)` (42), `client.messages.delete(messageId, channelId, reason?)` (78), `client.messages.fetch(messageId, channelId, force?)` (100).
- `client.messages.list(channelId, fetchOptions?)` (148) — query is optional in v5.
- `client.messages.purge(messages: string[], channelId, reason?)` (116), `client.messages.crosspost(messageId, channelId, reason?)` (67), `client.messages.endPoll(channelId, messageId)` (131).

Message structure methods (src/structures/Message.ts, class `Message`):
- `message.reply(body, fail = true)` (164) — sets `message_reference` for you; `body` omits `message_reference`.
- `message.edit(body)` (176), `message.write(body)` (180) — `write` sends to the same channel (no reference), `delete(reason?)` (184), `fetch(force = false)` (160), `crosspost(reason?)` (188).
- `message.endPoll()` (192), `message.getAnswerVoters(answerId, checkAnswer = false)` (196).

Builders (all exported from 'seyfert' via src/builders/index.ts):
- `Embed` — `.setTitle()`, `.setDescription()`, `.setColor()`, `.setFields()`, `.addFields()`, `.setFooter()`, `.setTimestamp()` (src/builders/Embed.ts).
- `ActionRow<T>` — `.addComponents(...RestOrArray)`, `.setComponents(...)` (src/builders/ActionRow.ts).
- `Button` — `.setCustomId(id)`, `.setLabel(label)`, `.setStyle(ButtonStyle)`, `.setURL(url)` for link buttons, `.setDisabled(bool)` (src/builders/Button.ts). NOTE: the v4 `.disabed`/`.disabled` typo property is gone — use `.setDisabled(...)`.
- `StringSelectMenu` — `.setCustomId(id)`, `.addOption(...RestOrArray<StringSelectOptionResolvable>)`, `.setPlaceholder()`, `.setValuesLength({ min, max })` (src/builders/SelectMenu.ts).
- `StringSelectOption` — `.setLabel(label)`, `.setValue(value)`, `.setDescription()`, `.setDefault()` (src/builders/SelectMenu.ts).
- `AttachmentBuilder` — `.setName(name)`, `.setFile(type, data)` where type is `'url' | 'path' | 'buffer' | ...`, `.setDescription()`, `.setSpoiler(bool)` (src/builders/Attachment.ts, line 41). Pass instances via `body.files`.
- `PollBuilder` — `.setQuestion({ text })`, `.setAnswers(...answers)`, `.setDuration(hours)`, `.allowMultiselect(bool)` (validates on `toJSON()` in v5 — a question and answers are required).
- `ButtonStyle` / `MessageFlags` are discord-api-types enums re-exported from 'seyfert'.

Body field shapes (src/common/types/write.ts): `content`, `embeds: (Embed | APIEmbed | InMessageEmbed)[]`, `components: TopLevelBuilders[]`, `files: (AttachmentBuilder | Attachment | RawFile)[]`, `poll: PollBuilder | RESTAPIPollCreate`, plus raw API fields like `flags` and `allowed_mentions`.

## Code Examples (verified)

Basic reply:

```ts
import { Command, Declare, type CommandContext } from 'seyfert';

@Declare({ name: 'helloworld', description: 'Sends a basic hello world message.' })
export default class HelloWorldCommand extends Command {
  async run(ctx: CommandContext) {
    return ctx.write({ content: 'Hello world 👋' });
  }
}
```

Defer then edit (good for slow work):

```ts
async run(ctx: CommandContext) {
  await ctx.deferReply();          // pass true for an ephemeral defer (interactions)
  // ...slow work...
  await ctx.editOrReply({ content: 'I did some stuff' });
}
```

Standalone message to a channel (no command response):

```ts
async run(ctx: CommandContext) {
  return ctx.client.messages.write(ctx.channelId, { content: 'Hello world 👋' });
}
```

Embed:

```ts
import { Embed, Command, type CommandContext } from 'seyfert';

const embed = new Embed()
  .setTitle('My Amazing Embed')
  .setDescription('Hello world 👋')
  .setColor('Blurple');               // EmbedColors name, hex string, or number
await ctx.write({ embeds: [embed] });
```

Components (button row + string select row):

```ts
import {
  ActionRow, Button, StringSelectMenu, StringSelectOption,
  Command, type CommandContext, ButtonStyle,
} from 'seyfert';

const button = new Button()
  .setCustomId('helloworld')
  .setLabel('Hello World')
  .setStyle(ButtonStyle.Primary);
const buttonRow = new ActionRow<Button>().addComponents(button);

const menu = new StringSelectMenu()
  .setCustomId('select-helloworld')
  .addOption(new StringSelectOption().setLabel('Hello').setValue('option_1'));
const menuRow = new ActionRow<StringSelectMenu>().addComponents(menu);

await ctx.write({ content: 'Hello world 👋', components: [buttonRow, menuRow] });
```

`ButtonStyle` can be in the same import as the rest; the doc splits it into a second `import` line, which is equivalent.

### Ephemeral reply (interactions)

Two ways. The simplest is `flags` on the body; for a deferred ephemeral response pass `true` to `deferReply`.

```ts
import { Command, type CommandContext, MessageFlags } from 'seyfert';

export default class SecretCommand extends Command {
  async run(ctx: CommandContext) {
    // Visible only to the invoking user.
    await ctx.write({ content: 'Only you can see this 🤫', flags: MessageFlags.Ephemeral });
  }
}
```

```ts
// Deferred + ephemeral: the eventual edit stays ephemeral too.
await ctx.deferReply(true);
await ctx.editOrReply({ content: 'Done, privately.' });
```

### Sending files / attachments

Use `AttachmentBuilder` and pass it in `files`. The `setFile` source supports `'url'`, `'path'`, and `'buffer'`.

```ts
import { AttachmentBuilder, Command, type CommandContext } from 'seyfert';

export default class FileCommand extends Command {
  async run(ctx: CommandContext) {
    const file = new AttachmentBuilder()
      .setName('greeting.txt')
      .setFile('buffer', Buffer.from('Hello from a file!'))
      .setDescription('A friendly greeting');

    await ctx.write({ content: 'Here is your file:', files: [file] });
  }
}
```

### Capturing the sent message back (response flag)

Interactions return `void` by default; pass the flag when you need the message object (e.g. to edit it later or read its id).

```ts
async run(ctx: CommandContext) {
  // withResponse: true -> resolves to a WebhookMessageStructure for interactions
  const msg = await ctx.write({ content: 'Working...' }, true);
  await msg.edit({ content: 'Done!' });
}
```

### Reply to a specific message + follow-ups

```ts
async run(ctx: CommandContext) {
  await ctx.write({ content: 'First response' });
  // A follow-up creates an additional message (a webhook message for interactions).
  await ctx.followup({ content: 'And a follow-up.' });

  // Replying to an arbitrary message (e.g. one you fetched / received in an event):
  // message.reply wires the message_reference for you.
  const channelMsg = await ctx.client.messages.write(ctx.channelId, { content: 'parent' });
  await channelMsg.reply({ content: 'a threaded reply', allowed_mentions: { replied_user: false } });
}
```

### Sending a poll

`PollBuilder` validates on serialization in v5 — a question and at least one answer are required or `toJSON()` throws.

```ts
import { PollBuilder, Command, type CommandContext } from 'seyfert';

export default class PollCommand extends Command {
  async run(ctx: CommandContext) {
    const poll = new PollBuilder()
      .setQuestion({ text: 'Deploy on Friday?' })
      .setAnswers({ text: 'Yes' }, { text: 'No' })
      .setDuration(24)        // hours
      .allowMultiselect(false);

    await ctx.write({ poll });
  }
}
```

### Localized reply (i18n)

`ctx.t` returns a `SeyfertLocale` proxy; resolve a value with `.get()` (uses the resolved locale for this context).

```ts
async run(ctx: CommandContext) {
  await ctx.write({ content: ctx.t.commands.hello.get() });
}
```

## Common patterns / gotchas

- After `deferReply()`, use `editOrReply()` (or `editResponse()`), NOT `write()` — calling `write` again on an interaction double-responds and throws. `editOrReply` is the safe default in reusable helpers: it edits if a response exists, else sends.
- Interaction responses return `void` unless you pass the response flag (`true`). Don't `await ctx.write(...).then(m => ...)` expecting a message — pass `true` to get a `WebhookMessageStructure`, or call `ctx.fetchResponse()` afterwards.
- `ActionRow` holds up to 5 buttons OR a single select menu, and cannot nest another `ActionRow`. Use a separate row per select menu. A message accepts up to 5 action rows.
- Ephemeral is set via `flags: MessageFlags.Ephemeral` on the body, or `deferReply(true)` for a deferred ephemeral flow. There is no `ephemeral: true` body shorthand.
- For a message to an arbitrary channel without responding to the command, use `ctx.client.messages.write(channelId, body)` — it returns a real `MessageStructure` you can later `.edit()` / `.delete()` / `.reply()`.
- `message.reply()` auto-fills `message_reference`; `message.write()` sends to the same channel without a reply reference. Set `allowed_mentions: { replied_user: false }` to reply without pinging.
- Builders validate on `toJSON()` in v5: `PollBuilder` needs a question + answers, `Modal` needs a title, invalid hex colors throw. Build them fully before sending.
- All builders, `ButtonStyle`, and `MessageFlags` import from the `seyfert` root — no deep `seyfert/lib/...` import needed here.

## Doc vs Source Corrections

- None in the upstream MDX — every method, builder, and signature in the doc matches the current source (verified `helloworld` examples, `editOrReply`, `client.messages.write`, `Embed`, components).
- Enrichment beyond the doc (all source-confirmed): the `withResponse`/response-flag behavior on `write`/`deferReply`/`editOrReply`/`followup`; `deferReply(true)` ephemeral defer; `flags: MessageFlags.Ephemeral`; `AttachmentBuilder` files; `PollBuilder`; `message.reply`/`followup`.
- v5 note (not a doc error, but relevant if copying component-handler code): `ComponentContext.editResponse` was REMOVED in v5 — use `ctx.editOrReply` in component contexts. `CommandContext.editResponse` and `ModalContext.editResponse` still exist.

## Source Anchors

- src/commands/applications/chatcontext.ts (write 88, modal 99, deferReply 107, editResponse 126, deleteResponse 140, editOrReply 145, followup 156, fetchResponse 163)
- src/common/shorters/messages.ts (write 16, edit 42, crosspost 67, delete 78, fetch 100, purge 116, endPoll 131, getAnswerVoters 139, list 148)
- src/structures/Message.ts (fetch 160, reply 164, edit 176, write 180, delete 184, crosspost 188, endPoll 192, getAnswerVoters 196)
- src/common/types/write.ts (ResolverProps / MessageCreateBodyRequest / InteractionCreateBodyRequest body shapes)
- src/builders/Attachment.ts (AttachmentBuilder 41, setFile 86), src/builders/Embed.ts, src/builders/ActionRow.ts, src/builders/Button.ts, src/builders/SelectMenu.ts, src/builders/index.ts

## Agent Guidance

- Prefer `ctx.write` for the first response; switch to `ctx.editOrReply` whenever the handler may run before or after a response already exists (e.g. after `deferReply`, or in shared helpers).
- Need the resulting message object from an interaction? Pass the response flag (`ctx.write(body, true)`) — without it you get `void`.
- For ephemeral responses use `flags: MessageFlags.Ephemeral` (or `deferReply(true)`). No `ephemeral: true` field exists.
- Send files with `AttachmentBuilder().setFile('buffer' | 'path' | 'url', data)` passed in `files: [...]`.
- For channel sends decoupled from the command, use `ctx.client.messages.write(channelId, body)` → returns a `MessageStructure`.
- In component handlers, do not reach for `editResponse` — it was removed from `ComponentContext` in v5; use `editOrReply`/`update`.
