# Locales Usage

Original source URL: https://seyfert-web-git-seyfert-v5-tiramisulabs.vercel.app/docs/learn/i18n/usage
Coverage reference: i18n-cache-recipes.md
Verification status: Source-verified (the authoritative Seyfert source)

## Page Summary

Covers how to localize Discord command metadata in Seyfert and (as the practical companion) how to read translated strings at runtime. Two families of decorators exist:

- Dot-path family — `@LocalesT`, `@GroupsT`, option `locales` object, choice `locales` string. These resolve `FlatObjectKeys<DefaultLocale>` dot-paths against your registered lang shape at command-load time and populate Discord's `name_localizations` / `description_localizations`.
- Raw-tuple family — `@Locales`, `@Groups`. These take explicit `[LocaleString, value][]` arrays (no lang shape needed). Use them when the localizations don't live in your lang files.

Runtime translation access is via `ctx.t.<path>.get(locale?)` and `client.t(locale)`, both returning the `SeyfertLocale` proxy. The runtime accessor type is kept as a named `SeyfertLocale` alias so inferred `ctx.t` typing references `DefaultLocale` instead of being baked to `{}` at build time.

## Key APIs (verified)

- `LocalesT(name?, description?)` — class decorator, `src/commands/decorators.ts:61`. Both args are `FlatObjectKeys<DefaultLocale>` (dot-paths). Sets `__t = { name, description }`. Exported from `'seyfert'`.
- `Locales({ name?, description? })` — class decorator, `src/commands/decorators.ts:47`. Args are RAW tuples `[language: LocaleString, value: string][]` (NOT dot-paths). Sets `name_localizations` / `description_localizations` directly. Use when localizations are inline, not in lang files.
- `GroupsT(groups)` — class decorator, `src/commands/decorators.ts:91`. `groups` is `Record<string, TranslatedGroupDefinition>` (dot-path values). Sets `__tGroups` + builds `groupsAliases` from each entry's `aliases`.
- `Groups(groups)` — class decorator, `src/commands/decorators.ts:107`. `Record<string, LocalizedGroupDefinition>` (raw tuple values). Sets `groups` + `groupsAliases`.
- `TranslatedGroupDefinition` — `src/commands/decorators.ts:68`: `{ name?: FlatObjectKeys<DefaultLocale>; description?: FlatObjectKeys<DefaultLocale>; defaultDescription: string; aliases?: string[] }`. Only `defaultDescription` is required.
- `LocalizedGroupDefinition` — `src/commands/decorators.ts:75`: `{ name?: [LocaleString, string][]; description?: [LocaleString, string][]; defaultDescription: string; aliases?: string[] }`.
- `GroupDefinition` / `GroupDefinitions` — `src/commands/decorators.ts:82-83`: union + record types over the two definition shapes.
- `defineGroups(groups)` — `src/commands/decorators.ts:85`. Two overloads (`Localized` / `Translated`); returns the object unchanged with `const` inference. Pair with `@Group(groupsDef, key)` for a type-checked key.
- Option `locales` property — `src/commands/applications/options.ts:31` & `:49`: `{ name?: FlatObjectKeys<DefaultLocale>; description?: FlatObjectKeys<DefaultLocale> }`, passed to `create*Option` factories. Omit a key to skip localizing it.
- Choice `locales` property — `src/commands/applications/options.ts:60-63`: a single `FlatObjectKeys<DefaultLocale>` STRING (not an object) localizing the choice `name`. Lives on each `SeyfertChoice`.
- `ParseLocales<T> = T` — identity type, `src/langs/router.ts:62`. Register your lang shape: `interface SeyfertRegistry { langs: ParseLocales<typeof lang> }`. Exported from `'seyfert'`.
- `DefaultLocale` — `src/commands/applications/shared.ts`: derived `SeyfertRegistry extends { langs: infer L } ? L : {}`. Falls back to `{}` (untyped paths) when no langs registered.
- `SeyfertLocale` — `src/langs/router.ts:60`: `__InternalParseLocale<DefaultLocale> & { get(locale?: string): DefaultLocale }`. String entries → `{ get(): string }`; array entries → `{ get(): T[] }`; function entries → `(...args) => { get(): ReturnType }`; nested objects → recurse + `get()`.
- `client.t(locale)` — `src/client/base.ts:1184` → `this.langs.get(locale)`. Returns the `SeyfertLocale` proxy bound to that locale.
- `ctx.t` getter — `src/commands/applications/chatcontext.ts:72` (mirrored in menu/entry/component/modal contexts). When `client.langs.preferGuildLocale` is true: `guildLocale ?? locale ?? defaultLang ?? 'en-US'`, else `locale ?? defaultLang ?? 'en-US'`.
- `LangsHandler` runtime fields/methods — `src/langs/handler.ts`: `defaultLang?`, `preferGuildLocale = false`, `aliases: [string, LocaleString[]][]`, `onReload?`, `getLocale(locale)` (alias resolution), `getKey(lang, message)` (dot-path; returns `undefined` unless the resolved value is a `string`), `get(userLocale)`, `reload(lang)` / `reloadAll(stopIfFail?)` (throw `RELOAD_NOT_SUPPORTED` in Cloudflare workers), `load(dir)`, `set(instances)`.
- Langs config — `ServicesOptions.langs` (`src/client/base.ts:1394`): `{ default?: string; preferGuildLocale?: boolean; aliases?: Record<string, LocaleString[]> }`, applied via `client.setServices({ langs })`. Note the config key is `default` (mapped onto `langs.defaultLang`).

## Code Examples (verified)

### 1. Define a lang file and register the shape

Lang files use a `default` export object (a single named object export is accepted as a fallback). Values can be strings, arrays, nested objects, or functions:

```ts
// src/languages/en-US.ts
export default {
  'my-command': { name: 'My Command', description: 'A demo command' },
  greeting: 'Hello!',
  welcome: (name: string) => `Welcome, ${name}!`,
  rules: ['Be nice', 'No spam'],
  options: { name: 'name', description: 'the name to use' },
};
```

Register the shape once (enables typed dot-paths AND typed `ctx.t`). Use ONE locale file as the canonical type source — typically your default:

```ts
import type { ParseLocales } from 'seyfert';
import type English from './languages/en-US';

declare module 'seyfert' {
  interface SeyfertRegistry {
    langs: ParseLocales<typeof English>;
  }
}
```

### 2. `@LocalesT` — localize command name/description by dot-path

```ts
import { Command, Declare, LocalesT } from 'seyfert';

@Declare({ name: 'my-command', description: 'A demo command' })
@LocalesT('my-command.name', 'my-command.description')
export default class MyCommand extends Command {}
```

### 3. `@Locales` — inline raw tuples (no lang shape needed)

Use when the localizations are not stored in your lang files:

```ts
import { Command, Declare, Locales } from 'seyfert';

@Declare({ name: 'ping', description: 'Pong!' })
@Locales({
  name: [['es-ES', 'ping'], ['fr', 'ping']],
  description: [['es-ES', '¡Pong!'], ['fr', 'Pong !']],
})
export default class PingCommand extends Command {}
```

### 4. `@GroupsT` — localize subcommand groups (only `defaultDescription` required)

```ts
import { Command, GroupsT } from 'seyfert';

@GroupsT({
  // key = the group name created on the parent command (via @Group on subcommands)
  supremacy: {
    defaultDescription: 'Ganyu Supremacy.', // required
    name: 'my-command.group.name',          // optional dot-path
    description: 'my-command.group.desc',    // optional dot-path
    aliases: ['sup'],                        // optional prefix-command aliases
  },
})
export default class SupremacyCommand extends Command {}
```

`defineGroups` + `@Group` for a reusable, key-checked definition:

```ts
import { Command, Declare, Group, GroupsT, defineGroups } from 'seyfert';

const groups = defineGroups({
  config: { defaultDescription: 'Configuration', name: 'cmd.config.name' },
});

@Declare({ name: 'admin', description: 'Admin tools' })
@GroupsT(groups)
@Group(groups, 'config') // 'config' is type-checked against the defineGroups keys
export default class AdminConfig extends Command {}
```

### 5. Per-option and per-choice localization

```ts
import { createStringOption } from 'seyfert';

const options = {
  // option keys MUST be lowercase (v5 compile-time rule)
  supremacy: createStringOption({
    description: 'Enter a supremacy name.',
    required: true,
    // option-level locales = OBJECT of dot-paths; omit name/description to skip
    locales: {
      name: 'my-command.options.name',
      description: 'my-command.options.description',
    },
    choices: [
      // choice-level locales = a single dot-path STRING (localizes the choice name)
      { name: 'Ganyu', value: 'ganyu', locales: 'my-command.choices.ganyu' },
      { name: 'Hu Tao', value: 'hutao', locales: 'my-command.choices.hutao' },
    ] as const, // choices/channel_types are readonly in v5 — use `as const`
  }),
};
```

### 6. Configure langs on the client

Langs config goes through `setServices`, NOT the `Client` constructor:

```ts
client.setServices({
  langs: {
    default: 'en-US',        // -> langs.defaultLang (fallback locale)
    preferGuildLocale: true, // ctx.t resolves guildLocale before the user's
    aliases: {
      // map extra Discord locales onto one of your lang files
      'en-US': ['en-GB'],
      'es-ES': ['es-419'],
    },
  },
});
```

### 7. Runtime access via `ctx.t`

```ts
// inside a command run(ctx)
const hi = ctx.t.greeting.get();              // string for the user's resolved locale
const welcome = ctx.t.welcome(ctx.author.username).get(); // function entry: call, then .get()
const rules = ctx.t.rules.get();              // array entry
const forced = ctx.t.greeting.get('es-ES');   // force a specific locale
const nested = ctx.t.options.name.get();      // nested object path

await ctx.write({ content: welcome });
```

### 8. Localized reply with a forced locale (e.g. logging in the guild's language)

```ts
import { type UsingClient } from 'seyfert';

function logLine(client: UsingClient, guildLocale: string, user: string) {
  // client.t(locale) returns the same SeyfertLocale proxy, bound to `locale`
  return client.t(guildLocale).welcome(user).get();
}
```

### 9. Hot-reload a lang file at runtime (non-worker)

```ts
// reload one locale, or all of them
await client.langs.reload('en-US');
await client.langs.reloadAll();
// throws SeyfertError('RELOAD_NOT_SUPPORTED') in Cloudflare workers
```

## Common patterns / gotchas

- Dot-path strings are typed as `FlatObjectKeys<DefaultLocale>`, but ONLY after you register `SeyfertRegistry.langs` via `ParseLocales`. Without registration `DefaultLocale` is `{}` and both the dot-paths and `ctx.t` are untyped — register first.
- Decorators/`locales` localize Discord command METADATA at load time. They never return runtime strings. For runtime strings use `ctx.t.<path>.get()` / `client.t(locale).<path>.get()`.
- `@LocalesT`/`@GroupsT`/option `locales`/choice `locales` = dot-paths into lang files. `@Locales`/`@Groups` = inline `[LocaleString, value][]` tuples. Don't mix the two shapes on one decorator.
- `getKey`/metadata resolution only yields a value when the resolved lang entry is a `string`. A path that lands on an object resolves to `undefined` and the localization is silently skipped (a warning is logged) — it is NOT thrown.
- `ctx.t.<path>.get(locale?)`: no arg uses the context's resolved user locale (baked into the proxy). On lookup failure the proxy silently retries with `defaultLang` (`LangRouter` try/catch in `router.ts:23-27`). For function-valued entries you MUST call the path first, then `.get()`: `ctx.t.msg(args).get()`.
- Option record keys must be lowercase (v5 compile-time). `choices` / `channel_types` are readonly → annotate with `as const`. Autocomplete choice `value`s are typed to the option's type (integer/number reject strings).
- `preferGuildLocale` only changes resolution when there's an interaction with a `guildLocale`. Prefix-command (message) contexts have no interaction locale, so they fall through to `defaultLang ?? 'en-US'`.
- Aliases let you serve one lang file to multiple Discord locales (e.g. `es-419` → `es-ES`). Configure via `langs.aliases`; resolved by `LangsHandler.getLocale`.

## Doc vs Source Corrections

- Docs/changelog show `new Client({ langs: { preferGuildLocale: true } })`. SOURCE: `BaseClientOptions` (`src/client/base.ts:1242`) has NO `langs` key. Langs config lives on `ServicesOptions.langs` (`:1394`) and is applied via `client.setServices({ langs: { default, preferGuildLocale, aliases } })`. Note the config key is `default`, not `defaultLang`.
- Docs only show `@GroupsT` with `defaultDescription` + `name` → src `TranslatedGroupDefinition` also supports `description` (dot-path) and `aliases: string[]` (`decorators.ts:68`).
- Docs omit the raw-tuple variants → src exports `@Locales` (`decorators.ts:47`) and `@Groups` (`decorators.ts:107`) taking `[LocaleString, value][]` instead of dot-paths.
- Docs don't mention choice localization → src supports a `locales` STRING dot-path on each choice (`options.ts:60`), distinct from the option-level `locales` OBJECT.
- Docs omit runtime access entirely → src exposes `ctx.t` / `client.t(locale)` returning the `SeyfertLocale` proxy with `.path.get(locale?)` (`router.ts`, context getters).
- Docs example uses a non-`export default` `MyCommand` class. Harmless; commands are typically `export default`.

## Source Anchors

- `src/langs/router.ts` — `LangRouter` proxy, `__InternalParseLocale`, `SeyfertLocale` (`:60`), `ParseLocales` (`:62`)
- `src/langs/handler.ts` — `LangsHandler` (getKey, getLocale, get, aliases, defaultLang, preferGuildLocale, reload/reloadAll, onFile default/named-export handling)
- `src/langs/index.ts` — barrel (re-exported via `src/index.ts`)
- `src/commands/decorators.ts` — `Locales` (`:47`), `LocalesT` (`:61`), `TranslatedGroupDefinition` (`:68`), `LocalizedGroupDefinition` (`:75`), `defineGroups` (`:85`), `GroupsT` (`:91`), `Groups` (`:107`), `Group` overload (`:151`)
- `src/commands/applications/options.ts` — option `locales` (`:31`,`:49`), choice `locales` (`:60`)
- `src/commands/applications/shared.ts` — `DefaultLocale`
- `src/commands/handler.ts:457-562` — `getLocales`, `getLocaleKey`, `parseGlobalLocales`, `parseCommandOptionLocales`, `parseCommandLocales` (choice locale resolution ~`:517`)
- `src/commands/applications/chatcontext.ts:72` — `ctx.t` getter (mirrored in menu/entry/component/modal contexts)
- `src/client/base.ts:1184` — `client.t(locale)`; `:354-359` setServices langs wiring; `:1394` `ServicesOptions.langs`

## Agent Guidance

- All decorators/factories/types here import from the `'seyfert'` root — no deep imports.
- Prefer the dot-path decorators (`@LocalesT`/`@GroupsT` + option/choice `locales`) when localizations live in lang files; use `@Locales`/`@Groups` only for inline one-off tuples.
- Register `SeyfertRegistry.langs` with `ParseLocales<typeof oneLocaleFile>` BEFORE expecting typed dot-paths or typed `ctx.t`.
- Configure runtime behavior with `client.setServices({ langs: { default, preferGuildLocale, aliases } })`, not the constructor.
- Runtime read pattern: string/array/nested → `ctx.t.path.get()`; function → `ctx.t.path(args).get()`; force locale → `.get('xx-YY')`.
- Missing-key behavior is forgiving: metadata localization silently skips (logs a warning); runtime `.get()` falls back to `defaultLang`. Neither throws under normal flow.
