# Embeds, Formatting & Attachments

Original source URL: https://seyfert-web-git-seyfert-v5-tiramisulabs.vercel.app/docs/recipes/embeds-and-formatting

Coverage reference: builders.md (+ i18n-cache-recipes.md)

Verification status: Source-verified (v5, the authoritative Seyfert source)

## Page Summary

Covers the `Embed` builder, the `Formatter` utility (Discord markdown, mentions, timestamps,
links, OAuth2 URLs), the `AttachmentBuilder` (files from buffer / URL / path), and how to reference
attachments inside embeds via the `attachment://` protocol. All of `Embed`, `Formatter`,
`AttachmentBuilder`, `TimestampStyle`, `HeadingLevel`, `EmbedColors`, plus the types
`ChannelLink` / `MessageLink` / `Timestamp` / `OAuth2URLOptions`, are exported from the root
`'seyfert'` package (`src/index.ts:17,26-41`).

## Key APIs (verified)

Embed — `src/builders/Embed.ts` (class, chainable setters return `this`):
- `new Embed(data: Partial<APIEmbed> = {})` — shallow-clones `data`; auto-initializes `fields: []`.
- `.setTitle(title?)`, `.setDescription(desc?)`, `.setURL(url?)`.
- `.setColor(color?: ColorResolvable)` — resolves via `resolveColor`.
- `.setAuthor(author?)` — camelCase keys (`{ name, iconUrl, url }`), converted to snake_case internally.
- `.setFooter(footer?)` — `{ text, iconUrl }` (camelCase; `proxy_icon_url` omitted from the type).
- `.addFields(...fields: RestOrArray<APIEmbedField>)` — accepts spread OR a single array; appends.
- `.setFields(fields?: APIEmbedField[])` — replaces all fields (`undefined` → `[]`).
- `.setImage(url?)`, `.setThumbnail(url?)` — store `{ url }` (or `undefined` when no url).
- `.setTimestamp(time: string | number | Date = Date.now())` — stores an ISO string.
- `.toJSON(): APIEmbed` — the serializer is `toJSON()`, NOT `json()` (despite a stale jsdoc `embed.json()`).

ColorResolvable / colors — `src/common/types/resolvables.ts`, `src/common/it/utils.ts:41`, `constants.ts`:
- `type ColorResolvable = `#${string}` | number | keyof typeof EmbedColors | 'Random' | [number, number, number]`.
- Named colors come from `EmbedColors` enum (e.g. `Blue=0x3498db`, `Blurple=0x5865f2`, `Green`, `Red`, ...).
- `'Random'` → random color. A non-`#` unknown string → `EmbedColors.Default` (0x000000).
- A number that is non-integer OR negative → throws `SeyfertError('INTERNAL_ERROR')`.
- A malformed `#hex` (non hex chars, e.g. `'#zzz'`) → throws `SeyfertError('INTERNAL_ERROR')` (v5; was silent `NaN`).
- An RGB tuple of length ≥ 3 → `(r<<16)|(g<<8)|b`; shorter array → `EmbedColors.Default`.

Formatter — `src/common/it/formatter.ts:110` (a const OBJECT of methods, NOT a `class`; no `new`):
- `codeBlock(content: string, language = 'txt')` — CONTENT IS THE FIRST ARG, language second.
- `inlineCode(content)`→`` `x` ``, `bold`→`**x**`, `italic`→`*x*`, `underline`→`__x__`,
  `strikeThrough`→`~~x~~`, `spoiler`→`||x||`, `quote`→`` > ${string}``, `blockQuote`→`` >>> ${string}``.
- `hyperlink(content, url)`→`[content](url)`, `header(content, level = HeadingLevel.H1)`,
  `list(items: string[], ordered = false)`.
- `userMention(id)`→`<@id>`, `roleMention(id)`→`<@&id>`, `channelMention(id)`→`<#id>`,
  `emojiMention(emojiId, name: string | null, animated = false)`→`<a?:name:emojiId>` (ID first, name second; `null` name → `_`).
- `timestamp(timestamp: Date | number, style = TimestampStyle.RelativeTime)` — accepts Date OR a
  millisecond number (NOT a string); returns typed `<t:${seconds}:${style}>`. Default is RELATIVE ('R').
- `messageLink(guildId, channelId, messageId): MessageLink`, `channelLink(channelId, guildId = '@me'): ChannelLink`.
- `generateOAuth2URL(appId, { scopes, permissions?, disableGuildSelect? = false }): string`.

Enums — `src/common/it/formatter.ts`: `TimestampStyle` (ShortTime `t`, MediumTime `T`, ShortDate `d`,
LongDate `D`, LongDateShortTime `f`, FullDateShortTime `F`, ShortDateShortTime `s`,
ShortDateMediumTime `S`, RelativeTime `R`); `HeadingLevel` (H1=1, H2=2, H3=3).

AttachmentBuilder — `src/builders/Attachment.ts:41`:
- Default constructor picks a random `<base64url>.jpg` filename if you set none.
- `.setName(name)` (sets `filename`), `.setDescription(desc)`, `.setSpoiler(bool)` (toggles `SPOILER_` prefix),
  `get spoiler` (filename starts with `SPOILER_`).
- `.setFile<T>(type: 'url' | 'buffer' | 'path', data)` — `buffer` accepts
  `ReadableStream | Buffer | ArrayBuffer | Uint8Array | Uint8ClampedArray` (and async-iterable buffers).
- Resolution (`resolveAttachmentData`) throws `SeyfertError('INVALID_ATTACHMENT_TYPE')` for a non-http URL,
  a non-OK HTTP response, a missing path, an unresolvable buffer, or an unknown type.

## Code Examples (verified)

Embed command (root imports):
```ts
import { Embed, Command, Declare, type CommandContext } from 'seyfert';

@Declare({ name: 'embed', description: 'Send a rich embed' })
export default class EmbedCommand extends Command {
    async run(ctx: CommandContext) {
        const embed = new Embed()
            .setTitle('Server Info')
            .setURL('https://discord.gg/example')
            .setDescription('Here is some information about this server.')
            .setColor('Blurple')
            .setAuthor({ name: ctx.author.username, iconUrl: ctx.author.avatarURL() })
            .setFooter({ text: 'Powered by Seyfert' })
            .addFields(
                { name: 'Members', value: '1,234', inline: true },
                { name: 'Channels', value: '42', inline: true },
            )
            .setImage('https://example.com/banner.png')
            .setTimestamp();

        await ctx.write({ embeds: [embed] });
    }
}
```

Embed from an i18n partial (constructor takes `Partial<APIEmbed>`):
```ts
// ctx.t.<path>.get() yields a plain object you can feed straight into Embed.
const embed = new Embed(ctx.t.commands.help.embed.get(ctx.author.locale)).setColor('Blurple');
```

Color forms (mind the throwing cases):
```ts
new Embed().setColor(0xff0000);          // numeric hex
new Embed().setColor('#ff0000');         // hex string (valid hex chars only)
new Embed().setColor('Blue');            // EmbedColors key
new Embed().setColor([255, 0, 0]);       // RGB tuple
new Embed().setColor('Random');          // random color
// new Embed().setColor('#zzz');         // THROWS INTERNAL_ERROR (malformed hex)
// new Embed().setColor(-1);             // THROWS INTERNAL_ERROR (negative)
```

Formatter — note `codeBlock` arg order and the root import of `TimestampStyle`:
```ts
import { Formatter, TimestampStyle } from 'seyfert';

Formatter.bold('text');                       // **text**
Formatter.italic('text');                     // *text*
Formatter.underline('text');                  // __text__
Formatter.strikeThrough('text');              // ~~text~~
Formatter.inlineCode('text');                 // `text`
Formatter.codeBlock('const x = 1', 'ts');     // ```ts\nconst x = 1\n```  (content FIRST)
Formatter.spoiler('text');                    // ||text||
Formatter.quote('text');                      // > text
Formatter.blockQuote('text');                 // >>> text
Formatter.hyperlink('Click here', 'https://example.com');
Formatter.header('Title', 2);                 // ## Title
Formatter.list(['one', 'two'], true);         // 1. one\n2. two

Formatter.userMention('userId');              // <@userId>
Formatter.channelMention('channelId');        // <#channelId>
Formatter.roleMention('roleId');              // <@&roleId>
Formatter.emojiMention('123', 'wave');        // <:wave:123>   (id first, then name)
Formatter.emojiMention('123', 'spin', true);  // <a:spin:123>  (animated)

Formatter.timestamp(new Date());                          // defaults to relative ('R')
Formatter.timestamp(Date.now(), TimestampStyle.LongDate); // ms number + explicit style
```

Attachment from a buffer:
```ts
import { AttachmentBuilder } from 'seyfert';

const attachment = new AttachmentBuilder()
    .setName('hello.txt')
    .setFile('buffer', Buffer.from('Hello, World!'));

await ctx.write({ content: 'Here is the file:', files: [attachment] });
```

Attachment from a URL + referencing it inside an embed image:
```ts
const attachment = new AttachmentBuilder()
    .setName('chart.png')
    .setFile('url', 'https://example.com/chart.png');

const embed = new Embed()
    .setTitle('Statistics')
    .setImage('attachment://chart.png'); // must match the attachment filename

await ctx.write({ embeds: [embed], files: [attachment] });
```

## Recipes / worked examples

### Rich userinfo embed combining Formatter helpers
```ts
import { Embed, Formatter, Command, Declare, type CommandContext } from 'seyfert';

@Declare({ name: 'userinfo', description: 'Show info about you' })
export default class UserInfo extends Command {
    async run(ctx: CommandContext) {
        const user = ctx.author;
        const embed = new Embed()
            .setTitle(Formatter.bold(user.username))
            .setThumbnail(user.avatarURL())
            .setDescription(
                Formatter.blockQuote(user.globalName ?? 'No display name set'),
            )
            .addFields(
                { name: 'Mention', value: Formatter.userMention(user.id), inline: true },
                { name: 'ID', value: Formatter.inlineCode(user.id), inline: true },
                { name: 'Bot', value: user.bot ? 'Yes' : 'No', inline: true },
            )
            .setColor('Blurple');

        await ctx.write({ embeds: [embed] });
    }
}
```

### Attachment from a local file path
```ts
import { AttachmentBuilder, SeyfertError } from 'seyfert';

try {
    const file = new AttachmentBuilder()
        .setName('report.pdf')
        .setDescription('Monthly report')
        .setFile('path', './assets/report.pdf'); // resolved + read at send time

    await ctx.write({ files: [file] });
} catch (error) {
    // Missing file / unreadable path throws INVALID_ATTACHMENT_TYPE.
    if (SeyfertError.is(error, 'INVALID_ATTACHMENT_TYPE')) {
        await ctx.write({ content: 'That file no longer exists.' });
    } else throw error;
}
```

### Paginated embeds with buttons (collector flow)
```ts
import {
    Embed, ActionRow, Button, Command, Declare, type CommandContext,
} from 'seyfert';
import { ButtonStyle } from 'seyfert';

@Declare({ name: 'pages', description: 'Browse paginated embeds' })
export default class Pages extends Command {
    async run(ctx: CommandContext) {
        const pages = [
            new Embed().setTitle('Page 1').setColor('Blue'),
            new Embed().setTitle('Page 2').setColor('Green'),
            new Embed().setTitle('Page 3').setColor('Red'),
        ];
        let index = 0;

        const row = new ActionRow<Button>().addComponents(
            new Button().setCustomId('prev').setLabel('Prev').setStyle(ButtonStyle.Secondary),
            new Button().setCustomId('next').setLabel('Next').setStyle(ButtonStyle.Primary),
        );

        // fetchReply=true returns the sent Message so we can attach a collector.
        const message = await ctx.write({ embeds: [pages[index]], components: [row] }, true);

        const collector = message.createComponentCollector({
            idle: 60_000,
            filter: i => i.user.id === ctx.author.id,
        });

        const show = async (i: any) => {
            await i.update({ embeds: [pages[index]] });
        };
        collector.run('prev', async i => {
            index = (index - 1 + pages.length) % pages.length;
            await show(i);
        });
        collector.run('next', async i => {
            index = (index + 1) % pages.length;
            await show(i);
        });
    }
}
```

### Build an OAuth2 invite link
```ts
import { Formatter, OAuth2Scopes } from 'seyfert';

const url = Formatter.generateOAuth2URL(ctx.client.applicationId, {
    scopes: [OAuth2Scopes.Bot, OAuth2Scopes.ApplicationsCommands],
    permissions: 'Administrator', // BitFieldResolvable (single value); optional
});
await ctx.write({ content: Formatter.hyperlink('Invite me', url) });
```

## Common patterns / gotchas

- `codeBlock(content, language)` — content FIRST. The single most common mistake copied from the docs.
- `Formatter` is a plain object literal, not a class; call methods directly, never `new Formatter()`.
- `setColor` is partly forgiving and partly strict: a plain unknown string silently becomes black
  (`EmbedColors.Default`), but a malformed `#hex`, a negative number, or a non-integer number THROWS.
  Validate/normalize user-supplied colors before passing them.
- `Embed`'s serializer is `toJSON()`. You usually never call it yourself — pass the builder straight
  into `ctx.write({ embeds: [embed] })`. Message helpers also accept any `toJSON()`-bearing object now.
- To show an image you already upload, set `.setImage('attachment://<filename>')` where `<filename>`
  exactly matches the attachment's `setName(...)`, and include the builder in `files: [...]`.
- URL/path attachment resolution performs real I/O (fetch/fs) at send time and throws
  `INVALID_ATTACHMENT_TYPE` on failure — wrap untrusted sources in try/catch and narrow with
  `SeyfertError.is(err, 'INVALID_ATTACHMENT_TYPE')`.
- `addFields` accepts either a spread of field objects or a single array (`RestOrArray`); `setFields`
  replaces them. Discord caps at 25 fields — Seyfert does not pad/trim for you.
- `timestamp(...)` defaults to relative time in v5; pass a `TimestampStyle` for an absolute format. It
  takes a `Date` or millisecond `number` — never an ISO string.
- For `ctx.write(body, fetchReply)`: the return is `void` unless you pass `true`, then you get the
  `Message` (needed to attach a component collector). Same rule for `editOrReply`.

## Doc vs Source Corrections

- Upstream MDX: `Formatter.codeBlock('ts', 'const x = 1')` → src signature is
  `codeBlock(content, language='txt')`, so CONTENT is first: `Formatter.codeBlock('const x = 1', 'ts')`
  (`src/common/it/formatter.ts:117`).
- Upstream MDX imports `TimestampStyle` from `'seyfert/lib/common'` → it is re-exported from the root
  `'seyfert'` package (`src/index.ts:40`). Prefer `import { TimestampStyle } from 'seyfert'`.
  Same for `HeadingLevel`, `Formatter`, `EmbedColors` (`src/index.ts:26-28`).
- Upstream MDX calls `Formatter` a "class" → it is a plain const object literal, no `new`
  (`src/common/it/formatter.ts:110`).
- `Formatter.timestamp` accepts only `Date | number` (ms), not a string (`formatter.ts:244`), and
  defaults to `RelativeTime` ('R') in v5.
- `Embed` serializer is `toJSON()` not `json()` (stale jsdoc in src shows `embed.json()`, which does
  not exist) (`src/builders/Embed.ts:165`).
- v5 hardening: `resolveColor('#zzz')` and negative/non-integer numbers now THROW `INTERNAL_ERROR`
  (was silent `NaN`/Default) (`src/common/it/utils.ts:44-62`).
- Note: `snowflakeToTimestamp(id)` exists (`src/common/it/utils.ts:324`, returns unix-ms `number` in
  v5) but is NOT in the root `'seyfert'` named exports — use a deep import if you need it.

## Source Anchors

- `src/common/it/constants.ts` (`EmbedColors`)
- `src/structures/Message.ts:76` (`createComponentCollector`)
- `src/components/handler.ts:82` (collector `run`/`stop`/`waitFor`)
