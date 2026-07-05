# Supporting different languages (i18n)

Original source URL: https://seyfert-web-git-seyfert-v5-tiramisulabs.vercel.app/docs/learn/i18n/languages
Coverage reference: i18n-cache-recipes.md
Verification status: Source-verified (v5, the authoritative Seyfert source)

## Page Summary

Seyfert has a built-in i18n system. You place language modules in the `langs` directory (configured via `locations.langs`), each exporting a default object whose values are strings, nested objects, arrays, or functions returning strings. You register the locale shape into TypeScript with `ParseLocales<typeof <lang>>` under the `langs` key of the `SeyfertRegistry` augmentation (v5: NOT `DefaultLocale` directly — that type is now derived from the registry). You then read translations through `ctx.t` (interaction locale auto-selected) and `client.t(locale)`. The file name (minus extension) becomes the locale code.

## Key APIs (verified)

All of the following are re-exported from the root `'seyfert'` (`src/index.ts:44` -> `export * from './langs'`).

- `ParseLocales<T extends Record<string, any>> = T` — identity type used only to register the locale shape in `SeyfertRegistry` (`src/langs/router.ts:62`). It does NOT transform the object.
- `SeyfertLocale` — the accessor type of `ctx.t` / `client.t(...)`. Defined as `__InternalParseLocale<DefaultLocale> & { get(locale?: string): DefaultLocale }` (`src/langs/router.ts:60`). Named alias introduced so inferred return types reference `DefaultLocale` instead of collapsing to `{}` at build time (the "preserve ctx.t typing" commit 7411731). Each leaf exposes `.get(locale?)`; calling a function leaf returns `{ get(locale?): ReturnType }`.
- `DefaultLocale = SeyfertRegistry extends { langs: infer L extends Record<string, any> } ? L : {}` (`src/commands/applications/shared.ts:26`) — the registered locale object, `{}` when unregistered. **v5: derived from `SeyfertRegistry`, no longer augmented by hand.**
- `client.t(locale: string): SeyfertLocale` (`src/client/base.ts:1184`) -> `this.langs.get(locale)` returns a `LangRouter` proxy.
- `LangsHandler` (`src/langs/handler.ts:13`), instance at `client.langs` (`src/client/base.ts:182`). Notable members:
  - `values: Partial<Record<string, any>>` — loaded locales keyed by locale code.
  - `defaultLang?: string`, `preferGuildLocale = false`, `aliases: [string, LocaleString[]][]`, `onReload?`.
  - `get(userLocale)` -> resolves alias via `getLocale`, then `LangRouter(locale, defaultLang ?? locale, values)()`.
  - `getLocale(locale)` — maps an alias to its canonical locale (`handler.ts:23`).
  - `getKey(lang, message)` — dot-path lookup, returns the string or `undefined` (`handler.ts:27`).
  - `reload(lang)` / `reloadAll(stopIfFail=true)` — re-import lang files; **throw `SeyfertError('RELOAD_NOT_SUPPORTED')` under Cloudflare Workers** (`handler.ts:78`).
  - `onReload?: (locale, value) => void` — fired after a successful `reload`. NOTE: when plugins ship lang overlays the client binds this internally (`base.ts:446` `bindPluginLangReload`), so prefer reading `reload()`'s return value over assigning `onReload` yourself.
- `LangRouter(userLocale, defaultLang, langs)` (`src/langs/router.ts:4`) — proxy factory. `.get(locale)` resolves the value (calling function leaves with applied args); throws `SeyfertError('UNDEFINED_LOCALE')` if no locale at all and `INTERNAL_ERROR` if a locale is missing, but internally falls back to `defaultLang` on failure (try/catch in `getValue`).
- `client.setServices({ langs: { default, preferGuildLocale, aliases } })` (`ServicesOptions.langs`, `src/client/base.ts:1394`): `default?: string`, `preferGuildLocale?: boolean`, `aliases?: Record<string, LocaleString[]>`. The service key is `default`, mapped onto `langs.defaultLang` (`base.ts:357`); `preferGuildLocale` onto the handler field (`base.ts:358`).
- `@Locales({ name, description })` (`src/commands/decorators.ts:47`) — manual `[LocaleString, string][]` localizations -> `name_localizations` / `description_localizations`.
- `@LocalesT(name?, description?)` (`decorators.ts:61`) — i18n-key-driven localization; args are `FlatObjectKeys<DefaultLocale>` dot-paths, stored as `__t`.
- Group localization (v5): `GroupsT(groups)` / `Groups(groups)` decorators and `defineGroups({...})` (`decorators.ts:85`); group def types `GroupDefinition`, `GroupDefinitions`, `LocalizedGroupDefinition`, `TranslatedGroupDefinition` (`TranslatedGroupDefinition.name/description` are `FlatObjectKeys<DefaultLocale>`).

## Code Examples (verified)

seyfert.config.mjs — set the langs directory:

```ts
// @ts-check
import { config } from 'seyfert';

export default config.bot({
  token: process.env.BOT_TOKEN ?? '',
  intents: ['Guilds'], // v5: inline string intents
  locations: {
    base: 'dist',
    commands: 'commands',
    events: 'events',
    langs: 'languages', // src/languages -> compiled to dist/languages
  },
});
```

languages/en.ts — default export; values are string | object | array | function:

```ts
export default {
  hello: 'Each key/value pair is a translation',
  foo: {
    bar: 'You may nest objects',
    baz: () => 'You may use functions for logic',
    ping: ({ ping }: { ping: number }) => `The ping is ${ping}`,
  },
  // multi-arg function leaf — called as t.welcome(user, count)
  welcome: (user: string, count: number) =>
    `Welcome ${user}, you are member #${count}`,
  qux: ['list item 1', 'list item 2'].join('\n'),
  // i18n keys reused by @LocalesT for command name/description localization
  commands: {
    ping: { name: 'ping', description: 'Show the ping with discord' },
  },
};
```

languages/es-ES.ts — `satisfies typeof English` keeps shapes in sync:

```ts
import type English from './en';

export default {
  hello: 'Hola, mundo!',
  foo: {
    bar: 'Puedes anidar objetos',
    baz: () => 'Puedes usar funciones para lógica',
    ping: ({ ping }) => `El ping es ${ping}`,
  },
  welcome: (user, count) => `Bienvenido ${user}, eres el miembro #${count}`,
  qux: ['elemento 1', 'elemento 2'].join('\n'),
  commands: {
    ping: { name: 'ping', description: 'Muestra la latencia con discord' },
  },
} satisfies typeof English;
```

src/index.ts — register the locale shape (v5: `SeyfertRegistry`, key `langs`):

```ts
import type English from './languages/en';
import { Client, type ParseClient, type ParseLocales } from 'seyfert';

const client = new Client();
client.start();

declare module 'seyfert' {
  interface SeyfertRegistry {
    client: ParseClient<Client<true>>;
    langs: ParseLocales<typeof English>;
  }
}
```

Set a default locale (and optionally prefer guild locale + aliases):

```ts
client.setServices({
  langs: {
    default: 'en',
    preferGuildLocale: true, // resolve guild locale before the user's
    // map non-standard / alternate codes onto a canonical locale file
    aliases: {
      en: ['en-US', 'en-GB'], // discord 'en-US'/'en-GB' -> your en.ts
      es: ['es-ES', 'es-419'],
    },
  },
});
```

Using translations in a command (auto locale + forced locale):

```ts
import {
  Command, Declare, Options,
  createBooleanOption, createStringOption,
  MessageFlags, type CommandContext,
} from 'seyfert';

const options = {
  hide: createBooleanOption({ description: 'Hide command output' }),
  language: createStringOption({
    description: 'Language to respond in',
    choices: [
      { name: 'English', value: 'en' },
      { name: 'Spanish', value: 'es' },
    ] as const, // v5: choices are readonly -> use `as const`
  }),
};

@Declare({ name: 'ping', description: 'Show the ping with discord' })
@Options(options)
export default class PingCommand extends Command {
  async run(ctx: CommandContext<typeof options>) {
    const flags = ctx.options.hide ? MessageFlags.Ephemeral : undefined;
    const ping = ctx.client.gateway.latency;

    // ctx.t auto-selects the interaction locale; .get(code) forces one
    const t = ctx.options.language ? ctx.t.get(ctx.options.language) : ctx.t.get();
    await ctx.write({ content: t.foo.ping({ ping }), flags });
  }
}
```

Note: `ctx.t` (no `.get`) resolves to `client.t(<locale>)`, where the locale is
`preferGuildLocale ? (interaction.guildLocale ?? interaction.locale ?? defaultLang ?? 'en-US') : (interaction.locale ?? defaultLang ?? 'en-US')` (`src/commands/applications/chatcontext.ts:72`; mirrored in entrycontext/menucontext/componentcontext/modalcontext).

### Function leaves: per-leaf `.get(locale?)`

A function leaf is invoked at the leaf, then resolved with `.get()`. Both styles work:

```ts
// auto locale (no arg to .get())
const line = ctx.t.welcome('Alice', 42).get();
// forced locale
const lineEs = ctx.t.welcome('Alice', 42).get('es');
// object-arg leaf
const pingLine = ctx.t.foo.ping({ ping: 12 }).get();
```

When you already grabbed a resolved object via `ctx.t.get(locale)`, leaves are plain
values/functions of `DefaultLocale` — call them directly: `t.foo.ping({ ping })`.

### Localizing a command's name/description via i18n keys (`@LocalesT`)

```ts
import { Command, Declare, LocalesT } from 'seyfert';

@Declare({ name: 'ping', description: 'Show the ping with discord' })
// dot-paths into DefaultLocale; Seyfert builds name_localizations/description_localizations
@LocalesT('commands.ping.name', 'commands.ping.description')
export default class PingCommand extends Command {
  async run(ctx: CommandContext) {
    await ctx.write({ content: ctx.t.foo.baz().get() });
  }
}
```

Manual localizations without i18n keys use `@Locales`:

```ts
import { Command, Declare, Locales } from 'seyfert';

@Declare({ name: 'ping', description: 'Show the ping with discord' })
@Locales({
  name: [['es-ES', 'latencia'], ['fr', 'latence']],
  description: [['es-ES', 'Muestra la latencia'], ['fr', 'Affiche la latence']],
})
export default class PingCommand extends Command {}
```

### Reading translations outside a context (events, services)

```ts
import { createEvent } from 'seyfert';

export default createEvent({
  data: { name: 'guildMemberAdd' },
  async run([member], client) {
    // pick a locale yourself; fall back to the configured default
    const t = client.t(client.langs.defaultLang ?? 'en');
    const text = t.welcome(member.user.username, member.guild?.memberCount ?? 0).get();
    // ...send welcome
  },
});
```

### Hot-reloading lang files in development

```ts
// re-import one locale (returns the new value, or null if unknown / not loaded)
const updated = await client.langs.reload('es-ES');

// re-import every loaded locale; stopIfFail=false keeps going past a broken file
await client.langs.reloadAll(false);
// NOTE: both throw SeyfertError('RELOAD_NOT_SUPPORTED') on Cloudflare Workers.
```

## Common patterns / gotchas

- Register exactly ONE `langs: ParseLocales<typeof <lang>>` in `SeyfertRegistry` — any locale module works since shapes are identical, but pick your default/base locale as the single source of truth.
- **End every non-default locale file with `satisfies typeof <BaseLang>`** (the base locale itself stays unconstrained). This catches, at compile time, three classes of drift that are otherwise silent runtime bugs in that one language: (1) missing keys, (2) misplaced/excess keys, (3) **function/placeholder-shape mismatch** — a leaf that is `(t) => \`...${t.x}...\`` in the base but a plain `string` in the translation (commonly from a malformed placeholder like `` `x}` `` missing its `{`), or an object-arg leaf with different param keys. Calling such a leaf (`ctx.t.a.b({ x }).get()`) only fails for the drifted locale, so it slips through untested locales — `satisfies` turns it into a build error.
- **Type function-leaf params concretely — never `any`.** A parameterized leaf is `({ x }: { x: string | number }) => \`...${x}...\`` (use `string | number` for interpolated placeholders — that is what template literals accept and what call sites pass). Typing params `any` (tempting when a locale is machine-generated) silently accepts wrong call-site values: an array injects a `,`-joined string, an `undefined` renders the literal text `"undefined"`, etc. `string | number` turns those into compile errors at the call site (e.g. forces `list.join(', ')` or `value ?? fallback`). This pairs with the `satisfies` rule above — together they keep both the shape and the call sites honest.
- Always set a `default` locale. Without it, `ctx.t` for a locale with no matching file can hit `LangRouter` with no resolvable locale and throw `UNDEFINED_LOCALE`/`INTERNAL_ERROR`. With a default, `getValue` falls back internally.
- File name (minus extension) IS the locale code (`parse()` -> `file.name.split('.').slice(0,-1)`). Name files to Discord locale strings (`en-US.ts`, `es-ES.ts`) OR wire alternate codes through `aliases` so e.g. `en-GB` resolves to `en.ts`.
- `aliases` is `Record<string, LocaleString[]>` in `setServices` (canonical -> list of alias codes) but stored as `[string, LocaleString[]][]` on the handler; `getLocale(alias)` returns the canonical key.
- `preferGuildLocale` only changes which locale `ctx.t` (no-arg) selects; `ctx.t.get('xx')` always forces `xx`.
- `getKey` only returns leaves that are `string` — function/array/object leaves return `undefined` from it (it is for flat key lookup, not the proxy path).
- v5 augmentation: do NOT `declare interface DefaultLocale` — augment `SeyfertRegistry.langs`. `DefaultLocale`, `UsingClient`, etc. are derived.

## Doc vs Source Corrections

- Docs imply each lang module MUST have a default export. Source: `onFile` (`handler.ts:109-152`) also accepts a single named object export (no default) as a fallback with a warning; ambiguous/invalid modules are skipped with explicit warnings. Default export remains the recommended path.
- Docs show registration against `DefaultLocale`/older interfaces. v5 SOURCE: register under `SeyfertRegistry.langs` (`commands/applications/shared.ts:26` derives `DefaultLocale` from it). Fixed.
- Docs only show `default`/`preferGuildLocale` indirectly. Source `ServicesOptions.langs` also supports `aliases: Record<string, LocaleString[]>` (`base.ts:1397`) mapping alternate locale codes via `getLocale`.
- `choices` arrays are readonly in v5 — examples now use `as const` (changelog: readonly option arrays).
- Otherwise aligned: `ParseLocales`, `ctx.t.get(lang)`, function-leaf calling, `client.t(locale)`, and `setServices({ langs: { default } })` all match source.

## Source Anchors

- `src/langs/index.ts`, `src/index.ts:44` (barrel exports)
