# Seyfert Source Truth And Coverage

## Provenance

This skill was built from three inputs:

- Rendered Vercel docs at `https://seyfert-web-git-seyfert-v5-tiramisulabs.vercel.app/docs/...`.
- The raw docs MDX from `tiramisulabs/seyfert-web` branch `seyfert-v5` (`content/docs/**.mdx`) — faithful code examples, fetched per page.
- Authoritative Seyfert source, chosen per task: the target project's installed `node_modules/seyfert`, the current HEAD of a Seyfert source checkout when working on the framework, or Seyfert source explicitly provided by the user.

For **core** APIs, that authoritative source wins over docs. For **external** packages (`@slipher/*`, `yunaforseyfert`, lavalink, `@slipher/testing`), verify the actual installed package before using docs-only APIs.

## How To Use This File

`references/pages/**` has a per-URL note with a "Doc vs Source Corrections" section for each page. This file is the **curated, cross-cutting** correction set — the high-impact items that bite across many tasks. Read it before trusting any docs example for a typed API.

## Local Source Landmarks

Resolve conflicts from these files in the installed package, current Seyfert source checkout, or source explicitly provided by the user:

- Root exports / helpers: `src/index.ts` — re-exports client/api/builders/cache/commands/components/events/langs/structures/types and defines `createEvent`, `config.bot`, `config.http`, `extendContext`.
- Clients: `src/client/{base,client,httpclient,workerclient,index,intents,types,transformers,collectors}.ts`.
- Commands: `src/commands/{index,decorators,handler,handle,optionresolver,basecontext}.ts` and `src/commands/applications/{chat,chatcontext,menu,menucontext,entryPoint,entrycontext,options,shared}.ts`. **`applications/shared.ts` is the key types hub** (middleware types, `SeyfertChannelMap`, `ParseClient`, `UsingClient`, `GlobalMetadata`, `InternalOptions`, `IgnoreCommand`).
- Components & modals: `src/components/{componentcommand,modalcommand,componentcontext,modalcontext,interactioncontext,handler,index}.ts` + per-component builders.
- Builders: `src/builders/**` (`Embed`, `Button`, `ActionRow`, `SelectMenu`, `Modal`, `Container`, `Section`, `TextDisplay`, `MediaGallery`, `Separator`, `Thumbnail`, `File`, `Attachment`, `Poll`, `Label`, `Checkbox`, `RadioGroup`). `Formatter` lives at `src/common/it/formatter.ts`.
- Events: `src/events/{event,handler,utils,index}.ts`, `src/events/hooks/**`.
- Plugins: `src/client/plugins.ts` + `src/client/plugins/{types,api,registry,shared,order,errors,index}.ts`.
- Langs/i18n: `src/langs/{handler,router,index}.ts` (the `SeyfertLocale` proxy + `ctx.t` typing live in `router.ts`).
- Cache: `src/cache/{index,adapters/**,resources/**}.ts`.
- API shorters/proxy: `src/api/{api,Router,bucket,shared}.ts` + `src/api/Routes/**` (route **types** only).
- Structures: `src/structures/**` (channel guards in `channels.ts`).
- Tests / type contracts: `tests/**` (e.g. `command-context-client-type.test.mts`, `plugin-api.test.mts`, `plugins.test.mts`, `root-export-contract.ts`).

## Known Docs Corrections (curated, source-cited)

Apply these unless the *installed* version proves otherwise.

### Module augmentation (BIGGEST change vs 5.0.0 / docs drafts)
- **Augment `interface SeyfertRegistry`, not `UsingClient`.** On v5 `UsingClient` is a derived **type alias** (`commands/applications/shared.ts:43`) and can no longer be interface-merged. Register typing via:
  ```ts
  declare module 'seyfert' {
    interface SeyfertRegistry {
      client: ParseClient<Client<true>>;     // or ParseClient<HttpClient> / ParseClient<WorkerClient<true>>
      middlewares: typeof middlewares;        // the middlewares object type DIRECTLY
      langs: typeof defaultLang;              // see i18n
    }
  }
  ```
  (`SeyfertRegistry` declared at `client/plugins/types.ts:32`, resolved in `shared.ts:30`.)
- **`ParseMiddlewares` no longer exists** — register the middlewares object type directly (the `middlewares` field is constrained to `Record<string, AnyMiddlewareContext>`). `ParseGlobalMiddlewares<typeof globalMiddlewares>` *does* still exist for augmenting `GlobalMetadata`.
- Registered client types are preserved via a `ParseClient` brand: `ParseClient<T> = T & { readonly [__seyfertClientType]?: T }` (`shared.ts:29,48`). Forgetting the `SeyfertRegistry.client` augmentation degrades typing to `BaseClient`.
- Still-separate augmentable **interfaces**: `GlobalMetadata`, `ExtendContext`, `ExtraProps`, `InternalOptions` (`withPrefix`, `asyncCache`), `ExtendedRC`/`ExtendedRCLocations`, `CustomEvents`, `CustomStructures`. `Cache` is a runtime **class**, not a `SeyfertRegistry` key; type custom cache resources with an explicit `interface Cache { ... }` augmentation or a local cast at the access site.

### Middleware
- The callback arg is `{ context, next, stop }` — **there is no `pass`** (`shared.ts:55` `MiddlewareContext`; `StopFunction = (error?: string | null) => void`). Removed before the current verified source (commit `22eb832` "replace middleware pass() with stop()").
- `stop("reason")` denies → routes to `onMiddlewaresError`. `stop()` / `stop(null)` **silently skips** (no run, no error) — this is the old `pass()`. `next(data)` attaches typed metadata read via `ctx.metadata.<name>`.
- A middleware that **throws** (sync) or returns a **rejected promise** (async) rejects the runner → `onInternalError` (with the thrown error) — it is an internal error, NOT a denial, and the runner does not log it. Only `stop('reason')` is a denial. (Earlier a sync throw → `onInternalError` but an async rejection diverged to `onMiddlewaresError`; now unified. `chat.ts` `__runMiddlewares`.)

### Commands & options
- **Options accept readonly arrays** (commit "accept readonly option arrays"): `choices` and `channel_types` are `readonly` (`commands/applications/options.ts:39,140`); `@Options(...)` / `@Middlewares([...])` accept readonly arrays. `as const` choices work.
- Context-menu commands: `@Declare` **must set `type` explicitly** (`ApplicationCommandType.User` / `.Message`) and the menu declare variant **omits `description`** (`Omit<DecoratorDeclareOptions,'description'>`, `decorators.ts:26-28`). Use `MenuCommandContext` + `ctx.target`.
- Autocomplete callbacks must call `interaction.respond([...])` — calling `reply` throws.
- Custom prefix parsers subclass `HandleCommand`, which is **not root-exported**: `import { HandleCommand } from 'seyfert/lib/commands/handle'`. Register via `client.setServices({ handleCommand: CustomHandleCommand })`.
- Prefix commands need `declare module 'seyfert' { interface InternalOptions { withPrefix: true } }`; `ctx.message` is only present on real prefix invocations; prefix `ctx.modal(...)` throws (text commands cannot open Discord modals).
- Channel narrowing (commit "add channel narrowing guards"): `CommandContext.inGuild()` narrows to a guild context; channel structures expose `this is X` guards in `structures/channels.ts` — `isTextable`, `isGuildTextable`, `isDM`, `isThread`, `isThreadOnly`, `isVoice`, `isTextGuild`, `isStage`, `isMedia`, `isForum`, `isNews`, `isCategory`, `isDirectory`, `isGuild`, `isNamed`. **There is no `isSendable()`.** `createChannelOption` `channel_types` are keyed by `SeyfertChannelMap` (`shared.ts:138`).

### i18n
- `ctx.t` returns a named `SeyfertLocale` proxy (commit "preserve ctx.t typing by returning a named SeyfertLocale alias"): `type SeyfertLocale = __InternalParseLocale<DefaultLocale> & { get(locale?: string): DefaultLocale }` (`langs/router.ts:60`). Resolve a value with `ctx.t.some.path.get(locale?)`. `DefaultLocale` is derived from `SeyfertRegistry.langs` (`shared.ts:26`) — do not augment `DefaultLocale` directly.

### Builders / formatting
- `Formatter.codeBlock(content, language = 'txt')` — **content is the FIRST argument** (`common/it/formatter.ts:117`). Docs showing `codeBlock('ts', '...')` are wrong; use `codeBlock('const x = 1', 'ts')`.

### Clients / runtime
- `start()` does **not** upload commands; `client.uploadCommands(...)` is separate (`base.ts`).
- `seyfert.config` resolves `.js/.mjs/.cjs/.ts/.mts/.cts` from `process.cwd()` (not only `.mjs`). `config.http` requires `publicKey` **and** `applicationId`, omits `events` from `locations`, and defaults `port` to `8080`; `config.bot` has no default port. A client option `getRC` can bypass file-based config.
- `RuntimeConfig.locations` only requires `base`; `commands`/`langs`/`events`/`components` are optional.
- `Logger` is root-exported, but `LogLevels`, `LoggerOptions`, and logger callback types come from `seyfert/lib/common` in the current root barrel. Do not import `LogLevels` from root unless the installed package proves it is exported there.

### Events
- Event `run` return type is **`Awaitable<unknown>`** (widened from `Awaitable<void>`; `createEvent` `src/index.ts:70`, `ClientEvent` `events/event.ts:32`, `EventValues` `events/handler.ts:49`), so handlers may `return` a value. `CallbackEventHandler` was already `=> unknown`. Landed in dev builds ≥ `28477111245` (older builds/docs still say `void`).
- Custom events: `client.events.runCustom(name, ...args)` (alias `emit`); the **client is auto-injected** as the last arg of `run` (don't pass it). Built-in custom events include `commandsLoaded`, `componentsLoaded`, `uploadCommands` (`events/event.ts:7`). An event file may default-export an **array** of `createEvent` objects. `events.reload*` throws `SeyfertError('RELOAD_NOT_SUPPORTED')` on Cloudflare Workers.

### Plugins
- Lifecycle: `register(api)` (sync, at construction) → `setup(client, api?)` (async, in `start()`) → `teardown(client, api?)` (async, in `close()`, reverse order). Hook keys are exact, e.g. `commands:afterLoad` (not `commandsLoaded`).
- `PluginLoadedMetadata` is passed to **`commands:afterLoad` and `components:afterLoad` only** (`types.ts:129`); `events:afterLoad` receives `[client, dir]` (`types.ts:208`).
- Contribution ordering (`order.ts` `orderBand`): `Before` (band 0) → any `number` (band 1, ascending) → unspecified/default (band 2) → `After` (band 3); ties by registration sequence. `0` is a normal number in band 1, **not** the "default".
- Plugin errors `SeyfertPluginError` / `SeyfertPluginAggregateError` are root-exported; catch BOTH with `instanceof` (the aggregate is thrown when multiple setup/teardown errors occur).

### Sharding & misc
- Sharding `mode` values are `'clusters'` (plural), `'threads'`, `'custom'` — **not `'cluster'`** (`websocket/discord/shared.ts`).
- Gateway intent helpers preserve the misspellings `NonPrivilaged` / `OnlyPrivilaged` — match the source spelling.
- The REST default user-agent string may still say `v4.0.0` — not a version signal.
- `Routes` from `seyfert` are **types only** (no runtime value export); the runtime route proxy is `client.proxy` (getter at `api/api.ts:139`, `Router` in `api/Router.ts`).
- `@slipher/testing` and official `@slipher/*` plugins are **not** in this repo — verify package availability and version in the target project.

## Import Policy

Prefer root imports:

```ts
import {
  Client, Command, SubCommand, ContextMenuCommand,
  CommandContext, MenuCommandContext,
  Declare, Options, Middlewares, AutoLoad, Group, Groups,
  createStringOption, createIntegerOption, createChannelOption,
  Embed, Button, ActionRow, Modal, Container, TextDisplay,
  createEvent, createMiddleware, extendContext,
  ApplicationCommandType, ChannelType, MessageFlags,
  type ParseClient, type ParseGlobalMiddlewares,
} from 'seyfert';
```

Use a deep import only when the installed package does not expose the API at root and the project already accepts deep imports:

```ts
import { HandleCommand } from 'seyfert/lib/commands/handle';
```

Before adding a deep import, inspect `lib/index.d.ts`, `src/index.ts`, or the package barrels.
