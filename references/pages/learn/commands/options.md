# Command Options

Original source URL: https://seyfert-web-git-seyfert-v5-tiramisulabs.vercel.app/docs/learn/commands/options
Coverage reference: commands.md
Verification status: Source-verified against ./src (the authoritative Seyfert source), cross-checked vs v5 changelog

## Page Summary

Slash command options are declared with the `@Options(...)` decorator using one factory per option type (`createStringOption`, `createIntegerOption`, etc.). Every option requires a `description`; all options accept `required`, a `value()` transform callback, and localization fields. Choiceable options (String/Integer/Number) additionally accept `choices` (typed to the option's value type) and `autocomplete`; channel options accept `channel_types`. Choice and channel-type arrays are `readonly`-friendly, so `as const` is supported and recommended to preserve literal inference into `CommandContext<typeof options>`.

v5 deltas vs v4: option record keys are enforced lowercase at COMPILE time (`LowercaseOptionsRecord`); autocomplete `choices` values are typed to the option type (numeric options reject string values and vice-versa); `Options(...)` gained a `(new () => SubCommand)[]` overload alongside the lowercase record.

## Key APIs (verified)

All factories are root exports from `'seyfert'` and live in `src/commands/applications/options.ts`. Each returns `{ ...data, type: ApplicationCommandOptionType.* } as const`.

- `createStringOption(data)` — `SeyfertStringOption`: `description` (required), `required?`, `choices?` (readonly `SeyfertChoice<string>[]`), `value?`, `autocomplete?`, `onAutocompleteError?`, `min_length?`, `max_length?`, localization fields. (options.ts:98-107, 146)
- `createIntegerOption(data)` — `SeyfertIntegerOption`: adds `min_value?`/`max_value?`, `choices?` of `SeyfertChoice<number>`, `autocomplete?`. (options.ts:108-117, 154)
- `createNumberOption(data)` — same shape as integer (decimals allowed). (options.ts:118-127, 162)
- `createBooleanOption(data)` — `SeyfertBooleanOption` (basic: no choices). (options.ts:128, 178)
- `createUserOption(data)` — basic. (options.ts:129, 184)
- `createRoleOption(data)` — basic. (options.ts:142, 188)
- `createMentionableOption(data)` — basic. (options.ts:143, 192)
- `createChannelOption(data)` — `SeyfertChannelOption`: `channel_types?` is a readonly array of `ChannelType` values; resolved value narrows via `SeyfertChannelMap`. (options.ts:130-141, 170)
- `createAttachmentOption(data)` — basic. (options.ts:144, 199)
- `Options(record | SubCommand[])` decorator — root export, two overloads:
  - `Options(options: (new () => SubCommand)[])` — list of subcommand classes. (decorators.ts:161)
  - `Options<const T extends OptionsRecord>(options: LowercaseOptionsRecord<T>)` — lowercase-keyed record. (decorators.ts:162)

Shared option fields (`SeyfertBasicOption`, options.ts:21 / `SeyfertBaseChoiceableOption`, options.ts:37):
- `description: string` — required for EVERY option (the only mandatory field).
- `required?: boolean` — when `true`, the option is non-optional in `ctx.options`; otherwise `T | undefined`.
- `description_localizations?`, `name_localizations?`, and `locales?: { name?; description? }` (keyed by `FlatObjectKeys<DefaultLocale>`) for i18n.
- `value?(data, ok, fail)` transform — `data` is `{ context: CommandContext; value }`. Call `ok(transformed)` to set the resolved value, or `fail(message?)` to reject (routes to `onOptionsError`). `data.value` is the raw resolved type: `string` for string, `number` for int/number, the narrowed `SeyfertChannelMap[type]` for channels, or the choice value union when `choices` is set `as const`. (options.ts:23-27, 77-96)

`SeyfertChoice<T extends string | number>` (options.ts:55-64): `{ readonly name; readonly value: T; name_localizations?; locales? }`, or a raw `APIApplicationCommandOptionChoice<T>` plus `locales?`. `ChoiceableTypes` = String | Integer | Number only — boolean/user/role/mentionable/channel/attachment have NO choices.

Autocomplete (chat.ts:63-69):
- `autocomplete?: AutocompleteCallback<ValueType>` = `(interaction: AutocompleteInteraction<boolean, ValueType>) => any`. `ValueType` is `string` for string options, `number` for integer/number options.
- `onAutocompleteError?: (interaction, error: unknown) => any`.
- `interaction.getInput()` returns the focused value as a string (`''` if none). (Interaction.ts:461-463)
- `interaction.respond(choices)` sends autocomplete results; `choices` values must match the option's `ValueType`. (Interaction.ts:465-467)
- An option cannot meaningfully use both `choices` and `autocomplete` (Discord constraint).

`ReturnOptionsTypes` (chat.ts) maps each option type to its resolved value:
- String -> `string`; Integer/Number -> `number`; Boolean -> `boolean`
- User -> `InteractionGuildMemberStructure | UserStructure`
- Channel -> `AllChannels` (narrowed by `channel_types`)
- Role -> `GuildRoleStructure`
- Mentionable -> `GuildRoleStructure | InteractionGuildMemberStructure | GuildMemberStructure | UserStructure`
- Attachment -> `Attachment`

`OnOptionsReturnObject` (shared.ts:119-130) — the metadata passed to `onOptionsError(ctx, metadata)`: a record keyed by option name of `{ failed: false; value }` or `{ failed: true; value: string; parseError? }`. `parseError` (text/prefix commands only) is a `MessageCommandOptionErrors` tuple like `['OPTION_REQUIRED']`, `['STRING_MAX_LENGTH', max]`, `['NUMBER_INVALID_CHOICE', choices]`, etc. (shared.ts:106-117)

## Code Examples (verified)

String option with choices, autocomplete, and length limits:

```ts
import { Options, createStringOption, Command } from 'seyfert';

@Options({
    normal: createStringOption({ description: 'A plain string' }),

    // Fixed choices — `as const` preserves the literal value union in ctx.options
    choices: createStringOption({
        description: 'Pick one',
        choices: [
            { name: 'The best library', value: 'seyfert' },
            { name: 'An odd stuff', value: 'meowdb' },
        ] as const,
    }),

    // Autocomplete (string option -> respond values must be strings)
    autocomplete: createStringOption({
        description: 'Type to search',
        autocomplete: interaction => {
            const select = ['bugs', 'actions', 'random'];
            const focus = interaction.getInput();
            return interaction.respond(
                select
                    .filter(ch => ch.includes(focus))
                    .map(ch => ({ name: ch, value: ch })),
            );
        },
    }),

    // Character limits
    limitlength: createStringOption({ description: 'len', max_length: 500, min_length: 200 }),
})
class Ping extends Command {}
```

Typed access via `CommandContext<typeof options>` (required vs optional inference):

```ts
import { Options, createStringOption, createUserOption, Command, type CommandContext } from 'seyfert';

const options = {
    best: createStringOption({
        description: 'Pick',
        choices: [
            { name: 'The best library', value: 'seyfert' },
            { name: 'An odd stuff', value: 'meowdb' },
        ] as const,
    }),
    target: createUserOption({ description: 'Who', required: true }),
    note: createStringOption({ description: 'Optional note' }),
};

@Options(options)
class Ping extends Command {
    async run(ctx: CommandContext<typeof options>) {
        ctx.options.best;   // 'seyfert' | 'meowdb' (literal union from `as const`)
        ctx.options.target; // UserStructure | InteractionGuildMemberStructure  (required -> non-optional)
        ctx.options.note;   // string | undefined  (not required)
    }
}
```

Integer option with numeric choices, min/max, and numeric autocomplete:

```ts
import { Options, createIntegerOption, Command } from 'seyfert';

@Options({
    // choices values are numbers; respond() rejects string values at compile time
    difficulty: createIntegerOption({
        description: 'Difficulty',
        choices: [
            { name: 'Easy', value: 1 },
            { name: 'Hard', value: 3 },
        ] as const,
    }),
    page: createIntegerOption({ description: 'Page', min_value: 1, max_value: 100 }),
    pick: createIntegerOption({
        description: 'Pick a number',
        autocomplete: interaction => {
            const focus = interaction.getInput(); // always a string
            const ids = ['10', '20', '30'];
            return interaction.respond(
                ids
                    .filter(id => id.includes(focus))
                    .map(id => ({ name: `#${id}`, value: Number(id) })), // value: number
            );
        },
        onAutocompleteError: (interaction, error) => {
            return interaction.respond([]); // graceful fallback on failure
        },
    }),
})
class Roll extends Command {}
```

Channel option restricted by type (narrows the resolved channel in `ctx.options`):

```ts
import { Options, createChannelOption, Command, ChannelType, type CommandContext } from 'seyfert';

const options = {
    channel: createChannelOption({ description: 'Any channel' }),
    voice: createChannelOption({
        description: 'Voice only',
        channel_types: [ChannelType.GuildVoice],
    }),
};

@Options(options)
class Move extends Command {
    async run(ctx: CommandContext<typeof options>) {
        ctx.options.channel; // AllChannels | undefined
        ctx.options.voice;   // narrowed voice-channel type | undefined
    }
}
```

`value()` transform with reject + `onOptionsError` handler:

```ts
import { Options, createIntegerOption, Command, type CommandContext } from 'seyfert';
import type { OnOptionsReturnObject } from 'seyfert';

const options = {
    amount: createIntegerOption({
        description: 'Positive number, stored doubled',
        // data.value is `number` here
        value: ({ value }, ok, fail) => {
            if (value <= 0) return fail('Must be positive');
            return ok(value * 2); // ctx.options.amount becomes the doubled value
        },
    }),
};

@Options(options)
class Give extends Command {
    async run(ctx: CommandContext<typeof options>) {
        await ctx.write({ content: `Stored: ${ctx.options.amount}` });
    }

    // Invoked when any value()/parse rejects; metadata describes each option
    async onOptionsError(ctx: CommandContext, metadata: OnOptionsReturnObject) {
        const failed = Object.entries(metadata)
            .filter(([, v]) => v.failed)
            .map(([name, v]) => `${name}: ${(v as { value: string }).value}`);
        await ctx.write({ content: `Invalid options:\n${failed.join('\n')}`, flags: 64 });
    }
}
```

Localized option (name/description pulled from lang keys):

```ts
import { Options, createStringOption, Command } from 'seyfert';

@Options({
    query: createStringOption({
        description: 'Search query', // fallback when no locale resolved
        required: true,
        locales: {
            name: 'commands.search.options.query.name',
            description: 'commands.search.options.query.description',
        },
    }),
})
class Search extends Command {}
```

The remaining factories (`createNumberOption`, `createBooleanOption`, `createUserOption`, `createRoleOption`, `createMentionableOption`, `createAttachmentOption`) all take `{ description, required?, value?, ...localization }` and import the same way from `'seyfert'`. `createNumberOption` mirrors integer (choices/min_value/max_value/autocomplete) but allows decimals.

## Common patterns / gotchas

- LOWERCASE KEYS ARE ENFORCED. `{ User: createStringOption(...) }` is a compile error in v5 (`LowercaseOptionsRecord`). Use `{ user: ... }`. This matches Discord's runtime rejection but catches it in your editor.
- `as const` on `choices` (and `channel_types`) is what gives you the literal value union in `ctx.options`. Without it you get the wide `string`/`number`/`ChannelType` type. The MDX omits `as const` on the integer-choices example — add it for the same inference the string example gets.
- Autocomplete value types are bound to the option type in v5: a numeric option's `respond([...])` rejects `{ value: 'four' }`; map to a `number`. `getInput()` is ALWAYS a string regardless of option type (it's the raw focused text) — convert with `Number(...)` when filtering numeric suggestions.
- `choices` and `autocomplete` are mutually exclusive per Discord — set one, not both, on a single option.
- Only String/Integer/Number support `choices`/`autocomplete`; only String supports `min_length`/`max_length`; only Integer/Number support `min_value`/`max_value`.
- `value()`'s `data.value` is the RAW resolved value before your transform. `ok(x)` replaces the stored value with `x` (its type flows into `ctx.options`); `fail('msg')` rejects and dispatches `onOptionsError(ctx, metadata)`. For prefix/text commands, `metadata[name].parseError` carries a typed `MessageCommandOptionErrors` tuple; slash commands populate `value`/`failed` only.
- `required: true` makes the option non-optional in `CommandContext<typeof options>`; leaving it off yields `T | undefined`. Define options as a named `const` and pass `CommandContext<typeof options>` to `run` to get full typing.
- Subcommand-based commands use the array overload: `@Options([SubA, SubB])` where each is a `SubCommand` class — not a record.

## Doc vs Source Corrections

- The MDX integer/number choice examples omit `as const` while the string example includes it. Source types `choices` as `readonly SeyfertChoice<...>[]` in every choiceable factory (options.ts:148/156/164), so `as const` is needed on ALL choice arrays (string, integer, number) for literal value inference. Docs are inconsistent, not wrong.
- The MDX never shows the reject path. Source confirms the third `value()` arg is `StopFunction = (error?: string | null) => void` (shared.ts:20); a rejection routes to `command.onOptionsError(context, metadata: OnOptionsReturnObject)` (chat.ts:354, handle.ts:425/755).
- `channel_types` accepts a `readonly` array (options.ts:140), so `as const` works there too.
- v5: autocomplete `respond()` choice values are now typed to the option's `ValueType` (changelog "Autocomplete choice types") — numeric options reject string values. The pre-v5 docs predate this constraint.

## Source Anchors

- `src/commands/applications/options.ts` — all `create*Option` factories, `SeyfertBasicOption`/`SeyfertBaseChoiceableOption`, `SeyfertChoice`, `ValueCallback`, per-type interfaces.
- `src/commands/applications/chat.ts` — `ReturnOptionsTypes`, `AutocompleteCallback`/`OnAutocompleteErrorCallback`, `OptionsRecord`, `ContextOptions`, `onOptionsError` signature.
- `src/commands/applications/shared.ts` — `OKFunction`, `StopFunction`, `SeyfertChannelMap`, `OnOptionsReturnObject`, `MessageCommandOptionErrors`, `DefaultLocale`.
- `src/commands/decorators.ts` — `Options` overloads (161-163), `LowercaseOptionsRecord` (135-137).
- `src/structures/Interaction.ts` — `AutocompleteInteraction.getInput` (461), `respond` (465).
- `src/commands/optionresolver.ts` — focused-value resolution and per-type value parsing.

## Agent Guidance

- Always supply `description` — the only mandatory field on every option.
- Use lowercase option keys; `Options` enforces `Lowercase` keys at the type level (decorators.ts:135).
- Add `as const` to every `choices` array (and `channel_types`) for literal types in `ctx.options`.
- Mark options `required: true` to make them non-optional in `CommandContext<typeof options>`.
- Choices vs autocomplete are mutually exclusive; only String/Integer/Number support either, plus min/max.
- In `value()`, call `ok(x)` to transform/store and `fail('msg')` to reject; handle rejections in `onOptionsError(ctx, metadata)`.
- Autocomplete `respond()` values must match the option type (string vs number); `getInput()` returns a string.
- All factories and `Options`/`Command`/`CommandContext`/`ChannelType` import from the root `'seyfert'` — no deep import needed.
